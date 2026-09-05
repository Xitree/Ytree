---
title: Unity UI 中的 MVC、MVP 与 MVVM
published: 2026-07-09
pinned: false
description: "用玩家血量 UI 示例对比 Unity 中 MVC、MVP、MVVM 三种分层模式的职责和适用场景"
image: ""
tags: ["Unity", "架构"]
category: UI
draft: false
---

# Unity 中的 MVC / MVP / MVVM

我在整理 Unity UI 架构时，发现 MVC、MVP、MVVM 这三个词很容易被讲得很抽象。真正落到项目里，我最关心的其实不是名字，而是：**Model 和 View 之间到底由谁协调？UI 逻辑放在哪里？后面好不好测试和维护？**

这篇文章用同一个玩家血量系统做例子：显示 HP 条，点击按钮受伤 `-10`，治疗 `+20`。三种模式都实现同一件事，这样更容易看出它们的差异。

核心差异体现在：**谁负责协调 Model 和 View 之间的通信，以及通过什么方式。**

---

## 模式对比速览

| 模式 | 协调者 | View 依赖谁 | View 是否被动 | Unity 典型场景 |
|---|---|---|---|---|
| MVC | Controller | Model + Controller | 否，主动拉取或接收 | 简单功能模块 |
| MVP | Presenter | `IView` 接口 | 是，只暴露接口 | 中型 UI 系统，强调可测试性 |
| MVVM | 数据绑定机制 | ViewModel，通过绑定 | 是，只做绑定 | 响应式 UI，状态复杂 |

如果只是小功能，我通常不会为了模式而模式；但当 UI 状态开始变多、按钮逻辑分散、多个界面共享同一份状态时，分层就会明显降低维护成本。

---

## 一、MVC：Model - View - Controller

### 关系图

```plain
Controller ──写入/读取──▶ Model
    │                        │
    └──调用 UpdateHP()──▶ View
```

Controller 是主动协调者。它直接持有 Model 和 View，收到事件后更新 Model，再主动推送数据给 View。

### 代码

```csharp
// ─────────────────────────────────────────────
// Model：纯数据 + 业务规则，不依赖 Unity
// ─────────────────────────────────────────────
public class PlayerModel
{
    public int HP { get; private set; }
    public int MaxHP { get; private set; }

    public PlayerModel(int maxHP)
    {
        MaxHP = maxHP;
        HP = maxHP;
    }

    public void TakeDamage(int amount) =>
        HP = Mathf.Max(0, HP - amount);

    public void Heal(int amount) =>
        HP = Mathf.Min(MaxHP, HP + amount);
}
```

```csharp
// ─────────────────────────────────────────────
// View：纯显示，向外暴露 UI 元素引用
// 不持有 Model，不含任何业务判断
// ─────────────────────────────────────────────
public class PlayerView : MonoBehaviour
{
    [SerializeField] private Slider hpSlider;
    [SerializeField] private TMP_Text hpText;
    [SerializeField] private Button damageButton;
    [SerializeField] private Button healButton;

    // 把按钮引用暴露给 Controller
    public Button DamageButton => damageButton;
    public Button HealButton => healButton;

    public void UpdateHP(int current, int max)
    {
        hpSlider.value = (float)current / max;
        hpText.text = $"{current} / {max}";
    }
}
```

```csharp
// ─────────────────────────────────────────────
// Controller：挂在同一 GameObject 上
// 持有 Model 和 View，负责全部流程控制
// ─────────────────────────────────────────────
public class PlayerController : MonoBehaviour
{
    private PlayerModel model;
    private PlayerView view;

    private void Awake()
    {
        model = new PlayerModel(maxHP: 100);
        view = GetComponent<PlayerView>();
    }

    private void Start()
    {
        view.DamageButton.onClick.AddListener(OnDamageClicked);
        view.HealButton.onClick.AddListener(OnHealClicked);
        Refresh();
    }

    private void OnDamageClicked() { model.TakeDamage(10); Refresh(); }
    private void OnHealClicked() { model.Heal(20); Refresh(); }

    private void Refresh() =>
        view.UpdateHP(model.HP, model.MaxHP);
}
```

