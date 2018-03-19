---
title: golang实现单链表反转
date: 2018-03-12 17:03:21
tags: [golang, 数据结构]
---

单链表是一种常见的数据结构。对单链表反转的操作有以下几种方式：
- 遍历缓存，再重组链表
- 递归的方式
- 单次遍历直接反转

下图演示了单次遍历直接反转的一步的操作过程，后续循环操作即可完成整个单链表的反转

<!--more-->

[![reverse_ssl.png](https://i.loli.net/2018/03/12/5aa642333f9a9.png)](https://i.loli.net/2018/03/12/5aa642333f9a9.png)


golang代码如下：

```golang
package main

import (
    "fmt"
)

func main() {
    head := initList()
    head = reverseList(head)
    printList(head)
}

type node struct {
    value interface{}
    next  *node
}

func initList() *node {
    var list *node
    var pre *node
    for i := 0; i < 10; i++ {
        tmp := &node{i + 1, nil}
        if i == 0 {
            list = tmp
            pre = tmp
            continue
        }
        pre.next = tmp
        pre = tmp
    }
    return list
}

func printList(head *node) {
    slice := make([]interface{}, 0, 10)
    for cur := head; cur != nil; cur = cur.next {
        slice = append(slice, cur.value)
    }
    fmt.Println(slice)
}

func reverseList(head *node) *node {
    cur := head
    if cur == nil {
        return nil
    }
    next := cur.next
    var pre *node

    for {
        if cur == nil {
            break
        }

        cur.next = pre

        // 整体往后偏移
        pre = cur
        cur = next
        if next != nil {
            next = next.next
        }
    }

    return pre
}
```
