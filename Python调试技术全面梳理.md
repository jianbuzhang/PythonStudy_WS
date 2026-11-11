# Python 调试技术全面梳理：从异常、日志、断点到性能、内存、并发与线上故障排查

> **文档定位**：一份可以直接当"调试手册 + 排障清单 + 面试/考试讲义"使用的 Python 工程文档。它不只罗列"有个 pdb 命令"，而是围绕一条主线展开：**先判断问题在哪一层，再设计最小观测点，用假设驱动的实验缩小范围，最后把结论固化成测试、断言、告警或文档**。调试的本质不是"猜"，而是"用最低成本获得最多的确定信息"。
>
> **覆盖范围**：以 CPython **3.8 – 3.13** 为基准，示例在 **3.13.5（Windows 11 x64）** 上实际运行验证；平台相关部分（Linux `ptrace`、core dump、信号）单独标注。涉及 3.9+/3.10+/3.11+/3.12+/3.13+ 的调试相关特性（`ExceptionGroup`、`add_note`、`sys.monitoring`、`FrameLocalsProxy`、彩色 traceback 等）都会显式标注版本。**3.14 的调试变化（PEP 768 远程 attach、pdb 双后端、`set_trace_async` 等）在附录 A 中依据官方文档补充。标注"3.13"的行为在 3.12 及以前可能不同，跨版本排查时务必先确认解释器版本。**
>
> **使用建议**：
> 1. 第一遍读第 1–2 章，建立"问题分层 + traceback 精读"的底层能力——90% 的日常 bug 在这两章就能解决；
> 2. 第二遍按场景精读：交互式调试看第 6–7 章，数据不对/逻辑错看第 8、13 章，慢看第 9 章，内存涨看第 10 章，卡死/乱序看第 11 章，线上故障看第 12 章；
> 3. 写代码之前看第 4–5 章（把 bug 挡在运行前），上线之前看第 12、15 章（把事故挡在发生前）；
> 4. 遇到具体报错，先查第 14 章"错误类型速查"和第 15 章"决策树"，再回到对应章节。
>
> **约定**：代码块中 `>>>` 表示交互式解释器，`(Pdb)` 表示 pdb 提示符，`$` 表示 shell；示例输出中的文件路径、行号、线程地址会随环境变化，**关注格式与含义，不要照抄数值**。

---

## 目录

- 第 1 章 调试的心智模型与工具全景
- 第 2 章 异常与 traceback：读懂程序留下的"第一现场"
- 第 3 章 日志与可观测性：不中断程序的调试
- 第 4 章 断言、契约与防御式编程：把 bug 挡在发生之前
- 第 5 章 静态分析、类型检查与"运行前调试"
- 第 6 章 交互式调试：`pdb` 与 `breakpoint()`
- 第 7 章 IDE 与远程调试：把调试器接到任意进程上
- 第 8 章 运行时自省与动态追踪：`inspect`、`settrace`、`sys.monitoring`、`dis`
- 第 9 章 性能剖析：定位"慢在哪"
- 第 10 章 内存与资源调试：找出"谁在涨"
- 第 11 章 并发、异步与多进程调试
- 第 12 章 线上/生产环境调试：不中断服务的排障
- 第 13 章 测试驱动的调试：把 bug 变成不会复发的测试
- 第 14 章 常见错误类型与经典陷阱案例库
- 第 15 章 决策树、速查表与检查清单
- 附录 A 版本差异速查（调试相关）
- 附录 B 代码片段库
- 附录 C 参考资料
- 附录 D 术语表

---

# 第 1 章 调试的心智模型与工具全景

## 1.1 调试是在"用信息换确定性"

调试之所以让人痛苦，通常不是因为工具不够，而是因为**信息不足就下结论**：看到 `AttributeError: 'NoneType' object has no attribute 'x'` 就去猜"哪里返回了 None"，看到接口 500 就去重启服务。专业调试者做的事情可以概括为一句话：

> **在每一个时间点，选择"能最大程度排除可能性"的下一步动作。**

这个动作可能是加一行日志、可能是打一个断点、可能是写一个最小复现脚本、也可能是 `git bisect` 回退到上一个好版本。**动作本身没有高低之分，关键是它能否证伪你当前的假设。**

由此得到三条基本纪律：

1. **先观察，后修改**：在没有确认根因之前，任何"顺手改一下试试"都是在污染现场；
2. **一次只改一个变量**：同时改三处，即使问题消失了你也不知道是哪一处生效；
3. **让信息可复现**：依赖"上次偶现"的经验不可靠，要把偶现变成稳定复现（固定随机种子、固定输入、固定并发时序、固定环境变量）。

## 1.2 问题分层：先判断"这是哪一类问题"

绝大多数调试走弯路，是因为在错误的层次上找答案。比如"服务 CPU 100%"可能是算法问题（第 9 章）、可能是死循环重试（第 11 章）、可能是日志同步刷盘（第 3 章）、也可能是 GC 抖动（第 10 章）。先分层，再选工具。

| 层次 | 典型问题 | 关键特征 | 首选工具 | 对应章节 |
| :--- | :--- | :--- | :--- | :--- |
| 语法 / 编译期 | `SyntaxError`、`IndentationError`、f-string 表达式错误 | 代码根本没跑起来 | 解释器报错、`python -m compileall`、IDE 实时检查 | 第 2、5 章 |
| 导入 / 环境 | `ModuleNotFoundError`、循环导入、版本冲突、`sys.path` 污染 | "在我机器上是好的" | `python -c "import x; print(x.__file__)"`、`pip list`、`-v`、`-X importtime` | 第 14 章 |
| 运行时异常 | `TypeError`、`AttributeError`、`KeyError`、`IndexError` | 有明确 traceback | 读 traceback、`pdb` post-mortem、异常链 | 第 2、6 章 |
| 逻辑错误 | 结果不对但不报错、边界条件、状态机错乱 | 需要断点/日志/断言 | 断点、`assert`、单测、属性测试 | 第 4、6、13 章 |
| 数据 / 输入 | 编码、时区、浮点精度、空值、越界 | 特定输入才复现 | 最小输入、`repr`、`pytest.mark.parametrize` | 第 13、14 章 |
| 性能 | 慢、CPU 高、延迟抖动 | 功能正确但指标不达标 | `cProfile`、`py-spy`、`timeit` | 第 9 章 |
| 内存 / 资源 | 内存持续涨、FD 泄漏、OOM | 运行越久越糟 | `tracemalloc`、`objgraph`、`psutil` | 第 10 章 |
| 并发 / 异步 | 偶发、死锁、数据竞争、任务丢失 | 无法稳定复现、和时序相关 | `faulthandler`、`py-spy dump`、asyncio debug | 第 11 章 |
| 并发之外的"偶发" | 随机种子、哈希顺序、文件系统顺序、时钟 | 换个进程就变 | 固定 `PYTHONHASHSEED`、排序、注入时钟 | 第 13 章 |
| 线上 / 环境 | 只在生产出现、无法 attach、容器内 | 不能中断服务 | 日志/指标/追踪、`faulthandler`、core dump、`py-spy` | 第 12 章 |

> ⚠️ **最贵的错误是层次错位**：用 `print` 调一个 CPU 热点问题，用重启解决一个内存泄漏，用"加个 try/except 吞掉"掩盖一个数据竞争。工具选错，努力全废。

## 1.3 科学调试六步法

```
① 复现（Reproduce）      → 有没有稳定、可重复的失败路径？
        ↓
② 最小化（Minimize）      → 去掉无关代码/输入/依赖，留下最小失败样本
        ↓
③ 定位（Localize）        → 二分、断点、日志，把范围缩到函数/行/变量
        ↓
④ 假设（Hypothesize）     → 写出"如果 X 成立，那么观察 Y 应出现"
        ↓
⑤ 验证（Verify）          → 做实验证伪；不要用修改代码代替验证
        ↓
⑥ 固化（Solidify）        → 回归测试、断言、监控、文档，防止复发
```

### 1.3.1 复现：把"偶现"变成"必现"

| 偶现来源 | 固定手段 |
| :--- | :--- |
| 随机数 | 记录并固定 `random.seed()` / `numpy.random.seed()`，或直接在测试中传固定种子 |
| 哈希顺序 | `PYTHONHASHSEED=0`（仅排查用）；设计上不要依赖 `set`/`dict` 顺序 |
| 并发时序 | 用 `threading.Event`/`Barrier`/`asyncio.Event` 精确控制执行顺序；加 `sleep` 只是下策 |
| 时间 | 注入 `clock` 函数或用 `freezegun`/`time-machine`，禁止在业务代码直接 `datetime.now()` |
| 文件系统顺序 | `os.listdir`/`glob` 结果显式排序 |
| 环境 | 记录 `python -V`、`pip freeze`、环境变量、OS/架构、locale、时区 |
| 上游数据 | 把线上请求/响应落盘（脱敏）作为 fixture 重放 |
| 内存/资源压力 | 用 `ulimit`、cgroup 限制复现；用故障注入模拟超时和错误 |

### 1.3.2 最小化：删除比添加更有价值

最小复现（minimal reproducer）是调试的核心资产：

- 删除一切与失败无关的 import、配置、装饰器、框架代码；
- 把输入缩到最小（列表从 1000 个元素缩到 2 个）；
- 把异步改成同步、把多线程改成单线程，看问题是否还在；
- 用 `git stash`/注释法定位"哪一行改动引入的"；
- 最小样本一旦成立，**立刻写成测试用例**（第 13 章），失败测试就是最好的 bug 说明书。

### 1.3.3 定位：二分法是最高性价比的工具

- **代码二分**：`git bisect`（配合 `git bisect run pytest tests/test_x.py::test_y` 自动定位引入提交）；
- **数据二分**：对输入列表不断对半，找到触发问题的最小元素；
- **层次二分**：先确认是"前端→网关→服务→DB"哪一段，再进代码；
- **时间二分**：问题从某个时刻开始 → 对比该时刻的发布、配置、数据量、证书过期时间；
- **注释二分**：注释掉一半逻辑，看是否仍失败（适合无版本控制的老脚本）。

### 1.3.4 假设与验证：写出可证伪的句子

差的假设：「可能是缓存的问题。」
好的假设：「如果缓存未命中导致每次请求都查库，那么把 `CACHE_TTL` 从 0 改成 60 后，DB QPS 应该从 800 降到 100 以下，且 p99 下降。」

后者可以直接设计实验：改配置 → 观察指标 → 验证或证伪。**能被观察证伪的假设才有调试价值。**

### 1.3.5 固化：调试的产出不是"修好了"

一次合格的修复至少包含：

1. 一个能稳定复现并失败的测试（修复前红、修复后绿）；
2. 一处断言或类型标注，防止同类错误再次静默通过；
3. 一条日志/指标，让下次出问题时能更快发现；
4. 一段注释或文档，解释"为什么这样写"（而不是"做了什么"）。

## 1.4 四种基本手法：二分、增量、对照、变异

| 手法 | 做法 | 适用场景 | 注意 |
| :--- | :--- | :--- | :--- |
| 二分（Bisect） | 把搜索空间对半切：代码、数据、版本、请求链路 | 大范围定位、回归问题 | 前提是"坏"可判定；边界要可复现 |
| 增量（Incremental） | 从最小可运行骨架开始，逐步加回代码，观察何时失败 | 复杂框架/异步/并发 | 每次只加一个维度 |
| 对照（Control） | 保留一个已知正常的对照组：旧版本、旧数据、另一台机器、另一个账号 | "只在某环境出现" | 对照组的变量要尽可能少 |
| 变异（Mutation） | 主动改坏一处，看测试/告警是否能发现 | 验证测试有效性、验证监控覆盖 | 与变异测试（mutmut）思想相同 |

**对照实验示例**：线上只有部分请求 500。不要直接读代码，先对比成功与失败请求的差异（用户、输入、实例、时间、依赖版本），往往差异本身就是答案。

## 1.5 可调试性设计：让 bug 自己开口

最好的调试是"不需要调试"。代码写的时候就应埋好信息：

| 设计点 | 具体做法 | 好处 |
| :--- | :--- | :--- |
| 错误信息带上下文 | `raise ValueError(f"解析失败: row={i}, raw={raw!r}")` | 一眼定位，不用重现 |
| 结构化日志 | JSON 字段 + `request_id`/`user_id`/`trace_id` | 可检索、可聚合 |
| 关键路径断言 | 入口校验参数、出口校验不变量（第 4 章） | 错误在源头暴露，而不是三层之后 |
| 注入依赖 | 时钟、随机数、HTTP 客户端、ID 生成器都做成可替换 | 可测试、可复现 |
| 故障注入开关 | 环境变量控制"注入延迟/异常/断网" | 提前演练异常路径 |
| 显式的宽严 | 库代码不吞异常，应用层在边界统一处理 | traceback 不被截断 |
| 版本与环境自述 | 启动日志打印版本、commit、配置摘要（脱敏） | 线上问题先确认"跑的是哪份代码" |
| 可观测性三件套 | 日志（Logs）、指标（Metrics）、追踪（Traces） | 从"用户报错"到"哪一行代码"的最短路径 |

> 经验：**日志不是越多越好，而是"出事时刚好够用"**。判断标准：如果线上报了一个错，你能否只靠日志复现出当时的入参、分支和耗时？不能的话，就在出错路径上补一行结构化日志。

## 1.6 工具全景：按层选工具

```
┌──────────────────────────────────────────────────────────────┐
│ 平台/服务层  Sentry、OpenTelemetry、APM、日志平台、告警         │
├──────────────────────────────────────────────────────────────┤
│ 进程/OS 层   py-spy、gdb/lldb、strace、perf、lsof、core dump   │
├──────────────────────────────────────────────────────────────┤
│ 解释器层     faulthandler、sys.settrace、sys.monitoring、pdb    │
├──────────────────────────────────────────────────────────────┤
│ 语言/库层    traceback、logging、assert、warnings、unittest     │
├──────────────────────────────────────────────────────────────┤
│ 编辑/静态层  VS Code、PyCharm、debugpy、ruff、mypy、pyright     │
└──────────────────────────────────────────────────────────────┘
```

| 需求 | 工具 | 一句话说明 |
| :--- | :--- | :--- |
| 看异常从哪来 | 内置 traceback、`pdb.pm()` | 信息密度最高的第一手材料 |
| 看变量当时的值 | `pdb`、IDE 断点、`ic()`/`PySnooper` | 停止世界，逐帧检查 |
| 看程序执行了哪些行 | `sys.settrace`、`sys.monitoring`、`coverage` | 动态追踪，适合"到底走没走这个分支" |
| 看函数耗时 | `cProfile`、`py-spy`、`line_profiler` | 统计剖析 vs 采样剖析 |
| 看内存去哪了 | `tracemalloc`、`objgraph`、`memray` | 分配点 vs 引用链 |
| 看卡在哪 | `faulthandler`、`py-spy dump`、`gdb` | 不修改代码拿到全线程栈 |
| 看线上错误 | 结构化日志、Sentry、OTel | 不中断服务，先取证 |
| 防止 bug 写出 | `ruff`、`mypy`、`pyright`、测试 | 把反馈提前到编辑器/CI |

## 1.7 本章要点与坑清单

**要点**

- 调试 = 用信息换确定性；每一步都应能排除可能性或证伪假设。
- 先分层再选工具：语法/导入/运行时/逻辑/数据/性能/内存/并发/线上，走错层就是浪费时间。
- 六步法：复现 → 最小化 → 定位 → 假设 → 验证 → 固化，缺一不可。
- 最小复现是核心资产，找到后立刻写成失败测试。
- 二分法是通用武器：代码、数据、版本、链路、时间。
- 可调试性是设计出来的：上下文错误信息、结构化日志、断言、依赖注入。

**坑**

- 不看完整 traceback 就开始改代码；只看最后一行，忽略异常链。
- 用"加 try/except"掩盖错误，把问题推迟到更难排查的地方。
- 同时改多处，导致无法判断哪处生效。
- 靠 `sleep` 复现并发问题，结果只是降低了复现概率。
- 在错误的层次上优化：CPU 打满却去优化日志格式。
- 修复后不写回归测试，几周后同一个 bug 以另一种形式回来。

---

# 第 2 章 异常与 traceback：读懂程序留下的"第一现场"

## 2.1 traceback 的解剖学

先看一段真实输出（CPython 3.13.5 实测）：

```text
Traceback (most recent call last):
  File "tb_chain.py", line 12, in main
    load_config("missing.toml")
    ~~~~~~~~~~~^^^^^^^^^^^^^^^^
  File "tb_chain.py", line 7, in load_config
    with open(path, encoding="utf-8") as f:
         ~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
FileNotFoundError: [Errno 2] No such file or directory: 'missing.toml'
```

逐部分解读：

| 部分 | 含义 | 常见误读 |
| :--- | :--- | :--- |
| `Traceback (most recent call last)` | 帧按**从旧到新**排列，最后一项是最内层出错点 | 以为第一行是出错位置 |
| `File "...", line 12, in main` | 该帧的文件、**行号**、函数名 | 忽略了"行号是语句起始行，不一定是实际出错子表达式" |
| 缩进的源码行 | 出错语句的原文（来自 `linecache`） | 动态生成/`exec` 的代码在这里可能只显示 `<string>` |
| `~~~~~~~~~~~^^^^^^^^^^^^` | 3.11+ 的**精确列指示**：`~` 标出调用/表达式范围，`^` 标出真正出错的位置 | 在老版本上没有这段，别以为输出被截断 |
| 最末 `FileNotFoundError: ...` | 异常类型 + 消息 | 只看类型不看消息，或忽略 `[Errno 2]` 这类 errno 信息 |
| `During handling of the above exception...` | 隐式异常链（`__context__`） | 误以为有两个不相关的错误 |
| `The above exception was the direct cause...` | 显式异常链（`raise ... from e`） | 忽略了原始异常才是有价值的根因 |

**读 traceback 的正确顺序**：

1. 先读**最后一行**（异常类型与消息），判断问题类别；
2. 再从**最后一个帧**往回看，找到"你自己代码"的最内层帧——库内部的帧通常是"果"，不是"因"；
3. 注意**行号旁边的列指示**，它常直接告诉你哪个参数是 `None`、哪个属性不存在；
4. 若有异常链，**从最底部的原始异常开始读**（`FileNotFoundError`），再看外层包装异常（`ConfigError`）；
5. 若出现 `During handling...` + 同一个异常重复，通常说明 `except` 块里又抛了异常或重复 `raise`。

> ⚠️ **动态代码的陷阱**：`exec`/`eval`/模板渲染/ORM 拼接出来的代码，traceback 里只有 `<string>` 和错误的行号，`linecache` 也取不到源码。此时要打印生成的源码本身，而不是死盯 traceback。

## 2.2 内置异常层次与选型

```
BaseException
├── SystemExit                     # sys.exit() 抛出，不该被 except Exception 捕获
├── KeyboardInterrupt              # Ctrl+C，同理
├── GeneratorExit                  # 生成器 close()，通常不要捕获
└── Exception
    ├── ArithmeticError            # ZeroDivisionError、OverflowError、FloatingPointError
    ├── AssertionError             # assert 失败（注意 -O 下 assert 被移除）
    ├── AttributeError             # 属性不存在 / 对象为 None 时访问属性
    ├── BufferError
    ├── EOFError
    ├── ExceptionGroup             # 3.11+，异常组（含 BaseExceptionGroup）
    ├── ImportError                # ModuleNotFoundError 是其子类
    ├── LookupError                # IndexError、KeyError
    ├── MemoryError
    ├── NameError                  # UnboundLocalError 是其子类
    ├── OSError                    # FileNotFoundError、PermissionError、ConnectionError...
    ├── ReferenceError             # 弱引用对象已被回收
    ├── RuntimeError               # RecursionError、NotImplementedError、coroutine 相关
    ├── StopAsyncIteration / StopIteration
    ├── SyntaxError                # IndentationError、TabError
    ├── TypeError
    ├── ValueError                 # UnicodeError 是其子类
    └── Warning                    # 各类警告（DeprecationWarning 等）
```

| 场景 | 该抛的异常 | 不该抛的异常 / 反例 |
| :--- | :--- | :--- |
| 参数类型不对 | `TypeError` | `ValueError("type error")` |
| 参数值不合法（类型对） | `ValueError` | `TypeError` |
| 找不到键/索引 | `KeyError`/`IndexError`（或 `LookupError`） | `return None` 让调用方猜 |
| 文件/网络/系统调用失败 | `OSError` 及其子类 | 自定义 `Exception("失败")` 丢失 errno |
| 功能未实现 | `NotImplementedError` | `pass` 静默返回 `None` |
| 内部不变量被破坏 | `AssertionError`（仅用于内部断言） | 用 `assert` 校验用户输入（`-O` 会消失） |
| 业务规则不满足 | 自定义异常（携带字段） | 通用 `Exception` |
| 外部依赖返回错误 | 包装成自定义异常并 `from e` 保留原因 | 直接 `raise` 原始底层异常泄露实现细节 |

**异常设计原则**：

```python
class AppError(Exception):
    """应用层异常基类：便于边界统一处理。"""

class ValidationError(AppError):
    def __init__(self, field: str, value: object, reason: str):
        # 1) 携带结构化字段，便于日志与前端展示
        self.field = field
        self.value = value
        self.reason = reason
        # 2) 消息要能独立看懂
        super().__init__(f"字段 {field!r} 校验失败: {reason} (value={value!r})")
```

- **异常层次要浅**：一层业务基类 + 若干具体异常足够；过深的继承会让 `except` 难以精确。
- **消息里带数据**，但注意脱敏（密码、token、身份证号不进日志）。
- **不要用异常做正常控制流**（例如用 `StopIteration` 表示"没找到"）；异常成本高且会掩盖真正的 bug。
- **不要捕获 `BaseException`**，除非你在写进程级守护逻辑，并且会重新抛出。

## 2.3 异常链：`__context__`、`__cause__` 与 `from None`

这是 Python 调试中最被低估的机制。三种写法，三种输出：

```python
import traceback

def load(path):
    with open(path, encoding="utf-8") as f:   # 抛 FileNotFoundError
        return f.read()

# 写法 A：隐式链（__context__）
try:
    load("missing.toml")
except OSError:
    raise ConfigError("加载配置失败")          # 自动附带 "During handling of the above exception"

# 写法 B：显式链（__cause__，推荐）
try:
    load("missing.toml")
except OSError as e:
    raise ConfigError("加载配置失败") from e   # 输出 "The above exception was the direct cause"

# 写法 C：切断链（抑制上下文，慎用）
try:
    load("missing.toml")
except OSError:
    raise ConfigError("加载配置失败") from None  # 只显示新异常，原始信息丢失
```

| 属性 | 触发方式 | traceback 中的提示 | 使用建议 |
| :--- | :--- | :--- | :--- |
| `__context__` | 在 `except`/`finally` 中抛出新异常，自动设置 | `During handling of the above exception, another exception occurred` | 默认行为，通常保留 |
| `__cause__` | `raise NewError(...) from e` | `The above exception was the direct cause of the following exception` | **跨层包装时首选**，语义明确 |
| `suppress_context=True` | `raise ... from None` | 不显示原始异常 | 仅在原始异常属于实现细节、且确实无帮助时使用 |

```python
try:
    ...
except OSError as e:
    err = ConfigError("加载配置失败")
    err.add_note(f"hint: 检查 --config 或 DSH_CONFIG，当前 cwd={os.getcwd()}")  # 3.11+
    raise err from e
```

> ⚠️ **三种最常见的错误用法**：
> 1. `except Exception: pass` —— 异常消失，问题延后爆发；
> 2. `except Exception as e: raise RuntimeError(str(e))` —— `str(e)` 丢掉 traceback，且丢失异常类型；
> 3. `except Exception as e: raise e` —— 在新位置重新抛出会重置 traceback 起点，看起来像"错误发生在 raise 处"。正确做法是裸 `raise`（保留原 traceback）。

## 2.4 `traceback` 模块：从"打印"到"结构化处理"

生产环境不能只靠 `print_exc()`。常用 API：

| API | 作用 | 备注 |
| :--- | :--- | :--- |
| `traceback.print_exc(limit=None, chain=True)` | 打印当前异常（只能用于 `except` 块内） | 默认写 `sys.stderr` |
| `traceback.format_exc(limit=None, chain=True)` | 返回字符串 | 适合拼日志 |
| `traceback.print_exception(exc, limit=None, chain=True)` | 打印指定异常对象 | **3.10+ 支持单参数**；旧式三参数仍兼容 |
| `traceback.format_exception(exc, ...)` | 返回行列表 | 3.10+ 单参数 |
| `traceback.format_exception_only(exc)` | 只格式化"类型: 消息" | 3.13 增加 `show_group` 参数 |
| `traceback.extract_tb(tb, limit=None)` | 把 traceback 转成 `StackSummary`（帧列表） | 可 JSON 序列化 |
| `traceback.extract_stack(f=None, limit=None)` | 当前调用栈（不含异常） | 打印"谁调用了我" |
| `traceback.walk_tb(tb)` / `walk_stack(frame)` | 逐帧迭代 `(frame, lineno)` | 大栈时省内存 |
| `traceback.TracebackException.from_exception(exc)` | 可序列化/可缓存的异常快照 | 支持 `capture_locals=True` |
| `traceback.clear_frames(tb)` | 清理帧局部变量，打破异常持有的引用 | 长生命周期异常对象的内存泄漏克星 |

**生产级记录方式**：

```python
import logging, traceback, json, sys

logger = logging.getLogger(__name__)

def log_exception(exc: BaseException) -> None:
    # ✅ 方式 1：交给 logging，自动带 traceback
    logger.exception("请求处理失败")

def exception_to_json(exc: BaseException) -> dict:
    # ✅ 方式 2：结构化，便于日志平台聚合与检索
    tb = traceback.TracebackException.from_exception(exc, capture_locals=False)
    return {
        "type": type(exc).__name__,
        "message": str(exc),
        "chain": [f"{f.filename}:{f.lineno}:{f.name}" for f in traceback.extract_tb(exc.__traceback__)],
        "notes": list(getattr(exc, "__notes__", ())),   # 3.11+
    }
```

**四个易错点**：

1. **`str(e)` 不是 traceback**：`logger.error(str(e))` 只留下消息，排查时没有调用栈。用 `logger.exception(...)` 或 `exc_info=True`。
2. **`limit` 截断**：默认 10 层 `StackSummary` 是 `limit=None` 不截断，但某些框架会传 `limit=5`；`pdb` 的 `where` 默认也只显示部分，需要 `bt` 全量。
3. **异常对象持有帧**：把异常存进全局列表/缓存（例如错误收集器）会让该帧的所有局部变量无法释放。必要时 `traceback.clear_frames(tb)`，或只保存 `TracebackException`。
4. **`capture_locals=True` 有泄密风险**：它会把局部变量值写进 traceback 文本（密码、token 可能泄露），线上日志默认不要开。

**保留"谁调用了我"**：有时问题在调用方而不是当前函数：

```python
import traceback

def parse(row):
    if not row:
        stack = "".join(traceback.format_stack(limit=5))
        raise ValueError(f"空行，调用栈:\n{stack}")
```

## 2.5 Python 3.11+：异常能力的三个跃迁

### 2.5.1 精确列位置

3.11 起 traceback 用 `~`/`^` 标出出错的具体子表达式：

```text
  File "demo.py", line 4, in <module>
    print(data["user"]["name"])
          ~~~~~~~~^^^^^^
KeyError: 'user'
```

`^` 指向真正找到/没找到的键或属性，`~` 指向整个表达式范围。这让"哪个键不存在"一目了然，尤其在链式调用中。

### 2.5.2 `add_note()`：给异常补充上下文

```python
try:
    int("abc")
except ValueError as e:
    e.add_note("输入来自 CSV 第 42 行，原始字段: user_id")
    raise
```

输出会在异常消息后追加所有 note（`__notes__` 列表）。适合在不改变异常类型的前提下补充"排查提示"，且不会被 `raise` 丢失。

### 2.5.3 `ExceptionGroup` 与 `except*`

并发/批量场景（如 `asyncio.TaskGroup`、并行校验）会同时失败多个子任务，3.11 引入异常组：

```python
def parse(rows):
    errors = []
    for i, row in enumerate(rows):
        if not row:
            errors.append(ValueError(f"row {i} empty"))
    if errors:
        raise ExceptionGroup("parse failed", errors)

try:
    parse([1, 0, 2, 0])
except* ValueError as eg:          # 注意是 except*，可以同时捕获多个
    print("caught:", eg.message, "count:", len(eg.exceptions))
```

3.13.5 实测输出（层级缩进是格式的一部分）：

```text
  + Exception Group Traceback (most recent call last):
  |   File "probe3.py", line 19, in <module>
  |     parse([1, 0, 2, 0])
  |     ~~~~~^^^^^^^^^^^^^^
  |   File "probe3.py", line 16, in parse
  |     raise ExceptionGroup("parse failed", errors)
  | ExceptionGroup: parse failed (2 sub-exceptions)
  +-+---------------- 1 ----------------
    | ValueError: row 1 empty
    +---------------- 2 ----------------
    | ValueError: row 3 empty
    +------------------------------------
```

