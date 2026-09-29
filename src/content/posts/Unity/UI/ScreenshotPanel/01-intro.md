---
title: 截图还原 UGUI 面板
published: 2026-09-28
pinned: false
description: "效果图还原成 UGUI 预制体。视觉认控件，引导层上的 RawImage 当尺子，确认之前不写入。"
image: ""
tags: ["Unity", "UGUI"]
category: UI
draft: true
---

> 一个根据美术出的效果图+预制体内的引导层生成具体的UGUI界面的SKILL

技能目录是 `ugui-panel-from-screenshot`。它把两件事拆开。视觉负责认控件、在效果图上框像素。预制体里已经摆好的那张 `RawImage`（节点名 `_ArtRef`）负责坐标系。人确认两道门闩之后，才经 Unity MCP 写进预制体。

第一道是树。它列出节点、组件、像素框 `texRect`、锚点，以及算完的 `size` 和 `anchoredPos`。你改某一行的框，它重算这一行。已有面板还要一张 diff：哪些不动、哪些改布局、哪些换图、哪些新增、哪些删除。树没确认，只许读。这次读取用固定的只读骨架：预制体没在 Prefab 模式打开时，`PrefabUtility.LoadPrefabContents`（`UnityEditor`，按路径把预制体载入一份临时内容）读完，在 `finally` 里 `UnloadPrefabContents`，不保存，也不换掉你当前的场景。

第二道是资源。它问一句：是否使用工程内正式资源。否，就用纯色 `Image`，字用 `project.md` 里那份字体。是，就在约定目录里搜一次，交出一张映射表。你补空的、改错的。资源表没确认，不创建控件，不写 Sprite。

**比较精准的关键：**

截图告诉AI认清每个控件，预制体中的引导层则告诉AI具体的布局像素

下一篇：[用一张弹窗把坐标算完](/posts/unity/ui/screenshotpanel/02-example/)
