---
title: Unity 加权随机
published: 2026-07-03
pinned: false
description: "用一个简单函数理解按权重随机选择 LevelPath 的实现思路"
image: ""
tags: ["Unity", "随机", "权重"]
category: 技巧
draft: false
---

# Unity 加权随机选择关卡路径

我在做关卡路径选择时，经常会遇到一种需求：不是每条路径都平均出现，而是希望某些路径更容易被选中。比如普通路线权重大一点，稀有路线权重小一点，这时就可以用“加权随机”。

下面这个函数就是一个很典型的写法：

```csharp
public LevelPath GetLevelPath()
{
    float random = Random.Range(0f, totalWeight);

    float sum = 0;

    for (int i = 0; i < levelPathConfigs.Length; i++)
    {
        sum += levelPathConfigs[i].weight;

        if (sum > random)
            return levelPathConfigs[i].levelPath;
    }

    return null;
}
```

## 它解决了什么问题

如果我有 3 条路线：

```plain
A 路线：weight = 50
B 路线：weight = 30
C 路线：weight = 20
```

那么 `totalWeight` 就是：

```plain
50 + 30 + 20 = 100
```

函数会先在 `0 ~ 100` 之间随机一个数，然后用累计权重判断这个数落在哪个区间：

```plain
A 路线：0  ~ 50
B 路线：50 ~ 80
C 路线：80 ~ 100
```

对应的概率可以直接按 `weight / totalWeight` 算：

| 路线 | 权重 | 命中区间 | 概率 |
|---|---:|---|---:|
| A 路线 | 50 | `0 ~ 50` | `50%` |
| B 路线 | 30 | `50 ~ 80` | `30%` |
| C 路线 | 20 | `80 ~ 100` | `20%` |

所以随机数如果是 `35`，就会选中 A；如果是 `66`，就会选中 B；如果是 `92`，就会选中 C。

## 核心逻辑

这段代码的关键在这几行：

```csharp
sum += levelPathConfigs[i].weight;

if (sum > random)
    return levelPathConfigs[i].levelPath;
```

`sum` 会不断累加当前遍历到的权重。只要累计值超过随机数，就说明随机数落在当前配置对应的权重区间里，于是直接返回当前的 `levelPath`。

换句话说，`weight` 不是百分比，而是“占总权重的比例”。权重越大，占据的区间越长，被随机命中的概率也越高。

## 为什么最后 return null

正常情况下，只要 `totalWeight` 正确，并且每个 `weight` 都是有效正数，循环里一定会返回一个 `LevelPath`。

最后的：

```csharp
return null;
```

更像是一个兜底，防止出现这些异常情况：

- `levelPathConfigs` 为空。
- `totalWeight` 算错了。
- 所有 `weight` 都是 0。
- 配置数据里存在非法权重。

实际项目里我会尽量在初始化阶段就校验这些配置，而不是等到随机时才发现返回了 `null`。

## 使用时要注意

这个函数本身很简洁，但依赖一个前提：`totalWeight` 必须等于所有配置权重之和。

例如可以在初始化时计算：

```csharp
private void CalculateTotalWeight()
{
    totalWeight = 0f;

    for (int i = 0; i < levelPathConfigs.Length; i++)
    {
        totalWeight += levelPathConfigs[i].weight;
    }
}
```

如果后续会动态修改权重，就要记得同步更新 `totalWeight`，否则随机结果会偏离预期。

## 总结

这类加权随机的思路可以记成一句话：

```plain
先把所有权重拼成一条连续区间，再随机一个点，看这个点落在哪一段。
```

它很适合用在关卡路线、掉落表、敌人刷新、随机事件等场景。只要权重配置清楚，逻辑就很好扩展，也方便之后把数据放到 ScriptableObject 或表格里维护。