要点：

- `except*` 不是 `except`：它匹配组内**所有**匹配类型的子异常，并把未匹配的重新组合成新的 `ExceptionGroup` 继续传播；
- `except ExceptionGroup`（普通 `except`）只能整体捕获，无法按类型分流；
- 不要把异常组"拆开再逐个 raise"当成常规做法——会丢失分组结构；
- 处理完部分子异常后，剩余部分要显式 `raise` 才能继续上报。

## 2.6 异常钩子：最后一道兜底

| 钩子 | 触发时机 | 典型用途 |
| :--- | :--- | :--- |
| `sys.excepthook(type, value, tb)` | 主线程未捕获异常 | 统一上报 Sentry、脱敏、附加环境信息 |
| `threading.excepthook(args)` | 子线程未捕获异常（3.8+） | 否则子线程异常默认只打印、不中断主线程 |
| `sys.unraisablehook(unraisable)` | `__del__`/`weakref` 回调等"无法抛出"的异常（3.8+） | 捕获析构期错误，避免静默泄漏 |
| `sys.excepthook` + `faulthandler` | 致命信号（段错误） | faulthandler 抓 C 层崩溃栈 |

```python
import sys, threading, logging

logger = logging.getLogger("app")

def excepthook(exc_type, exc, tb):
    logger.critical("未捕获异常", exc_info=(exc_type, exc, tb))
    # 可在此上报 Sentry / 写 dump 文件
    sys.__excepthook__(exc_type, exc, tb)   # 保留默认输出

def thread_excepthook(args):
    logger.critical("线程 %s 未捕获异常", args.thread and args.thread.name,
                    exc_info=(args.exc_type, args.exc_value, args.exc_traceback))

def unraisablehook(unraisable):
    logger.error("不可抛出异常: %s", unraisable.err_msg,
                 exc_info=(unraisable.exc_type, unraisable.exc_value, unraisable.exc_traceback))

sys.excepthook = excepthook
threading.excepthook = thread_excepthook
sys.unraisablehook = unraisablehook
```

> ⚠️ 钩子里**不要做重活**：不要访问数据库、不要发同步 HTTP、不要再抛异常。钩子本身失败可能导致进程直接崩溃或信息丢失。先落盘/入队，异步处理。

## 2.7 本章要点与坑清单

**要点**

- traceback 从下往上读：先看异常类型和消息，再找"自己代码的最内层帧"，有异常链时从最底层原始异常读起。
- 3.11+ 的列指示（`~~~^^^`）直接指出出错子表达式，优先利用。
- 跨层包装异常用 `raise ... from e`；不要 `raise RuntimeError(str(e))` 丢失信息。
- 记录异常用 `logger.exception`/`exc_info=True`，不要 `str(e)`。
- 异常对象会持有整帧局部变量；长期保存异常要 `clear_frames` 或只存 `TracebackException`。
- 3.11+ 用 `add_note` 补上下文、用 `ExceptionGroup` 表达并发多失败。
- 用三个钩子兜底：主线程、子线程、析构期异常。

**坑**

- 只看 traceback 最后一行就开始改代码，忽略 `from e` 的原始异常。
- `except` 里裸 `raise e` 重置 traceback；应使用裸 `raise`。
- `except BaseException`/`except:` 捕获 `KeyboardInterrupt`、`SystemExit`，导致程序无法正常退出。
- 在 `finally` 里 `return`/`raise`，静默吞掉正在传播的异常。
- 用 `assert` 校验外部输入：`python -O` 下断言全部消失。
- 把 traceback 存进缓存/全局列表导致内存泄漏。
- 异常消息里塞入密码、token、完整请求体，随日志扩散。

---

# 第 3 章 日志与可观测性：不中断程序的调试

## 3.1 `print` 调试的边界

`print` 不是原罪，但它有明确的适用边界。

**适合 `print` 的场景**：一次性脚本、探索性数据分析、快速确认"这行到底有没有执行"、以及所有断点不可用的环境（生产、容器、无 tty）。

**不适合 `print` 的场景**：需要长期保留的信息（用日志）、需要分级过滤的信息（用日志）、多线程/多进程并发输出（会交错）、需要结构化检索的信息（用日志/追踪）、以及任何要提交到仓库的代码。

**`print` 的正确姿势**：

```python
import sys

# 1) 调试信息写 stderr，避免污染 stdout 的管道/JSON 输出
print(f"[DEBUG] user={user!r}", file=sys.stderr, flush=True)

# 2) 用 !r 而不是直接插值：字符串的空格、换行、None、类型差异一目了然
print(repr(value))          # '1 ' vs 1 vs '1'
print(f"{value=}")          # 3.8+：自动输出 "value=..."
print(f"{value=!r}")        # 带 repr 的 f-string 调试写法

# 3) 打印位置信息，避免"这行是谁打的"
import inspect
frame = inspect.currentframe()
print(f"[DEBUG] {frame.f_code.co_filename}:{frame.f_lineno}: {value!r}", file=sys.stderr)

# 4) 多条信息用分隔符/缩进，保持可读
print("=" * 60)
print(f"payload: {payload!r}")
```

**`print` 的五个坑**：

| 坑 | 现象 | 解决 |
| :--- | :--- | :--- |
| 输出被缓冲 | 程序崩了/被杀，最后几行没打出来 | `print(..., flush=True)` 或 `python -u` |
| 多线程交错 | 两行日志粘在一起 | 用 `logging`（handler 有锁） |
| 编码错误 | Windows 控制台 `UnicodeEncodeError` | `PYTHONIOENCODING=utf-8`、`sys.stdout.reconfigure(encoding="utf-8")` |
| stdout 被占用 | 输出混进 HTTP 响应/JSON 数据 | 调试信息统一走 stderr |
| 忘记删除 | 生产环境刷屏、泄露敏感数据 | 用 `logging.debug` + 级别控制，别用裸 `print` |

> 经验：**临时排查用 `print`，进仓库的用 `logging`，要长期观察的用指标/追踪。**

## 3.2 比 `print` 更好的内置工具

| 工具 | 用途 | 示例 |
| :--- | :--- | :--- |
| `repr()` / `f"{x!r}"` | 区分 `1`/`'1'`/`True`、看到特殊字符 | `repr("a\nb")` → `'a\nb'` |
| `pprint.pprint` | 嵌套结构可读化、自动换行排序 | `pprint.pprint(config, width=100)` |
| `pprint.pformat` | 返回字符串，便于写日志 | `logger.debug("config=%s", pformat(cfg))` |
| `dataclasses.asdict` / `__repr__` | 数据类自动可读 | 默认 `repr` 已含全部字段 |
| `vars(obj)` / `obj.__dict__` | 查看对象属性快照 | `vars(user)` |
| `dataclasses.fields` | 列出字段与默认值 | 排查"字段为什么没赋值" |
| `inspect.signature` | 查看函数真实签名和默认值 | 排查装饰器/回调参数错位 |
| `sys.getsizeof` | 单对象浅层大小 | 注意不统计引用对象 |
| `rich.pretty` / `rich.traceback` | 彩色、可折叠、带局部变量 | `from rich import pretty; pretty.install()` |
| `icecream.ic()` | 打印表达式源码 + 值 + 位置 | `ic(user.name, len(items))` |
| `PySnooper` | 函数级逐行自动日志 | `@pysnooper.snoop()` |

```python
from pprint import pformat
import logging

logger = logging.getLogger(__name__)

def handle(payload: dict) -> None:
    # 结构化、可读、且在 INFO 以下默认不输出
    logger.debug("收到请求 payload=%s", pformat(payload, width=120, sort_dicts=False))
```

## 3.3 `logging` 架构：五个对象与一条传播链

```
Logger（你调用的对象，按名字组成树）
  │  logger.info(...) 先做级别判断 isEnabledFor
  ▼
LogRecord（把消息、参数、异常、线程/进程、时间等打包）
  │  经过 logger.filters，再沿 propagate 向父 logger 传播
  ▼
Handler（决定输出到哪里：Stream/File/Rotating/Queue/Syslog...）
  │  经 handler.filters、Formatter 格式化
  ▼
输出（stderr、文件、socket、队列）
```

| 对象 | 职责 | 常见误区 |
| :--- | :--- | :--- |
| `Logger` | 入口、级别判断、层级命名 | 用 `print` 式字符串拼接而不是 `%s` 惰性格式化 |
| `LogRecord` | 承载一条日志的全部字段 | 不知道可以用 `extra=` 注入自定义字段 |
| `Handler` | 决定去向 | 重复添加 handler 导致同一行打印两次 |
| `Formatter` | 决定格式 | 格式里放重计算/敏感字段 |
| `Filter` | 精细化过滤/注入上下文 | 用 filter 做业务逻辑 |
| `LoggerManager` | 全局注册表 | 库代码里 `basicConfig()` 污染应用 |

**级别选择**：

| 级别 | 数值 | 用途 | 是否默认输出 |
| :--- | :--- | :--- | :--- |
| `DEBUG` | 10 | 诊断细节、变量值、分支走向 | 否 |
| `INFO` | 20 | 正常业务里程碑（启动、请求完成、任务结束） | 是 |
| `WARNING` | 30 | 可恢复的异常、降级、重试 | 是 |
| `ERROR` | 40 | 请求失败、功能不可用，需要人看 | 是 |
| `CRITICAL` | 50 | 进程/服务级故障 | 是 |

**最小可用配置**（脚本/小工具）：

```python
import logging

logging.basicConfig(
    level=logging.DEBUG if __debug__ else logging.INFO,
    format="%(asctime)s %(levelname)-8s %(name)s [%(threadName)s] %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
    force=True,          # 3.8+：覆盖已有配置，避免重复 handler
)
logger = logging.getLogger(__name__)
```

**注意 `basicConfig` 的隐式行为**：如果 root logger 已有 handler，`basicConfig` 默认**什么都不做**（除非 `force=True`）。很多"日志不输出"的问题都源于此——某个库在你之前调用了 `basicConfig` 或 `addHandler`。

## 3.4 配置的分层：从 `basicConfig` 到 `dictConfig`

| 场景 | 推荐做法 |
| :--- | :--- |
| 单文件脚本 | `basicConfig` |
| 应用/服务 | `logging.config.dictConfig` + 配置字典（可从 YAML/JSON/环境变量加载） |
| 库代码 | **只 `getLogger(__name__)`，绝不配置 root、绝不加 handler** |
| 需要动态调整级别 | 暴露管理接口/信号，运行时 `logger.setLevel()` |
| 多进程 | `QueueHandler` + 独立日志进程，或每进程独立文件 |

```python
import logging.config

LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,        # ⚠️ 默认 True，会静默关掉其他库的 logger
    "formatters": {
        "default": {
            "format": "%(asctime)s %(levelname)-8s %(name)s [%(process)d:%(threadName)s] %(message)s",
        },
        "json": {"()": "app.logging.JsonFormatter"},   # 自定义工厂
    },
    "handlers": {
        "console": {"class": "logging.StreamHandler", "formatter": "json", "level": "INFO"},
        "file": {
            "class": "logging.handlers.RotatingFileHandler",
            "filename": "logs/app.log",
            "maxBytes": 50 * 1024 * 1024,
            "backupCount": 5,
            "encoding": "utf-8",
            "formatter": "default",
            "level": "DEBUG",
        },
    },
    "loggers": {
        "app": {"handlers": ["console", "file"], "level": "DEBUG", "propagate": False},
    },
    "root": {"handlers": ["console"], "level": "WARNING"},
}

logging.config.dictConfig(LOGGING)
logger = logging.getLogger("app.orders")
```

> ⚠️ `dictConfig` 的 `disable_existing_loggers` 默认是 `True`，它会禁用配置中未出现的已存在 logger。表现是"某个第三方库的日志突然消失了"。务必显式设为 `False`，或用 `incremental=True` 做增量配置。

## 3.5 调试型日志的八个技巧

### 3.5.1 用 `logger.exception` 记录完整栈

```python
try:
    process(order)
except Exception:
    logger.exception("处理订单失败 order_id=%s", order.id)   # 等价于 exc_info=True
    raise
```

`logger.exception` 只能用在 `except` 块内；其他位置用 `logger.error("...", exc_info=True)` 或 `exc_info=exc`（3.5+ 支持传异常实例）。

### 3.5.2 惰性格式化，别提前拼字符串

```python
# ❌ 即使 INFO 级别未开启，f-string 也已经算完
logger.debug(f"payload={pformat(huge_payload)}")

# ✅ 只有真的要输出时才格式化
logger.debug("payload=%s", pformat(huge_payload))

# ✅ 重计算场景再包一层级别判断
if logger.isEnabledFor(logging.DEBUG):
    logger.debug("dump=%s", expensive_dump())
```

### 3.5.3 用 `extra` 注入结构化字段

```python
logger.info("订单创建", extra={"order_id": order.id, "amount": order.amount, "user_id": user.id})
# Formatter 里使用 %(order_id)s；JSON formatter 会自动带上
```

注意：`extra` 的键不能与 `LogRecord` 内置属性（`message`、`asctime`、`levelname` 等）重名，否则会抛 `KeyError`。

### 3.5.4 用 `contextvars` 贯穿一次请求

```python
import contextvars, logging

request_id: contextvars.ContextVar[str] = contextvars.ContextVar("request_id", default="-")

class RequestIdFilter(logging.Filter):
    def filter(self, record: logging.LogRecord) -> bool:
        record.request_id = request_id.get()
        return True

logger = logging.getLogger("app")
logger.addFilter(RequestIdFilter())

# 入口处设置，线程/协程内自动继承（3.7+）
request_id.set("req-8f3a")
logger.info("开始处理")        # 格式里加 %(request_id)s
```

`contextvars` 在 `asyncio` 中会随任务复制，比 `threading.local()` 更适合异步服务。

### 3.5.5 `stack_info` 与 `stacklevel`：找到"谁调用的"

```python
logger.warning("配置项已废弃", stack_info=True)   # 打印当前调用栈
```

工具函数里用 `warnings.warn(..., stacklevel=2)`，让警告指向调用方而非工具函数本身（详见 4.4）。

### 3.5.6 把警告收进日志

```python
logging.captureWarnings(True)     # warnings.warn 的内容进入 "py.warnings" logger
```

配合 `-W error` 或 `PYTHONWARNINGS=default`，可以让隐藏的 `DeprecationWarning`、`ResourceWarning` 显形。

### 3.5.7 打开 logging 内部错误

```python
logging.raiseExceptions = True    # 默认 True；生产可关，开发务必开
```

默认情况下 handler 抛出的异常会被 logging 吞掉（只打一句 `--- Logging error ---`）。排查"日志为什么格式不对/丢字段"时，先确认它没在报错。

### 3.5.8 用 `QueueHandler` 避免日志阻塞业务

```python
import logging.handlers, queue

q: queue.Queue = queue.Queue(-1)
queue_handler = logging.handlers.QueueHandler(q)
root = logging.getLogger()
root.addHandler(queue_handler)
listener = logging.handlers.QueueListener(q, logging.StreamHandler(), respect_handler_level=True)
listener.start()
# ... 业务 ...
listener.stop()      # 3.12+ 也可作为上下文管理器使用
```

日志落盘/网络发送是 I/O，同步写会拖慢请求；高并发服务应改用队列或异步 handler。

## 3.6 结构化日志：让机器能查，人也能读

**推荐字段规范**（一行一个 JSON）：

| 字段 | 含义 | 示例 |
| :--- | :--- | :--- |
| `ts` | ISO8601 时间戳（带时区） | `2026-11-11T23:40:01.123+08:00` |
| `level` | 级别 | `ERROR` |
| `logger` | logger 名 | `app.orders` |
| `msg` | 人类可读消息 | `创建订单失败` |
| `event` | 稳定的事件名（用于聚合） | `order.create.failed` |
| `trace_id`/`span_id` | 分布式追踪关联 | `4bf92f...` |
| `request_id` | 单次请求关联 | `req-8f3a` |
| `service`/`env`/`version` | 服务、环境、版本 | `order-api`/`prod`/`v1.8.3` |
| `exc_type`/`exc_text` | 异常类型与栈 | `ValueError`/`...` |
| 业务字段 | 便于检索与聚合 | `order_id`、`user_id`、`amount` |

```python
import json, logging, datetime

class JsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        payload = {
            "ts": datetime.datetime.fromtimestamp(record.created).astimezone().isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "msg": record.getMessage(),
            "event": getattr(record, "event", None),
            "trace_id": getattr(record, "trace_id", None),
        }
        if record.exc_info:
            payload["exc_text"] = self.formatException(record.exc_info)
        return json.dumps(payload, ensure_ascii=False)
```

> ⚠️ **日志规范三原则**：① 事件名（`event`）用点分小写、稳定不随文案变；② 可变数据放字段，不要拼进消息文本；③ 敏感字段在 Formatter 层统一脱敏，而不是靠每个调用点自觉。

## 3.7 日志的十个经典坑

| # | 坑 | 表现 | 解决 |
| :--- | :--- | :--- | :--- |
| 1 | 库代码配置 root | 引入库后日志格式全变/重复输出 | 库只 `getLogger(__name__)` |
| 2 | 重复添加 handler | 每行日志打印两次、越来越多次 | `force=True` 或先 `handlers.clear()` |
| 3 | `disable_existing_loggers` | 第三方库日志消失 | 显式设 `False` |
| 4 | 传播重复 | 子 logger 和 root 都输出同一行 | `propagate: False` 或只在一层加 handler |
| 5 | 多进程写同一文件 | 日志交错、轮转互相覆盖 | `QueueHandler`、独立文件、`concurrent-log-handler` |
| 6 | 日志阻塞 | 高并发下 QPS 上不去、延迟毛刺 | 异步/队列 handler、减少同步 I/O |
| 7 | 级别用错 | DEBUG 刷屏生产、ERROR 没人看 | 约定级别语义，CI 检查 |
| 8 | 敏感信息 | 密码/token 进日志 | Formatter 脱敏 + 代码评审 |
| 9 | 异常被吞 | 日志 handler 报错无声 | 开发期 `raiseExceptions=True` |
| 10 | 退出丢日志 | 进程被 kill 时最后几条丢失 | `flush()`、`logging.shutdown()`、`atexit`、`QueueListener.stop()` |

## 3.8 日志之外的"可观测性三件套"

| 支柱 | 回答的问题 | Python 常用工具 | 与调试的关系 |
| :--- | :--- | :--- | :--- |
| Logs（日志） | "发生了什么" | `logging`、`structlog`、`loguru` | 单点证据，含异常栈 |
| Metrics（指标） | "有多少/多快/是否异常" | `prometheus_client`、`statsd`、OTel Metrics | 发现异常趋势，触发告警 |
| Traces（追踪） | "这一次请求慢在哪一段" | OpenTelemetry、Jaeger、Zipkin | 跨服务定位耗时与失败点 |

**关联方式**：让日志携带 `trace_id`，从告警/错误平台点进去，直接跳到那一条链路的所有日志。Sentry 的 breadcrumbs 也是同一思路（自动记录最近 N 条操作，异常发生时一并上报）。

```python
# 伪代码：Sentry 初始化 + 脱敏
import sentry_sdk

sentry_sdk.init(
    dsn=...,
    environment="prod",
    release="order-api@1.8.3",
    traces_sample_rate=0.05,          # 采样，避免全量开销
    before_send=lambda event, hint: scrub(event),   # 统一脱敏
)
```

**原则**：日志负责"细节不可枚举的单次事件"，指标负责"可聚合的趋势"，追踪负责"跨组件的因果链"。三者都有 `trace_id`/`request_id` 才叫可观测，否则只是三堆数据。

## 3.9 日志库选型：标准库、loguru、structlog

| 维度 | `logging` | `loguru` | `structlog` |
| :--- | :--- | :--- | :--- |
| 上手成本 | 中（配置繁琐） | 低（开箱即用） | 中（概念多） |
| 结构化 | 需自定义 Formatter | 需 `serialize=True` | 原生支持 |
| 与第三方库兼容 | 原生 | 可拦截标准库 | 可桥接标准库 |
| 上下文传递 | `extra`/`contextvars` 手写 | `bind()` | `contextvars` 绑定 |
| 适合 | 库、需要标准生态、企业规范 | 脚本、中小服务、快速上线 | 大型服务、日志即事件 |
| 注意 | 配置坑多 | 全局单例、测试需清理 | 学习曲线 |

> 建议：**库和基础设施用标准库 `logging`**（兼容性最好）；应用层可以用 `structlog` 或 `loguru` 提升体验，但团队里要统一，不要两套混用。

## 3.10 本章要点与坑清单

**要点**

- `print` 适合临时排查，且要写 stderr、加 `flush`、用 `repr`。
- 日志的核心是"分级 + 结构化 + 上下文"，而不是"多打几行"。
- Logger 树按 `__name__` 命名，库不配置 root，应用在入口统一配置。
- 惰性格式化（`%s`）、`isEnabledFor`、`QueueHandler` 是性能关键。
- 结构化字段（`event`、`trace_id`、业务 ID）决定日志能不能被检索和聚合。
- 日志、指标、追踪通过 `trace_id` 关联，才是完整可观测性。
- 异常用 `logger.exception`，警告用 `captureWarnings` 收编。

**坑**

- 用 f-string 拼日志，导致 DEBUG 关闭时仍付出格式化成本。
- `dictConfig` 默认 `disable_existing_loggers=True`，静默关闭第三方日志。
- 多进程同时 `RotatingFileHandler` 写同一个文件。
- 把异常对象直接 `%s` 进日志，丢掉 traceback。
- 日志里打印完整请求体/密码/token。
- 只在 `except` 里 `print`，生产没有日志系统，问题无法回溯。

---

# 第 4 章 断言、契约与防御式编程：把 bug 挡在发生之前

## 4.1 `assert` 的正确用途与三个陷阱

```python
def withdraw(account, amount):
    assert amount > 0, f"amount must be positive, got {amount}"
    ...
```

`assert` 的语义是：**这里是我认为不可能为假的不变量；如果为假，说明代码有 bug。** 它不是给用户看的错误处理。

| 陷阱 | 说明 | 后果 |
| :--- | :--- | :--- |
| `python -O` 移除断言 | `-O`/`-OO` 下 `assert` 语句整体消失（连表达式都不执行） | 依赖断言做校验的逻辑在优化模式下失效 |
| 用 `assert` 校验外部输入 | 用户输入、API 响应、文件内容都不该用断言 | 生产优化模式下校验消失，变成安全漏洞 |
| 断言里有副作用 | `assert self.cache.pop(key)` | `-O` 下缓存不会被 pop |
| 断言消息很贵 | `assert ok, expensive_dump()` | 消息只在失败时求值，但失败路径本身可能二次崩溃 |

**正确用法**：

```python
# ✅ 校验内部不变量（只在开发/测试中需要）
def _binary_search(a: list[int], x: int) -> int:
    assert a == sorted(a), "输入必须有序"
    ...

# ❌ 校验外部输入
def create_user(name: str):
    assert isinstance(name, str) and name.strip()   # -O 下失效
```

**替代方案**：对外部输入用显式校验并抛 `ValueError`/`TypeError`；对"绝不能发生"的内部状态，可以保留 `assert`，但要接受它在 `-O` 下消失（这通常可以接受，因为那是 bug，不是业务错误）。

> 经验：**断言是"给我自己看的检查"，异常是"给调用方看的契约"。** 两者的受众不同，不能互换。

## 4.2 防御式编程：fail fast，且失败得清楚

| 原则 | 做法 | 反例 |
| :--- | :--- | :--- |
| 入口校验 | 参数进函数就检查类型/范围/非空 | 一路透传到第 8 层才 `AttributeError` |
| 尽早失败 | 不合法状态立刻抛异常 | 返回 `None` 让调用方猜 |
| 显式优于隐式 | 明确 `raise`，不用静默 `pass` | `except: pass` |
| 不返回魔法值 | 不用 `-1`/`None`/`""` 表示错误 | 调用方忘记检查 |
| 不变量固定 | 对象构造后校验一次，之后不再变 | 每个方法都重复校验 |
| 错误信息可行动 | 说明"期望什么、实际什么、怎么改" | `"invalid input"` |

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Money:
    amount: int          # 以"分"为单位，避免浮点
    currency: str

    def __post_init__(self):
        if not isinstance(self.amount, int):
            raise TypeError(f"amount 必须是 int（分），实际 {type(self.amount).__name__}")
        if self.amount < 0:
            raise ValueError(f"amount 不能为负: {self.amount}")
        if len(self.currency) != 3:
            raise ValueError(f"currency 必须是 3 位 ISO 代码，实际 {self.currency!r}")
```

**错误信息模板**：

```text
<操作> 失败: <期望>, 实际 <实际值>; 上下文: <关键参数>
```

例如：`加载配置失败: 期望文件存在, 实际不存在; path=/etc/app/config.toml, cwd=/srv/app`。

## 4.3 契约：类型标注、文档与"可执行的约定"

| 契约形式 | 检查时机 | 成本 | 适用 |
| :--- | :--- | :--- | :--- |
| 类型标注 + mypy/pyright | 静态（CI/编辑器） | 低 | 所有公开 API |
| docstring（Google/Numpy/Sphinx 风格） | 人工/文档生成 | 低 | 公开 API、复杂函数 |
| `assert` / 显式校验 | 运行时（开发/边界） | 低 | 内部不变量、外部输入 |
| 运行时类型检查（`typeguard`/`beartype`/`pydantic`） | 运行时 | 中 | 边界层、反序列化 |
| 单测/属性测试 | 测试期 | 中 | 核心逻辑 |
| 数据库/配置 schema | 启动时 | 低 | 配置、外部数据 |

**用类型标注表达"可能失败"**：

```python
from typing import Protocol

class Clock(Protocol):
    def now(self) -> float: ...

def retry(fn, *, attempts: int = 3, clock: Clock) -> None:
    ...
```

依赖注入式签名（传入 `clock`、`http_client`、`id_factory`）不仅可测试，还能在调试时用"假实现"隔离问题。

**不要过度防御**：在每个函数里都 `isinstance` 检查一遍，会让代码变成"防御式意大利面"，还掩盖真正的类型错误。正确位置是**系统边界**（HTTP 入参、文件/DB 反序列化、消息队列消息）和**关键不变量**。

## 4.4 `warnings`：不致命的"软断言"

`warnings` 适合表达"现在能用，但将来会坏"或"这个用法可疑"。

```python
import warnings

def old_api(x):
    warnings.warn(
        "old_api() 已废弃，请改用 new_api()",
        DeprecationWarning,
        stacklevel=2,          # 让警告指向调用方，而不是本函数
    )
```

| 能力 | 用法 |
| :--- | :--- |
| 升级为错误 | `python -W error`、`-W error::DeprecationWarning` |
| 按模块过滤 | `warnings.filterwarnings("ignore", category=DeprecationWarning, module="thirdparty")` |
| 只显示一次 | 默认 `default` 过滤器按（消息、类别、模块、行号）去重 |
| 记录到日志 | `logging.captureWarnings(True)` |
| 环境变量 | `PYTHONWARNINGS="error::ResourceWarning"` |
| 开发模式 | `python -X dev` 会启用默认警告过滤（3.7+，实测 `warnoptions=['default']`） |

**`stacklevel` 是调试神器**：工具函数里不写 `stacklevel=2`，警告会指向工具函数内部，读者永远不知道是谁调用的。

## 4.5 运行时校验的代价与选择

| 方案 | 特点 | 开销 | 适合 |
| :--- | :--- | :--- | :--- |
| 手写 `if` + `raise` | 无依赖、最灵活 | 极低 | 关键校验 |
| `pydantic`（v2） | 声明式模型、JSON Schema、错误聚合 | 中（Rust 核心，比 v1 快） | 配置、API 入参/出参 |
| `typeguard` | 按标注做运行时检查 | 中 | 测试、边界层 |
| `beartype` | 装饰器式、开销较低 | 低-中 | 库的公开接口 |
| `attrs`/`dataclasses` 的 `__post_init__` | 构造时校验 | 低 | 领域对象 |

```python
from pydantic import BaseModel, Field, ValidationError

class OrderIn(BaseModel):
    order_id: str = Field(min_length=1, max_length=64)
    amount: int = Field(gt=0, description="金额，单位分")
    currency: str = Field(pattern=r"^[A-Z]{3}$")

try:
    order = OrderIn.model_validate(raw)
except ValidationError as e:
    logger.warning("参数校验失败: %s", e.errors())   # 结构化错误列表
    raise
```

> ⚠️ 运行时校验只应放在**边界**。在内部热路径上对每个对象做 `pydantic` 校验，会把性能拖垮（见第 9 章）。

## 4.6 不变量检查：只在开发期打开

对性能敏感的核心路径，可以用 `__debug__` 做"零成本开关"：

```python
if __debug__:                    # python -O 时整块被优化掉
    _validate_invariants(state)

