## 第一章 常见数据结构的实现原理

#### channel: 通道， go独有的数据结构， 比之unix的管道更加高效常用来实现队列和锁机制

> 数据结构

```go
type hchan struct {
  qcount uint // 当前队列中剩余的元素个数
  dataqsiz uint //环形队列长度，即可以存放的元素个数
  buf unnsafe.Pointer // 环形队列指针
  elemsize uint16 // 每个元素的大小
  closed uint32 // 标识关闭状态
  elemtype *_type // 元素的类型
  sendx uint // 元素存进队列中的下标位置
  recvx uint // 下个被读取的元素在队列中的下标位置
  recvq waitq // 等待读消息的队列
  sendq waitq // 等待写消息的队列
  lock mutex // 锁， chan不支持并发读写
}
```

> chan使用会发生panic的几种情况

```
1 向关闭的chan写数据
2 关闭关闭的chan
3 关闭值为nil的chan
	example: 
	func TestCloseNilChan(t *testing.T){
	defer func() {
      if err := recover; err != nil {
        fmt.Println("close nil chan")
      }
	}
      var testNilChan chan struct{}
      Close(testNilChann) #Close(chan) 也是一种信号
	}
```

> chan发生阻塞需要注意的情况

```
1 no_cache chan是同步的，在写入数据时会等待读取数据后返回响应
	example:
	func TestBlockNoCacheChan(t *testing.T){
      var testBlockNoCacheChan = make(chan struct{})
      testBlockNoCacheChan <- struct{}{}
      fmt.Println("See you again")
      <- testBlockNoCacheChan
	}
2 读nil chan
3 写nil chan
```

> chan 同slice可用 len()|cap() 来查看长度和容量

> chan 单向通道使用限制 

```go
1 recvChan <- chan struct{} 读取管道数据
2 sendChan chan <- struct{} 写入管道数据
```

> chan 数据结构中 "closed uint32" - 用来标识chan的开启关闭状态 某些特定场景是否有用到在关闭chan之后重新复用该chan的情况 是否可以使用unsafe.Pointer来改变该chan的标识状态从而达到复用chan的情况

> chan 搭配使用

```
1 select {
  case <- testChan:
  case aaa := <- testChan:
  case aaa, ok := <- testChan:
  default:
}
2 for aaa := range testChan {
  
}
```

#### Slice: 切片 go独有的数据结构， 依托数组实现， 比数组灵活容易出错

> 数据结构

```go
type slice struct {
  array unsafe.Pointer
  len int
  cap int
}
```

> slice 切取遵循左闭右开原则，切取数组时可采用max封闭切片防止之后进行操作时将误改底层数组

```go
1 切取数组|切片 // 切取数组 cap <= len(数组) | 切取切片 cap <= cap(切片)
example:
func Test(t *testing.T) {
  var array = [6]int{1, 2, 3, 4, 5, 6}
  s1 := array[1:4]
  fmt.Println("s1:", s1) // 2, 3, 4
  fmt.Println("len(s1):", len(s1)) // 3
  fmt.Println("cap(s1):", cap(s1)) // 5
}
2 封印切片
example:
func Test(t *testing.T){
  var array = [6]int{1, 2, 3, 4, 5, 6}
  ####
  s1 := array[1:4]
  s1 := append(s1, 55)
  fmt.Println("array:", array) // 1, 2, 3, 4, 55, 6
  ####
  s1 := array[1:4:3] // 封印 如果追加数据容量不足会产生新的slice而不会覆盖原始的数组或切片
  s1 = append(s1, 55) // 生成新的slice
  fmt.Println("array:", array) // 1, 2, 3, 4, 5, 6
}
3 切取字符串 切取字符串会产生字符串而不是切片
example:
func Test(t *testing.T){
  str := "helloWorld!"
  newStr := str[:5]
  fmt.Printf("%v\n:", reflect.TypeOf(newStr)) // string
}
```

> slice 扩容

