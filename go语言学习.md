## Go 语言学习

#### 1、未初始化变量，都有默认值

- ###### 数值类型

  | 类型   |               默认值               |
  | ------ | :--------------------------------: |
  | 布尔   |               false                |
  | String |                 ""                 |
  | int    |                 0                  |
  | 数组   | 每个元素或字段都是对应该类型的零值 |
  | 结构体 | 每个元素或字段都是对应该类型的零值 |

- ###### 接口或引用类型

  | 类型  | 默认值 |
  | ----- | ------ |
  | slice | nil    |
  | 指针  | nil    |
  | map   | nil    |
  | chan  | nil    |
  | 函数  | nil    |
  |       |        |

#### 2、变量声明

**Note：**包级变量的声明顺序随意，但是函数内部定义的变量必须先定义后使用

```go
///批量声明多个变量
var i, j, k int                 // int, int, int
var b, f, s = true, 2.3, "four" // bool, float64, string
```

局部变量可以使用简短声明

```go
///类型根据初始值自动推断
anim := gif.GIF{LoopCount: nframes}
freq := rand.Float64() * 3.0
t := 0.0
///也可以批量定义多个变量，如果其中某些在上面已经定义了，那相当于赋值；但必须至少要声明一个新的变量
i, j := 0, 1

///注意⚠️简短变量声明语句只有对已经在同级词法域声明过的变量才和赋值操作语句等价，如果变量是在外部词法域声明的，那么简短变量声明语句将会在当前词法域重新声明一个新的变量。
```

```go
///比如下面就会编译错误
f, err := os.Open(infile)
// ...
f, err := os.Create(outfile) // compile error: no new variables
```

**⚠注意：**不要和多重赋值搞混了，例如：

```go
i, j = j, i // 交换 i 和 j 的值,这是赋值操作，不是定义变量
```

#### 3、变量的生命周期

| 变量生命周期     | Go                             | C            | C++            | Objective-C  | Swift        |
| ---------------- | ------------------------------ | ------------ | -------------- | ------------ | ------------ |
| 全局变量         | 这是包一级声明的变量，一直存在 | 一直存在     | 一直存在       | 一直存在     | 一直存在     |
| 函数内部局部变量 | 自动垃圾收集器跟踪是否可达？   | 退出作用域后 | 退出作用域后   | 退出作用域后 | 退出作用域后 |
| new 创建的变量   | 自动垃圾收集器跟踪是否可达？   | 手动调用free | 手动调用delete | 引用计数为0  | NA           |

由此我们得出go语言以下结论：

|                | 返回类型 | 释放时机                     |
| -------------- | -------- | ---------------------------- |
| var 定义的变量 | 引用     | 自动垃圾收集器跟踪是否可达？ |
| new 创建的对象 | 指针     | 自动垃圾收集器跟踪是否可达？ |
| make函数创建的 | 引用     | 自动垃圾收集器跟踪是否可达？ |

***使用var定义的变量，和使用new定义的变量没多大区别。变量是否回收，跟它是局部变量无关（即便函数返回；new定义的变量也不一定在堆上。***

```go
var global *int

func f() {
    var x int ///你以为上在栈上，其实在堆上
    x = 1
    global = &x
}
func f1() *int {
    v := 1 ///看起来是局部变量，其实在堆上分配的，所以函数返回其地址是OK
    return &v ///返回其地址
}
func g() {
    y := new(int) ///你以为是在堆上，其实是在栈上
    *y = 1
}
```

**⚠注意**：这是跟其他编程语言很大的不同，go语言的垃圾回收机制为我们做了这些，我们几乎不用关心内存管理

#### 4、++、-- 操作是语句，不是表达式

这也是跟C、C++的不同之处。C中i++和++i，都是表达式，且二者稍有不同，初学者容易晕。

```go
v := 1
v++    // 等价方式 v = v + 1；v 变成 2
v--    // 等价方式 v = v - 1；v 变成 1
v = v++ ///错误，v++是语句，不是表达式
```

#### 5、多重赋值

这也是大部分语言不具备的语法，看起来有一定迷惑性。

