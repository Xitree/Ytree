---
title: Unity RectTransform 详解
published: 2026-07-03
pinned: false
description: "整理 Unity UGUI 中 RectTransform、锚点、Pivot、Offset、SizeDelta 与常用 API 的理解"
image: ""
tags: ["Unity", "UGUI", "RectTransform"]
category: UI
draft: false
---

# Unity RectTransform 详解

> [!NOTE] 转载说明
> 本文为转载导读与学习整理，参考原文：[《Unity——RectTransform详解》](https://www.jianshu.com/p/4592bf809c8b)。
> 原文著作权归原作者所有；本文不做全文搬运，仅整理 RectTransform 的核心概念、使用场景和个人理解，便于后续查阅。

## 前言

在 Unity 的 UGUI 系统里，`RectTransform` 是所有 UI 元素布局的核心组件。它不像普通 `Transform` 只关心位置、旋转和缩放，还要处理父子 UI 之间的尺寸关系、锚点关系、屏幕适配和边距约束。

理解 `RectTransform` 时，最关键的是先把几个概念串起来：

- `Anchor`：子 UI 相对父 UI 的参考点或参考框。
- `Pivot`：UI 自身旋转、缩放和定位时使用的中心点。
- `Offset`：UI 边界相对锚点或锚框边界的偏移。
- `sizeDelta`：UI 尺寸与锚点/锚框之间的差值。
- `anchoredPosition`：`Pivot` 相对锚点或锚框中心的位置。

---

## Anchor：锚点与锚框

`Anchor` 由两个值决定：

- `anchorMin`
- `anchorMax`

它们都是归一化坐标，基于父物体矩形范围计算。父物体左下角是 `(0, 0)`，右上角是 `(1, 1)`。

### 锚点

当 `anchorMin == anchorMax` 时，Anchor 表现为一个点。这个点就是常说的“锚点”。

这种情况下，子 UI 的位置和尺寸更像“固定尺寸 + 固定参考点”：

- UI 的宽高通常由 `Width` / `Height` 控制。
- `Pos X` / `Pos Y` 表示 `Pivot` 到锚点的距离。
- 父物体尺寸变化时，子 UI 尺寸一般不会自动拉伸。

这种方式适合固定大小的按钮、图标、角标等 UI。

### 锚框

当 `anchorMin != anchorMax` 时，Anchor 表现为一个矩形区域，也可以理解为“锚框”。

这种情况下，子 UI 会和父物体的某个范围建立拉伸关系：

- UI 的边界会参考锚框边界。
- Inspector 面板会显示 `Left`、`Right`、`Top`、`Bottom` 等边距。
- 父物体尺寸变化时，子 UI 可能随之拉伸或收缩。

这种方式适合背景板、列表区域、全屏遮罩、适配屏幕边缘的 UI。

---

## 绝对布局与相对布局

### 绝对布局

绝对布局通常对应“锚点”情况。此时 UI 元素的尺寸相对固定，位置由 `Pivot` 到锚点的距离决定。

例如一个按钮锚在父物体中心：

```plain
anchorMin = (0.5, 0.5)
anchorMax = (0.5, 0.5)
pivot     = (0.5, 0.5)
pos       = (0, 0)
size      = (200, 80)
```

这表示按钮中心贴在父物体中心，按钮大小为 `200 x 80`。父物体变大或变小时，按钮仍保持原尺寸。

绝对布局的问题是：不同分辨率下 UI 可能显得过大或过小。因此它更适合不需要跟随父物体缩放的局部元素。

### 相对布局

相对布局通常对应“锚框”情况。此时 UI 元素和父物体边界之间存在边距关系。

例如一个面板想要距离父物体四边各 `20`：

```plain
anchorMin = (0, 0)
anchorMax = (1, 1)
left      = 20
right     = 20
top       = 20
bottom    = 20
```

当父物体变大时，面板也会变大，但它和父物体四条边的距离仍保持为 `20`。

相对布局更适合做屏幕适配，但需要注意边距不能超过父物体尺寸，否则可能出现 UI 尺寸为负数的异常显示。

---

## Pivot：UI 自身的中心点

`Pivot` 表示 UI 自身的参考中心，取值范围同样是 `(0, 0)` 到 `(1, 1)`：

- `(0, 0)`：左下角
- `(0.5, 0.5)`：中心
- `(1, 1)`：右上角

`Pivot` 会影响：

- UI 的旋转中心
- UI 的缩放中心
- `anchoredPosition` 的含义
- 绝对布局下 `Pos X` / `Pos Y` 的计算

例如一个血条从左向右增长，可以把 `Pivot X` 设置为 `0`，这样缩放时左侧固定，右侧变化。

---

## Offset：边界偏移

`offsetMin` 和 `offsetMax` 描述的是 UI 边界相对锚点或锚框的偏移。

可以简单理解为：

- `offsetMin`：左下角相关偏移。
- `offsetMax`：右上角相关偏移。

在锚框模式下，它们常被用来表达 UI 与父物体边界之间的距离。比如全屏面板留出边距，本质上就是在调整这些 offset。

```csharp
RectTransform rectTransform = GetComponent<RectTransform>();

rectTransform.anchorMin = Vector2.zero;
rectTransform.anchorMax = Vector2.one;
rectTransform.offsetMin = new Vector2(20f, 20f);
rectTransform.offsetMax = new Vector2(-20f, -20f);
```

这段代码表示 UI 拉伸到父物体范围内，但四边留出 `20` 的距离。

---

## sizeDelta：尺寸还是差值？

`sizeDelta` 是 RectTransform 里最容易误解的属性之一。

在锚点模式下，`sizeDelta` 基本就等价于 UI 的宽高：

```plain
anchorMin = anchorMax
sizeDelta ≈ width / height
```

但在锚框模式下，`sizeDelta` 更像“UI 尺寸与锚框尺寸之间的差值”，不再直接等价于最终宽高。

所以实践中是这样：

- 固定锚点时，用 `sizeDelta` 设置尺寸比较直观。
- 拉伸锚框时，优先用 `offsetMin` / `offsetMax` 控制边距。
- 想稳定读取最终宽高时，用 `rect.width` / `rect.height`。

---

## rect：读取 UI 自身矩形

`rect` 是只读属性，描述 UI 自身矩形的信息。

常用的是：

```csharp
RectTransform rectTransform = GetComponent<RectTransform>();

float width = rectTransform.rect.width;
float height = rectTransform.rect.height;
```

`rect.width` 和 `rect.height` 不依赖你当前是锚点还是锚框模式，通常是读取 UI 实际尺寸时更稳妥的选择。

不过 `rect` 只能读，不能直接写。如果要改尺寸，需要使用 `sizeDelta`、`SetSizeWithCurrentAnchors` 或相关布局方法。

---

## anchoredPosition：Pivot 到参考位置的偏移

`anchoredPosition` 表示 UI 的 `Pivot` 相对 Anchor 参考位置的偏移。

在锚点模式下，它表示：

```plain
Pivot 到锚点的距离
```

在锚框模式下，它更接近：

```plain
Pivot 到锚框中心的距离
```

例如把 UI 放到锚点下方 `100`：

```csharp
RectTransform rectTransform = GetComponent<RectTransform>();
rectTransform.anchoredPosition = new Vector2(0f, -100f);
```

需要注意的是，`anchoredPosition` 的实际视觉效果会受到 `anchorMin`、`anchorMax` 和 `pivot` 一起影响。

---

## 常用 API

### SetSizeWithCurrentAnchors

`SetSizeWithCurrentAnchors` 可以在当前锚点设置不变的前提下，设置某个轴向上的尺寸。

```csharp
RectTransform rectTransform = GetComponent<RectTransform>();

rectTransform.SetSizeWithCurrentAnchors(RectTransform.Axis.Horizontal, 100f);
rectTransform.SetSizeWithCurrentAnchors(RectTransform.Axis.Vertical, 100f);
```

相比直接改 `sizeDelta`，这个方法在锚框模式下更容易得到符合预期的尺寸结果。

### SetInsetAndSizeFromParentEdge

`SetInsetAndSizeFromParentEdge` 可以指定 UI 相对父物体某一条边的距离和尺寸。

```csharp
RectTransform rectTransform = GetComponent<RectTransform>();

rectTransform.SetInsetAndSizeFromParentEdge(
    RectTransform.Edge.Right,
    200f,
    400f
);
```

这表示以父物体右边界为基准，UI 距离右边界 `200`，并在对应轴向上设置尺寸为 `400`。

使用这个方法时，Unity 会根据选择的边自动调整相关锚点：

- `Left`：横向锚点变为左侧。
- `Right`：横向锚点变为右侧。
- `Top`：纵向锚点变为上侧。
- `Bottom`：纵向锚点变为下侧。

### GetWorldCorners

`GetWorldCorners` 可以获取 UI 四个角的世界坐标。

```csharp
RectTransform rectTransform = GetComponent<RectTransform>();
Vector3[] corners = new Vector3[4];

rectTransform.GetWorldCorners(corners);

Vector3 leftBottom = corners[0];
Vector3 leftTop = corners[1];
Vector3 rightTop = corners[2];
Vector3 rightBottom = corners[3];
```

这个方法常用于：

- 判断 UI 是否超出屏幕。
- 做 UI 与世界坐标对象的对齐。
- 计算 UI 包围区域。

---

## 实用方式

可以把 RectTransform 的几个属性按职责拆开分类：

```plain
Anchor  ：我参考父物体的哪里？
Pivot   ：我自己的中心在哪里？
Offset  ：我的边距离参考边多远？
SizeDelta：我的尺寸或尺寸差是多少？
Rect    ：我当前实际矩形是多少？
Position：我的 Pivot 偏到哪里？
```

写 UI 适配时，先决定 Anchor，再决定 Pivot，最后再调整位置和尺寸。这样会比直接改 `Pos X`、`Width`、`Height` 更不容易乱。

## 总结

`RectTransform` 难理解的地方不在 API 数量，而在这些属性会根据锚点模式改变含义。

如果是固定大小 UI，可以优先使用锚点模式和 `sizeDelta`；如果是需要屏幕适配的 UI，则优先使用锚框模式和 `offsetMin` / `offsetMax`。读取实际尺寸时，`rect.width` 与 `rect.height` 通常比 `sizeDelta` 更可靠。
