---
title: 截图还原 UGUI 面板 - 样例
published: 2026-09-28
pinned: false
description: "用剧本解锁弹窗 UnlockDramaPopupPanel 的 720×1640，把确认过的节点表和两处坐标算完。"
image: ""
tags: ["Unity", "UGUI","SKILL"]
category: UI
draft: false
---

**截图还原系列：** [SKILL 介绍](/posts/unity/ui/screenshotpanel/01-intro/) · 样例 · [核心设计](/posts/unity/ui/screenshotpanel/03-design/) · [快速开始](/posts/unity/ui/screenshotpanel/04-quickstart/) · [总结](/posts/unity/ui/screenshotpanel/05-summary/)

## AI输出的中间结果

### 第一段（可读树+节点表）

**可读树**

```text
面板：UnlockDramaPopupPanel
校准：入口A artRef=_ArtRef texture=720x1640 artRect=720x1640 preserveAspect=false
root=720x1640 来源=效果图+舞台根
约定：已读 project.md
输出：Assets/_Remote/Modules/UnlockDrama/Prefabs/UnlockDrama/UnlockDramaPopupPanel.prefab

UnlockDramaPopupPanel                 RectTransform   居中 720x1640  保持现有根
├── _ArtRef                           RawImage        铺满根，建完关闭  raycast=false
├── Light                             Image           书背后的光
├── Book                              Image           剧本封面
├── BgWords                           Image           文案底
├── Title2                            嵌套预制体       剧本解锁 + 两侧线
├── [ExText]DramaName                 ExText          剧名
├── [ExText]Desc                      ExText          三行简介
├── [ButtonEffect]ConfirmBtn          Image+ButtonEffect  立即穿越
├── Hint                              ExText          点击其他区域返回（装饰，不加绑定）
├── PowerHudPanel                     嵌套预制体       左上体力
└── HomeButton                        嵌套预制体       左下首页
```

**节点表**

| path | components | texRect | anchorMinMax | pivot | size | anchoredPos |
| --- | --- | --- | --- | --- | --- | --- |
| _ArtRef | RawImage | {0,0,720,1640} | (0,0)-(1,1) | (0.5,0.5) | (0,0) | (0,0) |
| Light | Image | {0,377,720,720} | (0.5,0.5)-(0.5,0.5) | (0.5,0.5) | (720,720) | (0,83) |
| Book | Image | {162,469,396,537} | (0.5,0.5)-(0.5,0.5) | (0.5,0.5) | (396,537) | (0,82.5) |
| BgWords | Image | {17.5,1022,685,186} | (0.5,0.5)-(0.5,0.5) | (0.5,0.5) | (685,186) | (0,-295) |
| Title2 | 预制体根 | {0,300,720,50} | (0.5,0.5)-(0.5,0.5) | (0.5,0.5) | (720,50) | (0,495) |
| [ExText]DramaName | ExText | {100,1036,520,42} | (0.5,0.5)-(0.5,0.5) | (0.5,0.5) | (520,42) | (0,-237) |
| [ExText]Desc | ExText | {80,1100,560,96} | (0.5,0.5)-(0.5,0.5) | (0.5,0.5) | (560,96) | (0,-328) |
| [ButtonEffect]ConfirmBtn | Image, ButtonEffect | {183.5,1250,353,72} | (0.5,0.5)-(0.5,0.5) | (0.5,0.5) | (353,72) | (0,-466) |
| Hint | ExText | {200,1328,320,28} | (0.5,0.5)-(0.5,0.5) | (0.5,0.5) | (320,28) | (0,-522) |
| PowerHudPanel | 预制体根 | {19,213,259.4,63.1} | (0,1)-(0,1) | (0.5,0.5) | (259.4,63.1) | (148.7,-244.6) |
| HomeButton | 预制体根 | {14,1355,100,100} | (0,0)-(0,0) | (0.5,0.5) | (100,100) | (64,235) |

### 第二段（资源表）

树的数字按上一张表执行。工程内这些图都是单张 Sprite，路径如下。请确认这张资源表；确认前不会创建控件。

| 节点 | 用途 | 建议路径 | 用户确认 |
| --- | --- | --- | --- |
| _ArtRef | 引导层，建完关闭 | `Assets/_Remote/Modules/UnlockDrama/Prefabs/UnlockDrama/效果图.png` |  |
| Light | 书背后的光 | `Assets/_Remote/Modules/UnlockDrama/Textures/img_light.png` |  |
| Book | 剧本封面 | `Assets/_Remote/Modules/UnlockDrama/Atlas/UnlockDramaPopup/icon_book.png` |  |
| BgWords | 文案底 | `Assets/_Remote/Modules/UnlockDrama/Textures/bg_words.png` |  |
| Title2 | 标题嵌套预制体 | `Assets/_Remote/Common/Prefabs/Title2.prefab` |  |
| Title2 中心 Image | 把「成就展示」换成「剧本解锁」 | `Assets/_Remote/Modules/UnlockDrama/Atlas/UnlockDramaPopup/img_title.png` |  |
| [ExText]DramaName / [ExText]Desc / Hint | 字体 | `Assets/_Remote/Common/Fonts/RegularInGame SDF.asset` |  |
| [ButtonEffect]ConfirmBtn | 立即穿越 | `Assets/_Remote/Modules/UnlockDrama/Atlas/UnlockDramaPopup/btn_confirm.png` |  |
| PowerHudPanel | 左上体力 | `Assets/_Remote/Modules/Power/Prefabs/PowerHudPanel.prefab` |  |
| HomeButton | 左下首页 | `Assets/_Remote/Common/Prefabs/HomeButton.prefab` |  |

## 产出效果

**原效果图**

![](https://cdn.nlark.com/yuque/0/2026/png/55136814/1790592849412-9ae0c887-307f-4009-8d68-58784043ea19.png)

**产出的UGUI面板**

![](https://cdn.nlark.com/yuque/0/2026/png/55136814/1790593142201-2953bdb5-f983-4894-83b8-84d43015492e.png)
