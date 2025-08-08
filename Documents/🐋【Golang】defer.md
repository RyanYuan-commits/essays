关键字 defer 允许我们推迟到函数返回之前（或任意位置执行 `return` 语句之后）一刻才执行某个语句或函数。

关键字 defer 的用法类似于面向对象编程语言 Java 和 C# 的 `finally` 语句块，它一般用于释放某些已分配的资源。
```go
package main
import "fmt"

func main() {
	function1()
}

func function1() {
	fmt.Printf("In function1 at the top\n")
	defer function2()
	fmt.Printf("In function1 at the bottom!\n")
}

func function2() {
	fmt.Printf("Function2: Deferred until the end of the calling function!")
}
```
输出：
```
In Function1 at the top
In Function1 at the bottom!
Function2: Deferred until the end of the calling function!
```

当有多个 defer 行为被注册时，它们会以逆序执行（类似栈，即后进先出）：
```go
func f() {
	for i := 0; i < 5; i++ {
		defer fmt.Printf("%d ", i)
	}
}
```
输出 4 3 2 1 0 