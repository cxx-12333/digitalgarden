---
{"dg-publish": true, "title": "Tags", "dgPermalink": "tags", "dgHideInGraph": true, "dgPinned": true, "dgShowLocalGraph": false, "dgShowBacklinks": false, "dgShowToc": false, "dgShowTags": false, "dgEnableSearch": true, "dgShowInlineTitle": true, "permalink": "/Notes/Tags/", "dg-note-properties": {"title": "Tags", "dg-permalink": "tags", "dg-hide-in-graph": true, "dg-pinned": true, "dg-show-local-graph": false, "dg-show-backlinks": false, "dg-show-toc": false, "dg-show-tags": false, "dg-enable-search": true, "dg-show-inline-title": true}}
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
