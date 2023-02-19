---
title: "Complex Counts in Hive"
date: 2012-03-28T00:00:00-00:00
tags: ["hive", "sql"]
aliases:
    - /blog/2012/3/28/complex-counts-in-hive
    - /blog/2012/3/28/complex-counts-in-hive/
    - /blog/2012/3/28/complex-counts-in-hive.html
---

This came up on the Hive mailing list and I'm putting it here as a reminder to try it out. Here's how to do complex count statements to simplify queries.

```SQL
SELECT
    type
  , count(*)
  , count(DISTINCT u)
  , count(CASE WHEN plat=1 THEN u ELSE NULL END)
  , count(DISTINCT CASE WHEN plat=1 THEN u ELSE NULL END)
  , count(CASE WHEN (type=2 OR type=6) THEN u ELSE NULL END)
  , count(DISTINCT CASE WHEN (type=2 OR type=6) THEN u ELSE NULL END)
FROM
    t
WHERE
    dt in ("2012-1-12-02", "2012-1-12-03")
GROUP BY
    type
ORDER BY
    type
;
```