### MVC 要点

MVC 的优点是结构简单，上手成本低，很适合小模块。比如一个背包格子、一个技能按钮、一个简单设置项，用 MVC 直接写通常最快。

它的问题也很明显：Controller 容易膨胀；View 直接暴露 UI 元素给 Controller，耦合较紧；View 难以单独做单元测试。

---

## 二、MVP：Model - View - Presenter

### 关系图

```plain
Presenter ──读写──▶ Model
    ▲
    │ 通过 IPlayerView 接口
    ▼
  View（实现接口，触发事件，被动刷新）
```

MVP 里，View 与 Presenter 通过接口通信。Presenter 不知道也不关心 View 的具体实现，这使得 Presenter 可以用纯 C# 做单元测试，注入一个 Mock View 就能验证逻辑。

### 代码

```csharp
// ─────────────────────────────────────────────
// Model：与 MVC 版完全一致，略
// ─────────────────────────────────────────────
```

```csharp
// ─────────────────────────────────────────────
// IPlayerView：View 契约
// Presenter 只认这个接口，不认 MonoBehaviour
// ─────────────────────────────────────────────
public interface IPlayerView
{
    event Action OnDamageClicked;
    event Action OnHealClicked;

    void UpdateHP(int current, int max);
}
```

```csharp
// ─────────────────────────────────────────────
// Presenter：纯 C# 类（不继承 MonoBehaviour）
// 可以在编辑器外做单元测试
// ─────────────────────────────────────────────
public class PlayerPresenter
{
    private readonly PlayerModel model;
    private readonly IPlayerView view;

    public PlayerPresenter(PlayerModel model, IPlayerView view)
    {
        this.model = model;
        this.view = view;

        view.OnDamageClicked += HandleDamage;
        view.OnHealClicked += HandleHeal;

        Refresh(); // 初始同步
    }

    private void HandleDamage() { model.TakeDamage(10); Refresh(); }
    private void HandleHeal() { model.Heal(20); Refresh(); }

    private void Refresh() =>
        view.UpdateHP(model.HP, model.MaxHP);

    // 记得在 View 销毁时调用，避免内存泄漏
    public void Dispose()
    {
        view.OnDamageClicked -= HandleDamage;
        view.OnHealClicked -= HandleHeal;
    }
}
```

```csharp
// ─────────────────────────────────────────────
// View：MonoBehaviour 实现 IPlayerView
// 只负责转发 UI 事件、接受刷新调用
// ─────────────────────────────────────────────
public class PlayerView : MonoBehaviour, IPlayerView
{
    [SerializeField] private Slider hpSlider;
    [SerializeField] private TMP_Text hpText;
    [SerializeField] private Button damageButton;
    [SerializeField] private Button healButton;

    public event Action OnDamageClicked;
    public event Action OnHealClicked;

    private PlayerPresenter presenter;

    private void Awake()
    {
        damageButton.onClick.AddListener(() => OnDamageClicked?.Invoke());
        healButton.onClick.AddListener(() => OnHealClicked?.Invoke());
    }

    private void Start()
    {
        var model = new PlayerModel(maxHP: 100);
        presenter = new PlayerPresenter(model, this);
    }

    private void OnDestroy() =>
        presenter.Dispose();

    // IPlayerView 实现
    public void UpdateHP(int current, int max)
    {
        hpSlider.value = (float)current / max;
        hpText.text = $"{current} / {max}";
    }
}
```

### MVP 单元测试示例

```csharp
// MockView 模拟 View，不需要 Unity 运行时
public class MockPlayerView : IPlayerView
{
    public event Action OnDamageClicked;
    public event Action OnHealClicked;

    public int LastCurrent { get; private set; }
    public int LastMax { get; private set; }

    public void SimulateDamageClick() => OnDamageClicked?.Invoke();
    public void SimulateHealClick() => OnHealClicked?.Invoke();

    public void UpdateHP(int current, int max)
    {
        LastCurrent = current;
        LastMax = max;
    }
}

// NUnit 测试
[Test]
public void TakeDamage_ReducesHPBy10()
{
    var mock = new MockPlayerView();
    var model = new PlayerModel(100);
    var presenter = new PlayerPresenter(model, mock);

    mock.SimulateDamageClick();

    Assert.AreEqual(90, mock.LastCurrent);
}
```

