---
{"dg-publish":true,"dg-permalink":"home","permalink":"/home/","title":"知识花园","hideInGraph":true,"pinned":true,"tags":["gardenEntry"],"dgEnableSearch":true,"created":"2026-04-15","updated":"2026-04-15","dg-note-properties":{"title":"知识花园","created":"2026-04-15","updated":"2026-04-15"}}
---


# 欢迎来到我的知识花园

> 读、学、思、行。用 [[Notes/什么是双链笔记\|双链]] 连接知识，让想法自然生长。

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

按主题浏览 → [[MOC/MOC - 技术架构\|MOC - 技术架构]] · [[MOC/MOC - 效率工具\|MOC - 效率工具]] · [[MOC/MOC - 阅读笔记\|MOC - 阅读笔记]]

## 关于

本站基于 [Obsidian](https://obsidian.md) + [Digital Garden](https://github.com/oleeskild/obsidian-digital-garden) 构建。

所有笔记通过 `[[双链]]` 互相关联，支持知识图谱可视化。
