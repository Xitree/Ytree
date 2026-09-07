---
title: UIBinder 的核心设计
published: 2026-09-07
pinned: false
description: "用 [Type]Name 命名约定、Data 与 Window 双脚本，以及菜单入口，把 UI 绑定变成可重复生成的流程。"
image: ""
tags: ["Unity", "UIBinder"]
category: UI
draft: false
---

**UIBinder 系列：** [为什么要做](/posts/unity/ui/uibinder/01-uibinder-why/) · 核心设计 · [快速上手](/posts/unity/ui/uibinder/03-uibinder-quickstart/) · [特点总结](/posts/unity/ui/uibinder/04-uibinder-summary/)

## 命名约定：`[Type]Name`

| 节点名 | 解析结果 |
| --- | --- |
| `[Button]CloseBtn` | 字段 `CloseBtnButton`，类型 `Button` |
| `[Text]Title` | 字段 `TitleText`，类型 `Text` |
| `[GameObject]Panel` | 字段 `PanelGameObject`，类型 `GameObject` |
| `[Toggle]Music` | 字段 `MusicToggle`，类型 `Toggle` |

**快捷键(反引号)**

![右键菜单中的快速标记与生成入口，含反引号快捷键](./images/shortcut-key.avif)

**Hierarchy显示效果**

![Hierarchy 中按命名约定标记后的绿色高亮](./images/hierarchy-highlight.avif)

---

## 双脚本：数据与逻辑分离

**每个窗口生成两份脚本:**

| 脚本 | 职责 | 再生成策略 |
| --- | --- | --- |
| `XXXDataComponent.cs`<br/>继承mono | 序列化组件引用字段 + 事件接线（`InitComponent` / `Unbind`） | **整文件覆盖** |
| `XXX.cs`（窗口脚本） | 业务逻辑、事件回调方法 | **增量合并**，只往 `#region UI组件事件` 里补缺失方法，已写代码永不覆盖 |

---

## 菜单入口一览

| 入口 | 功能 |
| --- | --- |
| `Tools/UIBinder/设置` | 生成器配置窗口 |
| `Tools/UIBinder/绑定窗口` | 绑定工作台（拖预制体、拖控件、勾事件） |
| Hierarchy 选中节点按反引号键 | 快速标记：从节点已有组件里选一个，自动改名 `[Type]Name` |
| 右键 `GameObject/UIBinder/生成 DataComponent`（Shift+A） | 只生成数据脚本 |
| 右键 `GameObject/UIBinder/一键生成 Data+Window`（Shift+B） | 数据脚本 + 窗口脚本 |
| 右键 `GameObject/UIBinder/生成 Window 脚本`（Shift+C） | 只生成/增量合并窗口脚本 |

![GameObject 菜单下的 UIBinder 入口](./images/menu-gameobject.avif)

![Tools 菜单下的 UIBinder 设置与绑定窗口](./images/menu-tools.avif)
