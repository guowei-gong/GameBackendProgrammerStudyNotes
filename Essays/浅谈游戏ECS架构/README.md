## ECS 架构与 OOP 架构有什么区别？

- 内存布局不同

```go
// OOP 架构
type GameObject struct {
    Position    Vector3
    Health      int
    Attack      int
    Speed       float32
    // 更多属性...
}

// 内存中的布局（示意）
gameObjects := []*GameObject{
    // object1: [ptr]->[pos|health|attack|speed|inventory...]
    // object2: [ptr]->[pos|health|attack|speed|inventory...]
    // object3: [ptr]->[pos|health|attack|speed|inventory...]
}

// ECS 架构
type PositionComponent []Vector3
type HealthComponent []int
type AttackComponent []int
type SpeedComponent []float32

// 内存中的布局（示意）
positions := []Vector3{pos1, pos2, pos3, pos4...}       // 连续内存
healths := []int{health1, health2, health3, health4...} // 连续内存
speeds := []float32{speed1, speed2, speed3, speed4...}  // 连续内存
```

- ECS 架构的内存布局是连续的，而 OOP 架构的内存布局是分散的。

gameObjects 我们很好理解，每个实体就是一个玩家，那 ECS 架构中如何区分玩家呢？

在 ECS 架构中，通过 E - Entity 实体，来区分不同的实体，组件数组通过这个实体中的 ID 来索引定位数据。

- 数据装载方式不同