---
title: go切片slice的扩容机制
date: 2024-12-26 16:24:01
tags: 
    - golang
description: 那么go语言多个版本依赖，它的扩容机制是什么样的呢？网上关于扩容机制有很多说法，但是都是比较笼统，禁不住面试官深入提问，所以这篇文章带你深入了解slice的扩容，让你明白go语言的源码也不难读。
categories: 
    - golang
---

### go语言Slice的扩容机制



> 我们都知道在go语言中，slice的底层是数组，当我们对slice增加元素时，实际添加的是底层的数组，但是数组的长度是固定的。如果在新增的元素个数超过底层数组的容量，底层数组就会进行扩容，那么go语言多个版本依赖，它的扩容机制是什么样的呢？网上关于扩容机制有很多说法，但是都是比较笼统，禁不住面试官深入提问，所以这篇文章带你深入了解slice的扩容，让你明白go语言的源码也不难读。

目前go语言的切片扩容机制主要经历了两个大的思路改动，我把它分成1.7版本之前和1.8版本及之后，同时也在其他版本中对某个思路进行优化。那我们先来看看1.7版本及之前的扩容机制

#### 1.7版本

​	假如你现在是go语言的开发大佬，团队将slice的扩容的机制交给你设计。那么你会怎么思考呢？

让我们从简单的开始，给一个slice扩容，最简单的就是直接在原来的容量上增加一定的数量，你可能会写出如下代码

```go
func growslice(old int) int {
  return old + 一个常量  
}
```

但是身为大佬的你会想到，不管旧切片的容量大小，每次增加一个固定长度的容量都不合适，比如现在常量比较小，但是旧容量非常大，那你需要调用很多次`growslice`才能满足，每次调用都是要开辟新的空间的，这样性能不就下去了

​	聪明的你想到了直接乘以一个倍数，于是你有了新的思路：

```go
func growslice(old int) int{
  return old * 2 
}
```

那这是不是太简单了，假如old比较小，这样也可以，但是如果old非常大，那么2倍的old岂不是更大，如果内存分配更大地址，但是实际没有那么多的元素填满，既不是浪费了宝贵的内存，这样不用多久就会报 `内存溢出`的问题了

​	那解决上面的问题非常简单，设置个阈值，比如`1024`,只要在旧容量较少的时候扩展`2倍`，在旧容量比较大的时候，扩展比较少的倍数就行啦，比如`1.25倍`,然后再简单粗暴些，要是大于2倍还不够，直接设置成所需的容量，于是你写出了这个代码

```go
func growslice(old slice,cap int) int {   // cap 需要的容量  old 旧切片结构体
  	newcap := old.cap //定义新的容量，初始值为旧容量
		doublecap := newcap + newcap // 定义旧容量的2倍
  	
    if cap > doublecap { // 如果需要的容量大于旧容量2倍
      newcap = cap // 直接设置新容量位需要的容量
    } else {
      if old.cap < 1024 {  //如果旧容量少于1024
        newcap = doublecap  // 新容量是原来的2倍
      } else {
        for 0 < newcap && newcap < cap {  //如果大于1024，就每次增加1.25倍
          newcap += newcap / 4
        }
        // Set newcap to the requested cap when
        // the newcap calculation overflowed.
        if newcap <= 0 {
          newcap = cap
        }
      }
    }
}
```

恭喜你，你设计了golang 1.17版本之前的切片扩容机制！！！

源码路径: src/runtime/slice.go



#### 1.8版本

在1.18版本之后，扩容机制设计思路发生了一些变化，主要是取消了以一些简单粗暴的直接乘以1.25倍的操作，但是整体的设计思想还是变化不大的，我们先来看看源码

```go
	newcap := old.cap
	doublecap := newcap + newcap

	if cap > doublecap {  // 需要的容量大于旧切片的容量
		newcap = cap  //可以看到这一步是没有变化的，如果需要的容量大于旧的容量的2倍，直接扩容到新的容量
	} else {  // 需要的容量小于旧切片的2倍容量
		const threshold = 256 // 这里设置了一个阈值  256
    
   	// 接下来判断旧容量的阈值的关系 
		if old.cap < threshold { // 旧容量小于阈值
			newcap = doublecap  // 直接翻倍
		} else {  // 旧容量大于阈值
			// 这是一个循环，直到新容量大于等于需要的容量才退出	
			for 0 < newcap && newcap < cap {
				// 这里取两个极限值看，当newcap无限接近256时，
        // 等价于 newcap += (newcap + 3 * newcap) /4   也就是 newcap += newcap  取两倍
        // 当newcap远大于256时，threshold就可以忽略不看
        // 等价于 newcap += newcap / 4 也就是 newcap = 1.25 * newcap
				newcap += (newcap + 3*threshold) / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}
```

