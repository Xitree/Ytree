---
title: Unity中协程的原理
published: 2026-07-02
pinned: false
description: "协程的说明"
image: ""
tags: ["Unity", "协程"]
category: 原理
draft: false
---

# 协程
步骤规划伪代码:

1. 先说明：Unity 协程用 IEnumerator，Unity 每帧调用 MoveNext
2. 解释：yield return 的对象会放到 IEnumerator.Current
3. 对应规则：null=下一帧继续；WaitForSeconds=到时间再继续（Unity 记录开始时间/剩余）
4. 给一个“手写状态机 + 简化调度器”的示例，展示 Current 如何驱动恢复

```csharp
// C#
using System;
using System.Collections;
using UnityEngine;

public class CoroutineStateMachineMapping : MonoBehaviour
{
    private void Start()
    {
        // Unity 内部会做类似 StartCoroutine(...) 的调度
        StartCoroutine(DemoCoroutine());
    }

    // 你写的协程
    private IEnumerator DemoCoroutine()
    {
        Debug.Log("A (frame 0)");
        yield return null;                 // 1) 下一帧继续

        Debug.Log("B (next frame)");
        yield return new WaitForSeconds(1);// 2) 等 1 秒再继续

        Debug.Log("C (after 1s)");
    }

    // ----------------------------
    // 下面是“对应到状态机”的核心：yield return X => Current = X; return true;
    // Unity 调度器根据 Current 的类型决定何时再次调用 MoveNext()
    // ----------------------------

    // 手写等价状态机（结构接近编译器生成物）
    private sealed class DemoCoroutine_StateMachine : IEnumerator
    {
        private int _state;        // 0: start, 1: after yield null, 2: after WaitForSeconds, -1: end
        private object _current;   // IEnumerator.Current：存放 yield return 的对象

        public object Current => _current;

        public bool MoveNext()
        {
            switch (_state)
            {
                case 0:
                    Debug.Log("A (frame 0)");
                    _current = null;   // 对应：yield return null;
                    _state = 1;
                    return true;

                case 1:
                    Debug.Log("B (next frame)");
                    _current = new WaitForSeconds(1f); // 对应：yield return new WaitForSeconds(1);
                    _state = 2;
                    return true;

                case 2:
                    Debug.Log("C (after 1s)");
                    _state = -1;
                    return false;

                default:
                    return false;
            }
        }

        public void Reset() => throw new NotSupportedException();
    }

    // ----------------------------
    // 教学用：极简“调度器”示意（不是 Unity 源码，只是帮助理解 Current 的含义）
    // Unity 的实现更复杂，且 WaitForSeconds 的计时由引擎侧处理。
    // ----------------------------
    private sealed class MiniScheduler
    {
        private readonly IEnumerator _routine;
        private float _resumeTime; // 仅用于演示 WaitForSeconds 的“延迟恢复”

        public MiniScheduler(IEnumerator routine) => _routine = routine;

        public bool Tick()
        {
            // 如果上次 yield 了 WaitForSeconds，且时间未到，就不调用 MoveNext
            if (Time.time < _resumeTime) return true;

            if (!_routine.MoveNext()) return false;

            object yielded = _routine.Current;

            // 关键：Unity 就是根据 yielded 的类型做不同等待策略
            if (yielded == null)
            {
                // yield return null：下一帧再继续（这里什么都不做，下一次 Tick 就会继续）
                _resumeTime = 0f;
            }
            else if (yielded is WaitForSeconds wfs)
            {
                // 真实 Unity：WaitForSeconds 的等待逻辑在引擎侧，不一定能在 C# 里读到秒数
                // 这里仅示意：把“恢复时间”设为当前时间 + 1 秒
                _resumeTime = Time.time + 1f;
            }
            else
            {
                // 其他 YieldInstruction / CustomYieldInstruction / IEnumerator 等也有各自规则
                _resumeTime = 0f;
            }

            return true;
        }
    }
}
```

对应关系总结（最重要的点）：

+ **状态机层面**：`yield return X;` 只做三件事  
    1. `Current = X`  
    2. `state = 下一个状态`  
    3. `return true`（把控制权交回调度器）
+ **Unity 调度层面**：Unity 每帧（或满足条件时）再调用 `MoveNext()`；是否“满足条件”取决于 `Current`：
    - `null`：**下一帧继续**
    - `WaitForSeconds(t)`：**过 t 秒继续**（计时由 Unity 引擎处理）
    - `WaitForEndOfFrame`：**本帧渲染结束后继续**
    - `WaitForFixedUpdate`：**下次 FixedUpdate 前后继续**
    - `CustomYieldInstruction`：反复检查 `keepWaiting`
    - `IEnumerator`：嵌套协程（子协程跑完才继续父协程）



**关于停住和继续执行**



步骤规划伪代码:

1. 写一个协程 MyRoutine：A -> yield null -> B -> yield WaitSeconds(1) -> C
2. 手写等价状态机：用 int state + object current 保存“停在哪/交出什么”
3. 在 MoveNext 里用 switch(state) 决定从哪一行“恢复”
4. 用一个最小调度器每帧 Tick：根据 current 决定何时再次调用 MoveNext
5. 打印日志验证：恢复点就是 state 对应的 case

