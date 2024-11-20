# Go 语言泛型

## 概述

Go 语言从 Go 1.18 版本开始支持泛型。这是 Go 开源以来，在语法层面最大的一次变动。

泛型，是将算法与数据类型解耦，实现算法更广泛的复用性。

```go
package generics

import (
	"fmt"
	"testing"
)

func TestGenerics(t *testing.T) {
    // 没有泛型的情况下，我们需要针对不同数据类型，重复实现相同的算法逻辑
	fmt.Printf("Add(1, 2) = %d\n", Add(1, 2))
	// fmt.Printf("Add(1.1, 2.1) = %f\n", Add(1.1, 2.1)) // 编译错误：类型不匹配

	fmt.Printf("GenericsAdd(1, 2) = %d\n", GenericsAdd(1, 2))
	fmt.Printf("GenericsAdd(1.1, 2.1) = %f\n", GenericsAdd(1.1, 2.1))
}

func Add(a, b int32) int32 {
	return a + b
}

func GenericsAdd[T int | float64](a, b T) T {
	return a + b
}
```

没有泛型之前，通常使用空接口类型 interface{}，缺点：
- 性能有损失；
- 无法进行类型安全检查。

## 基本语法

### 类型参数

类型参数是在函数声明、方法声明的**入参的类型**，在泛型函数或泛型类型实例化时，类型参数会被一个类型实参替换。

以函数为例，对普通函数的参数与泛型函数的类型参数作一下对比。

```go
// 普通函数的参数列表
func Foo(a, b int, c string) {
	// a, b, c 是参数名，int 和 string 是类型
}

// 泛型函数的类型参数
func GenericFoo[P int, Q string](a, b P, c Q) {
	// a, b, c 是参数名，P 和 Q 是类型
	// int, string 代表对参数名类型的约束, 我们可以理解为对参数类型可选值的一种限定
}
```

GenericFoo 泛型函数的声明，比普通函数多了一个组成部分：参数类型列表。**它位于函数名和函数参数列表之间，通过一个方括号括起。** 

泛型函数不支持变长类型参数，例如：`func GenericFoo[P ...int](a, b P)`。

P、Q 在参数列表中，像一个未知类型的占位符，等到泛型函数具化时才能确定参数类型。

### 约束

约束，规定一个类型实参必须满足的条件要求。

在 Go 泛型中，**我们使用 interface 类型来定义约束。**

> Go 1.18 以后，interface 既可以声明接口的方法集合，也可以用作泛型函数入参的类型列表。

```go
// C1 定义约束
// ~int | ~int32，带 ~ 的约束，代表接受指定的类型；接受以这些类型为底层类型的所有类型
// ~ 更灵活，因为它支持自定义类型。如果不带 ~ 的约束，只接受确切的类型，不接受基于这些类型定义的新类型
type C1 interface {
	~int | ~int32
	M1()
}

func foo[P C1](t P) {

}

type T struct{}

func (t T) M1() {}

type T1 int

func (t T1) M1() {

}

func TestGenericConstraint(t *testing.T) {
	var t1 T1
	foo(t1)

	// var t T
	// foo(t) // 编译错误：T 没有实现 C1 接口，t 的底层类型是 struct，不是 int 或 int32
}
```

建议将做约束的接口类型与做传统接口的接口类型，分开定义。

## 类型具化

通过一个泛型版本 Sort 函数的调用例子，看看调用泛型函数的过程都发生了什么：

```go
// Sort 函数定义了一个泛型约束，要求 Elem 类型必须实现 Less 方法
func Sort[Elem interface{ Less(y Elem) bool }](list []Elem) {}

type book struct{}

func (x book) Less(y book) bool {
	return true
}

func TestGenericInstantiation(t *testing.T) {
	var bookshelf []book
	Sort[book](bookshelf) // 泛型函数调用
}
```

上面的泛型函数调用 Sort[book]，分为两个阶段：

- 阶段一：具化
```go
func TestGenericInstantiation(t *testing.T) {
    var bookshelf []book
    Sort(bookshelf) // 在这里，编译器会进行具化
    
    // 具化后相当于编译器生成了这样的函数：
    // func Sort_book(list []book) {}
```

说白了，具化，会让编译器会为每个不同的具体类型生成专门的函数。

- 阶段二：调用
和普通的函数调用没有区别，调用的是编译器具化后生成的函数 Sort_book。

有一个编译器的细节需要注意，编译器会进行实参类型参数的自动推导，我们例子中的泛型函数调用可以修改为：

```
func TestGenericInstantiation(t *testing.T) {
	var bookshelf []book
	Sort(bookshelf) // 去掉了[book]
}
```

## 泛型类型
除了函数可以携带类型参数变身为“泛型函数”外，类型也可以拥有类型参数而化身为“泛型类型”：

```go
// Vector 是一个泛型类型，它接受一个类型参数 T，T 的约束为 any
// any 在 Go 1.18 中，是 interface{} 的别名
// 用 any 作为类型参数的约束，代表没有任何约束
type Vector[T any] []T
```

使用泛型类型，也要遵循先具化，再使用：
```go
type Vector[T any] []T

func (v Vector[T]) Print() {
	fmt.Println(v)
}

func TestGenericType(t *testing.T) {
	// 用类型实参对泛型类型进行了具化
	// 具化后的底层类型为 []int
	var iv = Vector[int]{1, 2, 3}

	iv.Print() // [1, 2, 3]
}
```

## 泛型的性能

- Go 标准库 sort 包（非泛型版）的 Ints 函数；
- Go 团队维护 golang.org/x/exp/slices 中的泛型版 Sort 函数。

```shell
goos: darwin
goarch: amd64
pkg: guowei-gong/bryan/generics
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkSlicesSort-12               163           7611486 ns/op               0 B/op          0 allocs/op
BenchmarkIntSort-12                  174           6384800 ns/op               0 B/op          0 allocs/op
PASS
ok      guowei-gong/bryan/generics      7.962s
```

我本地使用的 Go 版本是 1.22，得出的结论是泛型是有一定性能损失的，不过内存效率上，两种都实现了零内存分配。

同样的代码，在 Go 1.18 上，得出的结论是泛型比无泛型的性能要高出一倍。

从编译速度的角度来说，Go 1.22 的编译速度比 Go 1.18 更快。

## 泛型的适用场景

- 编写通用数据结构

Go 中没有预定义的数据类型，例如树、链表等。
```go
// 泛型链表
type List[T any] struct {
    head *Node[T]
}

// 泛型树
type Tree[T any] struct {
    value T
    left, right *Tree[T]
}
```

- 函数支持多类型参数

当编写的函数的操作元素的类型为 slice、map、channel 等特定类型的时候。如果一个函数接受这些类型的形参，并且**函数代码没有对参数的元素类型作出任何假设**。在这种场合下，泛型方案可以替代反射方案，获得更高的性能。
