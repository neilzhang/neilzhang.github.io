---
title: 字符串大数相加
author: neil
date: 2022-08-06 12:17:00 +0800
categories: [技术, 算法]
tags: [面试, 算法]
---


## 题目

> 实现两个超大字符串整数加法，字符串整数会超过 long 存储上限，不允许使用相关系统库函数直接完成。注意：整数包含正负数。

## 思路

两个数都有可能是正负数，所以 `a+b` 有4种情况，

1. a，b 都是正数, 则 `a+b = |a|+|b|`
2. a，b 都是负数, 则 `a+b = (-|a|)+(-|b|) = -(|a|+|b|)`
3. a 是正数，b 是负数，则 `a+b = |a|+(-|b|) = |a|-|b|`
4. a 是负数，b 是正数，则 `a+b = (-|a|)+|b| = -(|a|-|b|)`

综上，只要实现 `|a|+|b|` 和 `|a|-|b|` 即可实现所有情况。

 `|a|+|b|` 实现很简单，只要模拟加法过程即可。`|a|-|b|` 实现需要注意下结果的正负，绝对值大的减去绝对值小的，结果为正数；绝对值小的减去绝对值大的，结果为负数。

## 代码

```golang
package main

import "fmt"

func main() {
	fmt.Println(StringAdd("456", "-789"))
}

func StringAdd(a, b string) string {
	// 确认正负数并且将字符串转成数组
	var asign, bsign int = 1, 1
	anum, bnum := reverse([]rune(a)), reverse([]rune(b)) // 倒序下，个位数放到前面
	if a[0] == '-' {
		asign = -1
		anum = anum[:len(anum)-1] // 去掉负号
	}
	if b[0] == '-' {
		bsign = -1
		bnum = bnum[:len(bnum)-1]
	}

	// 数字补齐
	if len(anum) > len(bnum) {
		bnum = arrRightPad(bnum, len(anum)-len(bnum), '0')
	} else if len(anum) < len(bnum) {
		anum = arrRightPad(anum, len(bnum)-len(anum), '0')
	}

	// 加减处理，同号相加符号对齐，异号相减符号随大
	var cSign int = 1
	var cnum []rune
	if asign == bsign {
		cnum = add(anum, bnum)
		cSign = asign
	} else if compare(anum, bnum) >= 0 {
		cnum = minus(anum, bnum)
		cSign = asign
	} else {
		cnum = minus(bnum, anum)
		cSign = bsign
	}

	// 逆序并去除前置0
	cnum = reverse(cnum)
	i := 0
	for i < len(cnum) && cnum[i] == '0' {
		i++
	}
	cnum = cnum[i:]

	// 输出结果
	if cSign == -1 {
		return "-" + string(cnum)
	}
	return string(cnum)
}

func add(anum, bnum []rune) []rune {
	var carry rune
	var cnum []rune
	for i := 0; i < len(anum); i++ {
		val := anum[i] - '0' + bnum[i] - '0' + carry
		cnum = append(cnum, val%10+'0')
		carry = val / 10
	}
	if carry > 0 {
		cnum = append(cnum, '1')
	}
	return cnum
}

func minus(anum, bnum []rune) []rune {
	var carry rune
	var cnum []rune
	for i := 0; i < len(anum); i++ {
		val := (anum[i] - '0') - (bnum[i] - '0') + carry
		if val < 0 {
			cnum = append(cnum, val+10+'0')
			carry = -1
		} else {
			cnum = append(cnum, val+'0')
			carry = 0
		}
	}
	return cnum
}

func compare(anum, bnum []rune) int {
	for i := len(anum) - 1; i >= 0; i-- {
		if anum[i] > bnum[i] {
			return 1
		} else if anum[i] < bnum[i] {
			return -1
		}
	}
	return 0
}

func arrRightPad(arr []rune, cnt int, val rune) []rune {
	for i := 0; i < cnt; i++ {
		arr = append(arr, '0')
	}
	return arr
}

func reverse(arr []rune) []rune {
	for i, cnt := 0, len(arr); i < cnt/2; i++ {
		arr[i], arr[cnt-i-1] = arr[cnt-i-1], arr[i]
	}
	return arr
}

```