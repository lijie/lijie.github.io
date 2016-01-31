---
layout: post
title:  "Go Internals"
date:   2015-05-22 15:06:00 +0800
categories: golang
---

### Container

#### Slice

1. slice在内存中是连续的

2. append在slice的cap不够的情况下, 会分配新内存并copy老的slice到新内存
具体的实现在runtime/slice.go的growslice(). 如果new_cap >= old_cap * 2 则之间增长到new_cap
else 如果old_cap<1024, 则反复double直到超过new_cap
else 反复增加old_cap/4直到超过new_cap

3. a := b[n:m] 实际上生成了一个新的slice 不过data是指向老的slice data
如果又执行a = append(a, ...) 则a可能指向老的, 也可能因为slice grow指向新的data

#### map

### Lock

#### Mutex


	// A Mutex is a mutual exclusion lock.
	// Mutexes can be created as part of other structures;
	// the zero value for a Mutex is an unlocked mutex.
	type Mutex struct {

	// 表示当前mutex的状态
	state int32

	// 用于 runtime.AcquireSem
	sema  uint32
	}

	const (
	// 表示当前 mutex已经被lock
	mutexLocked = 1 << iota // mutex is locked
	// 表示当前 mutex上有g被唤醒
	mutexWoken
	// 当前 mutex上有g在等待
	mutexWaiterShift = iota
	)

##### Mutex.Unlock()

1. 首先清除mutexLock状态
2. 如果当前没有g在mutex上等待, 就结束
3. 如果有正在spin的g立即获取到了lock, 也立即结束
4. 调用 runtime.AcqurieRelease 来唤醒一个正在睡眠的g


##### Mutex.Lock()

1. 首先CompareAandSwap的atomic操作设置Lock状态, 成功就立即结束, 获得锁
2. 如果已经被锁, 则需要等待, 等待的逻辑比较复杂
3. 如果当前状态可以spin(spin需要在多核, GOMAXPROC>1, 当前运行队列为空等情况下才执行)
则执行spin. spin的过程中, 如果当前有其它g也在wait此locker, 优先唤醒spin的g.
4. spin完毕并且未获得锁, 则调用runtime.AcquireSem睡眠

##### Mutex总结

1. go的Mutex没有使用os相关的mutex实现, 不会直接syscall, 所以它的开销理论上要比pthread_mutex要小.
2. 在条件允许的情况下, 支持spin.
3. 唤醒时不是按顺序, 而是优先唤醒spin中的g
4. mutex在block以及唤醒block的g时会调用runtime.sema, sema有futex锁.

#### runtime.sema

sema的实现使用到了runtime.mutex

sema的实现, 就是一个count计数, 代码中使用的uint32.
acquire时, 如果count=0, 等待, 将当前g enqueue到一个semRoot的tail, 否则-1, acquire成功.
release时, 则count+1, 运行等待queue里面的第一个g

semacquire

1. 如果count>0, 则-1, 直接返回.
2. 根据count的addr hash到semtable中, 找到自己semaRoot, 将当前g放到root的tail
3. block, 等待唤醒.
4. 醒来后重试第一步

semrelease

1. 将count+1
2. 根据count addr找到自己的semaRoot
3. 如果root上的wait g的数量等于0, 则直接返回
4. 根据count addr找到第一个wait的g, 唤醒它

关于sema的开销

最优情况下, 即acquire时不需要等待, release时没有g在waiting, 那sema就是atomic的add操作.
如果需要等待, sema增加了额外的几个开销:
1. 可能需要内存分配/释放sudog
2. 需要操作全局semtable, 并对一个semaRoot做mutex

### channel

	type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters
	lock     mutex
	}

#### sync chan

#### async chan

### goroutine

goroutine 相关的函数实现大部分在
runtime/proc1.go
另外, runtime/proc.go 应该是go编译后程序真正的入口

	//生成新的goroutine并将其放到等待执行的队列中
	func newproc1() {
	  // 从local或者global gfreelist 里面分配一个g
	  gfget()
	  // new一个g
	  malg()
	  // 放到local或者global runqueue 等待运行
	  runqput()
	}

#### gopark

### GC

### reference

[A Manual for the Plan 9 assembler](http://www.plan9.bell-labs.com/sys/doc/asm.html)

