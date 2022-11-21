

# Go八股文大全

哈喽大家好，我是`asong`，自己整理了一份八股文文档，这份文档都是都是面试真题，**并不是很全，欢迎大家补充，我们一起完善这份文档；**

**欢迎关注公众号：【Golang梦工厂】，定期分享Go语言知识；**

**注意：以下整理的答案并非全部原创，本文只做整合！！！**

![](https://song-oss.oss-cn-beijing.aliyuncs.com/golang_dream/article/static/%E6%89%AB%E7%A0%81_%E6%90%9C%E7%B4%A2%E8%81%94%E5%90%88%E4%BC%A0%E6%92%AD%E6%A0%B7%E5%BC%8F-%E7%99%BD%E8%89%B2%E7%89%88.png)

## 1. Go股大全

### 1.1 flag库了解吗？有什么陷阱？

flag是Go官方提供的标准库，flag包实现了命令行的解析，flag使得开发命令行工具更为简单；

陷阱一：

当我们把flag放置在cli应用的最后面时，需要小心参数传递的顺序，flag包的命令行参数的解析逻辑是：当碰到第一个非flag参数时，便停止解析，所以如果传入非法参数就会导致后面的参数解析错误；

陷阱二：

对于bool类型的flag参数，只支持以下两种形式：

```go
-arg
-arg=value
```

其他形式都会导致解析失败；



### 1.2 Go语言中的nil是什么？

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247484793&idx=1&sn=713c084f9bd23a6fbfaafe491244b1b6&chksm=c22b8325f55c0a3371acdb35ea5db04dff3a59d13e00612e86071bd4a1d626c566de435bcf01&token=337827600&lang=zh_CN#rd

`nil`不是关键字，是一个预先声明的标识符，指针、通道、函数、接口、map、切片的零值就是nil，`nil`是没有默认类型的，他的类型具有不确定性，我们在使用它时必须要提供足够的信息能够让编译器推断nil期望的类型；

两个nil不能进行比较，因为nil是无类型的；

1. 声明一个nil的map，map可以读数据，但是不能写数据
2. 关闭一个nil的channel会引发panic
3. nil切片不能进行索引访问，会引发panic
4. 方法接收者为nil时，如果在方法内使用到了会引发panic
5. 空指针一个没有任何值的指针



### 1.3 context了解吗？

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247485639&idx=1&sn=a544928d4ce1364bc6b8e61247da0aed&chksm=c22b8e9bf55c078dbdfe471883260f7b59f706a7c2b286808379092de61935deb86a701b334a&token=348125870&lang=zh_CN#rd

context包是在1.7版本引入的，context可以用在goroutine之间传递上下文信息，相同的context可以传递给运行在不同goroutine中的函数，上下文对于多个goroutine同时使用是安全的， context官方建议被当作第一个参数，并且不断透传下去，context可以使用background、TODO创建一个上下文，在函数调用链之间传播context，也可以使用`withDeadline`、`WithTimeout`、`WithCancel`或`WithValue`创建的修改副本替换他，总结：context的作用就是在不同的`goroutine`之间同步请求特定的数据、取消信号以及处理请求的截止日期。

我们常用的一些库都支持context，例如gin、database/sql等库都是支持context的，这样更方便我们做并发控制了；

**实现原理：**

基于一个父context可以随意衍生，这就是一个context树，树的每个节点都可以由任意多个节点，节点层级可以由任意多个，每个子节点都依赖于其父节点；

使用withValue时的注意事项：

- 不建议使用context值传递关键参数，关键参数应该显示声明出来，不应该隐式处理，context中最好时携带签名、trace_id这类值；
- 因为携带value也是key、value的形式，为了避免context因多个包同时使用context而带来冲突，key建议使用内置类型；
- 上面的例子我们获取`trace_id`是直接从当前`ctx`获取的，实际我们也可以获取父`context`中的`value`，在获取键值对是，我们先从当前`context`中查找，没有找到会在从父`context`中查找该键对应的值直到在某个父`context`中返回 `nil` 或者查找到对应的值。
- `context`传递的数据中`key`、`value`都是`interface`类型，这种类型编译期无法确定类型，所以不是很安全，所以在类型断言时别忘了保证程序的健壮性。

`withTimeout`、`WithDeadline`不同在于`WithTimeout`将持续时间作为参数输入而不是时间对象，这两个方法使用哪个都是一样的，看业务场景和个人习惯了，因为本质`withTimout`内部也是调用的`WithDeadline`。

日常业务开发中我们往往为了完成一个复杂的需求会开多个`gouroutine`去做一些事情，这就导致我们会在一次请求中开了多个`goroutine`确无法控制他们，这时我们就可以使用`withCancel`来衍生一个`context`传递到不同的`goroutine`中，当我想让这些`goroutine`停止运行，就可以调用`cancel`来进行取消。

**使用context优点：**

- 使用context可以更好的做并发控制，能更好的管理goroutine滥用
- context的携带者功能没有任何限制
- context包解决了goroutine的cancelation问题

**使用context的缺点：**

1. 影响代码美观，基本现在所有web框架都实现了context，这就导致我们的代码中每一个函数的一个参数都是context
2. `context`可以携带值，但是没有任何限制，类型和大小都没有限制，也就是没有任何约束，这样很容易导致滥用，程序的健壮很难保证；还有一个问题就是通过`context`携带值不如显式传值舒服，可读性变差了。
3. 可以自定义`context`，这样风险不可控，更加会导致滥用。
4. `context`取消和自动取消的错误返回不够友好，无法自定义错误，出现难以排查的问题时不好排查。
5. 创建衍生节点实际是创建一个个链表节点，其时间复杂度为O(n)，节点多了会掉支效率变低。


### 1.4 字符串

#### 1.4.1 string和[]byte转换会发生内存拷贝吗？

**https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247484941&idx=1&sn=326abc6706a092e36af32a5b682e6e3c&chksm=c22b8051f55c0947df20327d7993878f1086c5391ec5890ae78e17df2c7627bd4bf345010f9b&token=337827600&lang=zh_CN#rd**

type byte = uint8 byte就是uint8的别名，用来区分字节值和8位无符号整数值；

type string string string是一个8位字节的集合，通常但不一定代表UTF-8编码的文本，string可以为空，但是不能为nil；

string类型本质也是一个结构体：

```go
type stringStruct struct {
    str unsafe.Pointer
    len int
}
```

![image-20221009230237785](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221009230237785.png)

string类型底层是一个`byte`类型的数组，因为Go语言中string类型被设计为不可变的，`string`类型虽然不能更改的，但是可以被替换，`stringStruct`中的str指针是可以改变的，只是指针指向的内容是不可以改变的；可以看出来，指针指向的位置发生了变化，也就是每更改一次字符串，就需要重新分配一次内存，之前分配的空间会被gc；

介绍了基本的数据类型后，我们再来看一下string和[]byte的标准转换方法：

string转换为[]byte：具体实现在`src/runtime/slice.go`中的`slicestringcopy`方法，这里就不贴这段代码了，这段代码的核心思路就是：**将string的底层数组从头部复制n个到[]byte对应的底层数组中去，预先定义一个长度为32的数组，字符串的长度超过了这个数组的长度，就重新分配一块内存，否则就是用预先分配的内存来做**

[]byte转换为string：根据`[]byte`的长度来决定是否重新分配内存，最后通过`memove`可以拷贝数组到字符串。

标准的方法都会发生内存拷贝；

因为string的结构和切片的基本除了cap基本一致，所以可以使用强制转换的方式对两者进行转换，因为内存布局是一致的，所以就可以直接替换指针；

推荐大家使用标准的转换方式，因为她更安全，在一些高性能场景可以使用强制转换来做优化，但是要注意使用方式，毕竟这是一种不安全的写法，比如string转换为[]byte类型后，直接对byte数组中的元素进行修改会导致发生严重的错误，即使使用defer+recover也无法捕获；


#### 1.4.2 rune类型

官方对`rune`的定义如下：

```go
// rune is an alias for int32 and is equivalent to int32 in all ways. It is
// used, by convention, to distinguish character values from integer values.
type rune = int32
```

人工翻译：

`rune`是`int32`的别名，在所有方面都等同于`int32`，按照约定，它用于区分字符值和整数值。

说的通俗一点就是`rune`一个值代表的就是一个`Unicode`字符，因为一个`Go`语言中字符串编码为`UTF-8`，使用`1-4`字节就可以表示一个字符，所以使用`int32`类型范围就可以完美适配。



#### 1.4.3 len函数

Go语言源代码为UTF-8编码，Go语言获取字符串的字节长度使用len函数，获取字符串的字符个数使用`utf-8.RuneConutString`函数或者转换为rune切片求其长度；




### 1.5 Go语言的指针类型

Go语言的指针类型相比C语言的指针加了很多限制，这也是处于安全考虑，这样既可以享受指针带来的便利，又避免了指针的危险性，限制如下：

1. Go语言的指针不能进行数学运算
2. 不同类型的指针不能相互转换
3. 不同类型的指针不能作比较
4. 不同类型的指针不能相互赋值

#### 1.5.1 unsafe包

https://www.sohu.com/a/319106990_657921

Go语言的指针类型是安全的，有很多限制，但是有些场景我们使用非安全型指针更方便快捷，所以在unsafe包中提供了unsafe.Pointer类型，unsafe包用于Go编译器，在编译阶段使用，从名字就可以看出来，他是不安全的，官方并不建议使用，我们在用unsafe包的时候会有一种不舒服的感觉，可能这也是语言设计者的意图吧。但是有些场景我们还是要使用的，它可以绕过Go语言的类型系统，直接操作内存，例如：我们一般不能操作结构体的未导出成员，但是通过unsafe包就能做到，unsafe包让我们可以直接读写内存，不需要管导出与未导出；

Go语言类型系统为了安全和效率设计，有时，安全会导致效率低下，有了unsafe包，高阶的程序员就可以利用它绕过类型系统的低效，Go源码中有大量使用unsafe包的例子；

unsafe包提供unsafe.Pointer类型本质一个*int，他可以指向任意类型：

1. 任何类型的指针和unsafe.Pointer可以相互转换
2. uintptr类型和unsafe.Pointer可以相互转换

unsafe.Pointer不能直接进行数学运算，但是它可以转换成uintptr，对uintptr类型进行数学运算，在转换成pointer类型；

uintptr是一个整数类型，他足够大可以存储，uintptr并没有指针语义，uintptr所指向的对象会被gc回收，而unsafe.Pointer有指针语义，可以保护他所指向的对象在"有用"的时候不会被垃圾回收；

![image-20221010102858741](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221010102858741.png)

unsafe包提供三个函数：

1. func Sizeof(x ArbitraryType) uinptr : 返回x所占据的字节数，但不包含x所指向的内容的大小
2. func Offset(x ArbitraryType) uinptr：返回结构体成员在内存中的位置距离结构体起始处的字节数，所传参数必须是结构体的成员
3. func Alignof(x ArbitraryType) uintptr：返回对应参数的类型需要对齐的倍数



#### 1.5.2 uinptr和unsafe.Pointer的区别

- unsafe.Pointer只是单纯的通用指针类型，用于转换不同类型的指针，他不可以参与指针运算
- uintptr是用于指针运算的，GC不把uintptr当指针，也就说uintptr无法持有对象，uintptr类型的目标会被回收
- unsafe.Pointer 可以和 普通指针 进行相互转换；
- unsafe.Pointer 可以和 uintptr 进行相互转换。



#### 1.5.3 为什么屏蔽掉不同指针类型转换

在C语言中不同指针类型是可以直接进行转换，但是在Go语言中需要先将指针类型转换为unsafe.Pointer，然后在进行不同指针类型的转换，不同指针类型转换会遇到如下问题：内存截断或内存访问的扩展；



### 1.6 结构体

#### 1.6.1 Go语言的结构体的内存对齐

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247485151&idx=1&sn=a91a622a0ff70221232b125bab4c4d2d&chksm=c22b8083f55c09954e95c9d5ff2e6e4cdb721e20acdf0ec89c2c611c329c4014cdb33ba36dc5&token=337827600&lang=zh_CN#rd

**什么是内存对齐**

现代计算机内存空间都是按照字节（byte）进行划分的，所以理论上将对于任何类型的变量访问都可以从任意地址开始，但是实际情况中，在访问特定类型变量的时候经常在特定的内存地址访问，所以就需要把各种类型数据按照一定的规则在空间上排列，而不是按照顺序一个接一个的排放，这种就称为内存对齐；

**为何要有内存对齐**

1. 有些CPU是可以访问任意地址上的任意数据，而有些CPU只能在特定地址访问数据，因此不同硬件平台具有差异性，这样的代码就不具有移植性，如果在编译时，将分配的内存进行对齐，这就具有平台移植性了；
2. CPU每次寻址都是要浪费时间的，并且CPU访问内存时，并不是逐个字节访问，而是以字长为单位访问，所以数据结构应该尽可能地在自然边界上对齐，如果访问未对齐的内存，处理器需要做两次内存访问，而对齐的内存访问仅需要一次访问，内存对齐后可以提升性能。

**内存对齐规则**

每个特定平台都有自己的内存对齐系数，常用平台默认对齐系数如下：

- 32位系统对齐系数是4
- 64位系统对齐系数是8

这只是默认对齐系数，实际上对齐系数我们是可以修改的；

C语言的对齐规则与Go语言一样，所以C语言的对齐规则对Go同样适用：

- 对于结构体的各个成员，第一个成员位于偏移为`0`的位置，结构体第一个成员的偏移量(offset)为`0`，以后每个成员相对于结构体首地址的`offset`都是该成员大小与有效对齐值中较小那个的整数倍，如有需要编译器会在成员之间加上填充字节。
- 除了结构成员需要对齐，结构本身也需要对齐，结构的长度必须是编译器默认的对齐长度和成员中最长类型中最小的数据大小的倍数对齐。

**结构体中成员变量的顺序会影响结构体的内存布局，需要注意这个问题，节省内存空间。**

**特殊：空结构体作为第一位或者中间成员是，占用内存为0，作为最后一个成员时，占用内存等于前一个成员变量大小**



#### 1.6.2 空结构体

1. 空结构体也是一个结构体，不过他的size为0，所有的空结构体内存分配都是同一个地址，都是zeroBase的地址；
2. 空结构体作为内嵌字段时需要注意放置的顺序，当作为最后一个字段时会进行特殊填充，会被填充到对齐前一个字段的大小，地址偏移对齐规则不变；

#### 1.6.3 结构体打tag

Go语言提供了可通过反射发现的结构体标签，这些在标准库`JSON/XML`中得到了广泛的使用，`orm`框架也支持了结构体标签，上面的哪个例子的使用就是因为`encoding/json`持了结构体标签，不过他有自己的标签规则；但是他们都有一个总体规则，这个规则是不能更改的，具体格式如下：

```
`key1:"value1" key2:"value2" key3:"value3"...`  // 键值对用空格分隔
```

结构体标签可以有多个键值对，键与值要用冒号分隔，值要使用双引号括起来，多个键值对之间要使用一个空格分隔，千万不要使用逗号！！！

不同库实现的tag也都不相同，需要根据库文档获取；

结构体标签实在编译阶段就和成员进行关联的，以字符串的形式进行关联，在运行阶段可以通过反射读取出来；

**如何获取tag**

- 通过reflect.typeOf获取类型
- NumField()可以获取该结构体含有几个字段
- 遍历结构体字段，获取各个成员的tag属性

**json包不能导出私有变量的tag，这个不是因为反射获取不到，是因为json包里进行了特殊处理，如果是私有变量直接continue了，不会获取tag**



### 1.7 singleFligh库

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247484988&idx=1&sn=dff9fc3c072545354365b85765cc126f&chksm=c22b8060f55c0976a1c05a578270414bf144190453739477d9789c7faeeb451024c5ca7d7fd3&token=1817605393&lang=zh_CN#rd

singleflight能将对同一个资源访问的多个请求合并为一个请求，常见的应用场景比如缓存击穿。具体实现是使用了map对同一个资源访问的请求进行去重，使用互斥锁让当个协程进入临界区后进行资源访问，其他线程阻塞等待资源访问完后，共同拿到访问资源的结果返回。

singleFlight可以预防缓存击穿，内部结构如下：

```go
type Group struct {
    mu sync.Mutex // 锁住资源
    m map[string]*call // 存储相同的请求
}

type call struct {
 wg sync.WaitGroup
 // 存储返回值，在wg done之前只会写入一次
 val interface{}
  // 存储返回的错误信息
 err error

 // 标识别是否调用了Forgot方法
 forgotten bool

 // 统计相同请求的次数，在wg done之前写入
 dups  int
  // 使用DoChan方法使用，用channel进行通知
 chans []chan<- Result
}
// Dochan方法时使用
type Result struct {
 Val    interface{} // 存储返回值
 Err    error // 存储返回的错误信息
 Shared bool // 标示结果是否是共享结果
}
```

提供`Do`方法、`DoChan`方法：

Do方法是同步等待进行，使用waitGroup进行控制，`Dochan`方式是异步等待进行的，返回一个channel等待结果；

Forget方法用来删除某个`key`;



### 1.8 并发编程

#### 1.8.1 waitGroup

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247484784&idx=1&sn=368be2e2003b85f0e26337b566d0ebde&chksm=c22b832cf55c0a3a550ce385de1f073c91b7d90211086046b671a0483d75a9de7158db870540&token=337827600&lang=zh_CN#rd

waitGroup用于并发编程协同等待场景，一个waitGroup对象可以等待一组协程结束，也就时等待一组goroutine返回；有了`sync.Waitgroup`我们可以将原本顺序执行的代码在多个`Goroutine`中并发执行，加快程序处理的速度。其实他与`java`中的`CountdownLatch`类似，用于阻塞等待所有任务完成之后再继续执行。

有了`sync.Waitgroup`我们可以将原本顺序执行的代码在多个`Goroutine`中并发执行，加快程序处理的速度。其实他与`java`中的`CountdownLatch`类似，用于阻塞等待所有任务完成之后再继续执行。

有了`sync.Waitgroup`我们可以将原本顺序执行的代码在多个`Goroutine`中并发执行，加快程序处理的速度。其实他与`java`中的`CountdownLatch`类似，用于阻塞等待所有任务完成之后再继续执行。

**源码实现：**

基本结构：

```GO
// A WaitGroup must not be copied after first use.
type WaitGroup struct {
 noCopy noCopy

 // 64-bit value: high 32 bits are counter, low 32 bits are waiter count.
 // 64-bit atomic operations require 64-bit alignment, but 32-bit
 // compilers do not ensure it. So we allocate 12 bytes and then use
 // the aligned 8 bytes in them as state, and the other 4 as storage
 // for the sema.
 state1 [3]uint32
}
```

noCopy是为了保证该结构体不会被进行拷贝，这是一种保护机制，

waitGroup实现也是比较简单的，就是用了一个字段`state1`，`state1`主要存储着状态和信号量，总共被分配了12个字节，当数组的首地址是处于一个8字节对齐的位置上时，那么就将这个数组的前8个字节作为64位值表示状态，后4个字节作为32位值表示信号量；同理如果首地址没有处于8字节对齐的位置上时，那么就将前4个字节作为`semaphore`，后8个字节作为64位数值；

![image-20221010192358917](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221010192358917.png)

当goroutine waiter计数器减为0时，唤醒所有waiter；

- ADD方法不能和wait方法并发同时调用，ADD方法要在wait方法之前调用，
- ADD设置的值必须与实际等待的goroutine个数一致，否则panic
- Done方法只是对ADD方法的简单封装，我们可以向ADD方法传入任意复数，快速将计数器归零以唤醒等待的`Goroutine`，但是要保证goroutiner计数器不能为负数；
- waitGroup不能进行拷贝，否则会有意向不到的bug；





### 1.9 Go语言实现的no copy机制

```Go
type noCopy struct{}

// Lock is a no-op used by -copylocks checker from `go vet`.
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
```

因为如果结构体成员有指针类型，当该对象被拷贝时，会使得两个对象中的指针字段变得不再安全；所以为了安全性需要提供保护机制防止对象复制；Go语言中提供了两种`copy`机制，一种是在运行时检查，一种是静态检查，运行检查是影响程序性能的，Go官方目前只提供了strings.Builder和sync.Cond的runtime拷贝检查机制，对于其他需要nocopy对象类型来说，使用go vet工具来做静态编译检查。运行检查的实现可以通过比较所属对象是否发生变更，静态检查是提供了一个`nocopy`对象，只要是该对象或对象中存在`nocopy`字段，他就实现了`sync.Locker`接口, 它拥有Lock()和Unlock()方法，之后，可以通过go vet功能，来检查代码中该对象是否有被copy。



### 1.10 channel

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247484848&idx=1&sn=39c3d223bffdd8fe0238f2ab9720fac6&chksm=c22b83ecf55c0afac78913baf908f47abb0eddd7122a277dda77b656e126e5f612c0ab792b40&token=337827600&lang=zh_CN#rd

#### 1.10.1 什么是channel

channel是用于goroutine的数据通信，goroutine+channel的方式可以简单，高效的解决并发问题；

channel分为有缓存的channel和无缓冲的channel，无缓冲的channel必须双方都准备好才可以进行通讯，在无缓冲的channel中，只有出队方准备好，channel才会入队，否则就一直阻塞这，所以说无缓冲channel是同步的；在有缓冲的channel中，缓存未满时，就会执行入队操作；

channel的设计思想：**不要通过共享内存来通信，而是通过通信来实现共享内存**

共享内存来通信：实际就是多个线程/协程使用同一块内存，通过加锁的方式来宣布使用某块内存，通过解锁的来宣布不再使用某块内存；

通信来实现共享内存：使用发送消息来同步信息相比于直接使用共享内存和互斥锁是一种更高级得抽象，使用更高级得抽象可以能够让我们在程序设计上提供更好的封装，让程序得逻辑更加清晰，其次，消息发送在解耦方面与共享内存相比也有一定得优势，我们可以将线程得职责分成生产者和消费者，并通过消息传递得方式将他们解耦，不需要再依赖共享内存；

channel本质是一个有锁得环形队列，包括发送方队列、接收方队列、互斥锁等结构；

channel遵循先进先出的设计：

- 先从channel读取数据的goroutine会接收到数据
- 先向channel发送数据的channel会得到先发送数据的权力；



#### 1.10.2 select

select与switch具有相似的控制结构，与switch不同的是，select中的case中表达式必须是channel的收发操作，当select中两个case同时被触发时，会随机执行其中一个，随机的引入为了避免饥饿问题发生，如果每次都是按照顺序依次执行的，若两个case一直都是满足条件的，那么后面的case永远都不会执行；select中也可以使用default语句，当不存在可以收发的channel时，执行default中的语句



#### 1.10.3 往一个nil的channel发送数据

往一个nil的channel中发送数据时，会调用gopark函数将当前执行的goroutine从running状态转入waiting状态，导致进程出现死锁，表象出panic；



#### 1.10.4 一个nil的channel接收数据

```GO
if c == nil {
  if !block {
   return
}
  gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
  throw("unreachable")
}
```

根据写法决定，如果当前`nil channel`为非阻塞接收，则直接返回即可，如果`nil channel`为阻塞接收，则直接调用`gopark`方法挂起当前`goroutine`，造成死锁；



#### 1.10.4 关闭channel

使用close可以关闭channel，其经过编译器编译后对应的时runtime.closechan方法：

1. 一个为nil得channel不允许进行关闭
2. 不可以重复关闭channel
3. 获取当前正在阻塞的发送或者接收的`goroutine`，他们都处于挂起状态，然后进行唤醒。这是发送方不允许在向`channel`发送数据了，但是不影响接收方继续接收元素，如果没有元素，获取到的元素是零值。使用`val,ok := <-ch`可以判断当前`channel`是否被关闭。



#### 1.10.5 对已经关闭的chan进行读写，会怎么样？

已经关闭的chan能一直读东西，读到的内容根据通道内关闭前是否有元素而不同：

- 如果chan关闭前，buffer内有元素还未读，会正确读到chan值，且返回的第二个bool值为true
- 如果 `chan` 关闭前，`buffer` 内有元素**已经被读完**，`chan` 内无值，接下来所有接收的值都会非阻塞直接成功，返回 `channel` 元素的**零值**，但是第二个 `bool` 值一直为 `false`。
- 写**已经关闭**的 `chan` 会 `panic`



### 1.11 切片

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247486198&idx=1&sn=37797d74252df97fe15c85cf1563904d&chksm=c22b8caaf55c05bcd588ec423f567642e59b3542df69352ad7779b0099d2d3e17cc55d2d68e1&token=337827600&lang=zh_CN#rd

#### 1.11.1 数组和切片有什么区别？

`Go`语言中数组是固定长度的，不能动态扩容，在编译期就会确定大小，声明方式如下：

```
var buffer [255]int
buffer := [255]int{0}
```

切片是对数组的抽象，因为数组的长度是不可变的，在某些场景下使用起来就不是很方便，所以`Go`语言提供了一种灵活，功能强悍的内置类型切片("动态数组")，与数组相比切片的长度是不固定的，可以追加元素。切片是一种数据结构，切片不是数组，切片描述的是一块数组，切片结构如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/CqB2u93NwBibSrR2QWSGcIxMu1LPXC8KAxK10ZexxMQl6E1gmY9AEwbWMu28ibD4lgicVaicanWcUiaGjuXNMRnV08g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

我们可以直接声明一个未指定大小的数组来定义切片，也可以使用`make()`函数来创建切片，声明方式如下：

```
var slice []int // 直接声明
slice := []int{1,2,3,4,5} // 字面量方式
slice := make([]int, 5, 10) // make创建
slice := array[1:5] // 截取下标的方式
slice := *new([]int) // new一个
```

切片可以使用`append`追加元素，当`cap`不足时进行动态扩容。

#### 1.11.2 拷贝大切片一定比小切片代价大吗？

这道题本质是考察对切片本质的理解，`Go`语言中只有值传递，所以我们以传递切片为例子：

```
func main()  {
 param1 := make([]int, 100)
 param2 := make([]int, 100000000)
 smallSlice(param1)
 largeSlice(param2)
}

func smallSlice(params []int)  {
 // ....
}

func largeSlice(params []int)  {
 // ....
}
```

切片`param2`要比`param1`大`1000000`个数量级，在进行值拷贝的时候，是否需要更昂贵的操作呢？

实际上不会，因为切片本质内部结构如下：

```
type SliceHeader struct {
 Data uintptr
 Len  int
 Cap  int
}
```

切片中的第一个字是指向切片底层数组的指针，这是切片的存储空间，第二个字段是切片的长度，第三个字段是容量。将一个切片变量分配给另一个变量只会复制三个机器字，大切片跟小切片的区别无非就是 `Len` 和 `Cap`的值比小切片的这两个值大一些，如果发生拷贝，本质上就是拷贝上面的三个字段。



#### 1.11.3 切片的深浅拷贝

深浅拷贝都是进行复制，区别在于复制出来的新对象与原来的对象在它们发生改变时，是否会相互影响，本质区别就是复制出来的对象与原对象是否会指向同一个地址。在`Go`语言，切片拷贝有三种方式：

- 使用`=`操作符拷贝切片，这种就是浅拷贝
- 使用`[:]`下标的方式复制切片，这种也是浅拷贝
- 使用`Go`语言的内置函数`copy()`进行切片拷贝，这种就是深拷贝



#### 1.11.4 零切片、空切片、nil切片是什么

零切片：我们把切片内部数组的元素都是零值或者底层数组的内容全是nil的切片叫做零切片，使用`MAKE`创建的长度、容量都不为0的切片就是零值切片；

nil切片：nil切片的长度和容量都为0，并且和nil的比较结果为true，采用直接创建切片的方式，`new`创建的切片的方式都可以创建nil切片；

空切片：空切片的长度和容量也都为0，但是和nil的比较结果为false，因为所有的空切片的数据指针都指向同一个地址`0XC42003BDA0`，使用字面量、make可以创建空切片；



#### 1.11.5 切片的扩容策略

切片的扩容都是调用`growSlice`方法；

切片在扩容时会进行内存对齐，这个和内存分配策略有关，进行内存对齐后切片扩容的容量要大于等于旧的切片容量的2倍或者1.25倍，当原切片容量小于1024的时候，新切片容量按照原来的2倍进行扩容，原slice容量超过1024的时候，新slice容量变成原来的1.25倍；



#### 1.11.6 参数传递切片和切片指针有什么区别？

我们都知道切片底层就是一个结构体，里面有三个元素：

```
type SliceHeader struct {
 Data uintptr
 Len  int
 Cap  int
}
```

分别表示切片底层数据的地址，切片长度，切片容量。

当切片作为参数传递时，其实就是一个结构体的传递，因为`Go`语言参数传递只有值传递，传递一个切片就会浅拷贝原切片，但因为底层数据的地址没有变，所以在函数内对切片的修改，也将会影响到函数外的切片，举例：

```
func modifySlice(s []string)  {
 s[0] = "song"
 s[1] = "Golang"
 fmt.Println("out slice: ", s)
}

func main()  {
 s := []string{"asong", "Golang梦工厂"}
 modifySlice(s)
 fmt.Println("inner slice: ", s)
}
// 运行结果
out slice:  [song Golang]
inner slice:  [song Golang]
```

不过这也有一个特例，先看一个例子：

```
func appendSlice(s []string)  {
 s = append(s, "快关注！！")
 fmt.Println("out slice: ", s)
}

func main()  {
 s := []string{"asong", "Golang梦工厂"}
 appendSlice(s)
 fmt.Println("inner slice: ", s)
}
// 运行结果
out slice:  [asong Golang梦工厂 快关注！！]
inner slice:  [asong Golang梦工厂]
```

因为切片发生了扩容，函数外的切片指向了一个新的底层数组，所以函数内外不会相互影响，因此可以得出一个结论，当参数直接传递切片时，**如果指向底层数组的指针被覆盖或者修改（copy、重分配、append触发扩容），此时函数内部对数据的修改将不再影响到外部的切片，代表长度的len和容量cap也均不会被修改**。

参数传递切片指针就很容易理解了，如果你想修改切片中元素的值，并且更改切片的容量和底层数组，则应该按指针传递。



#### 1.11.7 for range 遍历切片有什么要注意的吗？

`Go`语言提供了`range`关键字用于for 循环中迭代数组(array)、切片(slice)、通道(channel)或集合(map)的元素，有两种使用方式：

```
for k,v := range _ { }
for k := range _ { }
```

第一种是遍历下标和对应值，第二种是只遍历下标，使用`range`遍历切片时会先拷贝一份，然后在遍历拷贝数据：

```GO
a := []int{1, 2, 3}
for k, v := range a{
    
}
// 会被编译器认为结构如下：
for_temp := a
len_temp := len(for_temp)
for index_temp:=0; index_temp < len_temp; index_temp++{
    value_temp := for_temp[index_temp]
    k := index_temp
    value := value_temp
}
```

所需要注意的是，如果使用range遍历切片得到的成员值是拷贝数据，修改拷贝数据不会对切片造成影响；



### 1.12 make和new的区别

- new是一个分配内存的内置函数，传入参数是类型，不是值，返回的值是指向该类型新分配的零值的指针，我们平常在使用指针的时候是需要分配内存空间的，未分配内存空间的指针直接使用会使程序崩溃；
- make函数是专门支持`slice`、`map`、`channel`三种数据类型的创建，make内置函数分配并初始化一个`slice`、`map`、`chan`；类型，使用make初始化切片如果指定长度，会初始化值为零值；



### 1.13 讲一讲内存逃逸

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247485075&idx=1&sn=8775dce15d49bc865e3a3d0225451923&chksm=c22b80cff55c09d9e7fef61c577f543133f5be0cc3d3c3089530b068ec25cbfd4fec837a547b&token=337827600&lang=zh_CN#rd

内存空间包含两个重要的区域，堆区和栈区，栈区域会专门存放函数的参数、局部变量等，栈有编译器自动分配与释放；堆区域在C语言中需要手动调用`malloc`函数去堆区域申请内存，使用完毕后需要手动调用free函数进行释放，如果没有释放就会导致内存泄漏，因为每一个函数都会分配一个栈帧，在函数运行结束后进行销毁，但是有些变量我们并不想让他在函数运行结束后销毁，那么我们就需要把这个变量放到堆上分配，这种从栈上逃逸到堆上的现象就称为内存逃逸；

C语言没有那么智能，没有逃逸分析，但是Go语言属于现代编程语言，提供了逃逸分析，堆内存的分配与释放完全不需要程序员来管理，Go语言引入了`gc`机制，`GC`机制会对位于堆上的对象进行自动管理，当某个对象不可达时，他将会被回收并被重用；虽然引入了GC可以让开发人员降低对内存管理的心智负担，但是GC也会给程序带来性能损耗，当堆内存中有大量待扫描的堆内存对象时，将会给GC带来过大的压力，虽然Go语言使用的时标记清除算法，并且在此基础上使用了三色标记算法和写屏障技术，提高了效率，但是如果我们的程序仍在堆上分配了大量内存，依然会对GC造成不可忽视的压力，因此为了减少GC造成的压力，Go语言引入了逃逸分析，也就是想尽办法减少在堆上的内存分配；

Go语言实现的逃逸分析分为两个版本：1.13以前一个版本，1.13以后一个版本，代码路径在src/cmd/compile/internal/gc/escape.go，其逃逸分析原理如下：

- 指向栈对象的指针不能存储在堆上
- 指向栈对象的指针不能超过该对象的存活期，也就是指针不能再栈对象被销毁后依旧存活；

因为逃逸分析是在编译阶段进行的，那么我们可以通过`go builds -gcflags '-m -m -l'`命令查看到逃逸分析结果；

几个逃逸分析的例子：

1. 函数返回局部指针变量
2. interface类型逃逸分析
3. 闭包产生的逃逸
4. 变量大小不确定及栈空间不足引发逃逸



### 1.14 装饰器或中间件如何实现？

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247484962&idx=1&sn=111966677e6c9d5c3e9097ccba78e4fe&chksm=c22b807ef55c09684a13341ba69ee6b03ee6fbc335c3a1e91c247fbcadd341753a9dd112a7c7&token=337827600&lang=zh_CN#rd

python/Java代码是都支持装饰器语法糖的，但是装饰器的使用可以大大减少我们代码的重复量，使用场景如下：

1. 插入日志，面向切面编程
2. 缓存：读写缓存使用装饰器来实现，减少了冗余代码
3. 事务处理：使代码看起来更简洁了
4. 权限校验：权限校验器都是一套代码，减少了冗余代码

装饰器的实现和闭包是分不开的，闭包指延伸了作用域函数，闭包是一种函数，它会保留定义函数时存在的自由变量的绑定，这样调用函数时，虽然定义作用域不可用了，但是仍能使用那些绑定。

在Gin框架中可以使用中间件，他的原理就是自定义了一个函数类型，使用切片进行存储，在注册路由的时候将中间件添加进去，当有请求进来时，先执行`gin.HandderFunc`函数，在`Gin`框架中使用一个切片存储，所以添加中间件的时候要注意顺序；



### 1.15 Go语言的错误处理

#### 1.15.1 panic与recover

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247484780&idx=1&sn=0468a1a4dc27c09732804798e5609def&chksm=c22b8330f55c0a2673e3c00ba6033fcf6e05eb6619b0b5ccea6eb461dfc957886b57bf660975&token=337827600&lang=zh_CN#rd

Go语言`panic`关键主要用于主动抛出异常，类似于`java`等语言中的`throw`关键字，`panic`能够改变程序的控制流，调用`panic`后会立刻停止执行当前函数的剩余代码，并在当前`goroutine`中递归执行调用方的`defer`；

Go语言中的`recover`关键字用于捕获异常，让程序回到正常状态，类似`java`等语言中的`try...catch`。`recover`中可以中止`panic`造成的程序崩溃，他只能在`defer`中发挥作用的函数，在其他域中调用不会发挥作用

panic与recover的特性：

1. panic允许在defer中嵌套多次调用，程序多次调用panic也不影响defer函数的正常执行，所以使用defer进行工作；
2. panic只会对当前goroutine有效；在newdefer中分配`_defer`结构体对象时，会把分配到对象链入当前`goroutine`的`_defer`链表的表头，也就是把延迟调用函数与调用方所在的goroutine进行关联，因此当程序发生`panic`时只会调用当前goroutine的延迟调用函数；

典型应用：

1. Gin框架在初始化`Engine`实例，调用`Default`方法把`recovery middleware`附上，`recovery`中使用了`defer`函数，通过`recover`来阻止`panic`，当发生panic时，会返回500错误码，这里有一个需要注意的点是只有主程序中的`panic`是会被自动`recover`的,协程中出现`panic`会导致整个程序`crash`。还记得我们上面讲的第三个特性嘛，**一个协程会发生`panic`，导致程序崩溃，但是只会执行自己所在`Goroutine`的延迟函数，所以正好验证了多个 `Goroutine` 之间没有太多的关联，一个 `Goroutine` 在 `panic` 时也不应该执行其他 `Goroutine` 的延迟函数。** 

源代码：

1. 编译器会负责做转换关键字的工作
   1. 将panic与recover分别转换成runtime.gopanic和runtime.gorecover
   2. 将defer转换成runtime.deferproc函数
   3. 调用defer的函数末尾调用`runtime.deferreturn`函数
2. 在运行过程中遇到`runtime.gopanic`方法时，会从goroutine的链表依次取出`runtime._defer`结构体并执行
3. 如果调用延迟函数时遇到了`runtime.gorecover`会将`_panic.recovered`标记为true并返回`panic`的参数：
   1. 在这次调用结束后，`runtime.gopanic`会从`runtime._defer`结构体中取出程序计数器pc和栈指针`SP`并调`runtime.recovery`函数进行恢复程序；
   2. runtime.recovery会根据传入的pc和sp跳转回runtime.deferproc
   3. 编译器自动生成的代码会发现 `runtime.deferproc` 的返回值不为 0，这时会跳回 `runtime.deferreturn`并恢ss复到正常的执行流程；
4. 如果没有遇到`runtime.gorecvoer`就会依次遍历所有的`runtime._defer`，并在最后调用`runtime.fatalpanic`中止程序，打印`panic`的参数并返回错误码；



### 1.16 defer的实现原理

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247484778&idx=1&sn=7ceb16f634b3d479a8d5b0b8c4d50b27&chksm=c22b8336f55c0a20e1099d062a69c16436d3b7cbc9c3da6c43bab34fc2f029e0c48e0cb0f08c&token=337827600&lang=zh_CN#rd

Go语言提供了defer关键字用于函数返回之前调用，Go语言的defer用于资源的释放，他经常会用于关闭文件描述符、关闭数据库连接以及解锁资源；

特性：

1. defer的被调用采用先进后出的特性，越后面的defer表达式越先被调用；
2. defer将语句放入到栈时，也会将相关的值拷贝同时入栈
3.  先执行return为返回值赋值，然后执行defer，return携带返回值返回
   1. 匿名返回值，函数在返回时，首先函数返回时会自动创建一个返回变量假设为ret，函数返回时要将变量赋值给ret，然后检查函数中是否有defer存在，若有则执行defer中部分，最后返回ret
   2. 命名返回值，因为返回变量已经命名，所以可以直接执行defer，最后返回命名变量

源码流程：

1. 首先编译器会把defer语句翻译成对`deferproc`函数的调用
2. 然后deferproc函数会负责调用`newdefer`函数分配一个`_defer`结构体对象并放入当前的`goroutine`的`_defer`链表的表头
3. 然后编译起会在`defer`所在函数的结尾处插入对`deferreturn`的调用，`deferreturn`负责递归的调用某函数(defer语句所在函数)通过`defer`语句注册的函数。

https://draveness.me/golang/docs/p



### 1.17 Go语言如何退出协程

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247485757&idx=1&sn=def8f2c429fd18dde5beaab01664c36e&chksm=c22b8f61f55c0677a6cd9f4db4e217bcee0e111a00bea499c430ecfe3581d29d7fcf02cc1cad&token=337827600&lang=zh_CN#rd

Go语言提供了`runtime.Goexit()`，只要执行`goexit`这个函数，当前协程就会退出，同时还能调度下一个可执行的协程出来跑；

源代码如下：

```GO
func Goexit() {
    // 以下函数省略一些逻辑...
    gp := getg() 
    for {
    // 获取defer并执行
        d := gp._defer
        reflectcall(nil, unsafe.Pointer(d.fn), deferArgs(d), uint32(d.siz), uint32(d.siz))
    }
    goexit1()
}
 
func goexit1() {
    mcall(goexit0)
}
// goexit continuation on g0.
func goexit0(gp *g) {
  // 获取当前的 goroutine
    _g_ := getg()
    // 将当前goroutine的状态置为 _Gdead
    casgstatus(gp, _Grunning, _Gdead)
  // 全局协程数减一
    if isSystemGoroutine(gp, false) {
        atomic.Xadd(&sched.ngsys, -1)
    }
  
  // 省略各种清空逻辑...
 
  // 把g从m上摘下来。
  dropg()
 
 
    // 把这个g放回到p的本地协程队列里，放不下放全局协程队列。
    gfput(_g_.m.p.ptr(), gp)
 
  // 重新调度，拿下一个可运行的协程出来跑
    schedule()
}
```

**实际用途**

实际在每个协程的栈底都会插入`goexit`，因为在runtime/proc.go中有一个newproc1方法，只要是创建协程都会用这个方法；main协程也是通过newproc函数创建的，栈底也会有一个goexit;



### 1.18 interface

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247484775&idx=1&sn=156ebaa2acfa31c9efb316569307d627&chksm=c22b833bf55c0a2dad59c6b4519a1dcf45fab5abc53f6b0c28758d06f1168b3660011c09a803&token=337827600&lang=zh_CN#rd

Go语言的接口是一组方法的签名，他是Go语言的重要组成部分，interface是一组method签名的组合，我们通过interface来定义对象的一组行为，interface是一种类型，定义如下：

```go
type Person interface {
    Eat(food string) 
}
```

它的定义可以看出来用了type关键字，更准确的说interface是一种具有一组方法的类型，这些方法定义了interface的行为，golang接口定义不能包含变量，但是允许不带任何方法，这种类型的接口叫`empty interface`。

如果一个类型实现了一个interface中所有方法，我们就可以说该类型实现了`interface`，所以我们所有类型都实现了empty interface，因为任何一种类型至少实现了0个方法。

Go语言会根据接口类型是否包含一组方法将接口类型分成了两类：

- 使用`runtime.iface`结构表示包含方法的接口
- 使用`runtime.eface`结构表示包不包含任何方法的`interface{}`类型

runtime.eface结构如下：

```go
type eface struct { // 16 字节
 _type *_type
 data  unsafe.Pointer
}
```

这里只包含指向底层数据和类型的两个指针，从这个`type`我们也可以推断出Go语言的任意类型都可以转换成`interface`。

runtime.iface结构如下：

```go
type iface struct { // 16 字节
 tab  *itab
 data unsafe.Pointer
}
```

**空切片注意事项：**

一个`interface{}`类型的slice不能接收任何类型的slice，只能接受把元素一个一个的去替换；

**非空切片注意事项：**

如果实现了接收者是值类型的方法，会隐含地也实现了接收者是指针的方法；



#### 1.18.1 类型断言

一个interface类型可以被多种类型实现，有时候我们需要区分`interface`的变量究竟存储哪种类型的值，go可以使用`comma,ok`的形式做区分：value, ok := em.(T)：em是interface类型的变量，T代表要断言的类型，value是interface变量存储的值，ok是bool类型表示是否为该断言的类型T。



#### 1.18.2 interface类型的比较

接口比较的时候只有当两个变量的动态类型、动态值都相等的时候，才是相等；

**一个nil的interface类型 , 是包含下面俩的 , 动态类型和动态值**

![image-20221012102615906](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221012102615906.png)



### 1.19 内联函数

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247485010&idx=1&sn=37ef820fa051a8a196509d501b74961a&chksm=c22b800ef55c091818b84dfe681a3e05e8e74f60703b1deb62b7ae38fae8d6f65eae8427ab0c&token=337827600&lang=zh_CN#rd

C语言的朋友对内联函数不陌生，在C语言中一个`inline`关键字，使用`inline`修饰的函数就是内联函数；

示例：

```
#include <stdio.h>    
inline char* chooseParity(int a) 
{  
 return (i % 2 > 0) ? "奇" : "偶";  
}   
  
int main()  
{  
 int i = 0;  
 for (i=1; i < 100; i++) 
 {  
  printf("i:%d    奇偶性:%s /n", i, chooseParity(i));      
 }  
} 
```

这段代码中函数`char* chooseParity(int a)`使用`inline`进行修饰，那么这段代码在执行的时候就会变成这样：

```
int i = 0;  
for (i=1; i < 100; i++) 
{  
 printf("i:%d    奇偶性:%s /n", i, (i % 2 > 0) ? "奇" : "偶");      
}  
```

这样就避免了频繁调用函数对栈内存重复开辟所带来的消耗，我们都知道一些函数被频繁调用，会不断地有函数入栈，即**函数栈**，会造成栈空间或**栈内存**的大量消耗，内联函数的出现节省了每次调用函数带来的额外时间开支。但并不是所有场景都可以使用内联函数的，必须在程序占用空间和程序执行效率之间进行权衡，因为过多的比较复杂的函数进行内联扩展将带来很大的存储资源开支。

Go语言的编译器会对函数调用进行优化，他没有提供任何关键字可以手动声明内联函数，不过我们可以在函数上添加`//go:noinline`注释告诉编译不要对他进行内联优化；



### 1.20 大端小端

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247484789&idx=1&sn=d86fa1109e2cb5090c1b1925e7acbaf2&chksm=c22b8329f55c0a3f954fc3ce86246cb7ba1353563562f65047a3da6eab19b4140e78e3bc8653&token=337827600&lang=zh_CN#rd

大小端问题与计算机硬件或者软件的创造者们有关，不同的操作系统和不同的芯片类型都有所不同，所以才会有这个区分；

大端：高位字节存放在内存的低地址端，低位字节排放在内存的高地址端

小端：低位字节存放在内存的低地址端，高位字节存放在内存的高地址端

**如何用Go语言验证当前计算机是大端还是小端**

我们可以使用`int32`类型强制转换成byte类型，判断起始存储位置来实现：

```GO
package main

func isLittleEndian() bool{
    var value int32 = 1
    // bigEndian: 00 00 00 01
    // littleEndian: 01 00 00 00
    pointer := unsafe.Pointer(&value)
    pb := (*byte)(pointer)
    if *pb == 1{
        reture ture
    }
    return false
}
```

**应用**

gRPC，gRPC封装message时，在封装header时，特意制定了使用大端字节序；



**Go语言标准库 encoding/binary 就是大端、小端包**

### 1.20 调度器

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247485530&idx=1&sn=616e482a683ef5c2d08b24287cbd74e0&chksm=c22b8e06f55c0710ce93099f28ae149d52acf9f56b21ebe91a135689d3db61eda3bffca3928c&token=348125870&lang=zh_CN#rd

Go语言调度经过几个大版本的迭代：

- 单线程调度器 0.x：
  - 只包含40多行代码
  - 程序中只能存在一个活跃线程，由G-M模型组成
- 多线程调度器 1.0：
  - 允许运行多线程程序
  - 全局锁导致竞争加重
- 任务窃取调度器：1.1
  - 引入处理器P，构成了G-M-P模型
  - 在处理器P的基础上实现了基于工作窃取的调度器
  - 在某些情况下，Goroutine不会让出线程，进而造成饥饿问题
  - 时间过长的垃圾回收，STW，会导致程序长时间无法工作
- 抢占式调度：1.2 - 至今
  - 基于协作式的抢占式调度器 1.2 ~ 1.3
    - 通过编译器在函数调用时插入抢占检查指令，在函数调用时检查当前goroutine是否发起了抢占请求，实现基于写作的抢占式调度；
    - goroutine可能会因为垃圾回收和循环长时间占用资源导致程序暂停
  - 基于信号的抢占式调度 1.14 ~ 至今
    - 实现基于信号真抢占式调度
    - 垃圾回收在扫描栈时会触发抢占调度
    - 抢占的时间点不够多，还不能够股改全部的边缘情况
- 非均匀存储访问调度器：提案
  - 对运行时的各种资源进行分区
  - 实现非常复杂，到今天还没有提上日程

本文讲述了 Go 语言基于信号的异步抢占的全过程，一起来回顾下：

1. M 注册一个 SIGURG 信号的处理函数：sighandler。
2. sysmon 线程检测到执行时间过长的 goroutine、GC stw 时，会向相应的 M（或者说线程，每个线程对应一个 M）发送 SIGURG 信号。
3. 收到信号后，内核执行 sighandler 函数，通过 pushCall 插入 asyncPreempt 函数调用。
4. 回到当前 goroutine 执行 asyncPreempt 函数，通过 mcall 切到 g0 栈执行 gopreempt_m。
5. 将当前 goroutine 插入到全局可运行队列，M 则继续寻找其他 goroutine 来运行。
6. 被抢占的 goroutine 再次调度过来执行时，会继续原来的执行流。



#### 1.20.1 GMP模型

Go语言运行时调度器三个重要的组成部分：
M: 表示操作系统的线程，他由操作系统的调度器调度和管理，一个M就是一个线程，goroutine就是跑在M之上的，M是一个很大的结构，里面维护小对象内存cache；Go 语言会使用私有结构体 `runtime.m` 表示操作系统线程，其中 g0 是持有调度栈的 Goroutine，`curg` 是在当前线程上运行的用户 Goroutine，这也是操作系统线程唯一关心的两个 Goroutine，g0 是一个运行时中比较特殊的 Goroutine，它会深度参与运行时的调度过程，包括 Goroutine 的创建、大内存分配和 CGO 函数的执行。

G：goroutine，他是一个待执行的任务，他有自己的栈，用于调度

P：处理器P，它可以被看作运行在线程上的本地调度器，主要用途是来执行goroutine，他维护了一个goroutine队列，里面存储了所有需要他来执行的goroutine;调度器中的处理器 P 是线程和 Goroutine 的中间层，它能提供线程需要的上下文环境，也会负责调度线程上的等待队列，通过处理器 P 的调度，每一个内核线程都能够执行多个 Goroutine，它能在 Goroutine 进行一些 I/O 操作时及时让出计算资源，提高线程的利用率。

**Go语言有两个运行队列，其中一个是处理器本地运行的队列，另一个是调度器持有的全局队列，只有本地队列没有剩余空间时或者当M发生系统调用完成后的G没有找到空闲的P运行时会使用全局队列；处理器本地队列是一个使用数组构成的环形链表，他最多可以存储256个待执行任务。**

![image-20221012110621594](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221012110621594.png)

P的数量可以通过GOMAXPROCS()来设置，他其实也就代表了真正的并发度，即有多少个goroutine可以同时运行；

图中灰⾊的那些goroutine并没有运⾏，⽽是出于ready的就绪态，正在等待被调度。P维护着这个队列（称之为

runqueue），Go语⾔⾥，启动⼀个goroutine很容易：go function 就⾏，所以每有⼀个go语句被执⾏，

runqueue队列就在其末尾加⼊⼀个goroutine，在下⼀个调度点，就从runqueue中取出（如何决定取哪个

goroutine？）⼀个goroutine执⾏。

当⼀个OS线程M0陷⼊阻塞时，P转⽽在运⾏M1，图中的M1可能是正被创建，或者从线程缓存中取出。

当MO返回时，它必须尝试取得⼀个P来运⾏goroutine，⼀般情况下，它会从其他的OS线程那⾥拿⼀个P过来， 

![image-20221012111244208](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221012111244208.png)

如果没有拿到的话，它就把goroutine放在⼀个global runqueue⾥，然后⾃⼰睡眠（放⼊线程缓存⾥）。所有的P也会周期性的检查global runqueue并运⾏其中的goroutine，否则global runqueue上的goroutine永远⽆法执⾏。

另⼀种情况是P所分配的任务G很快就执⾏完了（分配不均），这就导致了这个处理器P很忙，但是其他的P还有任

务，此时如果global runqueue没有任务G了，那么P不得不从其他的P⾥拿⼀些G来执⾏。通常来说，如果P从其他的P那⾥要拿任务的话，⼀般就拿run queue的⼀半，这就确保了每个OS线程都能充分的使用；



#### 1.20.2 如何有效控制线程数

Go默认设置的最大线程数是10000个线程，这个值存在的主要目的是限制可以创建无限数量线程的Go程序：在程序把操作系统干爆之前，干掉程序；

因为在GMP模型中，P与M是一对一挂载的形式，通过设定GOAMXPROCS变量就能控制并行线程数，当M遇到同步系统调用时，G和M会与P剥离，当系统调用完成后，G重新进入可运行状态，而M就会被闲置起来；

Go也暴漏了debug.SetMaxThreads方法可以让我们修改最大线程数量；

Go语言没有对闲置线程做清除处理，他们被当作复用的资源，以备后续需要，但是如果在Go程序中积累大量空闲线程，这是对资源的一种浪费，同时对操作系统也产生了威胁，因此，Go设定了10000的默认线程数限制。

我们发现了一种利用 `LockOSThread` 函数的 trik 做法，可以借此做一些限制线程数的方案：例如启动定期排查线程数的 goroutine，当发现程序的线程数超过某阈值后，就回收掉一部分闲置线程。

当然，这个方法也存在隐患。例如在 issues#14592 有人提到：当子进程由一个带有 `PdeathSignal: SIGKILL` 的 A 线程创建，A 变为空闲时，如果 A 退出，那么子进程将会收到 KILL 信号，从而引起其他问题。

当然，绝大多数情况下，我们的 Go 程序并不会遇到空闲线程数过多的问题。如果真的存在线程数暴涨的问题，那么你应该思考代码逻辑是否合理（为什么你能允许短时间内如此多的系统同步调用），是否可以做一些例如限流之类的处理。而不是想着通过 `SetMaxThreads` 方法来处理。



#### 1.20.3 介绍一下goroutine

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247485253&idx=1&sn=bd63ffef075ee64987ad6b527243a45b&chksm=c22b8119f55c080fa676952208d05c3590e448e3efedcf376408533774fe3e0bf52571030370&token=348125870&lang=zh_CN#rd

在Go语言中，goroutine的创建成本很低，调度效率很高，Go语言在设计时就按以数万个goroutine为规范进行设计的，数十万并不意外，但是goroutine在内存占用方面确实具有有限的成本，你不能创建无限数量的他们，goroutine多到无法控制，就会引发内存泄漏；

goroutine就是G-P-M调度模型中的G，我们可以把goroutine看成是一种协程，创建goroutine也是有开销的，但是开销很小，初始只需要2-4k的栈空间，成是一种协程，创建`goroutine`也是有开销的，但是开销很小，初始只需要`2-4k`的栈空间，当`goroutine`数量越来越大时，同时存在的`goroutine`也越来越多时，程序就隐藏内存泄漏的问题。

Goroutine在运行时定义的状态非常多且复杂，但是我们可以将这些不同的状态聚合成三种：等待中、可运行、运行中，运行期间会在这三种状态来回切换：

- 等待中：Goroutine正在等待某些条件满足，例如：系统调用结束等；包括 `_Gwaiting`、`_Gsyscall` 和 `_Gpreempted` 几个状态；
- 可运行：Goroutine已经准备就绪，可以在线程运行，如果当前程序中有非常多的goroutine，每个goroutine就可能会等待更多的时间，即`grunnable`
- 运行中：Goroutine 正在某个线程上运行，即 `_Grunning`；



#### 1.20.4 为什么GM中要有P

在单线程、多线程调度器时代，Go语言使用的就是GM模型，除了`G`和`M`以外，还有一个**全局协程队列**，这个全局队列里放的是多个处于**可运行状态**的`G`。`M`如果想要获取`G`，就需要访问一个**全局队列**。同时，内核线程`M`是可以同时存在多个的，因此访问时还需要考虑**并发**安全问题。因此这个全局队列有一把**全局的大锁**，每次访问都需要去获取这把大锁。

并发量小的时候还好，当并发量大了，这把大锁，就成为了**性能瓶颈**。

所以需要引入一个中间来解决这个问题，在原有的GM模型基础上加入了一个调度器P，可以简单理解为在GM中间加了个中间层。

P的加入，还带来了一个本地协程队列，用于存放G，想要获取等待运行的`G`，会**优先**从本地队列里拿，访问本地队列无需加锁。而全局协程队列依然是存在的，但是功能被弱化，不到**万不得已**是不会去全局队列里拿`G`的。

`GM`模型里M想要运行`G`，直接去全局队列里拿就行了；`GMP`模型里，`M`想要运行`G`，就得先获取`P`，然后从 `P` 的本地队列获取 `G`。

新建 `G` 时，新`G`会优先加入到 `P` 的本地队列；如果本地队列满了，则会把本地队列中一半的 `G` 移动到全局队列。

`P` 的本地队列为空时，就从全局队列里去取。

如果全局队列为空时，`M` 会从其他 `P` 的本地队列**偷（stealing）一半G**放到自己 `P` 的本地队列。

`M` 运行 `G`，`G` 执行之后，`M` 会从 `P` 获取下一个 `G`，不断重复下去。



#### 1.20.5 GMP调度流程

![image-20221014214504215](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221014214504215.png)

- 每个P都有自己的本地队列，局部队列保存待执行的goroutine，当M绑定的P的局部队列已经满了之后会把goroutine放到全局队列；
- 每个P和M绑定，M是真正的执行Pgoroutine的实体，M从绑定的P中的局部队列获取G来执行；
- 当M绑定的P的局部队列为空时，M会从全局队列获取到本地队列来执行
- G，当从全局队列中没有获取到可执行G的时候，M会从其他P的局部队列中偷取G来执行，这种从其他P偷的方式称为work sealing.
- 当G因系统调用阻塞时会阻塞M，此时P会和M解绑即handoff，并寻找新的idle的M，若没有idle的M就会新创建一个M
- 当G因channel或者network I/O阻塞时，不会阻塞M，M会寻找其他runnable的G，当阻塞的G恢复后会重新进入runnable进入P队列等待执行。



#### 1.20.6 GMP中的work stealing机制

`GMP`高性能的关键，在于引入了 `work-stealing` 机制，如下图所示：

![image-20221014215629531](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221014215629531.png)



#### 1.20.7 **GMP** **中** **hand off** **机制**

当本线程 M因为 G进⾏的系统调⽤阻塞时，线程释放绑定的 P，把 P转移给其他空闲的 M'执⾏。当发⽣上下⽂切换时，需要对执⾏现场进⾏保护，以便下次被调度执⾏时进⾏现场恢复。Go调度器 M的栈保存在 G对象上，只需要将 M所需要的寄存器(SP、PC等)保存到 G对象上就可以实现现场保护。当这些寄存器数据被保护起来，就随时可以做上下⽂切换了，在中断之前把现场保存起来。如果此时 G任务还没有执⾏完，M可以将任务重新丢到 P的任务队列，等待下⼀次被调度执⾏。当再次被调度执⾏时，M通过访问 G的 vdsoSP、vdsoPC寄存器进⾏现场恢复(从上次中断位置继续执⾏)。





### 1.21 互斥锁

sync.mutex是一把互斥锁，具体是锁住限定区域的代码逻辑，这段区域只能被一个协程占用，当一个协程去获取锁时，获取不到会自旋一定次数后加入到队列去休眠，当锁被释放时，正在自旋的协程+休眠中的任意一个协程才能会被唤醒去抢锁，这是正常模式，还有饥饿模式，由于被唤醒的协程有很大的几率是抢不到正在自旋的协程，所以当有协程超过1毫秒获取不到锁时就会进入饥饿模式，进入后会根据上面提到的队列中按照先进先出的模式依次获取锁，新来的协程直接进入队列等待，当队列已经清空或者队列中头个协程等待时间小于1毫秒后退化为正常模式；

- 互斥锁有两种模式：正常模式、饥饿模式，饥饿模式的出现是为了优化正常模式下刚被唤起的`goroutine`与新创建的`goroutine`竞争时长时间获取不到锁，在`Go1.9`时引入饥饿模式，如果一个`goroutine`获取锁失败超过`1ms`,则会将`Mutex`切换为饥饿模式，如果一个`goroutine`获得了锁，并且他在等待队列队尾 或者 他等待小于`1ms`，则会将`Mutex`的模式切换回正常模式

- 加锁的过程：

- - 锁处于完全空闲状态，通过CAS直接加锁
  - 当锁处于正常模式、加锁状态下，并且符合自旋条件，则会尝试最多4次的自旋
  - 若当前`goroutine`不满足自旋条件时，计算当前goroutine的锁期望状态
  - 尝试使用CAS更新锁状态，若更新锁状态成功判断当前`goroutine`是否可以获取到锁，获取到锁直接退出即可，若获取不到锁则陷入睡眠，等待被唤醒
  - goroutine被唤醒后，如果锁处于饥饿模式，则直接拿到锁，否则重置自旋次数、标志唤醒位，重新走for循环自旋、获取锁逻辑；

- 解锁的过程

- - 原子操作mutexLocked，如果锁为完全空闲状态，直接解锁成功
  - 如果锁不是完全空闲状态，，那么进入`unlockedslow`逻辑
  - 如果解锁一个未上锁的锁直接panic，因为没加锁`mutexLocked`的值为0，解锁时进行mutexLocked - 1操作，这个操作会让整个互斥锁混乱，所以需要有这个判断
  - 如果锁处于饥饿模式直接唤醒等待队列队头的waiter
  - 如果锁处于正常模式下，没有等待的goroutine可以直接退出，如果锁已经处于锁定状态、唤醒状态、饥饿模式则可以直接退出，因为已经有被唤醒的 `goroutine` 获得了锁.

- 使用互斥锁时切记拷贝`Mutex`，因为拷贝`Mutex`时会连带状态一起拷贝，因为`Lock`时只有锁在完全空闲时才会获取锁成功，拷贝时连带状态一起拷贝后，会造成死锁

- TryLock的实现逻辑很简单，主要判断当前锁处于加锁状态、饥饿模式就会直接获取锁失败，尝试获取锁失败直接返回；



### 1.22 读写锁

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247486958&idx=1&sn=30d49d24405a183acabba37185bb0e00&chksm=c22b8bb2f55c02a4418233e15b0303918ab7c328a85e93c6483b964d91b7040c38a2d7f9a2a5&token=348125870&lang=zh_CN#rd

读写锁提供四种操作：读上锁、读解锁、写上锁、写解锁，加锁规则是读读共享，写写互斥，读写互斥，写读互斥

读写锁中的读锁是一定要存在的，其目的也是规避原子性问题，只有写锁没有读锁的情况下会导致我们读取到中间值；

Go语言的读写锁在设计上也避免了写锁饥饿的问题，通过字段`readCount`、`readWait`进行控制，当写锁的`goroutine`被阻塞时，后面进来向要获取读锁的`goroutine`都会被阻塞住，当写锁释放时，会将后面的读操作`goroutine`、写操作的`goroutine`都唤醒，剩下的交给他们竞争；

- 读锁获取锁流程：

- - 锁空闲时，读锁可以立马被获取
  - 如果当前有写锁正在阻塞，那么想要获取读锁的`goroutine`就会被休眠

- 释放读锁流程：

- - 当前没有异常场景或写锁阻塞等待出现的话，则直接释放读锁成功
  - 若没有加读锁就释放读锁则抛出异常；
  - 写锁被读锁阻塞等待的场景下，会将`readerWait`的值进行递减，`readerWait`表示阻塞写操作goroutine的读操作goroutine数量，当`readerWait`减到`0`时则可以唤醒被阻塞写操作的`goroutine`了；

- 写锁获取锁流程

- - 写锁复用了`mutex`互斥锁的能力，首先尝试获取互斥锁，获取互斥锁失败就会进入自旋/休眠；
  - 获取互斥锁成功并不代表写锁加锁成功，此时如果还有占用读锁的`goroutine`，那么就会阻塞住，否则就会加写锁成功

- 释放写锁流程

- - 释放写锁会将负值的`readerCount`变成正值，解除对读锁的互斥
  - 唤醒当前阻塞住的所有读锁
  - 释放互斥锁



### 1.23 map

https://qcrao.com/post/dive-into-go-map/

map实现主要有两种数据结构：哈希查找表、搜索树；

哈希查找表是用一个哈希函数将key分配到不同的桶，这样开销主要在哈希函数的计算以及数组的常数访问时间；

哈希查找表一般会存在碰撞问题，就是说不同的key被哈希到了同一个bucket，一般有两种应对方法：链表法和方法地址法，链表法将一个bucket实现成一个链表，落在同一个bucket中的key都会插入这个链表，开发地址法则是碰撞发生后，通过一定的规律，在数组的后面挑选空位，用来放置新的key。

搜索树法一般采用自平衡搜索树，包括：AVL 树，红黑树

自平衡搜索树法的最差搜索效率是 O(logN)，而哈希查找表最差是 O(N)。当然，哈希查找表的平均查找效率是 O(1)，如果哈希函数设计的很好，最坏的情况基本不会出现。还有一点，遍历自平衡搜索树，返回的 key 序列，一般会按照从小到大的顺序；而哈希查找表则是乱序的。

Go语言就是使用哈希表+链表法实现的；

**当map的key和value都不是指针，并且size都小于128字节的情况下，会把bmap标记为不含指针，这样可以避免gc时扫描整个hmap。bmap其实有一个overflow字段，是指针类型，破坏了bmap不含指针的设想，这时会把overflow移动到extra字段来**



**扩容**

使用哈希表的目的就是要快速查找到目标key，然而随着向map中添加的key越来越多，key发生碰撞的概率就越来越大，bucket中的8个cell会逐渐被塞满，查找、插入、删除key的效率也会越来越低，所以需要有一个度，通过一个指标来衡量前面描述的情况，这就是装在因子，loadFactor := count/ (2^B); count就是map的元素个数，2^B表示bucket数量，map扩容时机是向map中插入新的key时候，会进行条件检测，符合下面两个条件就会触发扩容：

1. 装载因子超过阈值，源码定义为6.5
2. overflow 的 bucket 数量过多：当 B 小于 15，也就是 bucket 总数 2^B 小于 2^15 时，如果 overflow 的 bucket 数量超过 2^B；当 B >= 15，也就是 bucket 总数 2^B 大于等于 2^15，如果 overflow 的 bucket 数量超过 2^15。

对于条件 1，元素太多，而 bucket 数量太少，很简单：将 B 加 1，bucket 最大数量（2^B）直接变成原来 bucket 数量的 2 倍。于是，就有新老 bucket 了。注意，这时候元素都在老 bucket 里，还没迁移到新的 bucket 来。而且，新 bucket 只是最大数量变为原来最大数量（2^B）的 2 倍（2^B * 2）。

对于条件 2，其实元素没那么多，但是 overflow bucket 数特别多，说明很多 bucket 都没装满。解决办法就是开辟一个新 bucket 空间，将老 bucket 中的元素移动到新 bucket，使得同一个 bucket 中的 key 排列地更紧密。这样，原来，在 overflow bucket 中的 key 可以移动到 bucket 中来。结果是节省空间，提高 bucket 利用率，map 的查找和插入效率自然就会提升。

对于条件 2 的解决方案，曹大的博客里还提出了一个极端的情况：如果插入 map 的 key 哈希都一样，就会落到同一个 bucket 里，超过 8 个就会产生 overflow bucket，结果也会造成 overflow bucket 数过多。移动元素其实解决不了问题，因为这时整个哈希表已经退化成了一个链表，操作效率变成了 `O(n)`。

采用渐进式扩容，插入或修改、删除key的时候，都会尝试进行搬迁buckets的工作，先检查 oldbuckets 是否搬迁完毕，具体来说就是检查 oldbuckets 是否为 nil，每次只搬迁最多两个bucket.



**为什么负载因子是6.5**

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247485464&idx=1&sn=e153db7eb8487d297e86f393cd0d4303&chksm=c22b8e44f55c0752cc4afbccf69a8cf67add74af487dcdcc1eb5c9fe7991389c2a3c3a3d4372&token=348125870&lang=zh_CN#rd

Go语言官方通过测试得出的结论，通过以下指标综合衡量：

1. 负载因子
2. overflow溢出率，也就是hash冲突的链表法
3. 每对key/value的开销字节数
4. hitprobe：查找一个存在的key时，要查找的平均个数
5. missprobe：查找一个不存在的key时，要查找的平均个数

负载因子：用于衡量当前哈希表中空间占用率的核心指标，也就是每个bucket桶存储的平均元素个数。



#### 1.23.1 map中的key符合什么要求

Golang中能用 == 号直接比较的数据类型有如下：

整型、浮点型、字符串、布尔型、复数型、指针类型、通道型、接口类型、数组型



不能直接比较的：

切片类型、键值对类型、函数型func



Golang中的map的key必须是可以比较的，要使key值不一样就需要进行比较，因此map中的key可以使用==进行比较。





### 1.24 sync.map

https://cdn.modb.pro/db/106052

Go 语言原生 map 并不是线程安全的，对它进行并发读写操作的时候，需要加锁。而 `sync.map`
则是一种并发安全的 map，在 Go 1.9 引入;

使用sync.map，对map的读写不需要加锁，并且它通过空间换时间的方式，使用read和dirty两个map来进行读写分离，降低锁时间来提高效率；

sync.map适合读多写少的场景，对于写多的场景，会导致read map缓存失效，需要加锁，导致冲突变多，而且由于未命中read map次数过多，导致dirty map提升为 read map，这是一个O(n)的操作，会进一步降低性能；

```go
type Map struct {
 mu Mutex
 read atomic.Value // readOnly
 dirty map[interface{}]*entry
 misses int
}
```

mu：保护read和dirty

read：是atomic.Value类型，可以并发地读，但如果需要更新read，则需要加锁保护，对于read中存储的entry字段，可能会被并发地CAS更新，但是如果需要更新一个之前已经被删除的entry，则需要状态从 expunged 改为 nil，再拷贝到 dirty 中，然后再更新。

dirty：是一个非线程安全的原始 map。包含新写入的 key，并且包含 `read`中的所有未被删除的 key。这样可以快速地将 `dirty`提升为 `read`对外提供服务。如果 `dirty`为 nil，那么下一次写入时，会新建一个新的 `dirty`，这个初始的 `dirty`是 `read`的一个拷贝，但除掉了其中已被删除的 key。

misses：每当从 read 中读取失败，都会将 `misses`的计数值加 1，当加到一定阈值以后，需要将 dirty 提升为 read，以期减少 miss 的情形。

注意到 read 和 dirty 里存储的东西都包含 `entry`
，来看一下：

```
type entry struct {
 p unsafe.Pointer // *interface{}
}
```

很简单，它是一个指针，指向 value。看来，read 和 dirty 各自维护一套 key，key 指向的都是同一个 value。也就是说，只要修改了这个 entry，对 read 和 dirty 都是可见的。这个指针的状态有三种：

p==nil：键值已经被删除，此时dirty==nil或m.dirty[k]指向该entry;

p==expunged：键值已经删除，此时，m.dirty!=nil且m.dirty不存在该键值

其他状态代表正常状态；

![image-20221012145601882](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221012145601882.png)

![image-20221012145639927](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221012145639927.png)

![image-20221012151534333](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221012151534333.png)

### 1.25 如何查看Go源码

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247486646&idx=1&sn=8ac0ff0d8d76efac8917eadb5842fa99&chksm=c22b8aeaf55c03fc54e18886178711a42663f7110fadc461eecc3cc58dcb8aa864c3c271cfc1&token=348125870&lang=zh_CN#rd



### 1.24 垃圾回收

https://mp.weixin.qq.com/s/AaHk-yg8D4atbO-zVAvhKQ

https://cloud.tencent.com/developer/article/2072909

**垃圾分为两种垃圾：**

语义垃圾：有些场景也被称为内存泄漏，语义垃圾指的是从语法上可达（可以通过局部、全局变量被引用）的对象，但从语义上来将他们是垃圾，垃圾回收器对此无能为力；

语法垃圾：从语法上无法到达的对象，这个是垃圾回收器的目标；



垃圾回收是一种自动管理内存的机制，垃圾回收器会去尝试回收程序不再使用的对象已经占用的内存；

像C语言需要手动管理内存，管错或者管漏内存也很糟糕，会直接导致程序不稳定（持续泄漏）甚至崩溃；

所以现在编程语言大多都有GC，Go语言也实现了GC；

Go语言GC的触发场景分为两大类：分别是：

1. 系统触发，运行时根据内置的条件，检查、发现到，则进行GC处理，维护整个应用程序的可用性；系统触发的三个条件：
   1. gcTriggerHeap：当所分配的堆大小达到阈值（由控制器计算的触发堆的大小）时，将会触发；
   2. gcTriggerTime：当距离上一个GC周期的时间超过一定时间时，将会触发。时间周期以runtime.forcegcperiod变量为准，默认2分钟
   3. gcTriggerCycle：如果没有开启GC，则启动GC
2. 手动触发：开发者在业务代码中自行调用runtime.GC方法来触发GC行为

**GC基本流程：**

![image-20221017090359848](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221017090359848.png)





1. 在开始新的一轮GC周期前，需要调用gcWaitOnMark方法上一轮GC的标记结束（含扫描中止、标记、或标记终止等）
2. 开始新的一轮GC周期，调用gStart方法触发GC行为，开始扫描标记阶段
3. 需要调用gcWaitOnMark方法等待，直到当前GC周期的扫描、标记、标记终止完成；
4. 需要调用sweepone方法，扫描未切除的堆跨度，并持续扫除，保证清理完成，保证清理完成，在等待扫除完毕前的阻塞时间，会调用Gosched让出
5. 在本轮 GC 已经基本完成后，会调用 `mProf_PostSweep` 方法。以此记录最后一次标记终止时的堆配置文件快照。
6. 结束，释放M

**GC在哪触发**

监控线程

实质上在Go运行时（runtime）初始化时，会启动一个goroutine，用于处理GC机制的相关事项；处理完毕后，会调用goparkunlock方法让该goroutine陷入休眠等待状态，以减少不必要的资源开源；在休眠的时候会由sysmon这一个系统监控线程来进行监控、唤醒等行为；sysmon中不断的进行for循环判断`gcTriggerTime` 和 `now` 变量进行比较，判断是否达到一定的时间（默认为 2 分钟）。

若达到意味着满足条件，会将 `forcegc.g` 放到全局队列中接受新的一轮调度，再进行对上面 `forcegchelper` 的唤醒。

堆内存申请

在了解定时触发的机制后，另外一个场景就是分配的堆空间的时候，运行时申请堆内存的mallocgc方法：

小对象：如果申请小对象时，发现当前内存空间不存在空闲跨度时，就会需要调用nextFree方法获取新的可用对象，可能会触发GC行为

大对象：如果申请大于32K以上的大对象时，可能会触发GC行为；

**Go语言的垃圾回收**

在1.3 之前主要使用标记清除算法，分为两个步骤：

1. 暂停程序业务逻辑，找出不可达的对象，然后做标记；
2. 清除未被标记的对象
3. 程序暂停取消，周期重复该操作

在1.5以后开始引入三色标记法，这里的三色，对应了垃圾回收过程中对象的三种状态：

- 灰色：对象还在标记队列中等待
- 黑色：对象已经被标记，该对象不会再本次GC中被清理
- 白色：对象未被标记，该对象将会在本次GC中被清理

1. 初始状态所有对象都是白色
2. 从根节点开始遍历所有对象，把遍历到的对象变成灰色对象
3. 遍历灰色对象，将灰色对象引用的对象也变成灰色对象，然后将遍历过的灰色对象变成黑色对象
4. 循环步骤三，直到灰色对象全部变成黑色
5. 通过写屏障检测对象是否有变化，重复以上操作（因为mark和用户程序是并行的，所以在上一步执行的时候可能会有新的对象分配，写屏障是为了解决这个问题引入的）
6. 收集所有白色对象



**对象丢失问题**

GC线程/协程与应用线程/协程是并发执行的，在GC标记worker工作期间，应用还会不断地修改堆上对象引用的关系，所以需要解决漏标、错标的问题，我们先定义"三色不变性"，如果我们堆上对象的引用关系不管怎么修改，都能满足三色不变性，那么也不会发生对象丢失问题；

强三色不变性就是禁止黑色对象指向白色对象；

弱三色不变性就是黑色对象可以指向白色对象，但指向的白色对象，必须有能从灰色对象可达的路径；

无论应用于GC并发执行期间如何修改堆上对象的关系，只要修改之后，堆上对象能满足任意一种不变性，就不会发生对象丢失的问题；

实现强/弱三色不变性均需要引入屏障技术，Go语言使用写屏障；

Go语言只有write barrier，没有read barrier，在应用进入GC标记阶段前的STW阶段，会将全局变量runtime.writeBarrier.enabled修改为true，这时堆上指针修改操作在修改之前都会调用gcWritebarrier；

常见的write barrier有两种：

- Dijistra Insertion Barrier：指针修改时，指向的新对象要标灰（插入屏障）
- Yuasa Deletion Barrier：指针修改时，修改前指向的对象要标灰（删除屏障）

Go语言混合了两种屏障，官方选择了更为简单的实现，即指针断开的老对象和新对象都标灰的实现；





**写屏障优化**

STW的目的是防止GC扫描时内存变化而停掉goroutine，而写屏障就是让goroutine与GC同时运行的手段，虽然写屏障不能完全消除STW，但是可以大大减少STW的时间，写屏障类似一种开关，在GC的特定时机开启，开启后指针传递时会把指针标记，即本轮不回收，下次 GC 时再确定。GC 过程中新分配的内存会被立即标记，用的并不是写屏障技术，也即GC过程中分配的内存不会在本轮GC中回收。

**辅助GC**

为了防止内存分配过快，在 GC 执行过程中，如果 goroutine 需要分配内存，那么这个 goroutine 会参与一部分GC的工作，即帮助 GC 做一部分工作，这个机制叫作 Mutator Assist。



**回收流程**

相比复杂的标记流程，对象的回收和内存释放进程启动时会有两个特殊goroutine：

- 一个叫sweep.g，主要负责清扫死对象，合并相关的空闲页
- 一个叫scvg.g，主要负责向操作系统归还内存；



### 1.25 Go语言数据类型的并发安全

并发安全就是程序在并发的情况下执行的结果都是正确的；

Go中数据类型分为两大类：

基本数据类型：字节型、整型、布尔型、浮点型、复数型、字符串

符合数据类型：数组、切片、指针、结构体、字典、通道、函数、接口

**字节型、布尔型、整型、浮点型**取决于操作系统指令值，在64位的指令集架构中可以由一条机器指令完成，不存在被细分为更小的操作单位，所以这些类型的并发赋值是安全的，但是这个也跟操作系统的位数有关，比如int64在32位操作系统中，它的高32位和低32位是分开赋值的，此时是非并发安全的。

复数类型、字符串、结构体、数组，切片，字典，通道，接口， 这些底层都是struct，不同成员的赋值都不是一起的，所以都不是并发安全的。



### 1.26 系统监控

很多系统中都有守护进程，他们能够在后台监控系统的运行状态，再出现意外情况时及时响应。系统监控线程是Go语言运行时重要的组成部分，他会每隔一段时间检查GO语言运行时，确保程序没有进入异常状态。

在支持多任务的操作系统中，守护进程是在后台运行的计算机程序，它不会由用户直接操作，它一般会在操作系统启动时自动运行。

守护进程是很有效的设计，它在整个系统的生命周期中都会存在，会随着系统的启动而启动，系统的结束而结束。在操作系统和 Kubernetes 中，我们经常会将数据库服务、日志服务以及监控服务等进程作为守护进程运行。

Go 语言的系统监控也起到了很重要的作用，它在内部启动了一个不会中止的循环，在循环的内部会轮询网络、抢占长期运行或者处于系统调用的 Goroutine 以及触发垃圾回收，通过这些行为，它能够让系统的运行状态变得更健康。

```GO
func main() {
	...
	if GOARCH != "wasm" {
		systemstack(func() {
			newm(sysmon, nil)
		})
	}
	...
}
func sysmon() {
	sched.nmsys++
	checkdead()

	lasttrace := int64(0)
	idle := 0
	delay := uint32(0)
	for {
		if idle == 0 {
			delay = 20
		} else if idle > 50 {
			delay *= 2
		}
		if delay > 10*1000 {
			delay = 10 * 1000
		}
		usleep(delay)
		...
	}
}
```

当运行时刚刚调用上述函数时，会先通过 [`runtime.checkdead`](https://draveness.me/golang/tree/runtime.checkdead) 检查是否存在死锁，然后进入核心的监控循环；系统监控在每次循环开始时都会通过 `usleep` 挂起当前线程，该函数的参数是微秒，运行时会遵循以下的规则决定休眠时间：

- 初始的休眠时间是 20μs；
- 最长的休眠时间是 10ms；
- 当系统监控在 50 个循环中都没有唤醒 Goroutine 时，休眠时间在每个循环都会倍增；

当程序趋于稳定之后，系统监控的触发时间就会稳定在 10ms。它除了会检查死锁之外，还会在循环中完成以下的工作：

- 运行计时器 — 获取下一个需要被触发的计时器；
- 轮询网络 — 获取需要处理的到期文件描述符；
- 抢占处理器 — 抢占运行时间较长的或者处于系统调用的 Goroutine；
- 垃圾回收 — 在满足条件时触发垃圾收集回收内存；



### 1.27 内存管理

内存管理的三个主要参与者：mutator、allocator、garbge collector;

mutartor：指的是我们的应用，即application，我们将堆上的对象看作一个图，跳出应用来看的话，应用的代码就是在不停地修改这张堆对象图的指向关系。

allocator：内存分配器，应用需要内存的时候都要向allocator申请，allocator要维护好内存分配的数据结构，多线程分配器要考虑高并发场景下锁的影响，并针对性地进行设计以降低锁冲突；

![image-20221016223613846](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221016223613846.png)

**分配内存**

应用程序使用 mmap 向 OS 申请内存，操作系统提供的接口较为简单，mmap 返回的结果是连续的内存区域。

mutator 申请内存是以应用视角来看问题，我需要的是某一个 struct，某一个 slice 对应的内存，这与从操作系统中获取内存的接口之间还有一个鸿沟。需要由 allocator 进行映射与转换，将以“块”来看待的内存与以“对象”来看待的内存进行映射：

![image-20221016224837007](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221016224837007.png)

应用代码中的对象与内存间怎么做映射？

在现代 CPU 上，我们还要考虑内存分配本身的效率问题，应用执行期间小对象会不断地生成与销毁，如果每一次对象的分配与释放都需要与操作系统交互，那么成本是很高的。这需要在应用层设计好内存分配的多级缓存，尽量减少小对象高频创建与销毁时的锁竞争，这个问题在传统的 C/C++ 语言中已经有了解法，那就是 tcmalloc：

tcmalloc 的全局图

Go 语言的内存分配器基本是 tcmalloc 的 1:1 搬运。。毕竟都是 Google 的项目。

在 Go 语言中，根据对象中是否有指针以及对象的大小，将内存分配过程分为三类：

- tiny ：size < 16 bytes && has no pointer(noscan)
- small ：has pointer(scan) || (size >= 16 bytes && size <= 32 KB)
- large ：size > 32 KB

在内存分配过程中，最复杂的就是 tiny 类型的分配。

我们可以将内存分配的路径与 CPU 的多级缓存作类比，这里 mcache 内部的 tiny 可以类比为 L1 cache，而 alloc 数组中的元素可以类比为 L2 cache，全局的 mheap.mcentral 结构为 L3 cache，mheap.arena 是 L4，L4 是以页为单位将内存向下派发的，由 pageAlloc 来管理 arena 中的空闲内存。

| L1          | L2             | L3            | L4           |
| :---------- | :------------- | :------------ | :----------- |
| mcache.tiny | mcache.alloc[] | mheap.central | mheap.arenas |

若 L4 也没法满足我们的内存分配需求，便需要向操作系统去要内存了。

和 tiny 的四级分配路径相比，small 类型的内存没有本地的 mcache.tiny 缓存，其余与 tiny 分配路径完全一致。

| L1             | L2            | L3           |
| :------------- | :------------ | :----------- |
| mcache.alloc[] | mheap.central | mheap.arenas |

large 内存分配稍微特殊一些，没有上面复杂的缓存流程，而是直接从 mheap.arenas 中要内存，直接走 pageAlloc 页分配器。

页分配器在 Go 语言中迭代了多个版本，从简单的 freelist 结构，到 treap 结构，再到现在最新版本的 radix 结构，其查找时间复杂度也从 O(N) -> O(log(n)) -> O(1)。

在当前版本中，只需要常数时间复杂度就可以确定空闲页组成的 radix tree 是否能够满足内存分配需求。若不满足，则要对 arena 继续进行切分，或向操作系统申请更多的 arena。

**内存分配的数据结构之间的关系**

arenas 以 64MB 为单位，arenas 会被切分成以 8KB 为单位的 page，一个或多个 page 可以组成一个 mspan，每个 mspan 可以按照 sizeclass 划分成多个 element。

如下图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Lq0XA1b7xbdW8PVEJ7ldZOUK4V1VtuWPsdiaAmiaouic5aldkAoCeWx0oNaEU9ic6SOcibX2QnJZqqeWaBXmHbuw00g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

各种内存分配结构之间的关系，图上省略了页分配器的结构

每一个 mspan 都有一个 allocBits 结构，从 mspan 里分配 element 时，只要将 mspan 中对应该 element 位置的 bit 位置一即可。其实就是将 mspan 对应的 allocBits 中的对应 bit 位置一，在代码中有一些优化，我们就不细说了。



### 1.28 什么是反射

`reflect`实现了运行时的反射能力，能够让程序操作不同类型的对象[1](https://draveness.me/golang/docs/part2-foundation/ch04-basic/golang-reflect/#fn:1)。反射包中有两对非常重要的函数和类型，两个函数分别是：

- `reflect.TypeOf`能获取类型信息；
- `reflect.ValueOf`能获取数据的运行时表示；

反射的三大法则：

运行时反射是程序在运行期间检查其自身结构的一种方式。反射带来的灵活性是一把双刃剑，反射作为一种元编程方式可以减少重复代码，但是过量的使用反射会使我们的程序逻辑变得难以理解并且运行缓慢。我们在这一节中会介绍 Go 语言反射的三大法则，其中包括：

1. 从interface{} 变量可以反射出反射对象
2. 从反射对象可以获取interface{} 变量
3. 要修改反射对象，其值必须可设置



### 1.29 Goroutine过多为什么多导致内存泄漏

1. Goroutine本身的堆栈大小是2KB，我们开启一个新的Goroutine，至少会占用2KB的内存大小，当长时间的累积，数量较大时，就会导致占用内存越来越大
2. Goroutine长时间占用线程，就会导致栈内存不会被释放，导致CPU使用率升高
3. goroutine中的变量若指向了堆内存，那么该goroutine未被销毁，系统会认为该部分内存还不能被垃圾回收，那么就会占用大量的堆内存空间。




## 2. kafka八股文

### 2.1简述kakfa概念

kafka是一个基于发布订阅模式的消息队列中间件，分为producer和consumer，通过topic进行消息分类，producer生产的消息可以根据消息类型发布到不同的topic，由不同的consumer订阅进行消费，这样就组成了一个基本可用的消息队列中间件；

一个可靠的消息队列中间件，就要解决单点问题，kafka中解决单点问题引入了broker，一个kafka实例就叫做一个broker，单机、多机部署多个broker成为一个kafka集群，每个topic把自己复制到所在broker以外的broker，复制出来的topic叫做副本replicate，多份一样的topic就可以解决单点问题。为了管理这些topic引入了zk（以topic维度主从），选举leader、follower，leader会向follower复制topic数据，leader会和consumer打交道，而follower仅仅做备份，并且当一台broker挂掉后会重新选举新的leader（这时因为已经同步好topic就可以马上接力成为leader）

当一个topic数据非常多时，会严重影响性能及吞吐量，所以kafka会进行分区，一个topic可以把消息内容拆分多个平行的子topic提供给到外面进行消费，topic分区后，consumer也可以进行拆分成和分区一样的数量去消费，也就是消费组；



### 2.2kafka为什么这么快

1. 写入：顺序写、零拷贝（mmap+page cache）

kafka是基于磁盘对数据进行存储，但是的磁盘的一次IO要先让磁盘砖头找打对应的磁道以及扇区，但是如果顺序写就能节省掉这些步骤，提高写入效率。

写入时零拷贝是基于mmap实现的，也就是通过系统调用可以将磁盘文件和物理内存进行映射，在写入内存时相当写入了内存页缓存，等待刷入硬盘，省去了从用户态拷贝到内核态等一系列开销，和mysql写入日志文件的操作类似，也有配置设置是否flush同步还是异步写入。

2. 读取：零拷贝（sendfile）

在读取下零拷贝是基于sendFlile实现的，通过sendFile系统调用后，就不用走用户态，而是直接从内核态缓存区走到sokcet缓冲区；

3. 网络传输：reactor网络模型，压缩传输内容

kafka的网络传输模型使用的reator模型，重要的组件有三个acceptor、processor、worker线程，以及两个队列。客户端发起请求时，会先与acceptor建立连接，之后发请求进来后经过acceptor转发给processor、processor会把请求进行封装，放进请求队列里面，等待worker线程取出来进行业务逻辑处理，之后放回响应队列，等待processor捞回去返回给客户端，不经过acceptor。

reator模型的好处是采用了生产者消费者模型，将网络线程和工作线程分开，等到两端有瓶颈时可以调整配置参数进行升级和降级。

4. 分区设计提升吞吐量：基于发布订阅模式下topic可以分多个partition增加消费吞吐量；



### 2.3 kafka重复消费

1. Kafka Broker上存储的消息，都有一个offset标记，然后kafka的消费者是通过offset标记来维护当前已经消费的消息，每消费一批数据，kafka broker就会更新offset的值，避免重复消费，默认情况下，消费消费完以后，会自动提交offset的值，kafka消费端的自动提交逻辑有一个默认的5秒间隔，也就是说在5秒之后的下一次向broker拉取消息的时候提交，所以在Consumer消费的过程中，应用程序被强制kill掉或者宕机，可能会导致offset没提交，从而产生重复提交的问题；
2. 在kafka里面有一个partition balance机制，就是把多个partition均衡的分配给多个消费者，Consumer端会从分配的partition里面去消费信息，如果consumer在默认的5分钟内没办法处理完这一批消息，就会触发kafka的rebalance机制，从而导致offset自动提交失败；而在重新Rebalance之后，Consumer还是会从之前没提交的Offset位置开始消费，也会导致消息重复消费的问题。

基于这样的背景下，我认为解决重复消费消息问题的方法有几个：

1. 提高消费端的处理性能避免触发balance，比如可以用异步的方式来处理消息，缩短单个消息消费的时间，或者可以调整消息处理的超时时间，或者减少一次性从broker上拉取数据的条数；
2. 可以针对消息生成md5然后保存到mysql或者redis里面，在处理消息之前先去判断幂等性，或者使用业务判断幂等性；

### 2.4 kafka的延迟队列

![image-20221019201902075](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221019201902075.png)

延迟队列实现：Kafka支持延时生产、延时拉取、延时删除等，其基于时间轮和JDK的DelayQueue实现

a、时间轮(TimingWheel)：是一个存储定时任务的环形队列，底层采用数组实现，数组中的每个元素可以存放一个定时任务列表

 b、定时任务列表(TimerTaskList)：是一个环形的双向链表，链表中的每一项表示的都是定时任务项
 c、定时任务项(TimerTaskEntry)：封装了真正的定时任务TimerTask
 d、层级时间轮：当任务的到期时间超过了当前时间轮所表示的时间范围时，就会尝试添加到上层时间轮中，类似于钟表就是一个三级时间轮
 e、JDK DelayQueue：存储TimerTaskList，并根据其expiration来推进时间轮的时间，每推进一次除执行相应任务列表外，层级时间轮也会进行相应调整

3）缺点：
    a、延迟精度取决于时间格设置
    b、延迟任务除由超时触发还可能被外部事件触发而执行



### 2.5 kafka如何保证数据不丢失

我们无法保证kafka消息不丢失，只能保证某种程度下，消息不丢失；

**生产者**

对生产者来说，其发送消息到 Kafka 服务器的过程可能会发生网络波动，导致消息丢失。对于这一个可能存在的风险，我们可以通过合理设置 Kafka 客户端的 `request.required.acks` 参数来避免消息丢失。该参数表示生产者需要接收来自服务端的 ack 确认，当收不到确认或者超时，便会抛出异常，从而让生产者可以进一步进行处理。

该参数可以设置不同级别的可靠性，从而满足不同业务的需求，其参数设置及含义如下所示：

- **request.required.acks = 0** 表示 Producer 不等待来自 Leader 的 ACK 确认，直接发送下一条消息。在这种情况下，如果 Leader 分片所在服务器发生宕机，那么这些已经发送的数据会丢失。
- **request.required.acks = 1** 表示 Producer 等待来自 Leader 的 ACK 确认，当收到确认后才发送下一条消息。在这种情况下，消息一定会被写入到 Leader 服务器，但并不保证 Follow 节点已经同步完成。所以如果在消息已经被写入 Leader 分片，但是还未同步到 Follower 节点，此时 Leader 分片所在服务器宕机了，那么这条消息也就丢失了，无法被消费到。
- **request.required.acks = -1** 表示 Producer 等待来自 Leader 和所有 Follower 的 ACK 确认之后，才发送下一条消息。在这种情况下，除非 Leader 节点和所有 Follower 节点都宕机了，否则不会发生消息的丢失。

如上所示，如果业务对可靠性要求很高，那么可以将 `request.required.acks` 参数设置为 -1，这样就不会在生产者阶段发生消息丢失的问题。

**kafka服务器**

当 Kafka 服务器接收到消息后，其并不直接写入磁盘，而是先写入内存中。随后，Kafka 服务端会根据不同设置参数，选择不同的刷盘过程，这里有两个参数控制着这个刷盘过程：

```text
# 数据达到多少条就将消息刷到磁盘
#log.flush.interval.messages=10000
# 多久将累积的消息刷到磁盘，任何一个达到指定值就触发写入
#log.flush.interval.ms=1000
```

如果我们设置 log.flush.interval.messages=1，那么每次来一条消息，就会刷一次磁盘。**通过这种方式，就可以降低消息丢失的概率，这种情况我们称之为[同步刷盘](https://www.zhihu.com/search?q=同步刷盘&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2659956427})。** 反之，我们称之为[异步刷盘](https://www.zhihu.com/search?q=异步刷盘&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2659956427})。与此同时，Kafka 服务器也会进行副本的复制，该 Partition 的 Follower 会从 Leader 节点拉取数据进行保存。然后将数据存储到 Partition 的 Follower 节点中。

对于 Kafka 服务端来说，其会根据生产者所设置的 `request.required.acks` 参数，选择什么时候回复 ack 给生产者。对于 acks 为 0 的情况，表示不等待 Kafka 服务端 Leader 节点的确认。对于 acks 为 1 的情况，表示等待 Kafka 服务端 Leader 节点的确认。对于 acks 为 1 的情况，表示等待 Kafka 服务端 Leader 节点好 Follow 节点的确认。

但要注意的是，Kafka 服务端返回确认之后，仅仅表示该消息已经写入到 Kafka 服务器的 PageCache 中，并不代表其已经写入磁盘了。**这时候如果 Kafka 所在服务器断电或宕机，那么消息也是丢失了。而如果只是 Kafka 服务崩溃，那么消息并不会丢失。**

因此，对于 Kafka 服务端来说，即使你设置了每次刷 1 条消息，也是有可能发生消息丢失的，只是消息丢失的概率大大降低了。

**消费者**

在consumer消费阶段，对offset的处理，关系到是否丢失数据，是否重复消费数据，因此，我们把处理好offset就可以做到exactly-once && at-least-once(只消费一次)数据。

 **当enable.auto.commit=true时**

表示由kafka的consumer端自动提交offset,当你在pull(拉取)30条数据，在处理到第20条时自动提交了offset,但是在处理21条的时候出现了异常，当你再次pull数据时，由于之前是自动提交的offset，所以是从30条之后开始拉取数据，这也就意味着21-30条的数据发生了丢失。

 **当enable.auto.commit=false时**

由于上面的情况可知自动提交offset时，如果处理数据失败就会发生数据丢失的情况。那我们设置成手动提交。

当设置成false时，由于是手动提交的，可以处理一条提交一条，也可以处理一批，提交一批，由于consumer在[消费数据](https://www.zhihu.com/search?q=消费数据&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2580252586})时是按一个batch来的，当pull了30条数据时，如果我们处理一条，提交一个offset，这样会严重影响消费的能力，那就需要我们来按一批来处理，或者设置一个累加器，处理一条加1，如果在处理数据时发生了异常，那就把当前处理失败的offset进行提交(放在[finally代码块](https://www.zhihu.com/search?q=finally代码块&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2580252586})中)注意一定要确保offset的正确性，当下次再次消费的时候就可以从提交的offset处进行再次消费。



### 2.6 kafka是怎么实现exactly once

常见处理消息的承诺：

1. 最多一次（at most once）：消息可能会丢失，但绝不会被重复发送
2. 至少一次（at least once）：消息不会丢失，但有可能重复发送
3. 精确一次（exactly once）：消息不会丢失，也不会被重复发送

kafka默认提供的交付可靠性保障是第二种，即至少一次；

kafka分别通过幂等性 和 事务 这两种机制实现了精确一次语义（exactly once）；

幂等性：

在kafka中，producer默认不是幂等性的，但我们可以创建幂等性producer，他其实是0.11.0.0 版本引入的新功能，指定producer幂等性的方法简单，

为了实现生产者的幂等性，Kafka引入了 Producer ID（PID）和 Sequence Number的概念。

 PID：每个Producer在初始化时，都会分配一个唯一的PID，作为每个Producer会话的唯一标识，这个PID对用户来说，是透明的。
 Sequence Number：针对每个生产者（对应PID）发送到指定主题分区的消息都对应一个从0开始递增的Sequence Number。

首先，它只能保证单分区上的幂等性。即一个幂等性 Producer 能够保证某个主题的一个分区上不出现重复消息，它无法实现多个分区的幂等性。因为 SequenceNumber 是以 Topic + Partition 为单位单调递增的，如果一条消息被发送到了多个分区必然会分配到不同的 SequenceNumber ,导致重复问题
**他只能实现单会话上的幂等性，不能实现跨会话的幂等性，当你重启producer进程之后，这种幂等性保证就丧失了，重启producer后会分配一个新的producerID，相当于之前保存的sequenceNumber就丢失了。**

事务性：

上述幂等设计只能保证单个 Producer 对于同一个 <Topic, Partition> 的 Exactly Once 语义。

Kafka 现在通过新的事务 API 支持跨分区原子写入。这将允许一个生产者发送一批到不同分区的消息，这些消息要么全部对任何一个消费者可见，要么对任何一个消费者都不可见。这个特性也允许在一个事务中处理消费数据和提交消费偏移量，从而实现端到端的精确一次语义。

为了实现这种效果，应用程序必须提供一个稳定的（重启后不变）唯一的 ID，也即Transaction ID 。 Transactin ID 与 PID 可能一一对应。区别在于 Transaction ID 由用户提供，将生产者的 transactional.id 配置项设置为某个唯一ID。而 PID 是内部的实现对用户透明。

另外，为了保证新的 Producer 启动后，旧的具有相同 Transaction ID 的 Producer 失效，每次 Producer 通过 Transaction ID 拿到 PID 的同时，还会获取一个单调递增的 epoch。由于旧的 Producer 的 epoch 比新 Producer 的 epoch 小，Kafka 可以很容易识别出该 Producer 是老的 Producer 并拒绝其请求。



### 2.7 kakfa 发送指定分区

kafka发送分区设置：

- 如果在发送消息的时候指定了分区，则消息投递到指定分区
- 如果没有指定分区，但是消息的key不为空，则基于key的哈希值来选择一个分区
- 如果既没有指定分区，且消息的key也是空，则用轮询的方式选择一个分区；

kakfa的分区作用是提供负载均衡的能力，实现系统的高伸缩性。分区之后，不同的分区能够放在不同的物理设备上，而数据的读写操作也都是针对分区去进行，这样就可以使用每个分区都可以独立的处理自己分区的读写请求，而且，我们还可以通过添加新的节点机器来提高整个系统的吞吐量。

**分区策略：**

轮询策略：round-robin策略，顺序分配；

随机策略：随机地将消息放置到任意一个分区上，；

key-ordering：kafka允许为每条消息定义消息键，简称key，他是一个有着明确业务含义的字符串，也可以用来表征消息元数据，一旦消息被定义了key，那么就可以保证同一个key地所有消息都进入到相同地分区里面，由于每个分区下地消息处理都是顺序地，故这个策略被称为按消息键保序策略；




## 3. 计算机网络

### 3.1. TCP和UDP区别

![image-20220925125750469](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20220925125750469.png)

UDP在传送数据之前不需要先建立连接，远地主机在收到UDP报文后，不需要给出任何确认，虽然UDP不提供可靠交付，但是在某些情况下UDP确实一种最有效的工作方式（一般用于即时通讯），比如：qq语音，qq视频，直播等。

TCP提供面向连接的服务，在传送数据之前必须先建立连接，数据传送结束后要释放连接，TCP不提供广播或多播服务，由于TCP要提供可靠的、面向连接的传输服务（TCP的可靠体现在TCP在传递数据之前，会有三次握手来建立连接，而且在数据传递时，有确认、窗口、重传、拥塞控制，在数据传完之后，还会断开连接来节约系统资源），这就难以避免增加了需要开销，如确认，流量控制，计时器以及连接管理等，这不仅使协议数据单元的首部增大很多，还要占用许多处理机资源，TCP一般用户文件传输、发送和接收邮件、远程登录等场景。

**TCP报头格式：**

![image-20220925144120903](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20220925144120903.png)

**32位端口号：**

源端口和目的端口各占16位，2的16次方等于65536，看端口的命令：netstat。 

**32位序号：**

也称为顺序号（Sequence Number），简写为SEQ， 

**32位确认序号**： 

也称为应答号（Acknowledgment Number），简写为ACK。在握手阶段，确认序号将

发送方的序号加1作为回答。

**4位首部长度**： 

这个字段占4位，它的单位时32位（4个字节）。本例值为7，TCP的头长度为28字

节，等于正常的长度2 0字节加上可选项8个字节。，TCP的头长度最长可为60字节

（二进制1111换算为十进制为15，15*4字节=60字节）。

**6位标志字段**： 

ACK 置1时表示确认号（为合法，为0的时候表示数据段不包含确认信息，确认号被忽

略。

RST 置1时重建连接。如果接收到RST位时候，通常发生了某些错误。

SYN 置1时用来发起一个连接。

FIN 置1时表示发端完成发送任务。用来释放连接，表明发送方已经没有数据发送了。

**1. tcp报头格式**

PSH 置1时请求的数据段在接收方得到后就可直接送到应用程序，而不必等到缓冲区

满时才传送。注：一般不使用。

**16位检验和：**

检验和覆盖了整个的TCP报文段： TCP首部和TCP数据。这是一个强制性的字段，一

定是由发端计算和存储，并由收端进行验证。

**16位紧急指针：**

**注：**一般不使用。

只有当U R G标志置1时紧急指针才有效。紧急指针是一个正的偏移量，和序号字段中

的值相加表示紧急数据最后一个字节的序号。

**可选与变长选项**：

通常为空，可根据首部长度推算。用于发送方与接收方协商最大报文段长度

（MSS），或在高速网络环境下作窗口调节因子时使用。首部字段还定义了一个时间

戳选项。

最常见的可选字段是最长报文大小，又称为MSS (Maximum Segment Size)。每个

连接方通常都在握手的第一步中指明这个选项。它指明本端所能接收的最大长度的报

文段。1460是以太网默认的大小。

**UDP报文格式：**

![image-20220925144236387](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20220925144236387.png)

**2字节源端口字段**

源端口是一个大于1023的16位数字，由基于UDP应用程序的用户进程随机选择。

**2字节节的端口字段**

**2字节长度字段**

指明了包括首部在内的UDP报文段长度。UDP长字段的值是UDP报文头的长度(8字节)与

UDP所携带数据长度的总和。

**2字节校验和字段**

是指整个UDP报文头和UDP所带的数据的校验和（也包括伪报文头）。伪报文头不包括在真

正的UDP报文头中，但是它可以保证UDP数据被正确的主机收到了。因在校验和中加入了伪

头标，故ICMP除能防止单纯数据差错之外，对IP分组也具有保护作用

**面向报文：**

面向报文的传输方式是应用层交给UDP多长的报文，UDP就照样发送，即一次发

送一个报文。因此，应用程序必须选择合适大小的报文。若报文太长，则IP层需

要分片，降低效率。若太短，会是IP太小。UDP对应用层交下来的报文，既不合

并，也不拆分，而是保留这些报文的边界。这也就是说，应用层交给UDP多长的

报文，UDP就照样发送，即一次发送一个报文

**面向字节流：**

面向字节流的话，虽然应用程序和TCP的交互是一次一个数据块（大小不等），

但TCP把应用程序看成是一连串的无结构的字节流。TCP有一个缓冲，当应用程

序传送的数据块太长，TCP就可以把它划分短一些再传送。如果应用程序一次只

发送一个字节，TCP也可以等待积累有足够多的字节后再构成报文段发送出去。

![image-20220925144910352](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20220925144910352.png)

**TCP/UDP的优缺点：**

**TCP的优点：**

可靠，稳定

TCP的可靠体现在TCP在传递数据之前，会有三次握手来建立连接，而且在数据传递

时，有确认、窗口、重传、拥塞控制机制，在数据传完后，还会断开连接用来节约系

统资源。

**TCP的缺点：**

慢，效率低，占用系统资源高，易被攻击

TCP在传递数据之前，要先建连接，这会消耗时间，而且在数据传递时，确认机

制、重传机制、拥塞控制机制等都会消耗大量的时间，而且要在每台设备上维护

所有的传输连接，事实上，每个连接都会占用系统的CPU、内存等硬件资源。

而且，因为TCP有确认机制、三次握手机制，这些也导致TCP容易被人利用，实

现DOS、DDOS、CC等攻击。

**UDP的优点：**

快，比TCP稍安全

UDP没有TCP的握手、确认、窗口、重传、拥塞控制等机制，UDP是一个无状态

的传输协议，所以它在传递数据时非常快。没有TCP的这些机制，UDP较TCP被

攻击者利用的漏洞就要少一些。但UDP也是无法避免攻击的，比如：UDP Flood

攻击……

**UDP的缺点：**

不可靠，不稳定

因为UDP没有TCP那些可靠的机制，在数据传递时，如果网络质量不好，就会很

容易丢包。

### 3.2 TCP的三次握手、四次挥手

![image-20220925150301233](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20220925150301233.png)

**简述三次握手：**

![image-20220925170406685](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20220925170406685.png)

第一次握手：客户端从close状态变成syn_send状态，生成自身序号x，发送syn=1到服务端

第二次握手：服务收到客户端的syn包，从listen状态变成syn_receive状态，生成自身序号k, ack=x+1, syn=1,ack=1

第三次握手：客户端收到syn包，从syn_rend进入established状态，回复ACK=1,服务端收到ACK进入established状态

**简述四次挥手：**

![image-20220925170342548](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20220925170342548.png)

1）. 客户端进程发出连接释放报文，并且停止发送数据。释放数据报文首部，FIN=1，其序列号为seq=u（等于前面已经传送过来的数据的最后一个字节的序号加1），此时，客户端进入FIN-WAIT-1（终止等待1）状态。 TCP规定，FIN报文段即使不携带数据，也要消耗一个序号。
2）. 服务器收到连接释放报文，发出确认报文，ACK=1，ack=u+1，并且带上自己的序列号seq=v，此时，服务端就进入了CLOSE-WAIT（关闭等待）状态。TCP服务器通知高层的应用进程，客户端向服务器的方向就释放了，这时候处于半关闭状态，即客户端已经没有数据要发送了，但是服务器若发送数据，客户端依然要接受。这个状态还要持续一段时间，也就是整个CLOSE-WAIT状态持续的时间。
3）. 客户端收到服务器的确认请求后，此时，客户端就进入FIN-WAIT-2（终止等待2）状态，等待服务器发送连接释放报文（在这之前还需要接受服务器发送的最后的数据）。
4）. 服务器将最后的数据发送完毕后，就向客户端发送连接释放报文，FIN=1，ack=u+1，由于在半关闭状态，服务器很可能又发送了一些数据，假定此时的序列号为seq=w，此时，服务器就进入了LAST-ACK（最后确认）状态，等待客户端的确认。
5）. 客户端收到服务器的连接释放报文后，必须发出确认，ACK=1，ack=w+1，而自己的序列号是seq=u+1，此时，客户端就进入了TIME-WAIT（时间等待）状态。注意此时TCP连接还没有释放，必须经过2∗∗MSL（最长报文段寿命）的时间后，当客户端撤销相应的TCB后，才进入CLOSED状态。
6）. 服务器只要收到了客户端发出的确认，立即进入CLOSED状态。同样，撤销TCB后，就结束了这次的TCP连接。可以看到，服务器结束TCP连接的时间要比客户端早一些。



3次握手过程中的状态：

Listen: 表示服务器端的某个socket处于监听状态，可以接收连接了；

SYN_SENT: 当客户端socket执行connect连接时，他首先发送SYN报文，因此也会随即进入到了SYN_SENT状态，并等待服务端发送三次握手中的第2个报文。SYN_SENT状态表示客户端已发送SYN报文。

SYN_RCVD: 这个状态表示接收到了SYN报文，在正常情况下，这个状态是服务端的socket在建立TCP连接时的三次握手会话过程中的一个中间状态，很短暂，基本上用netstat很难看到这种状态。

ESTABLISHED: 表示连接已经建立了

4次挥手过程的状态：

FIN_WAIT-1: 这个状态要好好解释一下，**其实** **FIN_WAIT_1** **和** **FIN_WAIT_2** **状态的真** 

**正含义都是表示等待对方的** **FIN** **报文。**而这两种状态的区别是：FIN_WAIT_1 状态实际上 

是当 SOCKET 在 ESTABLISHED 状态时，它想主动关闭连接，向对方发送了 FIN 报文， 

此时该 SOCKET 即进入到 FIN_WAIT_1 状态。而当对方回应 ACK 报文后，则进入到 

FIN_WAIT_2 状态，当然在实际的正常情况下，无论对方何种情况下，都应该马上回应 ACK 

报文，所以 FIN_WAIT_1 状态一般是比较难见到的，而 FIN_WAIT_2 状态还有时常常可以 

用 netstat 看到。（主动方持有的状态） 

FIN_WAIT_2：上面已经详细解释了这种状态，实际上 FIN_WAIT_2 状态下的 SOCKET， 

表示半连接，也即有一方要求 close 连接，但另外还告诉对方，我暂时还有点数据需要传 

送给你(ACK 信息)，稍后再关闭连接。（主动方的状态） 

TIME_WAIT: **表示收到了对方的** **FIN** **报文，并发送出了** **ACK** **报文**，就等 2MSL 后即可 

回到 CLOSED 可用状态了。**如果** **FIN_WAIT_1** **状态下，收到了对方同时带** **FIN** **标志和** **ACK** 

**标志的报文时，可以直接进入到** **TIME_WAIT** **状态，而无须经过** **FIN_WAIT_2** **状态。**（主 

动方的状态）

**CLOSING****（比较少见）**: **表示双方同时关闭连接**。如果双方几乎同时调用 close 函数， 

那么会出现双方同时发送 FIN 报文的情况，就会出现 CLOSING 状态，表示双方都在关闭 

连接。这种状态比较特殊，实际情况中应该是很少见，属于一种比较罕见的例外状态。正 

常情况下，当你发送 FIN 报文后，按理来说是应该先收到（或同时收到）对方的 ACK 报 

文，再收到对方的 FIN 报文。但是 **CLOSING** **状态表示你发送** **FIN** **报文后，并没有收到对** 

**方的** **ACK** **报文，反而却收到了对方的** **FIN** **报文**。什么情况下会出现此种情况呢？其实细 

想一下，也不难得出结论：那就是**如果双方几乎在同时** **close** **一个** **SOCKET** **的话，那么** 

**就出现了双方同时发送** **FIN** **报文的情况，也即会出现** **CLOSING** **状态，表示双方都正在关** 

**闭** **SOCKET** **连接。** 

CLOSE_WAIT: 这种状态的含义其实是表示在等待关闭。**当对方** **close** **一个** **SOCKET** 

**后发送** **FIN** **报文给自己，你系统毫无疑问地会回应一个** **ACK** **报文给对方，此时则进入到** 

**CLOSE_WAIT** **状态。接下来，实际上你真正需要考虑的事情是察看你是否还有数据发送** 

**给对方，如果没有的话，那么你也就可以** **close** **这个** **SOCKET****，发送** **FIN** **报文给对方，** 

**也即关闭连接。**所以你在 CLOSE_WAIT 状态下，需要完成的事情是等待你去关闭连接。 

（被动方的状态） 

LAST_ACK: 这个状态还是比较容易好理解的，它是被动关闭一方在发送 FIN 报文后， 

最后等待对方的 ACK 报文。当收到 ACK 报文后，也即可以进入到 CLOSED 可用状态了。 

（被动方的状态）。 

CLOSED: 表示连接中断。

#### 3.2.1 **为什么要三次握手**

三次握手的目的是建立可靠的通信信道，因为TCP是全双工通信，三次握手的目的就是确认双方的发送与接收都是正常的,二次握手会让发送方知道服务端会收和发，但是缺了一次握手服务端不知道客户端可以收。



#### 3.2.2 **为什么要传回** **SYN**

接收端传回发送端所发送的 SYN 是为了告诉发送端，我接收到的信息确实就是你所发送的信号

了。

> SYN 是 TCP/IP 建⽴连接时使⽤的握⼿信号。在客户端和服务器之间建⽴正常的 TCP ⽹络
>
> 连接时，客户机⾸先发出⼀个 SYN 消息，服务器使⽤ SYN-ACK 应答表示接收到了这个消
>
> 息，最后客户机再以 ACK(Acknowledgement[汉译：确认字符 ,在数据通信传输中，接收站
>
> 发给发送站的⼀种传输控制字符。它表示确认发来的数据已经接受⽆误。 ]）消息响应。这
>
> 样在客户机和服务器之间才能建⽴起可靠的TCP连接，数据才可以在客户机和服务器之间传
>
> 递。

#### 3.2.3 **传了** **SYN,为啥还要传 ACK**

双⽅通信⽆误必须是两者互相发送信息都⽆误。传了 SYN，证明发送⽅到接收⽅的通道没有问

题，但是接收⽅到发送⽅的通道还需要 ACK 信号来进⾏验证。



#### 3.2.4 **什么是syn洪范攻击**

TCP三次握手中，客户端发送syn包到服务端，服务端收到SYN包后回复syn+ack，但是客户端不向服务器发送ACK包，因为连接处半开放连接，服务端在收到SYN就会分配资源，如果长时间没有ACK，就会浪费了大量的资源。

解决办法：

1. syn cookie，使用cookie维护半连接，取消半连接队列，将维护半连接的队列换成使用一个cookie去标识，等到三次握手成功才分配缓存区资源。
2. 加快半连接的过期时间
3. 增大半连接队列的容量



#### 3.2.5 为什么客户端还要等待2MSL

第一，保证客户端发送的最后一个ACK报文能够到达服务器，因为这个ACK报文可能丢失，站在服务器的角度看来，我们已经发送了FIN+ACK报文请求断开了，客户端还没有给我回应，应该是我发送的请求断开报文没有收到，于是服务器又会重新发送一次，而客户端就能在这个2MSL时间段内重传报文，接着给出回应报文，并且会重启2MSL计时器；

第二，保证该连接下的所有报文段都从网络中消失，这样新的连接中不会出现旧连接的请求报文；



#### 3.2.6 如果已经建立了连接，但是客户端突然出现了故障怎么半？

TCP有保活计时器，客户端如果出现故障，服务器不会一直等下去，白白浪费资源，服务器每收到一次客户端的请求后都会重新复位这个计时器，时间通常是2小时，若2小时没有收到客户端的任何数据，服务器就会发送一个探测报文段，以后每隔75秒发送一次，若一连发送10个探测报文仍然没有反应，服务器就认为客户端出了故障，接着就关闭连接；



#### 3.2.7 为什么建立连接是三次握手，关闭连接是四次挥手？

建立连接的时候，服务在listened状态，收到建立连接请求的SYN报文后，把ACK和SYN放在一个报文里发送给客户端，而关闭连接时，服务器收到对方的FIN包是，仅仅表示对方不再发送数据了，但是还能收到数据，而自己也未必全部数据都发送给对方了，所以己方可以理解关闭，也可以发送一些数据给对方后，在发送FIN报文给对方来表示同意现在关闭连接；所以关闭连接需要多一次；



#### 3.2.8 TCP四次挥手中间两次挥手可以合并吗？

TCP四次挥手里，第二次和第三次挥手之间，是有可能有数据传输的，第三次挥手的目的是为了告诉主动方，被动方没有数据要发了，所以在第一次挥手之后，如果被动方没有数据要发送给主动方，第二次和第三次挥手是有可能合并传输的，这样就出现了三次挥手；不过因为我们TCP有应答机制，所以我们第二次挥手一般都直接回复ACK。

TCP中有个特性叫延迟确认，可以简单理解为：接收方收到数据以后不需要立刻马上回复ACK确认包，在这个特性下，不是每一次发送的数据包都能对应收到一个ACK确认包，因此接收方可以合并确认。



### 3.3 OSI/五层协议

![image-20221004221108998](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221004221108998.png)

**应⽤层**

**应⽤层**(application-layer）的任务是通过应⽤进程间的交互来完成特定⽹络应⽤。应⽤层协议定

义的是应⽤进程（进程：主机中正在运⾏的程序）间的通信和交互的规则。对于不同的⽹络应⽤

需要不同的应⽤层协议。在互联⽹中应⽤层协议很多，如**域名系统**DNS，⽀持万维⽹应⽤的

**HTTP**协议**，⽀持电⼦邮件的 **SMTP协议等等。我们把应⽤层交互的数据单元称为报⽂。

**域名系统**

域名系统(Domain Name System缩写 DNS，Domain Name被译为域名)是因特⽹的⼀项核

⼼服务，它作为可以将域名和IP地址相互映射的⼀个分布式数据库，能够使⼈更⽅便的访问

互联⽹，⽽不⽤去记住能够被机器直接读取的IP数串。（百度百科）例如：⼀个公司的

Web ⽹站可看作是它在⽹上的⻔户，⽽域名就相当于其⻔牌地址，通常域名都使⽤该公司

的名称或简称。例如上⾯提到的微软公司的域名，类似的还有：IBM 公司的域名是 www.ib

m.com、Oracle 公司的域名是 www.oracle.com、Cisco公司的域名是 www.cisco.com 等。

**运输层**

**运输层**(transport layer)的主要任务就是负责向两台主机进程之间的通信提供通⽤的数据传输服

**务**。应⽤进程利⽤该服务传送应⽤层报⽂。“通⽤的”是指并不针对某⼀个特定的⽹络应⽤，⽽是

多种应⽤可以使⽤同⼀个运输层服务。由于⼀台主机可同时运⾏多个线程，因此运输层有复⽤和

分⽤的功能。所谓复⽤就是指多个应⽤层进程可同时使⽤下⾯运输层的服务，分⽤和复⽤相反，

是运输层把收到的信息分别交付上⾯应⽤层中的相应进程。

**运输层主要使⽤以下两种协议**

1. **传输控制协议** **TCP**（Transmission Control Protocol）--提供**⾯向连接**的，**可靠的**数据传输

服务。

2. **⽤户数据协议** **UDP**（User Datagram Protocol）--提供**⽆连接**的，尽最⼤努⼒的数据传输服

务（**不保证数据传输的可靠性**）。

**⽹络层**

**在 计算机⽹络中进⾏通信的两个计算机之间可能会经过很多个数据链路，也可能还要经过很多通**

**信⼦⽹。⽹络层的任务就是选择合适的⽹间路由和交换结点， 确保数据及时传送。** 在发送数据

时，⽹络层把运输层产⽣的报⽂段或⽤户数据报封装成分组和包进⾏传送。在 TCP/IP 体系结构

中，由于⽹络层使⽤ **IP** **协议**，因此分组也叫 **IP** **数据报** ，简称 **数据报**。

这⾥要注意：**不要把运输层的⽤户数据报**UDP和⽹络层的 IP 数据报弄混。另外，⽆论是哪

⼀层的数据单元，都可笼统地⽤“分组”来表示。

这⾥强调指出，⽹络层中的“⽹络”⼆字已经不是我们通常谈到的具体⽹络，⽽是指计算机⽹络体

系结构模型中第三层的名称.

互联⽹是由⼤量的异构（heterogeneous）⽹络通过路由器（router）相互连接起来的。互联⽹使

⽤的⽹络层协议是⽆连接的⽹际协议（Intert Protocol）和许多路由选择协议，因此互联⽹的⽹络

层也叫做**⽹际层**或**IP**层。

**数据链路层**

**数据链路层**(data link layer)通常简称为链路层。两台主机之间的数据传输，总是在⼀段⼀段的链

**路上传送的，这就需要使⽤专⻔的链路层的协议。** 在两个相邻节点之间传送数据时，**数据链路层**

**将⽹络层交下来的** **IP** **数据报组装成帧**，在两个相邻节点间的链路上传送帧。每⼀帧包括数据和

必要的控制信息（如同步信息，地址信息，差错控制等）。

在接收数据时，控制信息使接收端能够知道⼀个帧从哪个⽐特开始和到哪个⽐特结束。这样，数

据链路层在收到⼀个帧后，就可从中提出数据部分，上交给⽹络层。

控制信息还使接收端能够检测到所收到的帧中有误差错。如果发现差错，数据链路层就简单地丢

弃这个出了差错的帧，以避免继续在⽹络中传送下去⽩⽩浪费⽹络资源。如果需要改正数据在链

路层传输时出现差错（这就是说，数据链路层不仅要检错，⽽且还要纠错），那么就要采⽤可靠

性传输协议来纠正出现的差错。这种⽅法会使链路层的协议复杂些。

**物理层**

在物理层上所传送的数据单位是⽐特。

**物理层**(physical layer)的作⽤是实现相邻计算机节点之间⽐特流的透明传送，尽可能屏蔽掉具

**体传输介质和物理设备的差异。** 使其上⾯的数据链路层不必考虑⽹络的具体传输介质是什么。

“透明传送⽐特流”表示经实际电路传送后的⽐特流没有发⽣变化，对传送的⽐特流来说，这个电

路好像是看不⻅的。

在互联⽹使⽤的各种协中最重要和最著名的就是 TCP/IP 两个协议。现在⼈们经常提到的TCP/IP

并不⼀定单指TCP和IP这两个具体的协议，⽽往往表示互联⽹所使⽤的整个TCP/IP协议族。



### 3.4 TCP如何保证可靠传输的？

1. 应用数据被分割成TCP认为最适合发送的数据块
2. TCP给发送的每一个包进行编号，接收方对数据包进行排序，把有序数据传送给应用层；
3. 校验和：TCP将保持他首部和数据的校验和，这是一个端到端的校验和，目的是检测数据在传输过程中的任何变化，如果收到段的校验和有差错，TCP将丢弃这个报文段和不再确认收到此报文段；
4. TCP的接收端会丢弃重复的数据
5. 流量控制：TCP连接的每一方都有固定大小的缓存空间，当接收方来不及处理发送方的数据，能提示发送方降低发送的速率，防止包丢失；TCP使用的流量控制协议是可变大小的滑动窗口协议；
6. 拥塞控制：当网络拥塞时，减少数据的发送
7. ARQ协议：基本原理就是每发完一个分组就停止发送，等待对方确认，在收到确认后在发送下一个分组
8. 超时重传：当TCP发出一个段后，他启动一个定时器，等待目的端确认收到这个报文段，如果不能及时收到一个确认，将重发这个报文段；

### 3.5 ARQ协议

**⾃动重传请求**（Automatic Repeat-reQuest，ARQ）是OSI模型中数据链路层和传输层的错误纠

正协议之⼀。它通过使⽤确认和超时这两个机制，在不可靠服务的基础上实现可靠的信息传输。

如果发送⽅在发送后⼀段时间之内没有收到确认帧，它通常会重新发送。ARQ包括停⽌等待ARQ

协议和连续ARQ协议。

**停⽌等待**ARQ协议

停⽌等待协议是为了实现可靠传输的，它的基本原理就是每发完⼀个分组就停⽌发送，等待

对⽅确认（回复ACK）。如果过了⼀段时间（超时时间后），还是没有收到 ACK 确认，说

明没有发送成功，需要重新发送，直到收到确认后再发下⼀个分组；

在停⽌等待协议中，若接收⽅收到重复分组，就丢弃该分组，但同时还要发送确认；

**优点：** 简单

**缺点：** 信道利⽤率低，等待时间⻓

**1)** ⽆差错情况

发送⽅发送分组,接收⽅在规定时间内收到,并且回复确认.发送⽅再次发送。

**2)** 出现差错情况（超时重传）

停⽌等待协议中超时重传是指只要超过⼀段时间仍然没有收到确认，就重传前⾯发送过的分组

（认为刚才发送过的分组丢失了）。因此每发送完⼀个分组需要设置⼀个超时计时器，其重传时

间应⽐数据在分组传输的平均往返时间更⻓⼀些。这种⾃动重传⽅式常称为 **⾃动重传请求** **ARQ**

。另外在停⽌等待协议中若收到重复分组，就丢弃该分组，但同时还要发送确认。**连续** **ARQ** **协** 

**议** 可提⾼信道利⽤率。发送维持⼀个发送窗⼝，凡位于发送窗⼝内的分组可连续发送出去，⽽不

需要等待对⽅确认。接收⽅⼀般采⽤累积确认，对按序到达的最后⼀个分组发送确认，表明到这

个分组位置的所有分组都已经正确收到了。

**3)** **确认丢失和确认迟到**

**确认丢失** ：确认消息在传输过程丢失。当A发送M1消息，B收到后，B向A发送了⼀个M1确

认消息，但却在传输过程中丢失。⽽A并不知道，在超时计时过后，A重传M1消息，B再次

收到该消息后采取以下两点措施：1. 丢弃这个重复的M1消息，不向上层交付。 2. 向A发送

确认消息。（不会认为已经发送过了，就不再发送。A能重传，就证明B的确认消息丢

失）。

**确认迟到** ：确认消息在传输过程中迟到。A发送M1消息，B收到并发送确认。在超时时间内

没有收到确认消息，A重传M1消息，B仍然收到并继续发送确认消息（

B收到了2份M1）。

此时A收到了B第⼆次发送的确认消息。接着发送其他数据。过了⼀会，A收到了B第⼀次发

送的对M1的确认消息（

A也收到了2份确认消息）。处理如下：1. A收到重复的确认后，直接丢弃。2. B收到重复的M1后，也直接丢弃重复的M1。

**连续**ARQ协议

连续 ARQ 协议可提⾼信道利⽤率。发送⽅维持⼀个发送窗⼝，凡位于发送窗⼝内的分组可以连

续发送出去，⽽不需要等待对⽅确认。接收⽅⼀般采⽤累计确认，对按序到达的最后⼀个分组发

送确认，表明到这个分组为⽌的所有分组都已经正确收到了。

**优点：** 信道利⽤率⾼，容易实现，即使确认丢失，也不必重传。

**缺点：** 不能向发送⽅反映出接收⽅已经正确收到的所有分组的信息。 ⽐如：发送⽅发送了 5条

消息，中间第三条丢失（

3号），这时接收⽅只能对前两个发送确认。发送⽅⽆法知道后三个分

组的下落，⽽只好把后三个全部重传⼀次。这也叫 Go-Back-N（回退 N），表示需要退回来重传

已经发送过的 N 个消息。

### 3.6 滑动窗口进行流量控制

作用：

1. 提供TCP的可靠性，对发送的数据进行确认
2. 流量控制，窗口大小随链路变化

TCP窗口机制有两种，一种是固定窗口大小，另一种是滑动窗口，发送方和接收方都保持一个窗口，只有落在窗口内的数据才会被发送或接收，通过改变发送窗口、接收窗口的大小就可以实现流量控制。

TCP首部的window字段用来表示窗口的大小，窗口表示的就是接收方目前能接收的缓存区的剩余大小； TCP在传送数据时，第一次接收方窗口大小是由链路带宽决定的

**发送端的窗口：**

根据三个标准来划分：是否发送、是否收到ACK，是否在接收方通告处理范围内；

![image-20221005093858516](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221005093858516.png)

- 第一部分就是已经发送且收到 ACK 的部分，这一块我们知道已经成功发送，所以不需要在缓冲区保留了。

- 第二部分是已发送但尚未收到 ACK 的部分。

- 第三部分是还没有发送，但是还在接收方通告窗口也就是处理范围内的数据，这块我们也可以称为可用窗口；第二、第三部分一起构成了我们的整个发送窗口。

- 最后一部分则是我们需要发送，但已经超过接收方通告窗口范围的部分，这一部分在没有收到新的 ACK 之前，发送方是不会发送这些数据的。通过这个限制，发送的数据就一定不会超过接收方的缓冲区了。

但如果发送方一直没有收到 ACK，随着数据不断被发送，很快可用窗口就会被耗尽。在这种情况下，发送方也就不会继续发送数据了，这种发送端可用窗口为零的情况我们也称为“零窗口”。

![image-20221005094105441](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221005094105441.png)

正常来说，等接收端处理了一部分数据，又有了新的可用窗口之后，就会再次发送 ACK 报文通告发送端自己有新的可用窗口（因为发送端的可用窗口是受接收端控制的）。

但是，万一要是 ACK 消息在网络传输中正好丢包了，那发送端还能感知到接收端窗口的变化吗？其实是不会的，在这个情况下，接收端就会一直等着发送端发送数据，而发送端也还会以为接收端仍然处于零窗口的状态，这样一直互相等待，就好像进入了死锁状态。

解决办法也很简单，我们可以再引入一个零窗口定时器，如果发送端陷入零窗口的状态，就会启动这个定时器，去定时地询问接收端窗口是否可用了，这也是在分布式系统中常见的处理丢包的方式之一。

**接收端窗口**

相对发送端来说，接收端要简单的多，主要就分为已经接收并确认的数据和未收到但可以接收的数据，这一部分也就是接收窗口；剩下的就是缓冲区放不下的区域，也就是不可接收的区域。

![image-20221005094322336](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221005094322336.png)

如果进程读取缓冲区速度有所变化，接收端可能也会改变接收窗口的大小，每次通告给发送端，就可以控制发送端的发送速度了。这就是所谓的滑动窗口，也就是流量控制机制。

### 3.7 拥塞控制

在某段时间，若对⽹络中某⼀资源的需求超过了该资源所能提供的可⽤部分，⽹络的性能就要变

坏。这种情况就叫拥塞。拥塞控制就是为了防⽌过多的数据注⼊到⽹络中，这样就可以使⽹络中

的路由器或链路不致过载。拥塞控制所要做的都有⼀个前提，就是⽹络能够承受现有的⽹络负

荷。拥塞控制是⼀个全局性的过程，涉及到所有的主机，所有的路由器，以及与降低⽹络传输性

能有关的所有因素。相反，流量控制往往是点对点通信量的控制，是个端到端的问题。流量控制

所要做到的就是抑制发送端发送数据的速率，以便使接收端来得及接收

TCP发送维护一个拥塞窗口（cwnd），拥塞窗口的大小取决于网络的拥塞程度，并且动态变化，

TCP的拥塞控制采⽤了四种算法，即 **慢开始** 、 **拥塞避免** 、**快重传** 和 **快恢复**。在⽹络层也可以

![image-20221005143208748](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221005143208748.png)

![image-20221005143348407](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221005143348407.png)



使路由器采⽤适当的分组丢弃策略（如主动队列管理 AQM），以减少⽹络拥塞的发⽣

- 慢启动：慢开始算法的思路是当主机开始发送数据时，如果⽴即把⼤量数据字节注⼊到⽹

  络，那么可能会引起⽹络阻塞，因为现在还不知道⽹络的符合情况。经验表明，᫾好的⽅法

  是先探测⼀下，即由⼩到⼤逐渐增⼤发送窗⼝，也就是由⼩到⼤逐渐增⼤拥塞窗⼝数值。

  cwnd初始值为1，每经过⼀个传播轮次，cwnd加倍，当发送方每收到一个 ACK，拥塞窗口 cwnd 的大小就会加 1，当慢启动增加大于等于ssthresh(慢启动门限值)时开始使用拥塞避免算法；

- 拥塞避免：一般来说 ssthresh 的大小是 65535 字节。那么进入拥塞避免算法后，它的规则是：每当收到一个 ACK 时，cwnd 增加 1/cwnd，变成了线性增长

- 快重传：当接收方发现丢了一个中间包的时候，发送三次前一个包的 ACK，于是发送端就会快速地重传，不必等待超时再重传，TCP认为这种情况不严重，则只需要调整拥塞窗口减半，慢启动上限值等于当前的拥塞窗口值，进入快速恢复算法

- 快恢复：快速重传和快速恢复算法一般同时使用，快速恢复算法是认为，你还能收到 3 个重复 ACK 说明网络也不那么糟糕，正如前面所说，进入快速恢复之前，cwnd 和 ssthresh 已被更新了，快速恢复将拥塞窗口设置为ssthresh+3，重传丢失的数据包。

### 3.8 TCP的粘包

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247486329&idx=1&sn=bce6ac6c755d9251341f8138d450ee0c&chksm=c22b8d25f55c0433c5a93424d164a54030e744473b387cdd2d7cc326a47bd847091fb9826923&token=337827600&lang=zh_CN#rd

tcp粘包是指发送方发送的若干包数据到达接收方时粘成了一包，从接收缓冲区来看，后一包数据的头紧接着前一包数据的尾，出现粘包的原因是多方面的，可能是来自发送方，也有可能是来自接收方；

造成原因：

1.  发送方原因，TCP默认使用Nagle算法（主要作用：减少网络中报文段的数量），而Nagle算法主要做两件事
   1. 如果包长度达到MSS(或者FIN包)，立刻发送，否则等待下一个包到来，如果下一个包到来后两个包的总共长度超过MSS的话，就会立即发送
   1. 如果包长度没有大MSS，就会等待超时（一般为200ms），第一个包没到MSS长度，但是又迟迟等不到第二个包的到来，则立即发送；
2. 接收方原因：TCP接收到数据包时，并不会马上交到应用层进行处理，TCP将接收到的数据包保存在接收缓存里，然后应用程序主动从缓存读取收到的分组，。这样一来，如果TCP接收数据包到缓存的速度大于应用程序从缓存中读取数据包的速度，多个包就会被缓存，应用程序就有可能读取到多个首尾相接粘到一起的包。

怎么解决：

1. 如果时发送方造成的粘包，可以通过关闭Nagle算法来解决，使用TCP_NODELAY选项来关闭算法，Nagle算法其实是个有些年代的东西了，诞生于1984年，对于应用程序一次发送一个自己数据的场景，如果没有Nagle算法，包立即发送，会导致网络由于太多的包而过载，但是今天环境比以前好很多，所以可以关闭不使用该算法；
2. 接收方没有办法处理粘包，只能在应用层进行处理，因为粘包出现的根本原因是不确定的消息边界，常见的方法有：

   1. 加入特殊标志：通过特殊的标志作为头尾，像HTTP协议使用`chunked`编码传输时，使用若干个`chunk`组成消息，最后由一个标明长度为0的`chunk`结束；
   2. 加入消息长度信息，这个一般配合上面的特殊标志一起使用，在收到头标志时，里面还可以带上消息长度，以此表明在这之后多少 byte 都是属于这个消息的。如果在这之后正好有符合长度的 byte，则取走，作为一个完整消息给应用层使用。在实际场景中，HTTP 中的`Content-Length`就起了类似的作用，当接收端收到的消息长度小于 Content-Length 时，说明还有些消息没收到。那接收端会一直等，直到拿够了消息或超时
   3. 加入CRC校验，发送端在发送时还会加入各种校验字段（`校验和`或者对整段完整数据进行 `CRC` 之后获得的数据）放在标志位后面，在接收端拿到整段数据后校验下确保它就是发送端发来的完整数据。

   **UDP会发生粘包吗？**

   UDP协议是面向无连接的，不可靠的，基于数据包的传输层通信协议，基于数据报是指无论应用层交给UDP多长的报文，UDP都照样发送，即使在IP层需要分片，也是IP层的事情，正因为基于数据报和基于字节流的差异，TCP发送端发10次字节流数据，而这时候接收端可以分100次去取数据，每次取数据的长度可以根据处理能力做调整，但UDP发送端发了10次数据报，那接收端就要在10次收完，且发了多久，就取多少，确保每一次都是一个完整的数据报；

   所以UDP不会发生粘包；



### 3.9 在浏览器中输入url地址后处理流程

1. DNS解析：我们输入的一串域名，需要找到对应的IP地址，域名与IP地址进行绑定，DNS解析首先到本地域名服务器递归查找，然后再到根域名服务器迭代查找，再到顶级域名服务器迭代查询，最后到权限域名服务器迭代查找。也可以通过CName的方式，给DNS取别名，然后通过全局负载均衡器GSLB，解析就近同运营商的IP

2. TCP连接，HTTP/HTTPS是应用层协议，但是也都是使用TCP作为其传输层协议的，所以需要进行TCP三次握手建立TCP链接，如果使用的HTTPS，流程如下：

   ​		2.1 客户端向服务器发起请求，发送本地TLS版本、支持的加密算法套件，并且生成一个随机数R1

   ​        2.2 服务端确认TLS版本号，从client端支持的加密套件中选取一个，并生成一个随机数R2一起发送给Client; 服务端向客户端发送子的CA证书、数字签名

   ​        2.3 客户端判断证书签名与CA证书是否合法有效；服务端生成随机数pre-master，并使用第二步中的公钥对 Pre-Master 进行加密，将加密后的 Pre-Master 送给服务器
   这一步结束后，Client 与 Server 就都有 R1、R2、Pre-Master 了，两端便可以使用这 3 个随机数独立生成对称加密会话密钥了,避免了对称加密密钥的传输,同时可以根据会话密钥生成 6 个密钥（P1~P6）用作后续身份验证

   ​          2.4 Client 使用 P1 将之前的握手信息的 hash 值加密并发送给 Server，Client 发送握手结束消息，Server 计算之前的握手信息的 hash 值，并与 P1 解密客户端发送的握手信息的 hash 对比校验；验证通过后，使用 P2 将之前的握手信息的 hash 值加密并发送给 Client

   ​          2.5 Client 计算之前的握手信息的 hash 值，并与 P2 解密 Server 发送的握手信息的 hash 对比校验，验证通过后，开始发起 HTTPS 请求。 

   ![image-20221006183830418](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221006183830418.png)

3. 发送HTTP请求，构建HTTP请求报文并通过TCP协议发送到指定端口。

4. 服务处理请求并返回HTTP报文：后端在固定的端口接收到TCP报文开始，会对TCP连接进行处理，对HTTP协议进行解析，并按照报文格式进一步封装成HTTP request对象，供上层使用；

5. 浏览器解析渲染页面：前端在收到HTML、CSS、JS文件进行渲染；

6. 连接结束；

https://segmentfault.com/a/1190000006879700

### 3.10 HTTP1.0/HTTP1.1的主要区别？

![image-20221005192036021](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221005192036021.png)

主要区别体现如下：

1. **缓存处理：**在HTTP1.0中主要使用header里的If-Modified-Since,Expires来做为缓存判断的标准，HTTP1.1则引入了更多的缓存控制策略例如Entity tag，If-Unmodified-Since, If-Match, If-None-Match等更多可供选择的缓存头来控制缓存策略。
2. **带宽优化及网络连接的使用**，HTTP1.0中，存在一些浪费带宽的现象，例如客户端只是需要某个对象的一部分，而服务器却将整个对象送过来了，并且不支持断点续传功能，HTTP1.1则在请求头引入了range头域，它允许只请求资源的某个部分，即返回码是206（Partial Content），这样就方便了开发者自由的选择以便于充分利用带宽和连接。
3. **错误通知的管理**，在HTTP1.1中新增了24个错误状态响应码，如409（Conflict）表示请求的资源与资源的当前状态发生冲突；410（Gone）表示服务器上的某个资源被永久性的删除。
4. **Host头处理**，在HTTP1.0中认为每台服务器都绑定一个唯一的IP地址，因此，请求消息中的URL并没有传递主机名（hostname）。但随着虚拟主机技术的发展，在一台物理服务器上可以存在多个虚拟主机（Multi-homed Web Servers），并且它们共享一个IP地址。HTTP1.1的请求消息和响应消息都应支持Host头域，且请求消息中如果没有Host头域会报告一个错误（400 Bad Request）。
5. **长连接**，HTTP 1.1支持长连接（PersistentConnection）和请求的流水线（Pipelining）处理，在一个TCP连接上可以传送多个HTTP请求和响应，减少了建立和关闭连接的消耗和延迟，在HTTP1.1中默认开启Connection： keep-alive，一定程度上弥补了HTTP1.0每次请求都要创建连接的缺点。
6. 管道：HTTP1.1支持管道传输，只要第一个请求发送出去了，不必等其回来，就可以发第二个请求出去，可以减少整体的响应时间。



### 3.11 HTTPS与HTTP的一些区别

![image-20221005201903357](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221005201903357.png)

- HTTPS协议需要到CA申请证书，一般免费证书很少，需要交费。

- HTTP协议运行在TCP之上，所有传输的内容都是明文，HTTPS运行在SSL/TLS之上，SSL/TLS运行在TCP之上，所有传输的内容都经过加密的。

- HTTP和HTTPS使用的是完全不同的连接方式，用的端口也不一样，前者是80，后者是443。

- HTTPS可以有效的防止运营商劫持，解决了防劫持的一个大问题。

- HTTP协议运行在TCP之上，所有传输的内容都是明文，客户端和服务器端都无法验证对方的身份。HTTPS是运行在SSL/TLS之上的HTTP协议，SSL/TLS运行在TCP之上，所有的传输的内容都经过加密，加密采用对称加密，但对称加密的密钥用服务器的证书进行了非对称加密，所以说HTTP没有HTTPS的安全性高，但是HTTPS比HTTP耗费更多的服务资源:

  - 对称加密：密钥只有⼀个，加密解密为同⼀个密码，且加解密速度快，典型的对称加密

    算法有DES、AES等；
  - ⾮对称加密：密钥成对出现（且根据公钥⽆法推知私钥，根据私钥也⽆法推知公钥），

    加密解密使⽤不同密钥（公钥加密需要私钥解密，私钥加密需要公钥解密），相对对称

    加密速度᫾慢，典型的⾮对称加密算法有RSA、DSA等。
  
- HTTPS协议需要向CA(证书权威机构)申请数字证书，来保证服务器的身份是可信的。

HTTPS加密介绍：

采用混合加密算法：对称加密算法+非对称加密算法；

服务器用明文的方式给客户端发送公钥，客户端使用这个公钥生成一个密钥，用作对称加密使用，然后再把这个密钥传给服务器，服务器进行解密，最后服务器就可以安全得到这把密钥了，而客户端也同样一把密钥，这样就可以进行对称加密了；但是这样也并不是万无一失的，服务器明文传给客户端公钥时，如果中间人截取了这把属于服务器的公钥，中间人把自己的公钥冒充服务器的公钥传输给了客户端，这时中间人就可以通过转发的方式进行窃取；

为了解决这个问题我们需要找到一种策略来证明这把公钥就是服务器的，而不是别人冒充的，数字证书就是来接这个问题的，网站在使用HTTPS前需要向CA机构申领一份数字证书，数字证书里含有持有者信息、公钥信息，服务器把证书传给浏览器，浏览器从证书里获取公钥就行，数字证书使用数字签名来做防伪技术，我们把证书原本的内容生成一份“签名”，比对证书内容和签名是否一致就能判别是否被篡改；CA机构拥有非对称加密的私钥和公钥，CA机构对证书明文数据T进行Hash，对hash后的值用私钥加密，得到数字签名S。浏览器拿到证书后，得到明文T，签名S，用CA机构的公钥对S进行加密，得到S'，用证书里指明的hash算法对明文T进行Hash得到T'. 判断T'和S'即可。



### 3.12 HTTP 2.0的优势

- 新的二进制格式，：HTTP1.x的解析是基于文本。基于文本协议的格式解析存在天然缺陷，文本的表现形式有多样性，要做到健壮性考虑的场景必然很多，二进制则不同，只认0和1的组合。基于这种考虑HTTP2.0的协议解析决定采用二进制格式，实现方便且健壮。
- 多路复用：即连接共享，即每一个request都是是用作连接共享机制的。一个request对应一个id，这样一个连接上可以有多个request，每个连接的request可以随机的混杂在一起，接收方可以根据request的 id将request再归属到各自不同的服务端请求里面。
- **header压缩**，如上文中所言，对前面提到过HTTP1.x的header带有大量信息，而且每次都要重复发送，HTTP2.0使用encoder来减少需要传输的header大小，通讯双方各自cache一份header fields表，既避免了重复header的传输，又减小了需要传输的大小。
- **服务端推送**（server push），同SPDY一样，HTTP2.0也具有server push功能

HTTP1.1的性能瓶颈：

- 请求/响应头部（header）未经压缩就发送，首部信息越多延迟越大，只能压缩body部分；
- 发送冗长的首部，每次互相发送相同的首部造成资源浪费
- 没有优先级控制
- 服务器处理是按照请求的顺序响应的，如果服务器响应很慢，会导致客户端一直请求不到数据，也就是对头阻塞；
- 请求只能从客户端开始，服务器只能被动响应；

HTTP/2.0 做了哪些优化：

HTTP2.0协议是基于HTTPS的，所以HTTP/2的安全性是有保障的。

1. 头部压缩

HTTP/2会压缩头（header），如果同时发送多个请求，他们的头是一样的或者是相似的，那么他们会消除重复的部分；采用HPACk算法，在客户端和服务端同时维护一张头信息表，所有字段都会存入这个表，生成一个索引号，这样就提高速度了。

2. 二进制格式

HTTP2.0里面全面采用了二进制格式，不再像HTTP1.1里的纯文本形式的报文，头信息和数据体都是二进制，并且统称为帧（frame）：头信息帧和数据帧，增加了传输效率

![image-20221006103000700](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221006103000700.png)

3. 数据流

HTTP 2.0的数据包不是按顺序发送的，同一个连接里面连续的数据包，可能属于不同的回应，所以要对数据包做标记，指出它属于哪个回应；每个请求或回应的所有数据包，称为一个数据流（Stream）。

每个数据流都标志着一个独一无二的编号，其中规定客户端发出的数据流编号为奇数，服务器发出的数据流编号为偶数；客户端还可以指定数据流的优先级，优先级高的请求，服务器就先响应该请求；

![image-20221006103513031](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221006103513031.png)

4. 多路复用

HTTP/2.0是可以在一个连接中并发多个请求或回应，而不用按照顺序一一对应；

移除了HTTP1.1中的串行请求，不需要排队等待，也就不会出现对头阻塞的问题，降低了延迟，大幅度提高了连接的利用率；

5. 服务器推送

HTTP2.0 还在一定程度上改善了传统的请求-应答的工作模式，服务不再是被动地响应，也可以主动向客户端发送消息；

举例来说，在浏览器刚请求 HTML 的时候，就提前把可能会用到的 JS、CSS 文件等静态资源主动发给客户端，**减少延时的等待**，也就是服务器推送（Server Push，也叫 Cache Push）。

### 3.13 HTTP3.0做了什么优化

HTTP/2 主要的问题在于：多个 HTTP 请求在复用一个 TCP 连接，下层的 TCP 协议是不知道有多少个 HTTP 请求的。

所以一旦发生了丢包现象，就会触发 TCP 的重传机制，这样在一个 TCP 连接中的**所有的 HTTP 请求都必须等待这个丢了的包被重传回来**。

- HTTP/1.1 中的管道（ pipeline）传输中如果有一个请求阻塞了，那么队列后请求也统统被阻塞住了
- HTTP/2 多请求复用一个TCP连接，一旦发生丢包，就会阻塞住所有的 HTTP 请求。

这都是基于 TCP 传输层的问题，所以 **HTTP/3 把 HTTP 下层的 TCP 协议改成了 UDP！**

![image-20221006104413255](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221006104413255.png)

UDP 发生是不管顺序，也不管丢包的，所以不会出现 HTTP/1.1 的队头阻塞 和 HTTP/2 的一个丢包全部重传问题。

大家都知道 UDP 是不可靠传输的，但基于 UDP 的 **QUIC 协议** 可以实现类似 TCP 的可靠性传输。

- QUIC 有自己的一套机制可以保证传输的可靠性的。当某个流发生丢包时，只会阻塞这个流，**其他流不会受到影响**。
- TL3 升级成了最新的 `1.3` 版本，头部压缩算法也升级成了 `QPack`。
- HTTPS 要建立一个连接，要花费 6 次交互，先是建立三次握手，然后是 `TLS/1.3` 的三次握手。QUIC 直接把以往的 TCP 和 `TLS/1.3` 的 6 次交互**合并成了 3 次，减少了交互次数**。



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/J0g14CUwaZfXG1113Sjm0iaOXfoOv0tlUyP3HNicKS2J21mHQD9EepOiciakC8nRkrX9C3I0hjC6Fhjvd4nLiakuLeg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)TCP HTTPS（TLS/1.3） 和 QUIC HTTPS

所以， QUIC 是一个在 UDP 之上的**伪** TCP + TLS + HTTP/2 的多路复用的协议。

QUIC 是新协议，对于很多网络设备，根本不知道什么是 QUIC，只会当做 UDP，这样会出现新的问题。所以 HTTP/3 现在普及的进度非常的缓慢，不知道未来 UDP 是否能够逆袭 TCP。

### 3.14 URI和URL的区别是什么？

URI是统一资源标识符，可以唯一标识一个资源

URL是统一资源定位符，可以提供该资源的路径，他是一种具体的URI，即URL可以用来标识一个资源，而且还指明了如何locate这个资源；

URI的作用像身份证号一样，URL的作用更像家庭住址一样，URL是一种具体的URI，它不仅唯一标识资源，而且还提供了定位该资源的信息。



### 3.15 HTTP状态码

![image-20221006083637325](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221006083637325.png)

*1xx*

`1xx` 类状态码属于**提示信息**，是协议处理中的一种中间状态，实际用到的比较少。

*2xx*

`2xx` 类状态码表示服务器**成功**处理了客户端的请求，也是我们最愿意看到的状态。

「**200 OK**」是最常见的成功状态码，表示一切正常。如果是非 `HEAD` 请求，服务器返回的响应头都会有 body 数据。

「**204 No Content**」也是常见的成功状态码，与 200 OK 基本相同，但响应头没有 body 数据。

「**206 Partial Content**」是应用于 HTTP 分块下载或断电续传，表示响应返回的 body 数据并不是资源的全部，而是其中的一部分，也是服务器处理成功的状态。

*3xx*

`3xx` 类状态码表示客户端请求的资源发送了变动，需要客户端用新的 URL 重新发送请求获取资源，也就是**重定向**。

「**301 Moved Permanently**」表示永久重定向，说明请求的资源已经不存在了，需改用新的 URL 再次访问。

「**302 Moved Permanently**」表示临时重定向，说明请求的资源还在，但暂时需要用另一个 URL 来访问。

301 和 302 都会在响应头里使用字段 `Location`，指明后续要跳转的 URL，浏览器会自动重定向新的 URL。

「**304 Not Modified**」不具有跳转的含义，表示资源未修改，重定向已存在的缓冲文件，也称缓存重定向，用于缓存控制。

*4xx*

`4xx` 类状态码表示客户端发送的**报文有误**，服务器无法处理，也就是错误码的含义。

「**400 Bad Request**」表示客户端请求的报文有错误，但只是个笼统的错误。

「**403 Forbidden**」表示服务器禁止访问资源，并不是客户端的请求出错。

「**404 Not Found**」表示请求的资源在服务器上不存在或未找到，所以无法提供给客户端。

*5xx*

`5xx` 类状态码表示客户端请求报文正确，但是**服务器处理时内部发生了错误**，属于服务器端的错误码。

「**500 Internal Server Error**」与 400 类型，是个笼统通用的错误码，服务器发生了什么错误，我们并不知道。

「**501 Not Implemented**」表示客户端请求的功能还不支持，类似“即将开业，敬请期待”的意思。

「**502 Bad Gateway**」通常是服务器作为网关或代理时返回的错误码，表示服务器自身工作正常，访问后端服务器发生了错误。

「**503 Service Unavailable**」表示服务器当前很忙，暂时无法响应服务器，类似“网络服务正忙，请稍后重试”的意思。



### 3.16 HTTP介绍

https://mp.weixin.qq.com/s/bUy220-ect00N4gnO0697A

#### 3.16.1 HTTP包含哪些header

HTTP消息头是指在超文本传输协议的请求和响应消息中，协议头部分的那些组件；

HTTP消息头在客户端请求或服务器响应时传递的，位于请求或响应第一行；

常用HTTP请求头：

Connection：keep-alive,长连接保活机制

Host：发送请求时，该报头域是必需的

Accept-Encoding：浏览器申明自己接收的编码方法，通常指定压缩方法，是否支持压缩，支持什么压缩方法（gzip，deflate）

Accept：浏览器端可以接受的媒体类型，例如text/html

user-agent：告诉HTTP服务器，客户端使用的操作系统和浏览器的名称和版本；

referer：当浏览器向Web服务器发送请求的时候，一般会带上referer，告诉服务器我是从哪个页面链接过来的；

Content-length：请求内容的长度

Accept-Charset：浏览器可以接受的字符编码集

Accept-language：浏览器可以接受的语言

Range：请求实体的一部分





### 3.17 MTU和MSS

https://mp.weixin.qq.com/s/ZMV2izeYkBIqjPhsv_-wdw

**总结：**

MTU是以太网数据链路层中约定的数据载荷部分最大长度，数据不超过他时就不需要分片；

MSS是传输层的概念，由于数据往往很大，会超过MTU，网络层会进行IP分片，将很大的数据载荷分为多个分片发送出去，由于TCP为了IP层不用分片主动将数据包切割为MSS大小：

MSS = MTU - IP header大小 - TCP头大小

**MTU：**

MTU全称是MAximum Transmission unit 即最大传输单元，

在学习数据链路层时，我们学习过以太网协议，以太网定义了一个叫做帧的概念，一个帧中包含如下信息：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ta8IaFW1v1crAvR38qZdSSeomTrNqO86mcWJU98R8bYYLZPCImquLWPWhnYoVmakAxlEGmhPlibHcwLFpCWkicxQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

此外，我们学习了帧的大小，其中帧头大小为：

- 接收方和发送方的 MAC 地址分别占用 6 个字节；
- 第 3 层的协议用 2 个字节编码；
- CRC 用 4 个字节编码。

6 x 2 + 2 + 4 = 18。因此以太网的帧头一共有 18 个字节。并且以太网中还规定了最小帧长和最大帧长：

- 以太网帧的最小尺寸是 64 字节，那么一帧中最少报文长度为46字节。
- 以太网帧的最大尺寸是 1518 字节，那么一帧中中最大报文长度为1500字节。

其中1500字节往往就是以太网的MTU值了，传输的数据小于它时，就无需切片。

太大的数据就需要切分，就像一个超级大包裹需要切分为若干个小包裹才方便托运。假设传输100KB的数据，则需要切分为多少个帧进行传输呢？

100KB=100*1024B，由于帧中最大的报文长度是1500B，那么100KB/1500B≈68.27，显然需要69个以太网帧才能承载。

在学习数据链路层时，我们学习过以太网协议，以太网定义了一个叫做帧的概念，一个帧中包含如下信息：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ta8IaFW1v1crAvR38qZdSSeomTrNqO86mcWJU98R8bYYLZPCImquLWPWhnYoVmakAxlEGmhPlibHcwLFpCWkicxQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

此外，我们学习了帧的大小，其中帧头大小为：

- 接收方和发送方的 MAC 地址分别占用 6 个字节；
- 第 3 层的协议用 2 个字节编码；
- CRC 用 4 个字节编码。

6 x 2 + 2 + 4 = 18。因此以太网的帧头一共有 18 个字节。并且以太网中还规定了最小帧长和最大帧长：

- 以太网帧的最小尺寸是 64 字节，那么一帧中最少报文长度为46字节。
- 以太网帧的最大尺寸是 1518 字节，那么一帧中中最大报文长度为1500字节。

其中1500字节往往就是以太网的MTU值了，传输的数据小于它时，就无需切片。

太大的数据就需要切分，就像一个超级大包裹需要切分为若干个小包裹才方便托运。假设传输100KB的数据，则需要切分为多少个帧进行传输呢？

100KB=100*1024B，由于帧中最大的报文长度是1500B，那么100KB/1500B≈68.27，显然需要69个以太网帧才能承载。

**MSS**

MSS的英文全称叫Max Segment Size，是TCP最大段大小。

在建立TCP连接的同时，也可以确定发送数据包的单位，我们称为MSS，这个MSS正好是IP中不会被分片处理的最大数据长度。

TCP在传送大量数据时，是以MSS的大小将数据进行分割发送的，重发时也是以MSS为单位。

**MSS是在三次握手的时候，在两端主机之间被计算得出，两端主机在发出建立连接的请求时，会在TCP首部中写入MSS选项，告诉对方自己的接口能够适应的MSS的大小，然后在两者之间选择一个较小的值投入使用。**




## 4. 操作系统

### 4.1 堆栈区别

堆（heap）和 栈（stack）在不同的场景下，代表的含义也不同，主要有两种：

- 在存储方面，堆与栈表示两种内存管理方式
- 在计算领域中，堆与栈表示两种常用的数据结构

堆栈在内存分区上的区别：

![image-20220927113834684](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20220927113834684.png)

进程为每个程序提供自己的私有空间，通用结构如上：

堆（heap）: 堆用来存放进程运行中被动态分配的内存段，需要程序员分配和释放；

栈（stack）：由编译器管理自动分配，在栈存放函数的参数值、返回值、局部变量等，由系统自动分配和释放；

区别：

- 管理方式：
  - 对于栈来讲，是由编译器自动管理
  - 对于堆来说，释放由程序员控制，容易产生memory leak
- 空间大小
  - 一般来讲在32位系统下，堆内存可以达到接近4G的空间
  - 栈一般是有空间大小限制的，在vc6下面，默认的栈空间大小约是1m
- 碎片问题：
  - 对于堆来讲，频繁的创建与删除会造成空间不连续，从而造成大量的碎片，使程序效率降低
  - 对于栈来讲，不会存在这个问题，栈是先进后出的队列；永远不会出现一个内存块从栈中间弹出
- 生成方向：
  - 对于堆来讲，生长方向是向上的，也就是向着内存地址增加的方向
  - 对于栈来讲，他的生长方向是向下的，是向着内存地址减小的方向增长；
- 分配方式
  - 堆都是动态分配的，没有静态分配的堆
  - 栈有2种分配方式，静态分配和动态分配
    - 静态分配时编译器完成的，比如局部变量的分配，动态分配由alloca函数进行分配
    - 栈的动态分配也是由编译器进行释放，不需要手动实现
- 分配效率
  - 栈是机器提供的数据结构，计算机会在底层分配专门的寄存器存放栈的地址，压栈出栈都会有专门的指令执行，这就决定了栈的效率比较高；
  - 堆是c/c++函数库提供的，他的机制是复杂的，例如分配一块内存，库函数会按照一定的算法在堆内存中搜索可用的足够大小的空间，如果没有足够大小的空间，就有可能调用系统功能去增加程序数据段的内存空间，然后进行返回；

堆栈在数据结构上的区别：

堆：一种常用的树形结构，是一种特殊的完全二叉树，当前仅当满足所有节点的值总是不大于或不小于其父节点的值的完全二叉树被称之为堆，分为小根堆和大根堆，常用实现优先队列，堆的存储时随意的；

栈：是一种运算受限的线性表，其限制是指只允许在表的一端进行插入和删除操作

- 拥有”先进后出“的特性
  - 允许操作这一端被称为栈顶，相对的另一端称为栈底
  - 关于栈的两个重要的操作是PUSH和POP
    - PUSH操作在堆栈的顶部加入一个元素
    - POP操作则相反，在堆栈顶部移去一个元素，并将堆栈的大小减1
    - 栈常用来实现递归

### 4.2 进程和线程什么区别

进程是操作系统分配资源的基本单位，一个进程拥有的资源有自己的堆、栈、虚存空间（页表）、文件描述符等信息；

线程是操作系统能够进行运算调度的基本单位。他包含在进程中，是进程中实际运行单位，在Unix system V及 SunOS中线程也被称为轻量进程，但是轻量进程更多指内核线程，而把用户线程称为线程；一个进程中包含多个线程，因此多个线程间可以共享进程资源；

进程和线程的区别总结：

- 进程中包含了线程，进程是正在运行的程序实例，一个运行的程序至少包含一个线程，线程是进程真正执行任务的基本单位，一个进程可以包含多个线程，但是一个线程只能属于一个进程；
- 进程是操作系统分配资源的基本单位，线程是操作系统调度的基本单位
- 多个进程间不能共享资源，每个进程都有自己的堆、栈、虚存空间（页表）、文件描述符等信息，进程下的多个线程可以共享该进程资源（堆和方法区）
- 进程的操纵者是操作系统，线程的操作者是编程人员；
- 线程切换开销要比进程切换开销要小；进程更方便资源的管理和保护；



### 4.3 为什么要使用多线程呢?

从计算机底层来说：线程是轻量级进程，是程序执行的最小单位，线程间的切换和调度的成本远远小于进程，多核CPU时代意味着多个线程可以同时运行，这减少了线程上下文切换的开销；

互联网发展趋势来说：现在的系统动辄就百万级甚至千万级的并发量，而多线程并发编程正式开发高并发系统的基础，利用好多线程机制可以大大提搞系统整体的并发能力以及性能；

**单核时代**：在单核时代多线程主要是为了提高CPU和IO设备的综合利用率；举个例⼦：

当只有⼀个线程的时候会导致 CPU 计算时，IO 设备空闲；进⾏ IO 操作时，CPU 空闲。我

们可以简单地说这两者的利⽤率⽬前都是 50%左右。但是当有两个线程的时候就不⼀样了，

当⼀个线程执⾏ CPU 计算时，另外⼀个线程可以进⾏ IO 操作，这样两个的利⽤率就可以在

理想情况下达到 100%了。

**多核时代**:多核时代多线程主要是为了提⾼ CPU 利⽤率。举个例⼦：假如我们要计算⼀个复

杂的任务，我们只⽤⼀个线程的话，CPU 只会⼀个 CPU 核⼼被利⽤到，⽽创建多个线程就

可以让多个 CPU 核⼼被利⽤到，这样就提⾼了 CPU 的利⽤率。



### 4.4 使用多线程带来了什么问题

并发编程的目的就是提高程序的执行效率、程序的运行速度，但是并发编程会遇到：内存泄漏、上下文切换、死锁等问题。

### 4.4.1 什么是死锁

死锁：多个线程同时被阻塞，他们中的一个或多个全部都在等待某个资源被释放，因为资源一直不被释放，所以程序不能正常终止；

产生死锁的四个条件：

- 互斥条件：该资源任一时刻只能有一个线程占用；
- 请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放；
- 不剥夺条件：线程已获得的资源在未使用完之前不能被其他线程强行剥夺，只有自己使用完毕后才能释放
- 循环等待条件：若干进程之间形成一种头尾相接的循环等待资源关系；



### 4.5 说说并发和并行的区别

并发：同一时间段，多个任务都在执行（不是同时）

并行：单位时间内，多个任务同时执行



### 4.6 什么是操作系统

1. **操作系统（**Operating System，简称OS**）是管理计算机硬件与软件资源的程序，是计算机**

   **的基⽯**
2. **操作系统本质上是⼀个运⾏在计算机上的软件程序 ，⽤于管理计算机硬件和软件资源。** 举

   例：运⾏在你电脑上的所有应⽤程序都通过操作系统来调⽤系统内存以及磁盘等等硬件

3. **操作系统存在屏蔽了硬件层的复杂性。** 操作系统就像是硬件使⽤的负责⼈，统筹着各种相关

   事项

4. **操作系统的内核（**Kernel**）是操作系统的核⼼部分，它负责系统的内存管理，硬件设备的管**

   **理，⽂件系统的管理以及应⽤程序的管理**。 内核是连接应⽤程序和硬件的桥梁，决定着系统

   的性能和稳定性。


![image-20221005204200142](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221005204200142.png)

### 4.7 什么是系统调用

我们把进程在系统上的运行分为两个部分：

- 用户态：用户态运行的进程可以直接读取用户程序的数据
- 内核态：系统运行的进程或程序几乎可以访问计算机的任何资源，不受限制；

我们运行的程序基本都运行在用户态，如果我们调用操作系统提供的系统态级别的子功能就需要系统调用了，凡是与系统态级别的资源有关的操作，都必须通过系统调用方式向操作系统提出服务请求，并由操作系统代为完成。

分为如下几类：

- 设备管理：完成设备的请求或释放，以及设备启动等功能
- 文件管理：完成文件的读、写、创建以及删除等功能
- 进程控制：完成进程的创建、撤销、阻塞以及唤醒等功能
- 进程通信：完成进程之前的消息传递或信号传递等功能
-  内存管理：完成内存的分配、回收等功能；

### 4.8 进程间的通信

- 管道/匿名管道：用于具有亲缘关系的父子进程间或兄弟进程之间的通信，管道是半双工的，数据只能向一个方向流动，需要双方通信时需要建立两个管道，管道的缓冲区是有限的，管道所传送的是无格式字节流，这就要求管道的读出方和写入方必须事先约定好数据的格式，比如多少字节算作一个消息

- 有名管道：匿名管道由于没有名字，只能用于亲缘关系的进程间通信，为了克服这个缺点，提出了有名管道，有名管道严格遵循先进先出，有名管道以磁盘文件的方式存在，可以实现本机任意两个进程通信

- 信号：信号是一种比较复杂的通信方式，用于通知接收进程某个事件已经发生；

- 消息队列：消息队列是消息的链表,具有特定的格式,存放在内存中并由

  消息队列标识符标识。管道和消息队列的通信数据都是先进先出的原则。与管道（⽆名管

  道：只存在于内存中的⽂件；命名管道：存在于实际的磁盘介质或者⽂件系统）不同的是消

  息队列存放在内核中，只有在内核重启(即，操作系统重启)或者显示地删除⼀个消息队列

  时，该消息队列才会被真正的删除。消息队列可以实现消息的随机查询,消息不⼀定要以先进

  先出的次序读取,也可以按消息的类型读取.⽐ FIFO 更有优势。**消息队列克服了信号承载信**

  **息量少，管道只能承载⽆格式字 节流以及缓冲区⼤⼩受限等缺。**

- 信号量(Semaphores) ：信号量是⼀个计数器，⽤于多进程对共享数据的访问，信号量的意
图在于进程间同步。这种通信⽅式主要⽤于解决与同步相关的问题并避免竞争条件。

- 共享内存(Shared memory) ：使得多个进程可以访问同⼀块内存空间，不同进程可以及时
看到对⽅进程中对共享内存中数据的更新。这种⽅式需要依靠某种同步操作，如互斥锁和信
号量等。可以说这是最有⽤的进程间通信⽅式

- 套接字(Sockets) : 此⽅法主要⽤于在客户端和服务器之间通过⽹络进⾏通信。套接字是⽀
  持 TCP/IP 的⽹络通信的基本操作单元，可以看做是不同主机之间的进程进⾏双向通信的端
  点，简单的说就是通信的两⽅的⼀种约定，⽤套接字中的相关函数来完成通信过程。

  [进程间通信IPC (InterProcess Communication) - 简书 (jianshu.com)](https://www.jianshu.com/p/c1015f5ffa74)

### 4.9进程有哪几种状态

- 创建状态New：进程正在被创建，尚未到就绪状态
- 就绪状态READY：进程已经处于准备运行的状态，即进程获得了除处理器之外的一切所需资源，一旦得到处理器资源即可运行
- 运行状态RUNNING：进程正在处理器上运行
- 阻塞状态WAITING：等待状态，进程正在等待某一事件而暂停运行如等待某资源为可用或等待IO操作完成，即使处理器空闲，该进程也不能运行；
- 结束状态terminated：进程正在从系统中消失，可能是进程正常结束或其他原因中断退出运行；



### 4.10 线程间的同步方式

操作系统一般使用三种线程同步的方式：

- 互斥量：采用互斥对象机制，只有拥有互斥对象的线程才能访问公共资源的权限，

- 因为互斥对象只有⼀个，所以可以保证公共资源不会被多个线程同时访问。⽐如 Java 中的

  synchronized 关键词和各种 Lock 都是这种机制

- 信号量：他允许同一时刻多个线程访问同一资源，但是需要控制同一时刻访问此资源的最大线程数量

- 事件：wait/notify通过通知操作的方式来保持多线程同步，还可以方便的实现多线程优先级的比较。

### 4.11 进程调度算法了解多少？

- 时间片轮转调度算法：最简单、最公平且使用最广的算法，每个进程被分配一个时间段，称作他的时间片，即该进程允许运行的时间；
- 先到先服务调度算法（FCFS）：从就绪队列中选择一个最先进入该队列的进程为之分配资源，使它立即执行并一直执行到完成或发生某事件而被阻塞放弃占用CPU时再重新调度；
- 短作业优先调度算法（SJF）：从就绪队列中选出一个估计运行时间最短的进程为之分配资源，使他立即执行并一直执行到完成或发生某事件而被阻塞放弃占用CPU时在重新调度；
- 优先级调度：为每个流程分配优先级，首先执行具有最高优先级的进程，具有相同邮件的进程以先到先服务调度算法执行，根据内存要求，时间要求或任何其他资源要求来确定优先级
- 多级反馈队列调度算法：多级反馈队列调度算法既能使高优先级的作业得到响应又能使短作业迅速完成。



### 4.12 什么是原子性

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247484781&idx=1&sn=dcd593f3fe1fa8fe75a18dbd53802dc8&chksm=c22b8331f55c0a27895579ead0d5f694071ab206f2eb91273e520c57a1f9ec1c6fe62f2c5933&token=1817605393&lang=zh_CN#rd

原子本意是"不能进一步分割的最小粒子"，而原子操作意为"不可中断的一个或一些列操作"，其实用大白话说出来就是让多个线程对同一块内存的操作是串行的，不会因为并发操作把内存写的不符合预期，我们来看这样一个例子：假设现在是一个银行账户系统，用户A想要自己从自己的账户中转1万元到用户B的账户上，直到转帐成功完成一个事务，主要做这两件事：

- 从A的账户中减去1万元，如果A的账户原来就有2万元，现在就变成了1万元
- 给B的账户添加1万元，如果B的账户原来有2万元，那么现在就变成了3万元

假设在操作一的时候，系统发生了故障，导致给B账户添加款项失败了，那么就要进行回滚。回滚就是回到事务之前的状态，我们把这种要么一起成功的操作叫做原子操作，而原子性就是要么完整的被执行、要么完全不执行。



### 4.13 如何保证原子性

- 锁机制

在处理器层面，可以采用总线加锁或者对缓存加锁的方式来实现多处理器之间的原子操作。通过加锁保证从系统内存中读取或写入一个字节是原子的，也就是当一个处理器读取一个字节时，其他处理器不能访问这个字节的内存地址。

总线锁：处理器提供一个`Lock#`信号，当一个处理器上在总线上输出此信号时，其他处理器的请求将被阻塞住，那么该处理器可以独占共享内存。总线锁会把`CPU`和内存之间的通信锁住了，在锁定期间，其他处理就不能操作其他内存地址的数据，所以总线锁定的开销比较大，所以处理会在某些场合使用缓存锁进行优化。缓存锁：内存区域如果被缓存在处理器上的缓存行中，并且在`Lock#`操作期间，那么当它执行操作回写到内存时**，处理不在总线上声言`Lock#`信号，而是修改内部的内存地址，并允许它的缓存一致机制来保证操作的原子性，因为缓存一致性机制会阻止同时修改由两个以上处理器缓存的内存区域的数据，其他处理器回写已被锁定的缓存行的数据时，就会使缓存无效。**

锁机制虽然可以保证原子性，但是锁机制会存在以下问题：

- 多线程竞争的情况下，频繁的加锁、释放锁会导致较多的上下文切换和调度延时，性能会很差
- 当一个线程占用时间比较长时，就导致其他需要此锁的线程挂起.

上面我们说的都是悲观锁，要解决这种低效的问题，我们可以采用乐观锁，每次不加锁，而是假设没有冲突去完成某项操作，如果因为冲突失败就重试，直到成功为止。也就是我们接下来要说的CAS(compare and swap).

- CAS(compare and swap)

CAS的全称为`Compare And Swap`，直译就是比较交换。是一条CPU的原子指令，其作用是让`CPU`先进行比较两个值是否相等，然后原子地更新某个位置的值，其实现方式是给予硬件平台的汇编指令，在`intel`的`CPU`中，使用的`cmpxchg`指令，就是说`CAS`是靠硬件实现的，从而在硬件层面提升效率。简述过程是这样：

> 假设包含3个参数内存值(V)、预期原值(A)和新值(B)。`V`表示要更新变量的值，`E`表示预期值，`N`表示新值。仅当`V`值等于`E`值时，才会将`V`的值设为`N`，如果`V`值和`E`值不同，则说明已经有其他线程在做更新，则当前线程什么都不做，最后`CAS`返回当前`V`的真实值。CAS操作时抱着乐观的态度进行的，它总是认为自己可以成功完成操作。基于这样的原理，CAS操作即使没有锁，也可以发现其他线程对于当前线程的干扰。



### 4.14 CPU和内存的关系

当我们执行一个程序时，首先由输入设备向CPU发出操作指令，CPU收到操作指令后，硬盘中对应的程序就会直接加载到内存中，此后，CPU 再对内存进行寻址操作，将加载到内存中的指令翻译出来，而后发送操作信号给操作控制器，实现程序的运行或数据的处理。存在于内存中的目的就是为了`CPU`能够过总线进行寻址，取指令、译码、执行取数据，内存与寄存器交互，然后`CPU`运算，再输出数据至内存。

- `os`：`os`全称为`Operating System`，也就是操作操作系统，是一组主管并控制计算机操作、运用和运行硬件、软件资源和提供公共服务组织用户交互的相互关联的系统软件，同时也是计算机系统的内核与基石。
- 编译器：编译器就是将“一种语言（通常为高级语言）”翻译为“另一种语言（通常为低级语言）”的程序。一个现代编译器的主要工作流程：源代码 (source code) → 预处理器(preprocessor) → 编译器 (compiler) → 目标代码 (object code) → 链接器 (Linker) → 可执行程序(executables)。



### 4.15 父子进程

在一个进程的基础上创建出另一个完全独立的进程，这个两个进程就称为父子进程。

在Linux中，程序员可以通过`pid_t fork()`函数即可为当前进程创建出一个子进程：

![image-20221013191851049](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221013191851049.png)

fork函数如果成功返回，其返回值有两个，但两个返回值并不可能返回给一个进程，分别返回给父子进程：父进程收到的返回值>0，子进程收到的返回值=0；

为什么有父子进程：

子进程与父进程有很强的关联性，但是其运行过程并不影响父进程；因此子进程也可以称为父进程的守护进程，当一个进程需要做一些可能发生阻塞或中断的任务，父进程可通过子进程去做，来避免自己进程的崩溃；

什么是僵尸进程：

子进程先于父进程退出，子进程就会变成僵尸进程；一个进程退出时，会关闭所有的文件描述符，释放在用户空间中分配的内存，但是该进程的PCB仍然会保留，里面还存着进程的退出状态以及统计信息，这些PCB的信息需要该进程的父进程接收（Linux下所有进程都有父进程，即每个进程的PCB都需要由其父进程回收）

> linux下有3个特殊进程，idel进程（PID=0），init进程（PID=1），kthreadd进程（PID=2）
>
> `idle进程（0号进程）`是系统**所有进程**的先祖，内核静态创建的，运行在内核态；这也是唯一一个没有通过fork或者kernel_thread产生的进程；
>
> `init进程（1号进程）` 是系统中所有其它**用户进程**的祖先进程，由0进程创建，完成系统的初始化；
>
> `kthreadd进程（2号进程）`由0进程创建，始终运行在内核空间， 负责所有**内核线程的调度和管理**；

子进程退出时，会为父进程发送sigchld信号

父进程要主动捕捉这个信号才能回收子进程的PCB信息；

当父进程处于运行或者睡眠状态，，是**无法接收子进程的退出信号**, 子进程的退出状态信息无法被回收，其PCB将一直存在于内存（也就是变成所谓的僵尸态），久而久之便会**造成内存泄漏**。

什么是孤儿进程

就是父进程先于子进程退出，子进程就会变成孤儿进程；

孤儿进程的出现流程：

- 父进程先于子进程退出后，回收子进程的父进程就不在了，会使子进程变成孤儿；
- 随即该孤儿进程会马上**被操作系统的1号进程领养**；
- 该进程的PCB回收也由1号进程完成；

孤儿进程的危害：孤儿进程由系统回收，没有危害；



### 4.16 用户态和内核态的区别

https://zhuanlan.zhihu.com/p/447488276

这两个态都是操作系统的运行级别，用户态、内核态的指令都是CPU在执行，CPU指令需要根据其重要程度，也分为不同的权限，有一些指令失败了没什么影响，而有一些指令失败了会导致整个操作系统崩溃，，设置需要重启系统，如果这些指令随意开放给应用程序的话，整个系统崩溃的概率将大大增加；所以我们对CPU指令做了划分，例如intel x86中将CPU指令权限划分为4个等级：

![image-20221013235123724](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221013235123724.png)

它们之间的权限的高低程度可以通过这张图来识别：

![image-20221013235142224](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221013235142224.png)

在 Linux 系统中，由于只有 Ring0 和 Ring3 级别的指令，所以我们可以对用户态、内核态给一个**更细节的区别描述**：运行 Ring0 级别指令的叫**内核态**，运行 Ring3 级别指令的叫**用户态**。

什么态就代表当前的CPU正在执行什么级别的指令，发生系统调用的时候就会发生从用户态切换到内核态；

当用户态的程序向操作系统申请更高权限的服务时，就通过系统调用向内核发起申请，内核自然也会提供很多的接口来供调用，例如**申请动态内存空间**。但是申请了内存是不是还得考虑释放内存？如果把这块内存管理交给应用程序的话，复杂的管理工作会给开发带来很多负担。

所以**库函数**就是用于屏蔽掉内部复杂的细节的，我们的应用程序可以通过库函数来调用内核的提供的接口，而库函数就会发起**系统调用**，发起了系统调用之后，用户态就会切换成内核态去执行对应的内核方法。

> 除了系统调用之外，还有另外两种会导致态的切换：发生异常、中断

内核态：可以访问计算的所有资源：外围设备、硬盘、网卡、内存、CPU资源、存储资源、I/O资源等；向下控制硬件资源，向内管理操作系统资源：包括进程的调度和管理、内存的管理、文件系统的管理、设备驱动程序的管理以及网络资源的管理，向上则向应用程序提供系统调用的接口。

https://juejin.cn/post/6844903646216323086

那到底在什么情况下会发生从用户态到内核态的切换，一般存在以下三种情况：

当然就是系统调用：原因如上的分析；

异常事件： 当 CPU 正在执行运行在用户态的程序时，突然发生某些预先不可知的异常事件，这个时候就会触发从当前用户态执行的进程转向内核态执行相关的异常事件，典型的如缺页异常；

外围设备的中断：当外围设备完成用户的请求操作后，会向 CPU 发出中断信号，此时，CPU 就会暂停执行下一条即将要执行的指令，转而去执行中断信号对应的处理程序，如果先前执行的指令是在用户态下，则自然就发生从用户态到内核态的转换；



### 4.17 什么是缺页异常

https://www.jianshu.com/p/2bd572999ce2

对于操作系统而言，使用分页机制来实现虚拟地址到物理地址的转化，那么何为缺页异常？缺页异常就是想要访问的页不在内存中的情况。对于一个二级页表，包含页目录表和页表，其中页目录项和页表项如下图所示：



![img](https:////upload-images.jianshu.io/upload_images/13526929-e6183a711c46fd18.png?imageMogr2/auto-orient/strip|imageView2/2/w/597/format/webp)

image.png

其中的P位就表示当前页是否在内存中，为1则存在，为0则不存在，当CPU通过查找页表的时候，如果发现该页P位为0，那么会触发缺页中断异常，从而调用中断处理程序，将所缺的页从硬盘上加载到内存中，同时将P为置1。然后CPU会再次访问该页，此时P位为1，正常访问。



### 4.18 reactor是什么

reactor设计模式是一种事件驱动模式，用于处理通过一个或多个输入并发地传递给服务处理程序的服务请求，然后服务处理程序对传入的请求进行多路复用，并将他们同步地分派给相关的请求处理程序。

redis是一个事件驱动程序，有以下1两类事件：

- 文件事件：redis服务器通过套接字与客户端进行连接，文件事件是服务器对套接字操作的抽象，服务器与客户端的通信会产生相应的文件事件，而服务则通过监听并处理这些事件来完成一系列网络通信操作；
- 时间事件：redis中的一些定时操作（serverCron函数）需要在给定时间点执行，时间事件就是这类定时操作的抽象；

redis的网络事件处理器是基于reactor模式，又叫做文件事件处理器；

文件事件处理器使用I/O多路复用来同时监听多个套接字，并根据套接字执行的任务关联到不同的事件处理器；

文件事件以单线程方式运行，但通过使用I/O多路复用程序来监听多个套接字，文件事件处理器实现了高性能的网络通信模型。

![image-20221021084802033](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221021084802033.png)

套接字：文件事件是对套接字的抽象，每当一个套接字准备好执行连接应答、写入、读取、关闭等操作时，就会产生一个文件事件，因为一个服务器通常会连接多个套接字，所以文件事件有可能会并发地出现。

I/O多路复用：I/O多路复用程序负责监听多个套接字，并向文件事件分派器传送那些产生了事件的套接字。I/O多路复用将所有事件的套接字都放入一个队列中，该队列就保证有序同步的方式将套接字向分派器传送套接字。

文件事件分派器：接口I/O多路复用程序传来的套接字，并根据套接字产生的事件的类型，调用相应的事件处理器

事件处理器：服务器会为执行不同任务的套接字关联不同的事件处理器，这些处理器是一个个函数，它们定义了某个事件发生时，服务器应该执行的动作。



## 5. 数据结构

### 5.1 什么是满二叉树

在一颗二叉树中，如果每个节点都存在左子树和右子树，并且所有叶节点都在同一层，这样的树成为满二叉树；

![image-20221004104405974](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221004104405974.png)



### 5.2 什么是完全二叉树

完全二叉树是由满二叉树引出来的，若二叉树的深度为h，除第h层外，其他各层（1~h-1）的结点数都达到最大个数（即1~h-1层为一个满二叉树），第h层所有的结点都连续集中在最左边，这就是完全二叉树；

![image-20221004105416135](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221004105416135.png)

### 5.3 什么是二叉搜索树

它或者是一棵空树，或者是具有下列性质的[二叉树](https://baike.baidu.com/item/二叉树/1602879?fromModule=lemma_inlink)： 若它的左子树不空，则左子树上所有结点的值均小于它的[根结点](https://baike.baidu.com/item/根结点/9795570?fromModule=lemma_inlink)的值； 若它的右子树不空，则右子树上所有结点的值均大于它的根结点的值； 它的左、右子树也分别为[二叉排序树](https://baike.baidu.com/item/二叉排序树/10905079?fromModule=lemma_inlink)。二叉搜索树作为一种经典的数据结构，它既有链表的快速插入与删除操作的特点，又有数组快速查找的优势；所以应用十分广泛，例如在文件系统和数据库系统一般会采用这种数据结构进行高效率的排序与检索操作。



### 5.4 什么是红黑树



### 5.5 什么是二叉搜索树

二叉搜索树它或者是一颗空树，或者具有如下性质的二叉树：若他的左子树不为空，则左子树上所有节点的值均小于它的根节点的值，若他的右子树不为空，则右子树上所有节点的值均大于它的根节点的值；

前序遍历：访问根结点的操作发生在遍历其左右子树之前（根->左子树->右子树）

```go
func preOrder(root *TreeNode) {
    if root == nil{
        return nil
    }
    print root.Val
    preOrder(root.Left)
    preOrder(root.Right)
}
```



中序遍历：访问根结点的操作发生在遍历其左右子树之中（左子树->根->右子树)。

```go
func inOrder(root *TreeNode) {
    if root == nil {
        return nil
    }
    inOrder(root.Left)
    print root.Val
    inOrder(root.Right)
}
```



后序遍历：访问根结点的操作发生在遍历其左右子树之后（左子树->右子树->根）。

```go
func lateOrder(root *TreeNode) {
    if root == nil {
        return nil
    }
    lateOrder(root.Left)
    lateOrder(root.Right)
    print root.Val
}
```





## 6 mysql

### 6.1 介绍一下mysql

mysql是一种关系型数据库，mysql的默认端口号时3306。5.7以后默认存储引擎为InnoDB，因为mysql是开源的，很多大公司都在使用它。



### 6.2 MyISAM和InnoDB的区别

5.5版本之前默认的存储引擎是MyISAM，5.5版本以后默认存储引擎是InnoDB，MYISAM性能极佳，提供了大量特性，包括全文索引、压缩、空间函数等，但MyISAM不支持事务和行级锁，最大的缺点就是崩溃后无法恢复。

大多数情况我们都会选择使用InnoDB存储引擎：

两者对比：

- MyISAM只支持表级锁，而InnoDB支持行级锁和表级锁，默认为行级锁

- MYISAM强调的是性能，每次查询都具有原子性，其执行速度比InnoDB类型更快，但是不提供事务支持，

- 但是**InnoDB** 提供事务⽀持事务，外部键等

  ⾼级数据库功能。 具有事务(commit)、回滚(rollback)和崩溃修复能⼒(crash recovery

  capabilities)的事务安全(transaction-safe (ACID compliant))型表

- MyISAM不支持外键，而InnoDB支持外键

- InnoDB支持MVCC，应对高并发事务，MVCC比单纯的加锁更高效

- MVCC只 在 READ COMMITTED 和 REPEATABLE READ 两个隔离级别下⼯作;MVCC可以使⽤ 乐 

  观(optimistic)锁 和 悲观(pessimistic)锁来实现;各数据库中MVCC实现并不统⼀。
  
- MyISAM无论主键索引还是二级索引都是非聚簇索引，而InnoDB的主键索引是聚簇索引，二级索引是非聚簇索引；

- MyISAM将索引和数据分开存储，将表中的记录”按照记录的插入顺序“单独存储一个文件，称之为数据文件，这个文件并不划分若干个数据页，有多少记录就往这个文件中塞多少记录，因为插入没有大小顺序，所以不能使用二分查找；MyISAM会把索引信息另外存储到一个称为索引文件的另一个文件中，MyISAM会单独为表的主键创建一个索引，只不过在索引的叶子节点中存储的不是完整的用户记录，而是主键值+数据记录地址的组合；



### 6.3 什么是事务

事务是逻辑上的一组操作，要么都执行，要么都不执行；

经典例子就是转账：假如⼩明要给⼩红转账1000元，这个转账会涉及

到两个关键操作就是：将⼩明的余额减少1000元，将⼩红的余额增加1000元。万⼀在这两个操

作之间突然出现错误⽐如银⾏系统崩溃，导致⼩明余额减少⽽⼩红的余额没有增加，这样就不对

了。事务就是保证这两个关键操作要么都成功，要么都要失败。



#### 6.4.1 事务的ACID

1. 原子性：事务是最小的执行单位，不允许分割，事务的原子性确保动作要么全部完成，要么全部失败；
2. 一致性：执行的结果必须是使数据库从一个一致性状态变到另一个一致性状态。因此当数据库只包含成功事务提交的结果时，就说数据库处于一致性状态
3. 隔离性：并发访问数据库时，一个用户的事务不被其他事务干扰，各并发事务之间的数据库是独立的；
4. 持久性：一个事务被提交后，他对数据库中的改变是持久的，即使数据库发生故障也不应该对其有任何影响；



#### 6.4.2 并发事务问题

- 脏读：当⼀个事务正在访问数据并且对数据进⾏了修改，⽽这种修改还没有提

  交到数据库中，这时另外⼀个事务也访问了这个数据，然后使⽤了这个数据。因为这个数据

  是还没有提交的数据，那么另外⼀个事务读到的这个数据是“脏数据”，依据“脏数据”所做的

  操作可能是不正确的

- 丢失修改：指在⼀个事务读取⼀个数据时，另外⼀个事务也访问了该数

  据，那么在第⼀个事务中修改了这个数据后，第⼆个事务也修改了这个数据。这样第⼀个事

  务内的修改结果就被丢失，因此称为丢失修改。 例如：事务1读取某表中的数据A=20，事

  务2也读取A=20，事务1修改A=A-1，事务2也修改A=A-1，最终结果A=19，事务1的修改被丢失

- 不可重复读：指在⼀个事务内多次读同⼀数据。在这个事务还没有结

  束时，另⼀个事务也访问该数据。那么，在第⼀个事务中的两次读数据之间，由于第⼆个事

  务的修改导致第⼀个事务两次读取的数据可能不太⼀样。这就发⽣了在⼀个事务内两次读到

  的数据是不⼀样的情况，因此称为不可重复读

- 幻读：幻读与不可重复读类似。它发⽣在⼀个事务（T1）读取了⼏⾏数

  据，接着另⼀个并发事务（T2）插⼊了⼀些数据时。在随后的查询中，第⼀个事务（T1）

  就会发现多了⼀些原本不存在的记录，就好像发⽣了幻觉⼀样，所以称为幻读。

**不可重复读和幻读区别：**

不可重复读的重点是修改⽐如多次读取⼀条记录发现其中某些列的值被修改，幻读的重点在于新

增或者删除⽐如多次读取⼀条记录发现记录增多或减少了。



#### 6.4.3 事务隔离级别？

四个隔离级别：

**READ-UNCOMMITTED(**读取未提交)：最低的隔离级别，允许读取尚未提交的数据变更，

**可能会导致脏读、幻读或不可重复读**。

**READ-COMMITTED(**读取已提交)：允许读取并发事务已经提交的数据，**可以阻⽌脏读，但**

**是幻读或不可重复读仍有可能发⽣**。

**REPEATABLE-READ(**可重复读)： 对同⼀字段的多次读取结果都是⼀致的，除⾮数据是被

本身事务⾃⼰所修改，**可以阻⽌脏读和不可重复读，但幻读仍有可能发⽣**。

**SERIALIZABLE(**可串⾏化)： 最⾼的隔离级别，完全服从ACID的隔离级别。所有的事务依

次逐个执⾏，这样事务之间就完全不可能产⽣⼲扰，也就是说，**该级别可以防⽌脏读、不可**

**重复读以及幻读**

![image-20221006145850937](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221006145850937.png)

MySQL InnoDB 存储引擎的默认⽀持的隔离级别是 **REPEATABLE-READ**（可重读）。我们可以

通过 SELECT @@tx_isolation; 命令来查看

这⾥需要注意的是：与 SQL 标准不同的地⽅在于 InnoDB 存储引擎在 **REPEATABLE**

**READ**（可重读）

事务隔离级别下使⽤的是Next-Key Lock 锁算法，因此可以避免幻读的产⽣，这与其他数据库系

统(如 SQL Server)

是不同的。所以说InnoDB 存储引擎的默认⽀持的隔离级别是 **REPEATABLE-READ**（可重读）

已经可以完全保证事务的隔离性要求，即达到了

 SQL标准的 **SERIALIZABLE(**可串⾏化) 隔离级别。因为隔离级别越低，事务请求的锁越少，所

以⼤部分数据库系统的隔离级别都是 **READ-COMMITTED(**读取提交内容) ，但是你要知道的是

InnoDB 存储引擎默认使⽤ REPEAaTABLE-READ（可重读） 并不会有任何性能损失。

InnoDB 存储引擎在 **分布式事务** 的情况下⼀般会⽤到 **SERIALIZABLE(**可串⾏化) 隔离级别。



#### 6.4.5 mysql的锁机制

**全局锁：**全局锁就是对整个数据库实例加锁，Mysql提供了一个加全局读锁的方法，命令是Flush tables with read lock。全局锁的典型应用场景是做全库逻辑备份。

**表级锁**：Mysql中 粒度最大的一种锁，对当前操作的整张表加锁，实现简单，资源消耗也比较少，加锁快，不会出现死锁，其中锁定粒度最大，触发锁冲突的概率最高，并发度最低，MyISAM和InnoDB引擎都支持表级锁。表级锁分为 表锁 和 MDL锁，表锁的粒度比较大，一般不被使用，MDL锁是在mysql5.5版本中引入的，不需要显式使用，在访问一个表的时候自动加上，MDL锁可以保证读写的正确性，将对一个表做增删改查的时候，加MDL读锁，当要对表结构变更操作的时候，加MDL写锁，用来保证变更表结构操作的安全性；

**行级锁：**MySQL中锁定 **粒度最⼩** 的⼀种锁，只针对当前操作的⾏进⾏加锁。 ⾏级锁能⼤⼤减少数据库操作的冲突。其加锁粒度最⼩，并发度⾼，但加锁的开销也最⼤，加锁慢，会出现死锁。在InnoDB事务中，行锁是在需要的时候加上的，并不是不需要了就立刻释放，而是要等到事务结束才释放，这个就是两阶段锁协议；

InnoDB支持的行锁有如下几种：

- **Record Lock:** 对索引项加锁，锁定符合条件的行，其他事务不能修改和删除加锁项；
- **Gap Lock：**对索引项之间的间隙锁，锁定记录的范围（对第一条记录前的间隙或最后一条将记录后的间隙加锁），不包含索引项本身。其他事务不能在锁范围内插入数据，这样就防止了别的事务新增幻影行
- **Next-key Lock：**锁定索引项本身和索引范围。即Record Lock和Gap Lock的结合。可解决幻读问题。

**虽然使用行级索具有粒度小、并发度高等特点，但是表级锁有时候也是非常必要的**：

- 事务更新大表中的大部分数据直接使用表级锁效率更高；
- 事务比较复杂，使用行级索很可能引起死锁导致回滚。

表级锁和行级锁可以进一步划分为共享锁（s）和排他锁（X）。

**共享锁（s）**

共享锁（Share Locks，简记为S）又被称为读锁，其他用户可以并发读取数据，但任何事务都不能获取数据上的排他锁，直到已释放所有共享锁。

共享锁(S锁)又称为读锁，若事务T对数据对象A加上S锁，则事务T只能读A；其他事务只能再对A加S锁，而不能加X锁，直到T释放A上的S锁。这就保证了其他事务可以读A，但在T释放A上的S锁之前不能对A做任何修改。

**排他锁（X）：**

排它锁（(Exclusive lock,简记为X锁)）又称为写锁，若事务T对数据对象A加上X锁，则只允许T读取和修改A，其它任何事务都不能再对A加任何类型的锁，直到T释放A上的锁。它防止任何其它事务获取资源上的锁，直到在事务的末尾将资源上的原始锁释放为止。在更新操作(INSERT、UPDATE 或 DELETE)过程中始终应用排它锁。



**死锁检测**：因为InnoDB中的行锁是是在使用时才加锁的，所以多个事务操作同一行就会出现循环等待资源的问题，两种策略来处理这种情况，一种策略是直接进入等待，直到超时，这个超时时间可以通过参数 innodb_lock_wait_timeout来设置，另一种发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行，将参数inno_db_deadlock_detect设置为On，表示开启这个逻辑，死锁检测也是有额外负担的，每当一个事务被所得时候，就要看看他所依赖的线程有没有被锁住，如此循环，就会导致占用CPU资源过高；解决方案是可以修改mysql源码，对于相同行的更新，在进入引擎之前排队，这样在InnoDB内部就不会有大量的死锁检测工作了。



#### 6.4.6 online DDl的过程

1. 拿到MDL写锁
2. 降级成MDL读锁
3. 真正做DDL
4. 升级成MDL写锁
5. 释放MDL锁

#### 6.4.7 InnoDB如何解决幻读

幻读：幻读与不可重复读类似。它发⽣在⼀个事务（T1）读取了⼏⾏数

据，接着另⼀个并发事务（T2）插⼊了⼀些数据时。在随后的查询中，第⼀个事务（T1）

就会发现多了⼀些原本不存在的记录，就好像发⽣了幻觉⼀样，所以称为幻读。

产生幻读的原因是，行锁只能锁住行，但是新插入记录这个动作，要更新的是记录之间的”间隙“，因此，为了解决幻读问题，InnoDB只好引入新的锁，也就是间隙锁；

间隙锁，锁的就是两个值之间的空隙，例如一个表初始化了插入了6个记录，这就产生了7个间隙；

![image-20221006205624521](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221006205624521.png)

间隙锁不一样，跟间隙锁存在冲突关系的，是"往这个间隙中插入一个记录"这个操作，间隙锁之间都不存在冲突关系；

间隙锁和行锁合称为next-key lock，每个next-key lock是前开后闭合区间，间隙锁只在可重复读隔离级别下有效；

加锁规则：

1. 原则1：加锁的基本单位是next-key lock. next-key lock是前开后闭区间；
2. 原则2：查找过程中访问到的对象才会加锁；
3. 优化1：索引上的等值查询，给唯一索引加锁的时候，next-key lock退化为行锁
4. 优化2：索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，next-key lock退化为间隙锁
5. 一个bug：唯一索引上的范围查询会访问到不满足条件的第一个值为止；


### 6.4 InnoDB的索引模型

索引的出现其实就是为了提高数据查询的效率；

在InnoDB中，表都是根据主键顺序以索引的形式存放的，这种存储方式的表称为索引组织表，InnoDB使用了B+树索引模型，所以数据都是存储在B+树中的；

每一个索引在InnoDB里面对应一棵B+树；

主键索引的叶子节点存的是整行数据，在InnoDB里，主键索引也被称为聚簇索引；

非主键索引的叶子节点内容是主键的值，在InnoDB里，非主键索引也被称为二级索引；

基于主键索引和普通索引的查询方式有什么区别？

基于非主键索引的查询需要多扫描一棵索引树，也就是回表；



#### 6.4.1 覆盖索引

因为非主键索引的叶子节点内容是主键的值，所以使用非主键索引查询数据时需要进行回表，有些场景会导致我们回表的次数增加，所以我们可以考虑使用覆盖索引进行优化，因为非主键索引的叶子节点内容是主键的值，所以我们可以先查询出来主键值，然后再根据主键值取查询我们的数据，这就是覆盖索引，由于覆盖索引可以减少树的搜索次数，显著提升查询性能，所以使用覆盖索引是一个常用的性能优化手段；



#### 6.4.2 最左前缀原则

B+树这种索引结构，可以利用索引的”最左前缀“来定位记录；这个最左前缀可以是联合索引的最左N个字段，也可以是字符串索引的最左M个字符；

当创建(a,b,c)复合索引时，想要索引生效的话，只能使用 a和ab、ac和abc三种组合！

在建立联合索引的时候，如何安排索引内的字段顺序？

第一原则，如果通过调整顺序，可以少维护一个索引，那么这个顺序往往就是需要优先考虑采用的；

第二原则，考虑空间大小；

索引下推优化：在mysql5.6以后引入的索引下推优化，可以在索引遍历过程中，对索引中包含的字段先做判断，直到过滤掉不满足条件的记录，减少回表次数；



### 6.5 查询缓存

Mysql支持查询缓存，不过在mysql8.0版本后移除，因为这个功能不太实用，通过配置参数可以设置缓存类型以及缓存空间，开启查询缓存后同样的查询条件以及数据情况下，会直接在缓冲中返回结果，这⾥的查询条件包括查询本身、当前要查询的数据库、客户端协议版本号等⼀些可能影响结果的信息。因此任何两个查询在任何字符上的不同都会导致缓存不命中。此外，如果查询中包含任何⽤户⾃定义函数、存储函数、⽤户变量、临时表、MySQL库中的系统表，其查询结果也不会被缓存。缓存建⽴之后，MySQL的查询缓存系统会跟踪查询中涉及的每张表，如果这些表（数据或结构）发⽣变化，那么和这张表相关的所有缓存数据都将失效。

**缓存虽然能够提升数据库的查询性能，但是缓存同时也带来了额外的开销，每次查询后都要做⼀次缓存操作，失效后还要销毁。** 因此，开启缓存查询要谨慎，尤其对于写密集的应⽤来说更是如此。如果开启，要注意合理控制缓存空间⼤⼩，⼀般来说其⼤⼩设置为⼏⼗MB⽐᫾合适。此外，**还可以通过sql_cache和sql_no_cache来控制某个查询语句是否需要缓存**



### 6.6 一条SQL语句在MySQL中是如何执行的

来自极客时间：MySQL45讲

![image-20221007114427364](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221007114427364.png)

Mysql主要分为两层：server层和存储引擎层两部分；

server层包括连接器、分析器、优化器、执行器、查询缓存，涵盖mysql的大多数核心服务功能，以及所有的内置函数（如日期、时间、数学和加密函数等），所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等；

存储引擎层负责数据的存储和提取，其架构模式是插件式的，支持InnoDB、MyISAM、Memory等多个存储引擎，Mysql5.5版本以后默认的存储引擎就是InnoDB；

一条SQL语句执行流程如下：

**连接器**

第一步需要先连接到这个数据库上，这个时候接待我们的就是连机器，连接器负责和客户端建立连接、获取权限、维持和管理连接：mysql -h -P -u -p;  连接完成后，如果没有后续动作，这个连接就处于空闲状态，客户端如果太长时间没有动静，连接器就会自动将它断开；

**查询缓存**

连接建立成功后，就可以执行sql语句了，执行第二部分逻辑是查询缓存，mysql会先到查询缓存中看看，之前是不是执行过这条语句，使用key-value的方式缓存，key是查询语句，value是查询的结果，如果查询在缓存中就直接返回给客户端，如果不再就会走接下来的操作；

不建议使用查询缓存，查询缓存弊大于利，查询缓存失效非常频繁，只要对一个表的更新，这个表上的所有查询缓存都会被清空，对于写密集的应用，查询缓存命中率会非常的低；

**分析器**

没有命中查询缓存，就要真正执行语句了，在分析器中做"词法"分析，比如你输入的是由多个字符串和空格组成的一条SQL语句，MySQL需要识别出里面的字符串分别是什么，代表什么，做完这些识别后，就要进行"语法"分析，根据”词法“分析的结果，”语法“分析器会根据语法规则，判断你输入的这个SQL语句是否满足Mysql语法；

**优化器**

经过了分析器，Mysql直到你要做什么了，在开始执行之前，还要经过优化器的处理，优化器是在表里面有多个索引的时候，决定使用哪个索引，或者在一个语句有多涨表关联的时候，决定各个表的连接顺序；

**执行器**

Mysql通过优化器后知道该怎么做了，于是就进入了执行器阶段，开始执行语句，开始执行的时候，要先判断一下你对这个表有没有执行查询的权限，如果没有就会返回没有权限的错误（如果命中查询缓存，会在查询缓存返回结果的时候做权限验证），如果有权限，就打开表继续执行；打开表的时候，执行器就会根据表的引擎定义，去使用这个引擎提供的接口。



### 6.7 redo日志

在MySQL里，如果每一次的更新操作都需要写进入磁盘，然后磁盘也要找到对应的那条记录，然后在更新，整个过程IO成本，查找成本都很高，为了解决这个问题，MySQL使用WAL技术（Write-Ahead-logging），他的关键点就是先写日志，再写磁盘，当一条记录需要更新的时候，InnoDB引擎就会把记录写到redo log里面，并更新内存，这个时候就算更新完成了，同时，InnoDB引擎会在适当的时候，将这个操作记录更新到磁盘里面，而这个更新往往是在系统比较空闲的时候做；InnoDB的redo log是固定大小的，比如可以配置为一组4个文件，每个文件的大小是1GB，从头开始写，写道末尾又回到开头循环写；

![image-20221007144146603](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221007144146603.png)

write pos是当前记录的位置，一边写一边后移，写到第3号文件末尾后就回到0号文件开头，checkpoint是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要把记录更新到数据文件；如果writePos追上checkpoint了，那么就要暂停一下执行新的更新，得先擦掉一些记录；有了redo log，InnoDB就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为crash-safe。

redo log日志InnoDB引擎特有的日志；

### 6.8 binlog日志

Mysql主要分为两层：server层和引擎层，redo log是InnoDB引擎特有的日志，server层也有自己的日志，称为binlog日志；binlog有两种记录模式，一种是statement格式，这种就记录sql语句，另外一种是row格式记录行的内容，记两条，更新前和更新后都有；

**为什么会有两个日志？**

Mysql自带的引擎是MyISAM，但是MyISAM没有crase-safe能力，binlog只能用于归档，InnoDB是另一个公司以插件形式引入Mysql的，既然只能依靠binlog是没有carsh-safe能力的，所以InnoDB使用另外一套日志系统，也就是redo log来实现crash-safe能力；

**redo log和binlog有什么区别？**

1. redo log是InnoDB引擎特有的，binlog是Mysql的server层实现的，所有引擎都可以使用
2. redo log是物理日志，记录的是"在某个数据页上做了什么修改"，binlog是逻辑日志，记录这个语句的原始逻辑，比如"给ID=2这一行的c字段加1"
3. redo log是循环写的，空间固定会用完，binlog是可以追加写入的，"追加写"是指binlog文件写到一定大小后会切换到下一个，并不会覆盖以前的日志；



### 6.9 redo log、binlog的两阶段提交

binlog会记录所有的逻辑操作，并且采用"追加的"方式，所以使用binlog可以做全库备份，redolog是物理日志，记录对数据页的更改，因为这两个日志在不同的层，所以需要两阶段提交来维持数据逻辑一致性；

两阶段提交是先写binlog再写redolog，这样即使在数据库发生异常重启后，因为binlog记录了日志，那么就可以继续恢复这个事务；

如果先写redo log、再写binlog，那么当mysql进程异常重启后，因为binlog没有carsh-safe能力，使用binlog恢复临时库的时候，就会造成恢复出来的只与原库值不同；

### 6.10 大表优化

[mysql - MySQL大表优化方案_个人文章 - SegmentFault 思否](https://segmentfault.com/a/1190000006158186)

当MySQL单表记录数过大时，数据库的CRUD性能会明显下降，一些常见的优化措施：

**读/写分离：**主库负责写，从库负责读，核心业务场景为了准确性、一致性要使用主库

**查询限定数据的范围：**务必禁止不带任何限制数据范围条件的查询语句，比如：我们当用户正在查询历史订单的时候，我们可以控制在一个月的范围内；

**垂直分区：**根据数据库里面数据表的相关性进行拆分，例如：订单信息中既包括订单信息又包括商家结算信息，那么就可以拆分成两个表，甚至可以拆分到单独的库做，垂直分区就是把一张列比较多的表拆分为多张表，垂直分区的有点：可以使得列数据变小，在查询时减少读取的block数，减少IO次数，简化表结构，垂直拆分的缺点：主键会出现冗余，需要管理冗余队列，并会引起Join操作，可以通过在应用层进行Join解决，此外，垂直分区会让事务变得更加复杂；

**水平分区：**保持数据表结构不变，通过某种策略存储数据分片，这样每一片数据分散到不同的表或者库中，达到了分布式的目的，水平拆分可以支撑非常大的数据量。分表仅仅是解决了单⼀表数据过⼤的问题，但由于表的数据还是在同⼀台机器上，其实对于提升MySQL并发能⼒没有什么意义，所以**⽔平拆分最好分库** 。⽔平拆分能够 **⽀持⾮常⼤的数据量存储，应⽤端改造也少**，但 **分⽚事务难以解决** ，跨节点Join性能᫾差，逻辑复杂。《Java⼯程师修炼之道》的作者推荐 **尽量不要对数据进⾏分⽚，因为拆分会带来逻辑、部署、运维的各种复杂度** ，⼀般的数据表在优化得当的情况下⽀撑千万以下的数据量是没有太⼤问题的。如果实在要分⽚，尽量选择客户端分⽚架构，这样可以减少⼀次和中间件的⽹络I/O。



### 6.11 explain介绍一下

https://segmentfault.com/a/1190000008131735

Mysql提供了一个Explain命令，它可以对select语句进行分析，并输出select执行的详细信息，以针对开发人员进行性能优化，Explain输出内容大概如下：

```sql
mysql> explain select * from user_info where id = 2\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: user_info
   partitions: NULL
         type: const
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 8
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```

各列的含义如下:

- id: SELECT 查询的标识符. 每个 SELECT 都会自动分配一个唯一的标识符.
- select_type: SELECT 查询的类型.SIMPLE, 表示此查询不包含 UNION 查询或子查询，UNION表的查询是最外层的查询；
- table: 查询的是哪个表
- partitions: 匹配的分区
- type: 提供了判断查询是否高效的重要依据，通过type字段，我们判断此次查询是全表扫描还是索引扫描；index是全索引扫描，ALL全表扫描，range是使用索引范围查询，const是针对主键或唯一索引的等值查询扫描；
- possible_keys: 此次查询中可能选用的索引
- key: 此次查询中确切使用到的索引.
- ref: 哪个字段或常数与 key 一起被使用
- rows: 显示此查询一共扫描了多少行. 这个是一个估计值.
- filtered: 表示此查询条件所过滤的数据的百分比
- extra: 额外的信息

接下来我们来重点看一下比较重要



### 6.12 数据库连接池

池化设计会初始化预设资源，解决的问题就是抵消每次获取资源的消耗，如创建线程的开销，获取远程连接的开销，池化设计还包括如下特征：池子的初始值，池子的活跃值，池子的最大值等；

数据库连接本质就是一个socket连接，数据库服务端还要维护一些缓存和用户权限信息之类的，所以占用了一些内存，我们可以把数据库连接池看作是维护数据库连接的缓存，以便将来需要对数据库的请求时可以重用这些连接，为每个用户打开和维护数据库连接，尤其时对动态数据库驱动的网站应用程序的请求，既昂贵又浪费资源，在连接池中，创建连接后，将其放置在池中，并再次使用它，因此不必建立新的连接，如果使用了所有连接，则会建立一个新连接并将其添加到池中，连接池还减少了用户必须等待建立与数据库连接的时间；

数据库连接池负责分配、管理和释放数据库连接，它允许应用程序重复使用一个现有的数据库连接，而不是重新建立一个；

### 6.13 MVCC版本并发控制

https://www.cnblogs.com/jelly12345/p/14889331.html

MVCC即多版本并发控制，MVCC 在mysql InnoDB中的实现主要是为了提高数据库并发性能，用更好的方式去处理读-写冲突，做到即使有读写冲突时，也能做到不加锁，非阻塞并发读；

当前读：：像select lock in share mode(共享锁), select for update ; update, insert ,delete(排他锁)这些操作都是一种当前读，为什么叫当前读？就是它读取的是记录的最新版本，读取时还要保证其他并发事务不能修改当前记录，会对读取的记录进行加锁。

快照读：像不加锁的select操作就是快照读，即不加锁的非阻塞读；快照读的前提是隔离级别不是串行级别，串行级别下的快照读会退化成当前读；之所以出现快照读的情况，是基于提高并发性能的考虑，快照读的实现是基于多版本并发控制，即MVCC,可以认为MVCC是行锁的一个变种，但它在很多情况下，避免了加锁操作，降低了开销；既然是基于多版本，即快照读可能读到的并不一定是数据的最新版本，而有可能是之前的历史版本



### 6.14 ID主键方案选择

生成全局ID的几种方式：

- UUID：不适合做主键，因为太长了，并且无序不可读，查询效率低，比较适合用于生成唯一的名字的标识，比如文件的名字

- 数据自增ID：利用数据库的auto_increasement，这种生成的ID有序，但是会有性能瓶颈；

- 利用redis生成ID：利用redis的incr可以生成递增的ID，性能比较好，并且灵活方案，但是编码比较复杂，增加了系统成本

- 雪花算法：Twitter开源的雪花算法是由64位整数组成的分布式ID生成算法，特性如下：

  1. 全局唯一，保证不会出现重复ID(除非时钟回溯)
  2. 递增性，生成的ID是有序递增的，不像UUID一样无序
  3. 高性能，内存操作
  4. 高可用，确认任何时刻都能生成正确的ID 

- 美团的Leaf分布式ID生成系统：Leaf是美团开源的分布式ID生成器，能保证全局唯一性、趋势递增、单调递增、信息安全：https://tech.meituan.com/2017/04/21/mt-leaf.html

### 6.15 一条SQL语句执行的很慢的原因有哪些？

**一：偶尔很慢的情况**

**1.1 数据库在刷新脏页（flush）**

当我们要往数据库插入一条数据，或者要更新一条数据的时候，我们知道数据库会在内存中把对应的字段的数据更新了，但是更新之后，这些更新的字段并不会马上同步持久化到磁盘中取，而是把这些更新的记录写入到redo log日志中去，等到空闲的时候，在通过redo log里的日志把最新的数据同步到磁盘中取。

> 当内存数据页跟磁盘数据页内容不一致的时候，我们称这个内存页为"脏页"，内存数据写入磁盘后，内存和磁盘上的数据页的内容就一致了，称为"干净页"。

刷脏页有下面四种场景：

- redo log写满了：redo log里的容量是有限的，如果数据库一直很忙，更新又很频繁，这个时候redo log很快就会被写满了，这个时候就没办法等到空闲的时候再把数据同步到磁盘的，只能暂停其他操作，全身心来把数据同步到磁盘中取的，而这个时候，就会导致我们平时正常的SQL语句突然执行的很慢，所以说，数据库再同步数据到磁盘的时候，就有可能导致我们的sql语句执行的很慢；
- 内存不够用了：如果一次查询较多的数据，恰好碰到所查数据页不在内存中时，需要申请内存，而此时恰好内存不足的时候就需要淘汰一部分内存数据页，如果是干净页，就直接释放，如果恰好是脏页就需要刷脏页。

**1.2 拿不到锁**

在执行这条语句获取锁阻塞了，表锁、行锁、MDL锁都有可能，可以使用show processlist命令来查看当前的状态；

**二：一直很慢的情况**

**1.1  没有走索引**

1. 字段没有索引
2. 字段有索引，但却没有用索引
3. 函数操作导致没有用上索引

**1.2  数据库优化导致SQL语句不走索引或选错索引**

例如这条SQL语句：select * from t where 100 < c and c < 100000;

系统在执行这条语句的时候，会进行预测：究竟是走c索引扫描的行数少，还是直接扫描全表扫描的行数少呢，所以扫描行数越少当然越好，因为扫描行数越少，意味着IO操作的次数越少；

mysql系统会根据索引的区分度来判断索引预测的，索引区分度也是采样进行的，所以mysql优化导致不走索引也不一定是正确的，系统判断是否走索引，扫描行数的预测只是原因之一，还跟使用临时表，是否需要排序等也有关系，也是会有影响的。



### 6.16 B+树的优势

B树结构如下：

![image-20221014160147115](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221014160147115.png)

B+树结构：

![image-20221014161651850](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221014161651850.png)

B树是一种树状数据结构，它能够存储数据，对其进行排序并允许以O(log n)得时间复杂度进行查找、顺序读取、插入和删除得数据结构，B树符合如下特点：

- 根节点至少有两个子节点
- 非根节点至少有M/2个子节点
- 每个节点中得关键字都按照升序排列，每个关键字得左子树中所有关键字都小于它，而右子树中所有关键字都大于它
- 每个节点都保存索引和数据，也就是对应的key和value
- 每个节点最多有m-1个关键字（可以存有的键值对）。

B+树是对B树得一种变形树，他与B树得差异在于：

- 有k个子节点得节点必然有k个关键码
- 非叶子节点仅具有索引作用，跟记录有关得信息均存放在叶结点中
- 树得所有叶节点构成一个有序链表，可以按照关键码排序得次序遍历全部记录

B+树得优点在于：

- 由于B+树得内部节点上不包含数据信息，因此在内存页中能够存放更多得key，数据存放的更加紧密，具有更好的空间局部性。因此访问叶子节点上关联的数据也具有更好的缓存命中率。
- B+树的叶子结点都是相链的，因此对整棵树的便利只需要一次线性遍历叶子结点即可。而且由于数据顺序排列并且相连，所以便于区间查找和搜索。而B树则需要进行每一层的递归遍历。相邻的元素可能在内存中不相邻，所以缓存命中性没有B+树好。

B树的优点：

- B树的每一个节点都包含key和value，因此经常访问的元素可能离根节点更近，因此访问也更迅速；



#### 6.16.1 为什么说mysql数据库单表最大两千万

https://mp.weixin.qq.com/s/XX_NkIIf_PLyU4IE6lEEYQ

数据在硬盘上存放到user.ibd文件下，在user.ibd文件里他们被分成了很多小份的数据页，每份大小16k，每个页大概内容如下：

![image-20221014190052479](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221014190052479.png)



B+每个节点就是一个数据页：

![image-20221014201244525](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221014201244525.png)

从这个可以看出B+树的最末级叶子节点放了行记录，而非叶子点放了加速查询的索引数据：

假设：

- 非叶子节点内指向其他内存页的指针数量为x
- 叶子节点内能容纳的record数量为Y
- B+树的层数为z

那这棵B+树放的**行数据总量**等于 `(x ^ (z-1)) * y`。

我们假设主键8byte，页号（指针）在源码里叫`FIL_PAGE_OFFSET（4Byte）`，那么非叶子节点里的一条数据是`12Byte`左右。

整个数据页`16k`， 页头页尾那部分数据全加起来大概`128Byte`，加上页目录毛估占`1k`吧。那剩下的**15k**除以`12Byte`，等于`1280`，也就是可以指向**x=1280页**。

所以一个节点扇出是1280个节点；

叶子节点和非叶子节点的数据结构是一样的，所以也假设剩下`15kb`可以发挥。

叶子节点里放的是真正的行数据。假设一条行数据`1kb`，所以一页里能放**y=15行**。

已知`x=1280`，`y=15`。

假设B+树是**两层**，那`z=2`。则是`(1280 ^ (2-1)) * 15 ≈ 2w`

假设B+树是**三层**，那`z=3`。则是`(1280 ^ (3-1)) * 15 ≈ 2.5kw`

**这个2.5kw，就是我们常说的单表建议最大行数2kw的由来。**毕竟再加一层，数据就大得有点离谱了。三层数据页对应最多三次磁盘IO，也比较合理。

上面假设单行数据用了1kb，所以一个数据页能放个15行数据。

如果我单行数据用不了这么多，比如只用了`250byte`。那么单个数据页能放60行数据。

那同样是三层B+树，单表支持的行数就是 `(1280 ^ (3-1)) * 60 ≈ 1个亿`。

你看我一个亿的数据，其实也就三层B+树，在这个B+树里要查到某行数据，最多也是三次磁盘IO。所以并不慢。



#### 6.16.2 为什么不选择B树

B 树能够在非叶节点中存储数据，但是这也导致在查询连续数据时可能会带来更多的随机 I/O；而 B+ 树的所有叶节点可以通过指针相互连接，能够减少顺序遍历时产生的额外随机 I/O。B+ 树的扇出率较大，树高较小，因而在进行索引搜索的时候需要进行的 IO 也较其他树的少。B+ 树只有叶节点会存储数据，将树中的每一个叶节点通过指针连接起来就能实现顺序遍历，而遍历数据在关系型数据库中非常常见，所以这么选择是完全没有问题的。　



#### 6.16.3 为什么不选择跳表

　跳表是一种采用了用空间换时间思想的数据结构。它会随机地将一些节点提升到更高的层次，以创建一种逐层的数据结构，以提高操作的速度。在理论上能够在 O(log(n))时间内完成查找、插入、删除操作。跳表是Redis ZSET类型实现的一种重要数据结构。

**跳表的性质**

1. 由很多层结构组成，level是通过一定的概率随机产生的
2. 每一层都是一个有序的链表，默认是升序的，也可以根据创建映射时所提供的comparator进行排序
3. 最底层的链表包含所有元素，
4. 如果一个元素出现在level i的链表中，则他在level i之下的链表也都会出现；
5. 每个节点包含两个指针，一个指向同一个链表中的下一个元素，一个指向下面一层的元素；

优势：

1. 跳表比B树/B+树占用的内存更小
2. 以链表的形式遍历跳跃表，跳跃表的缓存局部性与其他类型的平衡树相当
3. 跳表更容易实现、调试等；



**为什么选择B+树**

https://www.isolves.com/it/sjk/MYSQL/2022-04-18/53124.html

**B+树**是多叉树结构，每个结点都是一个16k的数据页，能存放较多索引信息，所以**扇出很高**。**三层**左右就可以存储2kw左右的数据（知道结论就行，想知道原因可以看之前的文章）。也就是说查询一次数据，如果这些数据页都在磁盘里，那么最多需要查询**三次磁盘IO**。

**跳表**是链表结构，一条数据一个结点，如果最底层要存放2kw数据，且每次查询都要能达到**二分查找**的效果，2kw大概在2的24次方左右，所以，跳表大概高度在**24层**左右。最坏情况下，这24层数据会分散在不同的数据页里，也即是查一次数据会经历**24次磁盘IO**。

因此存放同样量级的数据，B+树的高度比跳表的要少，如果放在mysql数据库上来说，就是**磁盘IO次数更少，因此B+树查询更快**。

而针对**写操作**，B+树需要拆分合并索引数据页，跳表则独立插入，并根据随机函数确定层数，没有旋转和维持平衡的开销，因此**跳表的写入性能会比B+树要好。**



**redis为什么使用跳表而不使用B+树或二叉树呢？**

redis是纯内存数据库，进行读写数据都是操作内存，所以就减少了磁盘IO操作，所以跳表更具有优势，并且写入性能也比较高，不用像B+为了保持平衡进行旋转；








### 6.17 Mysql写操作

InnoDB存储引擎维护了buffer pool，buffer pool存放的是数据页·，一个数据页16k大小，一次性至少读取1页的数据到内存中或将1页数据写入磁盘，页是InnoDb中管理数据的最小单元，我们往mysql插入的数据都是存在页中的，页与页之间通过一个双向链表连接起来；

mysql更新时先判断数据页是否在缓存中，没有就会发生缺页中断从磁盘中读入内存，然后将更新的数据更新到内存，写入redolog处于prepare阶段、写入binlog，提交事务；

1. 如果是正常运行的实例，数据页被修改以后，跟磁盘页的数据页不一致，称为脏页，最终数据落盘，就是把内存中的数据页写盘，这个过程与redo log毫无关系；
2. 在崩溃恢复场景中，innoDB如果判断到一个数据页可能在崩溃恢复的时候丢失了更新，就会将它读到内存，然后让redo log更新内存内容，更新完成后，内存变成脏页；

redo log buffer是一块内存，用来先存redo日志的，也就是说，在执行第一个insert的时候，数据的内存被修改了，redo log buffer也写入了日志，但是真正把日志写入到redo文件是在执行commit语句的时候做的。

刷新脏页的场景：

1. redo log写满了，需要flush脏页，这种情况是InnoDB要尽量避免的，因为出现这种情况的时候，整个系统就不再能接受更新了，所有的更新都必须堵住；

2. 内存不够用了，要先将脏页写到磁盘，InnoDB用缓冲池管理内存，缓冲池中的内存页有三种状态：

   1. 第一种是还没有使用的
   2. 第二种是使用了并且是干净页
   3. 第三种是使用了并且是脏页

   InnoDB刷脏页的控制策略：

   1. InnoDB需要知道所在主机的IO能力，这样InnoDb才能知道需要全力刷脏页的时候，可以刷多块；使用到的是innodb_io_capacity这个参数，他会告诉InnoDB你的磁盘能力，这个我建议你设置成磁盘的IOPS；
   2. InnoDB需要知道脏页比例和redo log写盘速度，脏页比例上限默认值时75%，InnoDB会根据当前的脏页比列（假设M），算出一个范围在0到100之间的数字，然后根据当前写入的序号跟checkpoint对应的序号之间的差值，我们假设为N，M与N取较大的值记为R，然后按照R%等速度刷脏页

![image-20221014153815645](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221014153815645.png)

undoLog：存在的意义是确保数据库事务的原子性，原子性是指事务是一个不可分割的工作单位，事务中的操作要么都发生，要么都不发生。
redo log记录了事务的行为，可以很好地保证一致性，对数据进行“重做”操作。但事务有时还需要进行“回滚”操作，这时就需要undo log。当我们对记录做了变更操作的时候就需要产生undo log，其中记录的是老版本的数据，当旧事务需要读取数据时，可以顺着undo链找到满足其可见性的记录。
undo log通常以逻辑日志的形式存在。我们可以认为当delete一条记录时，undo log会产生一条对应的insert记录，反之亦然。当update一条记录时，会产生一条相反的update记录。
undo log采用段segment的方式来记录，每个undo操作在记录的时候占用一个undo log segment。
undo log也会产生redo log，因为undo log也要实现持久性保护。







## 7 redis八股文

### 7.1 介绍一下Redis

Redis是一个nosql数据库，使用C语言开发，与传统数据库不同的是Redis的数据是存在内存中的，他是内存数据库，读写速度非常快，因此redis被广泛应用于缓存方向；redis除了做缓存之外，也经常用来做分布式锁，甚至是消息队列，Redis提供了多种数据类型来支持不同的业务场景，Redis还支持事务、持久化、Lua脚本、多种集群方案；

- 性能极高，
- 支持数据的持久化，对数据的更新采用copy-on-write技术，异步保存到磁盘上
- 丰富的数据类型：string、list、hash、set、zset
- 原子性，Redis的所有操作都是原子性的，多个操作可以通过multi和exec指令支持事务
- 丰富的特性：key过期，publish/subscribe、notify
- 支持数据的备份，快速的主从复制
- 节点集群，很容易将数据分布到多个redis实例中；



### 7.2 redis和memcached的相同点？

1. 都是基于内存的数据库，一般都当作缓存使用
2. 都有过期策略
3. 两者的性能都非常高



### 7.3 redis与memcached的区别

- 存储方式Memecached把数据全部存在内存中，断电后会挂掉，数据不能超过内存大小，Redis有部分存在硬盘上，这样能保证数据的持久性；
- Redis支持更丰富的数据类型，Redis不仅仅支持简单的k/v类型的数据，同时还提供list、set、zset、hash等数据结构的存储，memcached只支持最简单的k/v数据类型；
- redis有灾难恢复机制，因为可以把缓存中的数据持久化到磁盘上；
- Memcached是多线程，非阻塞IO复用的网络模型，memcached采用的时master线程-worked线程的模型，执行逻辑是在worker线程里实现了真正的线程隔离，Redis使用单线程的多路IO复用模型（Redis6.0引入了多线程IO），在6.0以后也引入多线程，也采用了master线程-worker线程的模型，不过redis把处理逻辑交还给master线程，执行命令还是单线程执行的，也解决了线程并发安全的问题；
- Redis支持发布订阅模型、Lua脚本、事务等功能，而Memcached不支持，并且。redis支持更多的编程语言；
- Memcached过期的数据删除策略只用了惰性删除，而redis同时使用了惰性删除与定期删除；
- Redis在服务器内存使用完之后，可以将不用的数据放到磁盘上，但是memecached在服务器内存使用完之后，就会直接报异常；
- Memcached没有原生的集群模式，需要依靠客户端来实现往集群中分片写入数据，但是Redis原生支持cluster模式的；

### 7.4 string类型

string数据结构是简单的key-value类型，redis虽然使用C语言写的，但是redis并没有使用C的字符串表示，而是自己构建了一种 **简单动态字符串（SDS）**，Redis的SDS不光可以保存文本数据还可以保存二进制数据，并且获取字符串长度复杂度为O(1)；

**为什么要自己设计SDS字符串？**

C语言中，字符串可以用一个\0结尾的char数组来表示，这种简单的字符串表示在大多数情况下都能满足要求，但是它并不能高效地支持长度计算和追加（append）这两种操作，每次计算字符串长度（strlen(s)）的时间复杂度为O（N），对字符串进行N次追加，必定需要对字符串进行N次内存重分配（realloc）；Redis除了处理C字符串之外，还需要处理单纯的字节数组，以及服务器协议等内容，所以为了方便起见，Redis的字符串表示还应该市二进制安全的：程序不应该对字符串里面保存的数据做任何假设，数据可以是\0结尾的C字符串，也可以是单纯的字节数组，或者其他格式的数据；

考虑到这些原因，Redis使用sds类型替换了C语言的默认字符串表示，sds既可以高效地实现追加和长度计算，并且他还是二进制安全的；

**SDS的实现结构**

![image-20221007224614394](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221007224614394.png)

sds是char *的别名，结构sdshdr保存了len、free和buf三个属性；

len直接记录了已占用的长度，所以获取字符串长度的时间复杂度为O(1)

free可以减少追加（append）操作所需的内存重分配次数，第一写入字符串是，free的值为0，当执行append操作时，追加后字符串的长度如果 小于 SDS_MAX_PREALLOC(1024  * 1024)的值时，就分配多于所需大小一倍的空间，当追加后字符串的长度大于1MB，那么就为他们多分配1MB的空间；

**这种分配会浪费内存吗？**

执行过APPEND命令的字符串会带有额外的预分配空间，这些预分配空间不会被释放，除非该字符串所对应的键被删除，或者等到关闭Redis之后，再次启动时重新载入的字符串对象将不会有预分配空间；

因为执行APPEND命令的字符串键数量通常并不多，占用内存的体积通常也不大，所以这一般并不算什么问题；另一位方面，如果执行APPEND操作的键很多，而字符串的体积又很大的话，那可能就需要修改redis服务器，让他定时释放一些字符串键的预分配空间，从而更有效地使用内存；



### 7.5 RDB（快照）

Redis分别提供了RDB和AOF两种持久化模式，AOF文件的保存频率通常要高于RDB文件的保存频率，所以一般来说，AOF文件的数据会比RDB文件中的数据要新；因此在服务器启动时，打开了AOF功能，那么程序优先使用AOF文件来还原数据，只有AOF功能未被打开的情况下，Redis才会使用RDB文件来还原数据；

RDB时redis存储在内存里面的数据在某个时间点上的副本，redis创建快照后，可以对快照进行备份，RDB将数据库的快照以二进制的方式保存到磁盘中；

在 Redis 运行时，RDB 程序将当前内存中的数据库快照保存到磁盘文件中，在 Redis 重启动时，RDB 程序可以通过载入 RDB 文件来还原数据库的状态。可以将快照复制到其他服务器从⽽创建具有相同数据的服务器副本

（Redis 主从结构，主要⽤来提⾼ Redis 性能），还可以将快照留在原地以便重启服务器的时候

使⽤。

RDB 功能最核心的是 rdbSave 和 rdbLoad 两个函数，前者用于生成 RDB 文件到磁盘，而后者则用于将 RDB 文件中的数据重新载入到内存中；

![image-20221007230213761](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221007230213761.png)

- SAVE命令直接调用rdbSave，阻塞Redis主进程，BGSAVE用子进程调用rdbSave，主进程仍可继续处理命令请求；
- SAVE执行期间，AOF写入可以在后台线程进行，BGREWRITEAOF可以在子进程进行，所以这三种操作可以同时进行；
- 为了避免产生竞争条件，BGSAVE执行时，SAVE命令不能执行
- 为了避免性能问题，BGSAVE和BGREWRITEAOF不能同时执行
- 调用rdbLoad函数载入RDB文件时，不能进行任何和数据库相关的操作，不过订阅与发布方面的命令可以正常执行，因为他们和数据库不想关联；
- RDB文件的组织方式如下：

![image-20221007231954017](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221007231954017.png)

**全量快照：**

![image-20221017150959858](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221017150959858.png)

快照开始时，主线程会fork出一个用于快照操作的子线程，同时子进程会复制一份数据对应的映射页表给子进程，子进程通过这个映射页表可以访问主线程的原始数据，然后将数据生成快照文件，因为我们存储磁盘的时间是比较长的，当这个时候有请求进来时，这个时候要用到写复制，主线程会把新数据或修改后写到一个新的物理内存地址上，并修改主线程自己的页表映射，这样父子进程不相互影响；

**增量快照：**

 如果一直使用全量同步，一方面时间的推移，磁盘存储的快照文件会越来越多。另一方面如果频繁的进行全量同步，则需要主线线程频繁的fork出bgsvae线程，这样对Redis的性能是会产生影响的，并且也需要持续的对磁盘进行写操作。

我们可以采用另外一种方式：增量快照，所谓增量快照就是指做了一次全量快照后，后续的快照只对修改的数据进行快照记录，这样可以避免每次全量快照的开销，但是，这么做的前提是，我们需要记住哪些数据被修改了。你可不要小瞧这个“记住”功能，它需要我们使用额外的元数据信息去记录哪些数据被修改了，这会带来额外的空间开销问题。如下图所示：

![image-20221017151832138](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221017151832138.png)

 如果我们对每一个键值对的修改，都做个记录，那么，如果有 1 万个被修改的键值对，我们就需要有 1 万条额外的记录。而且，有的时候，键值对非常小，比如只有 32 字节，而记录它被修改的元数据信息，可能就需要 8 字节，这样的画，为了“记住”修改，引入的额外空间开销比较大。这对于内存资源宝贵的 Redis 来说，有些得不偿失。

所以说，全量快照和增量快照都有各自的优点和缺点，至于实际应用时，则要根据具体情况进行权衡。

**过期键如何处理：**

生成RDB文件：生成时对过期键进行检查，过期键不放入rdb文件

载入RDB文件：载入时，如果以主服务器模式运行，程序会对文件中保存的键进行检查，未过期的键会被载入到数据库中，而过期的键则会忽略，如果以从服务器运行，无论键过期与否，均会载入数据库，过期键会通过与主服务器同步而删除；



### 7.6 AOF

AOF也是redis的一种持久化方式，AOF持久化的实时性更好，目前是主流的持久化方案，默认redis是没有开启AOF方式的持久化，可以通过appendonly参数开启，开启AOF持久化后每执行一条会更改redis中的数据命令，redis就会将该命令写入硬盘中的AOF文件，AOF文件的保存位置和RDB文件的位置相同，都是通过dir参数设置的，默认文件名是appendonly.aof，AOF文件中的所有命令都以Redis通讯协议的格式保存

当AOF持久化功能处于打开状态时，服务在执行一个写命令后，会以协议格式将被执行的写命令追加到服务器状态aof_buf缓冲器的末尾，redis服务进程就是一个事件循环，这个循环中的文件事件负责接受客户端的命令请求，以及向客户端发送命令恢复，而事件事件负责执行serverCron函数这样需要定时运行的函数，在每一次结束一个事件循环之前，都会调用flushAppendOnlyFile函数，考虑是否需要将aof_buf缓冲区的内容写入和同步到AOF文件里

AOF存在三种不同的持久化方式：

- always：每次有数据修改发生时都会写入AOF文件，这样会严重降低redis的速度，因为SAVE时由Redis主进程执行的，在SAVE执行期间，主进程会被阻塞，不能接受命令请求；将 aof_buf 缓冲区中的所有内容写入并同步到 AOF 文件

- everysec：每秒钟同步一次，显示地将多个写命令同步到硬盘，将 aof_buf 缓冲区中的所有内容写入到 AOF 文件，如果上次同步 AOF 文件的时间距离现在超过一秒钟，那么再次对 AOF 文件进行同步，并且这个操作是由一个线程专门负责执行的

![image-20221007234542057](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221007234542057.png)

- no：让操作系统决定何时进行同步，只会在以下任意一种情况中才会被执行，这三种情况下的save操作都会引起redis主进程阻塞； 将 aof_buf 缓冲区中的所有内容写入到 AOF 文件，但并不对 AOF 文件进行同步，何时同步由操作系统来决定

  - Redis被关闭

  - AOF功能被关闭

  - 系统的写缓存被刷新（可能是缓存已经被写满，或者定期保存操作被执行）

![image-20221007234818242](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221007234818242.png)

​    **AOF重写**

AOF 文件通过同步 Redis 服务器所执行的命令，从而实现了数据库状态的记录，但是，这种同步方式会造成一个问题：随着运行时间的流逝，AOF 文件会变得越来越大。为了解决这个问题，redis需要对AOF文件进行重写，创建一个新的AOF文件来代替原有的AOF文件，新AOF文件和原有的AOF文件保存的数据库状态完全一样，但新AOF文件的体积小于等于原有AOF文件的体积；

根据键的类型，使用适当的写入命令来重现键的当前值，这就是AOF重写的实现原理，

AOF后台重写：

redis决定将AOF重写程序放到子进程里执行，这样的最大好处是：

1. 子进程进行AOF重写期间，主进程可以继续处理命令请求

2. 子进程带有主进程的数据副本，使用子进程而不是线程，可以在避免锁的情况下，保证数据安全性；

使用子进程也有一个问题需要解决：因为子进程在进行 AOF 重写期间，主进程还需要继续处理命令，而新的命令可能对现有的数据进行修改，这会让当前数据库的数据和重写后的AOF 文件中的数据不一致。

为了解决这个问题，Redis 增加了一个 AOF 重写缓存，这个缓存在 fork 出子进程之后开始启用，Redis 主进程在接到新的写命令之后，除了会将这个写命令的协议内容追加到现有的 AOF文件之外，还会追加到这个缓存中：

![image-20221007235932434](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221007235932434.png)

当子进程完成 AOF 重写之后，它会向父进程发送一个完成信号，父进程在接到完成信号之后，

会调用一个信号处理函数，并完成以下工作：

1. 将 AOF 重写缓存中的内容全部写入到新 AOF 文件中。

2. 对新的 AOF 文件进行改名，覆盖原有的 AOF 文件

这个信号处理函数执行完毕之后，主进程就可以继续像往常一样接受命令请求了。在整个 AOF

后台重写过程中，只有最后的写入缓存和改名操作会造成主进程阻塞，在其他时候，AOF 后台

重写都不会对主进程造成阻塞，这将 AOF 重写对性能造成的影响降到了最低。

以上就是 AOF 后台重写，也即是 *BGREWRITEAOF* 命令的工作原理。

AOF重写可以由用户通过调用 BGREWRITEAOF 手动触发。

AOF重写服务器触发条件：

1. 没有 *BGSAVE* 命令在进行。

2. 没有 *BGREWRITEAOF* 在进行。

3. 当前 AOF 文件大小大于 server.aof_rewrite_min_size （默认值为 1 MB）。

4. 当前 AOF 文件大小和最后一次 AOF 重写后的大小之间的比率大于等于指定的增长百分

比。默认情况下，增长百分比为 100% 

**aof过期键处理：**

- 当服务器以aof持久化模式运行时，如果数据库中的某个键已经过期，但它还没有被删除，那么aof文件不会因为这个过期键而产生任何影响；当过期键被删除后，程序会向aof文件追加一条del命令来显式记录该键已被删除。
- aof重写过程中，程序会对数据库中的键进行检查，已过期的键不会被保存到重写后的aof





### 7.7 双端链表

redis自己实现了一个双端链表结构，双端链表作为一种常见的数据结构，他是redis列表结构的底层实现之一，还被大量redis模块使用，用于构建redis的其他功能；

双端链表被很多Redis内部模块所应用：

- 事务模块使用双端链表来按顺序保存输入的命令
- 服务器模块使用双端链表来保存多个客户端
- 订阅/发送模块使用双端链表来保存订阅模式的多个客户端
- 事件模块使用双端链表来保存时间事件；

![image-20221008084108549](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008084108549.png)

其中，listNode 是双端链表的节点：

![image-20221008084225920](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008084225920.png)

而 list 则是双端链表本身：

![image-20221008084236907](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008084236907.png)

listNode的Value属性的类型是void *，说明这个双端链表对节点所保存的值的类型不做限制；

list类型保留了三个函数指针，dup、free、match，分别用于处理值的复制、释放和对比匹配，在对节点的值进行处理时，如果有给定这些函数，那么他们就会被调用；

双端链表及其节点的性能特性如下：

- 节点带有前驱和后继指针，访问前驱节点和后继节点的复杂度为O(1)，并且对链表的迭代可以在从表头到表尾和从表尾到表头两个方向进行；
- 链表带有指向表头和表尾的指针，因此对表头和表尾进行处理的时间复杂度为O(1)
- 链表带有记录节点数量的属性，所以可以在O(1)时间复杂度内返回链表的节点数量（长度）



### 7.8 字典结构

字典有名为映射（map）或关联数组，他是一种抽象数据结构，由一集键值对组成，各个键值对的键各不同相同，程序可以将新的键值对添加到字典中，或者基于键进行查找、更新或删除操作；

字典在redis中主要用途如下：

1. 实现数据库键空间，因为redis本身就是key/value类型的数据库
2. 用作Hash类型键的其中一种底层实现

实现字典的方法有很多种：

1. 最简单的就是使用链表或数组，但是这种方式只适用于元素个数不多的情况下
2. 要兼顾高效和简单性，可以使用哈希表
3. 如果追求更为稳定得性能特征，并且希望高效地实现排序操作得话，则可以使用更为复杂得平衡树；

redis中字典使用简单得哈希表来实现；

![image-20221008091050652](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008091050652.png)

**哈希表为什么有两个？**

0号哈希表是字典得主要哈希表，而1号哈希表只有在程序对0号哈希表进行rehash时才使用；

**如何解决hash碰撞**

使用链地址法来处理键碰撞：当多个不同得键拥有相同得哈希值时，哈希表用一个链表将这些键连接起来；

![image-20221008091700581](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008091700581.png)

**哈希算法**

1. MurmurHash2 32 bit 算法：这种算法的分布率和速度都非常好，具体信息请参考 Mur

   murHash 的主页：http://code.google.com/p/smhasher/ 。

2. 基 于 djb 算 法 实 现 的 一 个 大 小 写 无 关 散 列 算 法： 具 体 信 息 请 参 考

   http://www.cse.yorku.ca/~oz/hash.html 。

命令表以及Lua脚本缓存都用到了算法2

算法1得应用更加广泛：数据库、集群、哈希键、阻塞操作等功能都用到了这个算法；



**rehash**

因为使用链地址法来解决哈希碰撞，当哈希碰撞较高时，就会导致查找效率较低，退化为遍历链表多次，所以为了保持良好得性能，需要进行reHash；

reHash被触发时机：

1. 自然rehash： ratio >= 1 并且变量dict_can_resize为真
2. 强制rehash：ratio大于变量dict_force_resize_ratio，目前dict_force_resize_ratio为5；

ratio = userd/ size = 当前Hash节点数量 / 数组大小

**dict_can_resize什么时候为假？**

一个数据库就是一个字典，数据库里的哈希类型键也是一个字典，当Redis使用子进程执行后台持久化任务时，为了最大化地利用系统得copy on write机制，程序会暂时将dict_can_resize设置为假，避免自然执行rehash，从而减少程序对内存得触碰；

**rehash过程**

1. 创建ht[1] - table ，要比ht[0] - table更大
2. 将ht[0] -table 上的键值对都迁移到ht[1] - table上，rehashidx表示ht[0] rehash的哪个索引，采用渐进式rehash，两部分组成：
   1. _dictRehashStep 用于对数据库字典、以及哈希键的字典进行被动 rehash，每执行一次添加、查找、删除操作都会执行该函数，将ht[0]第一个不为空索引上的所有节点迁移至ht[1]
   2. dictRehashMilliseconds 可以在指定的毫秒数内，对字典进行 rehash ，在规定时间内尽可能地对数据库字典中那些需要rehash的字典进行rehash；
3. 将ht[0] -table的数据清空，并将ht[1]替换为新的ht[0]，并初始化一个空的哈希表，将他设置为ht[1]，并且将rehashidx属性设置为-1，标识rehash已经停止；

**字典压缩**

redis中字典不仅可以扩容，还是压缩，收缩规则：检查字典的使用率是否低于系统允许的最小比率，默认是10，通过哈希表已用节点数量、哈希表大小进行计算，当字典填充率低于10%，程序就可以对这个字典进行收缩操作了；

字典的扩展操作时自动触发的，而字典的收缩操作则是由程序手动执行的，当字典用于实现哈希键的时候，每次从字典中删除一个键值对，程序就会执行一次htNeedsResize 函数，如果字典达到了收缩的标准，程序将立即对字典进行收缩；



### 7.9 跳跃表

https://www.jianshu.com/p/9d8296562806

跳跃表是一种随机化的数据，这种结构以有序的方式在层次化的链表中保存元素，他的效率可以和平衡树匹配 -- 查找、删除、添加等操作可以在对数期望的时间下完成，并且比起平衡树来说，跳跃表的实现要简单直观得多；

跳跃表是一个可以实现二分查找的有序链表，跳跃列表的平均查找和插入时间复杂度都是O(logn)。

跳跃表是一种高效实现插入、删除、查找的内存数据结构，这些操作的时间复杂度是O(logn)，与红黑树以及其他的二分查找树相比，跳跃表的优势在于实现简单，而且在并发场景下加锁粒度更小，从而可以实现更高的并发性；

![image-20221008101653302](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008101653302.png)

表头：负责维护跳跃表的节点指针

跳跃表节点：保存着元素值，以及多个层

层：保存着指向其他元素的指针，高层的指针越过的元素数量大于等于低层的指针，为了提高查找的效率，程序总是从高层先开始访问，然后随着元素值范围的缩小，慢慢降低层次；

表尾：全部由NULL组成，表示跳跃表的末尾；

跳跃表在Redis的唯一作用，就是实现有序集合数据类型，跳跃表将指向有序集的score值和member域的指针作为元素，并以score值为索引，对有序集元素进行排序；

• 为了适应自身的需求，Redis 基于 William Pugh 论文中描述的跳跃表进行了修改，包括：

1. score 值可重复。

2. 对比一个元素需要同时检查它的 score 和 memeber 。

3. 每个节点带有高度为 1 层的后退指针，用于从表尾方向向表头方向迭代

![image-20221008102610808](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008102610808.png)

如何维护索引：

当每次有数据要插入时，先通过概率算法告诉我们这个元素需要插入到几级索引中，通过randomLevel方法，该方法会随机生成1~Max_LEVEL之间的数，且该方法由1/2概率返回1，1/4的概率返回2，1/8的概率返回3，以此类推，返回1表示不需要建索引，返回2表示当前插入的元素需要建1及索引，以此类推；



### 7.10 为什么Redis选择单线程模型

[为什么 Redis 选择单线程模型 - 面向信仰编程 (draveness.me)](https://draveness.me/whys-the-design-redis-single-thread/)

redis一开始就选择使用单线程模型处理来自客户端的绝大数网络请求，重要原因如下：

1. 使用单线程模型能带来更好的可维护性，方便开发和调试
2. 使用单线程模型也能并发的处理客户端的请求
3. redis服务中运行的绝大多数操作的性能瓶颈都不是CPU，而是内存和网络

**可维护性：**

可维护性对一个项目非常重要，多线程模型虽然在某些方面表现优异，但是引入多线程后程序执行的顺序具有不确定性，代码的执行过程不再是串行的，多线程的上下文切换开销增高，锁竞争加剧，引入并发控制就会导致复杂性变高；

**并发处理：**

使用单线程模型也并不意味着程序不能并发的处理任务，redis使用I/O多路复用机制并发处理来自客户端的多个连接，同时等待多个连接发送的请求，在I/O多路复用模型中，最重要的函数调用就是select以及类似函数，该方法能够同时监控多个文件描述符（也就是客户端的连接）的可读可写情况，当其中的某些文件描述符可读或者可写时，select方法就会返回可读以及可写的文件描述符个数，使用 I/O 多路复用技术能够极大地减少系统的开销，系统不再需要额外创建和维护进程和线程来监听来自客户端的大量连接，减少了服务器的开发成本和维护成本。

**性能瓶颈：**

多线程技术能够帮助我们充分利用 CPU 的计算资源来并发的执行不同的任务，但是 CPU 资源往往都不是 Redis 服务器的性能瓶颈。哪怕我们在一个普通的 Linux 服务器上启动 Redis 服务，它也能在 1s 的时间内处理 1,000,000 个用户请求。

Redis 并不是 CPU 密集型的服务，如果不开启 AOF 备份，所有 Redis 的操作都会在内存中完成不会涉及任何的 I/O 操作，这些数据的读写由于只发生在内存中，所以处理速度是非常快的；整个服务的瓶颈在于网络传输带来的延迟和等待客户端的数据传输，也就是网络 I/O，所以使用多线程模型处理全部的外部请求可能不是一个好的方案。

#### 7.10.1 IO多路复用如何理解？

这是IO模型的一种，即经典的Reactor设计模式，有时也称为异步阻塞IO。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/DkA0VYrOHvtWkvO02tIickx6ibOgCywvlibW8ELDCn2rxX7pJPbhkhJt1U01WTpI6uw5NWyA4iaB1iaXVxL8VunHZMQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



多路指的是多个socket连接，复用指的是复用一个线程。多路复用主要有三种技术：select，poll，epoll。epoll是最新的也是目前最好的多路复用技术。采用多路 I/O 复用技术可以让单个线程高效的处理多个连接请求（尽量减少网络IO的时间消耗），且Redis在内存中操作数据的速度非常快（内存内的操作不会成为这里的性能瓶颈），主要以上两点造就了Redis具有很高的吞吐量。



### 7.11 为什么redis也引用了多线程

redis在4.0版本以后也引入了多线程，在最新的几个版本中加入了一些可以被其他线程异步处理的删除操作，也就是我们上面提到的`UNLINK`、`FLUSHALL ASYNC` 和 `FLUSHDB ASYNC`，我们为什么会需要这些删除操作，而它们为什么需要通过多线程的方式异步处理？

我们可以在 Redis 在中使用 `DEL` 命令来删除一个键对应的值，如果待删除的键值对占用了较小的内存空间，那么哪怕是**同步地**删除这些键值对也不会消耗太多的时间。

但是对于 Redis 中的一些超大键值对，几十 MB 或者几百 MB 的数据并不能在几毫秒的时间内处理完，Redis 可能会需要在释放内存空间上消耗较多的时间，这些操作就会阻塞待处理的任务，影响 Redis 服务处理请求的 PCT99 和可用性。

然而释放内存空间的工作其实可以由后台线程异步进行处理，这也就是 `UNLINK` 命令的实现原理，它只会将键从元数据中删除，真正的删除操作会在后台异步执行。

### 7.12 Redis6.0之后为何还引入了多线程

[Redis 6.0 新特性-多线程连环13问！ (qq.com)](https://mp.weixin.qq.com/s/FZu3acwK6zrCBZQ_3HoUgw)

**Redis6.0** **引⼊多线程主要是为了提⾼⽹络** **IO** **读写性能**，因为这个算是 Redis 中的⼀个性能瓶颈

（Redis 的瓶颈主要受限于内存和⽹络）。

⽹络数据的读写这类耗时操作上使⽤了多线程， 执⾏命令仍然是单线程顺序执⾏。因此，你也不需要担⼼线程安全问题。

Redis6.0 的多线程默认是禁⽤的，只使⽤主线程。如需开启需要修改 redis 配置⽂件 redis.conf；

开启多线程后，还需要设置线程数，否则是不生效的。同样修改redis.conf配置文件，官方有一个建议：4核的机器建议设置为2或3个线程，8核的建议设置为6个线程，线程数一定要小于机器核数。还需要注意的是，线程数并不是越大越好，官方认为超过了8个基本就没什么意义了。

redis将所有数据放在内存中，内存的响应时长大约为100纳秒，对于小数据包，redis可以处理80，000 到 100，000 QPS，这也是redis处理的极限了，对于80%的公司来说，单线程redis已经够用了，但是随着越来越复杂的业务场景，有些公司动不动就上亿的交易量，因此需要更大的QPS。常见的解决方案是在分布式架构中对数据进行分区并采用多个服务器，但该方案有非常大的缺点，例如要管理的Redis服务器太多，维护代价大；某些适用于单个Redis服务器的命令不适用于数据分区；数据分区无法解决热点读/写问题；数据偏斜，重新分配和放大/缩小变得更加复杂等等；

 从Redis自身角度来说，因为读写网络的read/write系统调用占用了Redis执行期间大部分CPU时间，瓶颈主要在于网络的 IO 消耗, 优化主要有两个方向:

  • 提高网络 IO 性能，典型的实现比如使用 DPDK 来替代内核网络栈的方式

  • 使用多线程充分利用多核，典型的实现比如 Memcached。

 协议栈优化的这种方式跟 Redis 关系不大，支持多线程是一种最有效最便捷的操作方式。所以总结起来，redis支持多线程主要就是两个原因：

  • 可以充分利用服务器 CPU 资源，目前主线程只能利用一个核

  • 多线程任务可以分摊 Redis 同步 IO 读写负荷

#### 7.12.1 多线程实现机制

![image-20221008135450416](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008135450416.png)

**流程简述如下：**

1、主线程负责接收建立连接请求，获取 socket 放入全局等待读处理队列

2、主线程处理完读事件之后，通过 RR(Round Robin) 将这些连接分配给这些 IO 线程

3、主线程阻塞等待 IO 线程读取 socket 完毕

4、主线程通过单线程的方式执行请求命令，请求数据读取并解析完成，但并不执行

5、主线程阻塞等待 IO 线程将数据回写 socket 完毕

6、解除绑定，清空等待队列

![图片](https://mmbiz.qpic.cn/mmbiz_png/DkA0VYrOHvtWkvO02tIickx6ibOgCywvlibpbYER5698cvSsm69yWHQ7iaUwQ140o9PbN5R9kHtlIzGiaoBN2LbwmyA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

（图片来源：https://ruby-china.org/topics/38957）



**该设计有如下特点：**

1、IO 线程要么同时在读 socket，要么同时在写，不会同时读或写

2、IO 线程只负责读写 socket 解析命令，不负责命令处理



### 7.13 整数集合

整数集合（intset）用于有序、无重复地保存多个整数值，他会根据元素的值，自动选择该用什么长度的整数类型来保存元素；

Intset是集合键的底层实现之一，如果一个集合：

1. 只保存着整数元素
2. 元素的数量不多

那么redis就会使用intset来保存集合元素；

![image-20221008143657187](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008143657187.png)

contents 数组是实际保存元素的地方，数组中的元素有以下两个特性：

• 没有重复元素；

• 元素在数组中从小到大排列；

contents 数组的 int8_t 类型声明比较容易让人误解，实际上，intset 并不使用 int8_t 类型

来保存任何元素，结构中的这个类型声明只是作为一个占位符使用：在对 contents 中的元素

进行读取或者写入时，程序并不是直接使用 contents 来对元素进行索引，而是根据 encoding

的值，对 contents 进行类型转换和指针运算，计算出元素在内存中的正确位置。在添加新元

素，进行内存分配时，分配的容量也是由 encoding 的值决定。

小结：

- Intset用于有序、无重复地保存多个整数值，他会根据元素的值，自动选择该用什么长度的整数类型来保存元素；
- 当一个位长度更长的整数值添加到intset时，需要对intset进行升级，新的intset中每个元素的位长度都等于新添加值得位长度，但原有元素的值不变
-  升级会引起整个 intset 进行内存重分配，并移动集合中的所有元素，这个操作的复杂度为 *O*(*N*) 。
-  Intset 只支持升级，不支持降级。
- Intset 是有序的，程序使用二分查找算法来实现查找操作，复杂度为 *O*(lg *N*) 。

### 7.14 压缩列表

压缩列表是由一系列特殊编码得内存块构成得列表，一个压缩列表可以包含多个节点entry，每个节点可以保存一个长度受限得字符数组或者整数，包括：

- 字符数组：长度小于等于63、16383、4294967295字节得字符数组
- 整数：4位长，介于0至12之间得无符号整数，1字长，有符号整数，3字长，有符号整数，int16_t类型整数、int32_t类型整数、int64_t类型整数；

因为ziplist节约内存得性质，他被哈希键、列表键和有序集合键作为初始化得底层实现来使用；

压缩列表得分布结构如下：

![image-20221008202856766](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008202856766.png)

![image-20221008202929047](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008202929047.png)

![image-20221008203904947](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008203904947.png)

添加和删除 ziplist 节点有可能会引起连锁更新，因此，添加和删除操作的最坏复杂度为

*O*(*N*2) ，不过，因为连锁更新的出现概率并不高，所以一般可以将添加和删除操作的复

杂度视为 *O*(*N*) 

### 7.15 redis各种数据类型以及编码方式

![image-20221008205547358](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008205547358.png)

#### 7.15.1 字符串数据类型

字符串是Redis使用得最广泛得数据类型，他除了set、get等命令得操作对象之外，数据库中得所有键，以及执行命令时提供给Redis得参数，都是用这种类型保存的；

字符串类型分别使用REDIS_ENCODING_INT和REDIS_ENCODING_RAW两种编码：

- REDIS_ENCODING_INT使用long类型来保存long类型值
- REDIS_ENCODING_RAW则使用sdshdr结构来保存sds(char * 、long long、double 和long double类型值；

redis中，只有能表示long类型得值，才会以整数得形式保存，其他类型得整数、小数和字符串，都是用sds结构来保存，新创建字符串默认使用redis_encoding_raw编码，再将字符串作为键或者值保存进数据库时，程序会尝试将字符串转为REDIS_ENCODING_INT编码；



#### 7.15.2 哈希表

哈希表使用REDIS_ENCODING_ZIPLIST和REDIS_ENCODING_HT两种编码方式：

![image-20221008214935916](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008214935916.png)

创建空白哈希表时，程序默认使用REDIS_ENCODING_ZIPLIST编码，当以下任何一个条件被满足时，程序将编码切换为REDIS_ENCODING_HT：

- 哈希表中某个键或者某个值得长度大于server.hash_max_ziplist_value（默认值为64）
- 压缩列表中得节点数量大于server.hash_max_ziplist_entries（默认值为512）

#### 7.15.3 列表

 它 使 用REDIS_ENCODING_ZIPLIST 和 REDIS_ENCODING_LINKEDLIST 这两种方式编码：

![image-20221008215716515](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008215716515.png)

创建新列表时 Redis 默认使用 REDIS_ENCODING_ZIPLIST 编码，当以下任意一个条件被满足

时，列表会被转换成 REDIS_ENCODING_LINKEDLIST 编码：

• 试 图 往 列 表 新 添 加 一 个 字 符 串 值， 且 这 个 字 符 串 的 长 度 超 过

server.list_max_ziplist_value （默认值为 64 ）。

• ziplist 包含的节点超过 server.list_max_ziplist_entries （默认值为 512 ）。



#### 7.15.4 集合

它 使 用REDIS_ENCODING_INTSET 和 REDIS_ENCODING_HT 两种方式编码：

![image-20221008215909300](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008215909300.png)

第一个添加到集合的元素，决定了创建集合时所使用的编码：

• 如果第一个元素可以表示为 long long 类型值（也即是，它是一个整数），那么集合的初

始编码为 REDIS_ENCODING_INTSET 。 

• 否则，集合的初始编码为 REDIS_ENCODING_HT 。



#### 7.15.5 有序集合

 它 使 用REDIS_ENCODING_ZIPLIST 和 REDIS_ENCODING_SKIPLIST 两种方式编码：

![image-20221008220123815](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008220123815.png)

**编码得选择**

在通过 *ZADD* 命令添加第一个元素到空 key 时，程序通过检查输入的第一个元素来决定该创

建什么编码的有序集。

如果第一个元素符合以下条件的话，就创建一个 REDIS_ENCODING_ZIPLIST 编码的有序集：

• 服务器属性 server.zset_max_ziplist_entries 的值大于 0 （默认为 128 ）。

• 元素的 member 长度小于服务器属性 server.zset_max_ziplist_value 的值（默认为 64

）。

**编码转换**

否则，程序就创建一个 REDIS_ENCODING_SKIPLIST 编码的有序集。

对于一个 REDIS_ENCODING_ZIPLIST 编码的有序集，只要满足以下任一条件，就将它转换为

REDIS_ENCODING_SKIPLIST 编码：

• ziplist 所保存的元素数量超过服务器属性 server.zset_max_ziplist_entries 的值

（默认值为 128 ） 

• 新添加元素的 member 的长度大于服务器属性 server.zset_max_ziplist_value 的值

（默认值为 64 ）

**压缩链表的有序集**

![image-20221008221527744](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008221527744.png)

多个元素之间按score值从小到大排序，如果两个元素的score相同，那么按字典序对member进行对比，决定哪个元素排在前面，哪个元素排在后面；

**跳跃表编码的有序集合**

使用字典+跳跃表这两个数据结构来保存有序集合元素；

元素score是一个double类型的浮点数：

![image-20221008221823109](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008221823109.png)

通过使用字典结构，并将member作为键，score作为值，有序集可以在O(1)复杂度内：

- 检查给定member是否存在于有序集（很多底层函数使用）
- 取出member对应的score值（实现ZSCORE命令）

通过使用跳跃表可以让有序集合支持以下两种操作：

在O(logN)期望时间、O（N）最坏时间内根据score对member进行定位（被很多底层函数使用）

范围性查找和处理操作，这是高效地实现ZRANGE、ZRANK和ZINTERSTORE等命令的关键；



### 7.16 redis的事务是如何实现的

Redis通过MULTI、DISCARD、EXEC和WATCH四个命令来实现事务功能，事务提供了一种"将多个命令打包，然后一次性、按顺序地执行"的机制，并且事务在执行期间不被主动中断，一个事务从开始到执行会经历以下三个阶段：

1. 开始事务：

MULTI命令标记着事务的开始，这个命令唯一做的就是，将客户端的redis_multi选项打开，让客户端从非事务状态切换到事务状态：

![image-20221008223754518](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008223754518.png)

1. 命令入队

当客户端进入事务状态后，服务器在收到客户端的命令时，不会立即执行命令，而是将这些命令全部放进一个事务队列里，然后返回QUEUED，表示命令已入队；

事务队列是一个数组，每个数组项都包含三个属性：

1. 要执行的命令（cmd）
2. 命令的参数（argv）
3. 参数的个数（argc）

![image-20221008224516141](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008224516141.png)

1. 执行事务

如果客户端处于事务状态，那么当EXEC命令执行时，服务器根据客户端所保存的事务队列，以先进先出的方式执行事务队列中的命令，最先入队的命令最先执行，而最后入队的命令最后执行；执行事务中的命令所得的结果会以FIFO的顺序保存到一个回复队列中，当事务队列里的所有命令被执行完之后，*EXEC* 命令会将回复队列作为自己的执行结果返回给客户端，客户端从事务状态返回到非事务状态，至此，事务执行完毕。

DISCARD命令用于取消一个事务，他清空客户端的整个事务队列，然后将客户端从事务状态调整回非事务状态，最后返回字符串OK给客户端，说明事务已经被取消了；

WATCH只能在客户端进入事务状态之前执行，用于在事务开始之前监视任意数量的键，调用EXEC执行事务时，如果任意一个被监视的键已经被其他客户端修改了，那么整个事务不再执行，直接返回失败；

**watch命令的实现**

在每个代表数据库的redis.h/redisDb结构类型中，都保存了一个watched_keys字典，字典的键是这个数据库被监视的键，而字典的值则是一个链表，链表保存了所有监视这个键的客户端：

![image-20221008230026754](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008230026754.png)

在任何对数据库键空间（key space）进行修改的命令成功执行之后（比如 *FLUSHDB* 、*SET*

、*DEL* 、*LPUSH* 、*SADD* 、*ZREM* ，诸如此类），multi.c/touchWatchKey 函数都会被调用

——它检查数据库的 watched_keys 字典，看是否有客户端在监视已经被命令修改的键，如果

有的话，





![image-20221008230236115](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221008230236115.png)

当客户端发送 *EXEC* 命令、触发事务执行时，服务器会对客户端的状态进行检查：

• 如果客户端的 REDIS_DIRTY_CAS 选项已经被打开，那么说明被客户端监视的键至少有一

个已经被修改了，事务的安全性已经被破坏。服务器会放弃执行这个事务，直接向客户端

返回空回复，表示事务执行失败。

• 如果 REDIS_DIRTY_CAS 选项没有被打开，那么说明所有监视键都安全，服务器正式执行

事务。

Redis的事务保证了一致性和隔离性，但并不保证原子性和持久性；

隔离性：

在事务执行时，不会对事务进行中断，事务可以运行直到执行完所有事务队列中命令为止，因此，redis的事务总是带有隔离性的；

一致性：

入队错误，事务不会被执行，不影响数据库的一致性；

执行错误，只会将错误包含在事务的结果中，不会引起事务中断或整个失败

redis进程被终结，会使用RDB或AOF进行数据恢复，数据也是一致的；



### 7.17 内存淘汰机制

redis提供的过期策略采用的是定期删除和惰性删除，定期删除策略是对过期键的定期删除由 redis.c/activeExpireCycle 函执行：每当 Redis 的例行处理程序

serverCron 执行时，activeExpireCycle 都会被调用——这个函数在规定的时间限制内，尽

可能地遍历各个数据库的 expires 字典，随机地检查一部分键的过期时间，并删除其中的过期

键。

因为这两种删除过期数据的策略都是不能精准删除，所以就需要内存淘汰策略；

1. no eviction：当内存使用超过配置时就会返回错误，不会驱逐任何键
2. allkeys-lru：加入键的时候，如果过限，首先通过LRU算法驱逐最久没有使用的键
3. voliatile-lru：加入键的时候如果过限，首先从设置了过期时间的键集合中驱逐最久没有使用的键
4. allkey-random：加入键的时候如果过限，从所有key中随机删除
5. voliatile-random：加入键的时候如果过限，从设置了过期时间的键集合中随机删除
6. voliatile-ttl：从配置了过期时间的键中驱逐马上就要过期的键
7. volatile-lfu：从所有配置了过期时间的键中驱逐使用频率最少的键
8. allkeys-lfu：从所有键中驱逐使用频率最少的键



### 7.18 什么是缓存雪崩

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247484763&idx=1&sn=fe85cd9202da355b5df6e2fae6655ee5&chksm=c22b8307f55c0a11ea2fa8c62cf911ccf0e2e7f1fbf11b6bc6ad1dd5da7ee55ada17664c2917&token=1817605393&lang=zh_CN#rd

缓存在同一时间大面积的失效，后面的请求都直接落到了数据库上，造成数据库短时间内承受大量的请求；

解决办法：

事前：如果缓存雪崩造成的原因是因为缓存服务宕机造成的，可以将`redis`采用集群部署，可以使用 主从+哨兵 ，Redis Cluster 来避免 Redis 全盘崩溃的情况。若缓存雪崩是因为大量缓存因为失效时间而造成的，我们在批量往`redis`存数据的时候，把每个Key的失效时间都加个随机值就好了，这样可以保证数据不会在同一时间大面积失效，或者设置热点数据永远不过期，有更新操作就更新缓存就可以了。

事中：我们可以使用ehcache 本地缓存 + Hystrix 限流&降级 ,避免 MySQL 被打死的情况发生。

这里使用`echache`本地缓存的目的就是考虑在 Redis Cluster 完全不可用的时候，ehcache 本地缓存还能够支撑一阵。

使用 Hystrix 进行 限流 & 降级 ，比如一秒来了5000个请求，我们可以设置假设只能有一秒 2000 个请求能通过这个组件，那么其他剩余的 3000 请求就会走限流逻辑，然后去调用我们自己开发的降级组件（降级）。比如设置的一些默认值呀之类的。以此来保护最后的 MySQL 不会被大量的请求给打死。

事后：如果缓存服务宕机了，这里我们可以开启**「Redis」** 持久化 **「RDB」**+**「AOF」**，一旦重启，自动从磁盘上加载数据，快速恢复缓存数据。

综上所述，可画出如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/k5430ljpYPNic3ic6iaK8z8O1MpctWOYFLEm3nLmcLGxT5M3SVOoKn842IJWazutbibkpaxr7ia8hv1ic4PmqypiaI7xQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### 7.19 什么是缓存穿透

在正常的情况下，用户查询数据都是存在的，但是在异常情况下，缓存与数据都没有数据，但是用户不断发起请求，这样每次请求都会打到数据库上面去，这时的用户很可能是攻击者，攻击会导致数据库压力过大，严重会击垮数据库。

**解决办法：**

1. 添加参数校验，不合法的东西直接返回
2. 缓存空值，之所以会发生穿透，就是因为缓存中没有存储这些空数据的key。从而导致每次查询都到数据库去了。那么我们就可以为这些key 设置的值设置为null 丢到缓存里面去。后面再出现查询这个key 的请求的时候，直接返回null ,就不用在到 数据库中去走一圈了。但是别忘了设置过期时间。
3. 布隆过滤器：`redis`的一个高级用法就是使用布隆过滤器（Bloom Filter），BloomFilter 类似于一个hase set 用来判断某个元素（key）是否存在于某个集合中。这个也能很好的防止缓存穿透的发生，他的原理也很简单就是利用高效的数据结构和算法快速判断出你这个Key是否在数据库中存在，不存在你return就好了，存在你就去查了DB刷新KV再return。



### 7.20 什么是缓存击穿

我们在平常高并发的系统中，大量的请求同时查询一个key时，假设此时，这个key正好失效了，就会导致大量的请求都打到数据库上面去，这种现象我们称为击穿。

这么看缓存击穿和缓存雪崩有点像，但是又有一点不一样，缓存雪崩是因为大面积的缓存失效，打崩了DB，而缓存击穿不同的是**「缓存击穿」**是指一个Key非常热点，在不停的扛着大并发，大并发集中对这一个点进行访问，当这个Key在失效的瞬间，持续的大并发就穿破缓存，直接请求数据库，就像在一个完好无损的桶上凿开了一个洞。

缓存击穿带来的问题就是会造成某一时刻数据库请求量过大，压力剧增。

**解决办法：**

1. 不过期，直接让热点数据永远不过期，定时任务定期去刷新数据就可以了。不过这样设置需要区分场景，比如某宝首页可以这么做。
2. 互斥锁：为了避免出现缓存击穿的情况，我们可以在第一个请求去查询数据库的时候对他加一个互斥锁，其余的查询请求都会被阻塞住，直到锁被释放，后面的线程进来发现已经有缓存了，就直接走缓存，从而保护数据库。但是也是由于它会阻塞其他的线程，此时系统吞吐量会下降。需要结合实际的业务去考虑是否要这么做。Go语言中singleFlight库就可以解决缓存击穿；

https://mp.weixin.qq.com/s?__biz=MzkyNzI1NzM5NQ==&mid=2247484988&idx=1&sn=dff9fc3c072545354365b85765cc126f&chksm=c22b8060f55c0976a1c05a578270414bf144190453739477d9789c7faeeb451024c5ca7d7fd3&token=1817605393&lang=zh_CN#rd



### 7.21 什么是布隆过滤器

https://mp.weixin.qq.com/s/5zHQbDs978OoA3g83NaVmw

布隆过滤器是1970年由布隆提出，它实际上是一个很长的二进制向量和一系列随机映射函数，布隆过滤器可以用于检索出一个元素是否在一个集合中，他的优点是空间效率和查询时间远远超过一般的算法；

原理：

布隆过滤器的原理是，当一个元素被加入集合时，通过 K 个散列函数将这个元素映射成一个位数组中的 K 个点（offset），把它们置为 1。检索时，我们只要看看这些点是不是都是 1 就（大约）知道集合中有没有它了：如果这些点有任何一个 0，则被检元素一定不在；如果都是 1，则被检元素很可能在。这就是布隆过滤器的基本思想。

简单来说就是准备一个长度为 m 的位数组并初始化所有元素为 0，用 k 个散列函数对元素进行 k 次散列运算跟 len(m)取余得到 k 个位置并将 m 中对应位置设置为 1。

![image-20221009092952489](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221009092952489.png)

优点：

1. 空间占用极小，因为本身不存储数据而是用比特位表示数据是否存在，某种程度具有保密的效果
2. 插入于查询时间复杂度均为O(k)，常数级别，k表示散列函数执行次数
3. 散列函数之间可以相互独立，可以在硬件指令层加速计算

缺点：

1. 误差（假阳性率）
2. 无法删除

可以使用redis的bitmap来实现，redis提供了SETBIT、GETBIT、BITCOUNT、BITOP、





### 7.22 为什么redis不能存大key

大key通常指的是Redis存储的value过大，包括：

- 单个value过大，比如200M大小的string
- 集合元素过多，如List、Hash、Set、Zset有几百、上千万的数据

redis输出buf为16k，这意味着redis无法响应数据，需要注册【可写事件】，从而触发多次write系统调用，

这里有两个耗时点：

- 分配大内存（也可能释放内存，如DEL命令）
- 触发多次可写事件（频繁执行系统调用，如write、epoll_wait）

设一个 bigkey 为 1MB,每秒访问量为 1000，那么每秒产生 1000MB 的流量，对于普通千兆网卡，按照字节算 128M/S 的服务器来说可能扛不住。而且一般服务器采用单机多实例方式来部署，所以还可能对其他实例造成影响。



#### 7.23 redis热点key及其解决方案

热点key就是被经常访问的缓存，并且在一瞬间可能会流量激增，导致访问量增加；

可以考虑增加本地缓存，本地缓存可以减少分布式缓存的访问；



## 8 容器化技术

### 8.1 什么是docker

Docker是一个容器化平台，它包装你所有开发环境依赖成一个整体，像一个容器；

Docker容器：将一个软件包装在一个完整的文件系统中，该文件系统包含运行所需的一切：代码，运行时，系统工具，系统库等，可以安装在服务器上的任何东西，这保证软件总是运行在相同的运行环境，无需考虑基础环境配置的改变。

Docker如何解决不同系统环境的问题？

- Docker将用户程序与所需要调用的系统函数库一起打包
- Docker运行到不同操作系统时，直接基于打包的函数库，借助于操作系统的Linux内核来运行



### 8.2 docker和虚拟机的区别

虚拟机是在操作系统中模拟硬件设备，然后运行另外一个操作系统，比如在window里运行Unbuntu系统，这样就可以运行任意的Unbuntu应用了；

Docker仅仅是封装函数库，并没有模拟完整的操作系统：

![image-20221016214554894](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221016214554894.png)

Docker是一个系统进程，虚拟机是在操作系统中的操作系统

docker体积更小、启动速度快、性能好；虚拟机体积大、启动速度慢、性能一般；



### 8.3 docker的镜像和容器

**镜像**：Docker将应用程序以及所需的依赖、函数库、环境、配置等打包在一起，称为镜像；镜像就是把一个应用在硬盘上的文件及其运行环境、部分系统函数库文件一起打包形成的文件包，这个文件包是只读的。

**容器：**镜像中的应用程序运行后形成的进程就是容器，只是Docker会给容器进程做隔离，对外不可见，就是将镜像这些文件中编写的程序、函数加载到内存中形成进程；





### 8.4 Docker架构

Docker是一个CS架构的程序，由两部门组成：

- 服务端：Docker守护进程，负责处理Docker指令，管理镜像、容器等；
- 客户端：通过命令或RestAPI向Docker服务端发送指令，可以在本地或远程向服务端发送指令；

![image-20221016215816900](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221016215816900.png)



## 9 设计题

### 9.1 设计一个排行榜

https://www.dandelioncloud.cn/article/details/1497125238885421057





### 9.2 TCP长连接下，如何做负载均衡

- 轮询
- 权限（根据服务进行选择）
- 一致性hash，影响范围减小到最小，相同的请求尽可能落到同一个服务器上，稳定性更好，可以将所有的存储节点排列在首尾相接的hash环上，每个key在计算hash后会顺时针找到临近的存储节点存放，而当有节点加入或退出时，仅影响该节点在hash环上顺时针相邻的后续节点。
- 最少活跃数，将请求分发到连接数、请求数最少的候选服务器（目前处理请求最少的服务器）





## 10 微服务

#### 10.1 RPC协议

![image-20221019092458088](C:\Users\sunsong\AppData\Roaming\Typora\typora-user-images\image-20221019092458088.png)

RPC即远程过程调用，是一个用于建立适当框架的协议，从本质上讲，它使一台机器上的程序能够调用两一台机器上的子程序，而不会意识到他是远程的，RPC工作流程如下：

1. 调用方以本地调用方式调用服务
2. client stub接收到调用后负责将方法、参数等组装成能够进行网络传输的消息体，也就是进行序列化
3. client stub找到服务地址，并将消息发送到服务端；
4. server stub收到消息后进行解码，也就是反序列化
5. server stub根据解码结果调用本地服务
6. 本地服务执行并将结果返回给server stub
7. server stub 将返回结果打包成消息发送至消费方
8. client stub接收到消息，并进行解码
9. 服务消费方得到最终结果，并且返回值被设置在本地进程的堆栈中；



#### 10.2 HTTP协议和RPC协议不同？

https://www.jianshu.com/p/fe5ccfc5d7bd

HTTP是超文本传输协议，客户端和服务端约定好的一种通信格式；

而RPC则是远程过程调用，其对应的是本地调用，RPC的通信可以用HTTP协议，也可以自定义协议，是不做约束的，RPC就是通过网络进行远程调用；

RPC调用是因为服务的拆分，或者本身公司内部的多个服务之间的通信。

服务的拆分独立部署，那服务间的调用就必然需要网络通信，用WebClient调用当然可行，是比较麻烦的，我们想即使拆分了但是使用起来还是和之前本地调用一样方便，所以就出现了RPC框架，来屏蔽这些底层调用细节，使得我们编码上还是和之前本地调用相差不多，并且HTTP协议比较的冗余，RPC都是内部调用所以不需要态考虑通用性，只要公司内部保持格式统即可，所以可以做各种定制化的协议来使得通信更高效。所以公司内部服务的调用一般都用 RPC，而 HTTP 的优势在于通用，大家都认可这个协议。

所以**三方平台提供的接口都是通过 HTTP 协议调用的**。



![](https://song-oss.oss-cn-beijing.aliyuncs.com/golang_dream/article/static/%E6%89%AB%E7%A0%81_%E6%90%9C%E7%B4%A2%E8%81%94%E5%90%88%E4%BC%A0%E6%92%AD%E6%A0%B7%E5%BC%8F-%E7%99%BD%E8%89%B2%E7%89%88.png)