从上面的代码中可以看出，这次改动主要是改动了一个阈值和翻倍时扩容曲线的线性。不再是直接的直接线性，而是无限接近的取值。

而在后面的版本中，go团队将扩容单独写成了一个函数，并优化了代码，比如1.23中

```go
func nextslicecap(newLen, oldCap int) int { //newLen 就是新的容量，oldCap就是旧的容量
	newcap := oldCap
	doublecap := newcap + newcap
	if newLen > doublecap {  
		return newLen  
	}

	const threshold = 256  
	if oldCap < threshold { 
		return doublecap   // 如果旧的容量小于阈值，直接返回两倍的容量
	}
	for {                // 这里是一个循环
		newcap += (newcap + 3*threshold) >> 2 
    
		if uint(newcap) >= uint(newLen) {
			break  // 一直到新的容量大于等于需要的容量才会退出
		}
	}

	if newcap <= 0 {
		return newLen
	}
	return newcap
}
```

以上就是go语言的扩容机制。

#### 总结

- **1.7版本**：如果当前容量小于**1024**，则判断所需容量是否大于原来容量2倍，如果大于，当前容量加上所需容量；否则当前容量乘2。

- - 如果当前容量大于1024，则每次按照1.25倍速度递增容量，也就是每次加上cap/4。

- **1.8版本**：Go1.18不再以1024为临界点，而是设定了一个值为**256**的`threshold`，以256为临界点；超过256，不再是每次扩容1/4，而是每次增加（旧容量+3*256）/4；

- - 当新切片需要的容量cap大于两倍扩容的容量，则直接按照新切片需要的容量扩容；

  - 当原 slice 容量 < threshold 的时候，新 slice 容量变成原来的 2 倍；

  - 当原 slice 容量 > threshold，进入一个循环，每次容量增加（旧容量+3*threshold）/4。

    

**等等，你以为你结束了？不，前面只是介绍了新切片的容量，也就是新切片的元素个数，但是在代码中容量是体现在内存上的，所以如何为不同类型的切片设置合适的内存大小呢？**

众所周知

> 内存大小 = 容量个数 * 元素类型大小

那难道直接申请乘积大小的内存就行了？不不不，没那么简单。

在许多编程语言中，申请内存不是直接和操作系统沟通，而是和语言自身实现的内存管理模块挂钩，由程序想内存管理模块申请，管理模块一般会向操作系统申请一批内存，然后分成不同大小的常用的内存并管理，然后按照需求选择合适（`足够大且不浪费`）的内存分配给程序

这里就需要匹配到合适的内存规格

```go
switch {
	case et.Size_ == 1:  // 如果元素类型大小为 1 字节
		// ...
		capmem = roundupsize(uintptr(newcap), noscan)  // 返回内存大小
		newcap = int(capmem)
	case et.Size_ == goarch.PtrSize:  // 如果元素类型是默认指针大小（32位系统为4   64为系统为8）
		 // ...
		capmem = roundupsize(uintptr(newcap)*goarch.PtrSize, noscan) // 返回内存大小
		// ...
		newcap = int(capmem / goarch.PtrSize) //新的容量为申请的内存大小 / 默认指针大小
	case isPowerOfTwo(et.Size_):  // 如果是类型大小是否是2的幂
		// ... 位运算获取cap容量 
		newcap = int(capmem >> shift)
		capmem = uintptr(newcap) << shift
	default: // 其他情况 
		// ...
		newcap = int(capmem / et.Size_)  // 直接相除
		capmem = uintptr(newcap) * et.Size_
	}
```



在获取了相应的内存大小之后,调用`mallocgc`方法申请对应的内存

```go
 mallocgc(capmem, et, true)
// capmem 内存大小    et 元素类型
```



最后调用`memmove(p, oldPtr, lenmem)`方法将旧的切片移到新的切片里

```go
memmove(p, oldPtr, lenmem)
// p 移动目的地  oldPtr 从哪移动  lenmen 移动大小
```



#### 总结

现在才算完成了go语言的切片扩容和内存的申请。我们可以看到，上面也有一些关于go语言内存管理的细节没有提及，因为这里面的内容比较多，就暂时不在这篇文章描述了。如果读者感兴趣，可以自行了解go的内存管理