def _validate_invariants(state) -> None:
    assert state.balance == state.credits - state.debits
```

或者用环境变量控制：

```python
import os
STRICT = os.getenv("APP_STRICT", "0") == "1"

def compute(x):
    if STRICT:
        assert x >= 0, x
    ...
```

**关键**：调试开关本身要能被观测到（启动日志打印 `STRICT=True`），否则"为什么线上行为不一样"会变成新的谜题。

## 4.7 异常处理策略：不要用 `try/except` 掩盖问题

| 策略 | 适用 | Python 实现要点 |
| :--- | :--- | :--- |
| 快速失败 | 参数错误、编程错误 | 直接抛，不捕获 |
| 重试 | 瞬时故障（网络抖动、锁冲突） | 指数退避 + 抖动 + 上限 + 只重试幂等操作 |
| 降级 | 非核心依赖失败 | 返回兜底值并打 WARNING/ERROR，带指标 |
| 熔断 | 下游持续失败 | 连续失败计数 + 半开探测 |
| 边界统一处理 | Web 框架、任务队列 | 在入口处 `try/except`，转成错误响应/任务失败 |
| 隔离 | 插件、用户代码 | 单独进程/线程/`try` 包裹并记录 |

```python
import time, random

def retry_call(fn, *, attempts=3, base=0.2, retry_on=(TimeoutError, ConnectionError)):
    for i in range(attempts):
        try:
            return fn()
        except retry_on:
            if i == attempts - 1:
                raise
            delay = base * (2 ** i) * (0.5 + random.random())   # 退避 + 抖动
            time.sleep(delay)
```

**四条纪律**：

1. `try` 块只包**可能失败的那几行**，不要包整个函数；
2. `except` 要具体，至少不要裸 `except:`；
3. 捕获后要么处理、要么加日志后重抛，**不要静默**；
4. 后台线程/任务里的异常必须有兜底（否则默认行为可能是静默丢弃）。

## 4.8 本章要点与坑清单

**要点**

- `assert` 只用于内部不变量；`-O` 会移除它，不能用于校验外部输入。
- 防御式编程的核心是"边界校验 + fail fast + 错误信息可行动"。
- 用类型标注、docstring、显式校验构建契约；依赖注入让问题可隔离、可复现。
- `warnings` 是软断言，配 `stacklevel` 和 `-W error` 能提前暴露问题。
- 运行时校验放边界，不要放热路径。
- 异常处理策略要显式：失败、重试、降级、熔断、隔离，各有适用场景。

**坑**

- `assert` 校验用户输入，`python -O` 后校验消失。
- 吞异常（`except: pass`）导致问题延后爆炸。
- 在每个函数里重复 `isinstance`，掩盖真正的类型错误。
- `try` 包住整个函数，把无关异常也吞掉。
- 重试不设上限、不区分幂等，把一次故障放大成雪崩。
- 警告不设 `stacklevel`，永远定位不到调用方。

---

# 第 5 章 静态分析、类型检查与"运行前调试"

## 5.1 静态分析能抓什么、抓不到什么

静态分析不执行代码，只从源码/语法树/类型信息里找问题。它的价值是**把反馈提前到编辑器里**，成本远低于线上排障。

| 能稳定抓到 | 难以抓到 |
| :--- | :--- |
| 未定义变量、未使用 import/变量 | 运行期才确定的动态属性（`getattr(obj, name)`） |
| 参数个数/关键字错误（类型检查器） | 业务逻辑错误（算错、状态机错乱） |
| `None` 参与运算/属性访问（`Optional` 未收窄） | 并发竞争、死锁、时序问题 |
| 拼写错误的属性（有类型标注时） | 依赖外部数据的边界（越界、空值） |
| 明显的安全模式（硬编码密码、`pickle`、`shell=True`） | 性能问题 |
| 不可达代码、死代码、复杂度过高 | 与环境相关的 bug（版本、路径、编码） |
| 资源未关闭的部分模式（`with` 缺失） | 需要理解业务语义的问题 |

> 结论：**静态分析负责"消灭低级错误和类型错误"，动态调试负责"理解运行时状态"，两者不能互相替代。**

## 5.2 工具链全景

| 工具 | 定位 | 典型命令 | 建议 |
| :--- | :--- | :--- | :--- |
| `ruff` | 极快的 linter + formatter（Rust） | `ruff check .`、`ruff format .` | **首选**，替代 flake8/isort/pyupgrade 等 |
| `flake8` + 插件 | 经典 linter 框架 | `flake8 --max-line-length=100` | 老项目/需要特定插件时 |
| `pyflakes` | 只做逻辑错误检查 | `pyflakes .` | 轻量、误报少 |
| `pylint` | 检查最全、可定制 | `pylint src/` | 规则多但慢，适合作为补充 |
| `mypy` | 类型检查（PEP 484） | `mypy --strict src/` | 服务端/库的首选 |
| `pyright` / Pylance | 类型检查（微软，TS 实现） | `pyright` | VS Code 默认体验好，速度快 |
| `bandit` | 安全漏洞扫描 | `bandit -r src/` | CI 必跑 |
| `semgrep` | 自定义规则的模式匹配 | `semgrep --config auto` | 团队规范检查 |
| `vulture` | 死代码检测 | `vulture src/` | 清理前先人工确认 |
| `radon` / `xenon` | 复杂度度量 | `radon cc -s -a src/` | 复杂度门禁 |
| `deptry` | 依赖问题（未使用/缺失） | `deptry .` | 依赖治理 |
| `compileall` | 语法/字节码编译检查 | `python -m compileall -q src/` | CI 第一道关 |

## 5.3 `ruff`：现代 Python 项目的默认选择

`ruff` 单工具覆盖 pycodestyle（E/W）、pyflakes（F）、isort（I）、pyupgrade（UP）、bugbear（B）、simplify（SIM）、comprehensions（C4）、pylint 子集（PL）、bandit 子集（S）、print 检查（T20）等数百条规则。

**`pyproject.toml` 配置示例**：

```toml
[tool.ruff]
line-length = 100
target-version = "py311"
src = ["src", "tests"]

[tool.ruff.lint]
select = [
    "E", "W",    # pycodestyle
    "F",         # pyflakes：未定义名、未使用导入
    "I",         # isort：导入顺序
    "UP",        # pyupgrade：用新语法
    "B",         # bugbear：易错写法
    "SIM",       # 简化
    "C4",        # 推导式
    "S",         # bandit 安全规则
    "T20",       # 禁止 print（库代码）
    "RUF",       # ruff 自有规则
]
ignore = [
    "E501",      # 行宽交给 formatter
    "B008",      # 允许函数默认参数里调用（如 Depends(...)）
]

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101", "T20"]     # 测试里允许 assert / print
"scripts/**" = ["T20"]

[tool.ruff.lint.isort]
known-first-party = ["myapp"]
```

**高价值规则举例**：

| 规则 | 检查内容 | 为什么重要 |
| :--- | :--- | :--- |
| `F821` | 未定义的名字 | 拼写错误/漏 import，运行必崩 |
| `F401` | 未使用的 import | 掩盖循环导入、增加启动时间 |
| `F841` | 赋值后未使用的局部变量 | 常见"改了但没生效"的 bug |
| `B006` | 可变默认参数（`def f(x=[])`） | 跨调用共享状态（经典坑） |
| `B008` | 默认参数里调用函数 | 只在定义时求值一次 |
| `B904` | `except` 中 `raise` 未 `from` | 丢失异常链 |
| `B905` | `zip()` 未加 `strict=` | 长度不一致被静默截断（3.10+） |
| `S105/S106` | 硬编码密码/密钥 | 安全 |
| `PLR0912` | 分支过多 | 可维护性 |
| `RUF100` | 无用的 `# noqa` | 抑制失效后暴露问题 |

**抑制规则要精确**：

```python
value = cfg["key"]  # noqa: E501 —— 只忽略指定的规则码
```

不要用裸 `# noqa`（会静默忽略该行所有问题）；不要全局 `ignore` 一整个规则族来"让 CI 变绿"。

## 5.4 类型检查：`mypy` 与 `pyright`

### 5.4.1 为什么类型检查对调试价值极高

类型错误是运行时最常见的异常来源之一：`None` 上取属性、传错参数类型、返回类型不一致。类型检查把这些错误从"线上一小时后"提前到"保存文件时"。

```python
def get_user(uid: str) -> dict | None:
    ...

user = get_user("u1")
print(user["name"])      # mypy: Value of type "dict | None" is not indexable
```

### 5.4.2 `mypy` 的分级严格度

```toml
[tool.mypy]
python_version = "3.11"
strict = true                 # 打开下面所有严格选项的快捷方式
warn_unused_ignores = true    # 抑制过期会报错，防止掩盖新问题
warn_redundant_casts = true
no_implicit_optional = true
show_error_codes = true
pretty = true

[[tool.mypy.overrides]]
module = ["thirdparty.*"]
ignore_missing_imports = true
```

| 选项 | 效果 |
| :--- | :--- |
| `strict = true` | 等价于一堆严格选项的组合 |
| `disallow_untyped_defs` | 函数必须写类型 |
| `disallow_any_generics` | 禁止裸 `list`/`dict`，必须 `list[int]` |
| `check_untyped_defs` | 未标注函数也检查 |
| `warn_return_any` | 返回 `Any` 时警告 |
| `strict_equality` | 类型不可重叠的 `==` 报错 |
| `enable_error_code = ["redundant-expr", "truthy-bool"]` | 冗余条件/恒真判断 |

**渐进式落地**：老项目不要一上来 `--strict`，会得到上千个错误。做法是先 `check_untyped_defs`，再逐模块加严格度（per-module overrides），配合 `--ignore-missing-imports` 和 `warn_unused_ignores` 防止抑制腐烂。

### 5.4.3 常用类型工具

```python
from typing import Any, Literal, Protocol, TypeVar, TypedDict, cast, reveal_type

T = TypeVar("T")

class Comparable(Protocol):
    def __lt__(self, other: Any) -> bool: ...

class UserDict(TypedDict):
    id: str
    name: str

def first(items: list[T]) -> T:
    if not items:
        raise ValueError("empty")
    return items[0]

mode: Literal["r", "w"] = "r"

def handle(value: object) -> str:
    reveal_type(value)                 # mypy 会打印推断类型（仅调试用）
    s = cast(str, value)               # 显式断言类型，运行时无检查
    return s
```

| 工具 | 用途 | 注意 |
| :--- | :--- | :--- |
| `Optional[X]` / `X \| None` | 可能为空 | 必须先收窄再使用 |
| `Union` / `X \| Y` | 多种类型 | 用 `isinstance` 收窄 |
| `Literal` | 限定字面量取值 | 适合模式/状态参数 |
| `TypedDict` | 字典的键类型 | 与 `dict[str, Any]` 相比更安全 |
| `Protocol` | 结构化子类型（鸭子类型） | 不要求继承，适合依赖注入 |
| `cast` | 强制断言类型 | 无运行时检查，滥用等于关闭类型检查 |
| `# type: ignore[code]` | 精确抑制 | 必须带错误码；`warn_unused_ignores` 会清理过期抑制 |
| `assert_never`（3.11+） | 穷尽分支检查 | 新增枚举值时会编译期报错 |
| `TypeIs`（3.13+） | 比 `TypeGuard` 更精确的类型收窄 | 自定义类型守卫 |

### 5.4.4 静态检查常见的"值不回票价"场景

- 重度元编程（`__getattr__` 动态生成、ORM 模型、装饰器改写签名）——需要插件或 stub；
- 大量 `**kwargs` 透传——`TypedDict` + `Unpack`（3.11+）可改善；
- 第三方库无类型（`types-*` stub 包可缓解）；
- 科学计算里的 `numpy` 形状/广播——类型检查管不了，需要运行时断言或测试。

> 原则：**类型检查是"性价比最高的那 80%"，不要为了 100% 而引入大量 `cast`/`ignore` 把价值抵消掉。**

## 5.5 安全扫描：`bandit` 能抓的典型问题

| 检查 | 风险 | 修复 |
| :--- | :--- | :--- |
| `B105/B106` 硬编码密码 | 密钥进仓库 | 环境变量/密钥管理 |
| `B301` `pickle.loads` | 反序列化 RCE | 换 JSON/MessagePack，或签名校验 |
| `B307` `eval` | 任意代码执行 | 用 `ast.literal_eval` 或白名单解析 |
| `B602` `subprocess(..., shell=True)` | 命令注入 | 传列表参数、避免 shell |
| `B506` `yaml.load` | 任意对象构造 | `yaml.safe_load` |
| `B324` `hashlib.md5` | 弱哈希 | `sha256`（非密码场景）/`argon2`（密码） |
| `B201` Flask `debug=True` | 生产暴露调试器 | 生产关闭 debug |
| `B101` `assert` | 优化模式失效 | 只用于内部不变量 |

```bash
bandit -r src/ -x tests/ -ll          # 只看中高风险
semgrep --config p/python --error .   # 社区规则集
```

## 5.6 自定义 AST 检查：把团队规范变成可执行规则

`ast` 模块可以在 20 行内实现项目专属检查，比"口头规范"可靠得多。

```python
import ast, sys
from pathlib import Path

class RuleVisitor(ast.NodeVisitor):
    def __init__(self):
        self.problems: list[str] = []

    def visit_ExceptHandler(self, node: ast.ExceptHandler) -> None:
        if node.type is None:                     # 裸 except:
            self.problems.append(f"{node.lineno}: 禁止裸 except:")
        self.generic_visit(node)

    def visit_Compare(self, node: ast.Compare) -> None:
        for op, comp in zip(node.ops, node.comparators):
            if isinstance(op, (ast.Eq, ast.NotEq)) and isinstance(comp, ast.Constant):
                if comp.value is None:            # == None / != None
                    self.problems.append(f"{node.lineno}: 用 is None 而不是 == None")
        self.generic_visit(node)

    def visit_FunctionDef(self, node: ast.FunctionDef) -> None:
        for default in node.args.defaults:
            if isinstance(default, (ast.List, ast.Dict, ast.Set)):   # 可变默认参数
                self.problems.append(f"{node.lineno}: {node.name} 有可变默认参数")
        self.generic_visit(node)

for path in Path("src").rglob("*.py"):
    tree = ast.parse(path.read_text(encoding="utf-8"), filename=str(path))
    v = RuleVisitor()
    v.visit(tree)
    for p in v.problems:
        print(f"{path}:{p}")

sys.exit(1 if any(True for _ in []) else 0)
```

真实项目里更推荐用 `ruff` 的自定义规则（`flake8-*` 插件生态）或 `semgrep` 规则，但**理解 AST 检查的原理**有助于你写出针对自己项目的检查。

## 5.7 集成：让检查自动发生

```
编辑器保存  →  ruff/mypy 实时诊断（LSP）
   ↓
pre-commit  →  只检查改动的文件（快）
   ↓
CI          →  全量检查 + 测试 + 覆盖率门禁
   ↓
合并前      →  必须绿；禁止新增 ignore/noqa
```

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.9.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.14.0
    hooks:
      - id: mypy
        additional_dependencies: [pydantic]
```

**落地策略**：

1. 先加 `ruff format`（无争议、立刻统一风格）；
2. 再加 `ruff check` 的 `E/F`（几乎无争议）；
3. 加 `mypy` 但只对已标注模块严格；
4. 加 `bandit` 高危规则；
5. 用 `--baseline`/逐目录放开的方式避免"一次引入上千错误导致团队放弃"。

## 5.8 本章要点与坑清单

**要点**

- 静态分析把低级错误和类型错误挡在运行前，是"最便宜的调试"。
- `ruff` 一站式替代 flake8/isort/pyupgrade，优先采用；`mypy`/`pyright` 负责类型。
- 类型检查的价值在于 `Optional` 收窄、参数/返回值契约、`Protocol` 抽象。
- 抑制必须精确到规则码，并用 `warn_unused_ignores` 清理过期抑制。
- 安全扫描（bandit）应进 CI；团队规范可以用 AST/semgrep 变成可执行规则。
- 编辑器 → pre-commit → CI 三层自动执行，才可能长期坚持。

**坑**

- 一上来 `--strict` 导致上千错误，团队直接放弃。
- 用 `cast`/`Any`/`# type: ignore` 批量"修复"，把类型检查变成形式主义。
- 全局 `ignore` 一整个规则族，等于关掉检查。
- 认为"类型检查通过 = 逻辑正确"——类型系统管不了业务语义。
- 只跑 linter 不跑测试；静态分析对动态行为几乎无能为力。

---

# 第 6 章 交互式调试：`pdb` 与 `breakpoint()`

## 6.1 断点调试的本质：暂停世界，逐帧检查

断点调试解决的是"**我需要看到某一刻所有变量的真实值，并且能沿调用栈上下移动**"这类问题，这是日志很难替代的（日志需要提前猜到要打什么）。

`pdb` 是标准库自带的交互式调试器，无需安装、几乎任何环境都能用（包括容器、CI、无 IDE 的服务器）。**它是 Python 调试能力的地基**，即使你平时用 IDE，也应该会 `pdb`。

## 6.2 `breakpoint()`：现代入口

```python
def compute(items):
    total = 0
    for item in items:
        breakpoint()          # 3.7+：等价于 sys.breakpointhook()
        total += item
    return total
```

调用链：

```
breakpoint()  →  sys.breakpointhook()  →  默认实现 pdb.set_trace()
```

控制方式：

| `PYTHONBREAKPOINT` | 行为 |
| :--- | :--- |
| 未设置 | 使用默认 `pdb.set_trace()` |
| `0` | **完全禁用**，`breakpoint()` 变成空操作（实测：不进入调试器） |
| `pdb.set_trace` | 显式指定（默认值） |
| `mymodule.mydebug` | 调用自定义函数（必须是 `module.callable` 形式） |
| `pudb.set_trace` / `ipdb.set_trace` | 切换到增强调试器 |

**为什么生产环境要设 `PYTHONBREAKPOINT=0`**：避免误留的 `breakpoint()` 让服务在无人值守时挂起等待输入。CI 中也应设置该变量，并配合 lint 规则禁止提交 `breakpoint()`（`ruff` 的 `T100` 可查 `breakpoint`）。

**`breakpoint()` 的版本差异（官方文档确认）**：

- 3.13 起：`pdb.set_trace()` **立即**进入调试器，而不是停在"下一条要执行的代码"处；
- 3.13 起：通过 pdb 给名字赋值会立即影响活动作用域（PEP 667 的副作用，对调试是好事）；
- 3.14 起：`pdb` 支持 `'settrace'` 和 `'monitoring'` 两种后端，`breakpoint()`/`set_trace()` 固定使用 `monitoring` 后端；`pdb.set_trace(commands=[...])` 可在进入时执行命令序列。

**`breakpoint()` 的三个坑**：

1. 某些环境（daemon 线程、无 stdin 的容器、`pythonw`）中会直接报错或阻塞；
2. 在 Jupyter/IPython 中的行为取决于后端，可能变成 `ipdb`；
3. 用 `q` 退出时抛出 `bdb.BdbQuit`——它是 `Exception` 的子类，**会被外层宽泛的 `except Exception` 捕获**，导致"调试器退不出来/异常被吞"；不要把 `breakpoint()` 包在大范围 `try/except Exception` 里（3.14 的 inline 模式改为确认后直接 `sys.exit()`）。

## 6.3 六种启动方式

| 方式 | 命令/代码 | 适用 |
| :--- | :--- | :--- |
| 从第一行开始调试脚本 | `python -m pdb script.py` | 快速看脚本执行流程 |
| 调试模块/包 | `python -m pdb -m mypackage.module` | 入口是模块时 |
| 代码内断点 | `breakpoint()` / `import pdb; pdb.set_trace()` | 精确定位 |
| 先跑后停 | `python -m pdb -c continue script.py` | 直接进入异常现场（实测会在未捕获异常处进入 post-mortem） |
| 附加到运行中的进程 | `python -m pdb -p <pid>`（**3.14+**，基于 PEP 768） | 不中断服务地 attach；需目标进程到达下一个安全点 |
| 程序结束后进交互 | `python -i script.py` | 检查最终状态、手动调用函数 |
| 异常后现场调试 | `pdb.pm()` / `pdb.post_mortem(tb)` | 已在 except 块中，想看当时的栈 |

**`python -m pdb` 的 CLI 参数**（3.13 实测 `--help`；3.14 增加 `-p`）：

```text
usage: pdb [-h] [-c command] (-m module | -p pid | pyfile) [args ...]
  -c, --command command   pdb commands to execute as if given in a .pdbrc file
  -m module               debug a module/package
  -p, --pid pid           attach to a running process (added in 3.14)
Initial commands are read from .pdbrc files in your home directory
and in the current directory; commands supplied with -c are executed after them.
```

## 6.4 一个真实的 `pdb` 会话

测试脚本（故意写错循环边界）：

```python
# demo.py
def average(rows):
    total = 0
    for i in range(len(rows) + 1):     # ← bug: 多迭代一次
        total += rows[i]
    return total / len(rows)

def main():
    data = [10, 20, 30]
    print("avg =", average(data))

main()
```

会话（命令按输入顺序整理；真实终端会回显你输入的命令）：

```text
$ python -m pdb demo.py
> demo.py(1)<module>()
-> def average(rows):
(Pdb) b 3
Breakpoint 1 at demo.py:3
(Pdb) c
> demo.py(3)average()
-> for i in range(len(rows) + 1):
(Pdb) p len(rows)
3
(Pdb) n
> demo.py(4)average()
-> total += rows[i]
(Pdb) p i
0
(Pdb) display i                      # 每次停下自动显示 i（设置时先打印当前值）
display i: 0
(Pdb) n
> demo.py(3)average()
-> for i in range(len(rows) + 1):
(Pdb) n
> demo.py(4)average()
-> total += rows[i]
display i: 1
(Pdb) p rows
[10, 20, 30]
(Pdb) cl 1                           # 删掉断点，否则 c 会再次停在 b 3
(Pdb) c
...
IndexError: list index out of range
Uncaught exception. Entering post mortem debugging
Running 'cont' or 'step' will restart the program
> demo.py(4)average()
-> total += rows[i]
(Pdb) p i
3                                    # ← i == 3，但 rows 只有 3 个元素：根因确认
```

**读会话的要点**：

- `>` 行显示当前停在哪个文件、哪一行、哪个函数（`<module>` 表示模块顶层）；
- `->` 显示即将执行的源码行（不是已经执行的）；
- 最后进入 **post-mortem**（死后验尸）模式，说明未捕获异常被 pdb 接管，此时栈和变量都还在。

## 6.5 `pdb` 命令全表

### 6.5.1 执行控制

| 命令 | 缩写 | 作用 | 备注 |
| :--- | :--- | :--- | :--- |
| `step` | `s` | 单步，**进入**函数 | 会进入被调用函数 |
| `next` | `n` | 单步，**不进入**函数 | 最常用 |
| `return` | `r` | 运行到当前函数返回 | 适合快速跳出 |
| `continue` | `c`/`cont` | 继续运行到下一个断点/异常 | |
| `until` | `unt` | 运行到下一行（或指定行） | 跳出循环的利器 |
| `jump` | `j` | 跳转到指定行执行 | **仅最底层帧**，不能跳进循环体/跳出 `finally` |
| `run` / `restart` | | 重新运行 | 会丢失当前状态 |
| `quit` / `exit` | `q` | 退出调试器 | 抛 `bdb.BdbQuit` |
| `debug` | | 进入递归调试器执行一条语句 | 调试"求值过程" |
| `interact` | | 进入交互式解释器（共享当前命名空间） | 可执行复杂操作，`Ctrl+D` 返回 |

### 6.5.2 断点管理

| 命令 | 缩写 | 作用 | 示例 |
| :--- | :--- | :--- | :--- |
| `break` | `b` | 设置断点 | `b 42`、`b func`、`b file.py:42` |
| `tbreak` | | 临时断点（命中一次后删除） | `tbreak 42` |
| `condition` | | 条件断点 | `condition 1 i == 3 and rows[i] is None` |
| `ignore` | | 忽略断点前 N 次命中 | `ignore 1 100`（跳过前 100 次） |
| `commands` | | 命中时自动执行命令序列 | 配合 `silent` + `continue` 做"日志断点" |
| `clear` | `cl` | 删除断点 | `cl 1`、`cl`（无参删除全部） |
| `disable` / `enable` | | 禁用/启用断点 | `disable 2` |

**日志断点（不中断执行的"断点"）**：

```text
(Pdb) b 42
Breakpoint 1 at demo.py:42
(Pdb) commands 1
(com) silent
(com) p f"i={i}, rows[i]={rows[i]!r}"
(com) continue
(com) end
```

这样每次命中只打印一行、不停止。它比在代码里加 `print` 更灵活（不改代码、可随时删除），适合"循环里想看某几次"的场景。

### 6.5.3 查看状态

| 命令 | 缩写 | 作用 | 示例 |
| :--- | :--- | :--- | :--- |
| `list` | `l` | 显示当前行附近源码 | `l`、`l 1,50`、`l func` |
| `longlist` | `ll` | 显示整个函数源码 | |
| `print` | `p` | 求值并打印（用 `repr`） | `p rows`、`p {k: v for k, v in d.items()}` |
| `pp` | | 美化打印（`pprint`） | `pp config` |
| `!` | | 执行任意语句（可能有副作用） | `!rows.append(99)` |
| `where` | `w`/`bt` | 打印调用栈 | 看"谁调用了我" |
| `up` | `u` | 上移一帧（到调用者） | 检查调用方变量 |
| `down` | `d` | 下移一帧 | |
| `args` | `a` | 打印当前函数的参数 | 快速看入参 |
| `retval` | `rv` | 打印上一次返回的值 | 配合 `r` 使用 |
| `whatis` | | 打印表达式类型 | `whatis x` |
| `source` | | 显示对象的源码 | `source myfunc` |
| `exceptions` | | 列出/切换异常链中的当前异常（3.13 新增） | `exceptions`、`exceptions 1`（配合 `pdb.pm()`/`post_mortem` 处理链式异常） |
| `display` | | 每次停下自动显示表达式 | `display i`、`undisplay 1` |
| `alias` | | 自定义命令别名 | `alias pi for k in %1.__dict__: print(k, "=", %1.__dict__[k])` |

**`p` 与 `!` 的区别**：`p` 只能求值（表达式），`!` 可以执行语句（赋值、调用、`import`）。`p x = 1` 会报错，`!x = 1` 可以。

**3.12+ 的便捷变量（convenience variables）**：以 `$` 开头的变量在调试会话内有效，恢复执行时自动清除，不会污染被调试程序：

| 变量 | 含义 |
| :--- | :--- |
| `$_frame` | 当前帧 |
| `$_retval` | 帧正在返回时的返回值 |
| `$_exception` | 帧正在抛出的异常 |
| `$_asynctask` | 在异步函数中停下时的 asyncio 任务（3.14+） |

**`display` 的一个坑**：它比较的是"每次求值的结果对象"，对可变对象（列表、字典）可能检测不到原地修改。想观察列表变化，用切片/拷贝表达式：`display lst[:]`。

**修改变量继续跑**：`!rows.append(30)`、`!i = 0`，然后 `c`。这是断点调试相对日志的最大优势——**可以改变程序状态做假设实验**（注意副作用，别把现场搞乱）。

## 6.6 post-mortem：异常现场的"死后验尸"

程序崩溃后再进调试器，栈和局部变量仍在（只要进程没退出）：

```bash
# 方式 1：直接跳到未捕获异常处（实测输出见下）
python -m pdb -c continue script.py
```

```text
Uncaught exception. Entering post mortem debugging
Running 'cont' or 'step' will restart the program
> err.py(2)f()
-> return 1 / 0
(Pdb) p 1/0
*** ZeroDivisionError: division by zero
(Pdb) bt
...
```

```python
# 方式 2：在顶层捕获后手动进入
import pdb, sys

def main():
    ...

try:
    main()
except Exception:
    pdb.post_mortem(sys.exc_info()[2])   # 进入异常发生处的栈
```

```python
# 方式 3：pdb.pm()（使用 sys.last_traceback，需 -i 或交互环境）
import pdb
pdb.pm()
```

**注意**：post-mortem 只能看到"异常发生时"的栈；如果异常被 `except` 吞掉、或进程已退出，就无法使用。这也是"不要吞异常"的又一个理由。

**版本增强**：3.13 起 `pdb.post_mortem()` 可以直接接收**异常对象**（例如 `except ... as e: pdb.post_mortem(e)`），并能用 `exceptions` 命令在 `__cause__`/`__context__` 链之间切换，专门用于排查"包装异常掩盖根因"的场景；3.14 起还提供 `await pdb.set_trace_async()`，可在协程里安全进入调试器。

## 6.7 `pdb` 的局限与应对

