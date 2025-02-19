---
layout: post
title: "Go语言底层原理剖析"
date: 2025-02-10
tags: [读书笔记, go]
comments: false
author: 小辣条
toc : true
---

本节内容来源于《Go语言底层原理剖析》，详细介绍了各种结构的原理，希望对大家有帮助
<!-- more -->

# 5、字符串底层原理

Go语言标准库为字符串处理提供了强大的支持，包括用于解析utf-8编码的unicode/utf8包，用于字符串截取和解析的strings包，以及用于字符串与其他类型转换的strconv包。

字符常量(使用rune)存储于静态存储区，其内容不可以被改变，声明时有单撇号和双引号两种方法。字符常量的拼接发生在编译时，而字符串变量的拼接发生在运行时。当拼接后的s字符串小于32字节时，会有一个临时的缓存供其使用。当拼接后的字符串大于32字节时，会请求在堆区分配内存。需要注意的是，字节数组与字符串的相互转换并不是无损的指针引用，而是涉及了复制。因此，在频繁涉及字节数组与字符串相互转换的场景需要考虑转换的成本。


# 6、数组底层原理
特性：不能进行扩容，传递为值传递；类型和长度编译时确定；

备忘点：数组声明方式

数组字面量编译时内存优化：当数组的长度小于4时，在运行时数组会被放置在栈中，当数组的长度大于4时，数组会被放置到内存的静态只读区。

数组越界检查：编译和运行时期都会检查

# 7、切片底层原理
特性：切片是长度可变的序列；一个切片在运行时由指针（data）、长度（len）和容量（cap）3部分构成；

