# Golang 进阶语法学习笔记

本文是系列文章《从 PHP 到 AI + Golang，程序员自救转型手记》作者的学习笔记，完整版开源于：[github](https://github.com/ai-go-hub/ai-go-blog) | [gitee](https://gitee.com/ai-go-hub/ai-go-blog)，实操项目 ai-go-mall 开源于：[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)

#### 接口

基本接口(Basic Interface)：只包含方法集的接口就是基本接口
通用接口(General Interface)：只要包含类型集的接口就是通用接口

以下代码，被认为是 CraneA 结构体实现了 Crane 接口
```go
// 定义起重机接口
type Crane interface {
	JackUp() string
	Hoist() string
}

// 起重机A结构体
type CraneA struct {
	name string
}

// 自有方法
func (c CraneA) Work() {
    fmt.Println("使用了 技术A")
}

// 实现接口方法
func (c CraneA) JackUp() string {
	c.Work()
	return "jackup"
}
func (c CraneA) Hoist() string {
	c.Work()
	return "hoist"
}

// 建筑公司的起重机结构体
type ConstructionCompany struct {
	Crane Crane // 只根据 Crane 类型来存放起重机
}
// 为建筑公司结构体提供构建方法
func (c *ConstructionCompany) Build() {
	fmt.Println(c.Crane.JackUp())
	fmt.Println(c.Crane.Hoist())
	fmt.Println("建筑完成")
}
```

#### 泛型（参数化多态）

指通过类型参数化来实现代码的复用与灵活性。

```go
// 基础语法
func Sum[T int | float64](a, b T) T {
   return a + b
}

Sum[int](2012, 2022)

// 泛型切片
type GenericSlice[T int | int32 | int64] []T
GenericSlice[int]{1, 2, 3}

type GenericMap[K comparable, V int | string | byte] map[K]V
gmap1 := GenericMap[int, string]{1: "hello world"}
gmap2 := make(GenericMap[string, byte], 0)

// 泛型结构体
type GenericStruct[T int | string] struct {
   Name string
   Id   T
}

GenericStruct[int]{
   Name: "jack",
   Id:   1024,
}
GenericStruct[string]{
   Name: "Mike",
   Id:   "1024",
}

// 使用 ~ 符号表示底层类型，如果一个类型的底层类型属于该类型集，那么该类型就属于该类型集（底层类型相同，就算同一类型）
type Int interface {
   ~int8 | ~int16 | ~int | ~int32 | ~int64 | ~uint8 | ~uint16 | ~uint | ~uint32 | ~uint64
}
```

#### 类型

```go
// 类型别名
type Int = int

type TwoDMap = map[string]map[string]int
func PrintMyMap(mymap TwoDMap) {
   fmt.Println(mymap)
}

// 类型转换
int(x)
float64(x)
(func() int)(x)  // 将x转换为类型 func() int

// 类型断言
var a any = 1
ints, ok := a.(int)
fmt.Println(ints, ok)

// 类型判断
var a interface{} = 2
switch a.(type) {
    case int: fmt.Println("int")
    case float64: fmt.Println("float")
    case string: fmt.Println("string")
}
```

#### 错误
error：正常的流程出错，需要处理，直接忽略掉不处理程序也不会崩溃
panic：很严重的问题，程序应该在处理完问题后立即退出
fatal：非常致命的问题，程序应该立即退出，一般是被动触发，很少主动去触发它：`os.Exit(1)`

```go
if file, err := os.Open("README.txt"); err != nil {
    fmt.Println(err)
    return
}
fmt.Println(file.Name())

// errors.Unwrap 用于解包一个错误
// errors.Is 判断错误链中是否包含指定的错误
// errors.As 函数的作用是在错误链中寻找第一个类型匹配的错误，并将值赋值给传入的err
```

错误恢复，如下调用者完全不知道 dangerOp() 函数内部发生了 panic

```go
func dangerOp() {
	defer func() {
		if err := recover(); err != nil {
			fmt.Println(err)
			fmt.Println("panic 恢复")
		}
	}()
	panic("发生 panic")
}
```

错误恢复注意事项：
1. 必须在 defer 中使用
2. 多次使用也只会有一个能恢复 panic
3. 闭包 recover 不会恢复外部函数的任何 panic
4. panic 的参数禁止使用 nil


#### 文件
- os库，负责 OS 文件系统交互的具体实现
- io库，读写 IO 的抽象层
- fs库，文件系统的抽象层

其他使用时再查阅文档，只是库的使用

#### 反射
反射是一种在运行时检查语言自身结构的机制，它可以很灵活的去应对一些问题，但同时带来的弊端也很明显，例如性能问题等等。

```go
str := "hello world!"
reflectType := reflect.TypeOf(str)
fmt.Println(reflectType)

// 获取变量的基础类型
fmt.Println(reflectType.Kind())

// 获取 map 键/值的反射类型
var eface any
eface = map[string]int32{}
rType := reflect.TypeOf(eface)
fmt.Println(rType.Key().Kind())
fmt.Println(rType.Elem().Kind())

// 获取指针指向元素的反射类型（对于数组，切片，通道用使用起来都是类似的）
var eface any
eface = new(strings.Builder)
rType := reflect.TypeOf(eface)

// 拿到指针所指向元素的反射类型
vType := rType.Elem()
// 输出包路径
fmt.Println(vType.PkgPath())
// 输出其名称
fmt.Println(vType.Name())

// 对应类型所占的字节大小
reflect.TypeOf(0.1).Size()

// 类型是否可以被比较
reflect.TypeOf(1024).Comparable()

// 一个类型是否实现了某一接口
rIface := reflect.TypeOf(new(MyInterface)).Elem()
fmt.Println(reflect.TypeOf(new(MyStruct)).Elem().Implements(rIface))
fmt.Println(reflect.TypeOf(new(HisStruct)).Elem().Implements(rIface))

// 是否可以转换为指定的类型
rIface := reflect.TypeOf(new(MyInterface)).Elem()
fmt.Println(reflect.TypeOf(new(MyStruct)).Elem().ConvertibleTo(rIface))
fmt.Println(reflect.TypeOf(new(HisStruct)).Elem().ConvertibleTo(rIface))

// 反射接口的值
str := "hello world!"
reflectValue := reflect.ValueOf(str)
fmt.Println(reflectValue)

// 获取反射值的类型
num := 114514
rValue := reflect.ValueOf(num)
fmt.Println(rValue.Type())

// 获取一个反射值的元素反射值
num := new(int)
*num = 114514
rValue := reflect.ValueOf(num).Elem()
fmt.Println(rValue.Interface())

// 函数
func Max(a, b int) int {
	if a > b {
		return a
	}
	return b
}

func main() {
	rType := reflect.TypeOf(Max)
	// 输出函数名称,字面量函数的类型没有名称
	fmt.Println(rType.Name())
	// 输出参数，返回值的数量
	fmt.Println(rType.NumIn(), rType.NumOut())
	rParamType := rType.In(0)
	// 输出第一个参数的类型
	fmt.Println(rParamType.Kind())
	rResType := rType.Out(0)
	// 输出第一个返回值的类型
	fmt.Println(rResType.Kind())
}
```

#### 并发

```go
// 具有返回值的内置函数不允许跟随在 go 关键字后面
go make([]int,10) //  go discards result of make([]int, 10) (value of type []int)

// 三种启动协程的方式
func main() {
	go fmt.Println("hello world!")
	go hello()
	go func() {
		fmt.Println("hello world!")
	}()

    // 休眠 1ms，否则什么都不会输出，因为在协程内进行输出时，主线程可能已经运行结束
    time.Sleep(time.Millisecond)
}
func hello() {
	fmt.Println("hello world!")
}
```

并发控制方法有三种：
channel：管道，更适合协程间通信
WaitGroup：信号量，可以动态的控制一组指定数量的协程
Context：上下文，更适合子孙协程嵌套层级更深的情况

传统的锁控制：
Mutex：互斥锁
RWMutex ：读写互斥锁

##### channel
一种在协程间通信的解决方案，同时可以用于并发控制。管道中的数据流动方式与队列一样，即先进先出（FIFO），协程对于管道的操作是同步的，在某一个时刻，只有一个协程能够对其写入数据，同时也只有一个协程能够读取管道中的数据。

```go
var Intch chan int

// 管道值类型 string，管道缓冲大小 1
ch := make(chan string, 1)

// 向管道写入数据
go func() {
    ch <- "123"
}()
// 读取管道数据
fmt.Println(<-ch)

// 使用完记得关闭管道
defer close(ch)

// 读取操作时，还有第二个返回值，表示是否读取成功
strs, ok := <-ch

// 以下情况会导致协程阻塞
// 1. 当对一个无缓冲管道直接进行同步读写操作都会导致该协程阻塞
intCh := make(chan int)
defer close(intCh)
intCh <- 1
ints, ok := <-intCh
fmt.Println(ints, ok)

// 2. 读取空缓冲区的管道
intCh := make(chan int, 1)
defer close(intCh)
ints, ok := <-intCh // 缓冲区为空，阻塞等待其他协程写入数据
fmt.Println(ints, ok)

// 3. 写入满缓冲区的管道
intCh := make(chan int, 1)
defer close(intCh)

intCh <- 1
intCh <- 1 // 满了，阻塞等待其他协程来读取数据

// 只写通道
c <-chan Time
// 只读通道
c chan<- Type
// 只读通道使用示例，只能对管道发送数据
func write(ch chan<- int) {
   ch <- 1
}

// 4. 管道为 nil，无论怎样都会导致当前协程阻塞
var intCh chan int
intCh <- 1
fmt.Println(<-intCh)
```

##### WaitGroup
WaitGroup 即等待执行，使用它可以很轻易的实现等待一组协程的效果。
其内部的实现是计数器+信号量，程序开始时调用 Add 初始化计数，每当一个协程执行完毕时调用 Done，计数就-1，直到减为 0，而在此期间，主协程调用 Wait 会一直阻塞直到全部计数减为 0，然后才会被唤醒。

```go
func main() {
	var wait sync.WaitGroup
	// 指定子协程的数量
	wait.Add(1)
	go func() {
		fmt.Println(1)
		// 执行完毕
		wait.Done()
	}()
	// 等待子协程
	wait.Wait()
	fmt.Println(2)
}

func main() {
	var mainWait sync.WaitGroup
	var wait sync.WaitGroup
	// 计数10
	mainWait.Add(10)
	fmt.Println("start")
	for i := 0; i < 10; i++ {
		// 循环内计数1
		wait.Add(1)
		go func() {
			fmt.Println(i)
			// 两个计数-1
			wait.Done()
			mainWait.Done()
		}()
		// 等待当前循环的协程执行完毕
		wait.Wait()
	}
	// 等待所有的协程执行完毕
	mainWait.Wait()
	fmt.Println("end")
}
```

##### Context
Context 译为上下文，是 Go 提供的一种并发控制的解决方案，相比于管道和 WaitGroup，它可以更好的控制子孙协程以及层级更深的协程。

###### emptyCtx:Background/TODO
- 空的上下文，context 包下所有的实现都是不对外暴露的，但是提供了对应的函数来创建上下文。
- 通常是用来当作最顶层的上下文，在创建其他三种上下文时作为父上下文传入，一般不直接使用它本身。

###### valueCtx:WithValue
- 用于存储键值对的上下文类型，它允许你在上下文树中传递请求范围的值。
- 每次调用 WithValue() 都会创建一个新的 valueCtx，不会修改现有上下文。
- 当调用 Value() 查询值时，会从当前 valueCtx 开始向上查找，直到找到匹配的键或到达根上下文。
- 一般用于传递请求范围的元数据（如认证信息、请求ID、跟踪ID等）、在服务间传递配置参数、实现跨组件的上下文感知

```go
func main() {
	// 1. 创建根上下文
	rootCtx := context.Background()

	// 2. 添加键值对
	ctx1 := context.WithValue(rootCtx, "user", "alice")
	ctx2 := context.WithValue(ctx1, "request_id", "12345")
	ctx3 := context.WithValue(ctx2, "role", "admin")

	// 3. 从不同层级获取值
	fmt.Println("Request ID:", getValue(ctx3, "request_id"))    // 12345
	fmt.Println("User:", getValue(ctx3, "user"))                // alice
	fmt.Println("Role:", getValue(ctx3, "role"))                // admin
	fmt.Println("Non-existent:", getValue(ctx3, "nonexistent")) // nil

	// 4. 从中间层级获取值
	fmt.Println("From ctx2 - User:", getValue(ctx2, "user")) // alice
	fmt.Println("From ctx2 - Role:", getValue(ctx2, "role")) // nil (未设置)
}

// 辅助函数：安全获取值
func getValue(ctx context.Context, key string) any {
	if val := ctx.Value(key); val != nil {
		return val
	}
	return nil
}
```

###### cancelCtx:WithCancel

cancelCtx 译为可取消的上下文，创建时，如果父级实现了 canceler，就会将自身添加进父级的 children 中，否则就一直向上查找。如果所有的父级都没有实现 canceler，就会启动一个协程等待父级取消，然后当父级结束时取消当前上下文。当调用 cancelFunc 时，Done 通道将会关闭，该上下文的任何子级也会随之取消，最后会将自身从父级中删除。

```go
var waitGroup sync.WaitGroup
func main() {
	bkg := context.Background()
	// 返回了一个cancelCtx和cancel函数
	cancelCtx, cancel := context.WithCancel(bkg)
	waitGroup.Add(1)
	go func(ctx context.Context) {
		defer waitGroup.Done()
		for {
			select {
			case <-ctx.Done():
				fmt.Println(ctx.Err())
				return
			default:
				fmt.Println("等待取消中...")
			}
			time.Sleep(time.Millisecond * 200)
		}

	}(cancelCtx)
	time.Sleep(time.Second)

    // 调用取消方法
	cancel()
	waitGroup.Wait()
}
```

###### timerCtx:WithTimeout/Deadline

timerCtx 在 cancelCtx 的基础之上增加了超时机制。

```go
var wait sync.WaitGroup
func main() {
	deadline, cancel := context.WithDeadline(context.Background(), time.Now().Add(time.Second*5))
	defer cancel()
	wait.Add(1)
	go func(ctx context.Context) {
		defer wait.Done()
		for {
			select {
			case <-ctx.Done():
				fmt.Println("上下文取消", ctx.Err())
				return
			default:
				fmt.Println("等待取消中...")
			}
			time.Sleep(time.Millisecond * 200)
		}
	}(deadline)
	wait.Wait()
}
```

##### Select
select 是一种管道多路复用的控制结构。什么是多路复用，简单的用一句话概括：在某一时刻，同时监测多个元素是否可用，被监测的可以是网络请求，文件 IO 等。在 golang 中，select 只能检测管道。

- 在 select 的 case 中对值为 nil 的管道进行操作的话，并不会导致阻塞，该 case 则会被忽略，永远也不会被执行。

```go
func main() {
	chA := make(chan int)
	chB := make(chan int)
	chC := make(chan int)
	defer func() {
		close(chA)
		close(chB)
		close(chC)
	}()

	l := make(chan struct{})

	go Send(chA)
	go Send(chB)
	go Send(chC)

    // 在新的协程内执行 for
	go func() {
	Loop:
		for {
			select {
			case n, ok := <-chA:
				fmt.Println("A", n, ok)
			case n, ok := <-chB:
				fmt.Println("B", n, ok)
			case n, ok := <-chC:
				fmt.Println("C", n, ok)
			case <-time.After(time.Second): // 一个超时管道，设置1秒的超时时间
				break Loop // 退出循环
			}
		}
		l <- struct{}{} // 告诉主协程可以退出了
	}()

	<-l
}

func Send(ch chan<- int) {
	for i := range 3 {
		time.Sleep(time.Millisecond)
		ch <- i
	}
}


// 非阻塞的判断一个 context 是否已经结束
func IsDone(ctx context.Context) bool {
	select {
	case <-ctx.Done():
		return true
	default:
		return false
	}
}
```

##### 锁

Go 所提供的锁都是非递归锁，也就是不可重入锁，所以重复加锁或重复解锁都会导致 fatal

```go
Lock()
// 在这个过程中，数据不会被其他协程修改
Unlock()
```

###### 互斥锁

启了 10 协程分别修改 count 的值，如果不加上，count 的值会被各个进程改得乱七八糟

```go
var count = 0
var wait sync.WaitGroup
var lock sync.Mutex

func main() {
	wait.Add(10)
	for i := 0; i < 10; i++ {
		go func(data *int) {
			// 加锁
			lock.Lock()
			// 模拟访问耗时
			time.Sleep(time.Millisecond * time.Duration(rand.Intn(1000)))
			// 访问数据
			temp := *data
			// 模拟计算耗时
			time.Sleep(time.Millisecond * time.Duration(rand.Intn(1000)))
			ans := 1
			// 修改数据
			*data = temp + ans
			// 打印当前协程修改后的值
			fmt.Println(*data)
			// 解锁
			lock.Unlock()
			wait.Done()
		}(&count)
	}
	wait.Wait()
	fmt.Println("最终结果", count)
}
```

###### 读写锁

对于一个协程而言：
- 如果获得了读锁，其他协程进行写操作时会阻塞，其他协程进行读操作时不会阻塞
- 如果获得了写锁，其他协程进行写操作时会阻塞，其他协程进行读操作时会阻塞

```go
// 加读锁
func (rw *RWMutex) RLock()

// 尝试加读锁（非阻塞的，加锁失败会返回 false）
func (rw *RWMutex) TryRLock() bool

// 解读锁
func (rw *RWMutex) RUnlock()

// 加写锁
func (rw *RWMutex) Lock()

// 尝试加写锁
func (rw *RWMutex) TryLock() bool

// 解写锁
func (rw *RWMutex) Unlock()
```

###### 条件变量
条件变量，与互斥锁一同出现和使用，所以有些人可能会误称为条件锁，但它并不是锁，是一种通讯机制。

```go
// 创建条件变量的函数签名
func NewCond(l Locker) *Cond

// 阻塞等待条件生效，直到被唤醒
func (c *Cond) Wait()

// 唤醒一个因条件阻塞的协程
func (c *Cond) Signal()

// 唤醒所有因条件阻塞的协程
func (c *Cond) Broadcast()
```

```go
var count = 0
var rw sync.RWMutex
var wait sync.WaitGroup

// 条件变量
var cond = sync.NewCond(rw.RLocker())

func main() {
	wait.Add(12)
	// 读多写少
	go func() {
		for i := 0; i < 3; i++ {
			go Write(&count)
		}
		wait.Done()
	}()
	go func() {
		for i := 0; i < 7; i++ {
			go Read(&count)
		}
		wait.Done()
	}()
	// 等待子协程结束
	wait.Wait()
	fmt.Println("最终结果", count)
}

func Read(i *int) {
	time.Sleep(time.Millisecond * time.Duration(rand.Intn(500)))
	rw.RLock()
	fmt.Println("拿到读锁")
	// 使用 for，条件不满足就一直阻塞
	for *i < 3 {
		cond.Wait()
	}
	time.Sleep(time.Millisecond * time.Duration(rand.Intn(1000)))
	fmt.Println("释放读锁", *i)
	rw.RUnlock()
	wait.Done()
}

func Write(i *int) {
	time.Sleep(time.Millisecond * time.Duration(rand.Intn(1000)))
	rw.Lock()
	fmt.Println("拿到写锁")
	temp := *i
	time.Sleep(time.Millisecond * time.Duration(rand.Intn(1000)))
	*i = temp + 1
	fmt.Println("释放写锁", *i)
	rw.Unlock()
	// 唤醒所有因条件变量阻塞的协程
	cond.Broadcast()
	wait.Done()
}
```

##### sync 包

###### Once
顾名思义，Once 译为一次，sync.Once 保证了在并发条件下指定操作只会执行一次。

签名：
```go
func (o *Once) Do(f func())
```

示例：
```go
var wait sync.WaitGroup

func main() {
	var slice MySlice
	wait.Add(4)
	for range 4 {
		go func() {
			slice.Add(1)
			wait.Done()
		}()
	}
	wait.Wait()
	fmt.Println(slice.Len())
}

type MySlice struct {
	s []int
	o sync.Once
}

func (m *MySlice) Add(i int) {
	// 当真正用到切片的时候，才会考虑去初始化
	m.o.Do(func() {
		fmt.Println("初始化")
		if m.s == nil {
			m.s = make([]int, 0, 10)
		}
	})
	m.s = append(m.s, i)
}

func (m *MySlice) Len() int {
	return len(m.s)
}
```

###### Pool
sync.Pool 的设计目的是用于存储临时对象以便后续的复用，是一个临时的并发安全对象池，将暂时用不到的对象放入池中，在后续使用中就不需要再额外的创建对象可以直接复用，减少内存的分配与释放频率，最重要的一点就是降低 GC 压力。

注意：
1. 临时对象：sync.Pool 只适合存放临时对象，池中的对象可能会在没有任何通知的情况下被 GC 移除，所以并不建议将网络链接，数据库连接这类存入 sync.Pool 中。
2. 不可预知：sync.Pool 在申请对象时，无法预知这个对象是新创建的还是复用的，也无法知晓池中有几个对象。
3. 并发安全：官方保证 sync.Pool 一定是并发安全，但并不保证用于创建对象的 New 函数就一定是并发安全的，New 函数是由使用者传入的，所以 New 函数的并发安全性要由使用者自己来维护，这也是为什么上例中对象计数要用到原子值的原因。
4. 当使用完对象后，一定要释放回池中，如果用了不释放那么对象池的使用将毫无意义。

签名：
```go
// 申请一个对象
func (p *Pool) Get() any

// 放入一个对象
func (p *Pool) Put(x any)

// 用于对象池在申请不到对象时初始化一个对象
New func() any
```

使用：
```go
var wait sync.WaitGroup

// 临时对象池
var pool sync.Pool

// 用于计数过程中总共创建了多少个对象
var numOfObject atomic.Int64

// BigMemData 假设这是一个占用内存很大的结构体
type BigMemData struct {
	M string
}

func main() {
	pool.New = func() any {
		numOfObject.Add(1)
		return BigMemData{"大内存"}
	}
	wait.Add(1000)
	// 这里开启1000个协程
	for i := 0; i < 1000; i++ {
		go func() {
			// 申请对象
			val := pool.Get()
			// 使用对象
			_ = val.(BigMemData)
			// 用完之后再释放对象，过大的对象，可以不放回池中
			pool.Put(val)
			wait.Done()
		}()
	}
	wait.Wait()
	fmt.Println(numOfObject.Load())
}
```

###### Map

sync.Map 是官方提供的一种并发安全 Map 的实现，开箱即用，使用起来十分的简单，下面是该结构体对外暴露的方法：

> 为了并发安全肯定需要做出一定的牺牲，sync.Map 的性能要比 map 低 10-100 倍左右。

```go
// 根据一个key读取值，返回值会返回对应的值和该值是否存在
func (m *Map) Load(key any) (value any, ok bool)

// 存储一个键值对
func (m *Map) Store(key, value any)

// 删除一个键值对
func (m *Map) Delete(key any)

// 如果该key已存在，就返回原有的值，否则将新的值存入并返回，当成功读取到值时，loaded为true，否则为false
func (m *Map) LoadOrStore(key, value any) (actual any, loaded bool)

// 删除一个键值对，并返回其原有的值，loaded的值取决于key是否存在
func (m *Map) LoadAndDelete(key any) (value any, loaded bool)

// 遍历Map，当f()返回false时，就会停止遍历
func (m *Map) Range(f func(key, value any) bool)
```

示例：
```go
func main() {
	var syncMap sync.Map
	// 存入数据
	syncMap.Store("a", 1)
	syncMap.Store("a", "a")
	// 读取数据
	fmt.Println(syncMap.Load("a"))
	// 读取并删除
	fmt.Println(syncMap.LoadAndDelete("a"))
	// 读取或存入
	fmt.Println(syncMap.LoadOrStore("a", "hello world"))
	syncMap.Store("b", "goodbye world")
	// 遍历map
	syncMap.Range(func(key, value any) bool {
		fmt.Println(key, value)
		return true
	})
}
```

并发存入示例：
```go
func main() {
	var syncMap sync.Map
	var wait sync.WaitGroup
	wait.Add(10)
	for i := range 10 {
		go func(n int) {
			for range 100 {
                // 10个协程不停存入，如果不是 sync.Map，很可能会触发 fatal
				syncMap.Store(n, n)
			}
			wait.Done()
		}(i)
	}
	wait.Wait()
	syncMap.Range(func(key, value any) bool {
		fmt.Println(key, value)
		return true
	})
}
```

##### 原子

###### 原子类型与使用

在计算机学科中，原子或原语操作，通常用于表述一些不可再细化分割的操作，由于这些操作无法再细化为更小的步骤，在执行完毕前，不会被其他的任何协程打断，所以执行结果要么成功要么失败，没有第三种情况可言，如果出现了其他情况，那么它就是不是原子操作。

> 真正的原子操作是由硬件指令层面支持的。

Go 标准库 sync/atomic 包下已经提供了原子操作相关的 API，其提供了以下几种类型以供进行原子操作：

其中 Pointer 原子类型支持泛型，Value 类型支持存储任何类型，除此之外，还提供了许多函数来方便操作。因为原子操作的粒度过细，在大多数情况下，更适合处理这些基础的数据类型。

```go
atomic.Bool{}
atomic.Pointer[]{}
atomic.Int32{}
atomic.Int64{}
atomic.Uint32{}
atomic.Uint64{}
atomic.Uintptr{}
atomic.Value{}
```

> sync/atmoic 包下原子操作只有函数签名，没有具体实现，具体的实现是由 plan9 汇编编写。

使用示例：
```go
func main() {
	var aint64 atomic.Uint64
	// 存储值
	aint64.Store(64)
	// 交换值
	aint64.Swap(128)
	// 增加
	aint64.Add(112)
	// 加载值
	fmt.Println(aint64.Load())
}

// 也可以直接使用函数
func main() {
	var aint64 int64
	// 存储值
	atomic.StoreInt64(&aint64, 64)
	// 交换值
	atomic.SwapInt64(&aint64, 128)
	// 增加
	atomic.AddInt64(&aint64, 112)
	// 加载
	fmt.Println(atomic.LoadInt64(&aint64))
}
```

###### CAS
atomic 包还提供了 CompareAndSwap 操作，也就是 CAS。它是实现乐观锁和无锁数据结构的核心。乐观锁本身并不是锁，是一种并发条件下无锁化并发控制方式：线程/协程在修改数据前，不会先加锁，而是先读取数据，进行计算，然后在提交修改时使用CAS来判断在此期间是否有其他线程修改过该数据。如果没有（值仍等于之前读取的值），则修改成功；否则，失败并重试。因此之所以被称作乐观锁，是因为它总是乐观的假设共享数据不会被修改，仅当发现数据未被修改时才会去执行对应操作，而前面了解到的互斥量就是悲观锁，互斥量总是悲观的认为共享数据肯定会被修改，所以在操作时会加锁，操作完毕后就会解锁。由于无锁化实现的并发，其安全性和效率相对于锁要高一些，许多并发安全的数据结构都采用了 CAS 来进行实现，不过真正的效率要结合具体使用场景来看。看下面的一个例子：

```go
var lock sync.Mutex

var count int

func Add(num int) {
    lock.Lock()
    count += num
    lock.Unlock()
}

// 接下来使用 CAS 改造一下
var count int64
func Add(num int64) {
	for {
        // 使用 LoadInt64 获取期望值
		expect := atomic.LoadInt64(&count)
        // 它需要三个参数：内存值、期望值、新值，返回 bool
		if atomic.CompareAndSwapInt64(&count, expect, expect+num) {
			break
		}
	}
}
```

###### Value

atomic.Value 结构体，可以存储除了 nil 之外任意类型的值，结构体如下：
```go
type Value struct {
   // any 类型
   v any
}
```

> `atomic.Value` 一旦存储值，那么它的类型就固定了，前后存储的值类型应当一致

#### 模块

```go
// go get 命令安装第三方依赖
go get github.com/gin-gonic/gin@latest
go get github.com/gin-gonic/gin@none

// go install 命令会将第三方依赖下载到本地并编译成二进制文件（然后 go 会将其存放在 $GOPATH/bin 或者 $GOBIN 目录下）
go install github.com/go-delve/delve/cmd/dlv@latest

// 下载当前项目的依赖包
go mod download

// 编辑 go.mod 文件
go mod edit

// 输出模块依赖图
go mod graph

// 在当前目录初始化 go mod
go mod init

// 清理项目模块
go mod tidy

// 验证项目的依赖合法性
go mod verify

// 解释项目哪些地方用到了依赖
go mod why

// 用于删除项目模块依赖缓存
go clean -modcache

// 列出模块
go list -m
```