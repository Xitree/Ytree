---
title: UIBinder 快速上手
published: 2026-09-07
pinned: false
description: "从设置、三种标记方式到一键生成脚本，走一遍 UIBinder 的基本流程。"
image: ""
tags: ["Unity", "UIBinder"]
category: UI
draft: false
---

**UIBinder 系列：** [为什么要做](/posts/unity/ui/uibinder/01-uibinder-why/) · [核心设计](/posts/unity/ui/uibinder/02-uibinder-design/) · 快速上手 · [特点总结](/posts/unity/ui/uibinder/04-uibinder-summary/)

## 前置设置

> [!NOTE]
> 主要是适配各个项目的设置

打开 `Tools/UIBinder/设置`（配置持久化在 `UISetting.asset`）

![UIBinder 设置窗口](./images/settings.avif)

**DataComponent / 窗口脚本输出目录**：留空则自动推断

**窗口基类短名**：填项目自己的窗口基类（如 `BasePanel`）时，生成 `override` 生命周期；留空或填 `MonoBehaviour` 则生成原生 `Awake/OnEnable/OnDisable`。

**逻辑脚本形态**：`继承mono`或 `纯C#`。

**优先程序集**：当多个程序集里存在同名类时，决定 UIBinder 应该用哪一个。

---

## 标记节点（三种方式）

1. **快捷键**：Hierarchy 选中节点按反引号键，弹出该节点所有可绑定类型（含 GameObject、Transform 及其实际挂着的组件），选择后自动改名

![按反引号键后弹出的可绑定类型菜单](./images/mark-shortcut.avif)

2. **绑定窗口拖入**：把子控件直接拖进绑定窗口列表，同样弹类型菜单

![UIBinder 绑定窗口，先指定窗口根再拖入子控件](./images/mark-bind-window.avif)

3. **手动改名**：直接把节点改成 `[Button]CloseBtn`

**标记后 Hierarchy 实时染色反馈：**

| 颜色 | 含义 |
| --- | --- |
| 绿色 | 类型可解析且节点上有该组件，绑定有效 |
| 黄色 | 类型可解析但节点缺组件 |
| 红色 | 类型名无法解析（写错了） |

写错类型名时，解析报错会附带编辑距离拼写建议，例如 `[Botton]Close` 会提示"是否想写 Button？"。

---

## 生成脚本

选中窗口根物体，右键 `GameObject/UIBinder/一键生成 Data+Window`（或在绑定窗口点按钮），弹出代码预览窗口，确认后写入文件。Unity 编译完成的那一刻，`[DidReloadScripts]` 回调自动把 `DataComponent` 挂到预制体根上，并按相对路径把每个节点引用填进对应字段。
