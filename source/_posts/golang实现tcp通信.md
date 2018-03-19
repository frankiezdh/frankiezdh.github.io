---
title: golang实现tcp通信
date: 2018-03-12 18:01:18
tags: golang
---

golang net包中有包可以很方便的实现tcp的服务端和客户端。下面代码使用了context和waitgroup实现了，tcp的服务端和客户端互发互收。


<!--more-->

* 服务端
```golang
package main

import (
    "net"
    "log"
    "time"
)

func main() {
    run()
}

func run() {
    ln, err := net.Listen("tcp4", ":18888")
    if err != nil {
        log.Println("监听失败")
        return
    }
    defer ln.Close()
    log.Println("开始监听")

    for ; ; {
        conn, err := ln.Accept()
        if err != nil {
            log.Println("Accept失败")
            continue
        }
        log.Println("Accept成功")

        channel := make(chan []byte, 128)
        go read(conn, channel)
        go write(conn, channel)
    }
}

func read(conn net.Conn, channel chan []byte) {
    defer closeConn(conn)
    for ; ; {
        buf := make([]byte, 64)
        _, err := conn.Read(buf)
        if err != nil {
            log.Println("读取失败")
            return
        }
        channel <- buf
        time.Sleep(time.Duration(1) * time.Second)
    }
}

func write(conn net.Conn, channel chan []byte) {
    defer closeConn(conn)
    for ; ; {
        buf := <-channel
        _, err := conn.Write(buf)
        if err != nil {
            log.Println("写入失败")
            return
        }
        time.Sleep(time.Duration(1) * time.Second)
    }
}

func closeConn(conn net.Conn)  {
    conn.Close()
}

```


* 客户端
```golang
package main

import (
    "net"
    "log"
    "sync"
    "time"
    "fmt"
    "bufio"
)

func main() {
    clientRun()
}

func clientRun() {
    conn, err := net.Dial("tcp4", ":18888")
    if err != nil {
        log.Print("连接服务端失败")
        return
    }
    defer conn.Close()

    var group sync.WaitGroup
    group.Add(1)
    go clientRead(conn, &group)
    group.Add(1)
    go clientWrite(conn, &group)
    group.Wait()
}

func clientRead(conn net.Conn, group *sync.WaitGroup) {
    defer group.Done()

    for ; ; {
        reader := bufio.NewReader(conn)
        data, err := reader.ReadString(0)
        if err != nil {
            log.Println("客户端读取失败")
            return
        }

        log.Println("客户端收到消息：", data)
        time.Sleep(time.Duration(1) * time.Second)
    }
}

func clientWrite(conn net.Conn, group *sync.WaitGroup) {
    defer group.Done()

    for ; ; {
        data := "你好世界"
        _, err := fmt.Fprint(conn, data)
        if err != nil {
            log.Println("客户端写入失败")
            return
        }

        log.Println("客户端写入消息：", data)
        time.Sleep(time.Duration(1) * time.Second)
    }
}

```

