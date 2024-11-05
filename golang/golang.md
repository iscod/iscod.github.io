
## 值类型与引用类型

 哪些是引用类型？

`map`、`slice`、`interface`、`chan`、`pointer`、`func`

哪些是值类型？

`struct`、`array`、`int`、`float`、`bool`、`string`

当在函数中以值类型传递参数时，函数会创建一个原始变量的副本，并在函数内部使用该副本进行操作。因此，对该副本的修改不会影响到原始变量。
引用类型传递参数时，函数会传递变量的地址（指针），函数内部使用该地址来操作原始变量的值。因此，在函数内部对该地址指向的值进行修改会影响到原始变量。

> 引用类型的变量可以初始化为 `nil`, 值类型初始化为默认值。引用类型结构的数据都是通过`unsafe.Pointer`指针来指向底层数据，因此可以赋值为`nil`

```go
func change(slice []int) {
	for k, v := range slice {
		slice[k] = v + 10
	}
}

func main() {
	var slice = make([]int, 0)
	for i := 0; i < 10; i++ {
		slice = append(slice, i)
	}
	slice1 := slice
	change(slice)
	fmt.Println("slice:", slice)
	fmt.Println("slice1:", slice1)
}
```

输出结果：

```sh
# go run main.go
slice: [10 11 12 13 14 15 16 17 18 19]
slice1: [10 11 12 13 14 15 16 17 18 19]
```

可以看到引用类型，两个变量都发生了修改

> Go函数参数中都是值传递，只是引用类型形参和实参指向的内存地址相同。如果想获取一个互不影响的值，可以使用深`copy`


## panic 与 defer 谁先执行？

先说结论：`panic`相当于一个`return`。所以在函数`panic`前执行`defer`

```go
func main() {
	defer fmt.Println("defer 1")
	defer fmt.Println("defer 2")
	panic("err panic")
}
```

输出结果：

```
 # go run main.go
defer 2
defer 1
panic: err panic

goroutine 1 [running]:
main.main()
        /Users/ning/Data/xz_server/src/ningserver/main.go:126 +0xc5
exit status 2
```

## goroutine 需要错误处理吗？

在一个程序中存在多个`goroutine`。如果不进行错误处理，当一个`goroutine`发生`panic`会造成整个程序崩溃。所以为了保证程序还能正常运行，一般在子`goroutine`可以使用`recover`捕获异常，进而对异常处理

```go
func main() {
	wg := sync.WaitGroup{}
	wg.Add(1)
	go func() {
		defer func() {
			wg.Done()
			if err := recover(); err != nil {
				fmt.Println(err)
			}
		}()
		panic("err panic")
	}()
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < 10; i++ {
			time.Sleep(time.Second)
			fmt.Printf("%d,", i)
		}
	}()
	wg.Wait()
}

```

输出结果：

```
# go run main.go
err panic
0,1,2,3,4,5,6,7,8,9,%       
```


##  有趣的`defer`返回值

```go
func def1(i int) (t int) {
    t = i                  // t = 1
    defer func() { t += 3 }() // 闭包直接引用了有名返回值 t
    return t               // 第一步：赋值 (t = t，依然是 1)；第二步：执行 defer (t = 1 + 3)
}

func def2(i int) int {
    t := i                  // 局部变量 t = 1
    defer func() { t += 3 }() // 闭包修改的是局部变量 t
    return t                // 第一步：赋值 (匿名返回值 = t，副本值为 1)；第二步：执行 defer (t = 1 + 3)
}

func def3(i int) (t int) {
    defer func() { t += i }() // 闭包修改有名返回值 t
    return 2                  // 第一步：赋值 (t = 2)；第二步：执行 defer (t = 2 + 1)
}

func main() {
	fmt.Println(def1(1), def2(1), def3(1))
}

```

执行结果：
```sh
# go run main.go
4,1,3
```

要理解这三个结果，必须记住 Go 处理 return 的两步操作（非原子）：

