---
title: Golang 避坑指南之 time.After
author: neil
date: 2022-08-06 11:53:00 +0800
categories: [技术, Golang]
tags: [踩坑, Golang]
---

如何简单实现请求调用的超时机制呢？有了 `time.After` 当然就很简单啦，代码如下：

```go
func AsyncCallWithTimeout1() {
	ctx, cancel := context.WithCancel(context.Background())
	go func(ctx context.Context) {
		defer cancel()
		// 模拟请求调用
		time.Sleep(200 * time.Millisecond)
	}(ctx)

	select {
	case <-ctx.Done():
		fmt.Println("call successfully!!!")
		return
	case <-time.After(time.Duration(3 * time.Second)):
		fmt.Println("timeout!!!")
		return
	}
}
```

测试该方法，输出如下：

```sh
hello % go run ./hello.go 
call successfully!!!
```

和我们期待的一样，成功了，没有超时。
让我们再测试下超时场景，修改模拟请求调用的超时时间为 4 秒，然后再测试该方法，输出如下：

```sh
hello % go run ./hello.go
timeout!!!
```

果然发生了超时，这就是我们想要的功能，如此简单。
不过优秀的程序猿都会仔细阅读 API 文档，让我们一起来看下：

```go
// After waits for the duration to elapse and then sends the current time
// on the returned channel.
// It is equivalent to NewTimer(d).C.
// The underlying Timer is not recovered by the garbage collector
// until the timer fires. If efficiency is a concern, use NewTimer
// instead and call Timer.Stop if the timer is no longer needed.
func After(d Duration) <-chan Time {
	return NewTimer(d).C
}
```

