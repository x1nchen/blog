---
title: "Golang 服务内存泄漏问题分析一例"
author: ["ccx"]
keywords: ["golang", "madvice"]
lastmod: 2022-02-22T11:17:35+08:00
draft: false
---

<div class="ox-hugo-toc toc">

<div class="heading">Table of Contents</div>

- [调查过程记录和分析](#调查过程记录和分析)
- [20200824 更新](#20200824-更新)
- [20210219 更新](#20210219-更新)
- [ref](#ref)

</div>
<!--endtoc-->



## 调查过程记录和分析 {#调查过程记录和分析}

记录下这个问题是因为这不同于传统典型的 Go 服务内存泄漏。

2020年8月15日前后，钉钉监控告警群不定时出现服务机器内存占用超 80% 告警，一开始以为是
Prometheus exporter 堆积造成的，准备下掉 Prometheus exporter 的集成代码，后来觉
得这个堆积速度太快了点，不到一天就吃掉 6GB，十分夸张，所以有空的时候用 gops 快照了
一下，内存情况如下图

```bash
(pprof) /app/bin # gops memstats 1
alloc: 54.66MB (57317968 bytes)
total-alloc: 186.21GB (199938346560 bytes)
sys: 6.72GB (7215977904 bytes)
lookups: 0
mallocs: 2906600570
frees: 2906521966
heap-alloc: 54.66MB (57317968 bytes)
heap-sys: 6.44GB (6909984768 bytes)
heap-idle: 6.38GB (6846406656 bytes)
heap-in-use: 60.63MB (63578112 bytes)
heap-released: 6.33GB (6794371072 bytes)
heap-objects: 78604
stack-in-use: 2.12MB (2228224 bytes)
stack-sys: 2.12MB (2228224 bytes)
stack-mspan-inuse: 266.02KB (272408 bytes)
stack-mspan-sys: 31.88MB (33423360 bytes)
stack-mcache-inuse: 6.78KB (6944 bytes)
stack-mcache-sys: 16.00KB (16384 bytes)
other-sys: 4.58MB (4802459 bytes)
gc-sys: 251.55MB (263773240 bytes)
next-gc: when heap-alloc >= 100.19MB (105060944 bytes)
last-gc: 2020-08-14 20:54:01.063567774 +0800 CST
gc-pause-total: 982.706218ms
gc-pause: 17439
num-gc: 3484
enable-gc: true
debug-gc: false
```

分析从上面的指标可以得出以下结论

1.  当前正在使用的堆内存大致是 60 多M
2.  HeapReleased = 6.33 GB 返还给操作系统的物理内存的字节数是 6.33GB (
    HeapReleased 统计了从idle span中返还给操作系统，没有被重新获取的内存大小)

那问题就抽象为：\*为什么 HeapReleased 上升，RSS 没有下降?\*

搜索了一下有人踩过这个坑了，参考：<https://zhuanlan.zhihu.com/p/114340283>

> 这是因为 Go 底层用 mmap 申请的内存，会用 madvise 释放内存。具体见 go/src/runtime/mem_linux.go 的代码。
>
> madvise 将某段内存标记为不再使用时，有两种方式 MADV_DONTNEED 和 MADV_FREE（通过标志参数传入）：
>
> -   MADV_DONTNEED标记过的内存如果再次使用，会触发缺页中断
> -   MADV_FREE标记过的内存，内核会等到内存紧张时才会释放。在释放之前，这块内存依然
>     可以复用。这个特性从linux 4.5版本内核开始支持
>
> 显然，MADV_FREE是一种用空间换时间的优化。
>
> -   在Go 1.12之前，linux 平台下 Go runtime 中的sysUnsed使用madvise(MADV_DONTNEED)
> -   在Go 1.12之后，在MADV_FREE可用时会优先使用MADV_FREE
>
> 具体见 <https://github.com/golang/go/issues/23687>
>
> Go 1.12之后，提供了一种方式强制回退使用MADV_DONTNEED的方式，在执行程序前添加
> GODEBUG=madvdontneed=1。具体见 <https://github.com/golang/go/issues/28466>

另外还有一个疑点：现在知道问题出现的条件是 Go &gt; 1.12 &amp;&amp; Linux Kernel &gt; 4.5，但是
为什么之前阿里云没有出现，系统8月10号整体迁移到 aws 出现了？

调查发现

阿里云机器内核 4.4.0-105-generic，而 aws 机器内核 4.15.0-1054-aws，刚好符合预期，可以解释。


## 20200824 更新 {#20200824-更新}

使用 GODEBUG=madvdontneed=1 强制回退使用 MADV_DONTNEED，没有再出现内存泄漏问题。内存稳定在 200MB 左右


## 20210219 更新 {#20210219-更新}

Go 1.16 即将发布，默认使用 MADV_DONTNEED 方式，该问题在 Go 版本升级后自动解决。<https://golang.org/doc/go1.16#runtime>

不过看上去处理有些粗暴啊。<https://go-review.googlesource.com/c/go/+/267100/3/src/runtime/runtime1.go>

```go
func parsedebugvars() {
	// defaults
	debug.cgocheck = 1
	debug.invalidptr = 1
	if GOOS == "linux" {
		// On Linux, MADV_FREE is faster than MADV_DONTNEED,
		// but doesn't affect many of the statistics that
		// MADV_DONTNEED does until the memory is actually
		// reclaimed. This generally leads to poor user
		// experience, like confusing stats in top and other
		// monitoring tools; and bad integration with
		// management systems that respond to memory usage.
		// Hence, default to MADV_DONTNEED.
		debug.madvdontneed = 1
	}

	for p := gogetenv("GODEBUG"); p != ""; {
		field := ""
		i := bytealg.IndexByteString(p, ',')
		if i < 0 {
			field, p = p, ""
		} else {
			field, p = p[:i], p[i+1:]
		}
		i = bytealg.IndexByteString(field, '=')
```


## ref {#ref}

-   踩坑记：go服务内存暴涨_felix021 - SegmentFault 思否: <https://segmentfault.com/a/1190000022472459>