```
使用append进行数据追加时如果cap不足会发生扩容后生成一个新的slice slice的底层数组发生改变，当切片较小时会采用较大的扩容倍速，主要是可以避免频繁的进行扩容从而达到较少内存分配的次数和数据拷贝的代价；当切片较大时采用较小的扩容倍速从而达到避免空间浪费
1 < 1024 扩容1倍
2 >= 1024 1.25
```

#### Map  底层使用Hash表实现 键值对操作

> 数据结构

``` go
type hmap struct {
  count int
  B uint8
  buckets unsafe.Pointer
  oldbuckets unsafe.Pointer
  ...
}
```

> 发生panic的情况

```go
1 未初始化map 向nilMap写入值 (nilMap的长度同空map一致)
example:
func Test(t *testing.T){
  var nilMap = map[string]int
  var nullMap = map[string]int {}
  fmt.Println(len(nilMap)==len(nullMap)) // true
}
2 并发读写map （大多数场景不需要使用并发读写，可大大提升map速度 如有必要请使用锁或者sync提供的sync.map 可根据使用场景决定使用 mutex + map | sync.map）
```

> 可使用len(map)来查看map中键值对数量

> map扩容

```go
go map的bucket中规定能存储八个键值对（redis只允许一个）, 当发生hash冲突时，冲突的键值对会被分配到同一个bucket中，当bucket中的数据即将>8时会发生bucket溢出生成一个bucket溢出的bucket后使用指针指向溢出的bucket，这对map的存取效率是不利的。冲突不可避免需要采取措施。
1 负载因子 = 键数量/bucket数量  go 当负载因子大于6.5时，即平均每个bucket存取6.5个以上键值对时发生扩容或 overflow的数量达到 ？？？
	（1 过小 说明利用率不高
	（2 过大 说明冲突严重，存取效率低
2 扩容
	（1 增量扩容 扩容2倍
		使用hmap数据结构中的oldbuckets map实现 扩容时一次迁移过多数据会造成延时 所以采用当调用到改map时触发数据迁移（每次两个） 当map中该bucket负载因子为7时， 将oldbuckets指向buckets的指针 然后在buckets上新增bucket 将新数据写进新增bucket 而后每次调用该map时迁移两个数据直到oldbuckets中的数据迁移完毕后将oldbuckets中数据清除
	（2 等量扩容
		不新增buckets不扩容大小， 等量扩容针对数据不集中（可能是随机的也有可能是部分集中buckets中的数据被大量清除造成） 重新做一遍类似增量扩容的迁移操作 把松散的键值对重新排列一次（buckets中有多bucket但每个bucket中的键值对过少）
```

#### struct 

> 获取tag

```go
func Test(t *testing.T){
  type tmpTag struct {
    Key string `hellokiki:"key"`
    Value string `hellokiki:"value"`
  }
  var tag &tmpTag{}
  ty := reflect.TypeOf(tag)
  for i := 0; i < ty.NumField(); i++ {
    fmt.Printf("Field: %s, Tag:%s\n", ty.Field(i).Name, ty.Field(i).Tag.Get("hellokiki"))
  }
}
// Field: Key, Tag: key
// Field: Value, Tag: value
```

> string 和 []byte切片经常发生转换（数据结构也差不多, string转[]byte需要一次内存拷贝）

```go
1 数据结构
type stringStruct struct {
  str unsafe.Pointer // 字符串首地址
  len int // 字符串的长度
}
2 初始化string
var hello = "this is /"kiki/"!" = `this is kiki`
字符串生成时会 先构建stringStruct对象，再转换成string
源码：
func gostringnocopy(str *byte)string{
  // 先构造
  ss := stringStruct{str: unsafe.Pointer(str), len: findnull(str)}
  // 再将stringStruct转换成string
  s := *(*string)(unsafe.Pointer(&ss))
  return s
}
3 utf-8汉字占三个字节
4 字符串拼接 
	(1) []string -> concatstrings()
	(2) 遍历[]string 获取长度申请内存
	(3) copy
	func concatstrings(a []string)string{
      length := 0
      for _, str := range a {
        length += len(str)
      }
      // 分配内存，返回一个string和切片，二者共享内存空间
      s, b := rawstring(length)
      // string 无法修改， 只能通过切片修改
      for _, str := range a {
        copy(b, str)
        b = b[len(str):]
      }
      return s
	}
	func rawstring(size int)(s string, b []byte){
      p := mallocgc(uintptr(size), nil, false)
      stringStructOf(&s).str = p
      stringStructOf(&s).len = size
      *(*slice)(unsafe.Pointer(&b)) = slice(p, size, size)
      return
	}
```

