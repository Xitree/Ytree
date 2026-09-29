---
title: 确认之后才写入
published: 2026-09-28
pinned: false
description: "截图还原面板收住的四步：坐标按舞台根算完、两道确认才写入、改旧面板只动 diff、绑定留给 UIBinder。"
image: ""
tags: ["Unity", "UGUI"]
category: UI
draft: true
---

![](https://cdn.nlark.com/yuque/0/2026/png/55136814/1790592849412-9ae0c887-307f-4009-8d68-58784043ea19.png)

模型看着效果图，认得出书、光、立即穿越。以前接着就要它把 `anchoredPosition` 填进预制体。填错差的是半个面板，而且树还没看过就可能已经存盘。这套流程把这件事拆开：认控件还是模型的，进预制体的数和 Sprite 是确认过的。

## 坐标按量到的根算完

像素框画在引导层那张贴图上。`size` 和 `anchoredPos` 用当时舞台根量到的 `rect` 算完再写进节点表。剧本解锁这次量到的是 `720×1640`，和效果图 1:1。根节点是铺满，`sizeDelta=(0,0)`，`anchoredPosition=(0,0)`，矩形跟着父级 Canvas，不把 `720×1640` 存成固定尺寸。确认按钮是 `(353, 72)`、`(0, -466)`，首页按钮左下锚点是 `(64, 235)`。

## 两道确认才进预制体

树没确认，只跑只读骨架，`PrefabUtility.LoadPrefabContents` 读完就 `UnloadPrefabContents`，不保存。资源表没确认，只在 `searchPaths` 里搜图，不创建控件，不写 Sprite。

剧本解锁是你回了两次「确认」：第一次定节点表，第二次定正式图。图都是单张 Sprite，空槽没有用纯色补上。`Title2` 只换了中心图，两侧线仍是预制体里的 `ing_line`。

预制体当时开在 Prefab 模式里，舞台根还叫 `_ArtRef`。写入改的是 `PrefabStage.prefabContentsRoot`，回来再 `save_prefab_stage`。`LoadPrefabContents` 改的是磁盘上另一份，旧舞台存回去会盖掉这次的树。

## 改旧面板只动 diff

根下除了引导层已经有控件，走入口 C。执行的是 diff：`keep` 跳过，没进 `delete` 的不删，不按节点表重建一套同名节点。剧本解锁那次根下只有效果图，所以是入口 A，整张表都是新控件。下次再改这张，就在这棵树上改行。

## 绑定留给 UIBinder

`generateScripts` 是 false。这次停在预制体。节点名已经按约定写好，`[ButtonEffect]ConfirmBtn`、`[ExText]DramaName` 这种。选中面板按 Shift+B，绑定脚本由 UIBinder 生成。

摆界面时模型不用再临场发明锚点和 Sprite 路径。它交出树和资源表，经过自己确认，AI再写。

上一篇：[从 project.md 到确认写入](/posts/unity/ui/screenshotpanel/04-quickstart/)
