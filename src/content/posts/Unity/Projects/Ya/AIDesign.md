---
title: 敌人AI架构设计
published: 2026-07-02
pinned: false
description: "Github：Ya项目的AI设计"
image: ""
tags: ["Unity", "AI", "行为树", "Ya"]
category: 架构
draft: false
---

# 🧠 敌人 AI 系统 —— 架构设计文档

> 项目地址：[Xitree/Ya](https://github.com/Xitree/Ya)

## 一、系统定位
```plain
┌────────────────────────────────────────────────────────────────┐
│                      MyCombat 角色架构总览                       │
│                                                                │
│  ┌──────────────┐         ┌──────────────┐                     │
│  │    Player     │         │    Enemy     │                     │
│  │              │         │              │                     │
│  │  输入来源：   │         │  输入来源：   │                     │
│  │  PlayerInput │         │  BehaviorTree│                     │
│  │              │         │  (AI Brain)  │                     │
│  │      ↓       │         │      ↓       │                     │
│  │  MetaState   │         │  MetaState   │  ← 可复用           │
│  │  Machine     │         │  Machine     │                     │
│  │      ↓       │         │      ↓       │                     │
│  │  Sub State   │         │  Sub State   │  ← 可复用           │
│  │  Machine+CCT │         │  Machine+CCT │                     │
│  │      ↓       │         │      ↓       │                     │
│  │  Character   │         │  Character   │  ← 共享基类         │
│  │  MoveCtrl    │         │  MoveCtrl    │                     │
│  └──────────────┘         └──────────────┘                     │
└────────────────────────────────────────────────────────────────┘
```

**核心思想**：玩家的输入来自手柄/键盘，敌人的输入来自行为树。**状态机以下的层全部复用**。

---

## 二、行为树框架设计
### 2.1 节点类型体系
```plain
BTNode (abstract)                          ← 所有节点的基类
├── BTComposite (abstract)                 ← 组合节点（有子节点列表）
│   ├── BTSelector                         ← 选择节点：从左到右，第一个成功即返回 Success
│   ├── BTSequence                         ← 序列节点：从左到右，全部成功才返回 Success
│   └── BTParallel                         ← 并行节点：同时执行所有子节点
├── BTDecorator (abstract)                 ← 装饰节点（包装单个子节点）
│   ├── BTInverter                         ← 取反
│   ├── BTRepeater                         ← 重复执行 N 次
│   ├── BTCooldown                         ← 冷却期间返回 Failure
│   └── BTConditionalGuard                 ← 前置条件检查
└── BTLeaf (abstract)                      ← 叶节点（真正做事的）
    ├── BTCondition (abstract)             ← 条件检查叶节点
    │   ├── IsPlayerInRange                ← 玩家在范围内？
    │   ├── IsHPBelow                      ← 血量低于阈值？
    │   ├── IsPlayerVisible                ← 能看到玩家？
    │   └── HasCooldownReady               ← 技能 CD 好了？
    └── BTAction (abstract)                ← 行为执行叶节点
        ├── ChangeStateAction              ← ★核心：切换敌人状态机状态
        ├── MoveToTargetAction             ← 向目标移动
        ├── SetBlackboardValueAction       ← 写入黑板数据
        ├── WaitAction                     ← 等待一段时间
        └── PlayAnimationAction            ← 直接播放动画
```

### 2.2 节点返回值
```csharp
public enum BTStatus
{
    Success,    // 本节点完成
    Failure,    // 本节点失败
    Running,    // 本节点还在执行中（下帧继续）
}
```

### 2.3 Blackboard（共享数据黑板）
行为树节点之间通过 **Blackboard** 共享数据，而不是直接互相引用：

```plain
Blackboard
├── target        : Transform     ← 当前目标（玩家）
├── targetDistance : float         ← 与目标的距离（每帧更新）
├── hp            : float         ← 当前血量
├── hpRatio       : float         ← 血量百分比
├── isInCombat    : bool          ← 是否进入战斗
├── lastAttackTime: float         ← 上次攻击时间
├── patrolPoints  : Vector3[]     ← 巡逻路径点
├── currentPatrolIndex : int      ← 当前巡逻点索引
└── ... 自定义扩展
```

**Blackboard 的设计**参考 Unreal 的 BT 黑板——用字符串 Key 存取，运行时类型安全：

```csharp
// 使用方式
blackboard.Set("targetDistance", 5.2f);
float dist = blackboard.Get<float>("targetDistance");
```

### 2.4 Tick 机制
```plain
每帧 Enemy.Update()
    ↓
BehaviorTree.Tick()
    ↓
从 Root 节点开始向下遍历
    ↓
Selector 从左到右找到第一个 Success/Running 的分支
    ↓
如果当前执行的分支和上一帧不同 → 触发上一个分支的 Abort
    ↓
最终 Action 节点输出决策：
    - ChangeStateAction → 切换到 EStateType.XXX
    - MoveToTargetAction → 设置寻路目标
```

---

## 三、敌人系统类图
### 3.1 继承结构
```plain
CharacterMoveControllerBase (已有)
├── Player (已有)
└── Enemy (新建)                           ← 敌人 MonoBehaviour
    ├── enemySO: EnemySO                   ← 敌人配置数据
    ├── behaviorTree: BehaviorTree          ← AI 大脑
    ├── blackboard: Blackboard             ← 共享数据
    ├── enemyMetaStateMachine              ← 顶层 Meta 状态机
    ├── sensorSystem: EnemySensor          ← 感知系统（视野、听觉）
    └── allRegisteredStatuses              ← CCT 配置注册表
```

### 3.2 敌人状态机结构
```plain
Enemy
└── EnemyMetaStateMachine                  ← Meta 层
    ├── EnemyLocomotionMetaState           ← 移动阶段
    │   └── EnemyMovementStateMachine      ← 子状态机
    │       ├── EnemyIdleState
    │       ├── EnemyPatrolState
    │       ├── EnemyChaseState
    │       └── EnemyReturnState           ← 脱战回巡逻点
    └── EnemyCombatMetaState               ← 战斗阶段
        └── EnemyCombatStateMachine        ← 子状态机
            ├── EnemyLightATKState
            ├── EnemyHeavyATKState
            ├── EnemyDefendState
            ├── EnemyDodgeState
            └── EnemyStaggerState          ← 受击硬直
```

### 3.3 上下文系统扩展
```plain
StateTransitionContext (已有基类)
├── PlayerTransitionContext (已有)
│   ├── LocomotionTransitionContext
│   └── CombatTransitionContext
└── EnemyTransitionContext (新建)            ← 敌人公共上下文
    ├── blackboard: Blackboard             ← 引用黑板
    ├── targetDistance: float               ← 从黑板读
    ├── hpRatio: float                     ← 从黑板读
    ├── EnemyLocomotionTransitionContext   ← 敌人移动上下文
    └── EnemyCombatTransitionContext       ← 敌人战斗上下文
```

---

## 四、核心流程
### 4.1 每帧执行流程
```plain
Enemy.Update()
│
├── 1. base.Update()                        ← 地面检测、重力、垂直速度
│
├── 2. sensorSystem.UpdateSensors()         ← 感知：更新目标距离、视野检测
│      └── 结果写入 Blackboard
│
├── 3. behaviorTree.Tick(blackboard)        ← BT 决策
│      └── 遍历节点树
│      └── Action 节点可能调用：
│          ├── enemyMetaStateMachine.RequestTransition(...)  ← 跨 Meta 切换
│          └── currentStateMachine.ChangeState(...)          ← Meta 内切换
│
├── 4. enemyMetaStateMachine.HandleInput()  ← 状态机处理（和玩家一致）
│
└── 5. enemyMetaStateMachine.Update()       ← 状态机更新（CCT检查 + 动画驱动）
```

### 4.2 BT 与状态机的桥接
**关键设计**：BT 只做"大决策"（该巡逻还是追击还是攻击），状态内部的"小转换"（攻击中的连招、受击反应）仍交给 CCT 配置。

```plain
┌──────────────────────────────────┐
│  行为树 (Brain)                   │
│                                  │
│  Selector                        │
│  ├── [HP低] → DefendAction       │──→ ChangeState(Defend)
│  ├── [近距离] → AttackAction     │──→ RequestTransition(Combat, LightATK)
│  ├── [中距离] → ChaseAction      │──→ ChangeState(Chase)
│  └── PatrolAction                │──→ ChangeState(Patrol)
│                                  │
│  BT 输出的是 "该进入什么状态"      │
└──────────┬───────────────────────┘
           │
           ↓ 切换状态
┌──────────────────────────────────┐
│  状态机 (Body)                    │
│                                  │
│  EnemyLightATKState.Enter()      │
│    ├── 播放攻击动画               │
│    ├── 驱动根运动                 │
│    └── CCT 检查子转换             │ ← 比如连击窗口内按条件接下一招
│                                  │
│  EnemyLightATKState.Update()     │
│    ├── UpdatePlayerParam()       │
│    └── CheckTransitions()        │ ← CCT 配置的"攻击结束 → Idle"
└──────────────────────────────────┘
```

### 4.3 BT 打断机制
当 BT 决策发生变化时（比如正在 Chase，但 HP 突然变低），需要打断当前行为：

```plain
帧 N:  BT → ChaseAction (Running) → 状态机在 ChaseState
帧 N+1: HP 低了 → BT → DefendAction (Success) → ChaseAction 被 Abort
         └── ChaseAction.OnAbort() 
              └── 清理资源、通知状态机
         └── DefendAction.Execute()
              └── 切换到 DefendState
```

---

## 五、模块目录结构
```plain
Scripts/
├── Base/
│   └── CharacterMoveControllerBase.cs       ← (已有) 共享
├── Core/
│   ├── BehaviorTree/                         ← ★ 新建：BT 框架
│   │   ├── BTNode.cs                         ← 节点基类
│   │   ├── BTStatus.cs                       ← 返回值枚举
│   │   ├── BehaviorTree.cs                   ← BT 运行器
│   │   ├── Blackboard.cs                     ← 共享黑板
│   │   ├── Composite/
│   │   │   ├── BTSelector.cs
│   │   │   ├── BTSequence.cs
│   │   │   └── BTParallel.cs
│   │   ├── Decorator/
│   │   │   ├── BTInverter.cs
│   │   │   ├── BTRepeater.cs
│   │   │   ├── BTCooldown.cs
│   │   │   └── BTConditionalGuard.cs
│   │   └── Leaf/
│   │       ├── BTCondition.cs                ← 条件叶节点基类
│   │       └── BTAction.cs                   ← 行为叶节点基类
│   ├── StateMachine/                         ← (已有) 共享
│   └── ...
├── Enemy/                                    ← ★ 新建：敌人系统
│   ├── Enemy.cs                              ← 敌人 MonoBehaviour
│   ├── EnemySensor.cs                        ← 感知系统
│   ├── Data/
│   │   ├── ScriptableObject/
│   │   │   └── EnemySO.cs                    ← 敌人配置
│   │   └── ReusableData/
│   │       ├── EnemyMovementReusableData.cs
│   │       └── EnemyCombatReusableData.cs
│   ├── StateMachine/
│   │   ├── EnemyMetaStateMachine.cs
│   │   ├── EnemyMovementStateMachine.cs
│   │   ├── EnemyCombatStateMachine.cs
│   │   ├── Meta/
│   │   │   ├── EnemyLocomotionMetaState.cs
│   │   │   └── EnemyCombatMetaState.cs
│   │   ├── States/
│   │   │   ├── Movement/
│   │   │   │   ├── Base/
│   │   │   │   │   └── EnemyMovementState.cs
│   │   │   │   ├── EnemyIdleState.cs
│   │   │   │   ├── EnemyPatrolState.cs
│   │   │   │   └── EnemyChaseState.cs
│   │   │   └── Combat/
│   │   │       ├── Base/
│   │   │       │   └── EnemyCombatState.cs
│   │   │       ├── EnemyLightATKState.cs
│   │   │       └── EnemyStaggerState.cs
│   │   └── CCT/
│   │       └── Conditions/
│   │           └── Context/
│   │               └── EnemyTransitionContext.cs
│   └── BT/                                   ← ★ 敌人专用 BT 节点
│       ├── Conditions/
│       │   ├── IsPlayerInRange.cs
│       │   ├── IsHPBelow.cs
│       │   └── IsPlayerVisible.cs
│       ├── Actions/
│       │   ├── ChangeStateAction.cs           ← 关键桥接节点
│       │   ├── ChaseTargetAction.cs
│       │   ├── PatrolAction.cs
│       │   └── WaitRandomAction.cs
│       └── Trees/
│           ├── BasicEnemyBT.cs                ← 小怪行为树（代码构建）
│           └── BossEnemyBT.cs                 ← Boss 行为树
└── Player/                                    ← (已有)
```

---

## 六、可复用部分清单
| 模块 | 玩家 | 敌人 | 复用方式 |
| --- | --- | --- | --- |
| `CharacterMoveControllerBase` | ✅ | ✅ | 直接继承 |
| `StateMachine` | ✅ | ✅ | 直接使用 |
| `MetaStateMachine` | ✅ | ✅ | 直接继承 |
| `IState` / `IMetaState` | ✅ | ✅ | 直接实现 |
| `PlayerStateBase` | ✅ | 🔄 | 抽象出 `CharacterStateBase`，玩家/敌人各自继承 |
| `StateTransitionContext` | ✅ | ✅ | 敌人写 `EnemyTransitionContext` |
| `TransitionConditionSO` | ✅ | ✅ | SO 通用，条件可跨角色共享 |
| `StateTransitionRuleSO` | ✅ | ✅ | 直接复用 |
| `StateTransitionConfigSO` | ✅ | ✅ | 直接复用 |
| `StateTransitionRegistrySO` | ✅ | ✅ | 每种敌人有自己的 Registry 实例 |
| `TimerManager` | ✅ | ✅ | 单例共享 |
| `ObjectPool` / `PoolManager` | ✅ | ✅ | 单例共享 |
| `AnimStateHash` / `AnimatorID` | ✅ | ✅ | 敌人扩展自己的动画 Hash |


---# 🧠 敌人 AI 系统 —— 架构设计文档
## 一、系统定位
```plain
┌────────────────────────────────────────────────────────────────┐
│                      MyCombat 角色架构总览                       │
│                                                                │
│  ┌──────────────┐         ┌──────────────┐                     │
│  │    Player     │         │    Enemy     │                     │
│  │              │         │              │                     │
│  │  输入来源：   │         │  输入来源：   │                     │
│  │  PlayerInput │         │  BehaviorTree│                     │
│  │              │         │  (AI Brain)  │                     │
│  │      ↓       │         │      ↓       │                     │
│  │  MetaState   │         │  MetaState   │  ← 可复用           │
│  │  Machine     │         │  Machine     │                     │
│  │      ↓       │         │      ↓       │                     │
│  │  Sub State   │         │  Sub State   │  ← 可复用           │
│  │  Machine+CCT │         │  Machine+CCT │                     │
│  │      ↓       │         │      ↓       │                     │
│  │  Character   │         │  Character   │  ← 共享基类         │
│  │  MoveCtrl    │         │  MoveCtrl    │                     │
│  └──────────────┘         └──────────────┘                     │
└────────────────────────────────────────────────────────────────┘
```

**核心思想**：玩家的输入来自手柄/键盘，敌人的输入来自行为树。**状态机以下的层全部复用**。

---

## 二、行为树框架设计
### 2.1 节点类型体系
```plain
BTNode (abstract)                          ← 所有节点的基类
├── BTComposite (abstract)                 ← 组合节点（有子节点列表）
│   ├── BTSelector                         ← 选择节点：从左到右，第一个成功即返回 Success
│   ├── BTSequence                         ← 序列节点：从左到右，全部成功才返回 Success
│   └── BTParallel                         ← 并行节点：同时执行所有子节点
├── BTDecorator (abstract)                 ← 装饰节点（包装单个子节点）
│   ├── BTInverter                         ← 取反
│   ├── BTRepeater                         ← 重复执行 N 次
│   ├── BTCooldown                         ← 冷却期间返回 Failure
│   └── BTConditionalGuard                 ← 前置条件检查
└── BTLeaf (abstract)                      ← 叶节点（真正做事的）
    ├── BTCondition (abstract)             ← 条件检查叶节点
    │   ├── IsPlayerInRange                ← 玩家在范围内？
    │   ├── IsHPBelow                      ← 血量低于阈值？
    │   ├── IsPlayerVisible                ← 能看到玩家？
    │   └── HasCooldownReady               ← 技能 CD 好了？
    └── BTAction (abstract)                ← 行为执行叶节点
        ├── ChangeStateAction              ← ★核心：切换敌人状态机状态
        ├── MoveToTargetAction             ← 向目标移动
        ├── SetBlackboardValueAction       ← 写入黑板数据
        ├── WaitAction                     ← 等待一段时间
        └── PlayAnimationAction            ← 直接播放动画
```

### 2.2 节点返回值
```csharp
public enum BTStatus
{
    Success,    // 本节点完成
    Failure,    // 本节点失败
    Running,    // 本节点还在执行中（下帧继续）
}
```

### 2.3 Blackboard（共享数据黑板）
行为树节点之间通过 **Blackboard** 共享数据，而不是直接互相引用：

```plain
Blackboard
├── target        : Transform     ← 当前目标（玩家）
├── targetDistance : float         ← 与目标的距离（每帧更新）
├── hp            : float         ← 当前血量
├── hpRatio       : float         ← 血量百分比
├── isInCombat    : bool          ← 是否进入战斗
├── lastAttackTime: float         ← 上次攻击时间
├── patrolPoints  : Vector3[]     ← 巡逻路径点
├── currentPatrolIndex : int      ← 当前巡逻点索引
└── ... 自定义扩展
```

**Blackboard 的设计**参考 Unreal 的 BT 黑板——用字符串 Key 存取，运行时类型安全：

```csharp
// 使用方式
blackboard.Set("targetDistance", 5.2f);
float dist = blackboard.Get<float>("targetDistance");
```

### 2.4 Tick 机制
```plain
每帧 Enemy.Update()
    ↓
BehaviorTree.Tick()
    ↓
从 Root 节点开始向下遍历
    ↓
Selector 从左到右找到第一个 Success/Running 的分支
    ↓
如果当前执行的分支和上一帧不同 → 触发上一个分支的 Abort
    ↓
最终 Action 节点输出决策：
    - ChangeStateAction → 切换到 EStateType.XXX
    - MoveToTargetAction → 设置寻路目标
```

---

## 三、敌人系统类图
### 3.1 继承结构
```plain
CharacterMoveControllerBase (已有)
├── Player (已有)
└── Enemy (新建)                           ← 敌人 MonoBehaviour
    ├── enemySO: EnemySO                   ← 敌人配置数据
    ├── behaviorTree: BehaviorTree          ← AI 大脑
    ├── blackboard: Blackboard             ← 共享数据
    ├── enemyMetaStateMachine              ← 顶层 Meta 状态机
    ├── sensorSystem: EnemySensor          ← 感知系统（视野、听觉）
    └── allRegisteredStatuses              ← CCT 配置注册表
```

### 3.2 敌人状态机结构
```plain
Enemy
└── EnemyMetaStateMachine                  ← Meta 层
    ├── EnemyLocomotionMetaState           ← 移动阶段
    │   └── EnemyMovementStateMachine      ← 子状态机
    │       ├── EnemyIdleState
    │       ├── EnemyPatrolState
    │       ├── EnemyChaseState
    │       └── EnemyReturnState           ← 脱战回巡逻点
    └── EnemyCombatMetaState               ← 战斗阶段
        └── EnemyCombatStateMachine        ← 子状态机
            ├── EnemyLightATKState
            ├── EnemyHeavyATKState
            ├── EnemyDefendState
            ├── EnemyDodgeState
            └── EnemyStaggerState          ← 受击硬直
```

### 3.3 上下文系统扩展
```plain
StateTransitionContext (已有基类)
├── PlayerTransitionContext (已有)
│   ├── LocomotionTransitionContext
│   └── CombatTransitionContext
└── EnemyTransitionContext (新建)            ← 敌人公共上下文
    ├── blackboard: Blackboard             ← 引用黑板
    ├── targetDistance: float               ← 从黑板读
    ├── hpRatio: float                     ← 从黑板读
    ├── EnemyLocomotionTransitionContext   ← 敌人移动上下文
    └── EnemyCombatTransitionContext       ← 敌人战斗上下文
```

---

## 四、核心流程
### 4.1 每帧执行流程
```plain
Enemy.Update()
│
├── 1. base.Update()                        ← 地面检测、重力、垂直速度
│
├── 2. sensorSystem.UpdateSensors()         ← 感知：更新目标距离、视野检测
│      └── 结果写入 Blackboard
│
├── 3. behaviorTree.Tick(blackboard)        ← BT 决策
│      └── 遍历节点树
│      └── Action 节点可能调用：
│          ├── enemyMetaStateMachine.RequestTransition(...)  ← 跨 Meta 切换
│          └── currentStateMachine.ChangeState(...)          ← Meta 内切换
│
├── 4. enemyMetaStateMachine.HandleInput()  ← 状态机处理（和玩家一致）
│
└── 5. enemyMetaStateMachine.Update()       ← 状态机更新（CCT检查 + 动画驱动）
```

### 4.2 BT 与状态机的桥接
**关键设计**：BT 只做"大决策"（该巡逻还是追击还是攻击），状态内部的"小转换"（攻击中的连招、受击反应）仍交给 CCT 配置。

```plain
┌──────────────────────────────────┐
│  行为树 (Brain)                   │
│                                  │
│  Selector                        │
│  ├── [HP低] → DefendAction       │──→ ChangeState(Defend)
│  ├── [近距离] → AttackAction     │──→ RequestTransition(Combat, LightATK)
│  ├── [中距离] → ChaseAction      │──→ ChangeState(Chase)
│  └── PatrolAction                │──→ ChangeState(Patrol)
│                                  │
│  BT 输出的是 "该进入什么状态"      │
└──────────┬───────────────────────┘
           │
           ↓ 切换状态
┌──────────────────────────────────┐
│  状态机 (Body)                    │
│                                  │
│  EnemyLightATKState.Enter()      │
│    ├── 播放攻击动画               │
│    ├── 驱动根运动                 │
│    └── CCT 检查子转换             │ ← 比如连击窗口内按条件接下一招
│                                  │
│  EnemyLightATKState.Update()     │
│    ├── UpdatePlayerParam()       │
│    └── CheckTransitions()        │ ← CCT 配置的"攻击结束 → Idle"
└──────────────────────────────────┘
```

### 4.3 BT 打断机制
当 BT 决策发生变化时（比如正在 Chase，但 HP 突然变低），需要打断当前行为：

```plain
帧 N:  BT → ChaseAction (Running) → 状态机在 ChaseState
帧 N+1: HP 低了 → BT → DefendAction (Success) → ChaseAction 被 Abort
         └── ChaseAction.OnAbort() 
              └── 清理资源、通知状态机
         └── DefendAction.Execute()
              └── 切换到 DefendState
```

---

## 五、模块目录结构
```plain
Scripts/
├── Base/
│   └── CharacterMoveControllerBase.cs       ← (已有) 共享
├── Core/
│   ├── BehaviorTree/                         ← ★ 新建：BT 框架
│   │   ├── BTNode.cs                         ← 节点基类
│   │   ├── BTStatus.cs                       ← 返回值枚举
│   │   ├── BehaviorTree.cs                   ← BT 运行器
│   │   ├── Blackboard.cs                     ← 共享黑板
│   │   ├── Composite/
│   │   │   ├── BTSelector.cs
│   │   │   ├── BTSequence.cs
│   │   │   └── BTParallel.cs
│   │   ├── Decorator/
│   │   │   ├── BTInverter.cs
│   │   │   ├── BTRepeater.cs
│   │   │   ├── BTCooldown.cs
│   │   │   └── BTConditionalGuard.cs
│   │   └── Leaf/
│   │       ├── BTCondition.cs                ← 条件叶节点基类
│   │       └── BTAction.cs                   ← 行为叶节点基类
│   ├── StateMachine/                         ← (已有) 共享
│   └── ...
├── Enemy/                                    ← ★ 新建：敌人系统
│   ├── Enemy.cs                              ← 敌人 MonoBehaviour
│   ├── EnemySensor.cs                        ← 感知系统
│   ├── Data/
│   │   ├── ScriptableObject/
│   │   │   └── EnemySO.cs                    ← 敌人配置
│   │   └── ReusableData/
│   │       ├── EnemyMovementReusableData.cs
│   │       └── EnemyCombatReusableData.cs
│   ├── StateMachine/
│   │   ├── EnemyMetaStateMachine.cs
│   │   ├── EnemyMovementStateMachine.cs
│   │   ├── EnemyCombatStateMachine.cs
│   │   ├── Meta/
│   │   │   ├── EnemyLocomotionMetaState.cs
│   │   │   └── EnemyCombatMetaState.cs
│   │   ├── States/
│   │   │   ├── Movement/
│   │   │   │   ├── Base/
│   │   │   │   │   └── EnemyMovementState.cs
│   │   │   │   ├── EnemyIdleState.cs
│   │   │   │   ├── EnemyPatrolState.cs
│   │   │   │   └── EnemyChaseState.cs
│   │   │   └── Combat/
│   │   │       ├── Base/
│   │   │       │   └── EnemyCombatState.cs
│   │   │       ├── EnemyLightATKState.cs
│   │   │       └── EnemyStaggerState.cs
│   │   └── CCT/
│   │       └── Conditions/
│   │           └── Context/
│   │               └── EnemyTransitionContext.cs
│   └── BT/                                   ← ★ 敌人专用 BT 节点
│       ├── Conditions/
│       │   ├── IsPlayerInRange.cs
│       │   ├── IsHPBelow.cs
│       │   └── IsPlayerVisible.cs
│       ├── Actions/
│       │   ├── ChangeStateAction.cs           ← 关键桥接节点
│       │   ├── ChaseTargetAction.cs
│       │   ├── PatrolAction.cs
│       │   └── WaitRandomAction.cs
│       └── Trees/
│           ├── BasicEnemyBT.cs                ← 小怪行为树（代码构建）
│           └── BossEnemyBT.cs                 ← Boss 行为树
└── Player/                                    ← (已有)
```

---

## 六、可复用部分清单
| 模块 | 玩家 | 敌人 | 复用方式 |
| --- | --- | --- | --- |
| `CharacterMoveControllerBase` | ✅ | ✅ | 直接继承 |
| `StateMachine` | ✅ | ✅ | 直接使用 |
| `MetaStateMachine` | ✅ | ✅ | 直接继承 |
| `IState` / `IMetaState` | ✅ | ✅ | 直接实现 |
| `PlayerStateBase` | ✅ | 🔄 | 抽象出 `CharacterStateBase`，玩家/敌人各自继承 |
| `StateTransitionContext` | ✅ | ✅ | 敌人写 `EnemyTransitionContext` |
| `TransitionConditionSO` | ✅ | ✅ | SO 通用，条件可跨角色共享 |
| `StateTransitionRuleSO` | ✅ | ✅ | 直接复用 |
| `StateTransitionConfigSO` | ✅ | ✅ | 直接复用 |
| `StateTransitionRegistrySO` | ✅ | ✅ | 每种敌人有自己的 Registry 实例 |
| `TimerManager` | ✅ | ✅ | 单例共享 |
| `ObjectPool` / `PoolManager` | ✅ | ✅ | 单例共享 |
| `AnimStateHash` / `AnimatorID` | ✅ | ✅ | 敌人扩展自己的动画 Hash |


---

AI部分脚本数量比较多，让AI帮我整理总结了下