| 局限 | 表现 | 应对 |
| :--- | :--- | :--- |
| 只跟踪当前线程 | 多线程程序里其他线程继续跑，问题现场在变 | 用 `faulthandler`/`py-spy dump` 看全线程栈；或在目标线程内 `pdb.set_trace()` |
| 阻塞事件循环 | asyncio 里 `pdb` 停在协程里，其他任务全部停摆 | 用 asyncio debug 模式 + 日志，或 `aiomonitor`、IDE 的异步调试 |
| 看不到 C 扩展内部 | 栈只到 Python 层 | 用 `gdb`/`lldb` + CPython 辅助命令（见 7.4） |
| 无 tty 环境不可用 | 容器、CI、远程 | `remote-pdb`、`debugpy` attach、VS Code Remote |
| 热循环里断点太多 | 反复停下，效率极低 | 条件断点、`ignore`、`display`、日志断点 |
| 不能调试"已经过去"的状态 | 只在暂停点有效 | 日志、追踪、异常快照、`faulthandler` |

## 6.8 增强调试器

| 工具 | 安装 | 特点 | 启动方式 |
| :--- | :--- | :--- | :--- |
| `pdbpp`（`pdb++`） | `pip install pdbpp` | 粘性源码（sticky）、`ll`、`watch`、`hf`/`tb` 隐藏帧、`interact` 增强 | `PYTHONBREAKPOINT=pdb.set_trace` 或 `pdbpp.set_trace` |
| `pudb` | `pip install pudb` | 终端 GUI，可看变量窗口、栈、源码 | `python -m pudb script.py` |
| `ipdb` | `pip install ipdb` | IPython 语法、补全、彩色 | `import ipdb; ipdb.set_trace()` |
| `IPython.embed()` | 装 IPython | 完整 IPython 环境 | `from IPython import embed; embed()` |
| `remote-pdb` | `pip install remote-pdb` | 通过 telnet 连入容器内进程 | `python -m remote_pdb ...` 或代码中 `set_trace(host=...)` |
| `web-pdb` | `pip install web-pdb` | 浏览器调试 | `python -m web_pdb` |
| `birdseye` | `pip install birdseye` | 可视化记录每个表达式的值 | 装饰器 |
| `PySnooper` | `pip install PySnooper` | 逐行打印变量变化 | `@pysnooper.snoop()` |
| `snoop` | `pip install snoop` | 类似 PySnooper，更现代 | `@snoop` |

```python
# PySnooper：不想打断点，只想看函数里每行变量的变化
import pysnooper

@pysnooper.snoop(watch=("i", "rows[i]"), depth=2)
def average(rows):
    total = 0
    for i in range(len(rows) + 1):
        total += rows[i]
    return total / len(rows)
```

## 6.9 断点调试方法论

**打断点的四个原则**：

1. **打在"信息汇合处"**：函数入口（看全部入参）、分支判断前（看条件）、异常抛出前（看状态），而不是随机行；
2. **能用条件断点就不用循环断点**：`b 42` + `condition 1 i == 999`，比 `ignore` 更直观；
3. **先看栈再看变量**：`bt`/`w` 确认自己在哪个调用路径上，很多 bug 是"被另一条路径调用了"；
4. **一次只验证一个假设**：进断点前列出"我要确认什么"，确认完立即 `c`。

**断点不适合的问题**：

- 性能/时序问题（暂停会改变时序）→ 用剖析器；
- 偶发/并发问题（停下来世界就变了）→ 用 `faulthandler`、`py-spy`、日志；
- 服务不可中断的问题 → 用线上取证手段（第 12 章）；
- 需要观察长时间演化的问题（内存增长）→ 用 `tracemalloc` 快照对比。

## 6.10 本章要点与坑清单

**要点**

- `breakpoint()` 是最现代入口，`PYTHONBREAKPOINT=0` 可在生产/CI 全局禁用。
- 六种启动方式覆盖从"从头调试"到"异常现场验尸"。
- `step`/`next`/`return`/`continue` 是执行控制核心；`bt`/`up`/`down` 是栈导航核心。
- 条件断点 + `commands`（日志断点）能把 pdb 用出日志的效果，且不改代码。
- `post-mortem` 是异常调试的杀手锏，前提是异常没有被吞掉。
- pdb 只跟踪当前线程、会阻塞事件循环，这两点决定了它的适用边界。

**坑**

- 生产代码遗留 `breakpoint()`，导致服务挂起。
- 在 10 万次循环里打无条件断点，然后疯狂按 `c`。
- 用 `n` 单步走进第三方库深处，忘了用 `r`/`c` 跳出。
- `p` 里调用有副作用的函数（如 `p db.commit()`），污染现场。
- 多线程下用 pdb 查竞争，越查越乱。
- 以为 `pdb` 能看到 C 扩展内部，实际上只能到 Python 层。

---

# 第 7 章 IDE 与远程调试：把调试器接到任意进程上

## 7.1 IDE 调试的定位：pdb 的可视化外壳

主流 IDE（VS Code、PyCharm）的 Python 调试能力，底层大多是 `debugpy`（原 `ptvsd`）或 `pydevd`——它们实现了 DAP（Debug Adapter Protocol）或自有的远程调试协议。IDE 提供的是**可视化 + 断点管理 + 变量树 + 远程连接**，核心能力（断点、单步、栈、表达式求值）与 pdb 一致。

| 能力 | pdb | IDE |
| :--- | :--- | :--- |
| 断点/单步/栈 | ✅ | ✅ 更直观 |
| 变量树、监视表达式 | 手输 | ✅ 自动 |
| 条件断点/日志断点 | ✅（命令） | ✅ 图形化 |
| 远程/容器内进程 | 需插件 | ✅ 一等公民 |
| 多线程/多进程 | 有限 | ✅ 支持子进程、attach |
| 无图形环境 | ✅ | ❌ |
| 启动开销 | 极小 | 中（需要调试适配器） |
| 学习成本 | 命令要记 | 低 |

> 建议：**日常用 IDE，线上/容器/无图形环境用 pdb + 其他手段**。两者不是替代关系。

## 7.2 VS Code：`launch.json` 全解

VS Code 的 Python 调试由 `ms-python.debugpy` 扩展提供，`type` 字段在旧版本是 `"python"`，新版本为 `"debugpy"`（两者通常都可用，新配置建议写 `debugpy`）。

### 7.2.1 最小启动配置

```jsonc
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Python: 当前文件",
      "type": "debugpy",
      "request": "launch",
      "program": "${file}",
      "console": "integratedTerminal",
      "justMyCode": true
    }
  ]
}
```

### 7.2.2 常用字段

| 字段 | 作用 | 备注 |
| :--- | :--- | :--- |
| `program` | 入口脚本 | 可用 `${file}`、`${workspaceFolder}` |
| `module` | 以模块方式启动 | 如 `"module": "pytest"`、`"module": "uvicorn"` |
| `args` | 命令行参数（数组） | 每个参数一项，不要拼成一个字符串 |
| `env` / `envFile` | 环境变量 | `envFile: "${workspaceFolder}/.env"` |
| `cwd` | 工作目录 | 影响相对路径、`sys.path[0]` |
| `console` | 输出控制台 | `integratedTerminal` 支持交互输入（`input()`/TUI 必选） |
| `justMyCode` | 只调试自己的代码 | `false` 才能单步进第三方库/标准库 |
| `stopOnEntry` | 启动即暂停 | 配合"从第一行开始看流程" |
| `subProcess` | 是否调试子进程 | 默认 `true`（新版）；多进程程序关键 |
| `redirectOutput` | 捕获输出到调试控制台 | 与 `console` 有交互，TUI 程序可能需关闭 |
| `preLaunchTask` | 启动前任务 | 编译、生成代码、启动依赖 |
| `pathMappings` | 本地/远程路径映射 | **attach 远程时代码对不上的头号原因** |
| `purpose` | 配置用途 | `["debug-test"]`、`["debug-in-terminal"]` |
| `autoReload` | 修改代码后自动重载 | 实验性 |
| `logToFile` | 调试器日志 | 排查"断点不生效"时开启 |

### 7.2.3 调试测试

```jsonc
{
  "name": "Python: 调试当前 pytest 用例",
  "type": "debugpy",
  "request": "launch",
  "module": "pytest",
  "args": ["-x", "-q", "${file}", "-k", "${selectedText}"],
  "console": "integratedTerminal",
  "justMyCode": false,
  "purpose": ["debug-test"]
}
```

> 更简单的办法：在测试函数左侧点"Debug Test"（需要 Python 扩展的测试发现），或直接用 `pytest --pdb`（第 13 章）。

### 7.2.4 attach 到已经运行的进程

```jsonc
{
  "name": "Python: 附加到远程",
  "type": "debugpy",
  "request": "attach",
  "connect": { "host": "127.0.0.1", "port": 5678 },
  "pathMappings": [
    { "localRoot": "${workspaceFolder}", "remoteRoot": "/app" }
  ],
  "justMyCode": false
}
```

**断点"灰色不生效"的排查顺序**：

1. `pathMappings` 的 `localRoot`/`remoteRoot` 是否与实际路径一致（容器里常见 `/app`）；
2. 远程进程是否真的加载了这份文件（`inspect.getfile`/日志打印 `__file__`）；
3. `justMyCode` 是否把目标代码判为"库代码"；
4. 文件是否被 `PYTHONOPTIMIZE`/`-O` 编译（断言消失、行号偏移）；
5. 是否 attach 到了错误的进程/端口（多进程、多副本）。

## 7.3 PyCharm：本地与远程

| 场景 | 配置方式 | 关键点 |
| :--- | :--- | :--- |
| 本地脚本 | Run/Debug Configuration → Python | 与 VS Code 类似 |
| 附加到本地进程 | Run → Attach to Process（Professional） | Linux 需要 ptrace 权限 |
| 远程调试服务器 | `pip install pydevd-pycharm`，配置 "Python Debug Server" | 代码中 `pydevd_pycharm.settrace(host, port, suspend, stdoutToServer, stderrToServer)` |
| 多进程 | 调试配置里勾选 "Attach to subprocess automatically" / 设置 `PYDEVD_USE_CYTHON` | 子进程需要继承调试环境 |
| Docker/远程解释器 | 配置 Remote/Docker Compose 解释器 | 路径映射由 IDE 处理 |
| 科学计算 | Scientific Mode | 变量查看器、DataFrame 视图 |

```python
# PyCharm 远程调试（在远程机器上运行）
import pydevd_pycharm
pydevd_pycharm.settrace("10.0.0.5", port=12345, suspend=True, stdoutToServer=True, stderrToServer=True)
```

## 7.4 `debugpy`：命令行与代码内 attach

`debugpy` 是 VS Code Python 调试器的独立实现，也可以脱离 IDE 使用：

```bash
pip install debugpy

# 1) 启动并等待调试器连接（--wait-for-client 会阻塞，直到 IDE attach）
python -m debugpy --listen 5678 --wait-for-client script.py

# 2) 以模块方式启动
python -m debugpy --listen 127.0.0.1:5678 -m mypackage

# 3) attach 到已经运行的进程（需要目标进程 Python 3.7+，Linux 需 ptrace 权限）
python -m debugpy --listen 5678 --pid 12345
```

```python
# 代码内嵌：适合"只在满足条件时才允许调试"
import debugpy

debugpy.listen(("127.0.0.1", 5678))
# debugpy.wait_for_client()      # 阻塞等待，生产慎用
debugpy.breakpoint()             # 等价于 breakpoint()，但走 debugpy
```

**安全警告**：debugpy 监听端口等价于**远程代码执行入口**。生产环境必须：

- 只监听 `127.0.0.1`，通过 SSH 隧道/`kubectl port-forward` 访问；
- 用环境变量/开关控制是否启用；
- 不要在公网暴露 5678；
- 调试完关闭监听、移除代码。

**Linux attach 失败的常见原因**：`ptrace_scope` 限制（`/proc/sys/kernel/yama/ptrace_scope`）、容器缺少 `SYS_PTRACE` capability、进程以不同用户运行、SELinux/AppArmor 策略。

## 7.5 Jupyter / IPython 调试

| 方式 | 用法 | 说明 |
| :--- | :--- | :--- |
| 事后调试 | `%debug` | 单元格报错后进入 post-mortem |
| 自动事后调试 | `%pdb on` | 之后每次异常自动进入调试器 |
| 单步调试 | `%%debug` | 对整个单元格做逐行调试 |
| 断点 | `breakpoint()` / `from IPython.core.debugger import set_trace` | 取决于后端 |
| 检查变量 | `%who`、`%whos`、`%pdoc`、`%psource` | 交互式自省 |
| 计时 | `%timeit`、`%%time` | 微基准（第 9 章） |

> Jupyter 里 `pdb` 的输入循环与 notebook 前端可能有兼容问题；优先用 `%debug` 或 `ipdb`。

## 7.6 gdb/lldb：调试 CPython 与 C 扩展

当崩溃发生在 C 扩展、或需要看 Python 层之上的原生栈时，Python 调试器无能为力，需要原生调试器。

```bash
# Linux：附加到运行中的 Python 进程
gdb -p $(pgrep -f myapp.py)

(gdb) py-bt              # 当前线程的 Python 栈（需要 python-gdb.py）
(gdb) py-bt-full         # 带更多细节
(gdb) py-locals          # 当前 Python 帧的局部变量
(gdb) py-up / py-down    # 在 Python 帧间移动
(gdb) thread apply all py-bt   # 所有线程的 Python 栈
(gdb) bt                 # 原生 C 栈
```

**前提**：CPython 安装包自带 `python-gdb.py`（源码树 `Tools/gdb/`，部分发行版在 `python3-dbg` 包中），并需要调试符号（`-dbg`/`-debug` 包）。没有它，`py-bt` 未定义。

**core dump 分析**：

```bash
ulimit -c unlimited                  # 允许生成 core
cat /proc/sys/kernel/core_pattern    # 看 core 落到哪
coredumpctl list                     # systemd 系统
coredumpctl gdb <pid>                # 用 gdb 打开对应 core
(gdb) py-bt
```

**Windows**：没有 gdb 生态，可用 `procdump -ma <pid> dump.dmp` 抓转储，再用 WinDbg 分析原生栈；Python 层栈更适合用 `py-spy dump --pid <pid>`（Windows 也支持）或 `faulthandler` 输出。

**什么情况下需要 gdb**：

- 段错误（`SIGSEGV`）、`SIGABRT`、C 层死锁；
- C 扩展（numpy、cryptography、数据库驱动）崩溃；
- 怀疑 GIL 死锁、引用计数错误；
- `faulthandler` 只能给出 Python 栈，无法定位原生栈时。

## 7.7 子进程、多进程与容器

| 场景 | 做法 |
| :--- | :--- |
| `subprocess` 子进程 | VS Code `"subProcess": true`；或在子进程入口加 `breakpoint()` |
| `multiprocessing`（spawn） | 每个子进程是独立解释器，需要单独 attach；或在子进程入口 `debugpy.listen()` |
| `multiprocessing`（fork） | 子进程继承父进程状态，attach 父进程不一定覆盖子进程 |
| Celery/任务队列 | worker 进程启动命令加 `-m debugpy --listen ...` |
| Docker 容器 | `docker exec -it <c> python -m debugpy --listen 0.0.0.0:5678 --pid 1`，端口映射后 attach |
| Kubernetes | `kubectl port-forward pod 5678:5678`，或 `kubectl debug` 起临时容器 + `py-spy` |
| 生产只读环境 | 优先 `py-spy dump`/`faulthandler`（不侵入），attach 调试器是最后手段 |

> ⚠️ 多进程调试的核心难点是"你 attach 的到底是哪个进程"。先用 `ps -ef | grep python`、`py-spy dump --pid` 确认 PID 与进程角色，再动手。

## 7.8 本章要点与坑清单

**要点**

- IDE 调试是 pdb 的可视化外壳，底层是 debugpy/pydevd；无图形环境仍要会 pdb。
- `launch.json` 的关键是 `module`/`args`/`env`/`cwd`/`justMyCode`/`subProcess`/`pathMappings`。
- attach 远程时，`pathMappings` 与 `justMyCode` 是断点不生效的两大原因。
- debugpy 可以脱离 IDE 用：`python -m debugpy --listen ... --wait-for-client`。
- 原生崩溃、C 扩展问题用 gdb/lldb + `py-bt`，core dump 是最后的现场。
- 容器/K8s 里 attach 需要端口转发 + 正确路径映射 + ptrace 权限。

**坑**

- 把 debugpy 端口暴露到公网（等于开 RCE 后门）。
- attach 到错误的进程（多副本/多进程），然后困惑"断点为什么不命中"。
- 用 `justMyCode: true` 然后想单步进标准库/第三方库。
- 容器里代码路径与本地不一致，断点全部灰色。
- 在 TUI 程序里用 `console: internalConsole`，导致输入不可用。
- 用 IDE 调试性能问题：断点和变量采集会显著改变时序。

---

# 第 8 章 运行时自省与动态追踪：`inspect`、`settrace`、`sys.monitoring`、`dis`

## 8.1 自省（Introspection）：运行时问对象"你是谁"

| API | 作用 | 调试用途 |
| :--- | :--- | :--- |
| `type(x)` / `isinstance` | 真实类型 | 发现"以为它是 dict，其实是 list" |
| `dir(x)` | 属性/方法列表 | 找可用方法、发现拼写错误 |
| `vars(x)` / `x.__dict__` | 实例属性字典 | 看对象内部状态 |
| `getattr(x, "name", default)` | 安全取属性 | 避免 `AttributeError` 打断排查 |
| `hasattr` | 属性存在性 | 注意：会吞掉属性访问内部的异常（3.2+ 只捕 `AttributeError`） |
| `inspect.signature(fn)` | 函数真实签名 | 装饰器/回调参数错位 |
| `inspect.getsource(fn)` | 源码 | 确认"运行时执行的到底是哪份代码" |
| `inspect.getfile(fn)` | 定义文件 | 排查"装了两个版本的包" |
| `inspect.getmembers(x, inspect.isfunction)` | 按类型筛选成员 | 插件/钩子排查 |
| `inspect.unwrap(fn)` | 穿过装饰器 | 看被 `functools.wraps` 包装的原函数 |
| `help(x)` / `pydoc` | 文档 | 交互式查 API |

```python
import inspect

def debug_object(obj):
    print("type:", type(obj))
    print("file:", inspect.getfile(type(obj)))
    print("attrs:", [a for a in dir(obj) if not a.startswith("_")])
    print("dict:", getattr(obj, "__dict__", "<no __dict__>"))
    print("slots:", getattr(type(obj), "__slots__", "<no __slots__>"))
```

**"运行时到底加载了哪个文件"是最高频的自省问题**：

```python
import mymodule
print(mymodule.__file__)          # 包的物理路径
print(mymodule.__version__)       # 版本（如果有）
```

## 8.2 帧对象：Python 调用栈的可编程视图

帧（frame）是 Python 运行时最核心的自省对象：

| 属性 | 含义 | 调试用途 |
| :--- | :--- | :--- |
| `f_back` | 调用者帧 | 手动遍历调用栈 |
| `f_code` | 代码对象（`co_filename`、`co_name`、`co_firstlineno`） | 定位函数 |
| `f_lineno` | 当前行号 | 定位执行位置 |
| `f_locals` | 局部变量 | 看当前状态（3.13 起是 `FrameLocalsProxy`，可写回） |
| `f_globals` | 全局变量 | 看模块级状态 |
| `f_trace` | 该帧的 trace 函数 | 实现单帧追踪 |
| `f_lasti` | 字节码偏移 | 与 `dis` 对照 |

```python
import sys, traceback

def dump_stack(limit: int = 10) -> str:
    frame = sys._getframe()
    lines = []
    while frame and limit > 0:
        code = frame.f_code
        lines.append(f"{code.co_filename}:{frame.f_lineno} in {code.co_name}")
        frame = frame.f_back
        limit -= 1
    return "\n".join(lines)

def who_called_me():
    print(dump_stack())
```

### 8.2.1 `sys._getframe()` 与 `inspect.currentframe()`

两者都能拿到当前帧，`sys._getframe()` 更快（`inspect.currentframe()` 内部就是它 + 一些包装）。注意：**帧对象会持有局部变量引用**，长期保存会导致内存泄漏（第 10 章）。

### 8.2.2 3.13 的 `FrameLocalsProxy`（PEP 667）

3.13 起 `frame.f_locals` 返回 `FrameLocalsProxy`，**写入会同步回真实的局部变量**（实测）：

```python
import sys

def g():
    x = 1
    print(type(sys._getframe().f_locals))   # <class 'FrameLocalsProxy'>
    sys._getframe().f_locals["x"] = 42
    print("x =", x)                        # x = 42

g()
```

这在调试器、模板引擎、测试框架里非常有用（可以真正修改变量）。**但 3.12 及以前写入 `f_locals` 不会同步回局部变量**（旧文档明确说明"不应修改"），跨版本的工具代码要注意。

> `locals()` 在 3.13 也有对应的语义收紧（PEP 667）：函数内 `locals()` 返回快照/代理的行为更可预测，但**仍然不要依赖运行时写入 `locals()` 改变变量**。

## 8.3 `sys.settrace`：最强大的动态钩子

Python 用 `sys.settrace(fn)` 注册"每次进入新帧时调用"的全局 trace 函数；trace 函数返回的"局部 trace 函数"会收到该帧内的逐行/返回/异常事件。

```python
import sys

def trace(frame, event, arg):
    if event == "call":
        print(f"→ call {frame.f_code.co_name} at {frame.f_code.co_filename}:{frame.f_lineno}")
        return trace            # 返回局部 trace 函数（通常就是自己）
    if event == "line":
        print(f"   line {frame.f_lineno}")
    elif event == "return":
        print(f"← return {frame.f_code.co_name} -> {arg!r}")
    elif event == "exception":
        print(f"! exception {arg[0].__name__} in {frame.f_code.co_name}")
    return trace

sys.settrace(trace)
try:
    ...  # 目标代码
finally:
    sys.settrace(None)          # ⚠️ 一定要恢复，否则全局变慢
```

| 事件 | 触发时机 | `arg` |
| :--- | :--- | :--- |
| `call` | 进入新帧 | `None` |
| `line` | 执行新行（或回退到同一行） | `None` |
| `return` | 函数返回/抛异常退出 | 返回值或 `None` |
| `exception` | 帧内发生异常 | `(exc_type, exc_value, traceback)` |
| `opcode` | 每条字节码（需 `frame.f_trace_opcodes = True`） | `None` |

**实用示例 1：统计每个函数被调用了多少次**

```python
import sys, collections

counts = collections.Counter()

def tracer(frame, event, arg):
    if event == "call":
        counts[frame.f_code.co_name] += 1
    return tracer

sys.settrace(tracer)
target()
sys.settrace(None)
print(counts.most_common(10))
```

**实用示例 2：定位"到底走了哪个分支"**

```python
def branch_tracer(frame, event, arg):
    if event == "line" and frame.f_code.co_filename == __file__:
        print(f"{frame.f_lineno:>4}: {frame.f_code.co_name}")
    return branch_tracer
```

**代价与限制**：

- 开销极大：逐行回调，通常让程序慢 **10–100 倍**，且会改变并发时序；
- `sys.settrace` 只影响**当前线程**；子线程需要 `threading.settrace(fn)`（在创建线程前设置），或用 `threading.setprofile`；
- 不能修改正在执行的字节码（只能观察）；
- 3.12 起有了更低开销的 `sys.monitoring`（见 8.5），工具链应逐步迁移；
- 对 C 函数内部不可见（只看到 `c_call`/`c_return` 由 `sys.setprofile` 提供）。

> ⚠️ **务必在 `finally` 中 `sys.settrace(None)`**，否则整个进程（包括后续无关代码）都会变慢，而且异常会被吞在 trace 函数里。

## 8.4 `sys.setprofile`：只看调用，不看行

`sys.setprofile(fn)` 的回调事件为 `call`、`return`、`c_call`、`c_return`、`c_exception`，**没有 line 事件**，因此开销比 `settrace` 小，适合做调用统计、简易 profiler。

```python
import sys, time

def profiler(frame, event, arg):
    if event in ("call", "c_call"):
        name = getattr(arg, "__qualname__", None) or getattr(arg, "__name__", "?")
        print(f"{event}: {name}")

sys.setprofile(profiler)
target()
sys.setprofile(None)
```

标准库的 `cProfile` 内部就是基于 C 层 profile 钩子，所以比纯 Python 的 `sys.setprofile` 回调快得多。

## 8.5 `sys.monitoring`（3.12+，PEP 669）：低开销监控

`sys.monitoring` 是 3.12 引入的新 API，允许注册**按事件类型**的回调，并且可以在命中后"关闭该位置的事件"以消除热点开销。`coverage.py`、调试器等工具正在迁移到它上面。

```python
import sys

mon = sys.monitoring
TOOL = mon.PROFILER_ID          # 0=DEBUGGER_ID, 1=COVERAGE_ID, 2=PROFILER_ID, 5=OPTIMIZER_ID

mon.use_tool_id(TOOL, "line-counter")   # 声明工具 ID（已被占用会报错）
hits = []

def on_line(code, line):
    hits.append((code.co_name, line))

mon.register_callback(TOOL, mon.events.LINE, on_line)
mon.set_events(TOOL, mon.events.LINE)

target()
mon.set_events(TOOL, mon.events.NO_EVENTS)
mon.free_tool_id(TOOL)
print(len(hits), hits[:5])
```

实测输出（对递归 `fib(4)`）：

```text
line events: 20 [('<module>', 30), ('fib', 11), ('fib', 13), ('fib', 11), ('fib', 13), ...]
```

| API | 作用 |
| :--- | :--- |
| `use_tool_id(id, name)` / `free_tool_id(id)` | 占用/释放工具 ID（0–5） |
| `register_callback(id, event, fn)` | 注册回调 |
| `set_events(id, events)` / `set_local_events` | 全局/按代码对象启用事件 |
| `get_events` / `get_local_events` | 查询当前事件 |
| `restart_events()` | 重启被 `DISABLE` 关闭的事件 |
| `mon.events.*` | `PY_START`、`LINE`、`CALL`、`RAISE`、`PY_RETURN`、`PY_YIELD`、`INSTRUCTION` 等 |

**关键优势**：回调可以返回 `sys.monitoring.DISABLE`，让"某个代码位置的这个事件"不再触发，从而在热循环里保持低开销。这是 `settrace` 做不到的。

**限制**：工具 ID 数量有限（0–5），多个工具同时使用会冲突；回调里不能做太重的操作；需要 Python 3.12+。

## 8.6 `dis`：看懂字节码，解释"为什么行为不对"

很多"看起来不合逻辑"的行为，只有看字节码才能解释。

```python
import dis

def add(a, b=10):
    c = a + b
    return c

dis.dis(add)
```

3.13 实测输出：

```text
 11           RESUME                   0
 12           LOAD_FAST_LOAD_FAST      1 (a, b)
              BINARY_OP                0 (+)
              STORE_FAST               2 (c)
 13           LOAD_FAST                2 (c)
              RETURN_VALUE
```

**调试用途**：

| 现象 | 字节码线索 |
| :--- | :--- |
| 闭包变量值"不对" | `LOAD_DEREF` + cell 变量；多个闭包共享 cell（延迟绑定） |
| 局部变量被当成全局 | 函数内有赋值 → `STORE_FAST`，导致 `LOAD_FAST` 而非 `LOAD_GLOBAL`（`UnboundLocalError` 的根因） |
| `for` 循环里 lambda 都取最后一个值 | 闭包引用同一 cell |
| 性能热点 | `LOAD_GLOBAL`（慢）vs `LOAD_FAST`（快）、`BINARY_OP`、`CALL` |
| 短路求值/条件顺序 | `JUMP_IF_FALSE_OR_POP`、`POP_JUMP_IF_*` |

```python
import dis, types

# 查看 code 对象的元信息
code = add.__code__
print(code.co_varnames, code.co_names, code.co_consts, code.co_freevars)
print("指令数:", len(list(dis.get_instructions(code))))

# 反汇编任意源码字符串（不落盘）
dis.dis(compile("x = [i * 2 for i in range(3)]", "<string>", "exec"))
```

> 经验：**怀疑"作用域/闭包/默认参数"问题时，先 `dis.dis` 一下**，比读十遍源码更快。

## 8.7 `ast` 与 `compile`：在"代码成为程序"之前观察它

```python
import ast

src = """
def f(x=[]):
    return x
"""
tree = ast.parse(src)
print(ast.dump(tree, indent=2, include_attributes=True))
```

| 场景 | 用法 |
| :--- | :--- |
| 检查生成的代码 | 先 `ast.dump` 看语法树，再 `compile`/`exec` |
| 自定义静态检查 | `ast.NodeVisitor`（第 5.6 节） |
| 模板/DSL 转 Python | 生成 AST 后 `compile`，错误信息比字符串拼接清晰 |
| 分析 import 依赖 | 遍历 `ast.Import`/`ast.ImportFrom` |
| 调试装饰器 | `inspect.getsource` 看原始源码，`ast` 看改写 |