凭借多年 [google 翻译](https://translate.google.com/?hl=zh-CN) 使用经验，这里大概意思是：
>`Timer` 不会被 GC 回收直到它被触发，如果需要考虑效率的话，`Timer` 不再被需要时，需要主动调用 `Timer.Stop`。

卧槽，还好这次没有草率。如果有大量请求调用的场景，使用 `time.After` 会导致有大量的 `Timer` 对象被延迟释放，造成大量内存浪费。 原来正确的使用姿势应该是这个样子的：

```go
func AsyncCallWithTimeout2() {
	ctx, cancel := context.WithCancel(context.Background())

	go func() {
		defer cancel()
		// 模拟请求调用
		time.Sleep(200 * time.Millisecond)
	}()

	timer := time.NewTimer(3 * time.Second)
	defer timer.Stop()
	select {
	case <-ctx.Done():
		// fmt.Println("call successfully!!!")
		return
	case <-timer.C:
		// fmt.Println("timeout!!!")
		return
	}
}
```

本着优秀程序猿是不会有 ~~bug~~ 的精神，让我们来验证下这个姿势还有没有问题。简单的测试代码如下：

```go
func main() {
	var wg sync.WaitGroup
	for i := 0; i < 1000000; i++ {
		wg.Add(1)
		go func() {
			AsyncCallWithTimeout1()
			// AsyncCallWithTimeout2()
			wg.Done()
		}()
	}
	wg.Wait()
	fmt.Println("all done!!!")

	for i := 0; i < 6; i++ {
		runtime.GC()
		time.Sleep(1 * time.Second)
	}
}
```

运行程序，测试关键输出如下:

```sh
hello % GODEBUG=gctrace=1 go run ./hello.go
# command-line-arguments
gc 1 @0.001s 12%: 0.006+2.2+0.017 ms clock, 0.075+1.5/4.1/0.75+0.21 ms cpu, 5->6->6 MB, 6 MB goal, 12 P
gc 2 @0.013s 5%: 0.004+2.2+0.017 ms clock, 0.053+0.85/4.1/2.8+0.21 ms cpu, 11->11->10 MB, 12 MB goal, 12 P
gc 3 @0.043s 3%: 0.004+3.6+0.005 ms clock, 0.058+0.15/9.7/9.7+0.065 ms cpu, 20->20->18 MB, 21 MB goal, 12 P
gc 1 @0.010s 6%: 0.10+23+0.33 ms clock, 1.2+14/5.9/0+3.9 ms cpu, 4->7->6 MB, 5 MB goal, 12 P
gc 2 @0.041s 15%: 0.14+7.0+0.14 ms clock, 1.6+42/20/0+1.7 ms cpu, 10->11->10 MB, 12 MB goal, 12 P
gc 3 @0.059s 20%: 0.28+23+0.20 ms clock, 3.4+77/28/0+2.5 ms cpu, 17->19->17 MB, 21 MB goal, 12 P
gc 4 @0.108s 22%: 0.053+32+0.23 ms clock, 0.64+121/49/0+2.8 ms cpu, 29->32->30 MB, 35 MB goal, 12 P
gc 5 @0.183s 25%: 0.076+27+0.20 ms clock, 0.91+196/77/0+2.4 ms cpu, 49->52->50 MB, 60 MB goal, 12 P
gc 6 @0.293s 28%: 0.11+47+0.093 ms clock, 1.3+376/140/0.086+1.1 ms cpu, 85->86->79 MB, 100 MB goal, 12 P
gc 7 @0.463s 30%: 0.071+72+0.13 ms clock, 0.85+584/214/0+1.6 ms cpu, 142->144->128 MB, 158 MB goal, 12 P
gc 8 @0.745s 29%: 0.36+122+1.2 ms clock, 4.3+767/337/0+14 ms cpu, 237->237->202 MB, 256 MB goal, 12 P
gc 9 @1.194s 29%: 0.13+189+0.99 ms clock, 1.6+1234/513/0+11 ms cpu, 388->391->312 MB, 405 MB goal, 12 P
all done!!!
gc 10 @2.050s 20%: 30+138+0.006 ms clock, 362+0/366/1011+0.081 ms cpu, 565->565->364 MB, 624 MB goal, 12 P (forced)
gc 11 @3.271s 14%: 0.082+104+0.041 ms clock, 0.98+0/295/853+0.49 ms cpu, 364->364->353 MB, 729 MB goal, 12 P (forced)
gc 12 @4.411s 11%: 0.024+78+0.29 ms clock, 0.29+0/206/499+3.5 ms cpu, 353->353->250 MB, 706 MB goal, 12 P (forced)
gc 13 @5.534s 9%: 0.035+31+0.006 ms clock, 0.42+0/61/308+0.082 ms cpu, 250->250->166 MB, 500 MB goal, 12 P (forced)
gc 14 @6.596s 7%: 0.026+30+0.005 ms clock, 0.31+0/58/292+0.062 ms cpu, 166->166->166 MB, 332 MB goal, 12 P (forced)
gc 15 @7.641s 6%: 0.027+31+0.006 ms clock, 0.33+0/79/283+0.076 ms cpu, 166->166->166 MB, 332 MB goal, 12 P (forced)
```

切换到 `AsyncCallWithTimeout2` ，运行程序，测试关键输出如下：

```sh
hello % GODEBUG=gctrace=1 go run ./hello.go
# command-line-arguments
gc 1 @0.002s 8%: 0.007+2.2+0.021 ms clock, 0.091+0.19/4.9/0.39+0.25 ms cpu, 4->7->6 MB, 5 MB goal, 12 P
gc 2 @0.014s 5%: 0.005+2.9+0.019 ms clock, 0.065+0/5.8/0.39+0.23 ms cpu, 10->11->10 MB, 12 MB goal, 12 P
gc 3 @0.040s 4%: 0.003+3.8+0.020 ms clock, 0.044+0.19/10/6.5+0.24 ms cpu, 18->19->17 MB, 20 MB goal, 12 P
gc 4 @0.082s 3%: 0.004+6.3+0.019 ms clock, 0.048+0/17/4.3+0.23 ms cpu, 31->33->28 MB, 34 MB goal, 12 P
gc 1 @0.008s 2%: 0.054+141+0.72 ms clock, 0.64+31/5.5/0+8.7 ms cpu, 4->25->24 MB, 5 MB goal, 12 P
gc 2 @0.179s 13%: 0.17+28+0.10 ms clock, 2.0+214/82/0+1.2 ms cpu, 38->41->37 MB, 48 MB goal, 12 P
gc 3 @0.263s 18%: 0.19+66+0.15 ms clock, 2.3+286/111/0+1.9 ms cpu, 62->68->64 MB, 75 MB goal, 12 P
gc 4 @0.412s 24%: 0.080+57+0.10 ms clock, 0.96+441/167/0+1.2 ms cpu, 109->110->100 MB, 128 MB goal, 12 P
gc 5 @0.603s 27%: 0.16+92+0.29 ms clock, 1.9+713/261/0+3.5 ms cpu, 180->184->154 MB, 200 MB goal, 12 P
gc 6 @0.957s 27%: 1.0+127+1.9 ms clock, 12+799/359/0+23 ms cpu, 291->292->203 MB, 308 MB goal, 12 P
gc 7 @1.417s 27%: 0.59+189+2.2 ms clock, 7.0+1176/511/0+26 ms cpu, 386->389->255 MB, 407 MB goal, 12 P
all done!!!
gc 8 @2.171s 20%: 29+92+0.005 ms clock, 351+0/146/314+0.070 ms cpu, 408->408->158 MB, 510 MB goal, 12 P (forced)
gc 9 @3.344s 14%: 0.025+34+0.006 ms clock, 0.30+0/69/297+0.078 ms cpu, 158->158->158 MB, 317 MB goal, 12 P (forced)
gc 10 @4.394s 11%: 0.025+29+0.005 ms clock, 0.30+0/59/288+0.061 ms cpu, 158->158->158 MB, 317 MB goal, 12 P (forced)
gc 11 @5.438s 9%: 0.024+32+0.004 ms clock, 0.29+0/72/285+0.058 ms cpu, 158->158->158 MB, 317 MB goal, 12 P (forced)
gc 12 @6.486s 7%: 0.024+33+0.008 ms clock, 0.29+0/67/310+0.10 ms cpu, 158->158->158 MB, 317 MB goal, 12 P (forced)
gc 13 @7.535s 6%: 0.020+30+0.007 ms clock, 0.24+0/60/293+0.089 ms cpu, 158->158->158 MB, 317 MB goal, 12 P (forced)
```

对比两个的 GC 日志，可以发现 `AsyncCallWithTimeout2` 比 `AsyncCallWithTimeout1` 的内存回收会及时很多，这也符合 time.After 文档给的解释。

所以结论是：
> `time.After` 创建的 `Timer` 对象需要等到 `Timer` 被触发时才能被 GC 回收释放，使用不当会造成内存浪费

开发环境：
> go 1.14