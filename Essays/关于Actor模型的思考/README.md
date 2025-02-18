# Actor 模型是什么？

# Actor 模型解决了什么问题？

## 保证消息的有序性
假设玩家收到两个客户端请求：
- 购买武器，处理时长 100 ms；
- 将购买的武器装备到角色身上，处理时长 10 ms。
在 Actor 模型中，会严格顺序处理，先处理购买武器，再处理装备武器。而使用网关投递消息的情况下，网关只负责投递消息，并不保证处理顺序，这会出现消息乱序和数据竞争，接着刚才的例子，我们分析来看为什么网关会引起这些问题，如下。

**消息乱序**

- 实际发送顺序：购买武器 -> 装备武器；
- 可能的处理顺序：装备武器 -> 购买武器。
- 结果：装备了不存在的武器。从玩家的视角，这很奇怪，我明明购买了武器，为什么会提示我武器不存在呢？然后过一会儿，武器又出现在背包中，又可以装备了。
在实际游戏业务中，客户端有 UI 交互限制，这两个操作存在间隔是必然的，达到了秒级，但墨菲定律告诉我们，可能出问题的地方终将出问题，提前做好防护，总是明智的，就像我不会假设数据库响应永远很快，每个开发人员的水平都是优秀的。

**数据竞争**

背包有物品数量的属性。
- 玩家装备武器时，物品数量减少（武器到玩家身上了）
- 玩家购买武器时，物品数量增加
这种情况，多个操作都在修改物品数量，就可能出现计数错误，通常，我们的解决方案是加锁，但“锁”自身也有性能损耗，性能测试代码与结果如下：

```go
func BenchmarkMutexLoss(b *testing.B) {
    // 准备测试数据
    counter := 0

    // 重置计时器
    b.ResetTimer()

    // 测试无锁的情况
    b.Run("NoLock", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            counter++
        }
    })

    // 重置计数器
    counter = 0

    // 测试有锁的情况
    mutex := &sync.Mutex{}
    b.Run("WithLock", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            mutex.Lock()
            counter++
            mutex.Unlock()
        }
    })
    
    // 执行 go test -bench . -benchmem 结果如下。
    // BenchmarkMutexLoss/NoLock-12            804751552                1.369 ns/op
    // BenchmarkMutexLoss/WithLock-12          93302852                13.23 ns/op
}
```

有锁版本（13.23 ns） / 无锁版本（1.369 ns） ≈ 9.66，这意味着加锁操作大约比无锁操作慢了 10 倍。这就引出了 Actor 模型要解决的第二个问题，无锁编程。

## 避免数据竞争
我们接着用上面背包的例子，解释为什么 Actor 能避免数据竞争。

- 传统加锁处理方式
```go
type BackpackSystem struct {
    mu       sync.Mutex
    backpack *Backpack
}

func (s *BackpackSystem) HandleBuyWeapon(weaponId int32) {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    s.backpack.Used++
    s.backpack.Slots = append(s.backpack.Slots, newWeapon)
}
```

- Actor 方式
```go
type BackpackActor struct {
    mailbox  chan Message      // 消息队列
    backpack *Backpack         // 背包数据
}

func (a *BackpackActor) dispatch() {
    // 单线程处理消息
    for msg := range a.mailbox {
        switch msg.Type {
        case BuyWeapon:
            // 不需要加锁，因为消息是顺序处理的
            a.backpack.Used++
            a.backpack.Slots = append(a.backpack.Slots, newWeapon)
            
        case EquipWeapon:
            // 同样不需要加锁
            a.backpack.Used--
            // 移除背包物品...
        }
    }
}

// 外部调用只能通过发消息
func (a *BackpackActor) BuyWeapon(weaponId int32) {
    a.mailbox <- Message{
        Type: BuyWeapon,
        Data: weaponId,
    }
}
```

我们可以看到，修改数据，并不是直接访问数据，而是通过向 mailbox 发送消息，Actor 通过消息传递替代共享内存！这不正是 Go 的哲学之一：**通过消息共享内存**吗？
Actor 推崇的是**数据完全被封装在 Actor 内**，每个 Actor 的数据具备隔离性，请留意，外部调用只能通过发消息，**Actor 内部可以直接调用方法**，这里的内部可以理解为 Actor 中的逻辑处理过程，下面是伪代码解释这个点。

```go
// 玩家自己修改背包
func (a *BackpackActor) EquipWeapon(weaponId int32) {
    // 本地方法调用，直接修改
    a.removeWeaponFromBackpack(weaponId)
    a.equipWeapon(weaponId)
}

// 外部请求修改背包
func (a *BackpackActor) HandleMessage(msg Message) {
    a.mailbox <- msg  // 通过消息队列处理
}
```

## 小结
- 降低开发人员心智负担

# Actor 模型如何实现？