**注意**：`ast.parse` 只做语法检查，`compile` 才生成字节码；`exec`/`eval` 的 traceback 只有 `<string>` 行号，调试时把源码 `linecache.cache` 注入可以改善（进阶技巧）。

## 8.8 引用关系排查：谁在持有这个对象

```python
import sys, gc, weakref

class Node:
    pass

a = Node()
refs = gc.get_referrers(a)          # 谁引用了 a（返回引用者对象列表）
print([type(r).__name__ for r in refs])
print(sys.getrefcount(a))           # 引用计数（注意：传参本身会 +1）
print(gc.is_tracked(a))             # 是否被 GC 跟踪
```

| 工具 | 用途 |
| :--- | :--- |
| `sys.getrefcount(obj)` | 引用计数（结果比实际多 1，因为函数参数） |
| `gc.get_referrers(obj)` | 谁引用它（排查"为什么没释放"） |
| `gc.get_referents(obj)` | 它引用了谁（排查"它为什么这么大"） |
| `weakref.ref` / `WeakSet` | 不阻止回收的引用（缓存、观察者） |
| `objgraph.show_backrefs(obj)` | 图形化引用链（需要 `objgraph` + graphviz） |

> 注意：`gc.get_referrers` 对不可追踪对象（如 `int`、`str`）不返回 C 层引用；排查泄漏时优先看"业务对象/容器"。

## 8.9 动态导入、热重载与循环导入

| 现象 | 原因 | 排查 |
| :--- | :--- | :--- |
| `ModuleNotFoundError` 但文件明明存在 | `sys.path`/包结构/CWD 不对 | `python -c "import sys; print(sys.path)"`、`print(module.__file__)` |
| 循环导入 | A 导入 B，B 又导入 A | `python -X importtime` 看导入顺序；把导入移进函数/改用 `TYPE_CHECKING` |
| 改了代码不生效 | 旧 `.pyc`、同名的已安装包、`sys.modules` 缓存 | `python -B`、删 `__pycache__`、`importlib.invalidate_caches()` |
| `importlib.reload` 后对象行为不一致 | 旧类实例仍是旧类、`isinstance` 失败 | 重载后不要混用旧对象；重启进程更可靠 |
| 命名空间包/相对导入报错 | `__init__.py` 缺失或 `-m` 运行方式不对 | 用 `python -m package.module` 而不是 `python package/module.py` |

```python
import importlib, sys

importlib.invalidate_caches()        # 刷新查找器缓存
import mymodule
mymodule = importlib.reload(mymodule)   # 重载（⚠️ 旧实例不会自动更新）
```

## 8.10 本章要点与坑清单

**要点**

- `inspect` 解决"这个对象/函数/模块到底是什么"，是排障第一站。
- 帧对象（`f_back`/`f_lineno`/`f_locals`）是自建调试工具的原料；3.13 起 `f_locals` 可写回。
- `sys.settrace` 能力最强但开销最大，只用于短时间、精确定位；记得 `finally` 里复位。
- `sys.monitoring`（3.12+）是低开销的动态追踪方案，支持 `DISABLE` 热点。
- `dis` 能解释闭包、作用域、`UnboundLocalError` 等"反直觉"行为。
- `gc.get_referrers`/`weakref` 是排查"对象为什么不释放"的入口。

**坑**

- 忘记 `sys.settrace(None)`，导致进程整体变慢。
- 在 trace 回调里做 I/O/加锁，造成死锁或性能雪崩。
- 子线程没有设置 `threading.settrace`，以为 trace 覆盖了所有线程。
- 用 `importlib.reload` 热重载后继续用旧对象，得到"混合版本"的诡异行为。
- 依赖 `dir()`/`hasattr()` 探测动态对象，触发属性错误或副作用。
- 用 `sys._getframe()` 保存帧对象做缓存，导致内存泄漏。

---

# 第 9 章 性能剖析：定位"慢在哪"

## 9.1 性能调试的纪律：先测量，再优化

性能问题最容易犯的错误是"凭直觉优化"。正确顺序永远是：

```
① 定义指标（延迟 p50/p99？吞吐？CPU？内存？）
        ↓
② 建立基线（可重复的负载、固定的硬件/数据量）
        ↓
③ 找到热点（剖析器：采样 → 统计 → 行级）
        ↓
④ 验证瓶颈假设（改一处、测一次、对比基线）
        ↓
⑤ 检查副作用（内存、可读性、并发安全）
        ↓
⑥ 固化（基准测试进 CI，防止回退）
```

**三条纪律**：

- **不要用微基准替代真实负载**：`timeit` 测的是函数本身，真实系统的热点常在 I/O、序列化、锁等待上；
- **一次只改一个变量**，并记录改前/改后数据；
- **区分"快"和"省"**：CPU 降了但内存涨 3 倍，可能更糟。

## 9.2 计时与微基准

| API | 含义 | 适用 |
| :--- | :--- | :--- |
| `time.perf_counter()` | 单调高精度时钟（含睡眠/等待） | 测墙钟耗时 |
| `time.perf_counter_ns()` | 纳秒整数 | 极短区间，避免浮点误差 |
| `time.process_time()` | 进程 CPU 时间（用户+系统） | 判断是否 CPU 密集 |
| `time.monotonic()` | 单调时钟 | 超时、间隔 |
| `time.time()` | 日历时间（可被 NTP 调整） | 只用于记录"几点发生" |

```python
from time import perf_counter

start = perf_counter()
do_work()
print(f"elapsed = {perf_counter() - start:.6f}s")
```

### 9.2.1 `timeit` 的正确用法

```python
import timeit

# 1) 传表达式字符串（适合对比小片段）
t = timeit.timeit("'-'.join(str(n) for n in range(100))", number=10_000)
print(t / 10_000)

# 2) 传可调用对象（避免字符串编译开销）
def build():
    return "-".join(str(n) for n in range(100))

print(timeit.timeit(build, number=10_000) / 10_000)

# 3) 自动选择次数 + 多轮取最优（命令行）
# $ python -m timeit -s "import json" "json.dumps({'a': 1})"
# $ python -m timeit -r 7 -n 10000 "sum(range(1000))"
```

**`timeit` 的坑**：

| 坑 | 说明 | 处理 |
| :--- | :--- | :--- |
| 预热不足 | 第一次调用含 import/编译/JIT 效应 | 多跑几轮取最小值/中位数 |
| GC 干扰 | 测量期间触发 GC | `gc.disable()` 后再测（测完恢复） |
| 结果被优化掉 | 表达式无副作用时解释器/编译器可能不同 | 把结果赋值/累加，避免死代码消除（CPython 层一般不会，但仍需注意） |
| 测量环境噪声 | 后台任务、CPU 降频、电源模式 | 固定环境、多轮比较、看分布而非单次 |
| 用 `time.time()` 测短区间 | 精度不足且可能跳变 | 用 `perf_counter_ns` |
| 微基准结论外推 | 函数快 2 倍 ≠ 系统快 2 倍 | 用真实负载验证 |

> 经验：**微基准只回答"这两段代码谁常数项更小"，不回答"系统该优化哪里"。** 后者靠剖析器。

## 9.3 `cProfile`：统计式剖析的主力

```bash
# 直接剖析脚本，按累计时间排序
python -m cProfile -s cumulative script.py

# 保存结果，事后分析
python -m cProfile -o out.prof script.py
python -m pstats out.prof
# pstats 内：sort cumtime / stats 20 / callers foo / callees foo
```

```python
import cProfile, pstats, io

pr = cProfile.Profile()
pr.enable()
main()
pr.disable()

s = io.StringIO()
pstats.Stats(pr, stream=s).strip_dirs().sort_stats("cumulative").print_stats(20)
print(s.getvalue())
```

**输出字段含义**：

| 字段 | 含义 | 怎么用 |
| :--- | :--- | :--- |
| `ncalls` | 调用次数（递归显示为 `总次数/原始次数`） | 发现 N+1 调用、重复计算 |
| `tottime` | 函数自身耗时（不含子调用） | **找真正的热点** |
| `percall`（第一个） | `tottime / ncalls` | 单次成本 |
| `cumtime` | 累计耗时（含子调用） | 找"哪个入口最贵" |
| `percall`（第二个） | `cumtime / 原始调用次数` | 递归场景重要 |
| `filename:lineno(function)` | 位置 | 跳到源码 |

**读剖析结果的方法**：

1. 先按 `tottime` 排序，找"自己就很慢"的函数（真正的计算热点）；
2. 再按 `cumtime` 排序，找"入口慢"的调用链（可能只是调用了慢函数）；
3. 看 `ncalls` 异常大的函数——常见是循环里反复调用、N+1 查询、重复解析；
4. 对热点函数做行级剖析（9.4）或直接看源码/算法。

**`cProfile` 的局限**：

- 有开销（通常 2–5 倍，递归/大量小函数更明显），会扭曲时序；
- 只统计 **Python 函数调用**，纯 C 扩展内部（如 numpy 的算子）只表现为一次调用；
- 多线程/异步下无法区分"这个调用属于哪个线程/任务"，且会放大锁竞争的影响；需要按线程分析时用 `yappi` 或采样剖析；
- 不适合观察"等待"（锁、I/O、GC），这类要结合采样剖析（`py-spy`）和系统工具。

## 9.4 `line_profiler`：定位到"哪一行"

```bash
pip install line_profiler
kernprof -l -v script.py        # 需要给目标函数加 @profile
```

```python
@profile            # kernprof 注入的装饰器，直接运行脚本时需自行定义
def process(rows):
    total = 0
    for row in rows:            # ← 若这行占比高，说明循环体/迭代本身贵
        total += transform(row) # ← 若这行占比高，说明 transform 贵
    return total
```

**输出字段**：`Hits`（执行次数）、`Time`（总耗时）、`Per Hit`、`% Time`、`Line Contents`。它能直接告诉你"第 12 行占了 60% 时间"，这是 `cProfile` 给不到的粒度。

**使用建议**：

- 只对已定位的热点函数做行级剖析（开销大）；
- 循环里的行要结合 `Hits` 看：`Hits` 很大说明执行次数多，不一定是单次慢；
- 浏览器可视化：`line_profiler` 支持 `%lprun`（IPython）与 `snakeviz` 生态。

## 9.5 采样剖析：`py-spy`、`pyinstrument`、`scalene`

统计剖析（cProfile）需要插桩；采样剖析每隔几毫秒"抓一次栈"，开销极低，**可以在生产环境用**，且能看到 C 扩展/锁/GC 的等待。

### 9.5.1 `py-spy`：生产环境的事实标准

```bash
pip install py-spy

py-spy top --pid 12345                 # 类似 top 的实时视图（函数级）
py-spy dump --pid 12345                # 抓一次所有线程的 Python 栈（排障神器）
py-spy record -o profile.svg --pid 12345          # 生成火焰图（SVG）
py-spy record -o profile.svg -- python app.py      # 直接剖析启动的命令
py-spy record --format speedscope -o out.json --pid 12345
py-spy replay out.json                 # 回放已录制的数据
```

**特点**：

- **无需修改代码、无需重启进程**，通过读取进程内存/`ptrace` 采样；
- 支持 Linux/macOS/Windows；
- 能显示 C 扩展、GIL 等待、线程状态；
- 开销通常 < 1%，适合线上短时采样。

**权限问题**：Linux 下需要 `ptrace` 权限（`CAP_SYS_PTRACE` 或 `kernel.yama.ptrace_scope=0`），容器需要 `--cap-add=SYS_PTRACE`；否则报 `Permission denied`。

**`py-spy dump` 是最常用的线上"卡死"排查命令**，输出示例（每个线程一段）：

```text
Thread 0x7f8b2c0d1740 (idle): "MainThread"
    select (selectors.py:415)
    _run_once (base_events.py:1854)
    run_forever (base_events.py:703)
    run (runners.py:118)
    main (app.py:42)
```

从下往上读：`main` → `run` → `run_forever` → `select`，说明主线程在事件循环里等 I/O，正常。

### 9.5.2 `pyinstrument`：面向开发者的采样剖析器

```bash
pip install pyinstrument
pyinstrument script.py
pyinstrument -r html -o report.html script.py
```

特点：采样 + 调用树、支持 async（能显示 await 链）、HTML 报告交互好、对 Web 框架有中间件（`pyinstrument` 的 Flask/Django 集成）。适合"开发期看一次请求的耗时分布"。

### 9.5.3 `scalene`：CPU + 内存 + GPU 一起看

```bash
pip install scalene
scalene script.py
```

特点：同时给出 CPU（Python 层/原生层）、内存分配、复制量；能区分"Python 时间"和"系统/C 时间"，对 numpy/科学计算排查特别有用。

### 9.5.4 `yappi`：多线程/协程友好的统计剖析

```python
import yappi

yappi.set_clock_type("cpu")     # 或 "wall"
yappi.start()
main()
yappi.stop()

for func in yappi.get_func_stats().sort("tsub").top(20):
    print(func)
```

`yappi` 能按线程/协程统计，弥补 `cProfile` 在多线程场景下的不足。

| 工具 | 类型 | 开销 | 多线程/异步 | 生产可用 | 最适合 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `cProfile` | 统计（插桩） | 中（2–5×） | 一般 | 谨慎 | 函数级热点 |
| `line_profiler` | 统计（行级） | 高 | 一般 | 否 | 热点函数内部 |
| `py-spy` | 采样 | <1% | 好 | **是** | 线上卡顿/火焰图 |
| `pyinstrument` | 采样 | 低 | 好（async） | 可用 | 请求级耗时树 |
| `scalene` | 采样 + 分配 | 低-中 | 好 | 可用 | CPU+内存+GPU |
| `yappi` | 统计 | 中 | **好** | 谨慎 | 多线程/协程统计 |

## 9.6 启动与导入时间

```bash
python -X importtime -c "import myapp"      # 每个模块的导入耗时（微秒）
PYTHONPROFILEIMPORTTIME=1 python app.py      # 等价环境变量
python -X importtime -c "import myapp" 2> import.log
```

输出示例（`importtime` 的树状结构，`cumulative` 表示累计）：

```text
import time: self [us] | cumulative | imported package
import time:       123 |        456 | myapp
import time:        78 |        234 |   myapp.core
```

**常见启动慢的原因**：

| 原因 | 表现 | 处理 |
| :--- | :--- | :--- |
| 重型 import（pandas、numpy、torch、grpc） | 启动就几秒 | 延迟导入（函数内 import） |
| 包 `__init__.py` 里做初始化 | import 即建连接/读配置 | 改为显式初始化函数 |
| pkg_resources/entry points 扫描 | 慢且随依赖数增长 | 用 `importlib.metadata` |
| 大量小模块 | 累计几十毫秒 | 合并/减少分层 |
| 字节码未缓存 | 首次启动慢 | 保留 `__pycache__`、预热 |
| 网络文件系统 | 导入极慢 | 本地化依赖 |

## 9.7 `trace` 与 `coverage`：执行路径类工具

`trace` 模块适合"到底执行了哪些行/哪些函数"，不适合性能测量（开销极大）：

```bash
python -m trace --trace script.py        # 逐行打印
python -m trace --count --file=counts.txt script.py   # 统计每行执行次数
python -m trace --listfuncs script.py
```

`coverage.py` 用于覆盖率（第 13 章），它也能间接回答"这段代码在测试里有没有被执行"：

```bash
coverage run -m pytest
coverage report -m
coverage html
```

> ⚠️ 不要把覆盖率当性能工具，也不要用覆盖率证明"逻辑正确"——覆盖率只说明"执行过"。

## 9.8 火焰图怎么读

```
[================ main ================]
[== load_data ==][==== process ====]
[ db ][ parse ][ transform ][ write ]
```

| 规则 | 说明 |
| :--- | :--- |
| 横轴 | 不是时间先后，而是**采样占比**（宽度 = 出现频率） |
| 纵轴 | 调用栈深度，从下（调用者）到上（被调用者） |
| 宽 | 越宽越耗时；**顶层的宽块才是热点** |
| 同一层多个块 | 按字母序排列（不代表时间顺序） |
| 平台/尖峰 | 顶部"平顶"通常是叶子热点（如 `memcpy`、正则引擎） |
| 颜色 | 通常随机/按模块区分，不代表冷热 |

**读法**：

1. 先找最宽的叶子块（自己耗时最多的函数）；
2. 再看它的调用链，理解业务含义；
3. 对比优化前后火焰图，确认热点真的消失（而不是转移到别处）；
4. 采样数据量要足够（几秒到几十秒），太短会失真。

## 9.9 性能问题的常见根因清单

| 类别 | 典型模式 | 定位手段 |
| :--- | :--- | :--- |
| 算法 | O(n²) 嵌套循环、线性查找、重复排序 | 读代码 + `cProfile` 看 `tottime` |
| N+1 | 循环里查数据库/发 RPC | 统计调用次数、SQL 日志、`py-spy` |
| 循环内 I/O | 每条记录一次写文件/网络请求 | 日志/追踪、批量重写 |
| 重复计算 | 每次调用都重新解析/正则编译/读配置 | `functools.lru_cache`、提前编译 |
| 字符串拼接 | 大量 `+=` 或 f-string 循环 | `''.join`、`list.append` |
| 正则 | 灾难性回溯（嵌套量词 + 长输入） | 简化正则、超时、`regex` 模块 |
| 序列化 | 大对象反复 JSON/pickle | 换格式、流式、增量 |
| 锁竞争 | 全局锁、临界区过大 | `py-spy dump` 看栈、分片锁、无锁结构 |
| GC | 分配速率高、暂停长 | `gc.callbacks` 统计、减少临时对象 |
| 日志 | 同步写盘/网络、格式化开销 | 异步 handler、级别控制 |
| 数据库 | 缺索引、大事务、连接池不足 | 慢查询日志、执行计划 |
| 分页/缓存 | 全量加载、缓存穿透 | 分页、批量、缓存 |
| 启动 | 重型 import | `-X importtime` |

## 9.10 本章要点与坑清单

**要点**

- 先定指标再测量：墙钟/CPU/吞吐/分位数/内存，各看各的。
- `timeit` 只回答微基准问题；系统热点用剖析器。
- `cProfile` 看 `tottime` 找自身热点，看 `cumtime` 找入口热点，看 `ncalls` 找 N+1。
- `line_profiler` 定位到行；`py-spy` 无侵入采样，适合线上；`pyinstrument` 适合请求级耗时树。
- `-X importtime` 是启动慢的第一把钥匙。
- 火焰图横轴是采样占比，最宽的叶子块才是热点。
- 优化后必须用同一基线复测，并检查副作用。

**坑**

- 用平均值掩盖 p99；用本机空载数据推断线上。
- 测微基准不预热、不关 GC、不看分布。
- 看到 `cumtime` 高就优化该函数，其实它只是调用了慢函数。
- 在生产环境跑 `cProfile`（开销大、影响服务）。
- 用 `trace`/`coverage` 做性能分析（它们不是干这个的）。
- 优化后不写基准测试，几周后回退。

---

# 第 10 章 内存与资源调试：找出"谁在涨"

## 10.1 先理解 CPython 的内存模型

| 概念 | 含义 | 调试含义 |
| :--- | :--- | :--- |
| 引用计数 | 每个对象记录被引用次数，归零立即释放 | 大多数对象"即时"回收 |
| 循环 GC | 引用计数无法处理环，靠分代 GC 回收 | 环 + `__del__` 可能延迟回收 |
| 分代（gen0/1/2） | 新对象在 gen0，存活越久越靠后 | `gc.collect()` 可强制全量回收 |
| pymalloc | 小对象（≤512B）由 Python 内存池管理 | 释放后内存未必归还 OS |
| arena | 256KB 的池块，pymalloc 从中分配 | 碎片化会导致 RSS 不降 |
| RSS | 常驻物理内存 | 观察泄漏的主要指标 |
| VSZ | 虚拟地址空间 | 远大于 RSS 是正常的 |

**关键认知**：**"Python 释放了对象"不等于"操作系统收到的内存变少"**。pymalloc 保留 arena、glibc 保留 arena/brk 段，都会让 RSS 居高不下。所以"内存曲线只涨不降"不一定是泄漏，也可能是缓存或分配器行为。

## 10.2 观测工具

### 10.2.1 `sys.getsizeof` 与 `pympler.asizeof`

```python
import sys
sys.getsizeof([1, 2, 3])            # 只算 list 本身（约 88 字节），不含元素
sys.getsizeof("hello")              # 字符串对象大小
```

`getsizeof` 是**浅层**大小；要算"整个对象图"的大小，用 `pympler.asizeof`：

```python
from pympler import asizeof
print(asizeof.asizeof(big_dict))    # 递归统计（有性能开销）
```

### 10.2.2 `tracemalloc`：定位分配点

```python
import tracemalloc

tracemalloc.start(10)                     # 保留 10 层栈帧
snap1 = tracemalloc.take_snapshot()

leak = [bytearray(1024 * 10) for _ in range(100)]

snap2 = tracemalloc.take_snapshot()
for stat in snap2.compare_to(snap1, "lineno")[:5]:
    print(stat)
```

3.13.5 实测输出（格式示意）：

```text
tm.py:8: size=1006 KiB (+1006 KiB), count=201 (+201), average=5127 B
Lib/tracemalloc.py:560: size=328 B (+328 B), count=1 (+1), average=328 B
```

```text
current=1008.9 KiB peak=1009.2 KiB
```

| API | 作用 |
| :--- | :--- |
| `tracemalloc.start(nframes)` | 开始跟踪（`nframes` 越大越精确也越慢） |
| `take_snapshot()` | 当前分配快照 |
| `snap.compare_to(old, "lineno" \| "filename" \| "traceback")` | 差异对比（**找泄漏的核心**） |
| `snap.statistics("traceback")` | 聚合统计 |
| `snap.filter_traces([Filter(True, __file__)])` | 只看自己代码 |
| `get_traced_memory()` | `(current, peak)` 字节数 |
| `tracemalloc.stop()` | 停止跟踪 |

**命令行开启**（无需改代码）：

```bash
python -X tracemalloc=20 script.py     # 跟踪 20 帧，退出时打印 top 10
PYTHONTRACEMALLOC=20 python script.py
```

> `tracemalloc` 只跟踪 **Python 层分配**，看不到 numpy 大数组（C 层 `malloc`）和内存映射文件；这类要用 `memray`/`psutil`/RSS 对比。

### 10.2.3 RSS 与进程级观测

```python
import os, psutil

proc = psutil.Process(os.getpid())
print(proc.memory_info().rss / 1024 / 1024, "MiB")
print(proc.memory_percent())
print(proc.num_fds() if hasattr(proc, "num_fds") else proc.num_handles())  # Linux / Windows
```

| 平台 | 方式 |
| :--- | :--- |
| Linux | `/proc/<pid>/status`（`VmRSS`）、`smem`、`cgroup memory.current` |
| Windows | `psutil`、任务管理器、`Get-Process` |
| 跨平台 | `psutil`、`resource`（**Unix-only**，Windows 下 `import resource` 直接 `ModuleNotFoundError`） |

### 10.2.4 `gc` 统计与调试

```python
import gc

print(gc.get_count())                    # 各代对象数
print(gc.get_threshold())                # (700, 10, 10)
print(gc.get_stats())                    # 每代回收次数、回收对象数
gc.set_debug(gc.DEBUG_SAVEALL)           # 回收时保存到 gc.garbage（排查环）
gc.collect()
print(len(gc.garbage))
```

`gc.callbacks` 可以挂回调统计每次回收的耗时（定位 GC 毛刺）：

```python
import gc, time

def on_gc(phase, info):
    if phase == "start":
        on_gc.t0 = time.perf_counter()
    else:
        print(f"gc gen={info['generation']} 耗时 {time.perf_counter() - on_gc.t0:.3f}s")

gc.callbacks.append(on_gc)
```

### 10.2.5 第三方工具

| 工具 | 能力 | 命令/用法 |
| :--- | :--- | :--- |
| `objgraph` | 引用链、对象增长 | `objgraph.show_most_common_types()`、`show_backrefs(obj)` |
| `pympler` | 对象大小、类实例统计 | `asizeof`、`muppy`、`summary` |
| `memray` | 原生+Python 分配追踪、火焰图 | `memray run`、`memray flamegraph`、`memray attach` |
| `memory_profiler` | 逐行内存增量 | `@profile` + `python -m memory_profiler` |
| `psutil` | 进程级 RSS/FD/线程 | 见上 |
| `valgrind massif` | C 层堆剖析 | `valgrind --tool=massif python app.py` |
| `heaptrack` | 原生堆分析 | `heaptrack python app.py` |

```python
from memory_profiler import profile

@profile
def build(n):
    data = [str(i) * 100 for i in range(n)]   # 逐行显示内存增量
    return data

build(100_000)
```

## 10.3 内存泄漏诊断流程

```text
① 确认现象
   RSS 随时间单调上升？还是阶跃后稳定（缓存）？还是锯齿（GC）？
        ↓
② 排除"不是泄漏"
   缓存无上限？对象池？分配器不归还（碎片）？mmap？
        ↓
③ 定位"哪个对象在涨"
   tracemalloc 快照 diff / objgraph.show_most_common_types / gc.get_objects 按类型计数
        ↓
④ 定位"谁在引用"
   gc.get_referrers / objgraph.show_backrefs / memray 分配栈
        ↓
⑤ 修复并验证
   有界缓存、weakref、清理注册表、clear_frames、取消任务
        ↓
⑥ 加护栏
   内存上限告警、定期压测、RSS 监控、泄漏回归测试
```

**第一步的判据**：

| 曲线形态 | 更可能是 |
| :--- | :--- |
| 持续线性上升，不回落 | 真泄漏（容器/注册表/全局列表） |
| 阶跃后平台 | 缓存/预热（正常） |
| 周期性锯齿 | GC 正常回收 |
| 缓慢上升后突然下降 | 大对象释放/GC 全量回收 |
| 只涨不降且 RSS 远超对象总量 | 分配器碎片/arena 未归还 |

## 10.4 常见泄漏源与修复

| # | 泄漏源 | 机制 | 修复 |
| :--- | :--- | :--- | :--- |
| 1 | 全局缓存/字典只增不减 | 键永不淘汰 | `functools.lru_cache(maxsize=...)`、TTL 缓存、`cachetools` |
| 2 | 全局注册表 | 监听器/插件注册后不移除 | `weakref.WeakSet`/`WeakKeyDictionary`、显式 `unregister` |
| 3 | 闭包持有大对象 | 函数对象持有 cell/大对象 | 缩小闭包捕获范围、用完置 `None` |
| 4 | 异常对象持有帧 | traceback 引用整帧局部变量 | `traceback.clear_frames(tb)`、只存消息 |
| 5 | 循环引用 + `__del__` | 有 `__del__` 的环需 GC，3.4+ 可回收但仍延迟 | 避免在 `__del__` 里做重活，用 `weakref.finalize` |
| 6 | 线程/进程未回收 | 线程池无界、`Thread` 泄漏 | 有界池、`join`、`shutdown` |
| 7 | 文件/socket/连接未关闭 | `close()` 遗漏 | `with`、`contextlib.closing`、连接池 |
| 8 | `logging` handler 重复添加 | 每次初始化加一个 handler | `force=True`、检查已有 handler |
| 9 | 缓存装饰器无界 | `@lru_cache` 默认 `maxsize=128`（有界，但 `cache` 无界） | `maxsize`、`cache_clear()` |
| 10 | asyncio 任务未取消 | `create_task` 后不持有引用/不 cancel | 保存引用、`TaskGroup`、`shutdown` |
| 11 | C 扩展/原生库泄漏 | 引用计数错误、未释放缓冲 | `valgrind`/`memray`、升级依赖 |
| 12 | 大对象被全局单例持有 | 配置/客户端/模型常驻 | 按需加载、显式释放 |

**泄漏排查代码模板**：

```python
import gc, collections, tracemalloc, linecache

def type_summary(top: int = 20):
    counts = collections.Counter(type(o).__name__ for o in gc.get_objects())
    for name, n in counts.most_common(top):
        print(f"{name:>30}: {n}")

tracemalloc.start(15)
snap_before = tracemalloc.take_snapshot()
run_business()
snap_after = tracemalloc.take_snapshot()

for stat in snap_after.compare_to(snap_before, "traceback")[:10]:
    print(stat)
    for line in stat.traceback.format()[-6:]:
        print("   ", line)
```

## 10.5 FD、连接、线程泄漏

| 资源 | 观测 | 常见原因 | 修复 |
| :--- | :--- | :--- | :--- |
| 文件描述符 | `psutil.Process().num_fds()`、`/proc/<pid>/fd`、`lsof -p <pid>` | `open()` 未关闭、异常路径漏 `close` | `with`、`try/finally`、`contextlib.ExitStack` |
| socket/连接 | `lsof -i`、`ss -tanp`、连接池指标 | 未 `close`、连接池泄漏、keep-alive 过多 | 连接池上限、超时、`with` |
| 线程 | `threading.active_count()`、`threading.enumerate()` | 每请求建线程、池无界 | 有界线程池、复用 |
| 子进程 | `psutil.Process().children()`、`ps -ef --forest` | `Popen` 未 `wait`/`kill` | `with Popen`、超时、清理 |
| 临时文件 | 临时目录增长 | `NamedTemporaryFile` 未删 | `TemporaryDirectory`、`delete=True` |

