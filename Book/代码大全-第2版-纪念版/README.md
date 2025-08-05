# 第十四章 组织直线型代码

# 第十五章 使用条件语句

## if-else
- 正常情况的处理逻辑放 if 后面，而不是 else 后面
- if 子句后面跟随一个有意义的语句

不推荐的做法：
```go
func processFile(filename string) {
    // 检查文件是否存在
    if _, err := os.Stat(filename); err != nil {
        // 如果文件不存在或发生错误，这里是空的，没有有意义的语句
    } else {
        // 实际处理文件的逻辑都在 else 块中
        fmt.Printf("开始处理文件: %s\n", filename)
        // ... 其他文件处理逻辑
    }
}
```

代码阅读起来不直观，需要先看一个空的代码块，才能找到核心逻辑。

推荐的改进方式：
```go
func processFile(filename string) {
    // 检查文件是否存在，并处理错误情况
    if _, err := os.Stat(filename); err != nil {
        // 在 if 块中直接处理错误，让它成为一个有意义的语句
        fmt.Printf("文件 %s 不存在或访问出错: %v\n", filename, err)
        return
    }

    // 文件存在，可以继续处理，这里就是核心逻辑
    fmt.Printf("开始处理文件: %s\n", filename)
    // ... 其他文件处理逻辑
}
```

## case 语句

- 根据不同 case 的**执行频率**，对它们进行排序。如果各个 case 的执行频率没有明显差异时，**按照字母顺序进行排列**
- default 应作为一种“安全网”（防御性编程的手段），捕获所有未被其他 case 覆盖的输入，而不是处理常见情况

不推荐的做法：
```go
package main

import "fmt"

const (
	VIP      = "VIP"
	REGULAR  = "REGULAR"
	GUEST    = "GUEST"
)

func getDiscount(userType string) float64 {
	switch userType {
	case VIP:
		return 0.2 // VIP 用户 20% 折扣
	case REGULAR:
		return 0.1 // 普通用户 10% 折扣
	default: // 这里处理了最常见的 GUEST 情况
		return 0.05 // 游客 5% 折扣
	}
}

func main() {
	fmt.Printf("VIP 用户折扣: %.2f\n", getDiscount(VIP))
	fmt.Printf("普通用户折扣: %.2f\n", getDiscount(REGULAR))
	fmt.Printf("游客折扣: %.2f\n", getDiscount(GUEST))
	fmt.Printf("未知用户折扣: %.2f\n", getDiscount("UNKNOWN"))
}
```

在这个例子中，GUEST 是一个明确定义的常量，也是一个常见的用户类型，但它被放在了 default 子句中。
- 问题 1：读者需要查看 case 列表，发现没有 GUEST，然后才能在 default 中找到它的逻辑。这与我们预期的“从上到下查找”的阅读习惯相悖
- 问题 2：如果未来新增了一种用户类型，比如 PREMIUM，而你忘了添加 case，那么 PREMIUM 用户也会意外地被 default 处理，得到 5% 的折扣，这可能会引发意想不到的错误

推荐的做法：
```go
package main

import "fmt"

const (
	VIP      = "VIP"
	REGULAR  = "REGULAR"
	GUEST    = "GUEST"
)

func getDiscount(userType string) float64 {
	switch userType {
	case VIP:
		return 0.2
	case REGULAR:
		return 0.1
	case GUEST: // 将 GUEST 作为一个明确的 case
		return 0.05
	default: // default 只处理未知的、意料之外的情况
		fmt.Printf("警告：未知的用户类型 -> %s\n", userType)
		return 0.0
	}
}

func main() {
	fmt.Printf("VIP 用户折扣: %.2f\n", getDiscount(VIP))
	fmt.Printf("普通用户折扣: %.2f\n", getDiscount(REGULAR))
	fmt.Printf("游客折扣: %.2f\n", getDiscount(GUEST))
	fmt.Printf("未知用户折扣: %.2f\n", getDiscount("UNKNOWN"))
}
```

# 第二十五章 代码调整策略

优化性能是一个系统性的工程。解决性能问题要**从上往下**思考，先审视宏观设计，再考虑微观细节

## 性能和代码调整

- 硬件是性能的基础
- 选择合适的工具和技术栈
- I/O 是性能瓶颈的常见原因
- 聚焦于真正影响用户体验的地方，比如，最常见的操作路径
- 关注 90% 用户的体验，不需要为了 10% 的用户去投入大量精力和资源
- 性能问题往往不是某个函数或循环的锅，而是**整体架构和设计**的产物
- 代码调整是战术性的，而不是战略性的。在**程序设计**没有明显缺陷、**编译器**已充分利用、**I/O 瓶颈**已解决、**硬件**已足够的情况下，才对具体代码细节进行的微调。

## Pareto 法则
找出性能瓶颈，把你的优化努力集中在这些地方。

## 代码调整方法总结
1. 用设计良好的代码来开发软件。性能优化最好的时机是在项目开始之初
2. 程序已经存在性能问题时，应该采取的实际行动

	a. 保存程序的可运行版本

	b. 对系统进行分析测量，找出瓶颈

	c. 判断性能低下是否源于设计、数据类型或算法

	d. 对步骤 c 中所确定的瓶颈代码进行调整

	e. 每次调整后对性能提升进行测试

	f. 如果调整没有改进性能，就恢复到步骤 a 的代码
	
3. 重复步骤 2

# 第二十六章 代码调整技术

## 逻辑