1. 赋值：将返回值（或表达式的结果）写入返回值变量。
2. 执行 defer：按照后进先出的顺序执行 defer 函数。
3. 退出：函数真正携带返回值变量中的内容退出。

## iota

```go
const (
	a = iota
	b
	c = "zz"
	d
	e
	f = iota
	g
)

func main() {
	fmt.Println(a, b, c, d, e, f, g)

}

执行结果：
```sh
# go run main.go
0 1 zz zz zz 5 6
```

##  更好的 begin time

在业务中经常有需求要获取当天的凌晨时间，或者根据一个时间计算出该时间的凌晨时间点。

一般程序想到的是通过`time.ParseInLocation`结合`time.Now().Format`来使用，这种方式简单容易理解

```go
func GetTodayBegin() int64 {
	timeStr := time.Now().Format("2006-01-02")
	t, _ := time.ParseInLocation("2006-01-02 15:04:05", timeStr+" 00:00:00", time.Local)
	return t.Unix()
}
```

而好的程序可能会思考，`time`需要进行两次`string`类型转换才能得到结果，这中间可能会有巨大的性能优化空间，如果不进行类型转换，性能是否会更好？

```go
func GetTodayBeginModify() int64 {
	t := time.Now()
	_, offset := t.Local().Zone()
	return t.Unix() - int64(offset) - t.Unix()%86400
}
```


基准测试结果:
```sh
goos: darwin
goarch: amd64
BenchmarkGetDayBegin-8          15347680               390.8 ns/op
BenchmarkGetDayBeginModify-8    51300466               118.8 ns/op
```

可以看到减少两次类型的转换，就换来成倍的性能提升，如此还是值得进行一次优化


## unsafe.Pointer

`unsafe`包提供了两个重要的能力：

1. 任何类型的指针和`unsafe.Pointer`可以相互转换
2. `uintptr`类型和`unsafe.Pointer`可以相互转换

#### `unsafe.Pointer`实现更改私有成员值

```go
type User struct {
	name  string
	age   int
	score int
}

func (u *User) SetName(n string) {
	u.name = n
}

func main() {
	user := User{}
	user.SetName("iscod")
	fmt.Println(user)
	name := (*string)(unsafe.Pointer(&user))
	*name = "ascoon"
	age := (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&user)) + unsafe.Sizeof(user.name)))
	*age = 18
	score := (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&user)) + unsafe.Offsetof(user.score)))
	*score = 1
	fmt.Println(user)
}
```

执行结果：
```sh
# go run main.go
{iscod 0 0}
{ascoon 18 1}
```

> `unsafe.Sizeof`是返回字段大小，`unsafe.Offsetof`返回字段在结构内的偏移量

#### `unsafe.Pointer`实现`string`与`[]byte`无copy转换

```go
func StringToByte(s string) []byte {
	return *(*[]byte)(unsafe.Pointer(&s))
}