**让资源泄漏自己暴露**：

```bash
python -X dev -W error::ResourceWarning app.py    # 未关闭文件会变成异常
```

```python
import warnings
warnings.simplefilter("error", ResourceWarning)   # 测试中开启
```

## 10.6 内存峰值、OOM 与容器

| 现象 | 原因 | 排查 |
| :--- | :--- | :--- |
| 进程被 OOMKilled | cgroup 内存上限、RSS 峰值超限 | `kubectl describe pod` 看 `OOMKilled`、`container_memory_working_set_bytes` |
| `MemoryError` | 单次分配过大/地址空间不足 | 看分配点、分块/流式处理 |
| 内存慢慢涨到 OOM | 泄漏 | 第 10.3 节流程 |
| 容器内存不降 | 分配器不归还 + cgroup 统计口径 | 理解 `RSS` 与 `working_set` |
| 大量 mmap | `numpy.memmap`/`mmap` 文件 | `smaps` 看映射 |

**OOM 现场取证**：

- 容器：`kubectl describe`、`docker inspect` 的 `OOMKilled`、节点 `dmesg`；
- 进程：core dump（如果配置了 `core_pattern`）+ `gdb`/`pystack`；
- 提前预防：`tracemalloc` 定期快照、RSS 指标告警、压测；

> ⚠️ 发生 OOM 时 `faulthandler` 和调试器往往来不及工作（进程被内核直接杀死）。**OOM 靠预防和监控，不靠事后调试。**

## 10.7 降低内存占用的常用手段

| 手段 | 效果 | 注意 |
| :--- | :--- | :--- |
| `__slots__` | 去掉实例 `__dict__`，省 30%–50% | 不能动态加属性、多继承受限 |
| 生成器/迭代器 | 流式处理，避免全量列表 | 只能遍历一次 |
| `array`/`numpy` | 紧凑存储数值 | 需要转换、注意 dtype |
| 避免深拷贝 | `copy.deepcopy` 很贵 | 用不可变数据/写时复制 |
| 分块处理 | 内存上限可控 | 逻辑要支持分批 |
| 及时 `del`/`clear()` | 释放大对象引用 | 注意循环引用/其他引用 |
| 弱引用缓存 | 不阻止回收 | 缓存命中率下降 |
| `sys.intern` | 复用短字符串 | 只适合高频重复字符串 |
| 对象池/复用 | 减少分配 | 注意线程安全 |

```python
class Point:
    __slots__ = ("x", "y")          # 实例不再有 __dict__
    def __init__(self, x: float, y: float):
        self.x, self.y = x, y
```

## 10.8 本章要点与坑清单

**要点**

- 引用计数 + 分代 GC + pymalloc/arena = "释放了但 RSS 不降"的常见来源。
- `tracemalloc` 快照对比是定位泄漏点的第一工具；`-X tracemalloc=N` 可零改码开启。
- `gc.get_referrers`/`objgraph` 回答"谁在引用"；`memray` 能看到 C 层分配。
- 泄漏排查先分辨：真泄漏 / 缓存 / 碎片 / 分配器不归还。
- 资源泄漏（FD/连接/线程）用 `psutil`、`lsof`、`/proc`，并用 `ResourceWarning` 提前暴露。
- OOM 靠监控和预防，事后调试往往没有现场。

**坑**

- 把"RSS 不下降"一律当泄漏，实际是 arena/缓存。
- 用 `sys.getsizeof` 估算容器总大小（只算浅层）。
- `@lru_cache` 不加 `maxsize`（注意 `functools.cache` 是无界的）。
- 异常对象存进全局列表，导致整帧局部变量泄漏。
- 在多进程/容器里只看单进程 RSS，忽略子进程与 cgroup 统计口径。
- 用 `gc.collect()` 掩盖泄漏（只是把问题推迟）。

---

# 第 11 章 并发、异步与多进程调试

## 11.1 并发 bug 的分类

| 类别 | 表现 | 根因 | 典型定位手段 |
| :--- | :--- | :--- | :--- |
| 数据竞争（race） | 偶发数据错乱、计数不对 | 多线程无锁读写同一状态 | 代码审查、`threading` 栈 dump、压力测试 |
| 死锁（deadlock） | 进程卡住不退出、CPU 为 0 | 锁顺序不一致、锁内调用外部代码 | `faulthandler`/`py-spy dump` 看栈 |
| 活锁/饥饿 | CPU 高但不推进、某线程永远拿不到锁 | 重试策略、锁不公平 | `py-spy top`、栈采样 |
| 原子性缺失 | `+=` 丢失更新、check-then-act | 复合操作非原子 | 加锁/队列/原子结构 |
| 顺序依赖 | 结果依赖执行顺序 | 依赖隐式时序 | 显式同步原语、`Barrier`/`Event` |
| 任务泄漏 | 内存涨、协程未取消 | `create_task` 未持有/未取消 | `asyncio.all_tasks()` |
| 异步阻塞 | 事件循环卡顿、p99 抖动 | 在协程里做同步 I/O/CPU 计算 | asyncio debug、`py-spy` |
| 多进程状态分裂 | 各进程计数不一致 | 期望共享内存但实际各有一份 | 明确共享机制（Queue/Manager/共享内存） |

**为什么难**：并发 bug 依赖精确的时序，调试手段本身（断点、日志）会改变时序，形成"海森堡效应"。所以并发调试的核心不是"停下来看"，而是**低成本采样 + 时序控制 + 压力放大**。

## 11.2 第一原则：先拿到"所有线程此刻在做什么"

### 11.2.1 `faulthandler`：零依赖、可随时启用

```python
import faulthandler

faulthandler.enable()                     # 崩溃（段错误等）时打印所有线程栈
faulthandler.dump_traceback()             # 立刻打印当前所有线程栈
faulthandler.dump_traceback(file=open("stacks.txt", "w"))   # 输出到文件
faulthandler.dump_traceback_later(30, repeat=True)          # 30 秒后打印（可重复）
faulthandler.cancel_dump_traceback_later()
```

3.13.5 实测输出（`dump_traceback_later(0.5)`）：

```text
Timeout (0:00:00.500000)!
Thread 0x00007a4c (most recent call first):
  File "fh.py", line 4 in worker
  File "Lib/threading.py", line 994 in run
  File "Lib/threading.py", line 1043 in _bootstrap_inner
  File "Lib/threading.py", line 1014 in _bootstrap
Thread 0x000098f4 (most recent call first):
  File "fh.py", line 9 in <module>
```

| API | 用途 |
| :--- | :--- |
| `faulthandler.enable(file=sys.stderr, all_threads=True)` | 致命信号时打印栈 |
| `faulthandler.dump_traceback(...)` | 主动打印（可在信号处理里调用） |
| `faulthandler.dump_traceback_later(timeout, repeat=False, file=..., exit=False)` | 定时打印（排查卡死） |
| `faulthandler.register(signum, ...)` | 把一个信号注册为"打印栈"（Unix，如 `SIGUSR1`） |
| `faulthandler.is_enabled()` | 是否已启用 |

**命令行**：`python -X faulthandler app.py` 或 `PYTHONFAULTHANDLER=1`；`python -X dev` 也会启用（实测）。

**线上卡死排查的经典套路**：

```python
import faulthandler, signal

faulthandler.register(signal.SIGUSR1)      # Unix：kill -USR1 <pid> 打印所有线程栈
```

这样无需重启、无需 attach 调试器，就能在服务"疑似卡死"时立刻拿到现场。

### 11.2.2 `sys._current_frames()`：自己实现线程栈快照

```python
import sys, threading, traceback

def dump_all_threads() -> None:
    frames = sys._current_frames()
    for thread in threading.enumerate():
        frame = frames.get(thread.ident)
        print(f"\n=== {thread.name} (ident={thread.ident}) ===")
        if frame is not None:
            traceback.print_stack(frame)
```

适合在管理端口（如 `/debug/stacks`）里安全地暴露栈快照。

### 11.2.3 `faulthandler` 不够用时：`py-spy dump` 与 gdb

| 工具 | 优势 | 命令 |
| :--- | :--- | :--- |
| `py-spy dump --pid <pid>` | 无侵入、含 C 栈、可看线程状态 | 生产首选 |
| `gdb -p <pid>` + `thread apply all py-bt` | 原生栈 + Python 栈 | C 扩展/死锁疑难 |
| `faulthandler` | 零依赖、可在代码内 | 容器/嵌入环境 |
| `py-spy top --pid` | 实时看哪个函数在烧 CPU | 活锁/忙等 |

## 11.3 死锁排查

**死锁的三种常见形态**：

```
形态 1：锁顺序不一致
  线程 A: 锁 L1 → 要 L2
  线程 B: 锁 L2 → 要 L1

形态 2：锁内做外部调用
  持锁 → 发 HTTP/查库/写日志 → 回调里又请求同一把锁

形态 3：wait 永不满足
  线程等待一个从未被 set 的 Event / 已被消费的信号量 / 队列永不入队
```

**排查步骤**：

1. **确认是死锁而不是慢**：CPU 是否接近 0？`py-spy top` 无热点 → 更像阻塞；
2. **抓全线程栈**：`py-spy dump --pid`（或 `faulthandler`/gdb）；
3. **看栈里的锁等待**：栈顶停在 `lock.acquire()`/`queue.get()`/`select()`；对照源码找谁持有锁；
4. **找锁顺序矛盾**：列出每个线程"已持有 + 正在等待"的锁，画等待图，找环；
5. **验证**：给锁加 `timeout`，或临时改成 `RLock`/单锁，看是否恢复——这既是验证也是修复候选。

**复现死锁的工程手段**：

```python
import threading

# 用 Barrier 让两个线程"同时"进入临界区，把偶发变必现
barrier = threading.Barrier(2)

def worker(lock_a, lock_b):
    barrier.wait()                 # 同时起跑
    with lock_a:
        with lock_b:
            ...
```

**预防**：

- 全局锁顺序（所有代码按同一顺序获取多把锁）；
- 缩小临界区，锁内不做 I/O、不调用回调；
- 用 `lock.acquire(timeout=...)` 代替无限等待；
- 用 `queue.Queue`/`concurrent.futures` 等高层结构替代手写锁；
- 在 CI 中跑并发压力测试，并用 `faulthandler` 超时 dump 兜底。

## 11.4 数据竞争与原子性

**必须记住的事实**：GIL 保证"同一时刻只有一个线程执行字节码"，但**不保证复合操作的原子性**。

```python
counter = 0

def increment():
    global counter
    for _ in range(100_000):
        counter += 1        # ❌ 读取-加-写 三步，可能交错
```

| 操作 | 是否原子 | 说明 |
| :--- | :--- | :--- |
| `list.append(x)` | 是（CPython 实现层面） | 但依赖实现，跨解释器不保证 |
| `d[k] = v` | 是（单条） | 但 `d[k] += 1` 不是 |
| `x += 1` | 否 | 读-算-写 |
| `if k in d: d[k] += 1` | 否 | check-then-act |
| `set.add` | 是 | 同上，实现相关 |
| `queue.put/get` | 是（有锁） | 推荐用它做线程间通信 |

**定位数据竞争的手段**：

- 代码审查 + 明确"哪些状态被多个线程共享"（画共享状态表）；
- 用 `threading.Lock`/`queue` 重写后再压测；
- 并发压力测试（多线程 × 多轮 × 断言不变量）；
- 在测试里禁用 GIL 的替代解释器（free-threaded 3.13t）暴露问题——但注意生态兼容性；
- 使用 `pytest-xdist`/多进程压测不能替代多线程竞争测试（进程间不共享内存）。

## 11.5 `asyncio` 调试

### 11.5.1 开启 debug 模式

```python
import asyncio

asyncio.run(main(), debug=True)          # 推荐
# 或
loop = asyncio.new_event_loop()
loop.set_debug(True)
loop.slow_callback_duration = 0.05       # 默认 0.1 秒
```

```bash
PYTHONASYNCIODEBUG=1 python app.py       # 环境变量
python -X dev app.py                     # 开发模式也会开启 asyncio debug
```

debug 模式会提供：

- **慢回调警告**：`Executing <Handle ...> took 0.120 seconds`；
- **未取回异常**：`Task exception was never retrieved`；
- **任务创建位置**：配合 `tracemalloc` 记录 source traceback；
- 更严格的资源警告（未关闭的 transport 等）。

3.13.5 实测（`loop.set_debug(True)` + 阻塞回调）：

```text
Executing <Handle main.<locals>.slow() at adbg.py:8 created at adbg.py:11> took 0.120 seconds
Task exception was never retrieved
future: <Task finished name='Task-2' coro=<main.<locals>.boom() ...> exception=RuntimeError('async boom') ...>
source_traceback: Object created at (most recent call last):
  File "adbg.py", line 21, in <module>
    asyncio.run(main())
  ...
```

### 11.5.2 `asyncio` 的六类典型 bug

| # | bug | 症状 | 修复 |
| :--- | :--- | :--- | :--- |
| 1 | 在协程里调用阻塞函数 | 事件循环卡顿、p99 抖动、慢回调警告 | `await asyncio.to_thread(...)`、`run_in_executor` |
| 2 | 忘记 `await` | 协程从未执行（`RuntimeWarning: coroutine ... was never awaited`） | 加 `await`，开 `-W error` 提前暴露 |
| 3 | `create_task` 不保存引用 | 任务可能被 GC 回收（"Task was destroyed but it is pending!"） | 保存到集合、用 `TaskGroup` |
| 4 | 未取回异常 | `Task exception was never retrieved`、错误被吞 | `await` 任务、`add_done_callback`、`TaskGroup` |
| 5 | 取消语义错误 | `CancelledError` 被吞/资源未清理 | `try/finally` 清理，不要裸 `except Exception` 捕获取消 |
| 6 | 生成器未关闭 | `async generator ... was never closed` | `aclose()`、`contextlib.aclosing`、`async with` |

**排查工具**：

```python
import asyncio

async def dump_tasks():
    for task in asyncio.all_tasks():
        print(task.get_name(), task.done(), task.get_coro())
```

```python
# 自定义异常处理器：记录未处理的异常，而不是静默打印
def handler(loop, context):
    print("asyncio 错误:", context.get("message"))
    exc = context.get("exception")
    if exc:
        loop.default_exception_handler(context)   # 保留默认输出

loop.set_exception_handler(handler)
```

**统计事件循环延迟**（判断"是不是循环被卡了"）：

```python
async def monitor(loop, interval=0.1):
    while True:
        t0 = loop.time()
        await asyncio.sleep(interval)
        drift = loop.time() - t0 - interval
        if drift > 0.05:
            print(f"事件循环延迟 {drift:.3f}s")
```

### 11.5.3 asyncio 与 pdb/剖析器

- `pdb` 在协程里会阻塞整个事件循环，其他任务全部停摆；**不适合调试并发时序问题**；
- `py-spy` 能看到事件循环的 C 栈（`epoll_wait`/`select`），判断是"等 I/O"还是"Python 在算"；
- `pyinstrument`/`yappi`/`scalene` 支持 async 调用树；
- `asyncio` 的 debug 模式 + 结构化日志（任务名、请求 ID）通常比断点有效。

## 11.6 多进程调试

| 问题 | 说明 | 排查/修复 |
| :--- | :--- | :--- |
| 子进程异常不传播 | `Pool`/`Process` 里异常默认不打印到主进程 | `future.result()`/`pool.apply_async(...).get()` 会重新抛出 |
| 日志交错/丢失 | 多进程写同一 stderr/文件 | `QueueHandler`、每进程独立文件、单写者 |
| 状态不共享 | 每个进程有独立内存 | 用 `Queue`/`Pipe`/`Manager`/`shared_memory`，并显式设计 |
| 死锁 | 队列满/子进程等父进程、`join` 顺序错误 | 先消费再 join、设超时、用 `concurrent.futures` |
| `fork` 与线程冲突 | fork 后子进程只有调用线程 | 3.12 起多线程进程中 `os.fork()` 会触发 `DeprecationWarning`；优先 `spawn`/`forkserver` |
| 调试子进程 | 断点不命中 | 子进程入口加 `debugpy.listen()`/`breakpoint()`；IDE 开 `subProcess` |
| 僵尸进程 | `Popen` 未 `wait` | `with Popen(...)`、`wait(timeout)`、`kill` |

```python
import multiprocessing as mp

def worker(x):
    if x == 3:
        raise ValueError(f"bad input: {x}")     # 子进程异常
    return x * 2

if __name__ == "__main__":
    ctx = mp.get_context("spawn")               # 明确 start method，跨平台一致
    with ctx.Pool(2) as pool:
        for result in pool.imap(worker, range(5)):
            print(result)                       # ValueError 会在这里抛出
```

**多进程调试要点**：

- 先确认 `start method`（`spawn` 会重新导入主模块，`fork` 继承状态）；
- 子进程的 `breakpoint()` 在无 stdin 时可能阻塞，优先 `debugpy --listen`；
- 用 `multiprocessing.util.log_to_stderr()` 打开内部日志（3.x 可用）；
- 容器里多进程 + `py-spy` 需要分别 attach 每个 PID（或用 `py-spy record --pid` 逐个采）。

## 11.7 子进程与信号

```python
import subprocess

try:
    proc = subprocess.run(
        ["mycmd", "--flag"],
        capture_output=True, text=True, timeout=10,
    )
    if proc.returncode != 0:
        raise RuntimeError(f"命令失败 rc={proc.returncode}\nstderr={proc.stderr[-2000:]}")
except subprocess.TimeoutExpired as e:
    print("超时，已捕获输出:", e.stdout, e.stderr)
```

| 问题 | 排查 |
| :--- | :--- |
| 子进程"没输出" | `capture_output=True` 后要读 `stdout/stderr`；否则可能被缓冲 |
| 退出码非 0 | 打印 `returncode` 与 `stderr`；`-1` 常表示被信号杀死 |
| 卡住 | 加 `timeout`；管道写满会阻塞（大输出要用 `communicate`） |
| 僵尸进程 | `wait()`/`communicate()`；`Popen` 用 `with` |
| 信号处理 | 用 `proc.send_signal`/`terminate`/`kill`；进程组用 `start_new_session=True` + `os.killpg` |

**优雅退出的调试**：`KeyboardInterrupt`/`SIGTERM` 期间清理逻辑最容易出 bug（重复关闭、锁未释放、任务未取消）。测试方法：启动服务 → 发信号 → 检查日志与资源是否清理干净。

```python
import signal, sys

def on_term(signum, frame):
    print(f"收到信号 {signum}，开始优雅退出")
    # 设置停止标志、取消任务、关闭连接池（注意：handler 里不要做重活）
    sys.exit(0)

signal.signal(signal.SIGTERM, on_term)
```

## 11.8 并发测试

| 手段 | 目的 | 工具 |
| :--- | :--- | :--- |
| 多线程压力测试 | 放大竞争窗口 | `ThreadPoolExecutor` + 循环 + 不变量断言 |
| 时序控制 | 把偶发变必现 | `threading.Barrier`/`Event`/`Condition` |
| 超时兜底 | 死锁快速失败 | `faulthandler.dump_traceback_later`、`pytest-timeout` |
| 故障注入 | 验证重试/降级/取消 | 自定义 mock、`pytest` fixture |
| 属性测试 | 覆盖并发下的状态组合 | `hypothesis`（配合线程/任务） |
| 重复运行 | 提高复现概率 | `pytest --count=N`（pytest-repeat）、CI 多轮 |

```python
import threading
from concurrent.futures import ThreadPoolExecutor

def test_counter_thread_safe():
    counter = {"n": 0}
    lock = threading.Lock()

    def work():
        for _ in range(10_000):
            with lock:
                counter["n"] += 1

    with ThreadPoolExecutor(8) as ex:
        list(ex.map(lambda _: work(), range(8)))

    assert counter["n"] == 80_000
```

## 11.9 本章要点与坑清单

**要点**

- 并发调试先拿"全线程栈"：`faulthandler`、`sys._current_frames`、`py-spy dump`、gdb。
- 死锁的本质是等待环：列"已持有 + 正在等"的锁，找环，再用超时/单锁验证。
- GIL 不保证复合操作原子性；共享状态必须加锁或改用消息传递。
- asyncio 调试开 debug 模式：慢回调、未取回异常、任务创建位置都能直接看到。
- 多进程先明确 start method 与共享机制，子进程异常要显式取回。
- 并发问题靠"时序控制 + 压力放大 + 不变量断言"，不是靠断点。

**坑**

- 用 `sleep` 制造并发，结果只是降低复现率。
- 在持锁时做 I/O/回调，引入难查的死锁。
- 用 `except Exception` 吞掉 `CancelledError`，导致任务无法取消。
- `create_task` 不保存引用，任务被 GC。
- 多进程里期望共享全局变量，实际每个进程一份。
- 调试并发时开大量 DEBUG 日志，反而改变了时序（日志本身也是同步 I/O）。

---

# 第 12 章 线上/生产环境调试：不中断服务的排障

## 12.1 线上调试的原则

| 原则 | 说明 |
| :--- | :--- |
| 先取证，后操作 | 重启/回滚前先抓栈、抓日志、抓指标，否则现场永久丢失 |
| 最小侵入 | 优先只读手段（日志、指标、`py-spy dump`），再考虑 attach 调试器 |
| 可回滚 | 任何临时改动（加日志、改配置）都要能立即撤销 |
| 不依赖复现 | 线上问题常常无法复现，必须靠"现场数据"推理 |
| 时间线优先 | 先对齐"什么时候开始、和什么变更同时发生" |
| 隔离影响 | 限流/降级/摘流量优先于"找根因"（先止血，再治病） |

**排障顺序**（黄金 5 步）：

```
1. 影响面：多少用户/请求受影响？是否在扩大？
2. 时间线：第一次出现时间？与发布/配置/数据/依赖变更的关系？
3. 分层：入口 → 应用 → 依赖（DB/缓存/MQ）→ 基础设施（CPU/内存/磁盘/网络）
4. 取证：日志（错误率/异常类型）、指标（QPS/延迟/资源）、追踪（慢在哪段）、栈（是否卡死）
5. 处置：限流/降级/回滚/扩容 → 验证 → 复盘
```

## 12.2 内置武器：标准库就能做的线上取证

### 12.2.1 `faulthandler`：进程级"紧急出口"

```python
import faulthandler, signal

faulthandler.enable()                         # 段错误等致命信号打印栈
faulthandler.register(signal.SIGUSR1)         # kill -USR1 <pid> 打印全线程栈
faulthandler.dump_traceback_later(300, repeat=True)   # 每 5 分钟打印一次（谨慎）
```

- 适合容器/无外网环境（无第三方依赖）；
- `SIGUSR1` 是"按需触发"的最佳实践：平时零开销，需要时一条 `kill` 命令拿栈；
- 注意 `dump_traceback_later` 会周期性写 stderr，日志量要评估。

### 12.2.2 三个异常钩子

```python
import sys, threading, logging

logger = logging.getLogger("app")

def excepthook(t, v, tb):
    logger.critical("未捕获异常", exc_info=(t, v, tb))
    # 上报 Sentry/告警，注意限流与脱敏

sys.excepthook = excepthook
threading.excepthook = lambda args: logger.critical(
    "线程异常", exc_info=(args.exc_type, args.exc_value, args.exc_traceback))
sys.unraisablehook = lambda u: logger.error(
    "析构期异常: %s", u.err_msg, exc_info=(u.exc_type, u.exc_value, u.exc_traceback))
```

**线上钩子的纪律**：不阻塞（异步队列上报）、不抛异常、限流（避免异常风暴打爆日志/上报配额）、脱敏。

### 12.2.3 `atexit` 与优雅退出

```python
import atexit

@atexit.register
def flush_metrics():
    ...
```

`atexit` 在正常退出（含 `sys.exit`）时执行，**被 `SIGKILL`/OOM 杀死时不执行**；容器里 `SIGTERM` 后的清理需要通过信号处理器显式实现。

## 12.3 无侵入采样：`py-spy` 是线上首选

```bash
# 抓一次所有线程的 Python 栈（卡死/高 CPU 第一命令）
py-spy dump --pid <pid>

# 实时看热点函数
py-spy top --pid <pid>

# 录制 30 秒火焰图（带 C 栈、线程状态）
py-spy record -o /tmp/prof.svg --pid <pid> --duration 30

# 在容器里（需要 SYS_PTRACE）
kubectl exec -it <pod> -- py-spy dump --pid 1
```

**优势**：不修改代码、不重启、开销 < 1%、能看到 GIL 等待和 C 扩展。**限制**：需要 ptrace 权限；对高度优化的短函数可能采样不到；`--native` 可看原生栈（需要符号）。

## 12.4 日志/指标/追踪在生产中的正确用法

| 手段 | 生产要点 | 反例 |
| :--- | :--- | :--- |
| 日志 | 分级、采样、结构化、带 `trace_id`/`request_id`、脱敏、限流 | 全量 DEBUG 刷盘、打印完整请求体 |
| 指标 | RED（Rate/Errors/Duration）+ USE（Utilization/Saturation/Errors） | 只看 CPU 不看 p99 |
| 追踪 | 采样率（如 1%–5%）、关键 span、错误标记 | 全量追踪导致开销与存储爆炸 |
| 错误聚合 | 按 fingerprint 去重、带版本/环境 | 每个异常都发一条告警 |

**从"用户报错"到"根因"的最短路径**：

```
告警/错误平台（含 trace_id）
  → 追踪系统看这条请求的耗时与失败 span
  → 日志系统按 trace_id 查该请求的全部日志
  → 若进程卡死/CPU 高：py-spy dump / faulthandler 栈
  → 若崩溃：core dump / 退出码 / 内核日志
```

## 12.5 核心转储（core dump）

**Linux**：

```bash
ulimit -c unlimited                          # 允许生成 core（对当前 shell 生效）
cat /proc/sys/kernel/core_pattern            # core 文件位置/命名
# systemd 系统：
coredumpctl list
coredumpctl gdb <pid>                        # 打开对应 core
(gdb) bt
(gdb) py-bt
```

**容器**：需要挂载可写目录作为 core 输出、设置 `core_pattern`、并确保 `ulimit` 足够；K8s 里通常要 privileged/自定义 `securityContext`。

**Windows**：

```powershell
procdump -ma <pid> app.dmp        # Sysinternals 抓完整转储
# 用 WinDbg 打开，!analyze -v 看崩溃点；Python 栈可用 py-spy dump 辅助
```

**什么时候值得抓 core**：C 扩展崩溃、段错误、解释器级异常、无法用 Python 层工具定位的问题。

## 12.6 远程 attach：最后手段

| 方式 | 前提 | 风险 |
| :--- | :--- | :--- |
| `py-spy dump/record` | ptrace 权限 | 低（只读采样） |
| `gdb -p` + `py-bt` | ptrace + 调试符号 | 中（暂停进程） |
| `debugpy --pid` | ptrace + 网络 | 高（等于开放调试端口） |
| `pystack remote <pid>` | ptrace + 安装 | 中（快照，较安全） |

**原则**：

1. 能用"采样/快照"就不用"暂停/attach"；
2. attach 前先在预发环境演练；
3. 设置超时，避免进程长时间挂起导致健康检查失败被重启；
4. 调试端口只监听本地，通过 SSH 隧道访问；
5. 事后移除调试代码/关闭端口，并检查是否留下安全风险。

## 12.7 四类典型故障的 Playbook

### 12.7.1 服务卡死/无响应

| 步骤 | 动作 |
| :--- | :--- |
| 1 | 确认是"全挂"还是"部分接口慢"（看指标、探活） |
| 2 | `py-spy dump --pid` 抓全线程栈，重点看是否都停在锁/`select`/`queue.get` |
| 3 | 若是锁等待：找持有者栈；若是 I/O 等待：看下游依赖 |
| 4 | 检查是否 GIL 被长任务占用（`py-spy top` 看热点） |
| 5 | 先摘流量/重启止血，保留栈与日志，事后定位 |

### 12.7.2 CPU 高

| 步骤 | 动作 |
| :--- | :--- |
| 1 | `py-spy top --pid` 看哪个函数在烧 CPU |
| 2 | 区分"Python 计算"、"C 扩展"、"GC"、"系统调用" |
| 3 | `py-spy record` 火焰图确认热点；必要时 `cProfile` 复现 |
| 4 | 若与流量相关：限流/扩容；若与数据相关：查异常输入（正则回溯、超大对象） |
| 5 | 若在 GC：看分配速率与对象增长（第 10 章） |

### 12.7.3 内存持续上涨

