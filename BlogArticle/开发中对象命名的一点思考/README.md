链接：https://www.cnblogs.com/CareySon/p/18711135

## 个人打分
⭐️⭐️⭐️⭐️

## 阅读时间
2025/08/29 - 2025/08/29

## 读书心得

### 命名决定了代码的“第一印象分”

命名需要强调主体角色或“主观能动性”，弱化“被动执行”。

例如：**FoodMaker** 和 **Chef**：
- **FoodMaker**：让调用者详细描述每个步骤
- **Chef**：只需告诉它我要什么

```go
// FoodMaker 更像是“做饭机器”，调用者要自己控制过程
type FoodMaker struct{}

func (f *FoodMaker) AddIngredients(ingredients []string) {
	fmt.Println("添加食材:", ingredients)
}

func (f *FoodMaker) Cook() {
	fmt.Println("开始烹饪...")
}

func (f *FoodMaker) Serve() {
	fmt.Println("上菜")
}

func main() {
	// 调用方必须自己关心每一步
	maker := &FoodMaker{}
	maker.AddIngredients([]string{"鸡肉", "花生", "辣椒"})
	maker.Cook()
	maker.Serve()
}
```

```go
// Chef 更像是“专业的厨师”，调用者只需表达需求
type Chef struct{}

func (c *Chef) CookDish(dish string) {
	switch dish {
	case "宫保鸡丁":
		fmt.Println("准备食材: 鸡肉、花生、辣椒")
		fmt.Println("大火快炒...")
		fmt.Println("宫保鸡丁完成！")
	case "西红柿炒蛋":
		fmt.Println("准备食材: 西红柿、鸡蛋")
		fmt.Println("翻炒均匀...")
		fmt.Println("西红柿炒蛋完成！")
	default:
		fmt.Println("不会做这个菜，但我会临场发挥...")
	}
}

func main() {
	// 调用方只关心“想吃什么”，不关心细节
	chef := &Chef{}
	chef.CookDish("宫保鸡丁")
}
```

### 避免“上帝类”

文章批评了 RestaurantService 把烹饪、配送、排班、财务等职责混合在一起，变成“万能业务类”的设计弊端。

对应改进方案是将其拆解为多个专职类：

- Kitchen: 专注烹饪逻辑

- Delivery: 专注配送流程

- StaffScheduling: 专注人员排班

- Billing: 专注财务结算

应该遵守“单一职责”，并让命名直接反映职责，是确保系统结构清晰与可维护性的关键。

#### 错误示例
```go
type QuestService struct{}

func (s *QuestService) AcceptQuest(playerID, questID int) {
	fmt.Printf("玩家 %d 接受任务 %d\n", playerID, questID)
}

func (s *QuestService) CompleteQuest(playerID, questID int) {
	fmt.Printf("玩家 %d 完成任务 %d\n", playerID, questID)
}

func (s *QuestService) RewardQuest(playerID, questID int) {
	fmt.Printf("玩家 %d 领取任务 %d 的奖励\n", playerID, questID)
}

func (s *QuestService) CheckQuestCondition(playerID, questID int) bool {
	fmt.Printf("检查玩家 %d 是否满足任务 %d 条件\n", playerID, questID)
	return true
}

func (s *QuestService) SyncQuestToClient(playerID int) {
	fmt.Printf("同步玩家 %d 的任务数据到客户端\n", playerID)
}

func main() {
	qs := &QuestService{}
	qs.AcceptQuest(1, 1001)
	qs.CompleteQuest(1, 1001)
	qs.RewardQuest(1, 1001)
	qs.SyncQuestToClient(1)
}
```

#### 改进示例
```go

// 一个高层的协调器（Facade），调用各个职责类
type QuestManager struct {
	repo      *QuestRepository
	validator *QuestValidator
	rewarder  *QuestRewarder
	notifier  *QuestNotifier
}

// 任务数据存取
type QuestRepository struct{}

func (r *QuestRepository) Save(playerID, questID int) {
	fmt.Printf("保存玩家 %d 的任务 %d 到数据库\n", playerID, questID)
}

// 任务条件校验
type QuestValidator struct{}

func (v *QuestValidator) CheckCondition(playerID, questID int) bool {
	fmt.Printf("检查玩家 %d 是否满足任务 %d 条件\n", playerID, questID)
	return true
}

// 任务奖励处理
type QuestRewarder struct{}

func (r *QuestRewarder) GiveReward(playerID, questID int) {
	fmt.Printf("给玩家 %d 发放任务 %d 的奖励\n", playerID, questID)
}

// 网络同步
type QuestNotifier struct{}

func (n *QuestNotifier) SyncToClient(playerID int) {
	fmt.Printf("同步玩家 %d 的任务数据到客户端\n", playerID)
}

func (m *QuestManager) CompleteQuest(playerID, questID int) {
	if !m.validator.CheckCondition(playerID, questID) {
		fmt.Println("条件不满足，无法完成任务")
		return
	}
	m.repo.Save(playerID, questID)
	m.rewarder.GiveReward(playerID, questID)
	m.notifier.SyncToClient(playerID)
}

func main() {
	manager := &QuestManager{
		repo:      &QuestRepository{},
		validator: &QuestValidator{},
		rewarder:  &QuestRewarder{},
		notifier:  &QuestNotifier{},
	}

	manager.CompleteQuest(1, 1001)
}
```

### 命名中体现自适应能力

以食材库存变化为例，比较了两种设计：

- CookingProcess（过程导向）：无论食材是否缺货，都按固定流程处理——调用方被动承担判断职责。

- Chef（智能主体）：面对“花生缺货”这种变化，Chef 会在内部自动选择替代食材，不需调用方改动调用逻辑。