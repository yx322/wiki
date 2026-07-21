---
title: Lance 与 OSS 兼容性问题
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [lance, oss, troubleshooting, object-storage]
sources: [raw/articles/lance-oss-migration.md]
confidence: high
---

# Lance 与 OSS 兼容性问题

Lance 格式的 Rust `object_store` 对 S3 条件写入只实现了 `If-None-Match (etag)` 模式，阿里云 OSS 不支持此协议。

## 问题

```
NotImplemented: A header you provided implies functionality that is not implemented.
Header: If-None-Match
```

Lance 的 commit 逻辑默认使用 `If-None-Match: *` 头实现原子提交（条件写入），但阿里云 OSS 不支持。

## 尝试过的方案（均失败）

| 方案 | 结果 | 原因 |
|------|------|------|
| LegacyCommitHandler | 失败 | lance 版本没有这个类 |
| mock commit_lock | 失败 | Rust 层仍然发送条件写入请求 |
| AWS_CONDITIONAL_PUT = "disabled" | 失败 | Rust 层 `PutMode::Create` 未实现 |
| 自定义 header-with-status | 失败 | S3PutConditional 只接受 "etag" |

## 最终方案：先写本地再 oss2 上传

```python
# 1. Lance 写入本地临时目录（无问题）
lance.write_dataset(arrow_table, local_uri, mode=mode)

# 2. 用 oss2 SDK 上传到 OSS
for file_path in local_path.rglob("*"):
    bucket.put_object_from_file(oss_key, str(file_path))
```

## 核心结论

无法通过 storage_options 或 commit_lock 解决，只能绕过——先写本地文件系统，再用 oss2 上传。

## 参见
- [[oss]] — 阿里云对象存储
- [[lakehouse]] — 湖仓一体架构

^[raw/articles/lance-oss-migration.md]
