# Lance 格式转换问题排查记录

## 背景

将阿里云 OSS 上的 Delta Lake 数据（`oss/xmh-analysis/task_data/task_records/`）转换为 Lance 格式存储。

**脚本位置：** `D:\communicate-skills\skills\task\scripts\export_lance.py`

**依赖：** lance, polars, deltalake, oss2

---



---

## 问题 ：If-None-Match 不支持

**报错：**
```
NotImplemented: A header you provided implies functionality that is not implemented.
Header: If-None-Match
```

**原因：** Lance 的 commit 逻辑默认使用 `If-None-Match: *` 头实现原子提交（条件写入），但阿里云 OSS 不支持这个标准 HTTP 头。

**尝试 1：LegacyCommitHandler**
```python
from lance.commit import LegacyCommitHandler
lance.write_dataset(..., commit_handler=LegacyCommitHandler())
```
**结果：** 失败。`lance.commit` 模块中没有 `LegacyCommitHandler` 类。

**尝试 2：mock commit_lock**
```python
from contextlib import contextmanager

@contextmanager
def no_op_commit_lock(version: int):
    yield

lance.write_dataset(..., commit_lock=no_op_commit_lock)
```
**结果：** 失败。mock commit_lock 只是绕过了 Python 层的检查，但 Rust 层仍然会发送条件写入请求。

**尝试 3：AWS_CONDITIONAL_PUT = "disabled"**
```python
opts = {"AWS_CONDITIONAL_PUT": "disabled"}
```
**结果：** 失败。Lance 的 Rust 层报错：`Operation 'put_opts' with mode 'PutMode::Create' when conditional put is disabled not yet implemented`

**尝试 4：AWS_CONDITIONAL_PUT 用自定义头**
```python
opts = {"AWS_CONDITIONAL_PUT": "header-with-status:x-oss-forbid-overwrite:true:409"}
```
**结果：** 失败。Lance 的 `S3PutConditional` 只接受 `"etag"`，不支持 `header-with-status` 格式。

**根本原因：** Lance 的 Rust `object_store` 对 S3 的 `conditional_put` 只实现了 `etag` 模式（即 `If-None-Match`），而阿里云 OSS 不支持这个头。这是一个底层协议不兼容的问题，无法通过 storage_options 解决。

---

## 最终方案：先写本地再用 oss2 上传

Lance 写入本地文件系统没有问题，问题只出在写入 OSS 时的条件写入。因此采用两步走：

1. Lance 先写入本地临时目录
2. 用 `oss2` SDK 上传到 OSS

```python
def _write_lance_table(df, table_name, lance_uri, storage_options, mode="overwrite"):
    import lance
    import tempfile

    arrow_table = df.to_arrow()

    with tempfile.TemporaryDirectory() as tmp_dir:
        local_uri = str(Path(tmp_dir) / table_name)
        lance.write_dataset(arrow_table, local_uri, mode=mode)

        # 验证本地写入
        ds = lance.dataset(local_uri)
        row_count = ds.count_rows()

        # 用 oss2 上传到 OSS
        if lance_uri.startswith("s3://"):
            _upload_lance_to_oss(local_uri, lance_uri, table_name)


def _upload_lance_to_oss(local_uri, oss_uri, table_name):
    import oss2

    parts = oss_uri.replace("s3://", "").split("/", 1)
    bucket_name = parts[0]
    oss_prefix = parts[1] if len(parts) > 1 else ""

    auth = oss2.Auth(cfg.s3.access_key_id, cfg.s3.secret_access_key)
    endpoint = cfg.s3.duckdb_s3_endpoint
    bucket = oss2.Bucket(auth, endpoint, bucket_name)

    local_path = Path(local_uri)
    for file_path in local_path.rglob("*"):
        if file_path.is_file():
            rel_path = file_path.relative_to(local_path.parent)
            rel_str = str(rel_path).replace("\\", "/")
            if rel_str.startswith(f"{table_name}/"):
                rel_str = rel_str[len(table_name) + 1:]
            oss_key = f"{oss_prefix}/{rel_str}" if rel_str else oss_prefix
            bucket.put_object_from_file(oss_key, str(file_path))
```

---

## 其他修复

### 输出路径修正

用户要求输出到 `s3://xmh-analysis/task_data/lance_data/`，默认前缀从 `{prefix}_lance` 改为 `{prefix}/lance_data`。

### storage_options 精简

由于写入走本地 + oss2，storage_options 中的 `AWS_COPY_IF_NOT_EXISTS` 和 `AWS_CONDITIONAL_PUT` 不再需要，只保留读取所需的配置：

```python
opts = {
    "AWS_ACCESS_KEY_ID": cfg.s3.access_key_id,
    "AWS_SECRET_ACCESS_KEY": cfg.s3.secret_access_key,
    "AWS_ENDPOINT_URL": cfg.s3.deltalake_endpoint_url,
    "AWS_REGION": cfg.s3.region,
    "AWS_VIRTUAL_HOSTED_STYLE_REQUEST": "true",
    "allow_http": "true" if not cfg.s3.use_ssl else "false",
}
```

---

## 最终验证结果

```
[task_records]
  ✓ 读取 1868182 行, 20 列
  ✓ 本地写入 1868182 行
  ✓ 已上传到 OSS: s3://xmh-analysis/task_data/lance_data/task_records

[user_sync_cursors]
  ✓ 读取 3 行, 2 列
  ✓ 本地写入 3 行
  ✓ 已上传到 OSS: s3://xmh-analysis/task_data/lance_data/user_sync_cursors
```

---

## 结论

| 问题 | 根因 | 解决方案 |
|------|------|----------|
| Bucket 识别错误 | endpoint 不带 bucket 子域 | 用 `deltalake_endpoint_url` |
| If-None-Match 不支持 | 阿里云 OSS 不支持标准条件写入 | 先写本地再 oss2 上传 |
| LegacyCommitHandler 不存在 | lance 版本没有这个类 | 放弃，走本地上传方案 |
| AWS_CONDITIONAL_PUT 无效 | Lance Rust 层只支持 etag 模式 | 放弃，走本地上传方案 |

**核心结论：** Lance 的 Rust `object_store` 对 S3 条件写入只实现了 `If-None-Match (etag)` 模式，阿里云 OSS 不支持此协议。无法通过 storage_options 或 commit_lock 解决，只能绕过——先写本地文件系统，再用 oss2 上传到 OSS。