func ByteToString(b []byte) string {
	return *(*string)(unsafe.Pointer(&b))
}
```

## sync

`sync`包提供了三个常用的功能:

1. `sync.WaitGroup` 等待组常用于保证在并发环境中完成指定数量的任务
1. `sync.Once` 可以保证函数只执行一次的实现，比如加载配置文件场景，且是并发安全的
1. `sync.Cond` 条件变量用来协调想要访问共享资源的那些 goroutine，当共享资源的状态发生变化的时候，它可以用来通知被互斥锁阻塞的 goroutine


示例：

```go
//sync.Once
func main() {
	wg := sync.WaitGroup{}
	once := sync.Once{}
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < 10; i++ {
			fmt.Printf("%d", i)
			once.Do(func() {
				fmt.Println("once one")
			})
		}
	}()
	wg.Wait()
}
```


```go
func main() {
	//sync.Cond
	wg := sync.WaitGroup{}
	var b bool
	cond := sync.NewCond(&sync.RWMutex{})
	wg.Add(1)
	go func() {
		defer wg.Done()
		cond.L.Lock()
		for i := 0; i < 5; i++ {
			time.Sleep(time.Second)
		}
		b = true
		cond.L.Unlock()
		cond.Broadcast()
	}()
	wg.Add(1)
	go func() {
		defer wg.Done()
		fmt.Println("run wait 1")
		cond.L.Lock()
		for !b {
			cond.Wait()
		}
		fmt.Println("run 1")
		cond.L.Unlock()
	}()
	wg.Add(1)
	go func() {
		defer wg.Done()
		fmt.Println("run wait 2")
		cond.L.Lock()
		for !b {
			cond.Wait()
		}
		fmt.Println("run 2")
		cond.L.Unlock()
	}()

	wg.Wait()
}
```

## atomic & sync.Mutex

`atomic`提供了三种常用接口

1. `Add` 原子地将*delta*添加到*addr*并返回新值。
1. `CSA` 对值执行比较和交换操作。如果值与比较值相同则交换为设定的新值
1. `Store` 原子地将*val*存储到*addr*中，一般配合`atomic.Value`结构使用


`atomic`与`sync.Mutex`都可以实现并发模型下的数据安全，但是他们的应用场景偏向不同。

> `atomic`偏向于变量或者资源的更新保护。`sync.Mutex`偏向于保护一段逻辑处理。`sync.Mutex`底层也是通过`atomic`实现


```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func SumMu() {
	var count int
	wg := sync.WaitGroup{}
	mu := sync.Mutex{}
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			mu.Lock()
			count += 1
			wg.Done()
			mu.Unlock()
		}()
	}
	wg.Wait()
	fmt.Println(count)
}

func SumAtomic() {
	var count int32
	wg := sync.WaitGroup{}
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			atomic.AddInt32(&count, 1)
			wg.Done()
		}()
	}
	wg.Wait()
	fmt.Println(count)
}

func main() {
	SumMu()
	SumAtomic()
}
```


## 内存回收

Golang的垃圾回收是通过`GC`来实现的。

可以通过手动和自动方式进行垃圾回收。自动垃圾回收

```go
runtime.GC()//程序中手动调用GC
```

* `GC`采用的是`三色标记`算法来实现垃圾回收. 

`三色标记`可以理解成将一个内存标记为三种状态

1. 白色对象 — 潜在的垃圾，其内存可能会被垃圾收集器回收
1. 黑色对象 — 活跃的对象，包括不存在任何引用外部指针的对象 以及从根对象可达的对象
1. 灰色对象 — 活跃的对象，因为存在指向白色对象的外部指针，垃圾收集器会扫描这些对象的子对象

* 垃圾收回的过程

1. 初始状态所有对象都是白色
1. 从根节点开始变了所有对象, 把遍历对象标记为`灰色对象`
1. 遍历`灰色对象`, 将灰色对象引用的对象也变成`灰色对象`, 然后将遍历的`灰色对象`标记为`黑色对象`
1. 循环步骤 3, 直到`灰色对象`全部变为`黑色对象`

当三色标记工作完成后, 应用程序中就不存在任何`灰色对象`。我们只能看到`黑色对象`和`白色对象`. 垃圾回收器就可以回收这些`白色对象`垃圾.

> 内存回收有多种算法如: 引用标记, 标记清除, 分代收集等

* 混合写屏障

混合写屏障在插入写屏障和删除写屏障的基础上，混合写屏障分为四步：

1. GC开始时，将栈上的全部对象标记为黑色（不需要二次扫描，无需STW）
2. GC期间，任何栈上创建的对应均为黑色
3. 被删除的引用对象标记为灰色
4. 被添加引用的对象标记为灰色

混合写屏障的核心思想是确保黑色对象不能引用白色对象。


* 参考
	* [Golang调度器](https://studygolang.com/articles/9610)
	* [Golang并发模型](https://www.jianshu.com/p/f9024e250ac6)
	* [redis引用计数内存回收](https://iscod.github.io/#/nosql/redis?id=%e5%86%85%e5%ad%98%e5%9b%9e%e6%94%b6)