```go
x, y = y, x  ///交换x、y的值
///注意，编译器会先把右侧所有表达式一次性算出来，然后在批量赋值给左侧变量
///而不是逐个赋值,试想一下如下代码，跟上面的结果完全不一致
/// x = y
/// y = x
a[i], a[j] = a[j], a[i]
```

#### 6、每个go文件，都可以有一个init方法，程序加载时自动调用且只调用一次

这样的init初始化函数除了不能被调用或引用外，其他行为和普通函数类似。在每个文件中的init初始化函数，在程序开始执行时按照它们声明的顺序被自动调用，根据依赖情况，先初始化依赖的包,main包最后被初始化。

#### 7、访问权限

首字母大写的方法、类型、变量可以被其他包引用。it's easy as abc!

#### 8、defer函数

- 多个defer语句，按先进后出的方式执行

- 即使函数返回前发生panic，也能执行defer语句

- 多个 defer 注册，按 FILO 次序执行 ( 先进后出 )。哪怕函数或某个延迟调用发生错误，这些调用依旧会被执行。

  ```go
  package main
  
  func test(x int) {
      defer println("a")
      defer println("b")
  
      defer func() {
          println(100 / x) // div0 异常未被捕获，逐步往外传递，最终终止进程。
      }()
  
      defer println("c")
  }
  
  func main() {
      test(0)
  }
  
  输出结果：
   c
   b
   a
   panic: runtime error: integer divide by zero
  ```

  

- defer 函数调用时的参数，在defer声明时就已经复制了

- recover函数只能在defer中发挥作用

#### 9、异常捕获

#### 10、参数传递——pass by Value 传值

#### 11、Interface接口

#### 12、channel通道

channel是goroutine之间通信的机制，可以简单理解成一个队列，各个gorotinue之间可以安全访问（读取、写入）。往已满的channel中写入，会触发堵塞，同理，从空的channel中读取值也会堵塞。

```go
ch := make(chan int) // ch has type 'chan int'

ch <- x  // a send statement
x = <-ch // a receive expression in an assignment statement
<-ch     // a receive statement; result is discarded
```

| 通道Channel状态 | 发送                                         | 接收                                                 |
| :-------------- | :------------------------------------------- | :--------------------------------------------------- |
| 无缓存通道      | 堵塞直到被读取                               | 堵塞直到有人发送                                     |
| 有缓存通道      | 缓存未满：不堵塞<br />缓存满：堵塞直到被读取 | 缓存不为空：不堵塞<br />缓存为空：堵塞直到有新的发送 |
| 值为nil的通道   | 永远堵塞                                     | 永远堵塞                                             |
| 已关闭的通道    | panic                                        | 返回对应零值                                         |



#### 13、select多路复用

问题：对于一个channel通道，对其进行读取、或者写入，可能导致当前goroutine堵塞等待，如果需要同时对多个channel进行操作就没办法了？因为程序会按照代码顺序，依次操作多个通道，一旦其中一个channel触发堵塞，后续channel都无法操作。

- 普通channel使用时，触发堵塞

  ```go
  abort := make(chan struct{})
  
  abort <- struct{}{} ///write to channel，waiting...
  ```

- 不触发堵塞

  ```go
  select {
  case <-abort:
      fmt.Printf("Launch aborted!\n")
      return
  default:///上面的都没有，则运行此次，不触发堵塞
      // do nothing
  }
  ```

- 同时监听多个channel

  ```go
  select {
  case <-ch1:
      // ...
  case x := <-ch2:
      // ...use x...
  case ch3 <- y:
      // ...
  default://有default时不会堵塞，没有则会堵塞
      // ...
  }
  ```

  - 对一个nil的channel发送和接收操作会永远阻塞，所以在select语句中操作nil的channel永远都不会被select到。这使得我们可以用nil来激活或者禁用case，来达成处理其它输入或输出事件时超时和取消的逻辑。

#### 15、数组和Slice

