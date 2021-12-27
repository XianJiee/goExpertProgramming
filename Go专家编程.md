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





