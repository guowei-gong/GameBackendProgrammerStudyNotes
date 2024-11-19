# Go 语言类型转换

## 1. 类型别名转换规则

```go
// 这些类型本质上是完全等价的，只是名字不同而已
// 下面是 Go 语言预定义的类型别名
byte   = uint8
rune   = int32
[]byte = []uint8
```

你也可以用 type 自定义类型别名。当新类型与原类型完全等价时，可以进行隐式转换，如下：

```go
type Text = string
var s string = "hello"
var t Text = s // 可以直接赋值  
```

注意，`type Text string` 与 `type Text = string` 是不同的，前者是定义了一个新类型，后者是定义了一个类型别名。

如果定义的是新类型，即使新类型与原类型，底层类型相同，也不能进行隐式转换。如下：

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

底层类型是引用类型的时候，不能直接显式转换，因为引用类型包含了复杂的内部结构，可能包含元信息、指针、其他引用等，直接转换可能会导致数据丢失或损坏。引用类型包括：slice、map、channel。

## 3. slice 相关的转换例子：

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

## 4. map 相关的转换例子：

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

## 5. channel 相关的转换例子：

```go
func TestChannel1(t *testing.T) {
    type C chan string // 双向通道类型
    type C1 chan<- string
    type C2 <-chan string
    
    var ca C
    var cb chan string
    
    cb = ca // ok，可以隐式转换，因为底层类型相同
    ca = cb // ok
    
    // cb 为无名类型，且底层类型相同，可以被隐式转换
    var _ chan<- string = cb // ok
    var _ <-chan string = cb // ok
    var _ C1 = cb            // ok
    var _ C2 = cb            // ok
    
    // C 类型的值无法直接转换为类型 C1 或 C2
    // var _ = C1(ca) // compile error
    // var _ = C2(ca) // compile error
    
    // 但是类型C的值可以间接转换为类型C1或C2。
    var _ = C1((chan<- string)(ca)) // ok
    var _ = C2((<-chan string)(ca)) // ok
    var _ C1 = (chan<- string)(ca)  // ok
    var _ C2 = (<-chan string)(ca)  // ok
}
```

## 6. 指针相关的转换例子：

```go
func TestPtr(t *testing.T) {
	type MyInt int
	type IntPtr *int
	type MyIntPtr *int

	var pi = new(int)  // pi 类型为 *int
	var ip IntPtr = pi // 没问题，因为底层类型都相同。另外，pi 的类型为无名类型（注：无名类型，可以理解为没有使用 type 关键字定义的字面量/匿名结构体/函数类型）

	// var _ *MyInt = pi // 不能隐式转换
	var _ = (*MyInt)(pi) // 显式转换可以

	// Go 1.22.6 版本，类型 *int 的值可以直接转换为类型 MyIntPtr，间接转换反而不行了，使用的旧版本，可以间接地转换过去，直接转换不行
	var _ MyIntPtr = pi  // 隐式转换可以
	var _ = MyIntPtr(pi) // 显式转换也可以

	// 类型 IntPtr 的值，不能被隐转换为类型 MyIntPtr
	// var _ MyIntPtr = ip // 不可以隐式转换
	var _ = MyIntPtr(ip) // 可以显式转换
}
```

## 7. 和接口实现相关的类型转换规则
给定一个值 x 和一个接口类型 I，如果 x 的类型为 Tx 并且类型 Tx 实现了接口类型 I，则 x 可以被隐式转换为类型 I。

此转换的结果为一个类型为 I 的接口值。此接口值包裹了：

- x 的一个副本（如果 Tx 是一个非接口值）；
- x 的动态值的一个副本（如果 Tx 是一个接口值）。

这两种情况都确保了接口值的独立性，避免通过接口，修改原始值的可能性。

```go
type Speaker interface {
	Speak() string
}

type Dog struct {
	Name string
}

func (d Dog) Speak() string {
	return d.Name + " says woof!"
}

// 非接口值转换为接口
func TestInterface1(t *testing.T) {
	dog := Dog{Name: "旺财"}

	// dog 的类型是 Dog，实现了 Speaker 接口
	// 所以可以被隐式转换为 Speaker 接口
	var speaker Speaker = dog

	// 修改 dog 不会影响 speaker
	dog.Name = "小黑"

	fmt.Println(speaker.Speak()) // 输出: "旺财 says woof!"
	fmt.Println(dog.Speak())     // 输出: "小黑 says woof!"
}

type Animal interface {
	Speak() string
}

type Pet interface {
	Animal
	GetName() string
}

type Cat struct {
	Name string
}

func (c Cat) Speak() string {
	return c.Name + " says meow!"
}

func (c Cat) GetName() string {
	return c.Name
}

// 接口值转换为另一个接口
func TestInterface2(t *testing.T) {
	cat := Cat{Name: "咪咪"}

	// cat 转换为 pet 接口
	var pet Pet = cat

	// pet 接口值可以转换为 Animal 接口
	// 因为 Pet 接口包含了 Animal 接口的所有方法
	var animal Animal = pet
	
	fmt.Println(animal.Speak()) // 输出: "咪咪 says meow!"
}
```

## 8. 类型不确定值

```go
func TestUnspecified(t *testing.T) {
	// 值被隐式转换为目标类型了，如果转换不合法（例如 123.1 转换为 int），编译器会报错
	var a int = 123.0
	var b float32 = 123
}
```

