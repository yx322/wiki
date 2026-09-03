# 操作自动化转 Skill 工作流

## 核心理念

将人工浏览器操作转化为可复用、可定时执行的自动化 Skill，通过"抓包录制 → AI 编译 → 双模执行"的三阶段工作流，实现从交互式操作到生产级自动化的完整转化。

## 工作流总览

```
┌─────────────────────────────────────────────────────────────────┐
│  阶段一：开发期（交互式抓包录制）                                  │
│  Playwright 控制浏览器 → 人工操作 → 自动捕获 API → JSON 文件      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  阶段二：转化期（AI 编译生成 Skill）                              │
│  JSON + Prompt → Hermes → 生成统一编排器（Orchestrator）代码      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  阶段三：生产期（双模降级执行）                                    │
│  编排器自动选择最优执行策略：                                      │
│  ├─ 主模式：HTTP 客户端（毫秒级，0% 浏览器开销）                   │
│  └─ 备用模式：Headless 浏览器（仿真人类行为，强力兜底）            │
└─────────────────────────────────────────────────────────────────┘
```

**关键设计**：双模降级只是执行策略，工作流的核心价值在于"把人工操作转化为可复用的 Skill"。

---

## 第一阶段：开发期——交互式抓包工具

运行脚本后，浏览器保持打开状态，直到关闭窗口或在 F12 控制台输入 `exit()`。自动捕获 API 并保存到 `captured_api_logs.json`。

### 环境安装

```bash
pip install playwright requests httpx
playwright install chromium
```

### 核心抓包脚本 (stage1_dev_capture.py)

```python
import os
import json
from playwright.sync_api import sync_playwright

OUTPUT_FILE = "captured_api_logs.json"
captured_data = []

def packet_handler(response):
    """网络请求拦截过滤器"""
    try:
        # 过滤策略：只捕获包含 /api/ 且状态码为 200 的请求，可根据实际目标修改
        if "api" in response.url and response.status == 200:
            request = response.request
            record = {
                "url": response.url,
                "method": request.method,
                "headers": dict(request.headers),
                "payload": request.post_data if request.post_data else None
            }
            captured_data.append(record)
            print(f"🚀 [已捕获接口]: {response.url} ({request.method})")
    except Exception:
        pass

def start_recording(target_url):
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=False, args=["--start-maximized"])
        context = browser.new_context(no_viewport=True)
        page = context.new_page()
        page.on("response", packet_handler)

        # 注入控制台安全退出函数
        page.add_init_script("""
        window.exit = function() {
            console.log("=== 收到退出指令，正在保存退出... ===");
            window.__playwright_exit__ = true;
            return "正在退出录制...";
        };
        """)

        print(f"\n🔗 正在打开网页: {target_url}")
        print(f"💡 提示：请在浏览器中完成操作。完成后【关闭窗口】或在【F12控制台输入 exit()】结束。")
        page.goto(target_url)

        try:
            while True:
                if page.is_closed() or page.evaluate("() => window.__playwright_exit__"):
                    break
                page.wait_for_timeout(500)
        except Exception:
            pass
        finally:
            context.close()
            browser.close()

        if captured_data:
            with open(OUTPUT_FILE, "w", encoding="utf-8") as f:
                json.dump(captured_data, f, indent=4, ensure_ascii=False)
            print(f"\n💾 [成功] 数据已保存至: {os.path.abspath(OUTPUT_FILE)}")

if __name__ == "__main__":
    start_recording("https://example.com")  # 替换为你的目标网址
```

---

## 第二阶段：转化期——提交给 Hermes 编译

打开生成的 `captured_api_logs.json`，将其内容复制，配合以下提示词发送给 Hermes。

### 给 Hermes 的标准 Prompt

**角色**：高级爬虫工程师与自动化专家。

**任务**：请根据我提供的原始抓包 JSON 数据，为我编写一个生产环境使用的双模降级自动化 Skill 代码。

**要求**：
1. 提供一个统一的对外接口函数 `hermes_orchestrator(username, password, ...)`
2. **主模式（优先）**：使用 `requests.Session()` 完全脱离浏览器直接请求捕获到的 API 接口，需带上对应的 Headers，并将动态参数（如时间戳）代码化
3. **备用模式（兜底）**：如果主模式抛出异常或业务失败，自动切换到 playwright 无头浏览器模式 (`headless=True`)，模拟人类点击页面元素

**抓包数据内容如下**：
```
[在此处粘贴你的 captured_api_logs.json 内容]
```

---

## 第三阶段：生产期——无头双模独立运行脚本

经由 Hermes 编译转换后的标准生产模板。具备完全的独立性，可直接部署在 Linux 服务器或云函数中定时触发。

### 生产环境双模 Skill 源码 (stage3_prod_skill.py)

