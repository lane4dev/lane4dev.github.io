---
layout: post
title: "SQLite + simple 插件实现中文全文检索"
date: 2024-06-11 00:00:00 +0800
categories: Code
tags: ["SQLite", "全文检索"]
comments: true
---

做中文全文检索这件事，我一开始也以为离不开 Elasticsearch、Postgres + 插件这些“大件”。
但很多时候，我只是想在一颗 SQLite 里，把几万条新闻丢进去，安安静静地搜几个中文关键词——不想搭集群、不想运维服务，
更不想为一个小工具搞一套“大数据”基础设施。

后来发现 wangfenjin/simple 这个扩展，配合 FTS5，居然可以在 SQLite 里把中文检索做得又轻又顺手。

这篇文章就记录一下我自己从能用就行到用得舒服的这条小路，希望也能帮你把手上的 SQLite 变成一台好用的中文搜索引擎。

## 总体思路

1. 安装并加载 `wangfenjin/simple` 这个 SQLite 扩展，它提供了适合中文的 `tokenize='simple'` 和 `simple_query()`。
2. 建一个**原始数据表**存新闻内容。
3. 用 FTS5 建一个**虚拟表**做全文索引，指向原始表。
4. 用触发器保证原始表和 FTS 索引表自动同步。
5. 用 `MATCH simple_query('关键词')` 直接查。

---

## 加载 simple 插件

Python 侧流程大致是这样：

```python
import os
import sqlite3

db_path = os.path.join("<db-path>")
ext_path = os.path.join("<ext-path>")

# 连接数据库
conn = sqlite3.connect(db_path)

# 导入插件
conn.enable_load_extension(True)
conn.load_extension(ext_path)

cursor = conn.cursor()

cursor.execute(TABLE_SQL)
cursor.execute(SEARCH_TABLE_SQL)
cursor.executescript(TRIGGER_SQL)
conn.commit()
```

**小记**

- `ext_path`：指向编译出来的 `simple` 动态库（例如 `.so` / `.dylib` / `.dll`），具体路径自己填；
- `enable_load_extension(True)` 必须在 `load_extension()` 之前调用；
- 一般只需在应用启动时加载一次扩展即可。

---

## 原始数据表：`news`

原始新闻数据放在普通表里：

```sql
CREATE TABLE IF NOT EXISTS news(
    id INTEGER NOT NULL PRIMARY KEY,
    title TEXT NOT NULL,
    text TEXT,
    url TEXT,
    date TEXT,
    type TEXT,
    label INTEGER,
    timestamp INTEGER
);
```

这里的约定：

- `id`：主键，后面 FTS 表会用它作为 `rowid`；
- `title` / `text`：需要做全文检索的字段；
- 其他字段就是正常业务字段，随业务扩展。

---

## FTS5 虚拟表：`news_fts`

用于中文全文检索的索引表：

```sql
CREATE VIRTUAL TABLE IF NOT EXISTS news_fts USING fts5 (
    title,
    text,
    content=news,
    content_rowid=id,
    tokenize='simple'
);
```

这里几个关键点：

- `title, text`：和原表里需要检索的字段一一对应。

- `content=news`：

  告诉 FTS5：这张索引表是基于 `news` 这张表的内容。

- `content_rowid=id`：

  指明 `news` 表里用哪个字段对应 FTS 的 `rowid`，这里就是 `news.id`。

- `tokenize='simple'`：

  使用 simple 插件提供的分词器，对中文更友好。
  不用自己写分词逻辑。

---

## 用触发器保持索引同步

为了不每次增删改都手动维护 `news_fts`，直接用触发器做**跟随更新**。

```sql
-- 插入时：把新内容写进 FTS 索引
CREATE TRIGGER IF NOT EXISTS news_fts_i AFTER INSERT ON news
BEGIN
    INSERT INTO news_fts(rowid, title, text)
    VALUES (new.id, new.title, new.text);
END;

-- 删除时：通知 FTS 删除对应记录
CREATE TRIGGER IF NOT EXISTS news_fts_d AFTER DELETE ON news
BEGIN
    INSERT INTO news_fts(news_fts, rowid, title, text)
    VALUES ('delete', old.id, old.title, old.text);
END;

-- 更新时：先删旧，再插新
CREATE TRIGGER IF NOT EXISTS news_fts_u AFTER UPDATE ON news
BEGIN
    INSERT INTO news_fts(news_fts, rowid, title, text)
    VALUES ('delete', old.id, old.title, old.text);
    INSERT INTO news_fts(rowid, title, text)
    VALUES (new.id, new.title, new.text);
END;
```

有了这三类触发器：

只维护 `news` 表即可，
`news_fts` 会自动跟上，无需额外代码。

---

## 查询示例：用 `simple_query` 做中文检索

实际查询的时候，从原始表出发，再 JOIN 索引表：

```sql
SELECT *
FROM news
JOIN news_fts ON news.id = news_fts.rowid
WHERE news_fts MATCH simple_query('山东');
```

**要点：**

- `news_fts MATCH simple_query('山东')`：

  `simple_query()` 由插件提供，负责按中文分词规则构造查询表达式。
  参数就是你要搜的中文关键词。

- 通过 `JOIN news`：

  可以一次性拿到全文索引 + 原始表所有字段，
  前端展示起来比较方便。

如果只想看命中的 `id` 和标题，也可以简化成：

```sql
SELECT news.id, news.title
FROM news
JOIN news_fts ON news.id = news_fts.rowid
WHERE news_fts MATCH simple_query('山东');
```

---

## 备注

- **扩展路径问题**

  在 Python 里 `ext_path` 建议用绝对路径，避免工作目录变化导致找不到库。

- **事务**

  触发器会在同一个事务里运行，记得 `INSERT/UPDATE/DELETE` 完要 `commit()`。

- **批量导入**

  大批量导数据时，可以先关闭触发器、用 `INSERT INTO news_fts(...) SELECT ... FROM news;` 一次建索引，之后再打开触发器。

- **调试查询**

  如果怀疑分词或命中有问题，可以先查 `news_fts` 看实际存储情况，再根据 `rowid` 回查 `news`。