```csharp
// C#
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class HandWrittenStateMachineDemo : MonoBehaviour
{
    private MiniRunner _runner;

    private void Awake() => _runner = new MiniRunner();

    private void Start()
    {
        // 用“手写状态机”启动（等价于启动 MyRoutine() 的编译器生成状态机）
        _runner.Start(new MyRoutine_StateMachine("R1"));
    }

    private void Update()
    {
        Debug.Log($"Update other logic, frame={Time.frameCount}, time={Time.time:F2}");
        _runner.Tick();
    }

    // -------------------- 1) 原本写的协程（对照用，不参与运行） --------------------
    private IEnumerator MyRoutine(string name)
    {
        Debug.Log($"{name}: A");
        yield return null;

        Debug.Log($"{name}: B");
        yield return new WaitSeconds(1f);

        Debug.Log($"{name}: C");
    }

    // -------------------- 2) 自定义可读等待对象（便于演示） --------------------
    private sealed class WaitSeconds
    {
        public readonly float seconds;
        public WaitSeconds(float seconds) => this.seconds = seconds;
    }

    // -------------------- 3) 手写“编译器生成的状态机” --------------------
    // 关键字段：
    // - _state：记录“下次从哪一段继续”
    // - _current：记录“yield return 出去的对象”，给调度器看
    private sealed class MyRoutine_StateMachine : IEnumerator
    {
        private int _state;          // 0=刚开始, 1=yield null 后恢复点, 2=yield WaitSeconds 后恢复点, -1=结束
        private object _current;
        private readonly string _name;

        public MyRoutine_StateMachine(string name)
        {
            _name = name;
            _state = 0;
        }

        public object Current => _current;

        public bool MoveNext()
        {
            // 这段 switch 就是“恢复点”的本质：每次被调度器叫醒，就从对应 case 继续
            switch (_state)
            {
                case 0:
                    Debug.Log($"{_name}: A (state=0) frame={Time.frameCount}");
                    // ---- 对应：yield return null; ----
                    _current = null;   // 把“我要等什么”交给调度器
                    _state = 1;        // 记录：下次恢复要进 case 1（也就是 yield 后面那行）
                    return true;       // 立刻退出 => 协程“停住”，CPU 回到 Update 去跑别的逻辑

                case 1:
                    // 只有当调度器认为“null 等待已满足（下一帧）”才会再次调用 MoveNext，才会走到这里
                    Debug.Log($"{_name}: B (state=1) frame={Time.frameCount}");
                    // ---- 对应：yield return new WaitSeconds(1f); ----
                    _current = new WaitSeconds(1f);
                    _state = 2;        // 下次恢复点：case 2
                    return true;       // 再次退出 => 又“停住”

                case 2:
                    // 只有当调度器认为“1秒已到”才会再次调用 MoveNext，才会走到这里
                    Debug.Log($"{_name}: C (state=2) frame={Time.frameCount}");
                    _state = -1;
                    return false;      // 协程结束，调度器会把它移除

                default:
                    return false;
            }
        }

        public void Reset() => throw new NotSupportedException();
    }

    // -------------------- 4) 最小调度器：展示“怎么停住/怎么恢复” --------------------
    private sealed class MiniRunner
    {
        private sealed class Task
        {
            public IEnumerator routine;
            public float resumeAt;      // Time.time >= resumeAt 才能继续
            public int resumeFrame;     // Time.frameCount >= resumeFrame 才能继续
        }

        private readonly List<Task> _tasks = new();

        public void Start(IEnumerator routine)
        {
            _tasks.Add(new Task { routine = routine, resumeAt = 0f, resumeFrame = 0 });
        }

        public void Tick()
        {
            for (int i = _tasks.Count - 1; i >= 0; i--)
            {
                Task t = _tasks[i];

                // (1) 不满足条件 => 本帧不调用 MoveNext => 协程“停住”
                if (Time.time < t.resumeAt) continue;
                if (Time.frameCount < t.resumeFrame) continue;

                // (2) 满足条件 => 调用 MoveNext => 从 _state 对应 case “恢复”
                bool alive = t.routine.MoveNext();
                if (!alive)
                {
                    _tasks.RemoveAt(i);
                    continue;
                }

                // (3) 读取 Current，决定下次什么时候再“叫醒”它
                object yielded = t.routine.Current;

                // 默认：立刻可继续（这里一般不会用到）
                t.resumeAt = 0f;
                t.resumeFrame = 0;

                if (yielded == null)
                {
                    // yield return null：下一帧恢复
                    t.resumeFrame = Time.frameCount + 1;
                }
                else if (yielded is WaitSeconds ws)
                {
                    // yield return WaitSeconds：到时间恢复
                    t.resumeAt = Time.time + ws.seconds;
                }
                else
                {
                    // 其他：这里简单按下一帧
                    t.resumeFrame = Time.frameCount + 1;
                }
            }
        }
    }
}
```
