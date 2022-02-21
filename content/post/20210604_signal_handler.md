---
title: "集成 gops 的 go-micro 服务后 AfterStop 自定义函数不会执行的问题分析"
author: ["ccx"]
keywords: ["RCA"]
lastmod: 2022-02-22T00:46:55+08:00
draft: false
---

<div class="ox-hugo-toc toc">

<div class="heading">Table of Contents</div>

- [前情提要](#前情提要)
- [问题描述](#问题描述)
- [分析原因](#分析原因)
    - [容器停止这个操作到底执行的是什么？](#容器停止这个操作到底执行的是什么)
    - [go-micro 如何处理 linux signal？](#go-micro-如何处理-linux-signal)
    - [经过二分法代码测试检查，发现进程 gops 后的问题](#经过二分法代码测试检查-发现进程-gops-后的问题)
- [解决方案](#解决方案)
- [经验总结](#经验总结)

</div>
<!--endtoc-->

遇到一个很有意思的问题，在此记录一下。

<span class="underline">TLDR: golang 服务通过 signal.Notify 注册 signal handler 的行为有且只有一次，否则会出现不可预知的行为。</span>


## 前情提要 {#前情提要}

公司开发微服务用的 go-micro 框架，这个框架提供了一个及其有价值的特性：micro.AfterStop 函数允许服务退出之后执行一段自定义的逻辑，通常用于清理资源，等待 goroutine 退出等。

另一方面，我们在生产环境也对每个服务集成了 gops ，这个工具极大了提升了服务的可观察性。


## 问题描述 {#问题描述}

某天，某位细心的同事告知 micro.AfterStop 自定义函数在容器重启时，并没有执行。现场复现确认问题存在。

下面是一个 minimal reproducible example

```go
package main

import (
	"fmt"
	"log"

	"github.com/micro/go-micro/v2"
	grpcServer "github.com/micro/go-micro/v2/server/grpc"
	"github.com/google/gops/agent"
)

func main() {
	if err := agent.Listen(agent.Options{
		Addr:            "0.0.0.0:0",
		ShutdownCleanup: true}); err != nil {
		log.Fatal(err)
	}
	defer agent.Close()

	srv.Init(
		micro.Name("service.name"),
		micro.Server(grpcServer.NewServer()),
		micro.AfterStop(func() error {
			fmt.Println("after stop")
			return nil
		}),
	)

	if err := srv.Run(); err != nil {
		fmt.Println("server run err", err)
	}
}
```


## 分析原因 {#分析原因}


### 容器停止这个操作到底执行的是什么？ {#容器停止这个操作到底执行的是什么}

ref <https://docs.docker.com/engine/reference/commandline/stop/>

> The main process inside the container will receive SIGTERM, and after a grace period, SIGKILL. The first signal can be changed with the STOPSIGNAL instruction in the container’s Dockerfile, or the --stop-signal option to docker run.

简而言之，默认情况下会给容器对应的进程发送一个 SIGTERM 信号，然后在一段时间后（默认是10s）发送一个 SIGKILL 信号


### go-micro 如何处理 linux signal？ {#go-micro-如何处理-linux-signal}

go-micro 在 service.Run 方法注册了一个 signal handler，其中就包括了对 SIGTERM 的处理

```go
func (s *service) Run() error {
	......

	if err := s.Start(); err != nil {
		return err
	}

	ch := make(chan os.Signal, 1)
	if s.opts.Signal {
		signal.Notify(ch, syscall.SIGTERM, syscall.SIGINT,
			syscall.SIGQUIT, syscall.SIGKILL)
	}

	select {
	// wait on kill signal
	case sig := <-ch:
		// wait on context cancel
	case <-s.opts.Context.Done():
	}

	return s.Stop()
}
```

到目前为止，没有什么异常，观察到的现象是

1.  Stop() 每次都未执行完，但每次执行结束的地方都不同。
2.  每次退出之后用 echo $? 检查进程退出的 code 也不相同


### 经过二分法代码测试检查，发现进程 gops 后的问题 {#经过二分法代码测试检查-发现进程-gops-后的问题}

gops 在 gracefulShutdown 方法也注册了一个 signal handler

```go
func gracefulShutdown() {
	c := make(chan os.Signal, 1)
	gosignal.Notify(c, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)
	go func() {
		// cleanup the socket on shutdown.
		sig := <-c
		Close()
		ret := 1
		if sig == syscall.SIGTERM {
			ret = 0
		}
		os.Exit(ret)
	}()
}
```

显而易见，os.Exit(ret) 会比 go-micro 更快执行完，导致整个进程退出


## 解决方案 {#解决方案}

知道 root cause 之后就好解决了，移除 gops 的 signal handler 就好了。

恰好 gops 为此提供了一个 option 选项，那我们 disable 掉。注意退出 main goroutine 前，主动调用 agent.Close()

```go
if opts.ShutdownCleanup {
		gracefulShutdown()
}
```

看起来之前就有人提过类似的 issue

-   <https://github.com/google/gops/issues/24>
-   <https://github.com/google/gops/pull/19>


## 经验总结 {#经验总结}

1.  同一个进程不要注册复数个 signal handler，这可能会导致不可预知的行为；debug 类似现象的问题时，注意检查第三方库和集成的功能 (监控/pprof/metric-report 等) 是否存在这种情况
2.  对于一个 sdk lib or integration lib 来说，尽量避免自己注册 signal handler，特别是不要随意调用 os.Exit() 自行终止进程
3.  如果 2 不可避免，那么提供一个选项让调用者可以从外部控制这个行为，否则一定会被喷得怀疑人生。