## 第二章 控制结构

#### select  作用于channel

> 数据结构

```go
type scase struct {
  c *hchan // 操作的管道
  kind uint16 // case 类型
  elem unsafe.Pointer // data elemen
}
1 scase中只有一个操作的管道决定了每次只能处理一个管道
2 kind case语句的类型 
	const (
    	caseNil = iota  // 管道的值为nil
      	caseRecv  		// 读管道的case
      	caseSend		// 写管道的case
      	caseDefault 	// default
    )
3 类型为caseNil表示其操作的管道值为nil，nil管道不可读也不可写，意味着该case永远不会命中，所以在管道中向nilchan写数据不会panic
4 elem根据kind类型的不同而不同 在类型为caseRecv的case中，elem表示从管道读出的数据的存放地址 caseSend-> 将写入管道的数据的地址
5 selectgo()用于实现select -> P70
```

#### for range

> range chan时只有一个返回值，因为chan没有下标

## 第三章 协程

#### 进程

> 进程是应用程序的启动实例， 每个进程都有独立的内存空间， 不同进程通过进程间的通信方式来通信

#### 线程

> 线程从属于进程， 每个进程至少包含一个线程， 线程是cpu调度的基本单位， 多个线程之间可以通过共享进程的的资源并通过共享内存等线程间的通信方式来通信

#### 协程

> 协程可以理解成为一种轻量级线程， 与线程相比， 协程不受操作系统调度， 协程调度器由用户应用程序提供， 协程调度器按照调度策略把协程调度到线程中运行。

#### 使用协程的优势

> 我们都知道线程是由系统进行调度，频繁的进行创建和切换会造成大量的不必要开销， 所有就有了线程池技术的产生，为了尽可能的复用线程造成不必要的开销从而达到提升效率的实现。但是如果线程池中的线程都发生了系统调用阻塞，任务堆积消费能力会极大的降低。增减线程池中线程的数量能够得到一定的缓冲，但是过多的线程争抢cpu资源，消费能力会有上限甚至降低。而使用用户态的协程则能很好的处理此种问题。在线程下又分协程，将资源交给协程处理。这样当发生系统调用阻塞情况时，则可以将该阻塞的协程调度出线程，执行线程下其他协程从而避免线程的频繁切换提升速度。

#### GMP调度

> M:工作线程
>
> P:处理器，上下文
>
> G：执行的代码

> 都说任何东西来源于我们的生活。我愿意将M比作为提供建筑工地上负责某一块工作的团队， P为包工头，G为工作。 每一个团队M只有通过包工头P才能接领工作（M必须持有P才可以执行代码）M有时侯可能会因为材料问题而停止工作等待材料施工（系统调用，阻塞）。M的个数通常会比P多。每一个P都有自己的工作队列，还有一个全局队列（P自己本身的项目，自己本身的项目不用发生竞争-工作队列，全局队列需要竞争，拿全局项目需要竞标）

调度策略

+ 队列轮转

  > 就是将本地队列中的G一个个执行，同时会周期性的去查看全局队列是否有等待的G然后添加到自己的队列避免全局G饿死（全局G一般都是阻塞恢复的， 周期性的去查看避免全局G长时间得不到调度饿死

+ 系统调用

  > 当G发生系统调用阻塞时， M放弃P并且其他冗余的M会接管P的G并继续执行

+ 工作量切取

  > 当本地队列无任务执行时会从其他队列切取一般的G放到本地队列执行

