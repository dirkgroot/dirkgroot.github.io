---
layout: "post"
date: "2021-12-13 22:00:00 +0200"
title: "My first post"
---

Happy reading!

```clojure
(defn test [input]
  (lazy-seq (cons input (test input))))
```
