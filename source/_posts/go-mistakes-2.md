---
title: Golang 需要避免踩的 50 个坑（二）
categories: Golang
tags:
  - Golang
  - Go总结
abbrlink: 53194
date: 2019-01-25 00:00:00
---


> 最近准备写一些关于golang的技术博文，本文是之前在GitHub上看到的golang技术译文，感觉很有帮助，先给各位读者分享一下。

## 前言

Go 是一门简单有趣的编程语言，与其他语言一样，在使用时不免会遇到很多坑，不过它们大多不是 Go 本身的设计缺陷。如果你刚从其他语言转到 Go，那这篇文章里的坑多半会踩到。

如果花时间学习官方 doc、wiki、[讨论邮件列表](https://groups.google.com/forum/#!forum/golang-nuts)、 [Rob Pike](https://github.com/robpike) 的大量文章以及 Go 的源码，会发现这篇文章中的坑是很常见的，新手跳过这些坑，能减少大量调试代码的时间。

## 初级篇：1-35（二）

### 18. string 与索引操作符

对字符串用索引访问返回的不是字符，而是一个 byte 值。

这种处理方式和其他语言一样，比如 PHP 中：

```shell
> php -r '$name="中文"; var_dump($name);'	# "中文" 占用 6 个字节
string(6) "中文"

> php -r '$name="中文"; var_dump($name[0]);' # 把第一个字节当做 Unicode 字符读取，显示 U+FFFD
string(1) "�"	

> php -r '$name="中文"; var_dump($name[0].$name[1].$name[2]);'
string(3) "中"
```

```go
func main() {
	x := "ascii"
	fmt.Println(x[0])		// 97
	fmt.Printf("%T\n", x[0])// uint8
}
```

如果需要使用 `for range` 迭代访问字符串中的字符（unicode code point / rune），标准库中有 `"unicode/utf8"` 包来做 UTF8 的相关解码编码。另外 [utf8string](https://godoc.org/golang.org/x/exp/utf8string) 也有像 `func (s *String) At(i int) rune` 等很方便的库函数。



### 19. 字符串并不都是 UTF8 文本

string 的值不必是 UTF8 文本，可以包含任意的值。只有字符串是文字字面值时才是 UTF8 文本，字串可以通过转义来包含其他数据。

判断字符串是否是 UTF8 文本，可使用 "unicode/utf8" 包中的 `ValidString()` 函数：

```go
func main() {
	str1 := "ABC"
	fmt.Println(utf8.ValidString(str1))	// true

	str2 := "A\xfeC"
	fmt.Println(utf8.ValidString(str2))	// false

	str3 := "A\\xfeC"
	fmt.Println(utf8.ValidString(str3))	// true	// 把转义字符转义成字面值
}
```



### 20. 字符串的长度

在 Python 中：

```python
data = u'♥'  
print(len(data)) # 1
```

然而在 Go 中：

```go
func main() {
	char := "♥"
	fmt.Println(len(char))	// 3
}
```

Go 的内建函数 `len()` 返回的是字符串的  byte 数量，而不是像 Python  中那样是计算 Unicode 字符数。

如果要得到字符串的字符数，可使用 "unicode/utf8" 包中的 `RuneCountInString(str string) (n int)`

```go
func main() {
	char := "♥"
	fmt.Println(utf8.RuneCountInString(char))	// 1
}
```

**注意：** `RuneCountInString` 并不总是返回我们看到的字符数，因为有的字符会占用 2 个 rune：

```go
func main() {
	char := "é"
	fmt.Println(len(char))	// 3
	fmt.Println(utf8.RuneCountInString(char))	// 2
	fmt.Println("cafe\u0301")	// café	// 法文的 cafe，实际上是两个 rune 的组合
}
```

参考：[normalization](https://blog.golang.org/normalization)



### 21. 在多行 array、slice、map 语句中缺少 `,` 号

```go
func main() {
	x := []int {
		1,
		2	// syntax error: unexpected newline, expecting comma or }
	}
	y := []int{1,2,}	
	z := []int{1,2}	
	// ...
}
```

声明语句中 `}` 折叠到单行后，尾部的 `,` 不是必需的。



### 22. `log.Fatal` 和 `log.Panic` 不只是 log

log 标准库提供了不同的日志记录等级，与其他语言的日志库不同，Go 的 log 包在调用 `Fatal*()`、`Panic*()` 时能做更多日志外的事，如中断程序的执行等：

```go
func main() {
	log.Fatal("Fatal level log: log entry")		// 输出信息后，程序终止执行
	log.Println("Nomal level log: log entry")
}
```



### 23. 对内建数据结构的操作并不是同步的 

尽管 Go 本身有大量的特性来支持并发，但并不保证并发的数据安全，用户需自己保证变量等数据以原子操作更新。

goroutine 和 channel 是进行原子操作的好方法，或使用 "sync" 包中的锁。



### 24. range 迭代 string 得到的值

range 得到的索引是字符值（Unicode point / rune）第一个字节的位置，与其他编程语言不同，这个索引并不直接是字符在字符串中的位置。

注意一个字符可能占多个 rune，比如法文单词 café 中的 é。操作特殊字符可使用[norm](https://golang.org/pkg/vendor/golang_org/x/text/unicode/norm/) 包。

for range 迭代会尝试将 string 翻译为 UTF8 文本，对任何无效的码点都直接使用 0XFFFD rune（�）UNicode 替代字符来表示。如果 string 中有任何非 UTF8 的数据，应将 string 保存为 byte slice 再进行操作。

```go
func main() {
	data := "A\xfe\x02\xff\x04"
	for _, v := range data {
		fmt.Printf("%#x ", v)	// 0x41 0xfffd 0x2 0xfffd 0x4	// 错误
	}

	for _, v := range []byte(data) {
		fmt.Printf("%#x ", v)	// 0x41 0xfe 0x2 0xff 0x4	// 正确
	}
}
```



### 25. range 迭代 map

如果你希望以特定的顺序（如按 key 排序）来迭代 map，要注意每次迭代都可能产生不一样的结果。

Go 的运行时是有意打乱迭代顺序的，所以你得到的迭代结果可能不一致。但也并不总会打乱，得到连续相同的 5 个迭代结果也是可能的，如：

```go
func main() {
	m := map[string]int{"one": 1, "two": 2, "three": 3, "four": 4}
	for k, v := range m {
		fmt.Println(k, v)
	}
}
```

如果你去 [Go Playground](https://play.golang.org/) 重复运行上边的代码，输出是不会变的，只有你更新代码它才会重新编译。重新编译后迭代顺序是被打乱的：

 ![](https://camo.githubusercontent.com/fc483a934db4c09102bbe949fa9996f2b49732db/687474703a2f2f70326a357338666d722e626b742e636c6f7564646e2e636f6d2f6d61702d72616e67652e706e67)



### 26. switch 中的 fallthrough 语句

`switch` 语句中的 `case` 代码块会默认带上 break，但可以使用 `fallthrough` 来强制执行下一个 case 代码块。

```go
func main() {
	isSpace := func(char byte) bool {
		switch char {
		case ' ':	// 空格符会直接 break，返回 false // 和其他语言不一样
		// fallthrough	// 返回 true
		case '\t':
			return true
		}
		return false
	}
	fmt.Println(isSpace('\t'))	// true
	fmt.Println(isSpace(' '))	// false
}
```

不过你可以在 case 代码块末尾使用 `fallthrough`，强制执行下一个 case 代码块。

也可以改写 case 为多条件判断：

```go
func main() {
	isSpace := func(char byte) bool {
		switch char {
		case ' ', '\t':
			return true
		}
		return false
	}
	fmt.Println(isSpace('\t'))	// true
	fmt.Println(isSpace(' '))	// true
}
```



### 27. 自增和自减运算

很多编程语言都自带前置后置的 `++`、`--` 运算。但 Go 特立独行，去掉了前置操作，同时 `++`、`—`  只作为运算符而非表达式。

```go
// 错误示例
func main() {
	data := []int{1, 2, 3}
	i := 0
	++i			// syntax error: unexpected ++, expecting }
	fmt.Println(data[i++])	// syntax error: unexpected ++, expecting :
}


// 正确示例
func main() {
	data := []int{1, 2, 3}
	i := 0
	i++
	fmt.Println(data[i])	// 2
}
```



### 28. 按位取反

很多编程语言使用 `~` 作为一元按位取反（NOT）操作符，Go 重用 `^` XOR 操作符来按位取反：

```go
// 错误的取反操作
func main() {
	fmt.Println(~2)		// bitwise complement operator is ^
}


// 正确示例
func main() {
	var d uint8 = 2
	fmt.Printf("%08b\n", d)		// 00000010
	fmt.Printf("%08b\n", ^d)	// 11111101
}
```

同时 `^` 也是按位异或（XOR）操作符。

一个操作符能重用两次，是因为一元的 NOT 操作 `NOT 0x02`，与二元的 XOR 操作 `0x22 XOR 0xff` 是一致的。

Go 也有特殊的操作符 AND NOT `&^` 操作符，不同位才取1。

```go
func main() {
	var a uint8 = 0x82
	var b uint8 = 0x02
	fmt.Printf("%08b [A]\n", a)
	fmt.Printf("%08b [B]\n", b)

	fmt.Printf("%08b (NOT B)\n", ^b)
	fmt.Printf("%08b ^ %08b = %08b [B XOR 0xff]\n", b, 0xff, b^0xff)

	fmt.Printf("%08b ^ %08b = %08b [A XOR B]\n", a, b, a^b)
	fmt.Printf("%08b & %08b = %08b [A AND B]\n", a, b, a&b)
	fmt.Printf("%08b &^%08b = %08b [A 'AND NOT' B]\n", a, b, a&^b)
	fmt.Printf("%08b&(^%08b)= %08b [A AND (NOT B)]\n", a, b, a&(^b))
}
```

```shell
10000010 [A]
00000010 [B]
11111101 (NOT B)
00000010 ^ 11111111 = 11111101 [B XOR 0xff]
10000010 ^ 00000010 = 10000000 [A XOR B]
10000010 & 00000010 = 00000010 [A AND B]
10000010 &^00000010 = 10000000 [A 'AND NOT' B]
10000010&(^00000010)= 10000000 [A AND (NOT B)]
```



### 29. 运算符的优先级

除了位清除（bit clear）操作符，Go 也有很多和其他语言一样的位操作符，但优先级另当别论。

```go
func main() {
	fmt.Printf("0x2 & 0x2 + 0x4 -> %#x\n", 0x2&0x2+0x4)	// & 优先 +
	//prints: 0x2 & 0x2 + 0x4 -> 0x6
	//Go:    (0x2 & 0x2) + 0x4
	//C++:    0x2 & (0x2 + 0x4) -> 0x2

	fmt.Printf("0x2 + 0x2 << 0x1 -> %#x\n", 0x2+0x2<<0x1)	// << 优先 +
	//prints: 0x2 + 0x2 << 0x1 -> 0x6
	//Go:     0x2 + (0x2 << 0x1)
	//C++:   (0x2 + 0x2) << 0x1 -> 0x8

	fmt.Printf("0xf | 0x2 ^ 0x2 -> %#x\n", 0xf|0x2^0x2)	// | 优先 ^
	//prints: 0xf | 0x2 ^ 0x2 -> 0xd
	//Go:    (0xf | 0x2) ^ 0x2
	//C++:    0xf | (0x2 ^ 0x2) -> 0xf
}
```

优先级列表：

```go
Precedence    Operator
    5             *  /  %  <<  >>  &  &^
    4             +  -  |  ^
    3             ==  !=  <  <=  >  >=
    2             &&
    1             ||
```



### 30. 不导出的 struct 字段无法被 encode

以小写字母开头的字段成员是无法被外部直接访问的，所以  `struct` 在进行 json、xml、gob 等格式的 encode 操作时，这些私有字段会被忽略，导出时得到零值：

```go
func main() {
	in := MyData{1, "two"}
	fmt.Printf("%#v\n", in)	// main.MyData{One:1, two:"two"}

	encoded, _ := json.Marshal(in)
	fmt.Println(string(encoded))	// {"One":1}	// 私有字段 two 被忽略了

	var out MyData
	json.Unmarshal(encoded, &out)
	fmt.Printf("%#v\n", out) 	// main.MyData{One:1, two:""}
}
```



### 31. 程序退出时还有 goroutine 在执行

程序默认不等所有 goroutine 都执行完才退出，这点需要特别注意：

```go
// 主程序会直接退出
func main() {
	workerCount := 2
	for i := 0; i < workerCount; i++ {
		go doIt(i)
	}
	time.Sleep(1 * time.Second)
	fmt.Println("all done!")
}

func doIt(workerID int) {
	fmt.Printf("[%v] is running\n", workerID)
	time.Sleep(3 * time.Second)		// 模拟 goroutine 正在执行 
	fmt.Printf("[%v] is done\n", workerID)
}
```

如下，`main()` 主程序不等两个 goroutine 执行完就直接退出了：

 ![](https://camo.githubusercontent.com/a7d1ffa69ccb3ece7c04bdc962aa6a0c8afab2c8/687474703a2f2f70326a357338666d722e626b742e636c6f7564646e2e636f6d2f676f726f7574696e652d65786974732e706e67)

常用解决办法：使用 "WaitGroup"  变量，它会让主程序等待所有 goroutine 执行完毕再退出。

如果你的 goroutine 要做消息的循环处理等耗时操作，可以向它们发送一条 `kill` 消息来关闭它们。或直接关闭一个它们都等待接收数据的 channel：

```go
// 等待所有 goroutine 执行完毕
// 进入死锁
func main() {
	var wg sync.WaitGroup
	done := make(chan struct{})

	workerCount := 2
	for i := 0; i < workerCount; i++ {
		wg.Add(1)
		go doIt(i, done, wg)
	}

	close(done)
	wg.Wait()
	fmt.Println("all done!")
}

func doIt(workerID int, done <-chan struct{}, wg sync.WaitGroup) {
	fmt.Printf("[%v] is running\n", workerID)
	defer wg.Done()
	<-done
	fmt.Printf("[%v] is done\n", workerID)
}
```

执行结果：

 ![](https://camo.githubusercontent.com/80a2a3178a9f12a05da5cdaf6c0a92223b6c8de4/687474703a2f2f70326a357338666d722e626b742e636c6f7564646e2e636f6d2f646561642d676f726f7574696e652e706e67)

看起来好像 goroutine 都执行完了，然而报错：

> fatal error: all goroutines are asleep - deadlock!

为什么会发生死锁？goroutine 在退出前调用了 `wg.Done()` ，程序应该正常退出的。

原因是 goroutine 得到的 "WaitGroup" 变量是 `var wg WaitGroup` 的一份拷贝值，即 `doIt()` 传参只传值。所以哪怕在每个 goroutine 中都调用了 `wg.Done()`， 主程序中的 `wg` 变量并不会受到影响。

```go
// 等待所有 goroutine 执行完毕
// 使用传址方式为 WaitGroup 变量传参
// 使用 channel 关闭 goroutine

func main() {
	var wg sync.WaitGroup
	done := make(chan struct{})
	ch := make(chan interface{})

	workerCount := 2
	for i := 0; i < workerCount; i++ {
		wg.Add(1)
        go doIt(i, ch, done, &wg)	// wg 传指针，doIt() 内部会改变 wg 的值
	}

	for i := 0; i < workerCount; i++ {	// 向 ch 中发送数据，关闭 goroutine
		ch <- i
	}

	close(done)
	wg.Wait()
	close(ch)
	fmt.Println("all done!")
}

func doIt(workerID int, ch <-chan interface{}, done <-chan struct{}, wg *sync.WaitGroup) {
	fmt.Printf("[%v] is running\n", workerID)
	defer wg.Done()
	for {
		select {
		case m := <-ch:
			fmt.Printf("[%v] m => %v\n", workerID, m)
		case <-done:
			fmt.Printf("[%v] is done\n", workerID)
			return
		}
	}
}
```

运行效果：

 ![](https://camo.githubusercontent.com/0d75cf42415176fb17baaaf49f87066ce9572ef4/687474703a2f2f70326a357338666d722e626b742e636c6f7564646e2e636f6d2f72696768742d676f726f7574696e652e706e67)



### 32. 向无缓冲的 channel 发送数据，只要 receiver 准备好了就会立刻返回

只有在数据被 receiver 处理时，sender 才会阻塞。因运行环境而异，在 sender 发送完数据后，receiver 的 goroutine 可能没有足够的时间处理下一个数据。如：

```go
func main() {
	ch := make(chan string)

	go func() {
		for m := range ch {
			fmt.Println("Processed:", m)
			time.Sleep(1 * time.Second)	// 模拟需要长时间运行的操作
		}
	}()

	ch <- "cmd.1"
	ch <- "cmd.2" // 不会被接收处理
}
```

运行效果：

 ![](https://camo.githubusercontent.com/389f60f73f92af8cc0f5ea0c88fc1ff46a4cd514/687474703a2f2f70326a357338666d722e626b742e636c6f7564646e2e636f6d2f756e6275666665642d6368616e2e706e67)



### 33. 向已关闭的 channel 发送数据会造成 panic

从已关闭的 channel 接收数据是安全的：

接收状态值 `ok` 是 `false` 时表明 channel 中已没有数据可以接收了。类似的，从有缓冲的 channel 中接收数据，缓存的数据获取完再没有数据可取时，状态值也是 `false`

向已关闭的 channel 中发送数据会造成 panic：

```go
func main() {
	ch := make(chan int)
	for i := 0; i < 3; i++ {
		go func(idx int) {
			ch <- idx
		}(i)
	}

	fmt.Println(<-ch)		// 输出第一个发送的值
	close(ch)			// 不能关闭，还有其他的 sender
	time.Sleep(2 * time.Second)	// 模拟做其他的操作
}
```

运行结果：

 ![](https://camo.githubusercontent.com/ecf7f13f03d4ddfd84ed366a53bc55a0cc2d19ba/687474703a2f2f70326a357338666d722e626b742e636c6f7564646e2e636f6d2f6368616e6e6e656c2e706e67)

针对上边有 bug 的这个例子，可使用一个废弃 channel `done` 来告诉剩余的  goroutine 无需再向 ch 发送数据。此时 `<- done` 的结果是 `{}`：

```go
func main() {
	ch := make(chan int)
	done := make(chan struct{})

	for i := 0; i < 3; i++ {
		go func(idx int) {
			select {
			case ch <- (idx + 1) * 2:
				fmt.Println(idx, "Send result")
			case <-done:
				fmt.Println(idx, "Exiting")
			}
		}(i)
	}

	fmt.Println("Result: ", <-ch)
	close(done)
	time.Sleep(3 * time.Second)
}
```

 运行效果：

 ![](https://camo.githubusercontent.com/9d126912fef4aa62d975e2e0f254f2db2261a2a3/687474703a2f2f70326a357338666d722e626b742e636c6f7564646e2e636f6d2f6e6f726d616c2d6368616e6e656c2e706e67)





### 34. 使用了值为 `nil ` 的 channel

在一个值为 nil 的 channel 上发送和接收数据将永久阻塞：
```go
func main() {
	var ch chan int // 未初始化，值为 nil
	for i := 0; i < 3; i++ {
		go func(i int) {
			ch <- i
		}(i)
	}

	fmt.Println("Result: ", <-ch)
	time.Sleep(2 * time.Second)
}
```

runtime 死锁错误：

> fatal error: all goroutines are asleep - deadlock!
> goroutine 1 [chan receive (nil chan)]

利用这个死锁的特性，可以用在 select 中动态的打开和关闭 case 语句块：

```go
func main() {
	inCh := make(chan int)
	outCh := make(chan int)

	go func() {
		var in <-chan int = inCh
		var out chan<- int
		var val int

		for {
			select {
			case out <- val:
				println("--------")
				out = nil
				in = inCh
			case val = <-in:
				println("++++++++++")
				out = outCh
				in = nil
			}
		}
	}()

	go func() {
		for r := range outCh {
			fmt.Println("Result: ", r)
		}
	}()

	time.Sleep(0)
	inCh <- 1
	inCh <- 2
	time.Sleep(3 * time.Second)
}
```

运行效果：

 ![](https://camo.githubusercontent.com/c40fd06d273110da25d6c392160451d2ce0b3f26/687474703a2f2f70326a357338666d722e626b742e636c6f7564646e2e636f6d2f72756e6e732e706e67)



### 35. 若函数 receiver 传参是传值方式，则无法修改参数的原有值

方法 receiver 的参数与一般函数的参数类似：如果声明为值，那方法体得到的是一份参数的值拷贝，此时对参数的任何修改都不会对原有值产生影响。

除非 receiver 参数是 map 或 slice 类型的变量，并且是以指针方式更新 map 中的字段、slice 中的元素的，才会更新原有值:

```go
type data struct {
	num   int
	key   *string
	items map[string]bool
}

func (this *data) pointerFunc() {
	this.num = 7
}

func (this data) valueFunc() {
	this.num = 8
	*this.key = "valueFunc.key"
	this.items["valueFunc"] = true
}

func main() {
	key := "key1"

	d := data{1, &key, make(map[string]bool)}
	fmt.Printf("num=%v  key=%v  items=%v\n", d.num, *d.key, d.items)

	d.pointerFunc()	// 修改 num 的值为 7
	fmt.Printf("num=%v  key=%v  items=%v\n", d.num, *d.key, d.items)

	d.valueFunc()	// 修改 key 和 items 的值
	fmt.Printf("num=%v  key=%v  items=%v\n", d.num, *d.key, d.items)
}
```

运行结果：

 ![](https://camo.githubusercontent.com/bff142d944ec09a52485fb9c4de8f1f576675d5c/687474703a2f2f70326a357338666d722e626b742e636c6f7564646e2e636f6d2f6368616e67652d6f726967616c2e706e67)

#### 系列文章
[Golang 需要避免踩的 50 个坑](http://blueskykong.com/tags/Go%E6%80%BB%E7%BB%93/)


**本文转载自https://github.com/wuYin/blog/blob/master/50-shades-of-golang-traps-gotchas-mistakes.md**