+ 抢占式调度

  > 避免单个任务执行过长而阻碍其他任务被调度的机制


## 第四章 内存管理

#### 内存分配

#### 垃圾回收

> + go使用的垃圾回收算法是 标记-清除（三色标记法） 简单来说就是通过给对象添加标记 对象如果存在被引用则最终标记为黑色，而不被引用到的对象为最初的白色，最后触发gc时将白色（未被引用对象的内存进行回收）
> + 三色标记 从root对象开始标记  a -> root| b - root| c -> b 
>   + root - > 灰色
>   + root -> 黑色| b -> 灰色
>   + a c  -> gc
> + stop the world (std) std的长短直接影响应用的效率 也是标记清除法的诟病 因为在进行gc时需要停止所有的groutine专心做gc回收，因为如果gc过程有新引用产生会对gc的结果造成误差 如gc过程中为白色标记的对象突然被引用到但是却被gc等操作
> + std优化-写屏障 一种让gc可以和groutine同时运行的手段，虽然不能完全消除std但是也大大缩短了std的时间。 即新分配内存的对象留待下次gc回收，本次跳过
> + gc触发时间一般为2分钟一次以及手动触发(runtime.GC()) 
> + gc性能优化思路之一： 减少对象分配的次数如对象复用或使用大对象组合小对象 sync.pool池就是其实现的案例之一  当对象过多需要反复创建时可以使用pool来复用对象优化gc
>
>

#### 逃逸分析

> 逃逸分析是指由编译器决定内存分配的位置， 不需要程序员指定。在函数中申请一个新的对象
>
> + 如果分配在栈中， 则函数执行结束后可自动将内存回收
> + 如果分配在堆中， 则函数执行结束后可交给gc（垃圾回收）处理

+ 逃逸策略 当在函数中申请新的对象时，编译器会根据对象是否存在外部引用决定是否逃逸

  + 如果函数外部没有引用， 则分布在栈中
  + 如果函数外部存在应用， 则分布在堆中
  + 一般情况下按照上诉两种情况分配，但是某些时候外部没有存在外部引用的情况时也有可能发生逃逸 如内部对象申请的内存过大， 栈无法满足则可能放在堆中

+ 逃逸场景

  + 指针逃逸 函数内部定义指针并且返回会发生逃逸，其指针指向的内存地址不会存放在栈中而是在堆中

  ```
  func example() *int {
    return new(int)
  }
  ```

  + 栈空间不足逃逸 申请的内存过大， 栈内存无法满足
  + 动态类型逃逸 很多函数的参数为interface 编译期间无法确定参数的具体类型

  ```go
  func example() {
    s := "hello"
    fmt.Println(s)
  }
  ```

  + 闭包引用对象逃逸 闭包作为一个函数使用原函数中的对象发生引用

  ```go
  func exatple() {
    a := "hello"
    go func () {
      b := a
    }
  }
  ```

  ​


## 第五章 并发控制

#### WaitGroup

> waitGroup结构体

```go
type WaitGroup struct {
  statel [3]uint32
}
statel是一个长度为3的数组， 包含了state和一个信号量， state实际上是两个计数器
counter: 当前还未执行结束的goroutine计数器
waiter count:等待goroutine-group结束的goroutine数量， 既有多少个等待者
semaphore：信号量
```

> waitGroup对外开放了三个方法
>
> + Add(delta int):将delta值加到counter中（delta可为负值）
> + Wait():waiter递增1，并阻塞等待信号量semaphore
> + Done():counter递减1， 按照waiter数值释放相应次数的信号量（Done()==Add(-1))
>
> Add(int) ->  counter == 0 ?  |  N -> return
>
> ​						|  Y -> for ; waiter !=0; waiter-- {runtime_Semrelease(semap, false)}(发送信号量唤醒)
>
> Wait() -> waiter++ -> runtime_Semrecquire(semap)(等到被唤醒)

> waitGroup 源码实现 使用逻辑运算uint64前32位表示waiter的数量 后32位表示counter的数量->bitmap实现方式

