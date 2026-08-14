---
{"dg-home": true, "title": "知识花园", "created": "2026-04-15", "updated": "2026-04-15", "dg-permalink": "home", "dg-show-filetree": true, "dg-show-local-graph": false, "dg-show-backlinks": false, "dg-enable-search": true, "dg-show-tags": false, "dg-show-inline-title": false, "dg-pinned": true, "dg-hide-in-graph": true, "permalink": "/My%20first%20note/"}
---



# 欢迎来到我的知识花园

> 读、学、思、行。用 [[什么是双链笔记|双链]] 连接知识，让想法自然生长。

---

## 最近更新

```dataview
TABLE WITHOUT ID
  file.link AS "笔记",
  choice(tags != null, join(map(tags, (t) => "#" + t), " "), "") AS "标签",
  created AS "日期"
FROM -"Templates"
WHERE file.name != this.file.name AND created
SORT created DESC
LIMIT 10
```

## 知识地图

按主题浏览 → [[MOC - 技术架构]] · [[MOC - 效率工具]] · [[MOC - 阅读笔记]]

## 关于

本站基于 [Obsidian](https://obsidian.md) + [Digital Garden](https://github.com/oleeskild/obsidian-digital-garden) 构建。

所有笔记通过 `[[双链]]` 互相关联，支持知识图谱可视化。
