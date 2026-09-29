---
title: 引导层、门闩和三条入口
published: 2026-09-28
pinned: false
description: "引导层换算、两道确认、入口 A/B/C，以及 Prefab 模式打开时改哪一份根。"
image: ""
tags: ["Unity", "UGUI"]
category: UI
draft: true
---

![](https://cdn.nlark.com/yuque/0/2026/png/55136814/1790595076187-38ad5db9-d03c-48ad-9152-447f61658514.png)

![](https://cdn.nlark.com/yuque/0/2026/png/55136814/1790594776596-119b8c50-20f4-4872-a023-5ec748d6bc11.png)

```text
树未确认：只读。跑确认前只读骨架，可以看层级。不许保存，不许改层级。
树已确认、资源表未确认：只在 searchPaths 里搜贴图和预制体。
两张表都确认：才写 RectTransform，才换 Sprite，才存预制体。
```

## 尺子在引导层上

引导层是装饰节点。名字用 `project.md` 的 `artRefName`，本仓库是 `_ArtRef`。不加方括号。组件是 `RawImage`，`raycastTarget=false`。它和分组容器都挂在面板根下。控件放进分组容器，不挂到引导层下面。

像素框画在引导层那张贴图上，原点左上，Y 向下，记成 `texRect={x,y,w,h}`。你另外丢了一张聊天截图时，先对齐到这张贴图再框。

只读骨架返回的字段就够换算：贴图宽高、引导层 `rect`、`uvRect`、`preserveAspect`，以及根的 `rect`。`uvRect` 的原点在左下。不是 `(0,0,1,1)` 时，先按 UV 裁出可见区域，再框控件。`preserveAspect=false` 时，绘制区就是整个 `rect`。为 true 时用贴图比例在 `rect` 里做内接矩形，像素框映射到这块绘制区，不映射到整个 `rect`。

Unity 2021.3 及更早的 `RawImage` 没有 `preserveAspect`（`Image` 一直有）。只读骨架用反射读这个属性，不写 `artRaw.preserveAspect`。只读编译对着当前工程的 `UnityEngine.UI`，2021 上直接编译失败，`try/catch` 轮不到。反射找不到属性时按 false，返回里带 `missingProperty=1`。这个版本的 `RawImage` 把贴图铺满 `rect`。

换算顺序是：UV 裁可见区域，归一化，落到绘制区（中心原点，Y 向上），再转到面板根。引导层和根都铺满、缩放是 1、两边 `rect` 相同，绘制区的本地值就是根空间矩形。节点直接挂在根上，或者父节点铺满根且偏移为 0、缩放为 1，就用这组根空间数写成 `RectTransform`。其它父节点先平移到父矩形中心，父宽高用父节点自己的，再套同一组锚点公式。

写入顺序：`anchorMin`、`anchorMax`、`pivot`、`sizeDelta`、`anchoredPosition`。后写的数会盖掉中间值。贴边用边锚点。铺满父节点的底图，以及根，用 stretch `(0,0)-(1,1)`，`size=(0,0)`，`anchoredPos=(0,0)`。覆盖率超过 90% 的底图用 stretch，相对它的父节点。横向铺满、贴父节点顶边：锚点 `(0,1)-(1,1)`，pivot `(0.5,1)`，`sizeDelta=(0, 高度)`，`anchoredPosition=(0, -距顶)`。高度和距顶是父节点本地单位，从 `texRect` 换出来。

> [!WARNING] 根或引导层的 rect 为 0
> 引导层是 stretch，当前又不在 Canvas 下，`rect` 的宽或高会是 0。返回里出现 `RECT_ZERO` 就停。0 不能当除数。放到带 `CanvasScaler` 的 Canvas 下再量，或给引导层明确的 `sizeDelta`。根 `rect` 为 0 同样停。入口 A/C 不改用设计分辨率。

## 三条入口

入口 A：空壳或几乎只有引导层。按节点表添加控件，保留引导层。先父后子。已有的业务控件不覆盖。

入口 B：没有引导层，也没有可改的预制体，只有聊天截图。截图宽 `Sw`、高 `Sh`，设计画布 `Dw×Dh`。`Dw×Dh` 按这个顺序取，取到就停：`project.md` 的 `designResolution` 宽和高都有值；否则 `canvasSource` 指向的场景或预制体上的 `CanvasScaler.referenceResolution`；再否则当前已加载场景里的 `CanvasScaler`。有多个 Canvas 就问用哪一个。仍然没有就问，不默认 `1080×1920`。截图明显不是全屏时，先问它盖住哪一块。

```text
scale = Dw / Sw
localW = w * scale
localH = h * scale
cx = (x + w/2) * scale - Dw/2
cy = Dh/2 - (y + h/2) * scale
```

这里的 `cx`、`cy` 已经在根中心、Y 向上。居中锚点、pivot `0.5` 时，`sizeDelta=(localW, localH)`，`anchoredPosition=(cx, cy)`。树头写：`校准：入口B 未校准，尺寸可能偏`。

入口 C：已有面板，根下除引导层外已经有控件，要用参考图改布局、换资源、增删。没有可用的引导层时，先补参考图。不退回入口 B 去估设计分辨率。

节点表不写操作。增删改只认 diff。剧本解锁那次是入口 A，根下原先没有这些控件，执行的是整张节点表。下面这张是下次改它时的样子：

| op | 现有 path | 目标 path/name | 说明 |
| --- | --- | --- | --- |
| keep | _ArtRef | _ArtRef | 引导层不动 |
| keep | [ExText]DramaName | 同左 | 剧名还在 |
| layout | HomeButton | 同左 | 左下锚点，size=(100,100)，pos=(64,235) |
| resource | Title2 中心图 | 同左 | 「成就展示」换成 img_title.png |
| add | — | [ExText]Desc | 截图上新出现的简介 |
| delete | — | — | 这张图上没有要删的。不确定的节点标 keep |

`op` 还有 `layout+resource`，以及 `reparent`（改名或换父，写清旧 path 到新 path）。匹配时，path 或节点名相同视为同一个节点。对不上时，同父、同主组件，现有中心落在目标框附近（中心距小于两边较短边的 40%），也视为同一个，标 `layout` 或 `reparent`。截图有、现有没有，是 `add`。现有有、截图没有，默认 `keep`，注明「截图未见」。比较有把握是废弃装饰或旧按钮时才提案 `delete`，等你确认。引导层、挂了业务脚本的根、嵌套预制体实例，不提案删除，除非你点名。删父节点等于删整棵子树，diff 里写清会同时删掉哪些子节点。

执行时先核对 `reparent`、`delete`、`layout`、`resource` 的路径。有缺失就立刻返回，这时还不改层级。然后按这个顺序：改父或改名，删除（`Object.DestroyImmediate`，先子后父），改 Rect，换资源，新增，最后按确认树 `SetSiblingIndex`。`keep` 跳过。找不到路径时，不用 `add` 顶上一个同名节点。

## Prefab 模式打开时改舞台根

写入前再判一次，确认前那次返回的 `editingStage=` 不能代替。`UnityEditor.SceneManagement.PrefabStageUtility.GetCurrentPrefabStage()` 拿当前舞台。`stage` 为空，或者 `assetPath` 不是这次要改的预制体，走 `PrefabUtility.LoadPrefabContents`，改完 `SaveAsPrefabAsset`，再 `UnloadPrefabContents`。

保存后再跑只读骨架。`editingStage=true` 时读到的是舞台。节点名对不上，说明改动打在磁盘副本上、随后被旧舞台盖掉了。改到 `prefabContentsRoot`，再 `save_prefab_stage`。

主要是防止在开着预制体模式时，改动在磁盘副本会不成功，直接在预制体舞台上改

## 写进去时还守着的几条

可点控件 `raycastTarget=true`。装饰图、纯展示文字、引导层为 false。父节点带 `HorizontalLayoutGroup` 或 `VerticalLayoutGroup` 时，子节点不写 `anchoredPosition`。

正式资源模式下，Sprite 加载失败，或多 Sprite 却没有 `spriteName`，中止保存并报告。不改成纯色。只有你选了占位模式，才用纯色 `Image`。九宫格没在节点表写明时用 `Image.Type.Simple`。

没说「这就是正式背景」，构建结束时引导层 `SetActive(false)`，贴图引用留着。说了就保持显示。

## 如何搭配UIBinder的？

在该SKILL下的`project.md` 中 设置好对应的配置，这样在做预制体的时候就能按UIBinder的命名形式去做

上一篇：[用一张弹窗把坐标算完](/posts/unity/ui/screenshotpanel/02-example/)
下一篇：[从 project.md 到确认写入](/posts/unity/ui/screenshotpanel/04-quickstart/)