| 步骤 | 动作 |
| :--- | :--- |
| 1 | 看 RSS/容器 working set 曲线，区分阶跃/线性/锯齿 |
| 2 | 检查缓存/连接池/队列长度是否有上限 |
| 3 | `tracemalloc` 周期快照（若预先开启）或 `memray attach` |
| 4 | 若怀疑泄漏：`gc.get_objects` 统计类型增长、`objgraph` 引用链 |
| 5 | 先扩容/重启止血，保留快照；修复后加内存告警与回归测试 |

### 12.7.4 错误率突然升高

| 步骤 | 动作 |
| :--- | :--- |
| 1 | 按错误类型/接口/实例/版本分组，找集中点 |
| 2 | 对齐时间线：发布、配置、证书过期、依赖故障、流量突增 |
| 3 | 看异常栈（日志/Sentry），区分"新 bug"与"依赖失败" |
| 4 | 若依赖失败：重试/降级/熔断；若新版本引入：回滚 |
| 5 | 复盘：为什么测试没抓到？为什么监控没早发现？ |

## 12.8 无法复现问题的取证清单

| 类别 | 需要记录/获取 |
| :--- | :--- |
| 代码版本 | commit hash、镜像 tag、`__version__`、发布记录 |
| 运行环境 | Python 版本（`sys.version`）、OS/架构、容器镜像、locale、时区 |
| 依赖 | `pip freeze`/`uv.lock`、系统库版本、C 扩展版本 |
| 配置 | 生效配置（脱敏）、环境变量、feature flag |
| 输入 | 请求参数/消息体/数据文件（脱敏） |
| 时间线 | 首次出现、持续时长、频率、与变更的关系 |
| 资源 | CPU/内存/FD/线程/连接数曲线 |
| 日志 | 相关 `trace_id` 的完整日志、异常栈、慢查询 |
| 现场 | 线程栈、core dump、堆快照、网络连接状态 |

> 经验：**线上问题的"不可复现"通常只是"信息不全"**。把上表变成故障模板，每次按项填空，能显著缩短排障时间。

## 12.9 复盘（Postmortem）：把一次故障变成资产

**合格复盘的产出**：

1. **时间线**：从第一个异常信号到恢复，每步操作与观察；
2. **根因**：技术根因 + 为什么没被更早发现（流程/监控缺口）；
3. **影响面**：用户、数据、时长、金额；
4. **行动项**：可执行、有负责人、有截止时间（修复 bug、补测试、加监控、改流程）；
5. **防复发验证**：回归测试、故障演练、告警阈值调整。

**5 Whys 示例**：

```
服务 500 ↑
← 为什么？数据库连接池耗尽
← 为什么？慢查询占满连接
← 为什么？新上线接口缺少索引
← 为什么？评审没检查执行计划
← 为什么？上线 checklist 里没有"新查询必须 EXPLAIN"
→ 行动项：SQL 评审纳入 checklist；慢查询告警阈值为 1s；连接池加超时与监控
```

## 12.10 本章要点与坑清单

**要点**

- 线上调试先取证后操作：栈、日志、指标、core、时间线。
- `faulthandler` + `SIGUSR1` 是零依赖的"随时抓栈"方案。
- `py-spy dump/record` 是线上卡死/CPU 高的第一工具，开销 < 1%。
- 日志/指标/追踪通过 `trace_id` 关联，才能从告警直达根因。
- core dump 用于 C 层崩溃；OOM 靠监控与预防。
- attach 调试器是最后手段，安全与可用性风险最高。
- 四类故障（卡死/CPU/内存/错误率）各有 playbook；复盘要落到行动项。

**坑**

- 一出问题就重启，现场全丢。
- 在生产开 DEBUG 日志/全量追踪，把服务压垮。
- 公网暴露 debugpy 端口。
- 只盯 CPU 不看延迟分位数；只看单实例不看集群。
- 复盘写成"某某操作失误"，没有系统性行动项。
- 临时改动没有记录、没有回滚方案，引发二次故障。

---

# 第 13 章 测试驱动的调试：把 bug 变成不会复发的测试

## 13.1 调试的终点是测试

一次合格的调试，最终产物不是一个"看起来好了"的修改，而是**一个能证明 bug 存在、并在修复后通过的测试**。流程：

```
线上/测试中发现 bug
   ↓
最小复现（第 1 章）
   ↓
写成失败测试（红）
   ↓
修复代码
   ↓
测试变绿（绿）
   ↓
补充边界/回归用例，防止同类问题
```

好处：修复有据可查、回归可自动验证、后人能理解"为什么这样写"。

## 13.2 `pytest` 的调试能力

### 13.2.1 失败即进入调试器

```bash
pytest tests/test_order.py -x --pdb            # 第一个失败就进 pdb
pytest --pdb                                   # 每个失败都进 pdb
pytest --pdbcls=IPython.terminal.debugger:TerminalPdb   # 用 ipdb 风格
pytest --trace                                 # 从测试开始就进调试器（单步执行）
```

> `--pdb` 遇到 `--capture` 的默认捕获时，pdb 的输入/输出可能不完整。加 `-s`（关闭捕获）通常更顺手。

### 13.2.2 常用的"定位型"参数

| 参数 | 作用 |
| :--- | :--- |
| `-x` / `--maxfail=N` | 第一个失败就停 / 最多 N 个失败 |
| `--lf` / `--ff` | 只跑上次失败的 / 先跑上次失败的 |
| `--sw` / `--stepwise` | 逐步修复模式（失败处停，修好继续） |
| `-k "expr"` | 按名称表达式筛选用例 |
| `-s` / `--capture=no` | 不捕获 stdout/stderr（看到 print 和交互输出） |
| `--showlocals` | 失败时显示局部变量（**非常有用**） |
| `--tb=short/long/native/no` | 控制 traceback 详细度；`native` 用标准 traceback |
| `--durations=10` | 显示最慢的 10 个用例/阶段 |
| `--setup-show` | 显示 fixture 的 setup/teardown 过程 |
| `--fixtures` | 列出可用 fixture |
| `--co` / `--collect-only` | 只收集不执行，排查"用例没被发现" |
| `--log-cli-level=DEBUG` | 实时输出日志（配合 `caplog`） |
| `--full-trace` | 完整 traceback（包括被 pytest 截断的部分） |

### 13.2.3 `pytest.ini` / `pyproject.toml` 里的调试配置

```toml
[tool.pytest.ini_options]
addopts = "-ra --strict-markers --showlocals"
testpaths = ["tests"]
log_cli = false
log_cli_level = "INFO"
filterwarnings = [
    "error",                       # 警告视为错误（提前暴露弃用/资源问题）
    "ignore::DeprecationWarning:thirdparty.*",
]
```

`filterwarnings = ["error"]` 是把"警告类 bug"提前暴露的高性价比手段（注意排除已知的第三方警告）。

## 13.3 `fixtures` 调试

| 问题 | 表现 | 排查 |
| :--- | :--- | :--- |
| fixture 没生效 | 测试里拿到默认值 | `--setup-show` 看 setup 顺序 |
| 作用域错误 | 数据被多个测试共享、互相污染 | 检查 `scope="session/module/function"` |
| 资源未清理 | 文件/socket 泄漏、测试间干扰 | 用 `yield` fixture 并在 `finally` 清理 |
| 依赖链太深 | 报错栈看不清根因 | 拆小 fixture，或用工厂函数 |
| 异步 fixture 未等待 | 事件循环已关闭/任务未完成 | 用 `pytest-asyncio` 的 `async def` fixture + `await` |

```python
import pytest

@pytest.fixture
def db():
    conn = create_connection()
    try:
        yield conn
    finally:
        conn.close()          # 确保清理，即使测试失败

def test_insert(db):
    ...
```

**fixture 调试技巧**：`--setup-show` 会打印 `SETUP`/`TEARDOWN` 与作用域，能立刻发现"setup 没跑"或"teardown 顺序不对"。

## 13.4 `mock` 与 `monkeypatch`：替换依赖来隔离问题

```python
from unittest.mock import patch, MagicMock
import myapp.orders as orders

# ✅ patch "使用处"的名字，而不是"定义处"
with patch.object(orders, "send_email") as m:
    orders.create_order(...)
    m.assert_called_once()
```

| 坑 | 说明 | 正确做法 |
| :--- | :--- | :--- |
| patch 错位置 | patch 了 `email.send_email`，但 `orders` 已经 `from email import send_email` 绑定了引用 | patch `orders.send_email` |
| 过度 mock | 把所有依赖都 mock 掉，测试只验证"mock 被调用" | 只 mock 边界（网络/时钟/随机），核心逻辑用真实实现 |
| 没有 `autospec` | mock 接受任意参数，拼写错误测不出来 | `patch(..., autospec=True)` 或 `create_autospec` |
| 忘记清理 | 全局状态被污染，影响后续测试 | `monkeypatch` 自动回滚；`patch` 用 `with`/装饰器 |
| mock 掩盖真实错误 | 测试通过但线上失败 | 至少保留一个"集成测试"用真实依赖 |

```python
def test_retry(monkeypatch):
    calls = {"n": 0}

    def fake_get(url):
        calls["n"] += 1
        if calls["n"] < 3:
            raise TimeoutError
        return {"ok": True}

    monkeypatch.setattr("myapp.client.http_get", fake_get)   # 自动回滚
    assert myapp.client.fetch_with_retry("http://x") == {"ok": True}
    assert calls["n"] == 3
```

## 13.5 `unittest` 与 `doctest`

| 工具 | 调试价值 |
| :--- | :--- |
| `unittest.TestCase.subTest` | 一个用例里循环多个输入，失败时定位到具体子用例 |
| `assertEqual` 的 diff | 大对象比较失败时给出差异（比 `assert a == b` 信息多） |
| `unittest.mock` | 与 pytest 通用 |
| `doctest` | 文档即测试，适合验证小函数行为与示例正确性 |
| `python -m doctest -v module.py` | 运行文档中的示例并显示详细结果 |

```python
def add(a: int, b: int) -> int:
    """返回两数之和。

    >>> add(1, 2)
    3
    >>> add(0, 0)
    0
    """
    return a + b
```

> `doctest` 的失败信息会精确显示"期望 vs 实际"，是验证"文档是否与代码一致"的廉价手段。

## 13.6 属性测试（Hypothesis）：让机器帮你找边界

手写用例只能覆盖你想到的输入；属性测试会生成大量随机输入，并**自动缩小到最小失败样本**（shrinking），特别适合边界/编码/数值 bug。

```python
from hypothesis import given, strategies as st, settings

@given(st.lists(st.integers(), max_size=100))
def test_sort_is_idempotent(xs):
    assert sorted(sorted(xs)) == sorted(xs)

@given(st.text())
def test_json_roundtrip(s):
    assert json.loads(json.dumps(s)) == s
```

| 特性 | 价值 |
| :--- | :--- |
| 自动 shrink | 从 1000 个元素缩到触发失败的 2 个元素 |
| 数据库（`.hypothesis/`） | 记录失败样例，下次优先重跑（可作为回归用例） |
| `@example(...)` | 固定特定边界用例 |
| `settings(max_examples=...)` | 控制生成数量（CI 可调小） |
| 状态机测试 | `RuleBasedStateMachine` 测试并发/状态迁移 |

> 适合：解析器、序列化、排序/集合运算、数值计算、状态机。**不适合**：依赖真实网络/数据库的集成逻辑（除非用 fake）。

## 13.7 故障注入：提前演练"坏路径"

线上 bug 很多出现在异常路径（超时、部分失败、重试、磁盘满）。调试这些路径最好的方式是**主动注入故障**：

| 故障 | 注入方式 |
| :--- | :--- |
| 网络超时/错误 | mock 客户端抛 `TimeoutError`；toxiproxy 注入延迟/断连 |
| 依赖返回异常数据 | fake 服务返回畸形 JSON/空列表/超大响应 |
| 磁盘满/只读 | 临时目录 + mock `open`；容器限制 |
| 时钟跳变 | `freezegun`/`time-machine` 或注入 `Clock` |
| 内存不足 | `ulimit`/cgroup 限制，配合 `MemoryError` 测试 |
| 并发冲突 | `threading.Barrier` 制造同时进入临界区 |
| 进程被 kill | 测试优雅退出逻辑（发 `SIGTERM`，检查清理） |

```python
def test_handles_partial_failure(monkeypatch):
    def flaky(*args, **kwargs):
        raise ConnectionError("injected")
    monkeypatch.setattr("myapp.client.call", flaky)

    with pytest.raises(MyAppError) as ei:
        myapp.service.run()
    assert "injected" in str(ei.value.__cause__)      # 异常链保留了根因
```

## 13.8 测试本身的调试：为什么"测试通过、线上失败"

| 差异来源 | 表现 | 对策 |
| :--- | :--- | :--- |
| 环境 | 本地 SQLite/线上 PostgreSQL；本地 mock/线上真实 | 集成测试/容器化测试环境 |
| 时间 | 依赖 `datetime.now()`、时区、夏令时 | 注入时钟、固定时区（UTC） |
| 随机 | 测试偶尔失败（flaky） | 固定种子、记录种子、失败重跑机制 |
| 顺序 | 单独跑通过、全量跑失败（测试污染） | 隔离 fixture、清理全局状态、`pytest-randomly` 暴露依赖 |
| 并发 | 本地单线程、线上多线程 | 并发测试、`pytest-xdist`、不变量断言 |
| 数据量 | 小数据集通过、大数据集 O(n²) 超时 | 基准测试、性能回归 |
| 版本 | 本地 3.13、线上 3.10 | 多版本 CI（tox/nox） |

**Flaky 测试处理原则**：

1. 第一次出现就记录（issue + seed + 时间），不要"重跑三次就过"；
2. 用 `pytest-randomly` 固化/随机化顺序来暴露顺序依赖；
3. 对时间/随机/网络依赖做注入；
4. 允许重试的只应是**外部不稳定依赖**，而不是自己的逻辑。

## 13.9 覆盖率与变异测试：检验"测试是否真的有效"

```bash
coverage run -m pytest
coverage report -m                # 显示未覆盖行
coverage html                     # 生成 HTML 报告
coverage run --branch -m pytest   # 分支覆盖
```

| 指标 | 说明 | 局限 |
| :--- | :--- | :--- |
| 行覆盖率 | 哪些行被执行过 | 执行过 ≠ 验证过 |
| 分支覆盖率 | if/循环两个方向是否都走过 | 仍不检查断言质量 |
| 变异测试 | 主动改坏代码，看测试是否失败（`mutmut`、`cosmic-ray`） | 慢，适合核心模块 |

> 覆盖率是"发现未测试区域"的工具，不是"质量目标"。为了 100% 覆盖率写无断言测试，是自欺欺人。

## 13.10 本章要点与坑清单

**要点**

- 调试的终点是一个失败→通过的测试；没有测试的修复等于没修。
- `pytest --pdb`/`--trace`/`--showlocals`/`--tb` 是调试测试失败的主力。
- `--setup-show` 排查 fixture；`monkeypatch` 自动回滚；patch 要 patch "使用处"。
- Hypothesis 用于边界/解析/数值类 bug，shrinking 直接给出最小反例。
- 故障注入提前覆盖异常路径；flaky 测试要当 bug 处理。
- 覆盖率找盲区，变异测试验证测试有效性。

**坑**

- 修完 bug 不写回归测试。
- `filterwarnings` 没开，弃用/资源警告被忽略。
- mock 过度，测试只验证"mock 被调用"。
- 测试依赖真实时间和网络，导致 flaky。
- 测试之间共享全局状态，顺序一变就失败。
- 把覆盖率当 KPI，写出无断言的"假测试"。

---

# 第 14 章 常见错误类型与经典陷阱案例库

## 14.1 错误类型速查表

### 14.1.1 语法/导入/环境

| 异常/现象 | 常见根因 | 定位 | 修复 |
| :--- | :--- | :--- | :--- |
| `SyntaxError: invalid syntax` | 少括号/逗号、f-string 引号嵌套、用 Python 2 语法 | 看 `^` 指向；`python -m py_compile file.py` | 修正语法；确认解释器版本 |
| `IndentationError` / `TabError` | 空格与 Tab 混用 | `cat -A`/编辑器显示空白 | 统一 4 空格；配置编辑器 |
| `ModuleNotFoundError` | 没装、虚拟环境不对、`sys.path` 不含项目根、包名拼错 | `python -c "import sys; print(sys.executable, sys.path)"`、`pip list` | 激活正确环境、`pip install`、`-m` 运行 |
| 循环导入 | A/B 互相 import | `-X importtime`、traceback 里 `partially initialized module` | 延迟导入、`TYPE_CHECKING`、拆公共模块 |
| `ImportError: cannot import name X` | 名字不存在/版本不匹配/循环导入 | `python -c "import m; print(dir(m))"` | 核对版本与导出名 |
| 导入的是"另一个同名包" | 本地文件遮蔽、`sys.path` 顺序 | `print(module.__file__)` | 重命名本地文件、调整路径 |
| 依赖冲突 | 版本不兼容 | `pip check`、`pipdeptree` | 引脚版本、重建环境 |
| 改了代码不生效 | `__pycache__`、已安装包、`sys.modules` | `python -B`、删缓存、看 `__file__` | 清理缓存、`pip install -e .` |

### 14.1.2 名称/属性/类型

| 异常 | 常见根因 | 定位 | 修复 |
| :--- | :--- | :--- | :--- |
| `NameError` | 变量名拼错、作用域不对、忘记 import | 看 traceback 行 | 修正名字/导入 |
| `UnboundLocalError` | 函数内赋值使名字成为局部变量，但使用早于赋值 | `dis.dis` 看 `LOAD_FAST` | 提前初始化/`global`/`nonlocal` |
| `AttributeError: 'NoneType' ...` | 函数某分支返回 `None` | 在访问前 `print`/断点，检查返回值 | 处理 `None`、加类型标注 |
| `AttributeError: 'X' object has no attribute 'y'` | 拼写错误、版本 API 变化、`__slots__` | `dir(obj)`/`hasattr` | 修正/升级适配 |
| `TypeError: 'NoneType' object is not callable` | 同名变量覆盖了函数、装饰器返回 None | 看赋值链 | 重命名/修装饰器 |
| `TypeError: f() takes 1 positional argument but 2 were given` | 参数个数/`self` 绑定错误 | `inspect.signature(f)` | 修正调用/签名 |
| `TypeError: unsupported operand type(s) for +: 'int' and 'str'` | 类型混用（输入未转换） | 断点看类型 | 显式转换/校验 |
| `TypeError: unhashable type: 'list'` | 用可变对象做 dict key/set 元素 | 看 key 构造 | 转 `tuple`/`frozenset` |

### 14.1.3 数值/容器/比较

| 异常/现象 | 常见根因 | 定位 | 修复 |
| :--- | :--- | :--- | :--- |
| `KeyError` | 键不存在/类型不符（`'1'` vs `1`） | `print(d.keys())`、`d.get` | 用 `get`/`in` 判断、统一键类型 |
| `IndexError` | 越界、空列表、off-by-one | 打印 `len` 与索引 | 修正边界、用切片/迭代 |
| `ValueError: too many values to unpack` | 解包数量不匹配 | 打印实际对象 | 调整解包/用 `*rest` |
| `dict changed size during iteration` | 遍历时增删键 | 看循环体 | 遍历 `list(d.items())` 副本 |
| `RuntimeError: dictionary changed size`（同上） | 同上 | 同上 | 同上 |
| 结果差 1 / 越界 | 边界条件（`range(len+1)`、`<=` vs `<`） | 断点看循环变量与长度 | 修正边界，补测试 |
| 浮点比较失败 | 二进制浮点误差 | `repr(0.1+0.2)` | `math.isclose`、`Decimal` |
| `is` 比较失败 | 用 `is` 比较值/字符串 | 检查比较表达式 | 值比较用 `==`，`None` 用 `is` |

### 14.1.4 异常处理

| 现象 | 根因 | 修复 |
| :--- | :--- | :--- |
| 异常"消失" | `except: pass`、`except Exception: return` | 记录并重抛，至少 `logger.exception` |
| traceback 指向 `raise` 行而不是根因 | `raise e` 重置栈 | 用裸 `raise` |
| 原始异常丢失 | `raise NewError(str(e))` | `raise NewError(...) from e` |
| 异常被 `finally` 吞掉 | `finally` 中 `return`/`raise` | 不要在 `finally` 里返回 |
| `except` 顺序错误 | 父类异常在前面，子类分支永远不执行 | 子类在前，父类在后 |
| `KeyboardInterrupt` 被吞 | `except:`/`except BaseException` | 只捕 `Exception`，或显式重抛 |
| 异常组漏处理 | 用 `except` 处理 `ExceptionGroup` | 用 `except*`（3.11+） |

### 14.1.5 对象模型/框架

| 现象 | 根因 | 修复 |
| :--- | :--- | :--- |
| `dataclass` 实例共享列表 | 可变默认值 | `field(default_factory=list)` |
| `__eq__` 定义了但对象不能做 dict 键 | `__hash__` 被隐式设为 `None` | 同时定义 `__hash__`，或用 `frozen=True` |
| `__slots__` 子类报错 | 父类有 `__dict__`/未声明 slots | 所有父类都用 `__slots__` |
| `super()` 调用报错 | 多继承 MRO 顺序 | 查 `Cls.__mro__` |
| `property` 不生效/被覆盖 | 同名实例属性遮蔽 | 改名/用 `_x` 存储 |
| 枚举比较失败 | 拿字符串和枚举成员比较 | 用 `Enum("x")` 转换 |
| 上下文管理器吞异常 | `__exit__` 返回 `True` | 返回 `False`/`None` |
| 生成器提前结束 | 生成器只能消费一次 | 转 `list` 或重新创建 |

### 14.1.6 并发/资源/网络/编码

| 现象 | 根因 | 定位 | 修复 |
| :--- | :--- | :--- | :--- |
| 进程卡住 | 死锁/等待永不满足 | `py-spy dump`、`faulthandler` | 锁顺序/超时/显式唤醒 |
| 计数丢失 | 复合操作非原子 | 代码审查/压测 | 加锁/队列 |
| 内存持续上涨 | 缓存/泄漏/循环引用 | `tracemalloc`、`objgraph` | 有界缓存/弱引用/清理 |
| `Too many open files` | FD 泄漏 | `lsof`、`/proc/<pid>/fd` | `with`/连接池/提高上限 |
| `ConnectionResetError`/`BrokenPipeError` | 对端关闭、超时、协议错误 | 抓包/服务端日志 | 重试/心跳/超时配置 |
| `UnicodeDecodeError` | 编码不一致（GBK/UTF-8） | `print(data[:100])`、`chardet` | 显式 `encoding=`，`errors=` 谨慎 |
| `UnicodeEncodeError`（Windows 控制台） | 控制台编码非 UTF-8 | `sys.stdout.encoding` | `PYTHONIOENCODING=utf-8`、`reconfigure` |
| `SSLError: certificate verify failed` | 证书/代理/时间/CA 包 | `openssl s_client`、`certifi` | 更新 CA、配置代理、不要 `verify=False` 进生产 |
| 时间差 8 小时 | 时区处理错误（naive datetime） | 打印 `tzinfo` | 统一 UTC 存储、显式时区 |
| 路径找不到 | 相对路径/CWD 不同、Windows 反斜杠 | `print(os.getcwd(), path)` | `pathlib.Path(__file__).parent`、绝对路径 |

## 14.2 二十个经典陷阱（症状 / 根因 / 定位 / 修复）

### 陷阱 1：可变默认参数

```python
def append_bad(item, bucket=[]):        # ❌ 默认值只在定义时创建一次
    bucket.append(item)
    return bucket

print(append_bad(1), append_bad(2), append_bad(3))   # [1,2,3] [1,2,3] [1,2,3]
```

- **根因**：默认参数在函数定义时求值一次，所有调用共享同一对象；
- **定位**：多次调用后结果"累积"；`f.__defaults__` 可看到被污染的对象；
- **修复**：`bucket=None` + 函数内 `if bucket is None: bucket = []`；
- **静态检查**：`ruff` 的 `B006` 可直接发现。

### 陷阱 2：闭包延迟绑定

```python
funcs = [lambda: i for i in range(3)]
print([f() for f in funcs])             # [2, 2, 2]

funcs2 = [lambda i=i: i for i in range(3)]
print([f() for f in funcs2])            # [0, 1, 2]
```

- **根因**：闭包捕获的是变量（cell），不是值；循环结束后 `i == 2`；
- **定位**：`f.__code__.co_freevars` 可见 `i`；`dis` 显示 `LOAD_DEREF`；
- **修复**：默认参数绑定当前值、`functools.partial`、或工厂函数。

### 陷阱 3：`is` 与 `==`

```python
a = 256; b = 256
print(a is b)        # True（小整数缓存）
c = 257; d = 257
print(c is d)        # 同一代码对象内可能 True（常量折叠），交互式通常 False
print([] is [])      # False（每次新对象）
```

- **根因**：`is` 比较对象身份，不是值；小整数/短字符串驻留、常量折叠都会影响结果；
- **定位**：怀疑时打 `id()`；
- **修复**：值比较用 `==`，只有 `None`/`True`/`False`/哨兵对象用 `is`。

### 陷阱 4：`finally` 中的 `return` 吞异常

```python
def f():
    try:
        raise ValueError("lost")
    finally:
        return "from finally"

print(f())          # from finally —— 异常消失了
```

- **根因**：`finally` 的 `return` 覆盖了正在传播的异常；
- **修复**：`finally` 只做清理，不 `return`/`raise`。

### 陷阱 5：浅拷贝与深拷贝

```python
import copy
a = [[1, 2], [3, 4]]
b = a.copy()            # 浅拷贝：内层列表共享
b[0][0] = 99
print(a)                # [[99, 2], [3, 4]]  ← a 被改了

c = copy.deepcopy(a)    # 深拷贝：完全独立
```

- **定位**：修改一处却"另一处也变了"；`id(a[0]) == id(b[0])`；
- **修复**：需要独立副本时 `deepcopy`；注意深拷贝大对象很贵。

### 陷阱 6：遍历时修改容器

```python
d = {"a": 1, "b": 2}
for k in d:
    if k == "a":
        del d[k]        # RuntimeError: dictionary changed size during iteration
```

- **修复**：`for k in list(d):`，或先收集再修改。

### 陷阱 7：`except` 吞掉一切

```python
try:
    process()
except Exception:
    pass                # ❌ 问题被永久掩埋
```

- **修复**：至少 `logger.exception` 后重抛；只捕获预期异常。

### 陷阱 8：`raise e` 重置 traceback

```python
try:
    risky()
except Exception:
    raise                 # ✅ 保留原始 traceback

try:
    risky()
except Exception as e:
    raise e               # ❌ 新栈从这一行开始，原始调用位置被"抬高"
```

### 陷阱 9：`assert` 校验外部输入

```python
def create(name):
    assert name, "name required"   # -O 下失效
```

- **修复**：显式 `if not name: raise ValueError(...)`。

### 陷阱 10：`+=` 与不可变/可变对象

```python
t = (1, 2)
t += (3,)          # 新建元组，重新绑定
lst = [1]
lst += [2]         # 原地扩展（__iadd__）
```

- **坑**：函数默认参数里 `+=` 会改共享对象；`x += y` 对不可变对象会产生新对象，别处引用不受影响。

### 陷阱 11：整数/字符串驻留

- 小整数（-5~256）、短标识符字符串会驻留；
- 大整数/运行时拼接的字符串一般不会；
- 结论：**永远不要用 `is` 比较数值或字符串**。

### 陷阱 12：`datetime.now()` 与时区

```python
from datetime import datetime, timezone
naive = datetime.now()                    # 无时区，跨服务比较会错 8 小时
aware = datetime.now(timezone.utc)        # 推荐
```

- **修复**：存储用 UTC aware，展示时再转本地时区。

### 陷阱 13：浮点精度

```python
print(0.1 + 0.2 == 0.3)                  # False
from decimal import Decimal
print(Decimal("0.1") + Decimal("0.2") == Decimal("0.3"))   # True
```

- **修复**：金额用整数分或 `Decimal`；比较用 `math.isclose`。

### 陷阱 14：`dict.get` 与缺失值混淆

```python
value = d.get("k")            # 无法区分"键不存在"和"值为 None"
value = d.get("k", sentinel)  # 用哨兵区分
```

### 陷阱 15：字符串 `split`/`strip` 的意外行为

```python
"a,,b".split(",")      # ['a', '', 'b'] —— 保留空串
"  x  ".strip(" ")     # 'x'，但 strip 去掉的是字符集合，不是子串
"abc".strip("ab")      # 'c' —— 不是去掉前缀 'ab'
```

### 陷阱 16：`sort` 的稳定性与 key 类型

```python
sorted([1, "2"])        # TypeError: '<' not supported between int and str
sorted(items, key=lambda x: x.name)   # 同 key 时保持原顺序（稳定）
```

### 陷阱 17：`dataclass` 可变默认值