### MVP 要点

MVP 的最大好处是 Presenter 是纯 C#，单元测试很方便；View 也更被动，只负责把 UI 事件转出去，再接收 Presenter 的刷新调用。

代价是模板代码会变多。每个 View 都要定义接口，字段多时 `UpdateXxx()` 方法也会增多。因此一般会在中型 UI 或需要测试的 UI 逻辑里优先考虑 MVP。

### MVC 和 MVP 到底差在哪

MVC 和 MVP 看起来很像，是因为它们都把数据、界面、协调逻辑拆开了。如果例子写得比较干净，差异会更不明显。

我自己更推荐抓住一个核心点：**协调者认识的是具体 View，还是抽象 View 接口。**

```plain
MVC：
Controller 直接操作具体 View
Controller -> PlayerView -> Button / Slider / Text

MVP：
Presenter 只操作 View 接口
Presenter -> IPlayerView
真实 PlayerView / MockPlayerView 都可以替换进去
```

在 MVC 版本里，`PlayerController` 直接拿到的是 `PlayerView`。它知道 View 里面有 `DamageButton`、`HealButton`，也会直接调用 `view.UpdateHP()`。所以 Controller 和具体 Unity UI 结构绑定得更紧。

在 MVP 版本里，`PlayerPresenter` 只认识 `IPlayerView`。它不关心真实 View 是 `MonoBehaviour`，也不关心按钮、Slider、Text 怎么摆。只要有一个对象实现了 `IPlayerView`，Presenter 就能工作，所以测试里的 `MockPlayerView` 也能直接替换进去。

所以真正的差别不在“谁刷新血条”，而在耦合点：

- MVC 更像“Controller 管具体界面”。
- MVP 更像“Presenter 管抽象界面”。
- MVC 改 UI 结构时，Controller 可能跟着改。
- MVP 只要 `IPlayerView` 不变，Presenter 基本不用动。

如果项目只是一个简单面板，这点差异可能不明显；但 UI 变复杂、需要测试、或者多人协作时，MVP 的接口隔离优势会更容易体现出来。

---

## 三、MVVM：Model - View - ViewModel

### 关系图

```plain
View ──单向绑定──▶ ViewModel (ObservableProperty<T>)
                        │
                      读写
                        ▼
                      Model
```

MVVM 里，View 不主动调用 ViewModel 的刷新方法，而是订阅属性的 `OnChanged` 事件。属性变了，View 自动更新，这就是数据绑定的核心。

### 核心：ObservableProperty\<T\>

```csharp
// ─────────────────────────────────────────────
// 可观察属性：值改变时自动通知订阅者
// ─────────────────────────────────────────────
public class ObservableProperty<T>
{
    private T _value;

    public event Action<T> OnChanged;

    public T Value
    {
        get => _value;
        set
        {
            if (EqualityComparer<T>.Default.Equals(_value, value)) return;
            _value = value;
            OnChanged?.Invoke(_value);
        }
    }

    public ObservableProperty(T initial = default) => _value = initial;
}
```

### 代码

```csharp
// ─────────────────────────────────────────────
// Model：与前两版一致，略
// ─────────────────────────────────────────────
```

```csharp
// ─────────────────────────────────────────────
// ViewModel：纯 C# 类
// 持有 ObservableProperty，View 绑定这些属性
// 对外暴露"命令"方法（Command 模式）
// ─────────────────────────────────────────────
public class PlayerViewModel
{
    private readonly PlayerModel model;

    // View 绑定这两个属性
    public ObservableProperty<float> HPRatio { get; } = new(1f);
    public ObservableProperty<string> HPText { get; } = new("100 / 100");

    public PlayerViewModel(PlayerModel model)
    {
        this.model = model;
        SyncProperties();
    }

    // 对外暴露的"命令"：View 的按钮调用这里
    public void ExecuteTakeDamage() { model.TakeDamage(10); SyncProperties(); }
    public void ExecuteHeal() { model.Heal(20); SyncProperties(); }

    private void SyncProperties()
    {
        HPRatio.Value = (float)model.HP / model.MaxHP;
        HPText.Value = $"{model.HP} / {model.MaxHP}";
    }
}
```

