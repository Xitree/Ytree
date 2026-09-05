---
title: Unity Animator 动画过渡时长对应关系
published: 2026-07-03
pinned: false
description: "我在 Ya 项目中整理的 Animator 过渡期间 Current State、Next State、IsInTransition 与 normalizedTime 对应关系"
image: ""
tags: ["Unity", "Animator", "动画过渡", "Ya"]
category: 动画
draft: false
---

# 动画时长对应是怎样的

> 项目地址：[Xitree/Ya](https://github.com/Xitree/Ya)

## 为什么要单独整理这个问题

我在做 Ya 项目的动作状态机和 CCT 过渡规则时，经常需要判断“当前动画到底播到哪一帧了”。一开始我以为只要看 `GetCurrentAnimatorStateInfo()` 就够了，但真正遇到 `RunStart -> RunEnd`、攻击前摇转后摇、受击打断这类过渡时，就会发现事情没那么简单。

过渡期间 Unity Animator 不是简单地“源状态结束，目标状态开始”，而是存在一段混合区间：

- 源状态仍然是 `Current State`。
- 目标状态会进入 `Next State`。
- 两个状态的 `normalizedTime` 会同时推进。
- 画面结果由过渡权重混合出来。

这篇文章就是我给自己整理的一份速查笔记，主要解决三个问题：过渡期间 `Current/Next` 分别是谁、`normalizedTime` 怎么变化、实际写代码时应该查哪个值。

---

## 状态切换的基本对应关系

> [!NOTE] 速查
> 只要 Animator 还处于过渡中，源状态仍然是 `Current State`，目标状态会出现在 `Next State`，此时 `IsInTransition` 为 `true`。

```plain
时机      Current State     Next State      IsInTransition
过渡前    RunStart          -               false
过渡中    RunStart（源）     RunEnd（目标）   true
过渡后    RunEnd            -               false
```

所以在 `RunStart -> RunEnd` 的过程中，我一般按下面这个规则看：

- `GetCurrentAnimatorStateInfo()` 拿到的是源状态 `RunStart`。
- `GetNextAnimatorStateInfo()` 拿到的是目标状态 `RunEnd`。
- `Animator.IsInTransition(layer)` 返回 `true`。
- 过渡结束后，`RunEnd` 才会正式变成新的 `Current State`。

这点在写动画窗口判断时很重要。如果还在过渡中，却只拿 `Current` 去判断目标动画，很容易出现“明明已经开始混到目标动作了，但代码还认为自己在源动作里”的情况。

---

## 过渡期间 normalizedTime 的变化

### 完整时间线

我用一个最容易观察的例子来记：

- `RunStart` 时长为 `1` 秒。
- `RunEnd` 时长为 `1` 秒。
- 过渡时长为 `0.3` 秒，也就是 30%。
- `RunStart` 播放到 `0.7` 时开始过渡。

```plain
时间（秒）    0.0    0.2    0.4    0.6    0.7    0.8    1.0    1.2    1.4
             │──────── RunStart ────────│─ 过渡 ─│──────── RunEnd ────────│
                                        ↑        ↑
                                     过渡开始   过渡结束

GetCurrentAnimatorStateInfo (RunStart):
normalizedTime:  0.0    0.2    0.4    0.6    0.7    ×(Exit)
                                              ↑
                                           过渡中仍在推进

GetNextAnimatorStateInfo (RunEnd):
normalizedTime:                               0.0    0.1
                                              ↑      ↑
                                           过渡开始  过渡结束

过渡结束后 → RunEnd 变成 Current：
GetCurrentAnimatorStateInfo (RunEnd):
normalizedTime:                                      0.1    0.3    0.5 ...
```

这里最关键的结论是：**过渡期间两个 `normalizedTime` 会同时各自往前走，互不影响。**

也就是说，`RunStart` 没有因为开始过渡就立刻停住，`RunEnd` 也不是等过渡结束才开始计时。过渡区间里两个动画都在播放，只是最终画面由权重混合出来。

---

## 两个状态的 normalizedTime 对照表

```plain
帧    时间(s)   阶段      Current(RunStart)        Next(RunEnd)
                         normalizedTime           normalizedTime
────────────────────────────────────────────────────────────────
1     0.00     正常播放    0.00                     -
2     0.20     正常播放    0.20                     -
3     0.40     正常播放    0.40                     -
4     0.60     正常播放    0.60                     -
5     0.70     过渡开始    0.70                     0.00  ← RunEnd 从头开始
6     0.80     过渡中      0.80                     0.10
7     0.90     过渡中      0.90                     0.20
8     1.00     过渡结束    ×(Exit, ~1.00)           0.30  ← 变为 Current
9     1.10     正常播放    (现在是 RunEnd) 0.40      -
10    1.20     正常播放    0.50                     -
```

我自己更推荐把这张表当成调试时的参照：如果日志里 `Current` 到了 `0.9`，`Next` 到了 `0.2`，这并不矛盾，它只是说明现在正处于两个状态混合的阶段。

---

## 不同过渡设置的影响

### Transition Offset：目标状态偏移

`Transition Offset` 会影响目标状态从哪里开始播放。

```plain
Transition Offset = 0.5 时：
RunEnd 不从 0 开始，而是从 0.5 开始

帧    Current(RunStart)    Next(RunEnd)
5     0.70                 0.50  ← 从中间开始！
6     0.80                 0.60
7     0.90                 0.70
```

所以如果目标动画进入时看起来“已经播放了一半”，我会优先检查 `Transition Offset`。这个值在动作游戏里尤其容易影响手感：比如闪避结束动画、冲刺结束动画，如果从中段开始播，视觉上会非常突兀。

### Has Exit Time：过渡开始时机

```plain
Exit Time = 0.7 (RunStart 播到 70% 时开始过渡)

Has Exit Time ✓ :
  → RunStart.normalizedTime 到达 0.7 时，过渡自动开始

Has Exit Time ✗ :
  → 由 Trigger/Bool 参数决定何时过渡
  → RunStart.normalizedTime 可以在任意值时开始过渡
```

在 Ya 这种动作项目里，我不会把所有切换都交给 `Has Exit Time` 自动处理。像攻击取消、输入缓冲、受击打断这类逻辑，往往还要结合状态条件、动画事件或 CCT 规则一起判断。

我的理解是：

- 普通自然衔接可以依赖 `Has Exit Time`。
- 需要玩家输入或战斗判定参与的切换，不要只依赖 `Has Exit Time`。
- 如果切换窗口很严格，最好明确记录窗口开始帧和结束帧。

### Transition Duration：固定时长与百分比时长

```plain
Fixed Duration ✓ (秒):
  Transition Duration = 0.3 → 过渡持续 0.3 秒

Fixed Duration ✗ (百分比):
  Transition Duration = 0.3 → 过渡持续 源状态时长 × 30%
  如 RunStart 1秒 → 过渡 0.3秒
  如 RunStart 2秒 → 过渡 0.6秒
```

这里我之前也容易混：`Transition Duration = 0.3` 不一定永远是 `0.3` 秒，要看有没有勾选 `Fixed Duration`。

如果不同动画长度差异很大，`Fixed Duration` 会直接影响手感稳定性：

- 勾选时，过渡时间更稳定。
- 不勾选时，过渡时间会跟随源动画长度变化。

动作游戏里我更倾向于对关键状态使用固定秒数，因为角色动作的响应和取消手感需要稳定。百分比时长更适合一些不太敏感的自然动作衔接。

---

## 代码验证

为了避免只靠猜，我通常会直接在 `Update` 里把 `Current`、`Next` 和过渡进度一起打印出来：

```csharp
void Update()
{
    int layer = 0;
    var current = _animator.GetCurrentAnimatorStateInfo(layer);

    if (_animator.IsInTransition(layer))
    {
        var next = _animator.GetNextAnimatorStateInfo(layer);
        var trans = _animator.GetAnimatorTransitionInfo(layer);

        Debug.Log($"过渡中 | " +
            $"Current nTime: {current.normalizedTime:F3} " +
            $"(length: {current.length:F2}s) | " +
            $"Next nTime: {next.normalizedTime:F3} " +
            $"(length: {next.length:F2}s) | " +
            $"过渡进度: {trans.normalizedTime:F3}");

        // trans.normalizedTime: 0→1 表示过渡从开始到结束的进度
    }
    else
    {
        Debug.Log($"正常 | Current nTime: {current.normalizedTime:F3}");
    }
}
```

输出示例：

```plain
正常   | Current nTime: 0.600
正常   | Current nTime: 0.680
过渡中 | Current nTime: 0.700  Next nTime: 0.000  过渡进度: 0.000
过渡中 | Current nTime: 0.800  Next nTime: 0.100  过渡进度: 0.333
过渡中 | Current nTime: 0.900  Next nTime: 0.200  过渡进度: 0.667
过渡中 | Current nTime: 1.000  Next nTime: 0.300  过渡进度: 1.000
正常   | Current nTime: 0.400  ← 现在是 RunEnd 作为 Current
```

看日志时我主要盯三个值：

- `Current nTime`：源状态的播放进度。
- `Next nTime`：目标状态的播放进度，只在过渡中有效。
- `过渡进度`：这次 Transition 本身从 `0` 到 `1` 的混合进度。

---

## 混合权重

过渡期间动画不是瞬间切换，而是混合。可以粗略理解为权重随过渡进度变化：

```plain
过渡进度    RunStart 权重    RunEnd 权重    视觉效果
0.00        1.0              0.0           完全 RunStart
0.25        0.75             0.25          大部分 RunStart
0.50        0.5              0.5           各一半
0.75        0.25             0.75          大部分 RunEnd
1.00        0.0              1.0           完全 RunEnd
```

这也是为什么过渡期间看到的画面不一定完全等于 `Current` 或 `Next`。代码层面 `Current` 还是源状态，视觉层面却可能已经能明显看到目标状态的动作姿势。

这个差异就是我整理这篇笔记的原因：**视觉上已经像目标动作，不代表 `Current State` 已经变成目标状态。**

---

## 速查图

```plain
         RunStart.normalizedTime
         0.0 ──────────── 0.7 ────── 1.0
         │    正常播放     │  过渡中   │
         │                │          │
         │                │  RunEnd.normalizedTime
         │                │  0.0 ── 0.3 ────── 0.5 ────── 1.0
         │                │  │ 过渡 │     正常播放        │
         │                │  │      │                     │
         ├────────────────┤  ├──────┤                     │
         Current 阶段        过渡     Current 阶段(RunEnd)

         IsInTransition:
         false               true    false
```

## 总结

我最后给自己记一个判断原则：

```plain
IsInTransition == false：
  只看 Current。

IsInTransition == true：
  Current 是源状态，Next 是目标状态。
  两个状态的 normalizedTime 都在推进。
  GetAnimatorTransitionInfo().normalizedTime 表示过渡本身的进度。
```

放到 Ya 项目里，我会这样用：

- 判断当前是否已经进入目标动画：先看 `IsInTransition`，过渡中用 `Next` 判断。
- 判断源动画是否走到可取消窗口：过渡前看 `Current`，过渡中也要确认源状态是否仍然有意义。
- 判断这次过渡是否快结束：看 `GetAnimatorTransitionInfo().normalizedTime`。
- 做 CCT、攻击取消、前后摇检测时，不要把源状态进度、目标状态进度、过渡进度混成一个值。

这套规则记清楚之后，动画调试会轻松很多。至少我再看 Animator 日志时，不会再把“画面已经混过去了”和“状态已经切过去了”当成同一件事。