- 数组

  数组定义：值类型，一个数组赋值给另一个数组，二者修改其一，互不影响。

  ```go
  ///定义数组时，数组的大小必须指定
  var q [3]int = [3]int{1, 2, 3}
  var r [3]int = [3]int{1, 2}
  fmt.Println(r[2]) // "0"
  
  ///用...表示后面初始化列表有几个，大小就是多少
  q1 := [...]int{1, 2, 3}
  fmt.Printf("%T\n", q1) // "[3]int"
  
  ///初始化列表中可以指定索引初始化，顺序无所谓，没有指定的索引值为默认值
  symbol := [...]string{0: "$", 1: "€", 2: "￡", 3: "￥"}
  fmt.Println(symbol[3]) // "￥"
  
  ///第99个值为-1，前面0-98个值默认就为0；总大小100
  r := [...]int{99: -1}
  ```

  数组比较：如果一个数组的元素类型是可以相互比较的，那么数组类型也是可以相互比较的，这时候我们可以直接通过==比较运算符来比较两个数组，只有当两个数组的所有元素都是相等的时候数组才是相等的。不相等比较运算符!=遵循同样的规则。

  ```go
  a := [2]int{1, 2}
  b := [...]int{1, 2}
  c := [2]int{1, 3}
  fmt.Println(a == b, a == c, b == c) // "true false false"
  d := [3]int{1, 2}
  fmt.Println(a == d) ///编译错误， [2]int 和 [3]int 是不同的类型，无法比较
  ```

- Slice切片

  定义：定义方式和数组非常类似，区别就是不指定数组大小

  ```go
  ///其本质类似于下面的结构体，描述了一个完整数组的一小片区域（slice）
  type IntSlice struct {
      ptr      *int ///第一个元素的地址
      len int   ///总共多少个有效的元素
      cap int   ///总共多大的空间
  }  
  
  array := [...]string{"a","b","c","d","e","f","g","h"} ///定义一个数组，总共8个元素
             ///索引：   0   1   2   3   4   5    6  7
  s1 := array[2:6]///定义slice   |-----------|
  ////s1描述了array数组中2...<6的位置，也就是2、3、4、5,因此通过s1操作数组其实就是直接操作原数组
  
  ///定义一个slice，这会隐式地创建一个合适大小的数组，然后slice的指针指向底层的数组
  s2 := []int{0, 1, 2, 3, 4, 5} ///slice和数组的字面值语法很类似，区别是没有指定大小
  
  ///切片操作 
  ///array[:] 代表array[0:len(array)] 也就是整个数组
  ///array[:i] 代表数组从 第一个 -> i-1
  ///array[i:] 代表数组从 i -> 最后一个
  ```

  Slice之间不能用==直接做比较

  ```go
  var s []int    // len(s) == 0, s == nil
  s = nil        // len(s) == 0, s == nil
  s = []int(nil) // len(s) == 0, s == nil
  s = []int{}    // len(s) == 0, s != nil
  
  ```

  Slice的append操作,参考如下代码：

  ```go
  func appendInt(x []int, y int) []int {
      var z []int
      zlen := len(x) + 1
      if zlen <= cap(x) {
          // There is room to grow.  Extend the slice.
          z = x[:zlen]
      } else {
          // There is insufficient space.  Allocate a new array.
          // Grow by doubling, for amortized linear complexity.
          zcap := zlen
          if zcap < 2*len(x) {
              zcap = 2 * len(x)
          }
          z = make([]int, zlen, zcap)
          copy(z, x) // a built-in function; see text
      }
      z[len(x)] = y
      return z
  }
  ```

  

#### 16、反射

#### 17、锁

- 互斥锁【缓存为1的chan通道】

  由于channel的特点，当该通道的缓存为1的时候，其行为就已经是一个互斥锁了。

- 互斥锁【sync.Mutex】

  与iOS中的互斥锁大致一样，特点：

  1、尝试对已经被其他goroutine锁住的锁进行加锁，会使当前goroutine进入休眠，直到锁被释放

  2、任何goroutine尝试加锁多次会死锁，不像递归锁，go语言中没有提供递归锁

- 读写锁【sync.RWMutex】

  与iOS中的互斥锁大致一样，特点：

  1、支持多读单写。读操作可以安全并发，如果有写操作则需要按照顺序进行