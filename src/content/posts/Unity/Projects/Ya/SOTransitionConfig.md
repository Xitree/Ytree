---
title: SO 配置化转换表完整详解
published: 2026-07-02
pinned: false
description: "Ya 项目中基于 ScriptableObject 的状态转换配置表设计"
image: ""
tags: ["Unity", "ScriptableObject", "状态机", "Ya"]
category: 架构
draft: false
---

# SO 配置化转换表 —— 完整详解

> 项目地址：[Xitree/Ya](https://github.com/Xitree/Ya)

## 核心设计思想

```plain
把状态转换的"三要素"全做成 ScriptableObject：
1. 条件 (Condition)  → 什么时候转？
2. 规则 (Transition) → 从哪转到哪？播什么动画？
3. 总表 (Config)     → 一个状态的所有出口规则集合

最终效果：策划在 Inspector 里拖拖拽拽就能配置状态转换
程序员加新状态只需要写新的 Condition SO（如果现有条件不够用）
```

---

## 第一层：条件系统 (Condition)

### 条件基类

```csharp
// 所有转换条件的抽象基类
public abstract class TransitionConditionSO : ScriptableObject
{
    /// <summary>
    /// 评估条件是否满足
    /// </summary>
    /// <param name="context">状态机上下文，提供所有需要的引用</param>
    /// <returns>条件是否满足</returns>
    public abstract bool Evaluate(StateTransitionContext context);
}

/// <summary>
/// 状态机上下文：打包所有条件判断需要的引用，避免条件 SO 到处找依赖
/// </summary>
[System.Serializable]
public class StateTransitionContext
{
    public Animator Animator;
    public PlayerMovementReusableData ReusableData;
    public PlayerMovementStateMachine StateMachine;
    public Vector2 MoveInput;
    public bool IsSprintPressed;
    public bool IsJumpPressed;
    // ... 根据需要扩展

    /// <summary>
    /// 每帧刷新输入数据
    /// </summary>
    public void UpdateInput()
    {
        MoveInput = PlayerInputSystem.MainInstance.MoveInput;
        IsSprintPressed = PlayerInputSystem.MainInstance.IsSprintPressed;
        IsJumpPressed = PlayerInputSystem.MainInstance.IsJumpPressed;
    }
}
```

### 常用条件实现

```csharp
// ==================== 输入相关条件 ====================

/// <summary>
/// 检查是否有移动输入
/// </summary>
[CreateAssetMenu(menuName = "StateMachine/Conditions/HasMoveInput")]
public class HasMoveInputCondition : TransitionConditionSO
{
    [Tooltip("true = 要求有输入，false = 要求无输入")]
    public bool requireInput = true;

    public override bool Evaluate(StateTransitionContext context)
    {
        bool hasInput = context.MoveInput != Vector2.zero;
        return hasInput == requireInput;
    }
}

/// <summary>
/// 检查是否按下冲刺键
/// </summary>
[CreateAssetMenu(menuName = "StateMachine/Conditions/IsSprintPressed")]
public class IsSprintPressedCondition : TransitionConditionSO
{
    public bool requirePressed = true;

    public override bool Evaluate(StateTransitionContext context)
    {
        return context.IsSprintPressed == requirePressed;
    }
}

// ==================== 动画相关条件 ====================

/// <summary>
/// 检查当前动画状态是否匹配
/// </summary>
[CreateAssetMenu(menuName = "StateMachine/Conditions/AnimStateMatch")]
public class AnimStateMatchCondition : TransitionConditionSO
{
    [Tooltip("目标动画状态名")]
    public string animStateName;

    [Tooltip("Animator Layer 索引")]
    public int layerIndex = 0;

    private int _cachedHash = -1;

    public override bool Evaluate(StateTransitionContext context)
    {
        if (_cachedHash == -1)
            _cachedHash = Animator.StringToHash(animStateName);

        var stateInfo = context.Animator.GetCurrentAnimatorStateInfo(layerIndex);
        return stateInfo.shortNameHash == _cachedHash;
    }
}

/// <summary>
/// 检查当前动画播放进度（normalizedTime）
/// </summary>
[CreateAssetMenu(menuName = "StateMachine/Conditions/AnimProgressCheck")]
public class AnimProgressCondition : TransitionConditionSO
{
    public enum CompareType { LessThan, GreaterThan, LessOrEqual, GreaterOrEqual }

    [Tooltip("比较方式")]
    public CompareType compareType = CompareType.GreaterOrEqual;

    [Tooltip("目标进度值 (0~1)")]
    [Range(0f, 1f)]
    public float targetProgress = 0.9f;

    public int layerIndex = 0;

    public override bool Evaluate(StateTransitionContext context)
    {
        float normalizedTime = context.Animator.GetCurrentAnimatorStateInfo(layerIndex).normalizedTime;

        return compareType switch
        {
            CompareType.LessThan => normalizedTime < targetProgress,
            CompareType.GreaterThan => normalizedTime > targetProgress,
            CompareType.LessOrEqual => normalizedTime <= targetProgress,
            CompareType.GreaterOrEqual => normalizedTime >= targetProgress,
            _ => false
        };
    }
}

/// <summary>
/// 检查当前动画帧数
/// </summary>
[CreateAssetMenu(menuName = "StateMachine/Conditions/AnimFrameCheck")]
public class AnimFrameCondition : TransitionConditionSO
{
    public enum CompareType { LessOrEqual, GreaterOrEqual }

    public string animStateName;
    public CompareType compareType = CompareType.LessOrEqual;
    public int frameThreshold;
    public int layerIndex = 0;

    private int _cachedHash = -1;

    public override bool Evaluate(StateTransitionContext context)
    {
        if (_cachedHash == -1)
            _cachedHash = Animator.StringToHash(animStateName);

        var stateInfo = context.Animator.GetCurrentAnimatorStateInfo(layerIndex);
        if (stateInfo.shortNameHash != _cachedHash) return false;

        // 用 normalizedTime 换算帧数
        // 注意：这里需要你提供 clip 的总帧数，或者用你自己的 GetCurrentFrame 方法
        int currentFrame = Mathf.FloorToInt(stateInfo.normalizedTime * stateInfo.length * 30f);

        return compareType switch
        {
            CompareType.LessOrEqual => currentFrame <= frameThreshold,
            CompareType.GreaterOrEqual => currentFrame >= frameThreshold,
            _ => false
        };
    }
}

// ==================== 物理/状态相关条件 ====================

/// <summary>
/// 检查是否在地面
/// </summary>
[CreateAssetMenu(menuName = "StateMachine/Conditions/IsGrounded")]
public class IsGroundedCondition : TransitionConditionSO
{
    public bool requireGrounded = true;

    public override bool Evaluate(StateTransitionContext context)
    {
        // 根据你的实际接地检测方式修改
        return context.StateMachine.IsGrounded == requireGrounded;
    }
}
```

**Inspector 里看起来是这样的**：

```plain
HasMoveInput_NoInput (SO 资源)
├── Require Input: ☐ (false，即要求没有移动输入)

AnimFrameCheck_WalkStartQuickStop (SO 资源)  
├── Anim State Name: "WalkStart"
├── Compare Type: LessOrEqual
├── Frame Threshold: 3
```

---

## 第二层：转换规则 (Transition)

### 单条转换规则

```csharp
/// <summary>
/// 一条完整的状态转换规则
/// 包含：条件组 + 目标状态 + 动画过渡参数
/// </summary>
[CreateAssetMenu(menuName = "StateMachine/TransitionRule")]
public class StateTransitionRuleSO : ScriptableObject
{
    [Header("=== 转换目标 ===")]
    [Tooltip("目标状态类型")]
    public StateType targetState;

    [Tooltip("目标动画状态名")]
    public string targetAnimName;

    [Tooltip("CrossFade 过渡时间")]
    public float transitionDuration = 0.1f;

    [Tooltip("CrossFade 偏移")]
    public float transitionOffset = 0f;

    [Header("=== 转换条件 ===")]
    [Tooltip("所有条件都满足时才触发转换（AND 关系）")]
    public TransitionConditionSO[] conditions;

    [Header("=== 优先级 ===")]
    [Tooltip("数值越大优先级越高，同时满足多条规则时取最高优先级")]
    public int priority = 0;

    // 缓存 Hash
    private int _targetAnimHash = -1;
    public int TargetAnimHash
    {
        get
        {
            if (_targetAnimHash == -1)
                _targetAnimHash = Animator.StringToHash(targetAnimName);
            return _targetAnimHash;
        }
    }

    /// <summary>
    /// 检查所有条件是否都满足
    /// </summary>
    public bool AllConditionsMet(StateTransitionContext context)
    {
        if (conditions == null || conditions.Length == 0)
            return false; // 没有条件的规则不会触发（安全保护）

        for (int i = 0; i < conditions.Length; i++)
        {
            if (conditions[i] == null)
            {
                Debug.LogWarning($"TransitionRule [{name}] 条件[{i}] 为 null!");
                return false;
            }
            if (!conditions[i].Evaluate(context))
                return false;
        }
        return true;
    }
}
```

**Inspector 里看起来是这样的**：

```plain
Walking_To_Idle_QuickStop (SO 资源)
├── Target State: Idle
├── Target Anim Name: "WalkStartEnd"
├── Transition Duration: 0.08
├── Transition Offset: 0
├── Conditions: (数组，拖入 SO 引用)
│   ├── [0] HasMoveInput_NoInput       ← 没有移动输入
│   └── [1] AnimFrame_WalkStartQuick   ← 在 WalkStart 前3帧内
└── Priority: 10
```

---

## 第三层：状态转换配置表 (Config)

### 每个状态一张配置表

```csharp
/// <summary>
/// 一个状态的所有出口转换规则集合
/// 每个 State 挂一个这样的 SO
/// </summary>
[CreateAssetMenu(menuName = "StateMachine/StateTransitionConfig")]
public class StateTransitionConfigSO : ScriptableObject
{
    [Tooltip("该状态的所有转换规则，按优先级从高到低排列")]
    public StateTransitionRuleSO[] transitionRules;

    // 运行时排序标记
    private bool _sorted = false;

    /// <summary>
    /// 初始化时按优先级排序
    /// </summary>
    public void Initialize()
    {
        if (transitionRules == null) return;
        System.Array.Sort(transitionRules, (a, b) => b.priority.CompareTo(a.priority));
        _sorted = true;
    }

    /// <summary>
    /// 检查是否有满足条件的转换，返回第一个满足的（最高优先级）
    /// </summary>
    public bool TryGetTransition(StateTransitionContext context, out StateTransitionRuleSO result)
    {
        result = null;
        if (transitionRules == null) return false;

        if (!_sorted) Initialize();

        foreach (var rule in transitionRules)
        {
            if (rule != null && rule.AllConditionsMet(context))
            {
                result = rule;
                return true;
            }
        }
        return false;
    }
}
```

**Inspector 里看起来是这样的**：

```plain
WalkingState_TransitionConfig (SO 资源)
├── Transition Rules: (数组)
│   ├── [0] Walking_To_Idle_QuickStop  (Priority: 10)
│   ├── [1] Walking_To_Running         (Priority: 5)
│   └── [2] Walking_To_Idle_NormalStop (Priority: 0)
```

---

## 第四层：集成到状态机

### 状态基类改造

```csharp
public abstract class PlayerMovementState<TStateData> : IState
    where TStateData : ScriptableObject
{
    protected Animator animator;
    protected PlayerMovementStateMachine stateMachine;
    protected PlayerMovementReusableData reusableData;
    protected StateTransitionContext transitionContext;

    [Header("状态配置")]
    [SerializeField] protected TStateData stateData;

    [Header("转换规则配置")]
    [SerializeField] protected StateTransitionConfigSO transitionConfig;

    public virtual void Enter()
    {
        transitionConfig?.Initialize();
    }

    public virtual void Update()
    {
        // 每帧刷新上下文
        transitionContext.UpdateInput();

        // 统一检查转换
        CheckTransitions();
    }

    /// <summary>
    /// 统一的转换检查，子类一般不需要重写
    /// </summary>
    protected virtual void CheckTransitions()
    {
        if (transitionConfig == null) return;

        if (transitionConfig.TryGetTransition(transitionContext, out var rule))
        {
            ExecuteTransition(rule);
        }
    }

    /// <summary>
    /// 执行转换：切动画 + 切状态
    /// </summary>
    protected void ExecuteTransition(StateTransitionRuleSO rule)
    {
        // 切动画
        animator.CrossFadeInFixedTime(
            rule.TargetAnimHash,
            rule.transitionDuration,
            0,
            rule.transitionOffset
        );

        // 切状态
        stateMachine.ChangeState(rule.targetState);
    }

    public virtual void Exit() { }
}
```

### 具体状态类 —— 变得极其简洁

```csharp
/// <summary>
/// Walking 状态
/// 转换逻辑全部由 SO 配置驱动，这里只需要写状态自身的行为逻辑
/// </summary>
public class PlayerWalkingState : PlayerMovementState<PlayerWalkData>
{
    public override void Enter()
    {
        base.Enter();
        // 只处理进入状态时的逻辑
        reusableData.CurrentSpeed = stateData.walkSpeed;
    }

    public override void Update()
    {
        base.Update(); // 这里面会自动检查转换

        // 只处理状态自身的行为（移动、转向等）
        HandleMovement();
        HandleRotation();
    }

    private void HandleMovement()
    {
        // 移动逻辑...
    }

    private void HandleRotation()
    {
        // 转向逻辑...
    }

    public override void Exit()
    {
        base.Exit();
        // 退出状态时的清理逻辑
    }
}

/// <summary>
/// Idle 状态 —— 同样简洁
/// </summary>
public class PlayerIdleState : PlayerMovementState<PlayerIdleData>
{
    public override void Enter()
    {
        base.Enter();
        reusableData.CurrentSpeed = 0f;
    }

    public override void Update()
    {
        base.Update();
        // Idle 状态自身逻辑（如待机动画随机播放等）
    }
}
```

---

## 资源目录结构

```plain
Assets/
└── ScriptableObjects/
    └── StateMachine/
        ├── Conditions/                          ← 条件 SO 资源
        │   ├── HasMoveInput_True.asset
        │   ├── HasMoveInput_False.asset
        │   ├── IsSprintPressed_True.asset
        │   ├── AnimFrame_WalkStartQuick.asset
        │   └── IsGrounded_True.asset
        │
        ├── TransitionRules/                     ← 单条规则 SO 资源
        │   ├── Idle_To_Walking.asset
        │   ├── Walking_To_Idle_QuickStop.asset
        │   ├── Walking_To_Idle_NormalStop.asset
        │   └── Walking_To_Running.asset
        │
        └── StateConfigs/                        ← 每个状态的转换配置
            ├── IdleState_TransitionConfig.asset
            ├── WalkingState_TransitionConfig.asset
            └── RunningState_TransitionConfig.asset
```

---

## 完整工作流程图

```plain
每帧 Update
    │
    ▼
transitionContext.UpdateInput()     ← 刷新输入数据
    │
    ▼
transitionConfig.TryGetTransition() ← 遍历当前状态的所有规则
    │
    ├── Rule[0] (Priority:10) → 检查所有 Condition
    │   ├── HasMoveInput_False  → ✅
    │   └── AnimFrame_QuickStop → ❌ (当前帧 > 3)
    │   → 不满足，跳过
    │
    ├── Rule[1] (Priority:5)  → 检查所有 Condition
    │   └── IsSprintPressed    → ❌
    │   → 不满足，跳过
    │
    └── Rule[2] (Priority:0)  → 检查所有 Condition
        └── HasMoveInput_False → ✅
        → 全部满足！返回这条规则
    │
    ▼
ExecuteTransition(rule)
    ├── animator.CrossFadeInFixedTime("RunEnd", 0.15f)
    └── stateMachine.ChangeState(StateType.Idle)
```

---

## 注意事项

### 1. SO 资源数量会很多

Condition 可以复用，例如同一个 `HasMoveInput_False` 能被多条规则引用。真正需要注意的是目录结构和命名规范，否则项目中后期会很难找资源。

### 2. 复杂条件怎么办？

可以增加一个 `CompositeCondition`，例如 `AnyOfCondition`，用来支持 OR 逻辑：

```csharp
[CreateAssetMenu(menuName = "StateMachine/Conditions/AnyOf")]
public class AnyOfCondition : TransitionConditionSO
{
    public TransitionConditionSO[] subConditions;

    public override bool Evaluate(StateTransitionContext context)
    {
        foreach (var c in subConditions)
            if (c.Evaluate(context)) return true;
        return false;
    }
}
```

### 3. 调试困难？

可以加一个调试工具，在运行时显示每条规则的评估结果：

```csharp
#if UNITY_EDITOR
[Header("Debug")]
public bool showDebugLog = false;
#endif
```

### 4. 性能

每帧遍历条件数组有少量开销，但对于角色状态机级别通常不是问题。只要单个状态的规则数量控制在合理范围内，例如小于 20 条，实际压力很小。

---

## 总结

```plain
方案的本质：
  把 if-else 代码 → 变成 Inspector 里的拖拽配置

适合场景：
  ✅ 状态数量 10+，转换规则 20+
  ✅ 有策划需要独立调参
  ✅ 需要频繁迭代转换逻辑
  ✅ 团队协作，减少代码冲突

不适合场景：
  ❌ 原型阶段，状态还没定下来
  ❌ 独立开发，状态 < 5 个
  ❌ 转换条件涉及大量复杂运算逻辑
```