> 使用管道传递信号实现 单单通过判断counter的数量后循环查询counter的数值也可以实现 对比？

#### context

> 接口定义

```go
type Context interface {
  Deadline() (deadline time.Time, ok bool)// 返回deadline和是否已设置deadline的标识 默认为false
  Done() <-chan struct{}// 返回一个探测Context是否取消的channel， 当Context取消时会自动将该channel关闭
  Err() error // 返回context关闭的原因
  Value(key interface{}) interface{}// 用于在goroutine中传值
}
```

> 空context-> emptyCtx 空的context本身不包含任何值仅用于其他context的父节点
>
> type emptyCtx int
>
> context包中定义了一个公用的emptyCtx全局变量-> backgroud 可以使用context.Background()获取

> context提供了四个不同类型的context，使用这四个方法如果没有父节点需要传入background,即将background作为父节点

+ WithCancel()
+ WithDeadline()
+ WithTimeout()
+ WithValue()

>interface Context | struct emptyCtx 
>
>​				| struct cancelCtx| WithCancel()
>
>​				| struct valueCtx  | WithValue()
>
>​				| struct timerCtx | WithTimeout()
>
>​							       | WithDeadline()

> cancelCtx

```go
type cancelCtx struct {
  Context
  
  mu sync.Mutex
  done chan struct{}
  children map[canceler]struct // 记录所有由此context派生的child（派生！= 传递）
  err error // 记录错误的类型
}
cancelCtx与deadline和value无关， 只需要实现Done() 和 Err() 外露接口即可
```

+ Done() 接口实现 只需返回cancelCtx.done即可(第一个done()未初始化，所以需要作判断进行初始化操作--cancelCtx没有初始化函数)

  + ```go
    func (c *cancelCtx)Done() <-chan struct{} {
      c.mu.Lock()
      if c.done == nil {
        c.done = make(chan struct{}) // no_cache_channel 管道关闭时所有监听该管道都会得到关闭管道的信号
        d := c.done
        c.mu.Unlock()
        return d
      }
    }
    ```

+ Cancel() 方法实现伪代码

  + ```go
    func (c *cancelCtx) cancel(removeFromParent bool, err error) {
      c.mu.Lock()
      c.err = err
      close(c.done) // 关闭所有传递的ctx
      
      for child := range c.children {
        child.cancel(fasle, err) // 关闭派生child
      }
      c.children = nil 
      c.mu.Unlock()
      
      if removeFromParent {
        removeChild(c.Context, c) // 将自己从父节点移除
      }
    }
    ```

  + ```go
    example 1:
    func TestWithCancelContext(t *testing.T) {
    	ctx, _ := context.WithCancel(context.Background())
    	ctx2, cancelFunc := context.WithCancel(ctx)
    	go func(ctx context.Context) {
    		select {
    		case <- ctx.Done():
    			fmt.Println("p ctx")
    		}
    	}(ctx)

    	go func(ctx context.Context) {
    		select {
    		case <- ctx.Done():
    			fmt.Println("c ctx")
    		}
    	}(ctx2)

    	cancelFunc()
    	for {
    		time.Sleep(time.Second * 100)
    	}
    }
    result：c ctx

    example 2:
    func TestWithCancelContext(t *testing.T) {
    	ctx, cancelFunc := context.WithCancel(context.Background())
    	ctx2, _ := context.WithCancel(ctx)
    	go func(ctx context.Context) {
    		select {
    		case <- ctx.Done():
    			fmt.Println("p ctx")
    		}
    	}(ctx)

    	go func(ctx context.Context) {
    		select {
    		case <- ctx.Done():
    			fmt.Println("c ctx")
    		}
    	}(ctx2)

    	cancelFunc()
    	for {
    		time.Sleep(time.Second * 100)
    	}
    }
    result：p ctx
    ```

> WithCancel() 方法的实现

+ 初始化一个cancelCtx实例

+ 将cancelCtx实例添加到其父节点的children中（如果父节点也可以被“cancel”）