- 截取后的数组仍然指向原始切片的底层数据;
![alt text](https://raw.githubusercontent.com/swh0318/swh0318.github.io/refs/heads/main/_posts/2025/assets/2025-02-10-go_dicengyuanli/image.png)

- 切片完整复制，使用copy函数

- Go语言中切片扩容的策略：

    如果新申请容量（cap）大于2倍的旧容量（old.cap），则最终容量（newcap）是新申请的容量（cap）

    如果旧切片的长度小于1024，则最终容量是旧容量的2倍，即newcap=doublecap。

    如果旧切片长度大于或等于1024，则最终容量从旧容量开始循环增加原来的1/4，即newcap=old.cap，
    for {newcap+=newcap/4}，直到最终容量大于或等于新申请的容量为止，即newcap ≥ cap。

    如果最终容量计算值溢出，即超过了int的最大范围，则最终容量就是新申请容量。

- 切片字面量的初始化，会以数组的形式存储于静态区中。在使用make函数初始化时，如果make函数初始化了一个大于64KB的切片，那么这个切片会逃逸到堆中，在运行时调用makeslice函数创建切片，小于64KB的切片直接在栈中初始化。

- 切片扩容后返回的地址并不一定和原来的地址相同，因此必须小心其可能遇到的陷阱，一般会使用形如a=append（a，T）的方式保证其安全。

# 8、map底层原理
## 解决hash碰撞的方法
- 链地址法
    
    优点：实现简单，适用于链表长度较长的情况

    缺点：需要存储额外的指针用于链接元素，这增加了整个哈希表的大小。同时由于链表存储的地址不连续，所以无法高效利用CPU高速缓存

    ![alt text](https://raw.githubusercontent.com/swh0318/swh0318.github.io/refs/heads/main/_posts/2025/assets/2025-02-10-go_dicengyuanli/image-1.png)

- 开放地址法
    
    优点：不需要存储额外的指针，可以高效利用CPU高速缓存

    缺点：当哈希表中的元素较多时，开放地址法的性能会下降

    ![alt text](https://raw.githubusercontent.com/swh0318/swh0318.github.io/refs/heads/main/_posts/2025/assets/2025-02-10-go_dicengyuanli/image-2.png)

## map的key的比较性
没有办法比较map中的key是否相同，因此map的key只能是可比较的类型，即基本类型、数组、结构体、指针、通道、接口等类型，切片、函数、map是不可比较的

## hash表底层结构
```
type hmap struct {
   count     int   
   flags     uint8
   B         uint8
   noverflow uint16   
   hash0     uint32   
   buckets    unsafe.Pointer   
   oldbuckets unsafe.Pointer   
   nevacuate  uintptr   
   extra ＊mapextra
}
type bmap struct {
    tophash  [8]uint8  // 存储每个键的哈希值的高 8 位
    keys     [8]keyType   // 存储 8 个键
    values   [8]valueType // 存储 8 个值
    overflow *bmap        // 溢出指针，指向下一个桶
}
```
- count代表map中元素的数量。
- flags代表当前map的状态（是否处于正在写入的状态等）。
- 2的B次幂表示当前map中桶的数量，2^B=Buckets size。
- noverflow为map中溢出桶的数量。当溢出的桶太多时，map会进行same-size map growth，其实质是避免溢出桶过大导致内存泄露。
- hash0代表生成hash的随机数种子。
- buckets是指向当前map对应的桶的指针。
- oldbuckets是在map扩容时存储旧桶的，当所有旧桶中的数据都已经转移到了新桶中时，则清空。
- nevacuate在扩容时使用，用于标记当前旧桶中小于nevacuate的数据都已经转移到了新桶中。
- extra存储map中的溢出桶。

## hash表原理图解
- map查找原理

在进行hash[key]的map访问操作时，会首先找到桶的位置，找到桶的位置后遍历tophash数组，如果在数组中找到了相同的hash，那么可以接着通过指针的寻址操作找到对应的key与value。

![alt text](https://raw.githubusercontent.com/swh0318/swh0318.github.io/refs/heads/main/_posts/2025/assets/2025-02-10-go_dicengyuanli/image-3.png)

```
func mapLookup(m *hmap, key KeyType) (value ValueType, ok bool) {
    // 1. 计算哈希值
    hash := hashFunc(key)
    tophash := hash >> (64 - 8) // 取高 8 位

    // 2. 定位桶
    bucketIndex := hash & (1<<m.B - 1) // 取低 B 位
    bucket := m.buckets[bucketIndex]

    // 3. 遍历桶
    for b := bucket; b != nil; b = b.overflow {
        for i := 0; i < 8; i++ {
            if b.tophash[i] == tophash {
                // hash有可能相等，比较键是否相等
                if b.keys[i] == key {
                    return b.values[i], true
                }
            }
        }
    }

    // 4. 未找到
    return zeroValue, false
}
```
在Go语言中还有一个溢出桶的概念，在执行hash[key]=value赋值操作时，当指定桶中的数据超过8个时，并不会直接开辟一个新桶，而是将数据放置到溢出桶中，每个桶的最后都存储了overflow，即溢出桶的指针。在正常情况下，数据是很少会跑到溢出桶里面去的。

- map重建

情况：map超过了负载因子大小；溢出桶的数量过多。

负载因子=哈希表中的元素数量/桶的数量；当负载因子大于6.5时，会触发map的扩容操作。

![alt text](https://raw.githubusercontent.com/swh0318/swh0318.github.io/refs/heads/main/_posts/2025/assets/2025-02-10-go_dicengyuanli/image-4.png)

- map删除原理

![alt text](https://raw.githubusercontent.com/swh0318/swh0318.github.io/refs/heads/main/_posts/2025/assets/2025-02-10-go_dicengyuanli/image-5.png)

# 9、函数与栈
函数可以看作变量，并且它可以作为参数传递、返回及赋值。

闭包（Closure）是指一个函数与其词法环境（定义时的作用域）的组合。即使外部函数已经执行完毕，内部函数仍能访问其定义时的作用域中的变量。
尤其注意range情况下的闭包陷阱

```
错误代码
for _, val := range values {
  go func() {
   fmt.Println(val)
  }()
}
正确代码
for _, val := range values {
    go func(v int) { // 假设values是一个int切片
        fmt.Println(v)
    }(val)
}
```
- 函数栈

每个函数在执行过程中都使用一块栈内存来保存返回地址、局部变量、函数参数等，我们将这一块区域称为函数的栈帧（stack frame）
函数的调用层级越深，消耗的栈空间越大。
![alt text](https://raw.githubusercontent.com/swh0318/swh0318.github.io/refs/heads/main/_posts/2025/assets/2025-02-10-go_dicengyuanli/image-6.png)

- Go语言栈帧结构

函数调用时参数的压栈操作是从右到左进行的；

Go语言函数的参数和返回值存储在栈中，然而许多主流的编程语言会将参数和返回值存储在寄存器中。存储在栈中的好处在于所有平台都可以使用相同的约定，从而容易开发出可移植、跨平台的代码，同时这种方式简化了协程、延迟调用和反射调用的实现。寄存器的值不能跨函数调用、存活，这简化了垃圾回收期间的栈扫描和对栈扩容的处理。

- 栈扩容与栈转移原理

Go语言在线程的基础上实现了用户态更加轻量级的协程，线程的栈大小一般是在创建时指定的，为了避免出现栈溢出（stack overflow）的错误，默认的栈大小会相对较大（例如2MB）。而在Go语言中，每个协程都有一个栈，并且在Go 1.4之后，每个栈的大小在初始化的时候都是2KB。程序中经常有成千上万的协程存在，可以预料到，Go语言中的栈是可以扩容的。Go语言中最大的协程栈在64位操作系统中为1GB，在32位系统中为250MB。

# 10、defer延迟调用
- defer 在发生异常时仍会执行，确保资源清理和收尾工作。
- 可以通过recover捕获panic，防止程序崩溃。
- 用在解锁、计算消耗时间等中间件中
- 参数预计算：当函数到达defer语句时，延迟调用的参数将立即求值，传递到defer函数中的参数将预先被固定，而不会等到函数执行完成后再传递参数到defer中。
- defer返回值陷阱：return其实并不是一个原子操作，返回值保存在栈上→执行defer函数→函数返回

```
示例1
var g = 100
func f() (r int) {    
    defer func() {        
        g = 200    
    }()    
    fmt.Printf("f: g = %d\n", g)    
    return g 
}
func main(){   
    i := f()   
    fmt.Printf("main: i = %d, g= %d\n", i, g)
}

分析结果：
g = 100
r = g
g = 200
return

示例2
var g = 100
func f() (r int) {   
    r = g   
    defer func() {       
        r = 200   
    }()   
    r = 0   
    return r
}

func main() {   
   i := f()
   fmt.Printf("main: i = %d, g= %d\n",i, g)
}

分析结果
g = 100
r = g
r = 0
r = 200
return
```

# 11、异常与异常捕获

- Go语言中内置了panic与recover函数，用于异常的触发与捕获。panic不会导致程序直接退出，而是执行函数调用链上所有的defer语句。如果在defer语句中发现有recover异常捕获，那么recover会返回当前panic传递的异常信息，并使程序正常执行。涉及panic与recover的嵌套时，recover只能捕获当前函数和当前函数调用的函数，不能捕获上层函数，所以需要在代码中合适的位置放置recover。

- 在正常情况下，panic的底层会遍历执行整个_defer链表并返回。对于Go 1.14之后的内联defer，需要遍历函数的栈帧并将_defer结构体加入链表中。如果在遍历执行_defer调用链时有recover，则最终通过修改协程SP、BP寄存器的方式修改协程的执行路径，执行deferreturn函数，遍历完剩余的defer函数并正常返回。

- pannic和recover是Go语言中用于处理异常的两个内建函数。

如下的protect函数是一个中间件，在不破坏函数g的情况下可以避免函数异常退出。
```
func protect(g func()) {
    defer func() {        
        log.Println("done")        
        if x := recover(); x != nil {            
            log.Printf("run time panic: %v", x)       
        }    
    }()    
    log.Println("start")
    g()
}
```
- panic函数底层原理

todo

- recover函数底层原理

todo

# 16、通道与协程间通信

CSP并发编程模型：通过通信来共享内存，而不是通过共享内存来通信

传统的并发模型：共享内存，通过锁来保护共享数据，但是锁的使用会带来死锁、饥饿、活锁等问题。

通道基本使用方式：通道声明与初始化(可读可写、可读、可写)、写入数据、读取数据(可通过第二个参数ok判断是否关闭)、关闭通道(注意panic)

由于通道是Go语言中的引用类型而不是值类型，因此传递到其他协程中的通道，实际引用了同一个通道。

- select多路复用
    
    每个case语句都必须对应通道的读写操作、 select随机选择机制、select与定时器或者超时器配套使用、可使用default语句进行执行

    循环select: for与select结合  

    定时器time.Tick与time.After是有本质不同的。time.After并不会定时发送数据到通道中，而只是在时间到了后发送一次数据。 

    当select语句的case对nil通道进行操作时，case分支将永远得不到执行。

- 通道底层原理

    本质是一个循环队列，通道在运行时是一个特殊的hchan结构体，其中包含了通道的缓冲区、发送队列和接收队列。
    ![alt text](https://raw.githubusercontent.com/swh0318/swh0318.github.io/refs/heads/main/_posts/2025/assets/2025-02-10-go_dicengyuanli/image-7.png)

    通道初始化：通道的初始化在运行时调用了makechan函数，第1个参数代表通道的类型，第2个参数代表通道中元素的大小。makechan会判断元素的大小、对齐等。最重要的是，它会在内存中分配元素大小。

    通道写入原理
    - 有正在等待的读取协程
        通道hchan结构中的recvq字段存储了正在等待的协程链表，每个协程对应一个sudog结构，直接复制到协程缓冲区中，然后唤醒协程。
    - 缓冲区有空余
        当前缓冲区没有满，则向当前缓冲区中写入当前元素
    - 缓冲区无空余
        当前协程的sudog结构需要放入sendq链表末尾中，并且当前协程陷入休眠状态，等待被唤醒重新执行
    
    通道读取原理
    - 有正在等待的写入协程
        当有正在等待的写入协程时，直接从等待的写入协程链表中获取第1个协程，并将写入的元素直接复制到当前协程中，再唤醒被堵塞的写入协程。这样，当前协程将不需要陷入休眠
        ![alt text](https://raw.githubusercontent.com/swh0318/swh0318.github.io/refs/heads/main/_posts/2025/assets/2025-02-10-go_dicengyuanli/image-8.png)
    - 缓冲区有元素：如果队列中没有正在等待的写入协程，但是该通道是带缓冲区的，并且当前缓冲区中有数据，则读取该缓冲区中的数据，并写入当前的读取协程中
    - 缓冲区无元素：如果当前通道无缓冲区或者当前缓冲区已经空了，则代表当前协程的sudog结构需要放入recvq链表末尾，并且当前协程陷入休眠状态，等待被唤醒重新执行，

- select底层原理
    - select语句在运行时会调用核心函数selectgo
    - select中的每个case在运行时都是一个scase结构体，存放了通道和通道中的元素类型等信息
    ![alt text](https://raw.githubusercontent.com/swh0318/swh0318.github.io/refs/heads/main/_posts/2025/assets/2025-02-10-go_dicengyuanli/image-9.png)
    - scase一共有4种具体的类型，分别为caseNil、caseRecv、caseSend及caseDefault，如图16-10所示。caseNil代表当前分支中的通道为nil，caseRecv代表当前分支从通道接收消息，caseSend代表当前分支发送消息到通道，caseDefault代表当前分支为default分支。
    - 在selectgo函数中，有两个关键的序列，分别是pollorder和lockorder; pollorder代表乱序后的scase序列,pollorder通过引入随机数的方式给序列带来了随机性;ockorder是按照大小对通道地址排序的算法，对所有的scase按照其通道在堆区的地址大小，使用了大根堆排序算法进行排序。selectgo会按照该序列依次对select中所有的通道加锁。而按照地址排序的目的是避免多个协程并发加锁时带来的死锁问题
    - select一层循环：一轮循环主要是找出是否有准备好的分支，如果有，则根据具体的情况执行不同的分支。这些分支和普通的通道很类似，主要是将元素写入或读取到当前的协程中，解锁所有的通道，并立即返回
    ![alt text](https://raw.githubusercontent.com/swh0318/swh0318.github.io/refs/heads/main/_posts/2025/assets/2025-02-10-go_dicengyuanli/image-10.png)
    - select二层循环：当select完成一轮循环不能直接退出时，意味着当前的协程需要进入休眠状态并等待select中至少有一个通道被唤醒。不管是读取通道还是写入通道都需要创建一个新的sudog并将其放入指定通道的等待队列，之后当前协程将进入休眠状态。

# 17、并发控制
本章还将介绍Go语言中重要的并发控制手段，其中包括处理协程优雅退出的context、检查并发数据争用的race工具，以及传统的同步原语——锁。

为什么需要contex？
- 优雅退出：在Go语言中，协程是一种轻量级的线程，它的创建和销毁都是非常快速的。但是在实际的应用中，我们往往需要在协程中执行一些耗时的操作，这时候就需要一种机制来优雅地退出协程。
- 超时控制：在实际的应用中，我们往往需要对一些耗时的操作进行超时控制，以避免程序因为某个协程的阻塞而导致整个程序的阻塞。
- 传递请求范围：在实际的应用中，我们往往需要在不同的协程之间传递请求范围，以便在不同的协程中获取请求的上下文信息。

context的使用方式
```
type Context interface {
    Deadline() (deadline time.Time, ok bool)    
    Done() <-chan struct{}    
    Err() error   
    Value(key interface{}) interface{}
}
```
Deadline方法的第一个返回值表示还有多久到期，第二个返回值表示是否到期。Done是使用最频繁的方法，其返回一个通道，一般的做法是监听该通道的信号，如果收到信号则表示通道已经关闭，需要执行退出。如果通道已经关闭，则Err（）返回退出的原因。value方法返回指定key对应的value，这是context携带的值。Value主要用于安全凭证、分布式跟踪ID、操作优先级、退出信号与到期时间等场景

context的退出与传递
- context.Background函数或context.TODO函数会返回最简单的context实现。context.Background函数一般作为根对象存在，其不可以退出，也不能携带值
```
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
func WithValue(parent Context, key, val interface{}) Context
```
WithCancel函数返回一个子context并且有cancel退出方法。子context在两种情况下会退出，一种情况是调用cancel，另一种情况是当参数中的父context退出时，该context及其关联的子context都将退出。

WithTimeout函数指定超时时间，当超时发生后，子context将退出。因此子context的退出有3种时机，一种是父context退出；一种是超时退出；一种是主动调用cancel函数退出。

WithDeadline和WithTimeout函数的处理方法相似，不过其参数指定的是最后到期的时间。WithValue函数返回带key-value的子context。


## 参考
书籍：Go语言底层原理剖析
