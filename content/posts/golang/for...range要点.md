---
author: "李昌"
title: "for...range要点"
date: "2022-03-12"
tags: ["golang", "range"]
categories: ["Golang"]
ShowToc: true
TocOpen: true
---

1. range循环时，使用的是被迭代的元素的副本

```go
type T struct {
    n int
}

func main() {
    ts := [2]T{}
    for i, t := range ts {
        switch i {
        case 0:
            t.n = 3  // 被访问的是ts的副本
            ts[1].n = 9
        case 1:
            fmt.Print(t.n, " ")
        }
    }
    fmt.Print(ts)
}
```

```
输出：0 {{0} {9}}
```

2. range 循环语句使用的临时变量

```go
func main() {
        h := make([]*int, 3)
        u := []int{1, 2, 3}
        for i, v := range u {
                h[i] = &v
        }

        for i := range h {
                fmt.Println(*h[i])
        }
}
```

```
输出：
3
3
3
```

