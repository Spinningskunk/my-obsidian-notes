---
title: 后端技术栈 MOC
type: moc
tags:
  - moc
  - 后端
  - 技术
created: 2026-03-30
updated: 2026-03-30
---

# 后端技术栈 MOC

> [!abstract] Map of Content
> 后端技术知识体系索引，涵盖中间件、数据库、服务治理等核心技术栈

## 中间件技术

### 消息队列
- [[Kafka学习]]

### 搜索引擎
- [[ElasticSearch学习]]

### 数据库
- [[ClickHouse学习]]

### 服务治理
- [[Nacos学习]]

### 网络框架
- [[Netty]]

## 学习状态

> [!tip] Dataview 查询
> 使用以下查询追踪学习进度

### 学习中
```dataview
LIST
FROM #学习中
WHERE contains(file.tags, "后端")
```

### 已掌握
```dataview
LIST
FROM #已掌握
WHERE contains(file.tags, "后端")
```

### 待学习
```dataview
LIST
FROM #待学习
WHERE contains(file.tags, "后端")
```

## 相关索引
- [[Java技术栈MOC]]
- [[微服务架构MOC]]
- [[性能优化MOC]]
- [[知识体系总索引]]