```csharp
// ─────────────────────────────────────────────
// View：MonoBehaviour，只做"绑定"
// 没有任何业务逻辑，一眼就能看清 UI 与数据的对应关系
// ─────────────────────────────────────────────
public class PlayerView : MonoBehaviour
{
    [SerializeField] private Slider hpSlider;
    [SerializeField] private TMP_Text hpText;
    [SerializeField] private Button damageButton;
    [SerializeField] private Button healButton;

    private PlayerViewModel viewModel;

    // 用成员方法保存引用，OnDestroy 时才能正确取消订阅
    private void OnHPRatioChanged(float v) => hpSlider.value = v;
    private void OnHPTextChanged(string v) => hpText.text = v;

    private void Start()
    {
        var model = new PlayerModel(maxHP: 100);
        viewModel = new PlayerViewModel(model);

        // ── 绑定：属性变化 → 更新 UI ──
        viewModel.HPRatio.OnChanged += OnHPRatioChanged;
        viewModel.HPText.OnChanged += OnHPTextChanged;

        // 初始同步（属性赋值时 View 尚未订阅，需手动拉一次）
        OnHPRatioChanged(viewModel.HPRatio.Value);
        OnHPTextChanged(viewModel.HPText.Value);

        // ── 绑定：按钮事件 → 调用 ViewModel 命令 ──
        damageButton.onClick.AddListener(viewModel.ExecuteTakeDamage);
        healButton.onClick.AddListener(viewModel.ExecuteHeal);
    }

    private void OnDestroy()
    {
        viewModel.HPRatio.OnChanged -= OnHPRatioChanged;
        viewModel.HPText.OnChanged -= OnHPTextChanged;
    }
}
```

### MVVM 要点

MVVM 的优点是 View 极度轻量，绑定关系一目了然；ViewModel 同样是纯 C#，也比较适合测试。它很适合状态复杂、多处共享的 UI。

缺点是需要自己实现或引入绑定系统，比如 UniRx、R3、自制 `ObservableProperty` 等。订阅和取消订阅也要有纪律，否则很容易留下内存泄漏。

---

## 进阶：用 UniRx / R3 强化 MVVM 绑定

Unity 生态中 UniRx 或更新的 R3 可以替代手写的 `ObservableProperty`，提供更完整的响应式编程支持：

```csharp
// 使用 R3（ReactiveProperty）
public class PlayerViewModel
{
    private readonly PlayerModel model;

    public ReactiveProperty<float> HPRatio { get; } = new(1f);
    public ReactiveProperty<string> HPText { get; } = new("100 / 100");

    // ...（其余相同）
}

// View 中用 BindTo 扩展方法一行绑定
public class PlayerView : MonoBehaviour
{
    private CompositeDisposable disposables = new();

    private void Start()
    {
        // ...
        viewModel.HPRatio.BindTo(hpSlider).AddTo(disposables);
        viewModel.HPText.BindTo(hpText).AddTo(disposables);
        // AddTo 会在 GameObject 销毁时自动取消订阅
    }

    private void OnDestroy() => disposables.Dispose();
}
```

---

## 总结：选哪个？

我的选择倾向是：

- **MVC**：功能单一的小模块，比如背包格子、单个技能 UI，快速迭代时首选。
- **MVP**：需要为 UI 逻辑写单元测试，或团队规模较大、接口契约能明确分工时。
- **MVVM**：多个 View 共享同一份状态，比如 HUD 和背包同时显示血量；或者 UI 状态变化频繁、分支多，并且项目已经引入 UniRx / R3。

如果项目还在早期，我不会一开始就强行上最复杂的方案。先用简单结构把功能跑通，等 UI 状态和协作复杂度真的上来，再把逻辑从 View 里拆出来。