+ 返回cancelCtx实例和cancel()方法

+ 实现源码

  + ```go
    func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
      c := newCancelCtx(parent)
      propagateCancel(parent, &c)
      return &c, func() {c.cancel(true, Canceled)}
    }
    ```

+ 将自身添加到父节点的过程

  + 如果父节点也支持cancel， 也就是说其父节点肯定有children成员， 那么把新context添加到children中即可
  + 如果父节点不支持cancel， 就继续向上查询， 知道找到一个支持cancel的节点， 把新context添加到children中
  + 如果所有的父节点均不支持cancel， 则启动一个协程等待父节点结束， 再把当前context结束

> timerCtx

```go
type timerCtx struct {
  cancelCtx
  timer *time.Timer
  deadline time.Time
}
```

>  withDeadLine()方法的实现

+ 初始化一个timerCtx实例
+ 将timerCtx实例添加到其父节点的children中（如果父节点也可以被“cancel”)
+ 启动定时器， 定时器到期后会自动“cancel” 本context
+ 返回timerCtx实例和cancel（）方法

> withTimeout()方法的实现

```go
func WithTimeout(parent Context, timeout time.Druation)(Context, CancelFunc){
  return WithDeadLine(parent, time.Now().Add(timeout))
}
```

> valueCtx

```go
type valueCtx struct {
  Context
  key, val interface{} // 没有实现Cancel() Err()
}
当前context查找不到key时， 会向父节点查找， 如果查询不到则最终返回interface{}, 也就是说可以通过context查询到父节点的value值
```

> 典型使用案例

```go
func HandelRequest(ctx context.Context) {
  for {
    select {
      case <- ctx.Done(): // 永远无法返回， 需要返回的话 需要在创建时指定可以cancel()的context作为parent 如下 "这样才有效" 
      default:
      fmt.Println("HandelRequest runnint, parameter:", ctx.Value("parameter"))
    }
  }
}
func main() {
  var ctxx, cancel := context.WithCancel(context.Background) // 这样
   ctx := context.WithValue(ctxx, "parameter", 1) // 才
  cancel() // 有效
  ctx := context.WithValue(context.Background(), "parameter", 1)
  go HandelRequest(ctx)
  select {}
}
```

## Mutex

> 结构体

```go
type Mutex struct {
  state int32 // 一分为四 waiter 29bit| starving 1bit| woken 1bit| locked 1bit
  sema uint32 // 标识信号量
}
locked 1 表示加锁| 0 未加锁
woken 有协程自旋等待加锁时置为1 用于起到有协程解锁时无需释放信号量 直接为自旋协程加锁
starving normal正常模式（可以有自旋）| straving 饥饿模式（不可有自旋）
waiter 等待加锁的协程（用于释放信号量告知协程可以抢锁了）
```

> 自旋

> RWMutex

+ 写操作是如何阻止写操作的

  > 读写锁包含一个互斥锁，谁拿到锁谁写

+ 写操作是如何阻止读操作的

  > 读操作维持着一个readerCount的计数，理论上最大为2的30次方个。写操作时将readerCount-理论上最大的数，得到一个负值，当有读锁定来时检测到为负值便知道有写操作在进行，阻塞。而真实得读操作个数并不会丢失，只需要将readerCount加上理论上最大的个数即可。

+ 读操作是如何组织写操作的

  > 读锁定会将readerCount的计数加一， 即readerCount的值大于零时，写锁定便知道有读锁定在进行

+ 为什么写锁定不会被 “饿死”

  > 如果有读锁定在进行，然后一直陆陆续续有读锁定进来，那么写锁定不是一直等不到读锁定释放被饿死吗？这种情况不会发生的。当写锁定来时会维护一个readerWait的计数。会将readerCount的值复制到readerWait的值中，然后每一个读锁定解锁readerWait和readerCount就会减一，当readerWait为0时就会唤醒写锁定进行锁定。这样写锁定就不会饿死，并且还能维持readerCount中读锁定的计数。当写锁定完成后就去唤醒读锁定。





