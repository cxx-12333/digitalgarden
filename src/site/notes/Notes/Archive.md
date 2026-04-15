---
{"dg-publish":true,"dg-permalink":"archive","permalink":"/archive/","title":"Archive","hideInGraph":true,"pinned":true,"dgShowInlineTitle":true,"dgEnableSearch":true,"dg-note-properties":{"title":"Archive"}}
---


# Archive

所有已发布笔记按时间倒序排列。新建笔记会自动出现在此处。

```dataview
TABLE WITHOUT ID
  file.link AS "标题",
  created AS "创建日期",
  updated AS "更新日期",
  choice(tags != null, join(map(tags, (t) => "#" + t), " "), "") AS "标签"
FROM -"Templates"
WHERE file.name != this.file.name AND created
SORT created DESC
```
