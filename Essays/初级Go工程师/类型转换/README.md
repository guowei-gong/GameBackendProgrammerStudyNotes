# Go 语言类型转换

## 1. 类型别名转换规则

```go
// 这些类型本质上是完全等价的，只是名字不同而已
// 它们是 Go 语言预定义的类型别名
byte   = uint8
rune   = int32
[]byte = []uint8
```

用 type 定义类型别名时，新类型与原类型完全等价，可以进行隐式转换，如下：

```go
type Text = string
var s string = "hello"
var t Text = s // 可以直接赋值  
```

注意，`type Text string` 与 `type Text = string` 是不同的，前者是定义了一个新类型，后者是定义了一个类型别名。

新类型与原类型，底层类型相同，也不能进行隐式转换。如下：

```go
type Text string
var s string = "hello"
var t Text = s // 错误：不能进行隐式转换
var t2 Text = Text(s) // 正确：可以进行显示转换
```

## 2. 底层类型相关的类型转换规则

### 2.1 基本类型转换例子：
```go
type Age int
type Year int

// TestUnderlying 测试底层类型相关的类型转换规则
// 有相同的底层类型时，允许显示转换
func TestUnderlying(t *testing.T) {
	var age Age = 27
	var year Year

	//year = age // 错误：虽然底层类型相同，但不能直接赋值(隐式转换)
	year = Year(age) // 正确：允许显示转换，因为底层类型相同

	t.Logf("year: %v", year)
}
```

底层类型是引用类型的时候，不能直接显式转换，引用类型包括：slice、map、channel。看完示例后，再解释为什么不能直接显式转换。

### 2.2 slice 相关的转换例子：

```go
func TestSlice(t *testing.T) {
	var a []int32
	var b []int64
	// b = []int64(a) // 错误：不能直接转换

	// 正确的转换方式是遍历
	b = make([]int64, len(a))
	for i, v := range a {
		b[i] = int64(v)
	}
}

// TestSlice2 测试基于相同底层类型的 slice 的转换
func TestSlice2(t *testing.T) {
	// 定义两个基于 []int32 的类型
	type Nums1 []int32
	type Nums2 []int32

	var a Nums1 = []int32{1, 2, 3}
	var b Nums2
	// b = Nums2(a)  // 错误：即使底层类型相同，slice 也不能直接转换

	// 正确的转换方式是遍历
	b = make(Nums2, len(a))
	for i, v := range a {
		b[i] = v // 这里不需要类型转换，因为底层类型相同
	}
}
```

### 2.3 map 相关的转换例子：

```go
func TestMap(t *testing.T) {
	var userScores map[UserID]Score
	var rawData map[string]int

	// 初始化
	userScores = make(map[UserID]Score)
	rawData = map[string]int{
		"user1": 100,
		"user2": 200,
	}

	//userScores = rawData // 错误：不能直接转换

	// 正确的转换方式，手动转换数据
	for k, v := range rawData {
		userScores[UserID(k)] = Score(v)
	}
}
```

### 2.4 channel 相关的转换例子：

```go
func TestChannel(t *testing.T) {
	type Ch1 chan int
	type Ch2 chan int

	var a Ch1
	var b Ch2
	//b = a // 错误：不能直接转换

	// 正确的方式是创建新的 channel 并转发数据
	b = make(Ch2, len(a))
	for v := range a {
		b <- v
	}
}
```

### 2.5 引用类型为什么不能直接显式转换？

引用类型包含了复杂的内部结构，可能包含元信息、指针、其他引用等，直接转换可能会导致数据丢失或损坏。
