# GO编程模式：PIPELINE

[2020年12月26日 ](https://coolshell.cn/articles/21228.html) [陈皓](https://coolshell.cn/articles/author/haoel) 

本篇文章，我们着重介绍Go编程中的Pipeline模式。对于Pipeline用过Unix/Linux命令行的人都不会陌生，他是一种把各种命令拼接起来完成一个更强功能的技术方法。在今天，流式处理，函数式编程，以及应用网关对微服务进行简单的API编排，其实都是受pipeline这种技术方式的影响，Pipeline这种技术在可以很容易的把代码按单一职责的原则拆分成多个高内聚低耦合的小模块，然后可以很方便地拼装起来去完成比较复杂的功能。

### 本文是全系列中第8 / 10篇：[Go编程模式](https://coolshell.cn/articles/series/go编程模式)

- [Go编程模式：切片，接口，时间和性能](https://coolshell.cn/articles/21128.html)
- [Go 编程模式：错误处理](https://coolshell.cn/articles/21140.html)
- [Go 编程模式：Functional Options](https://coolshell.cn/articles/21146.html)
- [Go编程模式：委托和反转控制](https://coolshell.cn/articles/21214.html)
- [Go编程模式：Map-Reduce](https://coolshell.cn/articles/21164.html)
- [Go 编程模式：Go Generation](https://coolshell.cn/articles/21179.html)
- [Go编程模式：修饰器](https://coolshell.cn/articles/17929.html)
- Go编程模式：Pipeline
- [Go 编程模式：k8s Visitor 模式](https://coolshell.cn/articles/21263.html)
- [Go编程模式 ： 泛型编程](https://coolshell.cn/articles/21615.html)

« [上一篇文章](https://coolshell.cn/articles/17929.html)[下一篇文章](https://coolshell.cn/articles/21263.html) »

目录



[HTTP 处理](https://coolshell.cn/articles/21228.html#HTTP_处理)[Channel 管理](https://coolshell.cn/articles/21228.html#Channel_管理)[Channel转发函数](https://coolshell.cn/articles/21228.html#Channel转发函数)[平方函数](https://coolshell.cn/articles/21228.html#平方函数)[过滤奇数函数](https://coolshell.cn/articles/21228.html#过滤奇数函数)[求和函数](https://coolshell.cn/articles/21228.html#求和函数)[Fan in/Out](https://coolshell.cn/articles/21228.html#Fan_inOut)[延伸阅读](https://coolshell.cn/articles/21228.html#延伸阅读)

#### HTTP 处理

这种Pipeline的模式，我们在《[Go编程模式：修饰器](https://coolshell.cn/articles/17929.html)》中有过一个示例，我们在这里再重温一下。在那篇文章中，我们有一堆如 `WithServerHead()` 、`WithBasicAuth()` 、`WithDebugLog()`这样的小功能代码，在我们需要实现某个HTTP API 的时候，我们就可以很容易的组织起来。

原来的代码是下面这个样子：

```go
http.HandleFunc("/v1/hello", WithServerHeader(WithAuthCookie(hello)))
http.HandleFunc("/v2/hello", WithServerHeader(WithBasicAuth(hello)))
http.HandleFunc("/v3/hello", WithServerHeader(WithBasicAuth(WithDebugLog(hello))))
```

通过一个代理函数：

```go
type HttpHandlerDecorator func(http.HandlerFunc) http.HandlerFunc
func Handler(h http.HandlerFunc, decors ...HttpHandlerDecorator) http.HandlerFunc {
    for i := range decors {
        d := decors[len(decors)-1-i] // iterate in reverse
        h = d(h)
    }
    return h
}
```

我们就可以移除不断的嵌套像下面这样使用了：

```go
http.HandleFunc("/v4/hello", Handler(hello,
                WithServerHeader, WithBasicAuth, WithDebugLog))
```

#### Channel 管理

当然，如果你要写出一个[泛型的pipeline框架](https://coolshell.cn/articles/17929.html#泛型的修饰器)并不容易，而使用[Go Generation](https://coolshell.cn/articles/21179.html)，但是，我们别忘了Go语言最具特色的 Go Routine 和 Channel 这两个神器完全也可以被我们用来构造这种编程。

Rob Pike在 [Go Concurrency Patterns: Pipelines and cancellation](https://blog.golang.org/pipelines) 这篇blog中介绍了如下的一种编程模式。

##### Channel转发函数

首先，我们需一个 `echo()`函数，其会把一个整型数组放到一个Channel中，并返回这个Channel

```go
func echo(nums []int) <-chan int {
  out := make(chan int)
  go func() {
    for _, n := range nums {
      out <- n
    }
    close(out)
  }()
  return out
}
```

然后，我们依照这个模式，我们可以写下这个函数。

##### 平方函数

```go
func sq(in <-chan int) <-chan int {
  out := make(chan int)
  go func() {
    for n := range in {
      out <- n * n
    }
    close(out)
  }()
  return out
}
```

##### 过滤奇数函数

```go
func odd(in <-chan int) <-chan int {
  out := make(chan int)
  go func() {
    for n := range in {
      if n%2 != 0 {
        out <- n
      }
    }
    close(out)
  }()
  return out
}
```

##### 求和函数

```go
func sum(in <-chan int) <-chan int {
  out := make(chan int)
  go func() {
    var sum = 0
    for n := range in {
      sum += n
    }
    out <- sum
    close(out)
  }()
  return out
}
```

然后，我们的用户端的代码如下所示：（注：**你可能会觉得，`sum()`，`odd()` 和 `sq()`太过于相似。你其实可以通过我们之前的[Map/Reduce编程模式](https://coolshell.cn/articles/21164.html)或是[Go Generation的方式](https://coolshell.cn/articles/21179.html)来合并一下**）

```go
var nums = []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
for n := range sum(sq(odd(echo(nums)))) {
  fmt.Println(n)
}
```

上面的代码类似于我们执行了Unix/Linux命令： `echo $nums | sq | sum`

同样，如果你不想有那么多的函数嵌套，你可以使用一个代理函数来完成。

```go
type EchoFunc func ([]int) (<- chan int) 
type PipeFunc func (<- chan int) (<- chan int) 

func pipeline(nums []int, echo EchoFunc, pipeFns ... PipeFunc) <- chan int {
  ch  := echo(nums)
  for i := range pipeFns {
    ch = pipeFns[i](ch)
  }
  return ch
}
```

然后，就可以这样做了：

```go
var nums = []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}    
for n := range pipeline(nums, gen, odd, sq, sum) {
    fmt.Println(n)
  }
```

#### Fan in/Out

动用Go语言的 Go Routine和 Channel还有一个好处，就是可以写出1对多，或多对1的pipeline，也就是Fan In/ Fan Out。下面，我们来看一个Fan in的示例：

我们想通过并发的方式来对一个很长的数组中的质数进行求和运算，我们想先把数组分段求和，然后再把其集中起来。

下面是我们的主函数：

```go
func makeRange(min, max int) []int {
  a := make([]int, max-min+1)
  for i := range a {
    a[i] = min + i
  }
  return a
}

func main() {
  nums := makeRange(1, 10000)
  in := echo(nums)

  const nProcess = 5
  var chans [nProcess]<-chan int
  for i := range chans {
    chans[i] = sum(prime(in))
  }

  for n := range sum(merge(chans[:])) {
    fmt.Println(n)
  }
}
```

再看我们的 `prime()` 函数的实现 ：

```go
func is_prime(value int) bool {
  for i := 2; i <= int(math.Floor(float64(value) / 2)); i++ {
    if value%i == 0 {
      return false
    }
  }
  return value > 1
}

func prime(in <-chan int) <-chan int {
  out := make(chan int)
  go func ()  {
    for n := range in {
      if is_prime(n) {
        out <- n
      }
    }
    close(out)
  }()
  return out
}
```

到，

- 我们先制造了从1到10000的一个数组，
- 然后，把这堆数组全部 `echo`到一个channel里 – `in`
- 此时，生成 5 个 Channel，然后都调用 `sum(prime(in))` ，于是每个Sum的Go Routine都会开始计算和
- 最后再把所有的结果再求和拼起来，得到最终的结果。

其中的merge代码如下：

```go
func merge(cs []<-chan int) <-chan int {
  var wg sync.WaitGroup
  out := make(chan int)

  wg.Add(len(cs))
  for _, c := range cs {
    go func(c <-chan int) {
      for n := range c {
        out <- n
      }
      wg.Done()
    }(c)
  }
  go func() {
    wg.Wait()
    close(out)
  }()
  return out
}
```

用图片表示一下，整个程序的结构如下所示：

![img](./img/114-Go编程模式：Pipeline.assets/pipeline-1024x425.png)

#### 延伸阅读

如果你还想了解更多的这样的与并发相关的技术，可以参看下面这些资源：

- **Go** **Concurrency** **Patterns** – **Rob** **Pike –** 2012 Google I/O **
  **presents the basics of Go‘s concurrency primitives and several ways to apply them.
  https://www.youtube.com/watch?v=f6kdp27TYZs
- **Advanced Go Concurrency Patterns** – **Rob** **Pike** – 2013 Google I/O **
  **covers more complex uses of Go’s primitives, especially select.
  https://blog.golang.org/advanced-go-concurrency-patterns
- **Squinting at Power Series** – **Douglas McIlroy**‘s paper **
  **shows how Go-like concurrency provides elegant support for complex calculations.
  https://swtch.com/~rsc/thread/squint.pdf

（全文完）