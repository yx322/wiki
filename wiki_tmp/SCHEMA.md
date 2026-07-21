# Wiki Schema

## Domain
分布式系统架构：嵌入式 KV 存储引擎、共识算法、对象映射框架、AI Agent 基础设施。
聚焦 Rust 生态（Fjall、SlateDB、SurrealKV、Openraft）替代 Redis/SQL 的工程实践。

## Conventions
- File names: lowercase, hyphens, no spaces (e.g., `fjall-storage-engine.md`)
- Every wiki page starts with YAML frontmatter (see below)
- Use `[[wikilinks]]` to link between pages (minimum 2 outbound links per page)
- When updating a page, always bump the `updated` date
- Every new page must be added to `index.md` under the correct section
- Every action must be appended to `log.md`
- 语言：简体中文

## Frontmatter
```yaml
---
title: Page Title
created: YYYY-MM-DD
updated: YYYY-MM-DD
type: entity | concept | comparison | query
tags: [from taxonomy below]
sources: [raw/articles/source-name.md]
confidence: high | medium | low
---
```

## Tag Taxonomy
- **引擎**: fjall, slatedb, surrealkv, rocksdb
- **共识**: raft, openraft, redlock
- **编码**: composite-key, key-encoding, okm, wiskey
- **架构**: gateway, agent-memory, cloud-native, embedded-kv
- **对比**: redis, sql, sqlite, orm
- **技术**: lsm-tree, mvcc, compaction, bitmap
- **语言**: rust

## Page Thresholds
- Create: entity/concept appears in 2+ sources OR is central to one source
- Add to existing: source mentions something already covered
- Split: page exceeds ~200 lines
- Archive: content fully superseded
