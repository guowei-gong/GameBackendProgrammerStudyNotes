# Map

字典（哈希表）是 Go 语言的内置类型，它是无序键值对集合。可以根据 Key，在 O(1) 的时间复杂度获取到 Value。字典的 Key 可以为任意的可比较数据类型，例如数字、字符串和指针等。

## 底层结构

## 操作性能

字典对象本身就是指针包装，传参时无须再次取地址。
```go
func TestMapPointer(t *testing.T) {
	test := func(m map[string]int) {
		fmt.Printf("传参后 map 的内存地址：%p\n", m)
	}

	m := make(map[string]int)
	fmt.Printf("传参前 map 的内存地址：%p\n", m)
	test(m)
}
```

输出：

```shell
传参前 map 的内存地址：0xc000114510
传参后 map 的内存地址：0xc000114510
```

创建时预先分配内存，可以减少扩容时的内存分配和重新哈希操作，提升性能。
```go
func test() map[int]int {
	m := make(map[int]int)
	for i := 0; i < 1000; i++ {
		m[i] = i
	}

	return m
}

// 预先准备足够的空间
func testCap() map[int]int {
	m := make(map[int]int, 1000)
	for i := 0; i < 1000; i++ {
		m[i] = i
	}

	return m
}

func BenchmarkTest(b *testing.B) {
	for i := 0; i < b.N; i++ {
		test()
	}
}

func BenchmarkTestCap(b *testing.B) {
	for i := 0; i < b.N; i++ {
		testCap()
	}
}
```

输出：

```shell
# ns/op: 每次操作的纳秒数
# B/op: 每次操作，分配的内存字节数
# allocs/op: 每次操作，分配内存的次数
BenchmarkTest-12           18790             62315 ns/op           86551 B/op         64 allocs/op
BenchmarkTestCap-12        45966             26243 ns/op           41097 B/op          6 allocs/op
```

Map 存储海量小对象，应直接存储值，而不是指针，这样有利于减少垃圾回收的压力。

> 小对象没有一个明确的内存大小，根据 Go 社区的经验，一般认为小于 128 字节的对象被认为是小对象，我们可以使用 `unsafe.Sizeof` 来查看对象大小。

> 如果对象需要在多个地方共享，必须使用指针。注意，对象大小不是决定使用值还是指针的唯一要素，还需要考虑：使用频率；是否需要修改；是否需要共享。

使用指针的问题：
- 指针会导致内存碎片化；
- 每个指针都是一个需要被 GC 扫描的对象，如果有 100 万条记录，GC 就需要扫描 100 万个指针。

直接存储值的优势：
- 内存布局更紧凑，更有利于 CPU 缓存；
- 数据直接存储在 map 中，减少了 GC 需要扫描的对象数量。

另外，字典不会收缩内存，适当替换成新对象是必要的。例如删除的值数量，远大于保留的值数量。

## 扩容策略

## 安全

在迭代期间删除或新增键值是安全的。

```go
// 1. 创建了一个包含 0-9 的 map
// 2. 在迭代过程中，当 k=5 时添加了新的键值对 m[100] = 1000
// 3. 每次迭代都会删除当前的键
func TestSafeIterate(t *testing.T) {
	m := make(map[int]int)
	for i := 0; i < 10; i++ {
		m[i] = i + 10
	}

	fmt.Println("开始迭代前:", m)
	visited := make([]int, 0)

	for k := range m {
		visited = append(visited, k)
		if k == 5 {
			m[100] = 1000
			fmt.Println("添加新键值对 100:1000")
		}
		delete(m, k)
		fmt.Printf("当前迭代到 k=%d, 删除后的 %v\n", k, m)
	}

	// Go 的设计决定：保持 Map 迭代的性能和简单性
	//因此我们无法保证在迭代过程中，新增的键值对会被当前的迭代遍历到
	fmt.Println("访问过的键:", visited)
}
```

输出：

```shell
开始迭代前: map[0:10 1:11 2:12 3:13 4:14 5:15 6:16 7:17 8:18 9:19]
添加新键值对 100:1000
当前迭代到 k=5, 删除后的 map[0:10 1:11 2:12 3:13 4:14 6:16 7:17 8:18 9:19 100:1000]
当前迭代到 k=7, 删除后的 map[0:10 1:11 2:12 3:13 4:14 6:16 8:18 9:19 100:1000]
当前迭代到 k=0, 删除后的 map[1:11 2:12 3:13 4:14 6:16 8:18 9:19 100:1000]
当前迭代到 k=3, 删除后的 map[1:11 2:12 4:14 6:16 8:18 9:19 100:1000]
当前迭代到 k=4, 删除后的 map[1:11 2:12 6:16 8:18 9:19 100:1000]
当前迭代到 k=8, 删除后的 map[1:11 2:12 6:16 9:19 100:1000]
当前迭代到 k=9, 删除后的 map[1:11 2:12 6:16 100:1000]
当前迭代到 k=100, 删除后的 map[1:11 2:12 6:16]
当前迭代到 k=1, 删除后的 map[2:12 6:16]
当前迭代到 k=2, 删除后的 map[6:16]
当前迭代到 k=6, 删除后的 map[]
访问过的键: [5 7 0 3 4 8 9 100 1 2 6]
```

运行时会对字典并发操作做出检测。如果某个任务正在对字典进行写操作，应该避免其他任务对该字典执行并发操作（读、写、删除）。

```go
// 目前我使用的 Go 1.22 版本，map 的并发写操作并不会 panic
// Go 1.19 之前的版本中，并发读写 map 会 panic
// 但这并不意味着它是安全的，Go 1.22 版本，通过 go test -race 仍可以检测到数据竞争
func TestConcurrentWrite(t *testing.T) {
	m := make(map[string]int)

	go func() {
		for {
			// 写操作，操作同一个键 "a"
			m["a"] += 1
			fmt.Println("写操作：", m["a"])
		}
	}()

	go func() {
		for {
			// 读操作，也操作键 "a"
			v := m["a"]
			fmt.Println("读操作：", v)
		}
	}()

	time.Sleep(time.Second) // 给并发操作一些时间
}
```

输出：

```shell
# Go 1.22 版本，无异常
# 通过 go test -race 检测到数据竞争
--- FAIL: TestConcurrentWrite (1.00s)
    testing.go:1398: race detected during execution of test

# Go 1.19 版本，直接 panic
panic: runtime error: concurrent map writes
```