### 1.知道答案后就停止判断，即短路求值
```go
func processUser(user *User) bool {
    // 假设 user.IsActive() 是一个快速操作，而 user.CanAccessFeature() 是一个耗时的数据库查询
    if user != nil && user.IsActive() && user.CanAccessFeature() {
        // 如果 user 是 nil，后续的条件就不会被评估，避免了空指针 panic
        return true
    }
    return false
}
```

### 2.按结果出现的频率来调整判断顺序

### 3.用查询表替代复杂表达式

当你有多个 if-else 语句，并且条件是离散值时，用一个 map来查找会更高效和简洁。

```go
// 伪代码: 用查询表替代复杂表达式
// 原始的 if-else if
func getProductPrice(sku string) float64 {
    if sku == "SKU-A" { return 10.0 }
    if sku == "SKU-B" { return 15.0 }
    if sku == "SKU-C" { return 25.0 }
    return 0
}

// 优化后的查询表
var productPrices = map[string]float64{
    "SKU-A": 10.0,
    "SKU-B": 15.0,
    "SKU-C": 25.0,
}
func getProductPriceOptimized(sku string) float64 {
    if price, ok := productPrices[sku]; ok {
        return price
    }
    return 0
}
```

### 4.当数据真正被需要时，才去计算或加载它，即惰性求值

## 循环

### 1.将判断外提

如果循环中的某个判断条件，在循环体内不会改变，就把它提到循环外面，避免每次循环都做重复的判断。

> 性能优化的目标不是减少代码行数，而是减少 CPU 执行的指令数。

```go
// 伪代码: 将判断外提
// 原始代码
func processArray(arr []int, reverse bool) {
    for i := 0; i < len(arr); i++ {
        if reverse {
            // ...
        } else {
            // ...
        }
    }
}

// 优化后的代码
func processArrayOptimized(arr []int, reverse bool) {
    if reverse {
        for i := 0; i < len(arr); i++ {
            // ...
        }
    } else {
        for i := 0; i < len(arr); i++ {
            // ...
        }
    }
}
```

### 2.将相似循环合并

```go
// 伪代码: 将相似循环合并
// 原始代码
for i := 0; i < len(users); i++ {
    users[i].LastLogin = time.Now()
}
for i := 0; i < len(users); i++ {
    users[i].SessionCount++
}

// 优化后的代码
for i := 0; i < len(users); i++ {
    users[i].LastLogin = time.Now()
    users[i].SessionCount++
}
```

### 3.尽可能减少在循环中的工作

```go
// 伪代码: 减少循环中的工作
// 原始代码
func calculateTotalPrice(items []Item) float64 {
    total := 0.0
    for i := 0; i < len(items); i++ {
        taxRate := 0.08 // 每次循环都重新定义和赋值
        total += items[i].Price * (1 + taxRate)
    }
    return total
}

// 优化后的代码
func calculateTotalPriceOptimized(items []Item) float64 {
    total := 0.0
    taxRate := 0.08 // 移到循环外部
    for i := 0; i < len(items); i++ {
        total += items[i].Price * (1 + taxRate)
    }
    return total
}
```

### 4.将最忙的循环放在最内层

在嵌套循环中，将执行次数最多的循环放在最内层，可以提高缓存的局部性，并减少外层循环的开销。

```go
// 伪代码: 最忙的循环放在最内层
// 原始代码 (假设 cols >> rows)
for i := 0; i < rows; i++ {
    for j := 0; j < cols; j++ {
        // ...
    }
}

// 优化后的代码 (最忙的循环放在内层)
for j := 0; j < cols; j++ {
    for i := 0; i < rows; i++ {
        // ...
    }
}
```

## 表达式

### 1.削减运算强度
```go
// 伪代码: 削减运算强度
x := 10
// 原始代码
y := x * 2

// 优化后的代码（对于 2 的幂次乘除法）
y := x << 1
```

### 2.编译初始化
```go
// 伪代码: 编译初始化
// 原始代码
func calculate(x int) int {
    someValue := 1024 * 1024 // 在每次函数调用时都计算
    return x * someValue
}

// 优化后的代码
const someValue = 1024 * 1024 // 编译时确定
func calculateOptimized(x int) int {
    return x * someValue
}
```

### 3.将需要多次使用的计算结果保存下来
```go
// 伪代码: 将计算结果保存下来
// 原始代码
func processComplex(a, b float64) float64 {
    return math.Sin(a)*math.Cos(a) + math.Sin(a)*math.Cos(a) // 重复计算了 math.Sin(a)*math.Cos(a)
}

// 优化后的代码
func processComplexOptimized(a, b float64) float64 {
    val := math.Sin(a) * math.Cos(a) // 只计算一次
    return val + val
}
```

## 子程序

### 1.将良好的子程序拆解

Go 语言中的函数调用开销非常小，并且编译器通常会进行**内联**优化。所以，把大函数拆解成多个小函数，反而可能因为内联而提升性能，同时提高可读性。

```go
// 伪代码: 将良好的子程序拆解
// 原始代码: 一个大函数
func processRequest(req *Request) {
    // 1. 验证请求...
    // 2. 从数据库获取数据...
    // 3. 计算结果...
    // 4. 格式化响应...
}

// 优化后的代码: 拆解成小函数
func processRequestOptimized(req *Request) {
    if !validateRequest(req) {
        return
    }
    data := getDataFromDB(req)
    result := calculateResult(data)
    response := formatResponse(result)
    sendResponse(response)
}
```