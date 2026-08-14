---
{"dg-publish": true, "dg-permalink": "tags", "permalink": "/tags/", "hideInGraph": true, "pinned": true, "title": "Tags", "dgShowInlineTitle": true, "dgEnableSearch": true, "dg-note-properties": {"title": "Tags"}}
---



# Tags

所有标签按字母排序，点击可搜索相关笔记。

```dataview
LIST WITHOUT ID
  "<span style='font-size:1.1em'>" + tag + "</span> (" + length(rows) + ")"
FLATTEN tags AS tag
FROM -"Templates"
WHERE tag
GROUP BY tag
SORT tag ASC
```