```python
from dataclasses import dataclass, field

@dataclass
class Bad:
    items: list = []            # ValueError: mutable default

@dataclass
class Good:
    items: list = field(default_factory=list)
```

### 陷阱 18：`__eq__` 与 `__hash__`

```python
class User:
    def __init__(self, uid): self.uid = uid
    def __eq__(self, other): return isinstance(other, User) and self.uid == other.uid
    # ❌ 没有 __hash__ → 不可哈希，不能放进 set/dict 键
```

- **修复**：定义 `__hash__`（基于不可变字段），或用 `@dataclass(frozen=True)`。

### 陷阱 19：多进程里"全局变量不共享"

```python
counter = 0

def worker():
    global counter
    counter += 1        # 每个进程各有一份，父进程看到的仍是 0
```

- **修复**：`multiprocessing.Value`/`Manager`/`Queue`/共享内存，或改用线程 + 锁。

### 陷阱 20：asyncio 里调用同步阻塞函数

```python
async def handler():
    requests.get(url)        # ❌ 阻塞整个事件循环
    await asyncio.to_thread(requests.get, url)   # ✅
```

- **定位**：asyncio debug 模式的慢回调警告；
- **修复**：`to_thread`/`run_in_executor`/换异步客户端。

## 14.3 本章要点与坑清单

**要点**

- 大多数异常有"标准剧本"：看类型 → 看消息 → 看最后一帧自己的代码 → 看异常链。
- 语法/导入类问题优先排查"环境"而不是代码；先确认 `sys.executable`、`sys.path`、`__file__`。
- 名称/类型类异常优先用 `dis`/`inspect.signature`/类型标注确认，而不是猜。
- 数值/容器类问题高度集中在边界、类型、浮点、容器语义上。
- 异常处理类问题几乎都源于"吞异常"和"丢失 traceback"。
- 经典陷阱（可变默认参数、延迟绑定、`is`、`finally return`、深浅拷贝）值得背下来。

**坑**

- 看到异常就 `try/except`，把根因埋掉。
- 用 `is` 比较值；用 `==` 比较 `None`。
- 在 `finally` 里 `return`。
- 以为多进程共享全局变量。
- 在协程里调用同步 I/O。
- 用 Python 2 的思维写 Python 3（除法、`print`、`unicode`、异常语法）。

---

# 第 15 章 决策树、速查表与检查清单

## 15.1 按症状选工具

```
程序出问题
│
├─ 根本跑不起来（语法/导入/环境）
│   ├─ SyntaxError / IndentationError ────────→ 看 ^ 位置；python -m py_compile
│   ├─ ModuleNotFoundError / ImportError ────→ sys.executable、sys.path、pip list、__file__
│   └─ 版本冲突 ─────────────────────────────→ pip check、pipdeptree、重建 venv
│
├─ 抛异常（有 traceback）
│   ├─ 读最后一行 + 最后一帧自己的代码
│   ├─ 有异常链 ─────────────────────────────→ 从最底层原始异常读起
│   ├─ 需要看现场变量 ───────────────────────→ breakpoint() / python -m pdb -c continue
│   └─ 异常被吞了 ───────────────────────────→ 搜 except、加 excepthook、logger.exception
│
├─ 不报错但结果不对（逻辑/数据）
│   ├─ 能稳定复现 ───────────────────────────→ 断点 + 条件断点 + display
│   ├─ 需要看"走了哪条分支" ─────────────────→ sys.settrace / coverage / 日志
│   ├─ 数据边界问题 ─────────────────────────→ 最小输入 + 属性测试(hypothesis)
│   └─ 怀疑类型/契约 ────────────────────────→ mypy/pyright + assert + 类型标注
│
├─ 慢（性能）
│   ├─ 先定指标（p50/p99/吞吐/CPU）──────────→ 压测 + 监控
│   ├─ 线上且不能停 ─────────────────────────→ py-spy top / record / dump
│   ├─ 本地可复现 ───────────────────────────→ cProfile → line_profiler
│   ├─ 启动慢 ───────────────────────────────→ python -X importtime
│   └─ 异步/多线程 ──────────────────────────→ yappi / pyinstrument / asyncio debug
│
├─ 内存涨 / 资源泄漏
│   ├─ 先判断曲线形态（线性/阶跃/锯齿）─────→ 排除缓存与碎片
│   ├─ Python 对象 ──────────────────────────→ tracemalloc diff → gc.get_referrers
│   ├─ C 层/numpy ───────────────────────────→ memray / massif / RSS 对比
│   ├─ FD/连接/线程 ─────────────────────────→ psutil、lsof、/proc/<pid>/fd
│   └─ OOM ──────────────────────────────────→ cgroup 指标 + core dump + 预防
│
├─ 卡死 / 无响应
│   ├─ 先抓全线程栈 ─────────────────────────→ py-spy dump / faulthandler
│   ├─ 锁等待 ───────────────────────────────→ 找持有者；画等待环
│   ├─ I/O 等待 ─────────────────────────────→ 下游依赖、超时配置
│   └─ 事件循环阻塞 ─────────────────────────→ asyncio debug + py-spy
│
├─ 偶发 / 无法复现
│   ├─ 并发相关 ─────────────────────────────→ 压力测试 + Barrier 控时序 + 不变量断言
│   ├─ 随机/哈希/时间 ───────────────────────→ 固定种子、PYTHONHASHSEED、注入时钟
│   ├─ 环境相关 ─────────────────────────────→ 记录版本/配置/依赖/输入；对照实验
│   └─ 线上限定 ─────────────────────────────→ 日志/指标/追踪/trace_id 关联
│
└─ 线上故障
    ├─ 先止血（限流/降级/回滚），同时取证
    ├─ 栈 ───────────────→ py-spy dump / faulthandler(SIGUSR1) / gdb
    ├─ 错误 ─────────────→ 日志聚合/Sentry/trace 关联
    ├─ 崩溃 ─────────────→ core dump（Linux）/ procdump+WinDbg（Windows）
    └─ 复盘 ─────────────→ 时间线 + 根因 + 行动项 + 回归测试
```

## 15.2 按阶段的最佳实践

| 阶段 | 目标 | 关键动作 | 工具 |
| :--- | :--- | :--- | :--- |
| 写代码 | 让 bug 难写出来 | 类型标注、入口校验、显式异常、依赖注入 | mypy/ruff/IDE |
| 保存文件 | 最早反馈 | 实时 lint/类型检查、格式化 | ruff、pyright |
| 提交前 | 拦住低级错误 | pre-commit、单测、`-W error` | pre-commit、pytest |
| 测试失败 | 快速定位 | `--pdb`、`--showlocals`、`--tb`、fixture 调试 | pytest |
| 本地排查 | 理解运行时 | 断点、日志、`dis`、`inspect` | pdb、logging |
| 性能优化 | 找热点 | 采样 → 统计 → 行级 | py-spy、cProfile、line_profiler |
| 上线前 | 防事故 | 灰度、监控、告警、压测、故障演练 | 指标/追踪/告警 |
| 故障中 | 快速止血与取证 | playbook、栈、日志、core | py-spy、faulthandler、日志平台 |
| 复盘 | 防复发 | 时间线、根因、行动项、回归测试 | postmortem 模板 |

## 15.3 一页 `pdb` 速查

```text
启动：python -m pdb script.py | python -m pdb -m pkg.mod | breakpoint()
      python -m pdb -c continue script.py     # 直接进入异常现场
      python -m pdb -p <pid>                  # 3.14+ 附加到运行进程

执行：s(tep) 进入 | n(ext) 跳过 | r(eturn) 到返回 | c(ontinue) 继续
      unt(il) 到下一行/指定行 | j(ump) 跳行 | q(uit) 退出 | run/restart 重跑

断点：b(reak) 行号|函数 [, 条件]   tbreak 临时   cl(ear) 删除
      condition 编号 表达式        ignore 编号 N
      commands 编号 → silent/p/continue → end     # 日志断点
      disable/enable 编号

查看：l(ist) 源码 | ll 整个函数 | p 表达式 | pp 美化 | ! 执行语句
      w(here)/bt 栈 | u(p)/d(own) 切帧 | a(rgs) 参数 | rv 返回值
      whatis 类型 | source 源码 | display 表达式 | undisplay
      interact 进入 REPL | exceptions 异常链（3.13+）

文件：~/.pdbrc 与 ./.pdbrc（别名、display、启动命令；-c 在其后执行）
环境：PYTHONBREAKPOINT=0 禁用断点；PYTHONBREAKPOINT=pudb.set_trace 切换调试器
```

## 15.4 解释器命令行参数速查

| 参数 | 作用 | 调试用途 |
| :--- | :--- | :--- |
| `-m pdb script.py` | 以 pdb 运行脚本 | 从第一行调试 |
| `-m pdb -c continue` | 运行到异常 | post-mortem |
| `-m pdb -p <pid>` | attach（3.14+） | 线上/运行中进程 |
| `-i` | 脚本结束后进入交互 | 检查最终状态 |
| `-u` | 无缓冲 stdout/stderr | 崩溃前不丢输出 |
| `-B` | 不写 `.pyc` | 排除字节码缓存干扰 |
| `-O` / `-OO` | 优化（移除 assert、docstring） | 复现"生产行为不同" |
| `-W error` | 警告变错误 | 暴露弃用/资源警告 |
| `-X dev` | 开发模式 | faulthandler + 默认警告 + asyncio debug |
| `-X faulthandler` | 崩溃打印栈 | 段错误排查 |
| `-X tracemalloc=N` | 开启 tracemalloc | 内存分配追踪 |
| `-X importtime` | 导入耗时 | 启动慢 |
| `-X perf` | 支持 perf 剖析（3.12+） | Linux perf 采样 |
| `-v` | import 详细日志 | 导入问题 |
| `-m cProfile -s cumtime` | 剖析 | 热点 |
| `-m timeit` | 微基准 | 常数因子对比 |
| `-m trace --trace` | 逐行追踪 | 执行路径 |
| `-m pytest --pdb` | 测试失败进调试器 | 测试调试 |
| `-m asyncio` | 支持 top-level await 的 REPL（需 tty） | 异步交互调试 |

## 15.5 环境变量速查

| 变量 | 作用 | 典型值 |
| :--- | :--- | :--- |
| `PYTHONBREAKPOINT` | 控制 `breakpoint()` | `0`（禁用）、`pudb.set_trace` |
| `PYTHONFAULTHANDLER` | 崩溃时打印栈 | `1` |
| `PYTHONTRACEMALLOC` | 开启 tracemalloc | `10` |
| `PYTHONASYNCIODEBUG` | asyncio 调试模式 | `1` |
| `PYTHONDEVMODE` | 开发模式 | `1` |
| `PYTHONWARNINGS` | 警告过滤 | `error`、`default` |
| `PYTHONINSPECT` | 脚本后进交互 | `1` |
| `PYTHONHASHSEED` | 固定哈希种子 | `0`（仅排查用） |
| `PYTHONMALLOC` | 分配器（debug/malloc） | `debug`（开发期） |
| `PYTHONIOENCODING` | 标准流编码 | `utf-8` |
| `PYTHONPATH` | 额外模块搜索路径 | 项目根目录 |
| `PYTHONVERBOSE` | import 详细输出 | `1` |
| `PYTHONPROFILEIMPORTTIME` | 导入耗时 | `1` |
| `PYTHON_COLORS` | traceback/REPL 颜色（3.13+） | `0` / `1` |
| `NO_COLOR` / `FORCE_COLOR` | 通用颜色控制 | 任意值 |
| `PYTHON_DISABLE_REMOTE_DEBUG` | 禁用 PEP 768 远程调试（3.14+） | 任意值 |
| `PYTHONOPTIMIZE` | 等价 `-O` | `1` |

## 15.6 三张检查清单

### 15.6.1 提交代码前

- [ ] 异常没有被 `except: pass` 吞掉；包装异常时用了 `from e`；
- [ ] 错误信息包含上下文（入参、ID、路径），且不泄露敏感信息；
- [ ] 没有遗留 `breakpoint()`、`pdb.set_trace()`、调试用 `print`；
- [ ] 关键路径有测试；修复 bug 时补了回归用例；
- [ ] 类型标注/lint 通过；没有新增无理由的 `# type: ignore` / `# noqa`；
- [ ] 资源用 `with`/`finally` 关闭；日志级别合适；
- [ ] 断言只用于内部不变量，没有用于校验外部输入。

### 15.6.2 上线前

- [ ] 有结构化日志，关键路径带 `request_id`/`trace_id`；
- [ ] 有错误率/延迟/资源指标与告警阈值；
- [ ] 有 `faulthandler` 或等价的崩溃/卡死取证手段；
- [ ] 有超时、重试、熔断、降级配置；
- [ ] 灰度/回滚方案可用，且演练过；
- [ ] 依赖版本、配置、数据迁移都已确认；
- [ ] 故障 playbook（卡死/CPU/内存/错误率）可查；
- [ ] 禁止在生产开启的调试端口/开关已确认关闭。

### 15.6.3 排障时

- [ ] 先记录影响面与时间线，再动手；
- [ ] 先取证（栈、日志、指标、core），再重启/回滚；
- [ ] 用 `trace_id` 把日志/追踪/错误串起来；
- [ ] 每一步操作都记录（谁、何时、改了什么、结果）；
- [ ] 临时修改有回滚方案，并设置到期时间；
- [ ] 根因确认前不要"顺手重构"；
- [ ] 恢复后 24 小时内完成复盘与行动项登记。

## 15.7 团队规范建议

| 规范 | 具体要求 |
| :--- | :--- |
| 异常规范 | 自定义业务异常基类；禁止裸 `except`；跨层包装用 `from e`；异常消息含上下文 |
| 日志规范 | 结构化字段、稳定事件名、敏感字段统一脱敏、禁止 `print` 进主流程 |
| 可调试性 | 公开函数有类型标注；关键不变量有断言/校验；时钟/随机/HTTP 可注入 |
| 调试代码 | 禁止提交 `breakpoint()`/`pdb.set_trace()`；CI 用 lint 规则拦截 |
| 测试 | 新功能必须有用例；bug 修复必须有回归测试；禁止 flaky 测试长期存在 |
| 可观测性 | 所有服务有 RED/USE 指标；错误聚合；关键链路有追踪 |
| 线上操作 | 调试端口不暴露公网；attach 需审批；变更留痕 |

## 15.8 二十条快问快答

1. **`pdb` 和 `print` 选哪个？** 需要看"多个变量的实时状态和调用栈"用 pdb；只需要确认"有没有执行/值是多少"用日志/print。
2. **为什么生产不能留 `breakpoint()`？** 会让进程挂起等待输入；用 `PYTHONBREAKPOINT=0` 全局禁用并加 lint 规则。
3. **`assert` 能校验用户输入吗？** 不能，`-O` 会移除；用显式 `raise`。
4. **`logger.error(str(e))` 有什么问题？** 丢失 traceback；用 `logger.exception`/`exc_info=True`。
5. **`raise e` 和 `raise` 的区别？** 前者重置 traceback 起点，后者保留原始栈。
6. **为什么 `except Exception` 抓不到 `KeyboardInterrupt`？** 它是 `BaseException` 子类，不在 `Exception` 层级下。
7. **`is` 能比较小整数吗？** 结果依赖驻留/常量折叠，不可靠；值比较用 `==`。
8. **为什么函数默认参数 `[]` 会累积？** 默认值只在定义时创建一次，所有调用共享。
9. **为什么循环里的 lambda 都返回最后一个值？** 闭包按变量（cell）捕获，不是按值；用默认参数绑定。
10. **`cProfile` 该看 `tottime` 还是 `cumtime`？** 找热点看 `tottime`，找入口看 `cumtime`。
11. **线上卡死第一步做什么？** `py-spy dump --pid`（或 `faulthandler`）抓全线程栈，不要急着重启。
12. **内存涨就是泄漏吗？** 不一定，可能是缓存/分配器碎片；用 tracemalloc 对比与曲线形态判断。
13. **`tracemalloc` 能看到 numpy 数组吗？** 看不到 C 层分配；用 memray/RSS 对比。
14. **为什么多进程里全局计数不对？** 每个进程有独立内存；用 `Value`/`Manager`/`Queue`。
15. **asyncio 里调用 `requests.get` 会怎样？** 阻塞整个事件循环；用 `asyncio.to_thread`。
16. **`Task exception was never retrieved` 说明什么？** 任务异常没人取回；`await` 任务或用 `TaskGroup`。
17. **断点为什么不生效？** 路径映射、`justMyCode`、错误进程、优化模式、代码版本不一致。
18. **attach 调试器安全吗？** 调试端口等于代码执行入口；只监听本地、走隧道、用完即关。
19. **`-X dev` 有什么用？** 开启 faulthandler、默认警告过滤、asyncio debug 等开发期检查。
20. **修完 bug 最重要的一步？** 写一个能复现该 bug 的失败测试，并确保它现在通过。

---

# 附录 A 版本差异速查（调试相关）

> 说明：3.7–3.13 的条目以 3.13.5 实测 + 官方文档为准；3.14 条目来自 3.14 官方文档，**具体以对应版本文档为准**。

| 版本 | 调试相关变化 | 对调试的影响 |
| :--- | :--- | :--- |
| 3.7 | `breakpoint()` 内置函数；`-X dev` 开发模式；`pdb -m` | 断点入口标准化；开发期检查一键开启 |
| 3.8 | `threading.excepthook`；`sys.unraisablehook`；f-string `=` 调试写法；`logging.basicConfig(force=)` | 子线程/析构期异常可捕获；日志配置可覆盖 |
| 3.9 | `ast.unparse`；`zoneinfo`；dict 合并运算符 `\|`；`str.removeprefix` | 生成/查看代码更易；时区问题可用标准库解决 |
| 3.10 | `traceback.print_exception(exc)` 单参数形式；更精确的 `SyntaxError`；`zip(strict=True)`；`match` | traceback API 简化；语法错误定位更准；静默截断可暴露 |
| 3.11 | `ExceptionGroup`/`except*`；`BaseException.add_note()`；PEP 657 精确列指示；`sys.exception()`；`TaskGroup`；`.pdbrc` 改 UTF-8 | 并发多失败可表达；traceback 直接指出出错子表达式；异步任务管理更安全 |
| 3.12 | `sys.monitoring`（PEP 669）；`-X perf`；PEP 701 f-string；pdb 便捷变量；多线程进程 `os.fork()` 告警 | 低开销动态追踪；调试器/覆盖率工具性能提升；fork 风险显式化 |
| 3.13 | PEP 667 `FrameLocalsProxy`（`f_locals` 可写回）；彩色 traceback（`PYTHON_COLORS`）；pdb `exceptions` 命令、`set_trace()` 立即进入、`post_mortem` 支持异常对象；`interact` 支持 `exit()` | 调试工具可真正修改变量；traceback 更易读；链式异常可在 pdb 中切换；异步/交互体验改善 |
| 3.14+ | PEP 768 远程调试接口（`python -m pdb -p <pid>`、`sys.remote_exec()`、`-X disable-remote-debug`、`PYTHON_DISABLE_REMOTE_DEBUG`）；pdb 双后端（`settrace`/`monitoring`）；`pdb.set_trace(commands=...)`；`pdb.set_trace_async()`；`$_asynctask` 便捷变量；PEP 649 惰性注解 | 不依赖 ptrace 的安全 attach；调试器可在低开销后端运行；协程内可安全进调试器；`inspect`/框架获取注解的时机变化需注意 |

**跨版本排查提醒**：

- 先打印 `sys.version` 和 `sys.version_info`，不要假设"用户环境是 3.13"；
- traceback 的格式（列指示、彩色、异常组）随版本变化，网上搜到的输出可能对不上；
- `f_locals` 可写、`sys.monitoring` 可用、`ExceptionGroup` 存在与否，都是 3.11+/3.12+/3.13+ 的分水岭；
- 3.14 的惰性注解（PEP 649）会改变 `__annotations__` 的求值时机，依赖注解的调试/校验工具需要适配。

---

# 附录 B 代码片段库

## B.1 通用栈快照（含"谁调用了我"）

```python
import sys

def stack_snapshot(limit: int = 20) -> str:
    frame = sys._getframe(1)
    lines = []
    while frame and len(lines) < limit:
        code = frame.f_code
        lines.append(f"{code.co_filename}:{frame.f_lineno} in {code.co_name}")
        frame = frame.f_back
    return "\n".join(lines)
```

## B.2 耗时日志装饰器

```python
import functools, logging, time

logger = logging.getLogger(__name__)

def log_duration(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        t0 = time.perf_counter()
        try:
            return fn(*args, **kwargs)
        finally:
            cost = (time.perf_counter() - t0) * 1000
            logger.info("%s 耗时 %.2fms", fn.__qualname__, cost)
    return wrapper
```

## B.3 异常转结构化 JSON

```python
import json, traceback

def exception_payload(exc: BaseException) -> dict:
    tb = traceback.TracebackException.from_exception(exc)
    return {
        "type": type(exc).__name__,
        "message": str(exc),
        "chain": [f"{f.filename}:{f.lineno}:{f.name}" for f in traceback.extract_tb(exc.__traceback__)],
        "notes": list(getattr(exc, "__notes__", ())),
        "formatted": "".join(tb.format()),
    }
```

## B.4 `tracemalloc` 泄漏对比

```python
import tracemalloc

def diff_allocations(fn, top: int = 10):
    tracemalloc.start(15)
    before = tracemalloc.take_snapshot()
    fn()
    after = tracemalloc.take_snapshot()
    for stat in after.compare_to(before, "traceback")[:top]:
        print(f"{stat.size_diff / 1024:+.1f} KiB  {stat.count_diff:+d}  {stat.traceback[0]}")
    tracemalloc.stop()
```

## B.5 全线程栈 dump（管理端点用）

```python
import sys, threading, traceback

def dump_threads() -> str:
    frames = sys._current_frames()
    out = []
    for t in threading.enumerate():
        out.append(f"\n=== {t.name} (alive={t.is_alive()}) ===")
        frame = frames.get(t.ident)
        if frame:
            out.extend(traceback.format_stack(frame))
    return "".join(out)
```

## B.6 带退避的重试

```python
import random, time

def retry(fn, *, attempts: int = 3, base: float = 0.2, errors=(TimeoutError, ConnectionError)):
    for i in range(attempts):
        try:
            return fn()
        except errors:
            if i == attempts - 1:
                raise
            time.sleep(base * 2 ** i * (0.5 + random.random()))
```

## B.7 环境信息 dump（排查"环境不同"）

```python
import os, platform, sys, sysconfig

def env_report() -> dict:
    return {
        "python": sys.version,
        "executable": sys.executable,
        "platform": platform.platform(),
        "machine": platform.machine(),
        "cwd": os.getcwd(),
        "sys.path": sys.path,
        "prefix": sys.prefix,
        "base_prefix": sys.base_prefix,       # 是否在虚拟环境
        "encoding": sys.getdefaultencoding(),
        "filesystem_encoding": sys.getfilesystemencoding(),
        "stdlib": sysconfig.get_paths()["stdlib"],
    }
```

## B.8 一行 JSON 日志配置

```python
import logging, json, sys

class JsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        return json.dumps({
            "level": record.levelname,
            "logger": record.name,
            "msg": record.getMessage(),
            "time": self.formatTime(record),
        }, ensure_ascii=False)

handler = logging.StreamHandler(sys.stderr)
handler.setFormatter(JsonFormatter())
root = logging.getLogger()
root.handlers[:] = [handler]
root.setLevel(logging.INFO)
```

## B.9 Unix 信号抓栈（线上卡死应急）

```python
import faulthandler, signal

# 仅 Unix 有效：kill -USR1 <pid>
faulthandler.register(signal.SIGUSR1, all_threads=True)
```

## B.10 对象增长观察（轻量泄漏监控）

```python
import gc, collections

def object_growth(before: collections.Counter, top: int = 10):
    after = collections.Counter(type(o).__name__ for o in gc.get_objects())
    delta = {k: after[k] - before.get(k, 0) for k in after}
    return sorted(((v, k) for k, v in delta.items() if v > 0), reverse=True)[:top]
```

---

# 附录 C 参考资料

**官方文档**

- [pdb — Python 调试器](https://docs.python.org/3/library/pdb.html)
- [faulthandler — 转储 Python 回溯](https://docs.python.org/3/library/faulthandler.html)
- [traceback — 打印或获取栈回溯](https://docs.python.org/3/library/traceback.html)
- [sys.monitoring（PEP 669）](https://docs.python.org/3/library/sys.monitoring.html)
- [Python 性能分析器（profile/cProfile/pstats）](https://docs.python.org/3/library/profile.html)
- [tracemalloc — 内存分配追踪](https://docs.python.org/3/library/tracemalloc.html)
- [logging — 日志设施](https://docs.python.org/3/library/logging.html)
- [asyncio 开发模式](https://docs.python.org/3/library/asyncio-dev.html)
- [Python 开发模式（-X dev）](https://docs.python.org/3/library/devmode.html)
- [unittest.mock](https://docs.python.org/3/library/unittest.mock.html)
- [调试和性能分析（总览）](https://docs.python.org/3/library/debug.html)
- [gdb 辅助命令（python-gdb.py）](https://docs.python.org/3/library/gdb.html)
- [PEP 768 安全外部调试接口（3.14）](https://peps.python.org/pep-0768/)
- [PEP 667 帧局部变量访问](https://peps.python.org/pep-0667/)
- [PEP 669 低开销监控](https://peps.python.org/pep-0669/)
- [PEP 657 精确错误位置](https://peps.python.org/pep-0657/)

**工具**

- [py-spy](https://github.com/benfred/py-spy)：无侵入采样剖析
- [pyinstrument](https://github.com/joerick/pyinstrument)：调用树采样
- [scalene](https://github.com/plasma-umass/scalene)：CPU/内存/GPU
- [memray](https://github.com/bloomberg/memray)：内存剖析与 attach
- [objgraph](https://mg.pov.lt/objgraph/)：引用链可视化
- [pympler](https://github.com/pympler/pympler)：对象大小与统计
- [line_profiler](https://github.com/pyutils/line_profiler)：行级剖析
- [debugpy](https://github.com/microsoft/debugpy)：DAP 调试适配器
- [pdbpp](https://github.com/pdbpp/pdbpp) / [pudb](https://github.com/inducer/pudb) / [ipdb](https://github.com/gotcha/ipdb)：增强调试器
- [pytest](https://docs.pytest.org/) / [hypothesis](https://hypothesis.readthedocs.io/) / [coverage.py](https://coverage.readthedocs.io/)
- [ruff](https://docs.astral.sh/ruff/) / [mypy](https://mypy.readthedocs.io/) / [pyright](https://microsoft.github.io/pyright/) / [bandit](https://bandit.readthedocs.io/)
- [OpenTelemetry Python](https://opentelemetry.io/docs/languages/python/) / [Sentry Python](https://docs.sentry.io/platforms/python/)

**延伸阅读**

- CPython 源码：`Lib/pdb.py`、`Lib/bdb.py`、`Lib/traceback.py`、`Lib/faulthandler`（C 实现）、`Python/pystate.c`
- Brendan Gregg《Systems Performance》与火焰图方法
- 《Debugging: The 9 Indispensable Rules》——调试方法论经典

---

# 附录 D 术语表

| 术语 | 含义 |
| :--- | :--- |
| traceback | 异常发生时的调用栈记录，中文常称"回溯" |
| frame（帧） | 一次函数调用的运行时上下文，含局部变量、代码对象、行号 |
| breakpoint | 断点：程序执行到此处暂停，交给调试器 |
| post-mortem | 事后调试：异常发生后进入调试器检查现场 |
| 采样剖析 | 周期性抓取调用栈，按频率估算耗时（如 py-spy） |
| 统计剖析 | 通过钩子记录每次调用耗时（cProfile） |
| instrumentation | 插桩：插入记录代码以观测行为 |
| observability | 可观测性：日志 + 指标 + 追踪 |
| ptrace | Linux 系统调用，允许一个进程观察/控制另一个进程 |
| core dump | 进程崩溃时的内存转储文件 |
| GIL | 全局解释器锁，CPython 中限制同一时刻只有一个线程执行字节码 |
| race condition | 竞争条件：结果依赖不可控的执行时序 |
| deadlock | 死锁：多个执行单元互相等待对方持有的资源 |
| heisenbug | 观察行为本身会改变其表现（难以复现）的 bug |
| flaky test | 有时通过有时失败的不稳定测试 |
| shrink | 属性测试把失败输入自动缩小到最小反例 |
| fixture | pytest 中用于准备/清理测试环境的机制 |
| monkeypatch | 运行时替换模块/函数/属性（测试常用） |
| ABI / C 扩展 | 以 C 实现的 Python 扩展模块，Python 调试器无法进入其内部 |
| DAP | Debug Adapter Protocol，IDE 与调试器之间的协议 |
| PEP 768 | 3.14 引入的安全外部调试接口，支持不依赖 ptrace 的 attach |
