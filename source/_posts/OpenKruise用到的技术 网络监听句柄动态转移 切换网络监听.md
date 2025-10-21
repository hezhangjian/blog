---
title: OpenKruise用到的技术 网络监听句柄动态转移 切换网络监听
link:
date: 2021-08-28 20:04:58
tags:
---

今天看到了https://mp.weixin.qq.com/s/tTipcU8MKxTpFurWNEUIww 中热迁移网络句柄的功能，忍不住自己在机器上实验了一下，确实可以实现网络句柄的迁移。实验代码如下

## old_server

```go
package main

import (
	"net"
	"syscall"
)

func main() {
	tcpAddr := net.TCPAddr{Port: 8001}
	tcpLn, err := net.ListenTCP("tcp4", &tcpAddr)
	if err != nil {
		panic(err)
	}
	f, _ := tcpLn.File()
	fdNum := f.Fd()
	data := syscall.UnixRights(int(fdNum))
	// 与新版本sidecar通过Unix Domain Socket建立链接
	raddr, _ := net.ResolveUnixAddr("unix", "/dev/shm/migrate.sock")
	uds, _ := net.DialUnix("unix", nil, raddr)
	// 通过UDS，发送ListenFD到新版本sidecar容器
	_, _, _ = uds.WriteMsgUnix(nil, data, nil)
	// 停止接收新的request，并且开始排水阶段
	tcpLn.Close()
}
```

## new_server

```go
package main

import (
	"fmt"
	"net"
	"net/http"
	"os"
	"syscall"
	"time"
)

func main() {
	// 新版本sidecar 接收ListenFD，并且开始对外服务
	// 监听UDS
	addr, _ := net.ResolveUnixAddr("unix", "/dev/shm/migrate.sock")
	unixLn, _ := net.ListenUnix("unix", addr)
	conn, _ := unixLn.AcceptUnix()
	buf := make([]byte, 32)
	oob := make([]byte, 32)
	time.Sleep(5 * time.Second)
	// 接收 ListenFD
	_, oobn, _, _, _ := conn.ReadMsgUnix(buf, oob)
	scms, _ := syscall.ParseSocketControlMessage(oob[:oobn])
	if len(scms) > 0 {
		// 解析FD，并转化为 *net.TCPListener
		fds, _ := syscall.ParseUnixRights(&scms[0])
		f := os.NewFile(uintptr(fds[0]), "")
		ln, _ := net.FileListener(f)
		tcpLn, _ := ln.(*net.TCPListener)
		http.Serve(tcpLn, &MyHttp{})
	}
}

type MyHttp struct {
}

func (*MyHttp) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	fmt.Fprintf(w, "hello world")
}
```

## 实验结果

- 先启动new_server
- 再启动old_server
- 然后访问`localhost:8001`，返回`hello world`，这是`new_server`返回的结果

## 代码地址

https://github.com/hezhangjian/go_demo/tree/main/demo_base/fd_migrate