```python
import time
import requests
from playwright.sync_api import sync_playwright

def run_with_http_client(username, password):
    """【主模式】纯 HTTP 客户端发起请求（无浏览器开销，毫秒级响应）"""
    print("▶️ [主模式] 尝试通过 HTTP 客户端发起原生请求...")
    
    session = requests.Session()
    
    # 从开发期复制而来的浏览器伪装头
    base_headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36...",
        "Content-Type": "application/json",
        "Accept": "application/json"
    }
    
    # 步骤 1: 直接请求登录 API
    login_url = "https://example.com"
    login_payload = {
        "username": username,
        "password": password,
        "timestamp": int(time.time() * 1000)  # 动态时间戳
    }
    
    # 设置 5 秒超时，防止主模式卡死
    response = session.post(login_url, json=login_payload, headers=base_headers, timeout=5)
    
    # 验证 HTTP 状态码与业务逻辑字段
    if response.status_code == 200 and response.json().get("success") == True:
        print("✅ [HTTP 成功] 接口直接调用成功！自动跳过无头浏览器。")
        
        # 步骤 2: 继续使用带 Cookie 的 session 请求数据
        data_url = "https://example.com"
        data_response = session.get(data_url, headers=base_headers, timeout=5)
        return data_response.json()
    else:
        raise Exception(f"主模式业务调用失败，状态码: {response.status_code}")

def run_with_headless_browser(username, password):
    """【备用模式】主模式失败后，自动启动无头浏览器模拟人类行为进行兜底"""
    print("⚠️ [备用模式] HTTP 客户端失败！正在拉起无头浏览器进行兜底保护...")
    
    with sync_playwright() as p:
        # headless=True 确保在 Linux 生产环境中不需要图形界面即可运行
        browser = p.chromium.launch(headless=True)
        context = browser.new_context()
        page = context.new_page()
        
        try:
            # 1. 导航到登录页
            page.goto("https://example.com", timeout=20000)
            
            # 2. 模拟人工输入与点击
            page.fill("input[type='text']", username)
            page.fill("input[type='password']", password)
            page.click("button[type='submit']")
            
            # 3. 等待页面跳转完成
            page.wait_for_url("**/dashboard", timeout=10000)
            
            # 4. 获取渲染后的数据
            result_text = page.locator("#data-container").inner_text()
            print("✅ [无头浏览器成功] 已通过模拟浏览器完成兜底操作！")
            return {"success": True, "data": result_text}
        
        except Exception as browser_err:
            print(f"❌ [双重失败] 无头浏览器模拟执行也宣告失败: {browser_err}")
            raise browser_err
        finally:
            context.close()
            browser.close()

def hermes_orchestrator(username, password):
    """【核心编排器】对外暴露的统一自动化 Skill 接口"""
    start_time = time.time()
    
    try:
        # 1. 优先尝试轻量级主模式
        result = run_with_http_client(username, password)
        print(f"⏱️ 任务总耗时: {round(time.time() - start_time, 2)} 秒")
        return result
    
    except Exception as http_err:
        print(f"📢 [主模式触发降级]: {http_err}")
        
        try:
            # 2. 主模式失败后，无缝降级到无头浏览器备用模式
            result = run_with_headless_browser(username, password)
            print(f"⏱️ 任务总耗时: {round(time.time() - start_time, 2)} 秒 (含浏览器启动开销)")
            return result
        except Exception as final_err:
            print("🚨 [严重故障] 两种模式均已失效！目标网站可能更新了风控或验证码。")
            return None

if __name__ == "__main__":
    # 生产环境调度入口：
    # final_result = hermes_orchestrator("admin", "secure_password")
    pass
```

---

## 方案优势

| 维度 | 价值 |
|------|------|
| **开发效率** | 不需要死记硬背 API 或配置 Postman。像正常用户一样在网页操作，Playwright 自动记录所有账密通信、接口协议 |
| **生产稳定性** | 网站突然更改接口加密算法时，主模式（HTTP）报错 → 系统瞬间拉起无头浏览器继续跑完业务，保证生产任务不中断，同时留出充裕时间重构 HTTP 逻辑 |
| **性能优化** | 主模式命中时，0% 浏览器内存消耗，毫秒级响应；仅在主模式失败时才承担浏览器启动开销 |
| **容错能力** | 双模互备，单一模式失效不会导致整体任务失败 |

---

## 关键决策点

### 何时触发降级？

- **HTTP 状态码异常**（非 200）
- **业务逻辑字段失败**（如 `response.json().get("success") != True`）
- **超时**（主模式设置 5 秒 timeout，防止卡死）

### 何时需要重构主模式？

- 备用模式连续成功 N 次（说明主模式已完全失效）
- 目标网站更新了风控/验证码/加密算法
- 主模式耗时 > 备用模式（说明 HTTP 逻辑已不适用）

### 生产环境部署注意事项

- **Linux 服务器**：确保 `headless=True`，无需图形界面
- **超时设置**：主模式 5 秒，备用模式 20 秒（浏览器启动开销）
- **日志监控**：记录每次降级事件，用于分析主模式失效率
- **定时任务**：可直接部署在 cron / 云函数中定时触发 `hermes_orchestrator()`

---

## 参考工作流

1. **开发期**：运行 `stage1_dev_capture.py` → 手动操作浏览器 → 生成 `captured_api_logs.json`
2. **转化期**：将 JSON + Prompt 发送给 Hermes → 生成 `stage3_prod_skill.py`
3. **生产期**：部署 `stage3_prod_skill.py` → 调用 `hermes_orchestrator()` → 自动双模降级
