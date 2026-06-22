# Golang 基础语法学习笔记

本文是系列文章《从 PHP 到 AI + Golang，程序员自救转型手记》作者的学习笔记，完整版开源于：[github](https://github.com/ai-go-hub/ai-go-blog) | [gitee](https://gitee.com/ai-go-hub/ai-go-blog)，实操项目 ai-go-mall 开源于：[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)

#### 常量
```go
// 类型可省略，必须指定值
const name string = "Jack"

const (
   Count = 1
   Name  = "Jack"
)

// 枚举
const (
   Num = iota*2 // 0
   Num1 // 2
   Num2 // 4
   Num3 // 6
   Num4TEST // 8
)
```

#### 变量
```go
var name string = "jack"

name2 := "jack" // 自动推断为字符串类型的变量

name, age := "jack", 1
```

#### 条件控制/循环控制
```go
// 比起 php，表达式不要括号
if a > 1 {}

// 基础循环
for i := 0; i < 10; i++ {
    fmt.Println(i)
}

str := "hello"
for index, value := range str {
    fmt.Println(index, value)
}
```

#### 数组
```go
// 基本
var a [5]int

// 无值的下标将使用对应类型的 0 值填充
nums := [5]int{1, 2, 3}

// 自动推断长度
nums := [...]int{1, 2, 3, 4, 5}

// 多维
var nums [5][5]int

// 长度
len(nums)

// 切割 arr[startIndex:endIndex]
nums[:] // 子切片范围[0,5) -> [1 2 3 4 5]
nums[1:] // 子切片范围[1,5) -> [2 3 4 5]
nums[:5] // 子切片范围[0,5) -> [1 2 3 4 5]
nums[2:3] // 子切片范围[2,3) -> [3]
nums[1:3] // 子切片范围[1,3) -> [2 3]

// 数组转切片，并且不影响原数组
arr := [5]int{1, 2, 3, 4, 5}
slice := slices.Clone(arr[:])

// 创建切片
dest := make([]int, 0)
src := []int{1, 2, 3, 4, 5, 6, 7, 8, 9}

// 切片容量
cap(slice)

// 切片拓展表达式
slice[low:high:max]
s1 := []int{1, 2, 3, 4, 5, 6, 7, 8, 9} // cap = 9
s2 := s1[3:4]                          // cap = 9 - 3 = 6
s2 = append(s2, 1)                     // 添加新元素，由于容量为6.所以没有扩容，直接修改底层数组

s2 := s1[3:4:4]                        // cap = 4 - 3 = 1
s2 = append(s2, 1)                     // 容量不足，分配新的底层数组
```

#### 字符串
在 Go 中，字符串本质上是一个不可变的、只读的字节序列，字符串分为普通字符串和原生字符串。

```go
// 切割字符串
str := "this is a string"
fmt.Println(string(str[0:4]))

// 访问字符串
fmt.Println(str[0])

// 字符串与切片相互转换
bytes := []byte(str)        // 显式类型转换为字节切片
fmt.Println(bytes)
fmt.Println(string(bytes))  // 显式类型转换为字符串

// 使用 + 号拼接字符串
fmt.Println("a" + "b")

// 性能最高的字符串拼接方式
builder := strings.Builder{}
builder.WriteString("this is a string ")
builder.WriteString("that is a int")
fmt.Println(builder.String())

// 使用 for range 遍历字符串时，其默认的遍历单位类型是一个 rune（而不是一个字节，即中文友好）
str := "hello 世界!"
for _, r := range str {
    fmt.Printf("%d,%x,%s\n", r, r, string(r))
}
```

#### map
golang 中的 map 是无序存储的。

```go
mp := map[int]string{
   0: "a",
   1: "a",
   2: "a",
   3: "a",
   4: "a",
}

mp := map[string]int{
   "a": 0,
   "b": 22,
   "c": 33,
}

mp := make(map[string]int, 8)
mp := make(map[string][]int, 10)

// 访问基本和数组相同
// val 是值，exist 是健是否存在的布尔值，exist=false 时，val 是对应类型的零值
val, exist := mp["b"]

// 求长度
fmt.Println(len(mp))

// 删除和清空
delete(mp, "b")
clear(mp)

// 遍历
for key, val := range mp {
    fmt.Println(key, val)
}

// set
// map 的键正是无序且不能重复的，所以可以使用 map 来替代 set。
set := make(map[int]struct{}, 10)
for i := 0; i < 10; i++ {
    set[rand.Intn(100)] = struct{}{}
}
fmt.Println(set)
```

#### 指针
golang 中，保留了指针，在一定程度上保证了性能，同时为了更好的 GC 和安全考虑，又限制了指针的使用。

取地址符 `&`，获取变量的内存地址，返回一个指向该变量的指针：`ptr := &x`
解引用符 `*`，访问一个指针指向的底层值：`val := *ptr`
指针类型`*T`，指向 `T` 类型的指针：`var ptr *int`

```go
// 取地址符 `&`，解引用符 `*`
num := 2
p := &num
rawNum := *p
fmt.Println(rawNum)

// new 函数只有一个参数那就是类型，并返回一个对应类型的指针，函数会为该指针分配内存，并且指针指向对应类型的零值，例如：
fmt.Println(*new(string))
fmt.Println(*new(int))
fmt.Println(*new([5]int))
fmt.Println(*new([]float64))
```

#### 函数
```go
func 函数名([参数列表]) [返回值] {
  函数体
}

func sum(a int, b int) int {
  return a + b
}

var sum = func(a int, b int) int {
  return a + b
}

// 对于类型相同的参数而言，可以只需要声明一次类型，不过条件是它们必须相邻
func sum(a, b int) int {
  return a + b
}

// 变长参数可以接收 0 个或多个值，必须声明在参数列表的末尾
// Go 允许函数有多个返回值，此时就需要用括号将返回值围起来。
func Printf(format string, a ...any) (n int, err error) {
  return Fprintf(os.Stdout, format, a...)
}

// Go 也支持具名返回值，不能与参数名重复，使用具名返回值时，return关键字可以不需要指定返回哪些值。
// 如果 return 后面写了值，那么它的优先级最高
func Sum(a, b int) (ans int) {
  ans = a + b
  return
}

// 匿名函数，直接跟上小括号以调用
func(a, b int) int {
  return a + b
}(1, 2)

// 将函数作为参数传递时，不需要跟上小括号，因为此时不调用
test(func(){})

// 闭包，变量 e 并不会随着 Exp 函数调用结束而出栈，而是逃逸到了堆上
grow := Exp(2)
for i := range 10 {
  fmt.Printf("2^%d=%d\n", i, grow())
}

func Exp(n int) func() int {
  e := 1
  return func() int {
    temp := e
    e *= n
    return temp
  }
}

// 延迟调用函数，被调用函数的参数如果也是函数调用，那么参数值会被预计算而不是等待延迟结束一起计算
defer fmt.Println(4)
```

#### 结构体

基础

```go
type Person struct {
   name string
   age int
}

// 类型相同的字段，可以写在一行
type Rectangle struct {
  height, width, area int
  color               string
}

// 实例化
programmer := Person{
   Name:     "jack",
   Age:      19,
}

// 可以自己写一个专门用来实例化某个结构体的函数
func NewPerson(name string, age int) *Person {
  return &Person{Name: name, Age: age}
}
```

选项模式，可以更为灵活的实例化结构体

```go
type PersonOptions func(p *Person)

func WithName(name string) PersonOptions {
  return func(p *Person) {
    p.Name = name
  }
}

func WithAge(age int) PersonOptions {
  return func(p *Person) {
    p.Age = age
  }
}

func NewPerson(options ...PersonOptions) *Person {
  // 优先应用options
  p := &Person{}

  for _, option := range options {
    option(p)
  }

  // 字段默认值处理
  if p.Age < 0 {
    p.Age = 0
  }
  return p
}

// 实例化
pl := NewPerson(
  WithName("John Doe"),
  WithAge(25),
)

p2 := NewPerson(
  WithName("Mike jane"),
  WithAge(30),
)
```

结构体的组合

```go
// 显式组合
type Person struct {
   name string
   age  int
}

type Student struct {
   p      Person
   school string
}

type Employee struct {
   p   Person
   job string
}

student := Student{
   p:      Person{name: "jack", age: 18},
   school: "lili school",
}
fmt.Println(student.p.name)

// 隐式组合
type Person struct {
  name string
  age  int
}

type Student struct {
  Person
  school string
}

type Employee struct {
  Person
  job string
}

student := Student{
   Person: Person{name: "jack",age: 18},
   school: "lili school",
}
fmt.Println(student.name)
```

#### 方法
方法与函数的区别在于，方法拥有接收者，而函数没有，且只有自定义类型能够拥有方法。

```go
type IntSlice []int

// (i IntSlice) 就是声明接受者，普通函数没有这一段
func (i IntSlice) Get(index int) int {
  return i[index]
}
func (i IntSlice) Set(index, val int) {
  i[index] = val
}

func (i IntSlice) Len() int {
  return len(i)
}

func main() {
  var intSlice IntSlice
  intSlice = []int{1, 2, 3, 4, 5}
  fmt.Println(intSlice.Get(0))
  intSlice.Set(0, 2)
  fmt.Println(intSlice)
  fmt.Println(intSlice.Len())
}
```

#### 可见性
名称大写开头就是公有类型/变量/常量，小写开头就是私有的

#### 注意点
- 没有三元运算符
- 固定的代码格式化工具
- 在需要花括号时，单行代码也要花括号 `for i := 0; i < 10; i++ {fmt.Println(i)}`
- 导包时使用 _ 作为别名代表匿名导入
- 导包时使用 . 作为别名代表将包所有类型导入到当前作用域，这种方式导入后不需要通过 . 运算符访问
- 有非常精细的类型系统，类型声明后置以便重写类型声明
- `rune` 本质是 `int32` 的别名

- `make(类型, 长度, 容量)` 返回特定类型的值（而不是指针），专用于给切片，映射表，通道分配内存
- `new(类型)` 返回特定类型的指针，专用于分配好内存，并赋类型的零值

- receiver 接受器
- comparable 可比较的

#### 思考
- 结构体对标类
- 指针用来做大数据的便捷传递（只需传递一个指针地址）等