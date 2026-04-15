```dataview
LIST WITHOUT ID
  "<span style='font-size:1.1em'>" + tag + "</span> (" + length(rows) + ")"
FLATTEN tags AS tag
FROM -"Templates"
WHERE tag
GROUP BY tag
SORT tag ASC
```