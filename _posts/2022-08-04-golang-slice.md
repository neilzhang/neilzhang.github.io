---
title: Golang Slice 总结
author: neil
date: 2022-08-04 15:19:00 +0800
categories: [技术, Golang]
tags: [面试, Golang]
---


## 什么是 Slice

> 切片是对数组的抽象，提供动态数组的能力。切片的长度是不固定的，随着元素的增加而动态变化。

## 数组与 Slice 的区别

### 数组
- 值类型
- 长度固定
- 编译期间检查下标访问越界行为
- 可以作为 map 的 key


### Slice
- 引用类型
- 长度可变
- 运行期间检查下标访问越界行为
- 不可以作为 map 的 key 

## 适用场景

- 长度不固定
- 参数传递

## 用法示例

```golang
var nums []int               // 声明切片
var arr = [3]int{4, 5, 6}    // 初始化数组
nums = append(nums, 1, 2, 3) // 切片追加元素
l := len(nums)               // 获取元素个数
c := cap(nums)               // 获取切片容量
bigger := make([]int, 6)     // 创建指定长度和容量的切片
copy(bigger, nums[:])        // 将 nums 中所有元素拷贝到 bigger 切片中
copy(bigger[3:], arr[:])     // 将 arr 数组转变成切片并拷贝到 bigger 切片从下标 3 开始的地方
fmt.Println(bigger)          // 打印出 [1 2 3 4 5 6]
```

## 避坑指南【Bad Case】

### append 数据丢失

```golang
func main() {
	slice := make([]int, 0)
	slice = append(slice, 1, 2, 3)
	fmt.Println(slice) // 输出: [1 2 3]
	updateSlice(slice, 1, -2)
	fmt.Println(slice) // 输出: [1 -2 3]
	appendSlice(slice, 2)
	fmt.Println(slice) // 输出: [1 -2 3]
}

func updateSlice(slice []int, idx int, val int) {
	slice[idx] = val
	fmt.Println(slice) // 输出: [1 -2 3]
}

func appendSlice(slice []int, val int) {
	slice = append(slice, val) // 扩容
	fmt.Println(slice) // 输出: [1 -2 3 2]
}
```

### 并发 append 数据丢失

```golang
func main() {
	slice := make([]int, 0)
	var wg sync.WaitGroup
	var appendSlice = func(val int) {
		slice = append(slice, val)
		wg.Done()
	}
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go appendSlice(i)
	}
	wg.Wait()
	fmt.Printf("len = %d, cap = %d", len(slice), cap(slice)) // len <= 10, cap <= 16
}
```

## 数据结构

```golang
type slice struct {
	array unsafe.Pointer // 指针指向 heap 区一块连续内存地址
	len   int  // 长度，实际元素个数
	cap   int // 容量，最多元素个数
}
```


![image.png](https://raw.githubusercontent.com/neilzhang/blog-images/main/posts/20220804/20220804152930.png)


## 扩容机制

当 Slice 容量不足以 append 新的元素时，会触发扩容。扩容时会申请一块新的内存空间，然后将旧 Slice 数据拷贝到新 Slice，并 append 新元素。为避免频繁扩容，扩容规则如下：

1. 当新容量大于 2 倍旧容量时，按实际容量扩容
2. 否则当旧容量小于 1024 时，按 2 倍容量扩容
3. 否则按旧容量的 1.25 倍容量扩容

另外，为考虑内存对齐，避免内存浪费，最终的容量可能会基于上面规则计算的值进行微调，具体细节可以阅读[源码 ](https://github.com/golang/go/blob/master/src/runtime/slice.go#L178)