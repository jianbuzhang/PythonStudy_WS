# Python 性能与资源管理全面梳理：从测量、计算、内存到 I/O、并发与生产运维

> **文档定位**：一份可以直接当"性能调优手册 + 资源治理清单 + 考前讲义"使用的 Python 工程文档。它不只罗列"某个写法快几倍"，而是围绕一条主线展开：**先建立成本模型，再测量定位瓶颈，最后按"算法 → 数据结构 → 内置/向量化 → 并发 → 扩展/运行时"的顺序逐级优化；同时把 CPU、内存、文件描述符、线程、连接、磁盘与网络带宽当作有限资源来管理。**
>
> **适用版本**：以 CPython **3.8 – 3.13** 为基准，示例在 **3.13.5** 上验证，关键差异用 **3.12 / 3.11** 对照说明。涉及 3.9+/3.10+/3.11+/3.12+/3.13+ 的特性会显式标注版本。文中"本机实测"指 Windows 11 + CPython 3.13.5（64 位）；不同平台、不同小版本的数字会变，**结论以方法和趋势为主，不要照抄绝对值**。
>
> **配套题库**：同目录《Python性能与资源管理选择题_含答案解析.md》，共 **80 题**（单选 59、情景单选 5、多选 16），答案与解析默认折叠。建议读完一章做对应部分。
>
> **使用建议**：
> 1. 第一遍读第 1、2 章，建立"指标 + 测量"的底层习惯；
> 2. 第二遍按瓶颈类型精读：CPU 看第 3–7 章，内存看第 8 章，I/O 看第 9 章，并发看第 10 章，资源泄漏与限额看第 11–12 章；
> 3. 上线前只看第 13–15 章与各章末尾的"坑清单 / 检查表"；
> 4. 做题库时先独立作答，再展开解析，错题回到对应小节。

---

## 目录

- 第 1 章 总览：性能、资源与优化的心智模型
- 第 2 章 度量：先测量，再优化
- 第 3 章 CPython 运行时：Python 到底慢在哪里
- 第 4 章 算法与数据结构：决定数量级的第一杠杆
- 第 5 章 代码级优化：把常数因子压下去
- 第 6 章 缓存与惰性计算：用空间换时间，但要有边界
- 第 7 章 加速路线：NumPy、Cython、Numba、C/Rust 扩展与 PyPy
- 第 8 章 内存管理：引用计数、GC、对象开销与泄漏排查
- 第 9 章 I/O 性能：缓冲、批量、mmap、零拷贝、序列化与数据库
- 第 10 章 并发与并行：线程、进程、asyncio 的选型与背压
- 第 11 章 资源生命周期：with、ExitStack、子进程、临时文件与 finalizer
- 第 12 章 资源限额与容量：rlimit、cgroup、FD、线程与内存预算
- 第 13 章 启动、导入与部署性能
- 第 14 章 生产可观测性与调优 playbook
- 第 15 章 反模式、决策树、速查表与检查清单
- 附录 A 版本差异速查
- 附录 B 常用命令与代码片段
- 附录 C 参考资料

---

# 第 1 章 总览：性能、资源与优化的心智模型

## 1.1 性能不是一个数，而是一组指标

讨论"快不快"之前，先把指标说清楚；否则很容易用错误的指标做正确的优化，最后得到一个"跑分更快、线上更慢"的系统。

| 指标 | 含义 | 典型工具/接口 | 适用场景 |
| :--- | :--- | :--- | :--- |
| 墙钟时间（wall time） | 从开始到结束的真实时间，包含等待、调度、I/O | `time.perf_counter()` | 用户感知延迟、端到端耗时 |
| CPU 时间 | 进程实际占用 CPU 的时间，不含睡眠/等待 | `time.process_time()`、`resource.getrusage()` | 判断是否 CPU 密集、算力成本 |
| 吞吐量（throughput） | 单位时间完成的任务数/字节数 | 压测工具、计数器 | 批处理、数据管道、服务容量 |
| 延迟分位数（p50/p95/p99） | 多数请求与尾部请求的耗时分布 | 直方图、APM、压测 | 在线服务 SLA、尾延迟治理 |
| 内存峰值/RSS | 进程常驻内存、峰值内存 | `psutil`、`resource.getrusage().ru_maxrss` | 容器配额、OOM 风险 |
| 分配速率/GC 停顿 | 单位时间分配量、GC 暂停时长 | `tracemalloc`、`gc.callbacks` | 延迟毛刺、内存抖动 |
| 文件描述符/连接数 | 打开的文件、socket、管道数量 | `psutil.Process().num_fds()`、`/proc/self/fd` | 连接泄漏、`Too many open files` |
| I/O 吞吐/IOPS | 磁盘或网络的实际读写能力 | `psutil.io_counters()`、系统监控 | 数据密集任务、数据库 |
| 错误率/饱和度 | 失败比例、队列长度、CPU runqueue | 日志、监控、USE 方法 | 容量规划、故障定位 |

**三个必须同时看的维度**：

1. **快**：延迟与吞吐是否达标；
2. **省**：CPU、内存、I/O、连接数是否在预算内；
3. **稳**：p99、GC 停顿、内存增长、错误率是否可控。

只追求"平均耗时下降"而让 p99 翻倍、内存翻倍，通常不是优化，而是把问题推迟到高峰期爆发。

## 1.2 资源到底是什么：一张"预算表"

Python 进程消耗的资源远不止 CPU。生产事故里，"内存没超、CPU 不高，但服务挂了"的常见原因是文件描述符、线程数、连接池或磁盘空间耗尽。

| 资源 | 主要消费者 | 耗尽后的典型症状 | 主要治理手段 |
| :--- | :--- | :--- | :--- |
| CPU | 字节码执行、GC、序列化、压缩、加密、正则 | 延迟升高、runqueue 变长、CPU 打满 | 算法/向量化、减少解释器开销、进程池、扩容 |
| 内存（RSS） | 对象、容器、缓存、NumPy 数组、C 扩展、内存碎片 | OOMKilled、`MemoryError`、swap 抖动 | 流式处理、`__slots__`、有界缓存、分块、对象复用 |
| 虚拟内存（VSZ） | 线程栈、mmap、arena、保留地址空间 | 地址空间耗尽（32 位）、监控误报 | 控制线程数、`threading.stack_size`、理解 VSZ 与 RSS 差异 |
| 文件描述符（FD） | 文件、socket、管道、epoll、临时文件 | `OSError: [Errno 24] Too many open files` | `with`、连接池、及时 `close()`、提高 `RLIMIT_NOFILE` |
| 线程/进程 | 线程池、每请求一线程、子进程 | 上下文切换、内存翻倍、调度延迟 | 有界池、异步 I/O、进程数按核数、复用 worker |
| 连接 | 数据库、HTTP、Redis、消息队列 | 连接池耗尽、握手风暴、服务端拒绝 | 连接池、keep-alive、超时、熔断 |
| 磁盘空间 | 日志、临时文件、缓存、数据落盘 | `No space left on device` | 轮转、配额、临时目录清理、压缩 |
| 磁盘/网络带宽 | 大文件、序列化、备份、跨机房调用 | 吞吐上不去、超时 | 批量、压缩、零拷贝、就近访问、限流 |
| 锁/信号量 | 共享状态、连接池、限流器 | 死锁、线程饥饿、长尾延迟 | 缩小临界区、锁排序、超时、无锁设计 |
| 时间（超时预算） | 重试、排队、下游调用 | 请求堆积、级联失败 | 超时、重试预算、熔断、背压 |

**预算思维**：一个 worker 的 RSS × worker 数 + 共享缓存 + 解释器开销，必须小于容器内存上限并留出安全余量；FD 上限要覆盖"监听 socket + 每连接 FD + 文件/管道/日志 + 池化连接"；线程数要覆盖"池 + 后台任务 + GC/监控"。

## 1.3 一条铁律与两条定律

### 1.3.1 先测量，后优化（Measure twice, cut once）

没有测量的优化，99% 是在优化"你以为的瓶颈"。常见的错误路径：

- 凭直觉把 `for` 循环改成推导式，结果真正的瓶颈是数据库慢查询；
- 看到 CPU 高就加线程，结果是 GIL 下线程互相抢锁，CPU 更高、吞吐更低；
- 看到内存高就调大 GC 阈值，结果循环垃圾堆积，RSS 继续上涨；
- 用 `time.time()` 测耗时，系统时钟调整后得到负数；
- 在开发机上空载跑一次 `timeit`，就宣布"提速 3 倍"。

正确顺序是：**定义目标指标 → 建立可复现基线 → 用 profiler 定位热点 → 做一个最小改动 → 用同一基准复测 → 加回归保护**。

### 1.3.2 Amdahl 定律：优化没被执行的代码没有意义

若程序中可优化部分占比为 `p`，该部分加速比为 `s`，总体加速比上限为：

```text
speedup = 1 / ((1 - p) + p / s)
```

含义：

- 如果只有 20% 的时间花在可优化代码上，即使把这部分优化到无限快，总体最多快 `1 / 0.8 = 1.25` 倍；
- 反过来，先找占比最大的热点，收益最高；
- 并发也受此约束：串行部分决定扩展上限。

**实践推论**：优化前先问"这段代码占总时间多少？"如果不到 5%，除非它造成尾延迟或资源泄漏，否则不值得大动。

### 1.3.3 Little 定律：并发数、吞吐与延迟的关系

稳定系统中：

```text
并发数 L = 吞吐量 λ × 平均停留时间 W
```

例如下游平均延迟 50 ms，希望单进程吞吐 200 req/s，则平均需要 `200 × 0.05 = 10` 个并发在途请求。它解释了：

- 为什么"线程池开得越大越好"是错的：超过下游承载能力只会增加排队时间；
- 为什么延迟升高会反向压低吞吐（同样并发下，W 变大，λ 变小）；
- 为什么限流/背压是保护系统而不是"少做事"。

## 1.4 Python 的性能成本模型

Python 的快与慢，可以拆成几层成本。优化时从上往下找，通常越靠上收益越大：

| 层级 | 成本来源 | 典型量级 | 优先手段 |
| :--- | :--- | :--- | :--- |
| 1. 架构/算法 | 复杂度、重复计算、N+1 查询、全量加载 | 可能是 O(n) 与 O(n²) 的差别 | 换算法、加索引、批处理、缓存、流式 |
| 2. 数据搬运 | 序列化、拷贝、编码转换、跨进程传输 | 常被低估，可能占 30%+ | 零拷贝、批量、二进制协议、减少中间对象 |
| 3. 解释器开销 | 字节码分发、对象装箱、属性/名字查找、函数调用 | 每操作几十到几百纳秒 | 内置函数、向量化、局部变量、减少调用层数 |
| 4. 内存分配/GC | 对象创建、容器扩容、循环垃圾、缓存增长 | 分配越快、GC 越频繁 | 复用、`__slots__`、生成器、有界缓存、调 GC |
| 5. 系统调用/I/O | `read`/`write`、网络往返、磁盘寻道 | 微秒到毫秒级 | 缓冲、批量、mmap、异步、连接复用 |
| 6. 并发/调度 | 线程切换、锁竞争、进程启动、GIL | 微秒到毫秒级 | 选对并发模型、有界池、减少共享 |
| 7. 运行时/硬件 | 解释器版本、CPU 频率、内存带宽、缓存命中 | 常数因子 | 升级版本、向量化、数据布局、扩展 |

**一条重要经验**：在 Python 里，"少做一次 Python 层操作"往往比"把一次 Python 操作写得更漂亮"更有效。把 100 万次循环里的函数调用、属性访问、临时对象砍掉，收益通常大于微调表达式。

## 1.5 优化顺序：从杠杆最大的地方开始

推荐按下面的顺序逐级排查，不要跳级：

```text
第 0 级：需求与架构
  ├── 能否不查？→ 缓存、预计算、增量更新
  ├── 能否少传？→ 只取需要的列/字段、分页、投影
  ├── 能否批量？→ 批量读写、批量 RPC、合并小文件
  └── 能否异步/并行？→ 解耦、队列、流水线

第 1 级：算法与数据结构
  ├── 复杂度是否合理？→ 去嵌套循环、换 dict/set/heap/bisect
  ├── 是否重复计算？→ 提取循环不变量、memoization
  └── 是否全量加载？→ 生成器、流式、分块

第 2 级：内置与向量化
  ├── 能否用 C 实现的内置函数？→ sum/min/max/sorted/join/map
  ├── 能否用 NumPy/pandas 向量化？→ 去 Python 循环
  └── 能否用标准库的专用结构？→ deque/Counter/heapq/bisect

第 3 级：减少解释器开销
  ├── 局部变量绑定、减少属性查找
  ├── 减少函数调用/临时对象
  └── 选择合适的数据布局与对象模型

第 4 级：I/O 与并发
  ├── 缓冲/批量/零拷贝
  ├── 线程（I/O）/进程（CPU）/asyncio（高并发 I/O）
  └── 连接池、背压、超时

第 5 级：扩展与运行时
  ├── Cython/Numba/C/Rust 扩展
  ├── PyPy/JIT/free-threaded
  └── 升级 CPython、换硬件
```

> ⚠️ **不要在第 5 级解决第 1 级的问题**。用 C 扩展把一段 O(n²) 代码加速 10 倍，不如把它改成 O(n) 快几个数量级，而且后者更可维护。

## 1.6 按"症状"快速定位

| 症状 | 先怀疑 | 首选诊断 | 首选手段 |
| :--- | :--- | :--- | :--- |
| CPU 打满、延迟升高 | 热点循环、GC、序列化、压缩、正则 | `py-spy top`、`cProfile`、`perf` | 算法/向量化、减少分配、进程池 |
| CPU 不高但延迟高 | I/O 等待、锁、连接池、下游慢 | `py-spy dump`、分布式追踪、`strace` | 超时、连接池、异步、缓存 |
| 内存持续上涨 | 缓存无界、对象泄漏、异常引用、C 扩展 | `tracemalloc` 快照 diff、RSS 曲线 | 有界缓存、弱引用、修复引用链 |
| 周期性卡顿 | GC、日志轮转、批量任务、定时器 | `gc.callbacks` 统计、`py-spy` | 调 GC、分片、错峰、异步化 |
| FD 耗尽 | 文件/socket 未关闭、连接池泄漏 | `psutil.num_fds()`、`/proc/self/fd` | `with`、连接池、`-X dev` 抓 `ResourceWarning` |
| 线程/进程暴涨 | 每请求建线程、池无上限、递归创建 | 线程数监控、`faulthandler` | 有界池、信号量、复用 |
| 启动慢/冷启动高 | 重型 import、打包体积、初始化 | `-X importtime`、`PYTHONPROFILEIMPORTTIME` | 延迟导入、精简依赖、预编译 |
| 吞吐上不去但资源有余 | 串行处理、锁粒度、批太小、网络 RTT | 压测、追踪、队列深度 | 并发、批量、流水线、减少往返 |

## 1.7 本章要点与坑清单

**要点**

- 性能 = 延迟 + 吞吐 + 资源 + 稳定性，不能只看平均值。
- 资源是预算：CPU、内存、FD、线程、连接、磁盘、带宽、时间都要有上限。
- 优化前先测量；用 Amdahl 判断值不值得，用 Little 定律判断并发是否合理。
- 优化顺序：架构/算法 → 数据结构 → 内置/向量化 → 解释器开销 → I/O/并发 → 扩展/运行时。

**坑**

- 把"微基准快 2 倍"当成"系统快 2 倍"。
- 用平均延迟掩盖 p99 恶化。
- 只加机器/线程，不找串行瓶颈。
- 把缓存当免费午餐，不设上限、不设过期、不考虑失效。
- 在开发机空载环境测性能，忽略 CPU 频率、后台任务、容器配额。

---

# 第 2 章 度量：先测量，再优化

## 2.1 时间测量：别再用 `time.time()` 测耗时

| API | 含义 | 是否受系统时钟调整影响 | 典型用途 |
| :--- | :--- | :--- | :--- |
| `time.perf_counter()` | 最高精度的单调时钟，含睡眠/等待 | 否 | 测量墙钟耗时、基准测试 |
| `time.perf_counter_ns()` | 纳秒整数版 | 否 | 避免浮点误差的短区间 |
| `time.process_time()` | 当前进程 CPU 时间（用户+系统），不含睡眠 | 否 | 判断 CPU 密集、算力统计 |
| `time.monotonic()` | 单调时钟，精度通常低于 `perf_counter` | 否 | 超时、截止时间、间隔 |
| `time.time()` | 日历时间（Unix 时间戳） | **是** | 记录事件发生时刻、日志时间戳 |
| `datetime.now()` | 本地日历时间 | 是 | 人类可读时间、业务日期 |

**正确写法**：

```python
from time import perf_counter

start = perf_counter()
do_work()
elapsed = perf_counter() - start
print(f"elapsed: {elapsed:.6f}s")
```

**错误写法**：

```python
import time

start = time.time()          # ❌ 系统时钟被 NTP 调整时可能变负/跳变
do_work()
print(time.time() - start)
```

> 经验：**测耗时用 `perf_counter`，测 CPU 用 `process_time`，做超时/间隔用 `monotonic`，记录"几点发生"才用 `time.time`。**

## 2.2 `timeit`：微基准的正确打开方式

### 2.2.1 基本用法

```python
import timeit

# 方式 1：传字符串表达式（适合对比小片段）
t = timeit.timeit("'-'.join(str(n) for n in range(100))", number=10_000)
print(t / 10_000)

# 方式 2：传可调用对象（避免字符串编译开销）
def build():
    return "-".join(str(n) for n in range(100))

t = timeit.timeit(build, number=10_000)
print(t / 10_000)

# 方式 3：命令行，自动选择循环次数
# python -m timeit -s "data=list(range(1000))" "sum(data)"
```

### 2.2.2 关键细节

- `timeit` 会在计时期间**临时关闭 GC**（`gc.disable()`），避免 GC 抖动污染结果；这既是优点也是偏差——真实程序里 GC 是成本的一部分，涉及大量容器分配时要单独测 GC。
- 默认 `number=1_000_000`、`repeat=5`；短操作要跑够次数，否则分辨率不够。
- 单次结果不可信：至少看 `repeat` 的 5 次，取**最小值**（受噪声影响最小）或中位数；不要取平均后宣称精确到小数点后三位。
- 计时块里不要包含 `print`、日志、文件写入、`sleep`。
- 避免把"只执行一次的初始化"算进循环：用 `-s`/`setup` 或 `Timer(..., setup=...)`。
- 小心死代码/常量折叠：`for _ in range(n): 1 + 1` 可能被优化得几乎无成本，测不到你想测的东西。

```python
import timeit

timer = timeit.Timer("sorted(data)", setup="data = list(range(1000, 0, -1))")
runs = timer.repeat(repeat=7, number=2000)
print("best:", min(runs) / 2000)
print("all :", [round(r / 2000 * 1e6, 1) for r in runs])   # 微秒
```

### 2.2.3 微基准的五个陷阱

1. **数据太小**：10 个元素时"列表推导 vs for 循环"的差异，和 100 万个元素时完全不同；缓存、扩容、分支预测都会改变结论。
2. **数据分布不真实**：用全唯一键测 dict，和用大量哈希冲突的键测 dict，结果不同。
3. **忽略内存分配**：微基准只测时间，不测 RSS/分配次数；真实程序可能死于内存而不是 CPU。
4. **忽略冷启动**：第一次调用包含 import、JIT/特化、缓存预热；要区分冷路径与热路径。
5. **把环境差异当语言差异**：CPU 型号、频率、Python 版本、编译选项、容器配额都会影响结果。

## 2.3 CPU 性能分析：从 `cProfile` 到采样剖析

### 2.3.1 确定性剖析：`cProfile`

`cProfile` 通过函数调用钩子记录每次调用，优点是**精确到函数调用次数与累计时间**，缺点是**有固定开销、会扭曲短函数占比**。

```python
import cProfile
import pstats

def workload():
    data = [i * i for i in range(200_000)]
    return sum(data)

profiler = cProfile.Profile()
profiler.enable()
workload()
profiler.disable()
profiler.dump_stats("profile.prof")

stats = pstats.Stats("profile.prof")
stats.strip_dirs().sort_stats("cumulative").print_stats(20)
```

命令行方式：

```bash
python -m cProfile -o profile.prof your_script.py
python -m pstats profile.prof
# 交互式输入：sort cumulative / stats 20 / callers foo
```

**读懂 `cProfile` 输出**：

| 列 | 含义 | 怎么用 |
| :--- | :--- | :--- |
| `ncalls` | 调用次数（可能显示 `总次数/原始次数`） | 找"调用次数异常多"的函数 |
| `tottime` | 函数自身耗时（不含子调用） | 找真正的热点 |
| `percall` | `tottime / ncalls` | 找单次很贵的函数 |
| `cumtime` | 函数及其子调用的累计耗时 | 找"整条链路"的耗时 |
| `percall`（第二列） | `cumtime / ncalls` | 判断是否重复调用 |
| `filename:lineno` | 位置 | 定位代码 |

**常见误读**：

- `cumtime` 高不代表该函数是瓶颈，可能只是它调用了慢函数；要看 `tottime`。
- 递归函数的 `cumtime` 会重复计算；要用 `tottime` 或专门的调用图工具。
- `cProfile` 对 C 扩展内部不可见；NumPy 的耗时可能只显示在 `method '...' of 'numpy.ndarray' objects` 一行。
- I/O 等待在 `cProfile` 里表现为函数耗时，但不消耗 CPU；要结合 `process_time` 与 `perf_counter` 区分。

### 2.3.2 采样剖析：`py-spy`、`pyinstrument`、`scalene`

采样剖析每隔几毫秒抓一次调用栈，开销低，适合**生产环境**和**长跑进程**。

```bash
# 实时看热点函数（类似 top）
py-spy top --pid 12345

# 生成火焰图
py-spy record -o profile.svg --pid 12345 --duration 30

# 直接 dump 当前所有线程栈（排查卡死/死锁）
py-spy dump --pid 12345
```

| 工具 | 类型 | 优点 | 局限 |
| :--- | :--- | :--- | :--- |
| `cProfile` | 确定性 | 调用次数/累计时间精确，标准库自带 | 开销大，短函数偏差，C 扩展内部不可见 |
| `profile` | 确定性 | 纯 Python 实现，可读源码 | 更慢，一般只用于研究 |
| `py-spy` | 采样 | 低开销、无需改代码、可看 C 栈/线程 | 统计近似，需权限（ptrace） |
| `pyinstrument` | 采样 | 输出可读的树、支持 async | 采样精度有限 |
| `scalene` | 采样 + 内存 | 同时看 CPU/内存/GPU，低开销 | 第三方依赖 |
| `line_profiler` | 行级 | 精确到行 | 开销大，需装饰器，不适合生产 |
| `sys.monitoring` | 事件 | 3.12+ 低开销插桩，可自定义 | 需要自己写工具，生态较新 |

### 2.3.3 行级剖析：`line_profiler`

```python
# pip install line_profiler
from line_profiler import LineProfiler

def hot_function(rows):
    total = 0
    for row in rows:
        total += row["price"] * row["qty"]
    return total

lp = LineProfiler()
lp.add_function(hot_function)
lp.enable()
hot_function([{"price": 1.5, "qty": 2}] * 100_000)
lp.disable()
lp.print_stats()
```

行级剖析的典型用途：确认"慢在循环体、属性访问、还是某个分支"，而不是只知道"这个函数慢"。

### 2.3.4 看懂"CPU 时间 vs 墙钟时间"

```python
import time

wall_start = time.perf_counter()
cpu_start = time.process_time()

time.sleep(1)          # 不消耗 CPU

wall = time.perf_counter() - wall_start
cpu = time.process_time() - cpu_start
print(f"wall={wall:.3f}s cpu={cpu:.3f}s")   # wall≈1s，cpu≈0
```

判断规则：

- `wall ≈ cpu`：CPU 密集，优化方向是算法/向量化/进程并行；
- `wall >> cpu`：I/O 等待或锁等待，优化方向是并发/缓存/减少往返；
- `cpu > wall`：多线程/多进程并行，说明确实用上了多核（注意容器 CPU 配额）。

## 2.4 内存测量：`tracemalloc`、RSS 与对象尺寸

### 2.4.1 `tracemalloc`：Python 层分配的去向

`tracemalloc` 跟踪 Python 内存分配器（`PyMem`）的分配，能回答"哪一行分配了多少内存、相比上次快照增长在哪里"。它**不跟踪** NumPy、OpenBLAS、C 扩展直接 `malloc` 的内存。

```python
import tracemalloc

tracemalloc.start(25)          # 保留 25 帧调用栈

snapshot1 = tracemalloc.take_snapshot()

cache = {}
for i in range(100_000):
    cache[i] = {"id": i, "payload": "x" * 100}

snapshot2 = tracemalloc.take_snapshot()

for stat in snapshot2.compare_to(snapshot1, "lineno")[:10]:
    print(stat)

current, peak = tracemalloc.get_traced_memory()
print(f"current={current / 1024 / 1024:.2f} MiB peak={peak / 1024 / 1024:.2f} MiB")
tracemalloc.stop()
```

`tracemalloc` 的典型用法：

- 启动后跑一段时间，比较两个快照，找增长最大的文件/行；
- 用 `filters=[tracemalloc.Filter(True, "*myapp*")]` 只看自己的代码；
- 用 `snapshot.statistics("traceback")` 看完整分配栈；
- 生产环境用 `-X tracemalloc=25` 或 `PYTHONTRACEMALLOC=25` 开启，但要注意开销（分配越频繁，开销越大）。

### 2.4.2 `sys.getsizeof` 是浅层的

```python
import sys

print(sys.getsizeof(0))            # 28（3.13 本机）
print(sys.getsizeof([]))           # 56
print(sys.getsizeof([0] * 1000))   # 只算列表本身的指针数组，不算 1000 个 int 对象
```

`sys.getsizeof(obj)` 返回对象**自身**的字节数，不包含它引用的对象。深层大小需要递归遍历：

```python
import sys
from collections.abc import Mapping, Sequence, Set

def deep_size(obj, seen=None):
    """近似计算对象的深层大小；只用于诊断，不保证覆盖 C 扩展内部内存。"""
    if seen is None:
        seen = set()
    obj_id = id(obj)
    if obj_id in seen:
        return 0
    seen.add(obj_id)

    size = sys.getsizeof(obj)
    if isinstance(obj, Mapping):
        size += sum(deep_size(k, seen) + deep_size(v, seen) for k, v in obj.items())
    elif isinstance(obj, (str, bytes, bytearray)):
        pass
    elif isinstance(obj, Sequence) or isinstance(obj, Set):
        size += sum(deep_size(item, seen) for item in obj)
    return size
```

更可靠的工具：`pympler.asizeof.asizeof()`、`memory_profiler`、`tracemalloc`；对 NumPy 数组用 `arr.nbytes`（不含 Python 对象头）。

### 2.4.3 进程级内存：RSS、VMS、峰值

```python
import os
import psutil

proc = psutil.Process(os.getpid())
mem = proc.memory_info()
print(f"RSS={mem.rss / 1024 / 1024:.1f} MiB")
print(f"VMS={mem.vms / 1024 / 1024:.1f} MiB")

# 当前进程及子进程的总 RSS
total = sum(p.memory_info().rss for p in [proc, *proc.children(recursive=True)])
print(f"tree RSS={total / 1024 / 1024:.1f} MiB")
```

| 指标 | 含义 | 注意 |
| :--- | :--- | :--- |
| RSS | 常驻物理内存 | 最接近容器内存限制的指标 |
| VMS | 虚拟地址空间 | 可能远大于 RSS（线程栈、mmap、保留区），不能直接当内存用量 |
| `ru_maxrss` | 进程峰值 RSS | Linux 单位 KB，macOS 单位字节；跨平台要换算 |
| USS/PSS | 独占/按共享比例分摊的内存 | 看多进程共享库、fork COW 更准确 |

> `resource.getrusage(resource.RUSAGE_SELF).ru_maxrss` 在 Linux 返回 KB，在 macOS 返回字节；Windows 上 `resource` 模块不可用。跨平台优先用 `psutil`。

### 2.4.4 内存泄漏的三种"增长"

| 现象 | 可能原因 | 验证方式 |
| :--- | :--- | :--- |
| RSS 持续上涨且不回落 | 真实泄漏（引用未释放、缓存无界、C 扩展） | `tracemalloc` 快照 diff + `gc.get_objects()` 计数 |
| RSS 阶梯式上涨后稳定 | 分配器/arena 未归还 OS、碎片、缓存预热 | 观察长时间曲线；用 `PYTHONMALLOC=malloc` 对照 |
| RSS 周期波动 | GC、批处理、日志缓冲 | `gc.callbacks` 时间线 + 业务批次对齐 |

## 2.5 GC 与停顿测量

Python 的循环垃圾回收（GC）是"周期性停顿"的常见来源。测量方法：

```python
import gc
import time

gc_stats = []

def on_gc(phase, info):
    if phase == "start":
        gc_stats.append([time.perf_counter(), info["generation"], None, None])
    else:
        start, generation, _, _ = gc_stats[-1]
        gc_stats[-1][2] = time.perf_counter() - start
        gc_stats[-1][3] = info["collected"]

gc.callbacks.append(on_gc)
```

生产上更常用的是：

- `gc.get_stats()`：各代收集次数、收集对象数；
- `gc.get_count()`：当前计数，判断何时触发；
- `gc.callbacks`：记录每次 GC 的耗时与代数（注意回调本身有开销，采样开启）；
- `PYTHONPROFILEIMPORTTIME` 不是 GC；不要混淆。

## 2.6 I/O 与系统级观测

| 目标 | 工具/接口 | 说明 |
| :--- | :--- | :--- |
| 进程读写字节数 | `psutil.Process().io_counters()` | 跨平台，粗粒度 |
| 文件描述符数量 | `psutil.Process().num_fds()`（Unix）/ `num_handles()`（Windows） | 泄漏排查 |
| 线程数 | `threading.active_count()`、`psutil.Process().num_threads()` | 线程暴涨 |
| 系统调用 | `strace -c -p PID`（Linux）、ETW/WPA（Windows） | 看 `read/write/fsync` 是否过多 |
| 磁盘 I/O | `iostat`、`iotop`、`psutil.disk_io_counters()` | 判断是否 I/O 饱和 |
| 网络 | `ss`、`netstat`、`tcpdump`、`psutil.net_io_counters()` | 连接数、重传、RTT |
| Python 调用栈 | `faulthandler.dump_traceback()`、`py-spy dump` | 卡死/死锁现场 |

`faulthandler` 是排查"程序卡住"的低成本利器：

```python
import faulthandler

# 30 秒后把所有线程栈 dump 到 stderr（只 dump 一次）
faulthandler.dump_traceback_later(30, repeat=False)

# 也可以注册信号（Unix）
# faulthandler.register(signal.SIGUSR1)
```

## 2.7 基准测试方法论

### 2.7.1 一个可复现的基准长什么样

1. **明确问题**：比较的是什么？"JSON 解析吞吐"还是"端到端请求延迟"？
2. **固定环境**：Python 版本、依赖版本、CPU 型号/频率、容器配额、后台负载。
3. **固定数据**：真实规模、真实分布、真实字段；不要用 10 条假数据。
4. **预热**：让 import、缓存、JIT/特化、连接池进入稳定状态。
5. **多次重复**：至少 5–10 次，报告 min/median/p95，而不是单次。
6. **控制变量**：一次只改一个因素；A/B 交替运行，减少漂移。
7. **记录结果**：把环境、命令、commit、数据规模写进结果文件。
8. **回归保护**：把基准放进 CI 或定期任务，设置合理的退化阈值。

### 2.7.2 延迟基准的"协调遗漏"（coordinated omission）

如果压测工具在请求变慢时**减少发压**（等上一个响应再发下一个），测出来的 p99 会系统性偏低。正确做法是**开环压测**：按固定速率发请求，记录实际完成时间；或用支持 coordinated omission 校正的工具。

### 2.7.3 工具选择

| 需求 | 工具 |
| :--- | :--- |
| 微基准 | `timeit`、`pyperf`（统计更严谨，支持 JSON 输出） |
| 函数级剖析 | `cProfile` + `pstats`、`line_profiler` |
| 生产采样 | `py-spy`、`pyinstrument`、`scalene`、持续剖析平台 |
| 内存 | `tracemalloc`、`memory_profiler`、`pympler`、`psutil` |
| 基准回归 | `pytest-benchmark`、`asv`、`pyperf` |
| 压测 | `locust`、`k6`、`wrk`、`JMeter` |
| 系统调用 | `strace`、`perf`、ETW/WPA |

## 2.8 本章要点与坑清单

**要点**

- 墙钟用 `perf_counter`，CPU 用 `process_time`，超时用 `monotonic`，日历时间才用 `time.time`。
- `timeit` 计时期间会关闭 GC；要测 GC 影响就单独测。
- `cProfile` 看 `tottime` 找自身热点，看 `cumtime` 找链路；采样工具适合生产。
- `tracemalloc` 只跟踪 Python 分配；NumPy/C 扩展内存要用 RSS/`nbytes`。
- `sys.getsizeof` 是浅层大小；深层大小要递归或专用工具。
- 基准测试要固定环境、预热、重复、看分布、防协调遗漏。

**坑**

- 用一次 `timeit` 结果下结论。
- 在 `cProfile` 里把 `cumtime` 最高的函数当成热点（它可能只是调用者）。
- 用 `tracemalloc` 解释 NumPy 的内存（它看不到）。
- 把 VMS 当内存用量，误判 OOM 风险。
- 压测时等响应再发下一个，p99 好看但线上崩溃。
- 忽略 GC 停顿对尾延迟的影响。

---

# 第 3 章 CPython 运行时：Python 到底慢在哪里

## 3.1 从源码到执行：一条完整的链路

```text
.py 源码
  │  compile()
  ▼
AST（抽象语法树）
  │  compiler
  ▼
字节码（code object）
  │  eval loop / 自适应特化
  ▼
CPython 虚拟机执行
  ├── 对象分配（obmalloc / malloc）
  ├── 名字查找（locals / globals / builtins）
  ├── 属性查找（type cache / descriptor）
  ├── 函数调用（frame 创建）
  └── C 扩展调用（释放 GIL 的时机）
```

用 `dis` 可以直接看到字节码，这是理解"为什么这行慢"的最快方式：

```python
import dis

def add(a, b):
    return a + b

def total(items):
    result = 0
    for item in items:
        result += item
    return result

dis.dis(add)
dis.dis(total)
```

你会看到 `LOAD_FAST`、`BINARY_OP`、`LOAD_GLOBAL`、`FOR_ITER`、`STORE_FAST` 等指令。**每一条字节码都是一次解释器分发**，这就是 Python 层循环比 C 层循环慢的根本原因。

### 3.1.1 代码对象里有什么

```python
def f(x):
    y = x * 2
    return y + 1

print(f.__code__.co_code)          # 原始字节码
print(f.__code__.co_consts)        # 常量池
print(f.__code__.co_varnames)      # 局部变量名
print(f.__code__.co_stacksize)     # 栈深度
print(f.__code__.co_nlocals)       # 局部变量个数
```

理解 `co_varnames`/`co_nlocals` 有助于解释：**局部变量访问是数组下标级别的 `LOAD_FAST`，而全局变量是字典查找 `LOAD_GLOBAL`**。这也是"把全局名字绑定到局部变量"能提速的原因（第 5 章）。

### 3.1.2 常量折叠与窥孔优化

CPython 在编译期会做少量优化，例如：

```python
def f():
    return 2 * 3 + 4       # 编译期折叠为 10
```

但**不会**把 `sum(range(1000))` 折叠成 500500；也不会把 Python 循环向量化。不要指望编译器替你做算法优化。

## 3.2 对象模型：一切皆对象，一切都有成本

### 3.2.1 每个 Python 对象都有固定开销

一个 Python 对象至少包含：

- 引用计数（`ob_refcnt`）；
- 类型指针（`ob_type`）；
- 对象自身的数据（如 int 的数值、list 的指针数组）。

因此：

- `1` 不是 4 字节，而是一个完整的 PyLong 对象（本机 3.13 上 `sys.getsizeof(0) == 28`）；
- `[1, 2, 3]` 的 56+ 字节里，大部分是指针数组和对象头，三个 int 对象另算；
- 一个包含 100 万个整数的 list，内存远大于 `array('q')` 或 NumPy 的 `int64` 数组。

```python
import sys

print(sys.getsizeof(0))            # 28
print(sys.getsizeof(10**100))      # 更大的 int 需要更多字节
print(sys.getsizeof([]))           # 56
print(sys.getsizeof([0] * 1000))   # 列表本身的指针数组，不含 int 对象
```

> 精确数字随版本/平台变化；这里要建立的是"对象有头、容器存指针、小整数有缓存"的直觉。

### 3.2.2 小整数缓存与字符串驻留

CPython 会缓存 `-5..256` 的小整数；部分字符串会被驻留（intern）。这带来两个后果：

1. 小整数的 `is` 比较经常"碰巧"为真；
2. 但**不要依赖 `is` 比较值**，语义上应该用 `==`。

```python
a = 256
b = 256
print(a is b)          # True（小整数缓存）

x = 257
y = 257
print(x is y)          # 在交互式/不同代码对象中可能是 False；不要依赖

s1 = "hello"
s2 = "hello"
print(s1 is s2)        # 可能 True（驻留/常量折叠），但不要依赖
print(s1 == s2)        # 永远用 == 比较内容

import sys
print(sys.intern("hello") is sys.intern("hello"))   # True
```

### 3.2.3 可变对象与不可变对象

| 类型 | 可变性 | 典型成本 |
| :--- | :--- | :--- |
| `int`、`float`、`str`、`bytes`、`tuple`、`frozenset` | 不可变 | 每次"修改"都可能创建新对象 |
| `list`、`dict`、`set`、`bytearray`、`deque` | 可变 | 原地修改，但扩容/哈希/搬移有成本 |

字符串拼接是典型例子：

```python
# 不推荐：循环里 += 可能反复分配（CPython 有优化，但不保证、不可移植）
parts = []
for i in range(100_000):
    parts.append(str(i))
result = "".join(parts)          # 推荐：一次性分配，O(n)
```

## 3.3 名字查找与属性查找：最容易被忽视的固定成本

### 3.3.1 LEGB 与字节码

| 查找 | 字节码 | 成本 |
| :--- | :--- | :--- |
| 局部变量 | `LOAD_FAST` | 数组下标，最快 |
| 闭包变量 | `LOAD_DEREF` | 比局部慢一点 |
| 全局变量 | `LOAD_GLOBAL` | 字典查找 + 缓存（3.11+ 有内联缓存） |
| 内置名字 | `LOAD_GLOBAL` 回退到 builtins | 更慢 |
| 属性 | `LOAD_ATTR` | 类型字典查找 + 描述符协议 + 内联缓存 |
| 方法调用 | `LOAD_METHOD`/`LOAD_ATTR` + `CALL` | 绑定方法对象/调用协议 |

```python
import math

def f(items):
    # 每次循环都做一次全局查找 + 属性查找
    return [math.sqrt(x) for x in items]

def g(items):
    sqrt = math.sqrt          # 局部绑定，循环内只做 LOAD_FAST + 调用
    return [sqrt(x) for x in items]
```

### 3.3.2 属性查找为什么贵

`obj.attr` 大致经历：

1. 查找类型（`type(obj)`）；
2. 在类型 MRO 中找描述符/数据描述符；
3. 处理 `__get__`（property、函数、classmethod 等）；
4. 回退到实例 `__dict__`；
5. 触发 `__getattr__`（如果定义了）。

所以：

- `obj.x` 比局部变量慢；
- `property` 比普通属性慢（多一次函数调用）；
- `__getattr__`/`__getattribute__` 更慢；
- 在热循环里重复访问 `self.config.value`，不如先取到局部变量。

```python
class Config:
    def __init__(self):
        self.value = 42

cfg = Config()

def slow():
    total = 0
    for _ in range(1_000_000):
        total += cfg.value          # 每次属性查找
    return total

def fast():
    value = cfg.value               # 只查一次
    total = 0
    for _ in range(1_000_000):
        total += value
    return total
```

### 3.3.3 方法绑定

```python
text = "a,b,c"

# 每次循环都查找并绑定方法
for _ in range(1000):
    text.split(",")

# 绑定一次
split = text.split
for _ in range(1000):
    split(",")
```

绑定方法会创建临时对象（`method` 对象）；在百万级循环里，这个固定成本会累积。但**可读性优先**：只有在 profiler 证明它是热点时才这样做。

## 3.4 函数调用：frame 的代价

每次 Python 函数调用都会创建/复用 frame、压栈、传参、处理返回值。函数调用本身不贵，但**在热循环里对每个元素调用一次 Python 函数**就会很贵。

```python
def normalize(x):
    return x.strip().lower()

# 对 100 万个字符串逐个调用 Python 函数
result = [normalize(s) for s in strings]

# 如果逻辑简单，直接内联到推导式，减少调用层数
result = [s.strip().lower() for s in strings]
```

常见优化：

- 用内置函数/`operator` 模块替代手写小函数；
- 用 `map`/`filter` 配合 C 实现的函数（如 `str.strip`、`int`）；
- 批量处理，减少"每个元素一次调用"；
- 用 `functools.partial` 固定参数（仍有一次调用，但比 lambda 略轻）；
- 避免在热路径里用装饰器层层包裹（每层都是调用）。

> 注意：`functools.lru_cache` 的命中路径也是一次 Python 调用 + 字典查找；对极短函数，缓存可能比直接算更慢。

## 3.5 GIL：Python 并行的硬约束

### 3.5.1 GIL 是什么

CPython 的全局解释器锁（Global Interpreter Lock）保证**同一时刻只有一个线程执行 Python 字节码**。它简化了 CPython 的内存管理，但也意味着：

- CPU 密集的多线程**不能**真正并行利用多核；
- 线程在 I/O 阻塞、`time.sleep`、部分 C 扩展调用时会释放 GIL；
- 多线程适合 I/O 密集，不适合纯 Python 计算密集。

```python
import sys

print(sys.getswitchinterval())      # 默认 0.005 秒（5ms）
sys.setswitchinterval(0.001)        # 降低切换间隔：响应更及时，但切换开销更大
```

**不要**通过调小 `switchinterval` 来"提升性能"；它只影响线程切换频率，不解决 GIL 的串行本质。

### 3.5.2 GIL 下仍然需要锁

GIL 只保证字节码级别的原子性，不保证复合操作原子：

```python
import threading

counter = 0

def increment():
    global counter
    for _ in range(100_000):
        counter += 1        # 读-加-写三步，可能被线程切换打断

threads = [threading.Thread(target=increment) for _ in range(4)]
for t in threads:
    t.start()
for t in threads:
    t.join()

print(counter)              # 很可能小于 400000
```

正确做法：用 `threading.Lock` 保护共享状态，或改用 `queue.Queue`/进程/无共享设计。

### 3.5.3 3.13 的 free-threaded 构建

Python 3.13 提供**实验性**的 free-threaded 构建（通常叫 `python3.13t`），可关闭 GIL，让多线程真正并行。但要注意：

- 是实验特性，单线程性能通常有明显损失；
- C 扩展必须专门支持 free-threading，否则会重新启用 GIL 或不可用；
- 仍需要锁保护共享可变状态（没有 GIL 不等于自动线程安全）；
- 用 `sys._is_gil_enabled()` 检查当前是否启用 GIL。

```python
import sys

print(sys._is_gil_enabled())    # 3.13+；普通构建返回 True
```

## 3.6 内存分配器：小对象、arena 与碎片

CPython 使用分层分配器：

| 层级 | 管理对象 | 说明 |
| :--- | :--- | :--- |
| `pymalloc`（obmalloc） | 小块内存（通常 ≤ 512 字节） | 按 size class 管理，减少 malloc 调用 |
| 系统 `malloc` | 大块内存、扩展分配 | 由 libc 或替换分配器管理 |
| arena | 一组页 | pymalloc 向系统申请 arena，再切分给对象 |

工程含义：

- **小对象多**时，pymalloc 很快，但对象头开销占比高；
- **长期运行**且对象大小混合，可能产生碎片，RSS 不随对象释放而下降；
- `PYTHONMALLOC=malloc` 可让 Python 直接用系统 malloc（便于用 jemalloc/tcmalloc 替换，但通常更慢）；
- `sys._debugmallocstats()` 可打印分配器统计（CPython 实现细节，输出格式不稳定）；
- glibc 环境下 `MALLOC_ARENA_MAX` 可限制 arena 数量，缓解多线程内存膨胀（不是 Python 层参数）。

> 不要为了"省内存"盲目设置 `MALLOC_ARENA_MAX=1`；它会增加锁竞争。先测量，再调优。

## 3.7 解释器演进：3.11–3.13 带来了什么

| 版本 | 关键变化 | 对性能/资源的影响 |
| :--- | :--- | :--- |
| 3.11 | PEP 659 自适应解释器（专化字节码 + 内联缓存）；异常表（零成本异常）；`TaskGroup`/`asyncio.timeout` | 纯 Python 代码通常有明显提速；异常不抛出时 `try` 开销降低 |
| 3.12 | PEP 709 推导式内联；`sys.monitoring`（PEP 669）；`itertools.batched`；`type` 参数语法 | 推导式更快、帧更少；可写低开销监控工具 |
| 3.13 | 实验性 JIT（PEP 744，默认关闭）；free-threaded 构建（PEP 703）；`os.process_cpu_count()`；GC 年轻代阈值调整 | 普通构建单线程性能继续提升；JIT/free-threading 暂不适合无脑上生产 |
| 3.14+ | 持续演进（free-threading 逐步成熟、GC 策略调整等） | 升级前以 release notes 和实测为准 |

**结论**：升级 CPython 小版本/大版本，往往是不改代码就能拿到的性能收益；但必须用真实负载回归，尤其是依赖 C 扩展、GC 行为、线程调度的程序。

## 3.8 本章要点与坑清单

**要点**

- Python 慢在"字节码分发 + 对象装箱 + 动态查找 + 函数调用"的固定成本，不是所有操作都慢。
- 局部变量最快，全局/属性查找更贵；热循环里把不变的名字/属性提到循环外。
- GIL 让 CPU 密集多线程无法并行；I/O 密集才适合线程。
- 小对象有固定头开销；容器存指针，不存值。
- 3.11+ 的自适应解释器、3.12 的推导式内联、3.13 的 JIT/free-threading 都会改变性能曲线，升级要实测。

**坑**

- 用 `is` 比较值（小整数/字符串驻留导致的"假阳性"）。
- 在热循环里反复 `self.attr`、`module.func`、`re.compile`。
- 认为"多线程一定快"，在 CPU 密集任务上开几十个线程。
- 认为 free-threaded 构建"自动线程安全"。
- 用 `PYTHONMALLOC=malloc` 解决所有内存问题。

---

# 第 4 章 算法与数据结构：决定数量级的第一杠杆

## 4.1 复杂度是下限，不是全部

Big-O 描述的是增长趋势，但真实性能还受常数因子、缓存局部性、分配次数影响。工程判断要同时看：

- **n 的规模**：n=10 时，O(n²) 可能比 O(n log n) 更快（常数小）；
- **操作类型**：同样是 O(n)，遍历 list 比遍历链表快得多（连续内存）；
- **内存代价**：哈希表 O(1) 但占内存；排序 O(n log n) 但需要额外空间；
- **最坏情况**：哈希冲突、退化输入、对手构造的键。

**先问 n 有多大，再问复杂度。** 对 100 个元素做 100 次线性查找，可能完全没问题；对 100 万条记录做嵌套循环，才是灾难。

## 4.2 复杂度速查表

| 操作 | list | tuple | dict | set | deque | heapq |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 按下标访问 | O(1) | O(1) | — | — | 两端 O(1)，中间 O(n) | — |
| `append`/`appendleft` | 尾部摊还 O(1) | — | — | — | O(1) | — |
| `insert(0, x)` | O(n) | — | — | — | O(n) | — |
| `pop()` 尾部 | O(1) | — | — | — | O(1) | — |
| `pop(0)` / `popleft()` | O(n) / — | — | — | — | O(1) | — |
| 按值删除 | O(n) | — | — | O(1) 平均 | O(n) | — |
| 成员判断 `x in c` | O(n) | O(n) | O(1) 平均 | O(1) 平均 | O(n) | — |
| 按 key 取值 | — | — | O(1) 平均 | — | — | — |
| 有序插入 | — | — | — | — | — | O(log n) |
| 取最小/最大 | — | — | — | — | — | O(1)/O(log n) |
| 排序 | O(n log n) | — | — | — | — | — |
| 切片 | O(k) | O(k) | — | — | — | — |
| 遍历 | O(n) | O(n) | O(n) | O(n) | O(n) | O(n) |

> dict/set 的 O(1) 是**平均**；极端哈希冲突下最坏 O(n)。`list.append` 是摊还 O(1)，因为扩容时会预留空间；`list.insert(0)` 需要搬移全部元素。

## 4.3 五个最常见的复杂度反模式

### 4.3.1 在 list 里做成员判断

```python
# ❌ O(n²)：每个元素都线性扫描
seen = []
for x in data:
    if x not in seen:
        seen.append(x)

# ✅ O(n)：set 成员判断平均 O(1)；需要保序时用 dict
seen = set()
result = []
for x in data:
    if x not in seen:
        seen.add(x)
        result.append(x)

# ✅ 更短：利用 dict 键唯一 + 插入有序（3.7+）
result = list(dict.fromkeys(data))
```

### 4.3.2 用 list 当队列

```python
# ❌ pop(0) 是 O(n)，整体 O(n²)
queue = [1, 2, 3]
while queue:
    item = queue.pop(0)

# ✅ deque.popleft() 是 O(1)
from collections import deque

queue = deque([1, 2, 3])
while queue:
    item = queue.popleft()
```

### 4.3.3 循环里做字符串拼接

```python
# ❌ 可能反复分配（小规模没事，大规模危险）
s = ""
for part in parts:
    s += part

# ✅ 一次分配
s = "".join(parts)
```

> CPython 对 `s += part` 有一个"引用计数为 1 时原地扩容"的优化，所以某些情况下并不是 O(n²)；但这是实现细节，遇到 `s` 被别处引用、跨解释器、PyPy 等就可能退化。**可预测的写法是 `join`。**

### 4.3.4 循环里反复排序/去重

```python
# ❌ 每轮都排序
for batch in batches:
    batch.sort()
    process(batch)

# ✅ 如果只需要全局有序，先合并/排序一次；如果每批独立，保留
```

### 4.3.5 嵌套循环里做重复计算

```python
# ❌ 内层重复计算与 j 无关的量
for i in range(n):
    for j in range(m):
        value = expensive(i) * j

# ✅ 提到外层
for i in range(n):
    base = expensive(i)
    for j in range(m):
        value = base * j
```

## 4.4 让 C 代码替你循环

Python 内置函数和标准库大量用 C 实现。**同样一个操作，能用 C 循环就不要用 Python 循环。**

| 需求 | 慢写法 | 快写法 |
| :--- | :--- | :--- |
| 求和 | `for x in data: total += x` | `sum(data)` |
| 最值 | 手写循环比较 | `min(data)` / `max(data)` |
| 排序 | 手写冒泡/选择 | `sorted(data)` / `list.sort()` |
| 计数 | 手写 dict 累加 | `collections.Counter(data)` |
| 拼接 | 循环 `+=` | `"".join(parts)` |
| 任意/全部 | 手写循环 + break | `any(...)` / `all(...)` |
| 映射/过滤 | 循环 + append | 推导式 / `map` / `filter` |
| 累计 | 手写累加器 | `itertools.accumulate` |
| 分组 | 手写 dict of list | `itertools.groupby`（先排序）或 `defaultdict(list)` |
| Top-K | 全排序后切片 | `heapq.nlargest` / `nsmallest` |
| 二分查找 | 手写 while | `bisect.bisect_left` |
| 堆操作 | 手写堆 | `heapq.heappush` / `heappop` |
| 笛卡尔积 | 嵌套循环 | `itertools.product` |
| 组合/排列 | 递归手写 | `itertools.combinations` / `permutations` |

示例：

```python
import heapq
import itertools
from collections import Counter

data = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]

print(sum(data))
print(Counter(data).most_common(3))
print(heapq.nlargest(3, data))
print(list(itertools.accumulate(data)))
```

## 4.5 容器选型：按操作剖面选，不按名字选

| 需求 | 首选 | 备选/注意 |
| :--- | :--- | :--- |
| 有序、下标访问、批量遍历 | `list` | 频繁头插 → `deque` |
| 固定记录、多返回值、dict 键 | `tuple` | 字段多且要可读 → `namedtuple`/`dataclass` |
| 键值映射、计数、缓存 | `dict` | 需要默认值 → `defaultdict`；计数 → `Counter` |
| 去重、成员判断、集合运算 | `set` | 需要可哈希集合 → `frozenset` |
| 两端频繁增删（队列/栈） | `deque` | 随机访问多 → `list` |
| 优先队列、Top-K、调度 | `heapq`（list 上操作） | 需要删除任意元素 → 更复杂结构 |
| 有序数组查找 | `bisect` + `list` | 频繁插入 → 树/跳表/数据库 |
| 大量同类型数值 | `array` / NumPy | Python 对象列表内存开销大 |
| 只读、需要哈希的映射 | `types.MappingProxyType` | 防止外部修改 |

**一个容易忽略的点**：`dict` 的键集合在语义上就是一个 `set`。所以：

```python
# 去重且保序
unique = list(dict.fromkeys(items))

# 两个 list 是否有交集
if set(a) & set(b):
    ...

# 统计词频
freq = Counter(words)
```

## 4.6 字符串与字节：不可变对象的成本

### 4.6.1 字符串拼接与格式化

```python
name = "world"
n = 42

# 常用写法，通常 f-string 最快、可读性最好
msg = f"hello {name}, n={n}"

# 需要延迟格式化（如日志）时，传参数而不是先拼好字符串
# logger.debug("hello %s, n=%d", name, n)
```

### 4.6.2 编码/解码是隐藏的 CPU 与内存成本

- `str` 与 `bytes` 之间转换会分配新对象；
- 大文件逐行 `decode` 可能比一次性 `read().decode()` 更省内存，但更慢；
- 网络/磁盘边界尽量保持 `bytes`，只在业务层解码；
- `memoryview` 可以避免切片拷贝（见第 9 章）。

```python
raw = b"hello world"
text = raw.decode("utf-8")
back = text.encode("utf-8")
print(text, back)

# 避免不必要的中间 bytes
mv = memoryview(b"0123456789")
print(bytes(mv[2:5]))       # 只复制需要的 3 个字节
```

### 4.6.3 正则表达式：编译一次，避免灾难性回溯

```python
import re

# ❌ 循环里重复编译（re 模块有缓存，但显式编译更清晰、可控）
for line in lines:
    if re.match(r"^\d+$", line):
        ...

# ✅ 编译一次
PATTERN = re.compile(r"^\d+$")
for line in lines:
    if PATTERN.match(line):
        ...
```

正则的性能风险：

- 复杂嵌套量词（如 `(a+)+$`）可能导致灾难性回溯，输入越长越慢；
- 用 `re.fullmatch` 而不是 `^...$` 可以减少边界遗漏；
- 能用字符串方法（`startswith`、`split`、`in`）就不要上正则；
- 对不可信输入设置长度上限/超时（标准库 `re` 不支持超时，可考虑 `regex` 模块或隔离进程）。

## 4.7 迭代器与生成器：用时间换空间

生成器不会一次性构建列表，适合**大数据流式处理**：

```python
def read_rows(path):
    with open(path, encoding="utf-8") as f:
        for line in f:
            yield line.rstrip("\n").split(",")

# 逐行处理，内存占用与文件大小无关
for row in read_rows("big.csv"):
    process(row)
```

对比：

| 写法 | 内存 | 速度 | 适用 |
| :--- | :--- | :--- | :--- |
| `f.readlines()` | O(n) | 快，但可能爆内存 | 小文件 |
| 直接 `for line in f` | O(1) 行缓冲 | 稍慢但稳定 | 大文件 |
| 生成器管道 | O(1) | 有 Python 层开销 | 流式 ETL |
| `list(gen)` | O(n) | 一次性物化 | 需要随机访问/多次遍历 |

**注意**：生成器只能消费一次；需要多次遍历时物化成 list/tuple，或重新创建生成器。

## 4.8 数据布局：缓存局部性比你想的重要

同样是"100 万个数值"，不同布局的性能可能差一个数量级：

```python
import array
import sys

n = 1_000_000

py_list = list(range(n))
py_array = array.array("q", range(n))

print(sys.getsizeof(py_list))      # 指针数组本身
print(sys.getsizeof(py_array))     # 连续 8 字节整数
```

- Python list 存的是指向 PyLong 对象的指针，遍历时还要解引用；
- `array('q')` 连续存储原始整数，内存小、缓存友好；
- NumPy 数组更进一步：连续内存 + 向量化 + 底层 BLAS；
- 但对小数据、复杂对象，Python 容器更灵活，别为了"底层"过早换。

## 4.9 三个可复用的优化例子

### 4.9.1 去重且保序

```python
def dedupe(items):
    seen = set()
    result = []
    for item in items:
        if item not in seen:
            seen.add(item)
            result.append(item)
    return result

# 更短，但要求元素可哈希
def dedupe_short(items):
    return list(dict.fromkeys(items))
```

### 4.9.2 Top-K

```python
import heapq

def top_k(items, k):
    # O(n log k)，比全排序 O(n log n) 更适合大 n 小 k
    return heapq.nlargest(k, items, key=lambda x: x["score"])
```

### 4.9.3 按 key 分组

```python
from collections import defaultdict
from itertools import groupby

rows = [
    {"city": "bj", "n": 1},
    {"city": "sh", "n": 2},
    {"city": "bj", "n": 3},
]

# 方式 1：defaultdict，O(n)，不要求有序
groups = defaultdict(list)
for row in rows:
    groups[row["city"]].append(row)

# 方式 2：groupby，O(n log n) 排序 + O(n)，适合已排序数据
rows_sorted = sorted(rows, key=lambda r: r["city"])
for city, group in groupby(rows_sorted, key=lambda r: r["city"]):
    print(city, list(group))
```

## 4.10 本章要点与坑清单

**要点**

- 复杂度决定数量级；先看 n 的大小和数据分布，再谈优化。
- list 成员判断 O(n)、`pop(0)` O(n)、循环字符串拼接可能退化，是最常见的三个坑。
- 能用 C 实现的内置函数/标准库，就不要用 Python 循环重写。
- 按操作剖面选容器：查多用 dict/set，两端增删用 deque，Top-K 用 heapq，有序查找用 bisect。
- 生成器省内存，但只能消费一次，且有 Python 层开销。

**坑**

- 在循环里 `if x in list`。
- 用 list 当 FIFO 队列。
- 用 `+=` 拼大字符串。
- 用 `sorted()[:k]` 取 Top-K（n 很大时）。
- 把生成器传给需要多次遍历的函数。
- 以为 `dict` 永远 O(1)（哈希冲突最坏 O(n)）。
- 对小数据过度追求"底层结构"，牺牲可读性却没有收益。

---

# 第 5 章 代码级优化：把常数因子压下去

> 本章的技巧大多只影响常数因子（几倍以内），**必须建立在 profiler 已确认热点的基础上**。在没有测量的情况下堆砌这些技巧，通常只会让代码更难读。

## 5.1 局部变量绑定：把查找成本移出循环

### 5.1.1 全局/属性 → 局部

```python
import math

def hypotenuse(points):
    sqrt = math.sqrt                 # 全局 + 属性查找只做一次
    result = []
    append = result.append           # 方法绑定只做一次
    for x, y in points:
        append(sqrt(x * x + y * y))
    return result
```

适用场景：百万级循环、每轮多次使用同一函数/方法。**不要在只跑几次的代码里这样做。**

### 5.1.2 默认参数绑定

```python
def process(items, *, _len=len):
    # 默认参数在函数定义时求值，把全局 len 绑定为局部默认值
    return _len(items)
```

这是 CPython 社区常见的微优化，但会损害签名可读性；只在库的热点函数里考虑。

## 5.2 属性与方法：减少重复查找

```python
class Order:
    def __init__(self, price, qty):
        self.price = price
        self.qty = qty

def total_slow(orders):
    result = 0
    for order in orders:
        result += order.price * order.qty     # 每轮 4 次属性查找
    return result

def total_fast(orders):
    result = 0
    for order in orders:
        price = order.price
        result += price * order.qty
    return result
```

`property` 的调用成本更高：

```python
class Config:
    def __init__(self):
        self._value = 1

    @property
    def value(self):
        return self._value

cfg = Config()

def read_many():
    total = 0
    for _ in range(1_000_000):
        total += cfg.value          # 每次触发 property getter 调用
    return total
```

如果 profiler 显示 property 是热点，可以在循环外取值，或改用普通属性 + 约定（但不要为了性能破坏封装）。

## 5.3 函数调用：减少层数，批量处理

### 5.3.1 内联小函数

```python
# 每行调用一次 Python 函数
def clean(line):
    return line.strip().lower()

lines = [clean(line) for line in lines]

# 直接使用内置方法，减少一层 Python 调用
lines = [line.strip().lower() for line in lines]
```

### 5.3.2 用 `operator` 替代 lambda

```python
from operator import itemgetter, attrgetter

rows = [{"name": "b", "age": 2}, {"name": "a", "age": 1}]

# lambda 每次调用都要执行 Python 函数体
sorted(rows, key=lambda r: r["age"])

# itemgetter 是 C 实现，通常更快
sorted(rows, key=itemgetter("age"))

# 多字段排序
sorted(rows, key=itemgetter("age", "name"))
```

### 5.3.3 批量代替逐条

```python
# ❌ 每条一次数据库往返
for row in rows:
    cur.execute("INSERT INTO t VALUES (?, ?)", row)

# ✅ 一次 executemany / 一个事务
cur.executemany("INSERT INTO t VALUES (?, ?)", rows)
conn.commit()
```

网络/数据库场景中，**减少往返次数**的收益往往远大于减少 Python 指令。

## 5.4 循环与推导式

### 5.4.1 列表推导式通常比 `for + append` 快

```python
# for + append
result = []
for x in range(1000):
    result.append(x * x)

# 列表推导式：字节码更紧凑，CPython 有专门优化
result = [x * x for x in range(1000)]
```

但不要为了"快"写出难以阅读的嵌套推导式；可读性也是性能（维护成本）。

### 5.4.2 生成器表达式省内存，但可能更慢

```python
total = sum(x * x for x in range(1_000_000))    # 省内存，逐项生成
total = sum([x * x for x in range(1_000_000)])  # 先建列表，内存高，通常更快
```

选择依据：

- 数据量大、内存紧张 → 生成器；
- 数据量适中、需要速度/多次遍历 → 列表；
- 不确定时先测量。

### 5.4.3 用 `enumerate`/`zip` 代替下标循环

```python
# 不推荐
for i in range(len(items)):
    print(i, items[i])

# 推荐
for i, item in enumerate(items):
    print(i, item)

# 并行遍历
for name, score in zip(names, scores):
    print(name, score)
```

`zip` 在 3.10+ 支持 `strict=True`，长度不一致时抛错，避免静默截断：

```python
for a, b in zip(xs, ys, strict=True):
    ...
```

## 5.5 异常：不要用异常做热路径控制流

Python 3.11+ 的异常表让**未抛出异常**的 `try` 几乎零成本；但**抛出/捕获异常**仍然昂贵（构造 traceback、栈展开）。

```python
# ❌ 用异常处理"常见情况"（缺失键很常见时）
try:
    value = d[key]
except KeyError:
    value = default

# ✅ 用 get / setdefault / defaultdict
value = d.get(key, default)
```

但 EAFP（先做再捕获）在"异常极少发生"时可能比 LBYL（先检查）更快，也更 Pythonic：

```python
# 文件通常存在时，EAFP 更直接
try:
    with open(path) as f:
        data = f.read()
except FileNotFoundError:
    data = ""
```

**判断标准**：异常是"正常路径的常见分支"还是"真正的异常"。前者用返回值/`get`，后者用异常。

## 5.6 字符串格式化与日志

### 5.6.1 格式化方式

| 写法 | 示例 | 说明 |
| :--- | :--- | :--- |
| f-string | `f"{name}: {n}"` | 通常最快、最可读；3.12+ 解析更优 |
| `str.format` | `"{}: {}".format(name, n)` | 灵活但稍慢 |
| `%` 格式化 | `"%s: %d" % (name, n)` | 老写法，日志延迟格式化常用 |
| 模板字符串 | `string.Template` | 安全替换场景 |

### 5.6.2 日志：先判断级别，再格式化

```python
import logging

logger = logging.getLogger(__name__)

# ❌ 即使 DEBUG 关闭，f-string 也会先求值
logger.debug(f"payload={expensive_repr(payload)}")

# ✅ 参数化，只有需要输出时才格式化
logger.debug("payload=%s", payload)

# 更稳妥：显式判断
if logger.isEnabledFor(logging.DEBUG):
    logger.debug("payload=%r", expensive_repr(payload))
```

高并发服务里，日志本身可能成为瓶颈：

- 同步写 stderr/文件会阻塞请求线程；
- 用 `QueueHandler` + `QueueListener` 异步化；
- 控制日志量、采样、避免在热循环里打日志；
- 结构化日志的 JSON 序列化也有成本。

## 5.7 对象模型：`__slots__`、dataclass 与命名元组

### 5.7.1 `__slots__` 省内存

普通实例有一个 `__dict__`（字典），属性多时开销大；`__slots__` 用固定槽位替代：

```python
import sys

class PointDict:
    def __init__(self, x, y):
        self.x = x
        self.y = y

class PointSlots:
    __slots__ = ("x", "y")

    def __init__(self, x, y):
        self.x = x
        self.y = y

p1 = PointDict(1, 2)
p2 = PointSlots(1, 2)
print(sys.getsizeof(p1), sys.getsizeof(p1.__dict__))   # 实例 + 字典
print(sys.getsizeof(p2))                                # 只有实例
```

注意：

- `__slots__` 的类不能随意新增属性；
- 不会自动支持弱引用，需要 `__weakref__` 槽；
- 继承非 slots 基类时，子类仍可能有 `__dict__`；
- 对少量对象收益不大，对百万级对象才明显。

### 5.7.2 `dataclass(slots=True)`

Python 3.10+ 可以直接生成 slots：

```python
from dataclasses import dataclass

@dataclass(slots=True)
class Point:
    x: int
    y: int
```

### 5.7.3 `namedtuple` vs `dataclass` vs 普通类

| 结构 | 内存 | 可变性 | 适用 |
| :--- | :--- | :--- | :--- |
| `tuple` | 最小 | 不可变 | 纯数据、位置语义 |
| `namedtuple` | 小 | 不可变 | 需要字段名、轻量记录 |
| `dataclass` | 中等 | 默认可变 | 需要默认值、方法、类型注解 |
| `dataclass(slots=True)` | 较小 | 可变 | 大量实例、字段固定 |
| 普通类 + `__slots__` | 小 | 可变 | 手写控制、性能敏感 |

## 5.8 拷贝：能共享就共享，必须复制再复制

```python
import copy

a = [[1, 2], [3, 4]]
b = a                 # 同一对象，不是拷贝
c = a.copy()          # 浅拷贝：外层新列表，内层共享
d = copy.deepcopy(a)  # 深拷贝：递归复制，最贵
```

- 热路径里避免 `deepcopy`；
- 不可变对象（tuple、frozenset、str）可以安全共享；
- 需要"写时复制"语义时，先共享、修改前再复制；
- `copy.copy` 对自定义类可被 `__copy__` 定制，`deepcopy` 可被 `__deepcopy__` 定制。

## 5.9 条件、容器与微优化清单

| 场景 | 较慢写法 | 较好写法 | 说明 |
| :--- | :--- | :--- | :--- |
| 成员判断 | `x in list` | `x in set`/`dict` | O(n) → O(1) |
| 计数 | 手写 `if k in d: d[k]+=1` | `Counter`/`d[k]=d.get(k,0)+1` | 更简洁，C 辅助 |
| 默认值 | `try/except KeyError` | `d.get`/`defaultdict` | 异常不常见时更快 |
| 多条件 | 嵌套 `if` | `all()`/`any()` + 生成器 | 短路径，但注意生成器开销 |
| 交换 | 临时变量 | `a, b = b, a` | 字节码更少 |
| 累加 | 手写循环 | `sum()` | C 循环 |
| 过滤 | 循环 + append | 推导式/`filter` | 看数据量 |
| 字符串前缀 | `s[:3] == "abc"` | `s.startswith("abc")` | 不创建切片 |
| 判空 | `len(x) == 0` | `not x` | 稍快且更 Pythonic |
| 遍历 dict | `for k in d: v=d[k]` | `for k, v in d.items()` | 避免二次查找 |

## 5.10 不要做的"伪优化"

- **用 `while` 替代 `for`**：通常不会更快，还容易写错边界。
- **手动展开循环**：CPython 没有 JIT 时收益有限，代码难维护。
- **把类改成 dict**：可能快一点，但丢失封装/方法/类型检查，长期成本高。
- **到处 `lru_cache`**：缓存命中率低时反而更慢，还可能泄漏内存。
- **用 `exec`/`eval` 生成代码**：编译成本、安全风险、可维护性差。
- **用 `__slots__` 优化 10 个对象**：收益可以忽略。
- **在非热点路径上做局部变量绑定**：可读性下降，收益为零。

## 5.11 本章要点与坑清单

**要点**

- 代码级优化只压常数因子，必须由 profiler 驱动。
- 热循环里减少全局查找、属性查找、方法绑定、函数调用层数。
- 推导式/内置函数通常更快；生成器省内存但可能更慢。
- 不要用异常做常见分支；异常路径很贵。
- 日志用参数化 + 级别判断，必要时异步化。
- `__slots__`/dataclass slots 对大量小对象有效，少量对象无感。

**坑**

- 把 `lru_cache` 加到有副作用/返回可变对象的函数上。
- 在日志里用 f-string 拼昂贵内容。
- 为了微优化破坏可读性，却没有 profiler 证据。
- 对少量对象使用 `__slots__` 并困惑"为什么没变快"。
- 用 `deepcopy` 复制大对象，或用浅拷贝误以为隔离了嵌套数据。

---

# 第 6 章 缓存与惰性计算：用空间换时间，但要有边界

## 6.1 缓存的层次

| 层次 | 例子 | 命中成本 | 失效难度 | 适用 |
| :--- | :--- | :--- | :--- | :--- |
| CPU 缓存 | 连续数组、少指针跳转 | 纳秒 | 自动 | 数据布局优化 |
| 函数级 memoization | `lru_cache`、手写 dict | 百纳秒 | 中 | 纯函数、重复参数 |
| 对象级缓存 | `cached_property` | 属性访问 | 低 | 昂贵且稳定的派生属性 |
| 进程内缓存 | 本地 dict、TTL 缓存 | 微秒 | 中 | 配置、字典表、热点数据 |
| 跨进程/分布式缓存 | Redis、Memcached | 毫秒/网络 | 高 | 共享会话、热点查询 |
| 磁盘缓存 | 文件、SQLite、`diskcache` | 毫秒 | 中 | 大对象、可重建结果 |
| HTTP 缓存 | `Cache-Control`、ETag、CDN | 毫秒/更少 | 中 | 静态资源、API 响应 |

**缓存不是免费的**：它用内存换时间，用一致性复杂度换吞吐。上线前必须回答：

1. 缓存多大？（条目数 × 平均大小）
2. 什么时候失效？（TTL、版本号、主动清除）
3. 未命中时会不会打爆下游？（缓存击穿）
4. 缓存的是可变对象吗？（返回共享引用可能被调用方修改）
5. 进程重启/多副本时一致性如何？

## 6.2 `functools.lru_cache` / `cache`

### 6.2.1 基本用法

```python
from functools import lru_cache

@lru_cache(maxsize=1024)
def expensive_lookup(user_id: int, region: str) -> dict:
    # 模拟昂贵计算/查询
    return {"id": user_id, "region": region}

print(expensive_lookup(1, "cn"))
print(expensive_lookup(1, "cn"))      # 命中缓存
print(expensive_lookup.cache_info())  # CacheInfo(hits=1, misses=1, maxsize=1024, currsize=1)
```

Python 3.9+ 可用 `@cache`（等价于 `lru_cache(maxsize=None)`）：

```python
from functools import cache

@cache
def fib(n: int) -> int:
    return n if n < 2 else fib(n - 1) + fib(n - 2)
```

### 6.2.2 关键语义

| 语义 | 说明 | 工程影响 |
| :--- | :--- | :--- |
| 参数必须可哈希 | 内部用 dict 做键 | list/dict/set 参数不能直接缓存 |
| 关键字顺序不同 = 不同条目 | `f(a=1, b=2)` 与 `f(b=2, a=1)` 可能各存一份 | 统一调用风格可提高命中率 |
| `typed=False` 时部分类型视为等价 | `1` 与 `1.0` 可能命中同一条 | 类型敏感时用 `typed=True` |
| 强引用参数与返回值 | 缓存不释放，直到淘汰/清空 | 缓存大对象会导致内存上涨 |
| 线程安全但可能重复计算 | 并发未命中时可能都执行一次 | 昂贵计算需额外 single-flight |
| 方法缓存会包含 `self` | 每个实例一条 | 长生命周期对象可能泄漏 |

### 6.2.3 常见错误

```python
# ❌ 参数不可哈希
@lru_cache
def load(ids):
    return sum(ids)

# load([1, 2, 3])  # TypeError: unhashable type: 'list'


# ❌ 缓存返回可变对象，调用方修改后污染缓存
@lru_cache
def get_config():
    return {"debug": False}

cfg = get_config()
cfg["debug"] = True          # 下一次 get_config() 返回被改过的字典


# ❌ 缓存有副作用/依赖外部状态
@lru_cache
def read_file(path):
    with open(path) as f:    # 文件变了仍返回旧内容
        return f.read()
```

### 6.2.4 缓存方法：小心 `self`

```python
class Service:
    def __init__(self, name):
        self.name = name

    @lru_cache(maxsize=128)
    def get(self, key):
        return f"{self.name}:{key}"
```

问题：`self` 是缓存键的一部分，每个实例都会占一份缓存；实例不被释放时，缓存也不会释放。更安全的做法：

- 在模块级函数上缓存（参数显式传入所需数据）；
- 用 `weakref` 或实例级缓存 + 显式清理；
- 用 `functools.cached_property` 做"每个实例只算一次"的属性。

## 6.3 `cached_property`：每实例一次的派生属性

```python
from functools import cached_property
import statistics

class DataSet:
    def __init__(self, values):
        self._values = tuple(values)

    @cached_property
    def mean(self):
        print("computing...")
        return statistics.fmean(self._values)

ds = DataSet([1, 2, 3])
print(ds.mean)      # computing... 2.0
print(ds.mean)      # 直接返回缓存，不再 computing
```

要点：

- 第一次访问时计算，结果写入实例 `__dict__`；
- 删除属性可让缓存失效：`del ds.mean`；
- 要求实例有可变的 `__dict__`；`__slots__` 类若无 `__dict__` 不适用；
- Python 3.12+ 移除了内部锁，多线程下 getter 可能被并发执行多次，需要幂等或自行加锁；
- 适合"昂贵、稳定、只读"的派生值（统计量、解析结果、连接信息）。

## 6.4 手写 TTL 缓存

标准库没有 TTL cache；简单场景可以手写：

```python
import time
from dataclasses import dataclass
from typing import Any, Callable, Hashable

@dataclass
class _Entry:
    value: Any
    expires_at: float

class TTLCache:
    def __init__(self, ttl: float, maxsize: int = 1024):
        self.ttl = ttl
        self.maxsize = maxsize
        self._data: dict[Hashable, _Entry] = {}

    def get(self, key, loader: Callable[[], Any]):
        now = time.monotonic()
        entry = self._data.get(key)
        if entry is not None and entry.expires_at > now:
            return entry.value

        value = loader()
        if len(self._data) >= self.maxsize:
            # 简单淘汰：删除最早过期的项；生产环境建议用 LRU/分段
            oldest = min(self._data, key=lambda k: self._data[k].expires_at)
            self._data.pop(oldest, None)
        self._data[key] = _Entry(value, now + self.ttl)
        return value
```

生产环境更推荐 `cachetools`（`TTLCache`、`LRUCache`）或专门的缓存服务；手写缓存要处理并发、淘汰策略、击穿、雪崩。

### 6.4.1 缓存击穿与 single-flight

热点 key 过期瞬间，大量请求同时未命中，全部打到数据库（击穿）。常见缓解：

- 互斥锁：只允许一个请求回源，其他等待/返回旧值；
- 逻辑过期：缓存不物理过期，异步刷新；
- 随机 TTL：避免同一时刻大面积过期（雪崩）；
- 预热：启动/定时任务提前加载热点；
- 降级：回源失败时返回旧值或默认值。

```python
import threading

class SingleFlight:
    def __init__(self):
        self._locks: dict[Hashable, threading.Lock] = {}
        self._guard = threading.Lock()

    def lock_for(self, key: Hashable) -> threading.Lock:
        with self._guard:
            return self._locks.setdefault(key, threading.Lock())
```

> 注意：进程内锁只能防单进程击穿；多副本部署需要分布式锁/逻辑过期。

## 6.5 弱引用缓存：让对象可以被回收

`lru_cache` 强引用键和值；如果缓存的是对象、且希望对象没被外部引用时自动消失，用 `weakref`：

```python
import weakref

class User:
    def __init__(self, uid):
        self.uid = uid

# 值弱引用：对象被回收后条目自动消失
cache = weakref.WeakValueDictionary()

def get_user(uid):
    user = cache.get(uid)
    if user is None:
        user = User(uid)
        cache[uid] = user
    return user

u = get_user(1)
print(cache.get(1) is u)     # True
del u
# 下一次 gc 后，cache 中可能已无 1
```

| 结构 | 键 | 值 | 适用 |
| :--- | :--- | :--- | :--- |
| `WeakValueDictionary` | 强 | 弱 | 对象缓存，值可被回收 |
| `WeakKeyDictionary` | 弱 | 强 | 给对象附加元数据 |
| `WeakSet` | 弱 | — | 跟踪一组对象，不阻止回收 |

注意：`int`、`str` 等不可弱引用；自定义类默认支持弱引用，`__slots__` 类需加 `__weakref__` 槽。

## 6.6 缓存失效策略

| 策略 | 做法 | 优点 | 缺点 |
| :--- | :--- | :--- | :--- |
| TTL | 固定时间过期 | 简单、最终一致 | 过期瞬间可能击穿；数据可能旧 |
| 版本号 | key 带版本/更新时间 | 可控、易批量失效 | 需要版本管理 |
| 主动删除 | 写操作后删缓存 | 实时性好 | 漏删/并发时序问题 |
| 写穿/写回 | 写时更新缓存 | 读命中高 | 实现复杂、一致性难 |
| 逻辑过期 | 值里带过期时间，异步刷新 | 避免击穿 | 可能返回旧值 |
| 不缓存 | 直接查 | 简单、强一致 | 性能差 |

**缓存一致性没有银弹**：先明确业务能接受多旧的数据（秒级/分钟级/最终一致），再选策略。

## 6.7 惰性计算：能不现在算就不现在算

### 6.7.1 惰性属性

```python
class Report:
    def __init__(self, rows):
        self._rows = rows
        self._summary = None

    @property
    def summary(self):
        if self._summary is None:
            self._summary = self._compute_summary()
        return self._summary

    def _compute_summary(self):
        return {"count": len(self._rows)}
```

`cached_property` 更简洁，但要求 `__dict__`；`property + 手写缓存` 更可控。

### 6.7.2 惰性导入

把重型依赖推迟到真正使用时：

```python
def render_pdf(html: str) -> bytes:
    import weasyprint          # 只有调用时才导入
    return weasyprint.HTML(string=html).write_pdf()
```

适用：CLI 子命令、冷启动敏感的服务、可选功能。注意：惰性导入会把 ImportError 推迟到运行时，需在启动检查/测试中覆盖。

### 6.7.3 惰性序列与流水线

```python
def read_numbers(path):
    with open(path, encoding="utf-8") as f:
        for line in f:
            yield int(line)

def square(nums):
    for n in nums:
        yield n * n

def take(nums, limit):
    for i, n in enumerate(nums):
        if i >= limit:
            return
        yield n

# 只读到满足 limit 的位置，不会加载整个文件
for value in take(square(read_numbers("numbers.txt")), 10):
    print(value)
```

生成器流水线的代价是每层一次 Python 调用；如果数据量不大，直接列表推导更简单、也可能更快。

## 6.8 缓存与惰性的决策树

```text
结果是否可重复计算且开销大？
├── 否 → 不缓存
└── 是 → 结果是否依赖外部可变状态？
    ├── 是 → 需要 TTL/版本号/主动失效；否则不缓存
    └── 否 → 纯函数/稳定派生值
        ├── 参数少且可哈希 → lru_cache/cache
        ├── 每实例一次 → cached_property
        ├── 对象可回收 → weakref 缓存
        ├── 需要 TTL/多进程共享 → 外部缓存（Redis 等）
        └── 结果很大且可重建 → 磁盘缓存
```

## 6.9 本章要点与坑清单

**要点**

- 缓存有边界：大小、TTL、失效策略、击穿保护，缺一不可。
- `lru_cache` 适合纯函数、参数可哈希、结果稳定；注意强引用和 `self`。
- `cached_property` 适合每实例一次的昂贵派生值；要求 `__dict__`。
- 弱引用缓存能让对象被回收，适合对象缓存。
- 惰性导入/惰性属性/生成器能降低启动和峰值内存，但会增加运行时分支/调用开销。

**坑**

- 无界缓存导致内存持续上涨。
- 缓存可变对象并直接返回共享引用。
- 缓存有副作用或依赖文件/数据库状态的函数。
- 缓存方法时把 `self` 也缓存进去。
- 只设 TTL 不做击穿保护，热点过期瞬间打爆下游。
- 惰性导入把 ImportError 推迟到线上第一次调用。

---

# 第 7 章 加速路线：NumPy、Cython、Numba、C/Rust 扩展与 PyPy

> 原则：**先穷尽算法、数据结构和内置函数，再考虑扩展。** 扩展会带来构建、部署、调试、跨平台和团队技能成本。

## 7.1 加速手段全景

| 手段 | 加速原理 | 适用 | 成本/限制 |
| :--- | :--- | :--- | :--- |
| 算法/数据结构 | 降低复杂度 | 所有场景 | 需要分析能力，收益最大 |
| 内置函数/标准库 | C 循环、专用实现 | 通用 | 受限于已有 API |
| 列表推导/生成器 | 更紧凑的字节码 | 通用 | 仍是 Python 层循环 |
| NumPy/pandas | 连续内存 + 向量化 + BLAS | 数值/表格计算 | 需要数组化思维，内存放大 |
| Numba | JIT 编译为机器码 | 数值循环、NumPy | 首次编译慢，仅支持部分 Python |
| Cython | 静态类型 + C 编译 | 通用热点、C 交互 | 构建链、类型调优 |
| mypyc | 用类型注解编译 | 已类型化的代码 | 语言子集限制 |
| C/C++ 扩展 | 原生性能 | 极致性能、库绑定 | 开发/构建/内存安全成本 |
| Rust（PyO3） | 原生性能 + 内存安全 | 新扩展、性能敏感 | 学习曲线、构建链 |
| PyPy | JIT 追踪热点 | 长跑纯 Python 程序 | C 扩展兼容性、预热、内存 |
| CPython 新版本 | 解释器优化 | 所有场景 | 需回归测试 |
| free-threaded 3.13t | 多线程并行 | CPU 密集多线程 | 实验性、单线程变慢、扩展兼容 |
| GPU（CuPy/Numba CUDA） | 大规模并行 | 大数组/矩阵 | 传输开销、显存、复杂度 |

## 7.2 NumPy：向量化不是"把循环换成 np 函数"这么简单

### 7.2.1 向量化示例

```python
import numpy as np

n = 1_000_000
xs = np.arange(n, dtype=np.float64)
ys = np.arange(n, dtype=np.float64)

# ❌ Python 层循环：每个元素一次解释器操作
result = [x * x + y for x, y in zip(xs, ys)]

# ✅ 向量化：C 层循环 + 连续内存
result = xs * xs + ys
```

### 7.2.2 关键原则

1. **保持数组，不要掉回 Python 循环**：`for x in arr` 会把 NumPy 标量装箱，通常比列表还慢。
2. **选对 dtype**：`float64` 比 `float32` 占双倍内存；整数溢出/精度要小心。
3. **避免不必要的拷贝**：切片是视图，`arr[::2]` 不复制；`arr.copy()`、`np.ascontiguousarray` 才复制。
4. **注意内存布局**：C 顺序 vs Fortran 顺序，转置后再运算可能触发拷贝。
5. **减少临时数组**：`a = b + c + d` 会创建多个中间数组；大数组可用 `np.add(b, c, out=tmp)` 复用缓冲区。
6. **用 `out=` 和原地操作**：`arr *= 2` 比 `arr = arr * 2` 省内存。
7. **批量而不是逐元素调用**：`np.where`、`np.select`、布尔掩码比 Python `if` 循环快。

```python
a = np.ones((1000, 1000))
b = np.ones((1000, 1000))

# 多个临时数组
c = a + b * 2

# 复用缓冲区，减少分配
tmp = np.empty_like(a)
np.multiply(b, 2, out=tmp)
np.add(a, tmp, out=tmp)
```

### 7.2.3 pandas 的常见性能点

- 用 `category` dtype 存低基数字符串列；
- 读取时指定 `dtype`、`usecols`、`parse_dates`，避免全量推断；
- 大文件分块 `chunksize`，或改用 Parquet/Arrow；
- 避免 `iterrows()`（逐行装箱，极慢），优先向量化或 `itertuples()`；
- 避免链式赋值和 `SettingWithCopyWarning`；
- 合并前先过滤/聚合，减少数据量；
- 需要极致性能时考虑 Polars/DuckDB/Arrow。

## 7.3 Numba：给数值循环加 JIT

```python
# pip install numba
from numba import njit
import numpy as np

@njit(cache=True)
def sum_squares(arr):
    total = 0.0
    for x in arr:
        total += x * x
    return total

arr = np.arange(1_000_000, dtype=np.float64)
print(sum_squares(arr))
```

要点：

- 首次调用触发编译（冷启动慢），`cache=True` 可缓存编译结果；
- 只支持部分 Python 语义，遇到不支持的类型会回退/报错；
- 对 NumPy 数组的循环效果最好；对字符串/字典等复杂对象支持有限；
- 编译后的函数不适合交互式频繁改代码的场景。

## 7.4 Cython：静态类型 + C 编译

```cython
# fib.pyx（Cython 语法，不是普通 Python）
def fib(int n):
    cdef int a = 0
    cdef int b = 1
    cdef int i
    for i in range(n):
        a, b = b, a + b
    return a
```

构建（`setup.py` 或 `pyproject.toml` + `cythonize`）后得到 C 扩展。优化要点：

- 给循环变量/数组加类型（`cdef int`、`cdef double[:]` memoryview）；
- 减少 Python 对象交互；
- 用 `prange` + `nogil` 做并行（需注意线程安全）；
- 用 `cython -a` 生成 HTML 注解，黄色越深表示 Python 交互越多。

## 7.5 C/C++ 扩展与 Rust

- **C 扩展**：性能上限最高，但引用计数、GIL、异常、内存管理都要手动处理，容易出 segfault/泄漏。
- **ctypes**：无需编译，调用 C 库方便；但每次调用有 FFI 开销，不适合超高频小调用。
- **cffi**：比 ctypes 更 Pythonic，支持 ABI/API 模式；PyPy 上表现好。
- **Rust + PyO3/maturin**：内存安全、工具链现代，适合新写的高性能扩展；需要 Rust 构建链。
- **C++ + pybind11**：适合已有 C++ 库的绑定。

选择建议：

| 需求 | 推荐 |
| :--- | :--- |
| 调用已有 C 库，低频 | `ctypes`/`cffi` |
| 新写高性能扩展，团队会 Rust | PyO3 |
| 已有 C++ 代码 | pybind11 |
| 数值计算 | 先 NumPy/Numba，不够再 C |
| 字符串/复杂逻辑 | Cython 或 Rust |

## 7.6 PyPy：JIT 的甜区与盲区

PyPy 通过 tracing JIT 加速**长时间运行的纯 Python 代码**，常见收益来自：

- 热循环、数值计算、解释器密集型任务；
- 内存分配模式稳定的程序。

但要注意：

- 预热慢：短进程/CLI 工具可能还没 JIT 完就结束了；
- C 扩展兼容性：依赖 CPython C API 的库可能不兼容或变慢；
- 内存占用通常更高；
- 性能特征与 CPython 不同，必须用真实负载测。

## 7.7 CPython 3.11–3.13 的"免费加速"

| 特性 | 版本 | 说明 |
| :--- | :--- | :--- |
| 自适应专化字节码 | 3.11+ | 热代码自动特化，减少动态查找 |
| 零成本异常 | 3.11+ | 未抛异常的 `try` 几乎无额外成本 |
| 推导式内联 | 3.12+ | 推导式不再创建独立帧，速度提升 |
| 实验性 JIT | 3.13+ | 默认关闭，需专门构建；当前收益有限 |
| free-threaded | 3.13+ | 实验性；多线程 CPU 密集场景的长期方向 |
| `sys.monitoring` | 3.12+ | 低开销监控/剖析基础设施 |

**升级建议**：先在测试环境用真实负载对比 3.11/3.12/3.13；关注 C 扩展轮子、GC 行为、线程调度、启动时间的变化。

## 7.8 加速路线决策树

```text
热点是 CPU 密集的数值/数组计算？
├── 是 → 能否向量化（NumPy/pandas）？
│   ├── 能 → 向量化；不够 → Numba/Cython/C
│   └── 不能 → 算法/数据结构是否可优化？
└── 否 → 热点是 Python 对象/字符串/业务逻辑？
    ├── 能否用内置函数/标准库 C 实现？→ 是则替换
    ├── 能否缓存/批处理/减少调用？→ 是则先做
    ├── 长跑纯 Python、无 C 依赖？→ 试 PyPy
    └── 仍不够 → Cython/Rust/C 扩展
```

## 7.9 本章要点与坑清单

**要点**

- 加速顺序：算法 → 数据结构 → 内置/向量化 → JIT/编译扩展 → 运行时。
- NumPy 的核心是"保持数组、连续内存、减少临时数组"，不是简单换 API。
- Numba 适合数值循环，注意首次编译；Cython 适合类型明确的 hotspot；Rust 适合新扩展。
- PyPy 适合长跑纯 Python，不适合短进程和重 C 扩展。
- 升级 CPython 可能带来免费收益，但必须回归。

**坑**

- 在 NumPy 数组上写 Python `for` 循环。
- 用 `iterrows()` 处理百万行 DataFrame。
- 为了一个小热点引入完整 C++ 构建链。
- 在短 CLI 工具上用 PyPy，JIT 还没热就退出。
- 认为 free-threaded 构建一定更快（单线程可能更慢）。

---

# 第 8 章 内存管理：引用计数、GC、对象开销与泄漏排查

## 8.1 先分清几种"内存"

| 术语 | 含义 | 常见误判 |
| :--- | :--- | :--- |
| RSS（Resident Set Size） | 进程实际占用的物理内存页 | 最接近容器内存限制；但包含共享库、页缓存映射 |
| VMS/VSZ | 虚拟地址空间大小 | 可能远大于 RSS（线程栈、mmap、保留区），不能直接当用量 |
| USS | 进程独占的物理内存 | 多进程/共享库场景更准确 |
| PSS | 按共享比例分摊后的物理内存 | 容器/多进程统计更公平 |
| 分配器持有量 | pymalloc/arena 已向 OS 申请的内存量 | 对象释放后不一定立刻归还 OS |
| 跟踪分配量 | `tracemalloc` 统计的 Python 分配 | 不含 NumPy/C 扩展的 `malloc` |

**结论**：排查内存问题，先看 RSS 曲线；定位 Python 对象用 `tracemalloc`；定位 C 扩展/NumPy 用 `memray`、`psutil`、`/proc` 或 allocator 统计。

## 8.2 引用计数：Python 的主要回收机制

CPython 用引用计数管理对象生命周期：每个对象记录被引用的次数，归零立即释放（除非有循环）。

```python
import sys

value = []
print(sys.getrefcount(value))   # 2：value 本身 + getrefcount 的临时参数

another = value
print(sys.getrefcount(value))   # 3

del another
print(sys.getrefcount(value))   # 2
```

注意：

- `sys.getrefcount()` 的参数本身会临时增加一次引用，所以看到的值通常比"实际引用数"多 1；
- 引用计数是**即时**的，但循环引用不会归零，需要 GC；
- 对象被释放时会调用 `__del__`（如果定义），但 `__del__` 不保证在解释器退出时执行（见第 11 章）。

## 8.3 循环垃圾回收（GC）

### 8.3.1 为什么需要 GC

引用计数无法回收循环引用：

```python
class Node:
    pass

a = Node()
b = Node()
a.peer = b
b.peer = a

del a, b
# 两个对象的引用计数都不为 0（互相引用），但已不可达
```

GC 通过分代收集找出"不可达的循环垃圾"。

### 8.3.2 分代与阈值

CPython 把对象分为三代（0 最年轻、2 最老）：

- 新对象进入第 0 代；
- 第 0 代收集后存活的对象晋升到第 1 代，再存活晋升到第 2 代；
- 第 0 代收集最频繁、最快；第 2 代收集最少、最慢。

```python
import gc

print(gc.get_threshold())     # 3.13 本机：(2000, 10, 10)；3.12 本机：(700, 10, 10)
print(gc.get_count())         # 当前各代计数
print(gc.get_stats())         # 各代收集次数/对象数
```

> **版本差异**：3.13 把第 0 代默认阈值从 700 提高到 2000（降低 GC 频率/开销），3.12 及更早通常是 700。不同小版本可能调整，**以运行时 `gc.get_threshold()` 为准**，不要硬编码假设。

### 8.3.3 手动收集与调试

```python
import gc

gc.collect()                  # 全量收集（默认 generation=2）
gc.collect(0)                 # 只收集第 0 代

gc.set_debug(gc.DEBUG_STATS)  # 打印每次 GC 统计（调试用，开销大）
gc.set_debug(0)               # 关闭

print(gc.garbage)             # 无法回收的对象（通常为空；DEBUG_SAVEALL 时会堆积）
```

**不要**在业务代码里频繁调用 `gc.collect()`：全量收集会扫描大量对象，造成停顿。只在明确时机使用，例如：

- 大批量任务结束后、fork 前、内存快照前；
- 已知创建了大量循环引用后；
- 测试/诊断时。

### 8.3.4 哪些对象会被 GC 跟踪

GC 只跟踪**可能包含引用的容器对象**（list、dict、set、自定义实例等）；`int`、`str`、`bytes` 等原子对象通常不被跟踪。

```python
import gc

print(gc.is_tracked(0))          # False
print(gc.is_tracked("a"))        # False
print(gc.is_tracked([]))         # True
print(gc.is_tracked({}))         # False（3.13 对纯原子键值 dict 有优化）
print(gc.is_tracked({"a": []}))  # True
```

这解释了一个现象：**创建大量不含循环的原子对象，不会给 GC 造成太大压力；创建大量容器/实例才会。**

## 8.4 对象开销：内存都花在哪了

### 8.4.1 容器存的是指针

```python
import sys

n = 1_000_000
numbers = list(range(n))

print(sys.getsizeof(numbers))          # 列表的指针数组（约 8MB）
print(sys.getsizeof(numbers[0]))       # 每个 int 对象（约 28B）
# 总内存远大于 8MB：指针数组 + 100 万个 PyLong 对象
```

对比：

```python
import array
import sys

arr = array.array("q", range(n))
print(sys.getsizeof(arr))              # 连续 8 字节整数，约 8MB，无逐对象开销
```

### 8.4.2 `__slots__` 的实际收益

```python
import sys

class WithDict:
    def __init__(self):
        for i in range(10):
            setattr(self, f"a{i}", i)

class WithSlots:
    __slots__ = tuple(f"a{i}" for i in range(10))

    def __init__(self):
        for i in range(10):
            setattr(self, f"a{i}", i)

x = WithDict()
y = WithSlots()
print(sys.getsizeof(x), sys.getsizeof(x.__dict__))   # 本机 3.13：48 296
print(sys.getsizeof(y))                              # 本机 3.13：112
```

粗略估算：普通实例 = 实例对象 + `__dict__`（以及 dict 的哈希表），slots 实例 = 实例对象 + 固定槽位。字段越多、实例越多，差距越明显。

注意：

- 实例可能通过 `__dict__` 共享键（PEP 412），所以普通实例的 dict 不一定很大；
- `__slots__` 类不能动态加属性；
- 需要弱引用时加 `__weakref__` 槽；
- 继承非 slots 基类时，子类可能仍有 `__dict__`。

### 8.4.3 字符串驻留

大量重复字符串会占用大量内存。`sys.intern()` 可以让相同内容的字符串共享一个对象：

```python
import sys

# 适合大量重复的标识符/枚举值/字段名
keys = [sys.intern(f"field_{i % 100}") for i in range(1_000_000)]
```

但：

- 驻留表本身占内存；
- 只对"大量重复的短字符串"划算；
- 不要对用户输入/动态生成的长字符串无脑驻留（可能造成驻留表膨胀/DoS）；
- 字符串常量、标识符通常已被自动驻留。

### 8.4.4 用数组/NumPy 替代对象列表

| 存储 | 内存 | 访问速度 | 适用 |
| :--- | :--- | :--- | :--- |
| `list[int]` | 指针 + 每个 int 对象 | 灵活，但缓存不友好 | 通用 |
| `array('q')` | 连续 8 字节 | 快，元素需同类型 | 大量数值 |
| `numpy.ndarray` | 连续 + dtype | 向量化最快 | 数值计算 |
| `list[dict]` | 每个 dict 开销大 | 灵活 | 小数据/异构 |
| `pandas.DataFrame` | 列式 + dtype | 表格分析 | 结构化数据 |

## 8.5 用 `tracemalloc` 定位增长

一个标准的排查流程：

```python
import tracemalloc

tracemalloc.start(25)

# 1. 建立基线（服务启动/请求处理前）
baseline = tracemalloc.take_snapshot()

# 2. 执行可疑操作（处理 N 个请求/导入一批数据）
handle_requests(1000)

# 3. 再拍快照，按行比较
current = tracemalloc.take_snapshot()
for stat in current.compare_to(baseline, "lineno")[:20]:
    print(stat)

# 4. 看总分配
print(tracemalloc.get_traced_memory())
tracemalloc.stop()
```

输出解读：

```text
/path/app/cache.py:42: size=12.3 MiB (+12.3 MiB), count=100000 (+100000)
```

表示第 42 行新增了 10 万个对象、约 12.3 MiB。再看该行代码，通常就能找到无界缓存/未释放列表。

### 8.5.1 过滤噪声

```python
import tracemalloc

snapshot = tracemalloc.take_snapshot()
snapshot = snapshot.filter_traces([
    tracemalloc.Filter(False, "<frozen importlib._bootstrap>"),
    tracemalloc.Filter(False, tracemalloc.__file__),
    tracemalloc.Filter(True, "*myapp*"),
])
for stat in snapshot.statistics("lineno")[:10]:
    print(stat)
```

### 8.5.2 `tracemalloc` 的盲区

- 不跟踪 NumPy/OpenBLAS/C 扩展直接分配的内存；
- 不跟踪 mmap 文件映射；
- 开启后有额外开销（分配越频繁越明显）；
- `tracemalloc.start()` 之前分配的对象不在统计中。

对 C 扩展/NumPy 内存，用 `memray`（可跟踪 C/C++ 分配）、`psutil` RSS、`/proc/<pid>/smaps`、对象自身的 `nbytes`（NumPy）。

## 8.6 六类常见内存泄漏

### 8.6.1 无界缓存/全局容器

```python
# ❌ 永远增长
_cache = {}

def get(key):
    if key not in _cache:
        _cache[key] = compute(key)
    return _cache[key]
```

修复：有界 LRU、TTL、弱引用、定期清理、按业务分区。

### 8.6.2 异常对象持有 traceback

异常对象的 `__traceback__` 会引用整条调用栈的 frame，frame 又引用局部变量。如果把异常存到全局/实例上，可能把大对象一起留住。

```python
# ❌ 把异常对象长期保存
class Worker:
    def run(self):
        try:
            self.process()
        except Exception as exc:
            self.last_error = exc        # 可能持有整个栈帧


# ✅ 只保存字符串/结构化信息，不持有 traceback
class WorkerFixed:
    def run(self):
        try:
            self.process()
        except Exception as exc:
            self.last_error = f"{type(exc).__name__}: {exc}"
```

必要时清理 traceback 中的帧引用：

```python
import traceback

def describe_exception(exc: BaseException) -> str:
    tb = exc.__traceback__
    traceback.clear_frames(tb)          # 释放帧中局部变量的引用
    return repr(exc)
```

### 8.6.3 闭包/回调/监听器持有对象

```python
# ❌ 全局注册后永不移除，回调闭包引用 self
event_bus.subscribe("tick", lambda: self.refresh())

# ✅ 使用弱引用/可注销的订阅
event_bus.subscribe("tick", weakref.WeakMethod(self.refresh))
```

常见持有者：全局事件总线、日志 handler、信号处理器、`atexit`、线程池回调、`functools.partial`、装饰器缓存。

### 8.6.4 循环引用 + `__del__`

PEP 442 之后，带 `__del__` 的循环对象也可以被 GC 回收，但：

- 回收时机不确定；
- `__del__` 执行顺序不确定；
- `__del__` 中抛异常会被忽略；
- 不要在 `__del__` 里做关键资源释放（见第 11 章）。

### 8.6.5 线程/进程/连接未回收

- 线程池未 `shutdown`；
- 每请求新建线程且不 join；
- 数据库连接/socket 未关闭；
- 子进程僵尸（未 `wait`）；
- 临时文件未删除。

这些通常先表现为 FD/线程数上涨，然后才是内存上涨。

### 8.6.6 C 扩展/NumPy 泄漏

- C 扩展忘了 `Py_DECREF`；
- NumPy 数组被 Python 对象长期引用；
- 大数组切片视图持有原数组（`view` 引用 base，不能释放）；
- `numpy` 运算产生大量临时数组，峰值内存翻倍。

## 8.7 内存碎片与分配器

### 8.7.1 为什么释放了对象 RSS 还不降

pymalloc 从 OS 申请 arena，对象释放后内存回到 arena 的空闲池，**不一定归还 OS**。如果长期运行且对象大小混合，可能出现碎片：

- RSS 高但 `tracemalloc` 显示 Python 分配不高；
- 重启进程后 RSS 下降；
- 随着运行时间缓慢上升。

缓解：

- 减少"大小差异极大"的对象混跑；
- 分批处理，及时释放大对象；
- 用多进程 worker + `max_requests`/定期重启隔离碎片；
- glibc 下评估 `MALLOC_ARENA_MAX`、jemalloc/tcmalloc；
- 不要盲目 `gc.collect()` 期待归还 OS（它只回收 Python 对象，不保证 `malloc_trim`）。

### 8.7.2 替换分配器

| 方式 | 作用 | 注意 |
| :--- | :--- | :--- |
| `PYTHONMALLOC=malloc` | Python 改用系统 malloc | 绕过 pymalloc，通常更慢；便于配合 jemalloc/tcmalloc |
| `LD_PRELOAD=libjemalloc.so` | 用 jemalloc 替换 malloc | 需构建/平台支持，调优参数多 |
| `MALLOC_ARENA_MAX` | 限制 glibc arena 数 | 降低多线程内存膨胀，可能增加锁竞争 |
| `MALLOC_TRIM_THRESHOLD_` | 控制归还阈值 | 需理解 glibc 参数，谨慎 |

> 分配器调优属于"最后 5%"：先用 `tracemalloc`/`memray` 确认没有真实泄漏，再考虑。

## 8.8 GC 调优：什么时候该动，怎么动

### 8.8.1 先判断是否需要

大多数程序**不需要**调 GC。只有以下情况才考虑：

- GC 停顿造成明显的 p99 毛刺；
- 大量循环垃圾导致内存上涨；
- fork 前希望减少 COW；
- 长生命周期对象很多，GC 扫描成本高。

### 8.8.2 常用手段

```python
import gc

# 1. 查看当前阈值与统计
print(gc.get_threshold())
print(gc.get_stats())

# 2. 提高第 0 代阈值：减少 GC 频率，可能增加内存峰值
gc.set_threshold(5000, 10, 10)

# 3. fork 前冻结长期存活对象，减少子进程 GC 扫描/COW
#    （Linux/Unix；父进程先 gc.disable()，fork 前 gc.freeze()，子进程 gc.enable()）
if hasattr(gc, "freeze"):
    gc.freeze()

# 4. 大批量任务后手动全量收集（谨慎）
gc.collect()
```

### 8.8.3 `gc.freeze()` 与 fork

如果使用 `fork` 创建子进程（3.13 及以前 Linux 的默认方式；3.14 起 POSIX 默认改为 `forkserver`），子进程会继承父进程的内存页（copy-on-write）。GC 在子进程中扫描/修改对象会触发 COW，导致内存翻倍。

推荐模式（来自 `gc.freeze` 文档）：

```python
import gc
import os

gc.disable()          # 父进程尽早关闭自动 GC
# ... 加载数据、建立长期对象 ...
gc.freeze()           # fork 前冻结，移入永久代
pid = os.fork()
if pid == 0:
    gc.enable()       # 子进程重新启用
    # 子进程逻辑
    os._exit(0)
else:
    gc.enable()
    os.waitpid(pid, 0)
```

> Windows 没有 `fork`，`multiprocessing` 默认 `spawn`，不存在同样的 COW 问题，但进程启动/导入成本更高。

### 8.8.4 不要做的 GC 操作

- **不要**在请求处理路径里 `gc.collect()`；
- **不要**因为"内存高"就 `gc.disable()`（循环垃圾会越积越多）；
- **不要**照抄网上的阈值（不同负载差异巨大）；
- **不要**用 `gc.set_debug(gc.DEBUG_SAVEALL)` 上生产（会阻止回收、疯狂吃内存）。

## 8.9 内存上限与 `MemoryError`

```python
try:
    huge = [0] * (10 ** 12)
except MemoryError:
    # 释放缓存、降级、返回 503
    ...
```

注意：

- `MemoryError` 可能在任何分配点抛出，包括异常处理/日志代码中；
- 捕获后不要做大量分配；
- 容器环境可能直接被 OOM Killer 杀死（SIGKILL），轮不到 Python 抛 `MemoryError`；
- 预留应急内存/降级路径（清缓存、拒绝新任务、落盘）比"捕获后继续跑"更可靠。

## 8.10 内存优化清单

| 手段 | 收益 | 代价/注意 |
| :--- | :--- | :--- |
| 流式/分块处理 | 峰值内存从 O(n) 降到 O(chunk) | 代码复杂、可能更慢 |
| 生成器 | 不物化中间列表 | 只能消费一次，Python 层开销 |
| `__slots__`/dataclass slots | 大量小对象省内存 | 牺牲动态属性/弱引用需额外槽 |
| `array`/NumPy | 连续存储、无逐对象开销 | 类型固定、需要向量化思维 |
| 有界缓存 | 防无界增长 | 需要淘汰策略 |
| 弱引用 | 对象可被回收 | 只适用于可弱引用对象 |
| 减少拷贝 | 省内存/CPU | 注意共享可变对象的别名问题 |
| `memoryview` | 切片零拷贝 | 生命周期/可写性需管理 |
| 及时 `del` 大对象 | 尽快释放 | 注意其他引用 |
| 压缩/列式存储 | 磁盘/内存下降 | CPU 开销、随机访问变慢 |
| 进程隔离/定期重启 | 隔离碎片/泄漏 | 架构复杂、冷启动 |

## 8.11 本章要点与坑清单

**要点**

- 引用计数即时回收，循环引用靠分代 GC。
- 3.13 第 0 代阈值默认 2000，3.12 为 700；以运行时为准。
- 容器存指针，小对象有固定开销；大量小对象优先考虑 slots/数组/流式。
- `tracemalloc` 看 Python 分配，RSS 看进程真实占用，二者不能互相替代。
- 泄漏最常见来源：无界缓存、异常 traceback、闭包/回调、线程/连接、C 扩展。
- GC 调优是最后手段；fork 场景可用 `gc.freeze()` 降低 COW。

**坑**

- 把 RSS 不降当成泄漏（可能只是分配器/碎片）。
- 用 `sys.getsizeof` 估算整个容器（浅层）。
- 把异常对象长期保存，导致整条栈帧无法释放。
- 无脑 `gc.disable()` 或频繁 `gc.collect()`。
- 用 `lru_cache` 缓存实例方法，导致 `self` 无法回收。
- 用 `tracemalloc` 解释 NumPy 大数组（看不到）。

---

# 第 9 章 I/O 性能：缓冲、批量、mmap、零拷贝、序列化与数据库

## 9.1 I/O 的成本结构

一次 `read()`/`write()` 的成本大致包括：

```text
Python 方法调用
  → 参数解析/对象创建
  → 系统调用（用户态/内核态切换）
  → 内核缓冲区/页缓存
  → 设备/网络（真正的 I/O）
  → 数据拷贝（内核 ↔ 用户空间）
```

优化 I/O 的核心就三件事：

1. **减少系统调用次数**（缓冲、批量）；
2. **减少数据拷贝次数**（零拷贝、`memoryview`、`sendfile`）；
3. **减少等待时间**（异步、并发、连接复用、就近访问）。

## 9.2 文件读写：缓冲与批量

### 9.2.1 `open()` 的缓冲语义

| 参数 | 含义 | 注意 |
| :--- | :--- | :--- |
| `buffering=-1` | 使用默认缓冲（通常 8192 字节，`io.DEFAULT_BUFFER_SIZE`） | 最常用 |
| `buffering=0` | 无缓冲，仅二进制模式 | 每次 `read/write` 都走系统调用，通常更慢 |
| `buffering=1` | 行缓冲，仅文本模式 | 适合交互式输出，不适合大文件 |
| `buffering=N` | 指定缓冲区大小 | 大块顺序 I/O 可调大（如 64KB–1MB） |
| `write_through=True` | 文本层不额外缓冲 | 仍需 `flush`/`fsync` 才落盘 |

```python
import io

print(io.DEFAULT_BUFFER_SIZE)      # 通常 8192

# 大文件顺序写：适度增大缓冲，减少系统调用
with open("big.bin", "wb", buffering=1024 * 1024) as f:
    for chunk in chunks:
        f.write(chunk)
```

### 9.2.2 读文件的三种方式

```python
# 1. 小文件：一次读完
with open("small.txt", encoding="utf-8") as f:
    text = f.read()

# 2. 大文本：逐行迭代（内存 O(1) 行）
with open("big.txt", encoding="utf-8") as f:
    for line in f:
        process(line)

# 3. 大二进制：分块读
with open("big.bin", "rb") as f:
    while chunk := f.read(1024 * 1024):
        process(chunk)
```

**不要**对大文件用 `f.readlines()` 或 `list(f)`：会把整个文件加载进内存。

### 9.2.3 `readinto`：复用缓冲区

```python
buf = bytearray(1024 * 1024)

with open("big.bin", "rb", buffering=0) as f:
    while n := f.readinto(buf):
        process(memoryview(buf)[:n])    # 避免切片拷贝
```

`readinto` 把数据读入已有缓冲区，适合固定大小分块、需要复用内存的场景。

### 9.2.4 写文件的批量策略

```python
# ❌ 每行一次 write + 频繁 flush
with open("out.txt", "w", encoding="utf-8") as f:
    for row in rows:
        f.write(format_row(row) + "\n")
        f.flush()          # 每次 flush 都可能触发系统调用

# ✅ 让缓冲工作；需要时批量拼接
with open("out.txt", "w", encoding="utf-8") as f:
    f.writelines(format_row(row) + "\n" for row in rows)
```

需要持久化保证时，才 `flush()` + `os.fsync()`：

```python
import os

with open("important.txt", "w", encoding="utf-8") as f:
    f.write("data")
    f.flush()                       # Python 缓冲 → OS
    os.fsync(f.fileno())            # OS → 磁盘（代价高）
```

**`flush` 不等于落盘**：`flush` 只把用户态缓冲交给内核；`fsync` 才要求内核把数据写到持久介质。每行 `fsync` 会让吞吐下降几个数量级。

### 9.2.5 原子写文件

直接覆盖写有风险：写到一半崩溃会留下半个文件。常用模式是"临时文件 + 原子替换"：

```python
import os
import tempfile

def atomic_write(path: str, data: bytes) -> None:
    directory = os.path.dirname(os.path.abspath(path)) or "."
    fd, tmp_path = tempfile.mkstemp(dir=directory)
    try:
        with os.fdopen(fd, "wb") as f:
            f.write(data)
            f.flush()
            os.fsync(f.fileno())
        os.replace(tmp_path, path)      # 同文件系统内原子替换
    except BaseException:
        try:
            os.unlink(tmp_path)
        except FileNotFoundError:
            pass
        raise
```

注意：跨文件系统 `os.replace` 可能失败；需要目录 `fsync` 才能保证 rename 持久化（详见数据库/文件系统文档）。

## 9.3 目录遍历：`os.scandir` 通常比 `listdir + stat` 快

```python
import os

# ❌ 每个文件一次 stat 系统调用
for name in os.listdir("data"):
    path = os.path.join("data", name)
    if os.path.isfile(path):
        size = os.path.getsize(path)

# ✅ DirEntry 缓存了类型/stat 信息，通常只需一次 scandir
with os.scandir("data") as entries:
    for entry in entries:
        if entry.is_file():
            size = entry.stat().st_size
```

| API | 特点 | 适用 |
| :--- | :--- | :--- |
| `os.listdir` | 只返回名字 | 只需要文件名 |
| `os.scandir` | 返回 `DirEntry`，缓存类型/stat | 需要判断类型/大小 |
| `os.walk` | 递归遍历 | 目录树处理 |
| `pathlib.Path.iterdir` | 面向对象，基于 scandir | 可读性优先 |
| `glob.glob` | 模式匹配 | 按扩展名/模式筛选 |

> `pathlib` 的便利性很好，但在超大规模目录遍历的热点里，`os.scandir` 的显式控制通常更直接。先测量再选。

## 9.4 `mmap`：把文件映射进地址空间

`mmap` 把文件映射到进程地址空间，读写像操作字节数组，由操作系统按页调度。适合：

- 大文件随机访问；
- 多进程共享只读数据；
- 避免一次性读入内存。

```python
import mmap

with open("big.bin", "rb") as f:
    with mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_READ) as mm:
        print(mm[:16])                 # 切片返回 bytes
        print(mm.find(b"needle"))      # 在映射中查找
        # mm[i] 返回 int（0-255）
```

访问模式：

| 模式 | 含义 | 用途 |
| :--- | :--- | :--- |
| `ACCESS_READ` | 只读 | 只读大文件、索引 |
| `ACCESS_WRITE` | 读写，修改写回文件 | 原地修改 |
| `ACCESS_COPY` | 读写，修改不写回（COW） | 临时修改、测试 |

注意：

- `mmap` 不是"一定更快"：小文件、顺序读、页缓存已命中时，普通 `read` 可能更简单更快；
- 映射长度 0 表示整个文件（Unix/Windows 语义略有差异）；
- 关闭映射前确保没有未完成的切片/视图引用；
- 写入后是否需要 `flush()` 取决于持久化要求；
- 文件被截断/修改时，映射访问可能触发 `SIGBUS`（Unix）。

## 9.5 零拷贝：少一次拷贝就少一份成本

### 9.5.1 `memoryview` 避免切片拷贝

```python
data = b"0123456789"

# 切片会复制
part = data[2:5]

# memoryview 切片是视图，不复制
mv = memoryview(data)
part_view = mv[2:5]
print(bytes(part_view))       # 只有真正需要 bytes 时才复制
```

`memoryview` 还支持：

- `cast()` 改变视图类型（如把 bytes 看成 int）；
- 与 `struct.unpack_from` 配合解析二进制协议；
- 传给支持 buffer protocol 的 C 扩展/NumPy，避免中间拷贝。

```python
import struct

buf = bytearray(8)
struct.pack_into("!I", buf, 0, 123)
value = struct.unpack_from("!I", buf, 0)[0]
print(value)
```

### 9.5.2 `sendfile`：文件 → socket 的内核态传输

```python
import socket

with open("large.bin", "rb") as f:
    with socket.create_connection(("example.com", 80)) as sock:
        # 跨平台：socket.sendfile 会在可用平台上走 sendfile/TransmitFile
        sock.sendfile(f)
```

Linux 还有 `os.sendfile(out_fd, in_fd, offset, count)`；标准库的 `shutil.copyfile` 在支持的平台上也会使用快速路径。**不要假设一定零拷贝**：不同平台/文件系统可能回退到用户态拷贝。

### 9.5.3 其他减少拷贝的手段

- 网络协议尽量用 `bytes`/`bytearray`，避免 `str` 与 `bytes` 来回转换；
- 用 `bytearray` 拼接二进制数据，而不是反复 `bytes + bytes`；
- 解析大 JSON 时考虑流式解析（`ijson`）或二进制协议（protobuf/flatbuffers）；
- 图片/视频处理尽量在 C/GPU 侧完成，避免 Python 层大块拷贝。

## 9.6 序列化：格式选择就是性能选择

| 格式 | 速度 | 体积 | 可读性 | 安全性 | 适用 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `json` | 中 | 中 | 好 | 安全（解析） | API、配置文件 |
| `pickle` | 较快 | 中 | 差 | **不安全**（反序列化可执行代码） | 内部可信数据、缓存 |
| `orjson`/`ujson` | 快 | 中 | 好 | 安全 | 高吞吐 JSON |
| `msgspec` | 很快 | 小 | 差 | 安全 | 高性能结构化数据 |
| protobuf | 快 | 小 | 差 | 安全 | 跨语言 RPC、存储 |
| flatbuffers | 很快（零解析） | 小 | 差 | 安全 | 游戏/低延迟 |
| Parquet/Arrow | 快（列式） | 小 | 差 | 安全 | 数据分析、大数据 |
| CSV | 慢 | 大 | 好 | 安全 | 交换、人工查看 |

### 9.6.1 JSON 的实用优化

```python
import json

data = {"a": 1, "b": [1, 2, 3]}

# 默认 separators 带空格，体积略大
s1 = json.dumps(data)
# 紧凑输出，减少字节数
s2 = json.dumps(data, separators=(",", ":"))
print(len(s1), len(s2))
```

其他：

- 避免对同一对象反复 `dumps`；
- 大响应考虑流式/分页；
- 需要极致性能用 `orjson`/`msgspec`（需评估依赖/兼容性）；
- 不要用 `json` 传二进制大字段（base64 会膨胀 33%）。

### 9.6.2 `pickle` 的性能与安全

- `pickle` 是 Python 专用，协议 5（3.8+）支持 out-of-band buffer，适合 NumPy 大数组；
- 反序列化不可信数据 = 任意代码执行，**永远不要**对用户输入/网络数据 `pickle.loads`；
- `pickle` 对类结构变化敏感，长期存储要版本化；
- 跨语言/长期存档优先 JSON/Parquet/protobuf。

## 9.7 压缩：CPU 与 I/O 的权衡

| 算法 | 压缩比 | 速度 | 适用 |
| :--- | :--- | :--- | :--- |
| gzip/zlib | 中 | 中 | 通用、HTTP |
| bz2 | 较高 | 慢 | 归档、冷数据 |
| lzma/xz | 高 | 很慢 | 极致压缩、离线 |
| zstd（第三方） | 高 | 快 | 现代通用 |
| lz4（第三方） | 低 | 很快 | 低延迟、日志 |

```python
import gzip
import zlib

raw = b"x" * 1_000_000

compressed = zlib.compress(raw, level=6)
print(len(raw), len(compressed))

with gzip.open("data.gz", "wb", compresslevel=6) as f:
    f.write(raw)
```

选择原则：

- 网络/磁盘是瓶颈、CPU 有余 → 压缩；
- CPU 是瓶颈、I/O 很快 → 不压缩或低级别；
- 压缩级别不是越高越好，6 左右通常是性价比点（因数据而异）；
- 大文件用流式压缩，避免一次性读入内存。

## 9.8 数据库：往返次数和事务比 SQL 微调更重要

### 9.8.1 批量写入

```python
import sqlite3

conn = sqlite3.connect("app.db")
cur = conn.cursor()

rows = [(i, f"name-{i}") for i in range(100_000)]

# ❌ 每条一次 execute + commit
for row in rows:
    cur.execute("INSERT INTO t VALUES (?, ?)", row)
    conn.commit()

# ✅ 一个事务 + executemany
with conn:
    cur.executemany("INSERT INTO t VALUES (?, ?)", rows)
```

### 9.8.2 查询与读取

- 只取需要的列（避免 `SELECT *`）；
- 用索引支撑 `WHERE`/`ORDER BY`，用 `EXPLAIN QUERY PLAN` 验证；
- 大结果集用 `fetchmany()`/服务端游标，避免 `fetchall()`；
- 避免 N+1 查询：用 JOIN/批量 `IN`/预加载；
- 连接池复用连接，避免每请求握手；
- 设置合理的 `timeout`/`busy_timeout`，避免锁等待无限挂起。

```python
cur.execute("SELECT id, name FROM users WHERE status = ? LIMIT ?", ("active", 100))
while rows := cur.fetchmany(50):
    process(rows)
```

### 9.8.3 SQLite 的常用性能设置

```python
import sqlite3

conn = sqlite3.connect("app.db")
conn.execute("PRAGMA journal_mode=WAL")        # 读写并发更好
conn.execute("PRAGMA synchronous=NORMAL")      # 性能/ durability 折中
conn.execute("PRAGMA foreign_keys=ON")         # 正确性
```

> `synchronous=NORMAL` 在 WAL 模式下通常足够；对"掉电不能丢数据"的场景保持 `FULL`。改 PRAGMA 前先理解 durability 语义。

## 9.9 网络 I/O：连接复用、超时与批量

| 手段 | 收益 | 注意 |
| :--- | :--- | :--- |
| 连接复用/连接池 | 省 TCP/TLS 握手 | 连接数上限、空闲回收 |
| keep-alive | 减少握手 | 服务端/中间件可能超时断连 |
| 批量/合并请求 | 减少 RTT | 增加单次延迟，需权衡 |
| 超时 | 防止无限等待 | 连接/读/写分别设置 |
| 压缩 | 减少传输字节 | CPU 开销 |
| 就近接入/CDN | 降低 RTT | 架构成本 |
| HTTP/2 多路复用 | 减少连接数 | 服务端支持 |

```python
import requests

# 复用 Session：连接池、默认头、keep-alive
with requests.Session() as session:
    session.headers.update({"User-Agent": "my-app/1.0"})
    for url in urls:
        resp = session.get(url, timeout=(3.05, 10))   # (连接, 读取)
        resp.raise_for_status()
```

**超时的三条经验**：

1. 永远不要用无限超时；
2. 连接超时通常短于读取超时；
3. 重试要有上限 + 退避 + 抖动，避免重试风暴。

## 9.10 异步 I/O：高并发等待场景

同步阻塞 I/O 在等待时占用线程；线程数受内存/调度限制。异步 I/O 用事件循环在单线程内管理大量并发连接，适合：

- 高并发网络客户端/服务端；
- 大量慢速 I/O（API 聚合、爬虫、消息消费）；
- 连接数远大于 CPU 核数的场景。

```python
import asyncio
import aiohttp

async def fetch(session, url):
    async with session.get(url, timeout=10) as resp:
        return await resp.text()

async def main(urls):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in urls]
        return await asyncio.gather(*tasks)

# asyncio.run(main(urls))
```

注意：

- 协程里**不能**有阻塞调用（`time.sleep`、同步 `requests`、大文件读），否则会卡住整个事件循环；用 `asyncio.to_thread()` 或异步库；
- 并发要有上限（`Semaphore`），否则会打爆下游/本地 FD；
- 超时、取消、异常处理要完整（`asyncio.timeout`、`TaskGroup`，3.11+）；
- CPU 密集任务不要放事件循环，用进程池/`to_thread`（仍受 GIL 限制）。

## 9.11 I/O 性能决策树

```text
瓶颈是 I/O 吗？（wall >> cpu，或 profiler 显示 read/write/网络等待）
├── 否 → 回到 CPU/内存章节
└── 是 → 是文件、数据库还是网络？
    ├── 文件
    │   ├── 小文件多 → 合并/打包/异步/批量
    │   ├── 大文件顺序读 → 分块/更大缓冲/流式
    │   ├── 大文件随机读 → mmap/索引/列式格式
    │   └── 写多 → 批量/缓冲/事务/顺序写
    ├── 数据库
    │   ├── 往返多 → 批量/连接池/合并查询
    │   ├── 查询慢 → 索引/EXPLAIN/减少数据量
    │   └── 写入慢 → 事务/批量/PRAGMA/分片
    └── 网络
        ├── 握手多 → 连接复用/TLS 会话复用
        ├── RTT 高 → 就近/合并/压缩/缓存
        └── 并发等待多 → asyncio/线程池/连接池
```

## 9.12 本章要点与坑清单

**要点**

- I/O 优化三板斧：减少系统调用、减少拷贝、减少等待。
- 缓冲不是越大越好；`flush` 不等于 `fsync`。
- 大文件用流式/分块；随机访问考虑 `mmap`。
- `memoryview`/`readinto`/`sendfile` 可以减少拷贝。
- 序列化格式要同时考虑速度、体积、安全和跨语言。
- 数据库优先减少往返和事务次数，再用索引/EXPLAIN 优化查询。
- 异步适合高并发 I/O，但协程里不能有阻塞调用，且必须限流。

**坑**

- 对大文件 `readlines()`。
- 每行 `flush()`/`fsync()`。
- 用 `pickle` 反序列化不可信数据。
- 循环里每条 SQL 一次 commit。
- 协程里调用同步 `requests`/`time.sleep`。
- 异步任务无上限，瞬间打开上万个连接。
- 不设超时，下游一慢就拖垮整个服务。

---

# 第 10 章 并发与并行：线程、进程、asyncio 的选型与背压

## 10.1 先判断：CPU 密集还是 I/O 密集

| 任务类型 | 特征 | 首选模型 | 不推荐 |
| :--- | :--- | :--- | :--- |
| CPU 密集（纯 Python） | `wall ≈ cpu`，CPU 打满 | 多进程、C 扩展、向量化 | 多线程（受 GIL 限制） |
| I/O 密集（少量并发） | `wall >> cpu`，等待多 | 线程池 | 多进程（重、通信贵） |
| 高并发 I/O（大量连接） | 连接数 >> CPU 核数 | asyncio/事件驱动 | 每连接一线程 |
| 混合型 | 既有计算又有等待 | 进程池 + 异步/线程池分层 | 一个模型硬套全部 |
| 需要共享大内存 | 数据量大、避免复制 | 线程 + 锁/共享内存 | 多进程（复制/序列化） |

**判断顺序**：

1. 用 `perf_counter` 与 `process_time` 比较，确认是 CPU 还是 I/O；
2. 用 profiler 看热点是否在 Python 字节码、C 扩展、I/O 等待；
3. 再看任务粒度：任务太小，并发开销会吃掉收益；
4. 最后才选线程/进程/协程。

## 10.2 多线程：适合 I/O，不适合纯 Python CPU 计算

### 10.2.1 基本用法

```python
import threading

def worker(name):
    print(f"start {name}")

threads = [threading.Thread(target=worker, args=(i,)) for i in range(4)]
for t in threads:
    t.start()
for t in threads:
    t.join()
```

### 10.2.2 共享状态必须加锁

```python
import threading

class Counter:
    def __init__(self):
        self._value = 0
        self._lock = threading.Lock()

    def increment(self):
        with self._lock:
            self._value += 1

    @property
    def value(self):
        with self._lock:
            return self._value
```

**GIL 不等于线程安全**：`+=`、`dict` 的"检查再写入"、`list` 的复合操作都可能被切换打断。规则：

- 共享可变状态 → 锁/原子结构/队列；
- 尽量无共享（消息传递、不可变数据）；
- 锁的临界区尽量小，不要在持锁时做 I/O；
- 多把锁要规定顺序，避免死锁；
- 用 `RLock` 只在确实需要可重入时。

### 10.2.3 线程池

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def fetch(url):
    # 模拟网络 I/O
    return url.upper()

urls = [f"https://example.com/{i}" for i in range(20)]

with ThreadPoolExecutor(max_workers=8) as executor:
    futures = {executor.submit(fetch, url): url for url in urls}
    for future in as_completed(futures):
        url = futures[future]
        try:
            result = future.result()
        except Exception as exc:
            print(url, "failed:", exc)
        else:
            print(url, result)
```

优点：

- 复用线程，避免每任务创建/销毁；
- 统一超时、异常、结果收集；
- 上下文管理器自动 `shutdown(wait=True)`。

注意：

- 线程池默认 `max_workers` 在 3.13 为 `min(32, (os.process_cpu_count() or 1) + 4)`；3.8–3.12 为 `min(32, os.cpu_count() + 4)`。I/O 密集可以显式调大，但要结合下游承载能力。
- `Executor.map` 会**立即收集所有输入**，不是惰性的；超大数据流要分批提交。
- 线程无法被强制杀死；取消只对"尚未开始"的任务有效。
- 线程池线程会在解释器退出前 join；不要在池里跑永不结束的任务。

## 10.3 多进程：绕开 GIL，但付出通信与内存成本

### 10.3.1 基本用法

```python
import multiprocessing as mp

def square(x):
    return x * x

def main():
    with mp.Pool(processes=4) as pool:
        results = pool.map(square, range(10))
    print(results)

if __name__ == "__main__":
    main()
```

`if __name__ == "__main__":` 在 Windows/macOS（spawn）上是**必须的**：子进程会重新导入主模块，缺少保护会递归创建进程。

### 10.3.2 启动方式

| 方式 | 平台/行为 | 优点 | 缺点 |
| :--- | :--- | :--- | :--- |
| `fork` | Unix（传统默认；3.14 起 POSIX 默认改为 `forkserver`） | 启动快，COW 共享内存 | 多线程 + fork 可能死锁；COW 内存上涨 |
| `spawn` | Windows/macOS 默认 | 干净、跨平台 | 启动慢，需要可导入的主模块，数据要 pickle |
| `forkserver` | 部分 Unix/3.14+ | 比 spawn 快，比 fork 安全 | 需要 server 进程，平台限制 |

显式选择：

```python
import multiprocessing as mp

ctx = mp.get_context("spawn")      # 或 "fork"/"forkserver"
with ctx.Pool(4) as pool:
    ...
```

### 10.3.3 进程间通信的成本

进程不共享内存（默认），参数/返回值要经过序列化（pickle）和管道：

```python
# 每次 submit 都要 pickle 参数、反序列化、再 pickle 结果
future = executor.submit(process_large_data, huge_list)
```

成本来源：

- pickle/unpickle CPU 与内存；
- 管道/队列拷贝；
- 进程启动/导入；
- 结果收集的调度开销。

优化：

- 任务粒度不要太细（不要为 1 毫秒的计算起进程）；
- 用 `chunksize` 批量提交（`Pool.map`/`Executor.map`）；
- 大数组用 `multiprocessing.shared_memory` 或 NumPy memmap，避免复制；
- 用 `initializer` 在 worker 启动时加载只读数据，避免每个任务重复加载；
- 用 `max_tasks_per_child` 定期回收 worker，隔离内存泄漏/碎片（3.11+）。

### 10.3.4 进程池示例（带初始化与分块）

```python
import multiprocessing as mp

def init_worker(config):
    global _CONFIG
    _CONFIG = config

def work(chunk):
    return sum(x * x for x in chunk)

def main():
    chunks = [list(range(i, i + 1000)) for i in range(0, 10_000, 1000)]
    with mp.Pool(processes=4, initializer=init_worker, initargs=({"mode": "fast"},)) as pool:
        results = pool.map(work, chunks)
    print(sum(results))

if __name__ == "__main__":
    main()
```

## 10.4 `concurrent.futures`：统一线程/进程接口

### 10.4.1 选择器

```python
from concurrent.futures import ProcessPoolExecutor, ThreadPoolExecutor

def io_task(x):
    return x

def cpu_task(x):
    return sum(i * i for i in range(x))

# I/O 密集
with ThreadPoolExecutor(max_workers=16) as ex:
    list(ex.map(io_task, range(100)))

# CPU 密集
with ProcessPoolExecutor(max_workers=4) as ex:
    list(ex.map(cpu_task, range(100)))
```

### 10.4.2 常用 API 的语义差异

| API | 行为 | 注意 |
| :--- | :--- | :--- |
| `submit(fn, *args)` | 提交单个任务，返回 `Future` | 最灵活 |
| `map(fn, *iterables)` | 并行 map，返回迭代器 | **立即收集输入**；结果按输入顺序 |
| `as_completed(fs)` | 按完成顺序产出 | 适合"谁先完成先处理" |
| `wait(fs, timeout, return_when)` | 等待一组任务 | 不抛异常，返回 done/not_done |
| `Future.result(timeout)` | 取结果，可超时 | 超时**不会**停止任务 |
| `Future.cancel()` | 尝试取消 | 仅未开始的任务能取消 |
| `Executor.shutdown(wait, cancel_futures)` | 关闭池 | `cancel_futures` 3.9+ |

### 10.4.3 进程池的坑

- 子进程里不能再调用同一个 `ProcessPoolExecutor`（会死锁）；
- 提交的函数/参数必须可 pickle；lambda、局部函数、REPL 定义通常不行；
- Windows 上 `max_workers` 有上限（文档：≤61）；
- 子进程崩溃会抛 `BrokenProcessPool`，池不可继续使用；
- 任务抛出的异常在 `future.result()` 时重新抛出，未取结果可能静默丢失。

```python
from concurrent.futures import ProcessPoolExecutor

def main():
    with ProcessPoolExecutor(max_workers=2) as ex:
        futures = [ex.submit(pow, 2, i) for i in range(10)]
        for f in futures:
            try:
                print(f.result())
            except Exception as exc:
                print("task failed:", exc)

if __name__ == "__main__":
    main()
```

## 10.5 asyncio：单线程内的高并发 I/O

### 10.5.1 基本模型

```python
import asyncio

async def fetch(i):
    await asyncio.sleep(0.1)       # 模拟 I/O
    return i * 2

async def main():
    results = await asyncio.gather(*(fetch(i) for i in range(10)))
    print(results)

asyncio.run(main())
```

关键概念：

- **协程**：`async def` 定义，`await` 挂起；
- **事件循环**：调度协程/回调，单线程执行；
- **Task**：被事件循环调度的协程；
- **Future**：低层结果占位符，通常用 Task 即可；
- `asyncio.run()`：创建/关闭事件循环，适合程序入口，只调用一次。

### 10.5.2 并发限制与背压

```python
import asyncio

async def fetch(i, sem):
    async with sem:                 # 最多同时 N 个
        await asyncio.sleep(0.1)
        return i

async def main():
    sem = asyncio.Semaphore(5)
    results = await asyncio.gather(*(fetch(i, sem) for i in range(100)))
    print(len(results))

asyncio.run(main())
```

不要无限制地 `gather` 十万个任务：每个 Task 都有内存/调度开销，且会瞬间打开大量连接，打爆下游或本地 FD。

### 10.5.3 超时、取消与异常

```python
import asyncio

async def slow():
    await asyncio.sleep(10)
    return "done"

async def main():
    try:
        async with asyncio.timeout(1):      # 3.11+
            result = await slow()
    except TimeoutError:
        print("timeout")

asyncio.run(main())
```

要点：

- `asyncio.timeout()` 是 3.11+；更早版本用 `asyncio.wait_for()`；
- 取消是协作式的：协程必须在 `await` 点才会响应 `CancelledError`；
- 不要吞掉 `CancelledError`（会破坏取消语义），除非明确处理后重新抛出；
- 清理逻辑放在 `finally` 中，用 `async with` 管理资源；
- `TaskGroup`（3.11+）能更好地管理一组任务的结构化并发与异常传播。

```python
async def main():
    async with asyncio.TaskGroup() as tg:
        tg.create_task(slow())
        tg.create_task(slow())
```

### 10.5.4 协程里不要阻塞

```python
import asyncio
import time

async def bad():
    time.sleep(1)          # ❌ 阻塞整个事件循环，所有任务一起卡住

async def good():
    await asyncio.to_thread(time.sleep, 1)   # ✅ 3.9+，放到线程池
```

CPU 密集任务：

- 用 `asyncio.to_thread` 仍受 GIL 限制，不会真正并行；
- 真正 CPU 并行用 `ProcessPoolExecutor` + `loop.run_in_executor`；
- 不要把 CPU 密集任务直接放在协程里。

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

def cpu_bound(n):
    return sum(i * i for i in range(n))

async def main():
    loop = asyncio.get_running_loop()
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, cpu_bound, 1_000_000)
    print(result)

asyncio.run(main())
```

## 10.6 背压：并发系统的生命线

**背压（backpressure）**：当下游处理不过来时，让上游慢下来/排队/拒绝，而不是无限堆积。

没有背压的典型后果：

- 队列无限增长 → 内存 OOM；
- 任务无限提交 → 线程/协程/FD 耗尽；
- 下游被打爆 → 超时、重试、雪崩。

### 10.6.1 有界队列

```python
import queue
import threading

q = queue.Queue(maxsize=100)     # 有界

def producer():
    for i in range(1000):
        q.put(i)                 # 队列满时阻塞，形成背压
    q.put(None)                  # 结束信号

def consumer():
    while True:
        item = q.get()
        try:
            if item is None:
                break
            process(item)
        finally:
            q.task_done()
```

`multiprocessing.Queue(maxsize=...)` 也支持上限，但其 `maxsize` 受 feeder 线程影响，通常只应视为近似值。

### 10.6.2 信号量限流

```python
import asyncio

sem = asyncio.Semaphore(100)

async def handle(request):
    async with sem:
        return await process(request)
```

### 10.6.3 常见背压策略

| 策略 | 做法 | 适用 |
| :--- | :--- | :--- |
| 阻塞等待 | 队列满时`put`阻塞 | 生产/消费速率可协调 |
| 丢弃 | 队列满时丢弃新/旧数据 | 实时监控、可丢数据的场景 |
| 拒绝 | 返回 429/503 | 在线服务保护 |
| 限流 | 令牌桶/漏桶/信号量 | API 网关、下游保护 |
| 降级 | 返回缓存/默认值 | 弱依赖失败 |
| 批量 | 聚合小请求，降低下游压力 | 消息/日志/指标 |
| 分页/游标 | 按需拉取，而不是一次推完 | 大数据传输 |

## 10.7 超时、重试与取消

### 10.7.1 超时的层次

| 层次 | API | 是否停止工作 |
| :--- | :--- | :--- |
| socket 连接/读写 | `socket.settimeout()`、`requests timeout` | 是（抛异常） |
| 子进程 | `subprocess.run(timeout=...)` | 是（杀掉并等待） |
| `Popen.communicate` | `timeout=` | 否（抛 `TimeoutExpired`，需自行 kill） |
| Future | `future.result(timeout=...)` | 否（任务继续跑） |
| asyncio | `asyncio.timeout`/`wait_for` | 协作式取消 |
| 线程 | 无强制 kill | 只能协作式停止 |

```python
import subprocess

try:
    result = subprocess.run(
        ["long-running-command"],
        capture_output=True,
        text=True,
        timeout=30,
        check=True,
    )
except subprocess.TimeoutExpired:
    print("command timed out")
```

> `subprocess.run(timeout=...)` 超时会杀掉子进程并等待其结束；`Popen.communicate(timeout=...)` 不会自动杀，需要自己 `kill()` 后再次 `communicate()`/`wait()`。

### 10.7.2 重试要有边界

```python
import random
import time

def retry(fn, attempts=3, base_delay=0.1):
    for i in range(attempts):
        try:
            return fn()
        except TransientError:
            if i == attempts - 1:
                raise
            delay = base_delay * (2 ** i)
            time.sleep(delay + random.uniform(0, base_delay))  # 抖动
```

原则：

- 只重试**可恢复**错误（超时、限流、5xx），不要重试参数错误/鉴权失败；
- 指数退避 + 抖动，避免重试风暴；
- 设置总时间预算，不要无限重试；
- 幂等操作才能安全重试（或使用幂等键）；
- 重试会放大下游压力，必须与熔断/限流配合。

## 10.8 并发规模怎么定

### 10.8.1 CPU 密集

- 进程数 ≈ 可用 CPU 核数（容器里用 `os.process_cpu_count()`/cgroup 配额，而不是 `os.cpu_count()` 的主机核数）；
- 留出 1 个核给系统/GC/监控，或根据实测调整；
- 任务粒度要足够大，抵消进程启动/pickle 成本。

### 10.8.2 I/O 密集

用 Little 定律估算：

```text
所需并发 ≈ 目标吞吐（req/s） × 单请求平均等待时间（s）
```

例如目标 500 req/s、下游平均 100 ms，则需要约 50 个并发。再用压测验证：

- 并发太小 → 吞吐上不去；
- 并发太大 → 下游排队、超时、内存上涨；
- 找"吞吐不再上升、延迟开始陡增"的拐点。

### 10.8.3 线程/进程数量与资源

| 资源 | 每单位成本 | 风险 |
| :--- | :--- | :--- |
| 线程 | 栈（虚拟内存）、调度、上下文切换 | 线程过多导致切换开销/内存上涨 |
| 进程 | 独立解释器、内存、启动/导入 | 内存翻倍、启动慢、通信贵 |
| 协程 | 对象/栈帧、调度 | 数量过大导致内存/调度开销 |
| 连接 | FD、内存、服务端配额 | FD 耗尽、握手风暴 |

## 10.9 fork、COW 与共享内存

### 10.9.1 fork + 线程 = 危险

在有多线程的进程里 `fork`，子进程只继承调用 fork 的线程；其他线程持有的锁可能永远无法释放，导致子进程死锁。常见规避：

- 用 `spawn`/`forkserver`；
- fork 前确保没有后台线程/锁；
- 用 `os.register_at_fork` 注册 fork 前后回调（谨慎）；
- 不要在子进程里使用父进程的数据库连接/日志锁。

### 10.9.2 COW 不是免费的

fork 后父子进程共享内存页，写时复制。如果子进程触发 GC 扫描/修改大量对象，会导致大量 COW，内存翻倍。

缓解：

- fork 前 `gc.freeze()`（见 8.8）；
- 父进程尽早 `gc.disable()`，子进程 `gc.enable()`；
- 避免在子进程里修改大块继承数据；
- 用 `spawn` + 共享内存/只读数据文件。

### 10.9.3 `shared_memory`

Python 3.8+ 提供 `multiprocessing.shared_memory`，让多进程共享一块内存，避免大数组复制：

```python
from multiprocessing import shared_memory
import numpy as np

# 创建共享内存并写入
shm = shared_memory.SharedMemory(create=True, size=1024 * 1024)
arr = np.ndarray((1024 * 1024 // 8,), dtype=np.float64, buffer=shm.buf)
arr[:] = 1.0

# 另一个进程通过名字附加
# existing = shared_memory.SharedMemory(name=shm.name)
# view = np.ndarray(..., buffer=existing.buf)

shm.close()
shm.unlink()       # 负责创建的一方最后 unlink
```

注意：共享内存需要显式清理（`close` + `unlink`），否则会泄漏；并发读写仍需同步。

## 10.10 本章要点与坑清单

**要点**

- CPU 密集用进程/扩展，I/O 密集用线程/协程；高并发 I/O 用 asyncio。
- GIL 不等于线程安全；共享可变状态必须加锁或无共享。
- 进程间通信要 pickle/拷贝，任务粒度不能太细；大数组用共享内存。
- 并发必须有上限和背压；无界队列/无限制 `gather` 是事故温床。
- 超时、重试、取消要分层设计；`Future.result(timeout)` 不会停止任务。
- 容器里按 cgroup 配额/`os.process_cpu_count()` 定并发，不要按主机核数。

**坑**

- CPU 密集任务开多线程，越开越慢。
- 线程池里跑永不结束的任务，导致进程无法退出。
- 进程池里提交 lambda/局部函数，pickle 失败。
- `Executor.map` 对超大迭代器一次性收集，内存爆炸。
- 协程里调用同步阻塞库，卡住整个事件循环。
- 无限制创建协程/线程/连接，打爆下游或本地 FD。
- 用 `future.result(timeout)` 以为能停止任务。
- fork 多线程进程导致子进程死锁。

---

# 第 11 章 资源生命周期：with、ExitStack、子进程、临时文件与 finalizer

## 11.1 确定性清理：不要等 GC 帮你收尾

Python 有 GC，但**资源释放不能依赖 GC**：

- GC 只负责内存，不负责文件/socket/锁/子进程/临时目录；
- 循环引用会让对象延迟回收；
- 解释器退出时 `__del__` 不保证执行；
- 进程被 SIGKILL 时，任何清理代码都不会运行。

正确的做法是**确定性地释放**：谁申请，谁释放；用 `with`/`try...finally` 保证异常路径也释放。

## 11.2 上下文管理器协议

```python
class Resource:
    def __enter__(self):
        print("acquire")
        return self

    def __exit__(self, exc_type, exc, tb):
        print("release")
        # 返回 True 会吞掉异常；通常返回 None/False
        return None

with Resource() as r:
    print("use")
```

`__exit__` 的三个参数：

| 参数 | 正常退出 | 异常退出 |
| :--- | :--- | :--- |
| `exc_type` | `None` | 异常类型 |
| `exc` | `None` | 异常实例 |
| `tb` | `None` | traceback |

**返回值**：`True` 表示抑制异常（慎用）；`None`/`False` 表示继续传播。

### 11.2.1 `contextlib.contextmanager`

```python
from contextlib import contextmanager
from time import perf_counter

@contextmanager
def timer(label):
    start = perf_counter()
    try:
        yield
    finally:
        print(f"{label}: {perf_counter() - start:.3f}s")

with timer("work"):
    do_work()
```

要点：

- `yield` 之前是 acquire，之后是 release；
- 必须用 `try/finally` 保证异常时也执行 release；
- 生成器只能进入一次；
- 不要在 `yield` 之后吞掉异常而不处理。

### 11.2.2 `closing`、`suppress`、`nullcontext`

```python
from contextlib import closing, suppress, nullcontext
import socket

# 对象有 close() 但不支持 with
with closing(socket.socket()) as sock:
    ...

# 忽略特定异常
with suppress(FileNotFoundError):
    os.remove("maybe-missing.txt")

# 可选上下文：根据条件决定是否真正进入
ctx = open("out.txt", "w") if write else nullcontext()
with ctx as f:
    ...
```

## 11.3 `ExitStack`：动态管理多个资源

当资源数量在运行时才确定（如动态打开一组文件/连接），用 `ExitStack`：

```python
from contextlib import ExitStack

def process_files(paths):
    with ExitStack() as stack:
        files = [stack.enter_context(open(p, encoding="utf-8")) for p in paths]
        for f in files:
            process(f)
    # 退出时按后进先出顺序关闭所有文件
```

其他用法：

```python
from contextlib import ExitStack

with ExitStack() as stack:
    conn = stack.enter_context(connect_db())
    lock = stack.enter_context(acquire_lock())
    stack.callback(cleanup_temp_dir, "/tmp/work")
    do_work(conn)
```

- `enter_context(cm)`：进入上下文并注册退出；
- `callback(fn, *args)`：注册普通清理函数；
- `push(cm)`：注册一个上下文管理器；
- 退出顺序是**后进先出**（LIFO），与嵌套 `with` 一致。

异步版本：

```python
from contextlib import AsyncExitStack

async def main():
    async with AsyncExitStack() as stack:
        session = await stack.enter_async_context(create_session())
        conn = await stack.enter_async_context(connect_db())
        await do_work(session, conn)
```

`contextlib.aclosing`（3.10+）用于异步 `close()` 对象：

```python
from contextlib import aclosing

async def main():
    async with aclosing(open_async_stream()) as stream:
        async for item in stream:
            process(item)
```

## 11.4 文件与 socket

### 11.4.1 文件

```python
# 推荐：with 自动关闭
with open("data.txt", encoding="utf-8") as f:
    for line in f:
        process(line)
```

常见错误：

```python
# ❌ 异常时不会关闭
f = open("data.txt")
data = f.read()
f.close()

# ❌ 把文件对象传给别处，生命周期失控
def read_later(path):
    return open(path)          # 调用方很容易忘记 close
```

需要把"打开的文件"作为返回值时，考虑返回上下文管理器/使用 `ExitStack`，或明确文档要求调用方负责关闭。

### 11.4.2 socket

`socket.socket` 支持上下文管理器，退出时关闭：

```python
import socket

with socket.create_connection(("example.com", 80), timeout=5) as sock:
    sock.sendall(b"HEAD / HTTP/1.0\r\nHost: example.com\r\n\r\n")
    data = sock.recv(4096)
```

HTTP 客户端：

```python
import requests

with requests.Session() as session:
    resp = session.get("https://example.com", timeout=5)
    print(resp.status_code)
    # Response 也支持 with，确保连接释放
    with session.get("https://example.com", timeout=5, stream=True) as resp:
        for chunk in resp.iter_content(8192):
            process(chunk)
```

### 11.4.3 文件描述符继承

子进程默认 `close_fds=True`（POSIX），不会继承无关 FD；如果确实要传 FD：

```python
import subprocess

with open("data.bin", "rb") as f:
    subprocess.run(
        ["consumer"],
        pass_fds=(f.fileno(),),     # POSIX：显式传递
        check=True,
    )
```

Windows 的 FD 继承规则不同，跨平台代码要特别小心。

## 11.5 子进程：最容易泄漏的资源之一

### 11.5.1 `subprocess.run`：首选

```python
import subprocess

result = subprocess.run(
    ["git", "status", "--short"],
    capture_output=True,
    text=True,
    timeout=30,
    check=False,
)
print(result.returncode, result.stdout)
```

优点：自动 wait、自动关闭管道、支持超时、`check=True` 统一抛错。

### 11.5.2 `Popen`：需要交互/流式时

```python
import subprocess

with subprocess.Popen(
    ["python", "-c", "print('hello')"],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True,
) as proc:
    stdout, stderr = proc.communicate(timeout=10)
print(stdout.strip())
```

**关键：用 `communicate()`，不要先 `wait()` 再读管道。**

`PIPE` 的缓冲区有限（通常 64KB）；如果子进程输出很多，而你只 `wait()` 不读，子进程会因管道写满而阻塞，父进程又在等子进程退出 → **死锁**。

```python
# ❌ 可能死锁
proc = subprocess.Popen(["producer"], stdout=subprocess.PIPE)
proc.wait()
data = proc.stdout.read()
```

`Popen` 上下文管理器会关闭管道并等待子进程；如果子进程需要输入，先 `communicate(input=...)`，否则可能在 `__exit__` 等待时挂住。

### 11.5.3 超时与杀进程树

`Popen.communicate(timeout=...)` 超时不会自动杀子进程：

```python
import subprocess

proc = subprocess.Popen(["long-task"], stdout=subprocess.PIPE, text=True)
try:
    out, _ = proc.communicate(timeout=10)
except subprocess.TimeoutExpired:
    proc.kill()
    out, _ = proc.communicate()      # 回收僵尸，读取剩余输出
```

如果子进程又创建了孙进程，杀父进程不一定杀掉整棵树：

```python
# POSIX：让子进程成为新进程组组长，然后杀整个组
proc = subprocess.Popen(
    ["bash", "-c", "long-task"],
    start_new_session=True,
)
# 超时后：
# os.killpg(os.getpgid(proc.pid), signal.SIGKILL)
```

Windows 可用 `taskkill /PID <pid> /T /F`，或使用 Job Object（需 `pywin32`/ctypes）。跨平台库（如 `psutil`）可以递归终止子进程。

### 11.5.4 `shell=True` 的风险

```python
# ❌ 用户输入拼进 shell，命令注入
subprocess.run(f"ls {user_input}", shell=True)

# ✅ 参数列表，不经 shell
subprocess.run(["ls", user_input], check=True)
```

资源角度：`shell=True` 多一层 shell 进程，信号/退出码/清理更复杂；除非确实需要 shell 特性，否则不用。

## 11.6 临时文件与临时目录

```python
import tempfile
from pathlib import Path

# 临时目录：退出时递归删除
with tempfile.TemporaryDirectory() as tmpdir:
    path = Path(tmpdir) / "work.txt"
    path.write_text("data", encoding="utf-8")
    process(path)

# 临时文件：delete=True 时关闭即删除
with tempfile.NamedTemporaryFile("w+", encoding="utf-8", delete=True) as f:
    f.write("data")
    f.flush()
    f.seek(0)
    print(f.read())
```

注意：

- Windows 上 `NamedTemporaryFile(delete=True)` 打开期间通常不能再次打开同名文件；需要"关闭后仍保留"时用 `delete=False` 并自行清理；
- `tempfile.mkstemp()` 返回 `(fd, path)`，记得 `os.close(fd)` 或 `os.fdopen`；
- 临时目录要确保在异常路径也删除（用 `with`/`try...finally`）；
- 安全：不要用可预测的临时文件名，不要给临时文件过宽权限（参见安全编码规范）。

## 11.7 数据库连接与游标

### 11.7.1 `sqlite3` 的上下文管理器只负责事务

```python
import sqlite3

conn = sqlite3.connect("app.db")
try:
    with conn:                 # 提交/回滚事务，但不会关闭连接
        conn.execute("INSERT INTO t VALUES (1)")
finally:
    conn.close()               # 必须显式关闭
```

需要"关闭 + 事务"时：

```python
from contextlib import closing

with closing(sqlite3.connect("app.db")) as conn:
    with conn:
        conn.execute("INSERT INTO t VALUES (1)")
```

### 11.7.2 连接池

- Web 服务不要每请求新建连接；
- 连接池大小要匹配数据库 `max_connections` 与并发；
- 连接要有最大生命周期/空闲回收（防止服务端断连后拿到坏连接）；
- 使用前 `ping`/健康检查，使用后归还；
- 事务失败要回滚，连接要归还而不是泄漏。

```python
# 伪代码：连接池的典型使用
with pool.connection() as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT 1")
```

## 11.8 异步资源与取消

```python
import aiohttp

async def fetch(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url, timeout=10) as resp:
            return await resp.text()
```

要点：

- 异步资源用 `async with`，确保 `__aexit__` 被 await；
- 取消时 `CancelledError` 会在 `await` 点抛出，`finally` 中的清理要避免再次长时间等待；
- 不要在 `finally` 中 `await` 一个可能永远挂起的操作；给清理设置超时；
- 用 `AsyncExitStack` 管理动态异步资源；
- 事件循环关闭前确保所有任务已取消/等待完成。

```python
import asyncio

async def worker():
    try:
        await do_work()
    finally:
        # 清理也要有超时，避免取消时卡死
        async with asyncio.timeout(5):
            await cleanup()

async def main():
    task = asyncio.create_task(worker())
    await asyncio.sleep(1)
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        print("cancelled")

asyncio.run(main())
```

## 11.9 `__del__`、`weakref.finalize` 与 `atexit`

### 11.9.1 `__del__` 不是可靠的清理钩子

```python
class Bad:
    def __del__(self):
        self.file.close()      # 可能不执行、可能执行时模块已销毁、异常会被忽略
```

问题：

- 引用计数归零/GC 时才调用，时机不确定；
- 解释器退出时，全局对象/模块可能已经不可用；
- `__del__` 中的异常会被忽略（只打印到 stderr）；
- 对象可能被复活（在 `__del__` 中保存 `self`），导致再次回收；
- 循环引用中的 `__del__` 顺序不确定（PEP 442 后可回收，但顺序不保证）。

**结论**：`__del__` 只适合做"最后一道保险"的日志/断言，不要依赖它释放关键资源。

### 11.9.2 `weakref.finalize`

```python
import weakref

class Resource:
    def __init__(self):
        self.handle = open("data.txt")

    def close(self):
        if not self.handle.closed:
            self.handle.close()

def cleanup(path):
    print("finalize", path)

res = Resource()
finalizer = weakref.finalize(res, cleanup, "data.txt")
finalizer.atexit = True        # 解释器正常退出时也执行（默认 True）

res.close()
finalizer()                    # 也可以显式调用，保证只执行一次
```

优点：

- 不要求对象定义 `__del__`；
- 可显式调用，且保证只执行一次；
- 可设置 `atexit=False`；
- 不阻止对象被回收。

但它仍然**不能替代显式 `close()`**：进程被 SIGKILL、调用 `os._exit()`、解释器崩溃时不会执行。

### 11.9.3 `atexit`

```python
import atexit

def shutdown():
    print("shutting down")

atexit.register(shutdown)
```

特点：

- 后进先出（LIFO）执行；
- 正常解释器退出时执行；
- `os._exit()`、SIGKILL、崩溃时不执行；
- 回调中的异常会被忽略；
- 不要在 `atexit` 里做长时间阻塞/等待线程（可能卡住退出）。

### 11.9.4 优雅停机与信号

```python
import signal
import sys

shutting_down = False

def handle_sigterm(signum, frame):
    global shutting_down
    shutting_down = True
    # 设置停止标志，让主循环退出；不要在信号处理器里做复杂操作

signal.signal(signal.SIGTERM, handle_sigterm)
```

要点：

- `signal.signal` 只能在主线程调用；
- 信号处理器应尽量短小，只设置标志/写入自管道；
- Web 服务收到 SIGTERM 后：停止接收新请求 → 等待在途请求 → 关闭连接池/数据库 → 退出；
- Windows 的信号支持有限；服务框架通常有自己的优雅停机机制。

## 11.10 资源泄漏检测

### 11.10.1 开启 `ResourceWarning`

`ResourceWarning` 默认被忽略：

```python
import warnings
warnings.simplefilter("always", ResourceWarning)
```

更简单的方式是用开发模式运行：

```bash
python -X dev your_script.py
# 或
PYTHONDEVMODE=1 python your_script.py
```

开发模式下，未关闭的文件/socket 会在 GC 时发出 `ResourceWarning`，帮你定位"忘记 close"的代码。

### 11.10.2 统计 FD/线程

```python
import os
import psutil

proc = psutil.Process(os.getpid())
print("fds:", proc.num_fds())          # Unix
print("threads:", proc.num_threads())

# Linux：查看打开的 FD 指向
# ls -l /proc/<pid>/fd
```

### 11.10.3 常见泄漏点清单

| 资源 | 泄漏原因 | 修复 |
| :--- | :--- | :--- |
| 文件 | 忘记 `close`、异常路径未关闭 | `with open` |
| socket | 连接未关闭、响应未释放 | `with`、连接池、`response.close()` |
| 子进程 | 只 `Popen` 不 `wait/communicate` | 上下文管理器/`communicate` |
| 管道 | `PIPE` 未读导致子进程阻塞 | `communicate` |
| 临时文件 | `delete=False` 未清理 | `TemporaryDirectory`/`finally` |
| 数据库连接 | 连接池未归还、异常未回滚 | 上下文管理器/池 |
| 锁 | 异常时未释放 | `with lock` |
| 线程 | 线程池未 shutdown、无限创建 | 有界池/`with` |
| 日志 handler | 重复添加 handler | 检查/移除、`logging.shutdown` |
| 事件监听 | 注册后未注销 | 弱引用/显式注销 |

## 11.11 资源生命周期检查表

- [ ] 所有文件/socket/数据库连接都用 `with` 或 `try...finally` 释放。
- [ ] 子进程用 `communicate()`，避免 `PIPE` 死锁；设置超时。
- [ ] 临时文件/目录在异常路径也会清理。
- [ ] 锁的获取/释放成对，临界区尽量小。
- [ ] 线程池/进程池用 `with` 或显式 `shutdown`。
- [ ] 异步资源用 `async with`，取消时清理有超时。
- [ ] 不依赖 `__del__` 释放关键资源；必要时用 `weakref.finalize`/`atexit` 兜底。
- [ ] 开发/测试环境开启 `-X dev` 或 `ResourceWarning`。
- [ ] 监控 FD/线程/连接数，设置告警。

## 11.12 本章要点与坑清单

**要点**

- 资源释放要确定性：`with` > `try/finally` > finalizer。
- `ExitStack` 适合动态数量资源；`AsyncExitStack` 管理异步资源。
- 子进程用 `communicate()`，避免管道死锁；超时要 kill + 回收。
- `sqlite3` 的 `with conn` 只管事务，不管关闭。
- `__del__` 不可靠；`weakref.finalize` 可显式调用且只执行一次；`atexit` 只在正常退出时执行。
- 用 `-X dev`/`ResourceWarning` 抓未关闭资源。

**坑**

- 把 `close()` 写在 `try` 最后一行，异常时执行不到。
- `Popen` 只 `wait()` 不读 `PIPE`，子进程写满管道后死锁。
- 以为 `with sqlite3.connect(...)` 会关闭连接。
- 在 `__del__` 里做网络/文件操作。
- 线程池未关闭，进程无法退出。
- 信号处理器里做复杂清理/加锁。

---

# 第 12 章 资源限额与容量：rlimit、cgroup、FD、线程与内存预算

## 12.1 为什么要在 Python 层关心限额

Python 程序通常不是"把 CPU 算完"就结束，而是活在某个资源边界内：

- 容器有 memory/cpu/pids 限制；
- 操作系统有 FD/进程数/文件大小限制；
- 下游数据库有连接数上限；
- 磁盘有容量/IOPS 上限；
- 云厂商按 CPU/内存计费。

不设限的后果：一个慢查询/内存泄漏/连接泄漏，拖垮整个进程甚至整台机器。

## 12.2 Unix `resource`：进程级硬限制

`resource` 模块仅在 Unix 可用（Windows 不可用）。常用限制：

| 常量 | 限制 | 典型用途 |
| :--- | :--- | :--- |
| `RLIMIT_AS` | 进程地址空间（虚拟内存） | 防无限内存分配 |
| `RLIMIT_DATA` | 数据段大小 | 限制堆增长 |
| `RLIMIT_NOFILE` | 打开 FD 数 | 防 FD 泄漏 |
| `RLIMIT_NPROC` | 用户进程/线程数 | 防 fork 炸弹 |
| `RLIMIT_CPU` | CPU 秒数 | 防死循环/超长任务 |
| `RLIMIT_FSIZE` | 单文件最大字节数 | 防日志/临时文件写满磁盘 |
| `RLIMIT_CORE` | core dump 大小 | 禁 core/限制大小 |

```python
import resource

# 查看当前限制（soft, hard）
soft, hard = resource.getrlimit(resource.RLIMIT_NOFILE)
print(soft, hard)

# 设置软限制（不能超过硬限制）
resource.setrlimit(resource.RLIMIT_NOFILE, (4096, hard))

# 限制 CPU 时间：超过后收到 SIGXCPU（默认终止）
resource.setrlimit(resource.RLIMIT_CPU, (60, 60))
```

注意：

- 子进程继承 rlimit；在父进程设置会影响所有子进程；
- 硬限制只有特权进程能提高；
- `RLIMIT_AS` 限制的是虚拟地址空间，可能误伤线程栈/mmap；
- `RLIMIT_NPROC` 是"每用户"限制，容器/多用户环境要谨慎；
- 超限通常表现为 `OSError`/`MemoryError`/信号，错误信息不一定直观。

## 12.3 容器限额：cgroup v1/v2

容器（Docker/Kubernetes）通过 cgroup 限制资源：

| 资源 | cgroup v2 文件 | 说明 |
| :--- | :--- | :--- |
| 内存 | `memory.max`、`memory.current` | 超限触发 OOM kill |
| CPU | `cpu.max` | 配额/周期，限制 CPU 时间 |
| 进程数 | `pids.max` | 防 fork 炸弹 |
| IO | `io.max` | 限制磁盘带宽/IOPS |

常见陷阱：

- **`os.cpu_count()` 返回宿主核数，不是容器配额**；3.13+ 可用 `os.process_cpu_count()`，它考虑进程可用 CPU（仍不等于 cgroup 配额，需结合 cgroup 读取）；
- 内存限制是 cgroup 的，Python 的 `MemoryError` 不一定先发生，可能直接被 OOM killer 杀；
- `pids.max` 会限制线程数（Linux 线程也是任务），线程池过大可能触发；
- 容器内 `/proc/meminfo` 可能显示宿主内存，不能直接当可用内存。

```python
import os

print(os.cpu_count())                 # 宿主逻辑 CPU 数
print(os.process_cpu_count())         # 3.13+：当前进程可用 CPU 数
```

读取 cgroup（Linux）：

```python
from pathlib import Path

def read_cgroup_memory_limit():
    # cgroup v2
    p = Path("/sys/fs/cgroup/memory.max")
    if p.exists():
        value = p.read_text().strip()
        return None if value == "max" else int(value)
    # cgroup v1
    p = Path("/sys/fs/cgroup/memory/memory.limit_in_bytes")
    if p.exists():
        return int(p.read_text().strip())
    return None
```

> 生产环境建议用 `psutil`、容器运行时 API 或平台 SDK 读取配额，而不是自己解析所有 cgroup 版本。

## 12.4 文件描述符预算

每个打开的以下对象都占一个 FD：

- 文件、目录（`os.scandir` 的迭代器）；
- socket、管道；
- `epoll`/`kqueue`/IOCP 句柄；
- 数据库连接；
- 子进程的 stdin/stdout/stderr 管道；
- 临时文件。

**预算公式（粗略）**：

```text
所需 FD ≈ 监听 socket
        + 每连接 1 个 socket
        + 连接池（数据库/Redis/HTTP）
        + 日志文件/轮转句柄
        + 临时文件/管道
        + 监控/追踪 exporter
        + 安全余量（20%–30%）
```

排查：

```python
import os
import psutil

proc = psutil.Process(os.getpid())
print("fds:", proc.num_fds())

# Linux：看具体打开了什么
# ls -l /proc/<pid>/fd | head
```

常见症状：

- `OSError: [Errno 24] Too many open files`；
- `ConnectionResetError`/`Connection aborted`（连接池拿不到 FD）；
- 日志突然写不进去。

处理：

- 修复泄漏（`with`、连接池归还）；
- 提高 `RLIMIT_NOFILE`/容器限制；
- 降低连接池/线程池上限；
- 用 `selectors`/异步减少"每连接一线程一 FD"的模型。

## 12.5 线程与进程预算

### 12.5.1 线程栈

每个线程都有栈（默认大小因平台而异，Linux 常见 8MB 虚拟内存，Windows 常见 1MB 左右）。大量线程会消耗虚拟内存/调度资源。

```python
import threading

# 必须在启动线程前设置；影响之后创建的线程
threading.stack_size(512 * 1024)      # 512KB，需平台支持
```

注意：

- 栈太小可能导致递归/深调用栈崩溃；
- `stack_size` 不是所有平台都生效；
- 线程数不是越多越好：上下文切换、锁竞争、内存都会上升。

### 12.5.2 进程数

- CPU 密集：进程数 ≈ 可用核数；
- 每个进程有独立解释器/内存，启动/导入成本高；
- 进程池要设置 `max_workers`，不要用默认值无脑放大；
- 容器 `pids.max` 会限制进程+线程总数。

### 12.5.3 线程/进程数上限的经验

| 场景 | 建议起点 | 调整依据 |
| :--- | :--- | :--- |
| CPU 密集进程池 | `os.process_cpu_count()`（3.13+）或 `os.cpu_count()` | CPU 利用率、任务粒度 |
| I/O 线程池 | 16–64（视下游） | 压测吞吐/延迟拐点、下游连接上限 |
| 高并发异步 | 由信号量限制在几百到几千 | 下游承载、FD、内存 |
| 每请求一线程 | 不推荐 | 改用池/异步 |

## 12.6 时间限额

| 场景 | 手段 | 注意 |
| :--- | :--- | :--- |
| socket 连接/读写 | `settimeout()`、客户端 timeout | 连接与读取分开设置 |
| 子进程 | `subprocess.run(timeout=)` | 超时杀进程并等待 |
| 线程任务 | 无法强制终止 | 协作式标志/超时返回 |
| 异步任务 | `asyncio.timeout`/`wait_for` | 协作式取消 |
| 整个请求 | 框架中间件/网关超时 | 与下游超时分层 |
| 死循环/长计算 | `RLIMIT_CPU`、`signal.alarm`（Unix） | 信号处理要谨慎 |

`signal.alarm` 仅 Unix、整数秒：

```python
import signal

def on_alarm(signum, frame):
    raise TimeoutError("operation timed out")

# signal.signal(signal.SIGALRM, on_alarm)
# signal.alarm(5)
```

> 不要在库代码里随意改全局信号处理器；这会与宿主程序/框架冲突。

## 12.7 容量规划：从"能跑"到"扛得住"

### 12.7.1 关键指标

- **吞吐**：每秒请求/任务数；
- **延迟**：p50/p95/p99；
- **饱和度**：CPU、内存、FD、连接池、队列长度的使用率；
- **错误率**：超时、5xx、重试次数；
- **成本**：每请求 CPU/内存成本。

### 12.7.2 容量估算步骤

1. 压测单实例，找到"吞吐-延迟"拐点；
2. 留出安全余量（常见 30%–50%，视业务峰谷）；
3. 按峰值 QPS / 单实例容量 = 实例数；
4. 校验下游（数据库/缓存/第三方）是否能承受总并发；
5. 设计限流/降级/熔断，防止雪崩；
6. 定期复测（代码/依赖/流量变化后）。

### 12.7.3 过载保护

| 手段 | 作用 |
| :--- | :--- |
| 限流（令牌桶/漏桶） | 控制进入系统的请求速率 |
| 并发限制（信号量） | 控制同时在途请求数 |
| 队列上限 | 防止无限排队导致内存爆炸 |
| 超时 | 防止请求无限等待 |
| 熔断 | 下游持续失败时快速失败 |
| 降级 | 返回缓存/默认值/简化结果 |
| 隔离（bulkhead） | 不同依赖用独立池，避免互相拖垮 |
| 负载 shedding | 过载时主动拒绝低优先级请求 |

## 12.8 内存预算

```text
容器内存上限
  = Python 解释器/依赖基础内存
  + 每 worker 的峰值对象内存 × worker 数
  + 共享缓存/共享内存
  + 连接/缓冲区/栈
  + 安全余量（20%–30%）
```

注意：

- 多进程 `fork` 后 COW 可能让实际内存接近翻倍；
- 线程栈是虚拟内存，通常不计入 RSS，但会计入地址空间；
- 页缓存不属于进程 RSS，但容器 `memory.current` 可能包含；
- NumPy/Arrow 的大块内存可能绕过 pymalloc，OOM 风险更高；
- 容器 OOM 通常是 SIGKILL，应用来不及清理。

## 12.9 本章要点与坑清单

**要点**

- Unix 用 `resource.setrlimit` 限制 FD/CPU/内存/文件大小；Windows 需 Job Objects 等机制。
- 容器配额看 cgroup；`os.cpu_count()` 不等于可用 CPU，3.13+ 用 `os.process_cpu_count()`。
- FD、线程、连接都要做预算，不能只盯 CPU/内存。
- 超时要分层：连接、读取、请求、下游调用。
- 容量规划要压测找拐点、留下余量、保护下游。

**坑**

- 以为 `os.cpu_count()` 是容器可用核数。
- 只限制内存不限制 FD/线程，最后死于 `Too many open files`/`pids.max`。
- 线程池默认值直接上生产，不压测。
- 只设一个总超时，没有连接/读取分层超时。
- 过载时继续接请求，导致队列无限增长。

---

# 第 13 章 启动、导入与部署性能

## 13.1 冷启动都花在哪

一个 Python 服务的启动大致经历：

```text
解释器初始化
  → site 初始化（site-packages、.pth、sitecustomize/usercustomize）
  → 导入主模块/框架
  → 导入业务模块与第三方依赖
  → 建立连接池/加载配置/预热缓存
  → 开始监听/处理请求
```

冷启动慢的常见原因：

- 导入重型库（NumPy、pandas、TensorFlow、云 SDK）；
- 模块级执行昂贵初始化（读文件、建连接、加载模型）；
- 插件/入口点扫描（`importlib.metadata` 遍历）；
- 大量小模块/深层包结构；
- 字节码未预编译/文件系统慢；
- 容器镜像大、拉取慢；
- 服务框架的反射/装饰器扫描。

## 13.2 测量导入时间

```bash
# 解释器级：打印每个模块的导入耗时（微秒）
python -X importtime -c "import myapp"

# 环境变量等价写法
PYTHONPROFILEIMPORTTIME=1 python -c "import myapp"

# 只测某几个包
python -X importtime -c "import pandas, numpy"
```

输出按累计时间排序，重点看：

- `cumulative` 很大的顶层包（通常是 NumPy/pandas/框架）；
- 被重复导入的模块；
- 模块级执行的昂贵代码（不是 import 本身）；
- 循环导入导致的重复工作。

在代码里测量：

```python
import time

start = time.perf_counter()
import heavy_module
print(f"import heavy_module: {time.perf_counter() - start:.3f}s")
```

## 13.3 减少导入成本

### 13.3.1 延迟导入

```python
# ❌ 模块级导入，启动即加载
import pandas as pd

def analyze(rows):
    return pd.DataFrame(rows)

# ✅ 延迟到真正使用时
def analyze(rows):
    import pandas as pd
    return pd.DataFrame(rows)
```

适用：

- CLI 子命令（只有部分命令需要重型依赖）；
- Web 服务的可选功能；
- 冷启动敏感的 Serverless；
- 测试中不总是用到的依赖。

注意：延迟导入会把 `ImportError` 推迟到运行时，需在启动自检/测试中覆盖；并发首次导入可能有锁竞争（Python 导入锁）。

### 13.3.2 模块级不要做重活

```python
# ❌ 导入即建连接/读大文件/加载模型
# db = connect()
# model = load_model("big.bin")

# ✅ 延迟到初始化函数/首次使用
_db = None

def get_db():
    global _db
    if _db is None:
        _db = connect()
    return _db
```

### 13.3.3 模块级 `__getattr__` 惰性导出（PEP 562）

```python
# mypkg/__init__.py
def __getattr__(name):
    if name == "heavy":
        from . import heavy
        return heavy
    raise AttributeError(f"module {__name__!r} has no attribute {name!r}")
```

这样 `from mypkg import heavy` 会触发导入，而 `import mypkg` 不会。

### 13.3.4 精简依赖

- 用轻量库替代重型库（如用 `orjson` 替代部分 JSON 场景、用 `httpx`/`requests` 按需）；
- 拆分可选依赖（extras）；
- 避免在 `__init__.py` 里导入整个包的所有子模块；
- 用 `pipdeptree`/`importtime` 找出"大而不用"的依赖；
- 云 SDK 往往很重，按服务拆分导入。

## 13.4 字节码缓存与预编译

Python 导入 `.py` 时会编译为字节码并缓存到 `__pycache__/*.pyc`：

| 方式 | 作用 |
| :--- | :--- |
| 默认 | 首次导入写 `.pyc`，后续读取 |
| `python -m compileall -q .` | 构建时预编译所有模块 |
| `PYTHONDONTWRITEBYTECODE=1` / `-B` | 不写 `.pyc`（只读文件系统/容器） |
| `sys.pycache_prefix` / `PYTHONPYCACHEPREFIX` | 指定 pyc 目录 |
| 只读容器 | 构建阶段预编译，运行时不写 |

```bash
python -m compileall -q app/
```

注意：

- `.pyc` 与 Python 版本/魔数绑定，升级解释器后需重新编译；
- 容器只读文件系统时，运行时无法写 `__pycache__`，预编译可减少首次导入开销；
- 不要提交 `__pycache__` 到版本库；
- `-O`/`-OO` 会生成不同优化级别的 pyc（`assert`/docstring 处理不同）。

## 13.5 解释器启动参数

| 参数 | 作用 | 性能/资源影响 |
| :--- | :--- | :--- |
| `-S` | 不导入 `site` | 启动更快，但可能缺少 site-packages 路径 |
| `-E` | 忽略环境变量 | 避免外部环境影响 |
| `-s` | 不加载用户 site-packages | 减少启动扫描 |
| `-I` | 隔离模式（隐含 `-E -s`） | 更可预测的启动 |
| `-B` | 不写 pyc | 只读环境 |
| `-O` | 去掉 `assert`，`__debug__=False` | 微幅提速；不要依赖 assert 做校验 |
| `-OO` | 再去掉 docstring | 减小 pyc；影响依赖 docstring 的工具 |
| `-X importtime` | 打印导入耗时 | 诊断 |
| `-X dev` | 开发模式（警告、调试分配器等） | 诊断，不建议生产常开 |
| `-X tracemalloc=N` | 开启 tracemalloc | 诊断内存，有开销 |
| `-X frozen_modules=on/off` | 控制冻结模块 | 影响启动/导入 |
| `-X faulthandler` | 启用 faulthandler | 崩溃/卡死诊断 |

```bash
python -X importtime -X dev app.py
```

## 13.6 打包与分发

| 方式 | 优点 | 缺点 | 适用 |
| :--- | :--- | :--- | :--- |
| wheel + venv | 标准、可复现 | 需要 Python 环境 | 服务端、库 |
| zipapp | 单文件、简单 | 依赖需自带/环境提供 | 小工具 |
| PyInstaller | 单文件/目录可执行 | 体积大、启动慢、平台绑定 | 桌面/交付 |
| Nuitka | 编译为 C，可能更快 | 构建复杂、兼容性需验证 | 性能/闭源交付 |
| Docker 镜像 | 环境一致 | 镜像大、冷启动 | 服务端 |
| Serverless 包 | 按需运行 | 冷启动、包大小限制 | 事件驱动 |

### 13.6.1 容器镜像优化

```dockerfile
FROM python:3.13-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.13-slim
WORKDIR /app
COPY --from=builder /install /usr/local
COPY . .
RUN python -m compileall -q app
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1
CMD ["python", "-m", "app"]
```

要点：

- 多阶段构建，运行镜像不带编译工具；
- `PYTHONUNBUFFERED=1` 让日志及时输出（否则可能缓冲到进程退出）；
- 用 `exec` 形式启动（`CMD ["python", ...]`），让 Python 成为 PID 1 或正确转发信号；
- 非 root 运行；
- 预编译字节码；
- 固定基础镜像/依赖版本，保证可复现。

### 13.6.2 Serverless 冷启动

- 包越小越好（只打必要依赖）；
- 初始化逻辑放在 handler 外（复用执行环境），但要避免连接失效；
- 延迟导入重型依赖；
- 预热/定时 ping（如平台支持）；
- 合理设置内存（CPU 通常随内存比例分配）；
- 注意 `/tmp` 空间和生命周期。

## 13.7 版本升级与 ABI

- CPython 3.11/3.12/3.13 有持续的性能优化，升级可能"免费提速"；
- C 扩展需要与 Python ABI 匹配的 wheel；升级大版本通常要重装依赖；
- free-threaded 构建需要专门的扩展 wheel；
- `sys.version_info`、`sys.abiflags` 可用于兼容判断；
- 升级后重点回归：启动时间、内存、GC 行为、线程/进程池、C 扩展、序列化兼容性。

## 13.8 本章要点与坑清单

**要点**

- 冷启动 = 解释器 + site + 导入 + 初始化；先用 `-X importtime` 定位。
- 延迟导入、模块级 `__getattr__`、精简依赖能显著降低启动成本。
- 预编译 `.pyc` 对只读容器/首次导入有帮助。
- `-S/-E/-s/-I/-B/-O` 各有取舍，不要盲目全开。
- 容器要处理缓冲、信号、PID 1、只读文件系统。
- Serverless 重点控制包体积、初始化时机、冷启动延迟。

**坑**

- 在模块顶层建数据库连接/加载模型。
- `CMD python app.py` 用 shell 形式，信号无法正确转发。
- 容器只读文件系统却期望运行时写 pyc。
- 升级 Python 不回归 C 扩展/序列化兼容性。
- 用 `-OO` 后依赖 docstring 的代码失效。

---

# 第 14 章 生产可观测性与调优 playbook

## 14.1 可观测性的三根支柱

| 支柱 | 回答的问题 | Python 侧手段 |
| :--- | :--- | :--- |
| Metrics（指标） | 系统整体是否健康？趋势如何？ | Prometheus client、statsd、APM |
| Logs（日志） | 具体发生了什么？ | `logging`、结构化日志、集中采集 |
| Traces（追踪） | 一次请求慢在哪一段？ | OpenTelemetry、APM、分布式追踪 |

补充：**Profiling（剖析）** 回答"CPU/内存具体花在哪个函数"，通常按需开启。

## 14.2 指标：少而准

### 14.2.1 四个黄金信号

- **延迟**：p50/p95/p99，按接口/依赖拆分；
- **流量**：QPS、任务数、字节数；
- **错误**：错误率、超时率、重试次数；
- **饱和度**：CPU、内存、FD、连接池、队列长度。

### 14.2.2 USE 方法

对每个资源检查：

- **Utilization（使用率）**：CPU 使用率、内存使用率、连接池占用；
- **Saturation（饱和度）**：runqueue、队列长度、等待时间；
- **Errors（错误）**：分配失败、超时、拒绝。

### 14.2.3 Python 进程指标

| 指标 | 采集方式 | 告警建议 |
| :--- | :--- | :--- |
| CPU 使用率 | `psutil.cpu_percent()`、cgroup | 持续 > 80% |
| RSS/内存使用率 | `psutil.Process().memory_info().rss` | 接近容器上限 |
| FD 数 | `psutil.Process().num_fds()` | 接近 rlimit |
| 线程数 | `threading.active_count()`、`psutil` | 异常上涨 |
| GC 次数/耗时 | `gc.get_stats()`、`gc.callbacks` | 停顿影响 p99 |
| 队列长度 | 业务埋点 | 持续增长 |
| 连接池使用 | 客户端库指标 | 接近上限/等待超时 |
| 请求延迟 | 中间件/框架 | p99 超 SLA |

> 指标标签（label）不要用高基数维度（用户 ID、请求 ID、原始 URL），否则监控系统会被拖垮。

## 14.3 日志：性能与可读性的平衡

```python
import logging

logger = logging.getLogger(__name__)

# 结构化日志（JSON）便于采集，但序列化有成本
logger.info("request done", extra={"path": path, "status": status, "ms": elapsed_ms})
```

性能注意：

- 用参数化日志，避免关闭级别时仍拼字符串；
- 高频日志采样/限流；
- 用 `QueueHandler` + `QueueListener` 把日志 I/O 移出请求线程；
- 避免在热循环里打日志；
- 日志文件轮转（`RotatingFileHandler`）配置上限，防写满磁盘；
- 敏感信息脱敏（安全要求）。

```python
import logging
import logging.handlers
import queue

log_queue = queue.Queue(maxsize=10_000)
handler = logging.handlers.QueueHandler(log_queue)
root = logging.getLogger()
root.addHandler(handler)

listener = logging.handlers.QueueListener(
    log_queue,
    logging.StreamHandler(),
    respect_handler_level=True,
)
listener.start()
# ... 程序运行 ...
# listener.stop()
```

## 14.4 追踪：定位跨服务/跨依赖的慢点

- 为每个请求生成 trace id，贯穿日志/下游调用；
- 用 OpenTelemetry 自动/手动埋点；
- 采样：全量追踪开销大，生产通常按比例/尾部采样；
- 关注"自身耗时"与"下游耗时"的拆分；
- 异步任务要传递上下文（`contextvars`）。

## 14.5 生产剖析：安全第一

| 工具 | 开销 | 适用 |
| :--- | :--- | :--- |
| `py-spy top/record/dump` | 低 | 生产 CPU/卡死排查 |
| `pyinstrument` | 低-中 | 请求级采样 |
| `cProfile` | 高 | 测试/预发，短时间 |
| `line_profiler` | 高 | 预发，单函数 |
| `tracemalloc` | 中-高 | 预发/按需采样内存 |
| `sys.monitoring` | 低（可定制） | 3.12+ 自定义监控 |
| `faulthandler` | 极低 | 卡死时 dump 栈 |

**生产剖析原则**：

1. 先采样（py-spy），不要直接上 cProfile；
2. 限定时间窗口/请求比例；
3. 记录剖析期间的流量/版本，避免误判；
4. 剖析本身可能改变时序，结论要交叉验证；
5. 敏感数据/密钥不要进入剖析输出。

## 14.6 故障 playbook

### 14.6.1 CPU 100%

**排查顺序**：

1. `py-spy top --pid PID` 看哪个函数在烧 CPU；
2. `py-spy record` 生成火焰图，看调用链；
3. 检查是否是 GC（`gc.callbacks`/`gc.get_stats`）；
4. 检查是否有正则灾难性回溯、JSON 大对象序列化、压缩、加密；
5. 检查线程竞争（锁自旋/频繁切换）；
6. 检查是否被流量打满/死循环。

**常见修复**：算法/缓存、减少分配、向量化、进程池、限流、修复死循环/正则。

### 14.6.2 内存持续上涨

1. 看 RSS 曲线：是否与请求量/时间相关；
2. `tracemalloc` 快照 diff，找增长行；
3. 检查缓存/全局容器/注册表；
4. 检查异常对象/闭包/回调是否持有大对象；
5. 检查线程/连接/FD 是否泄漏；
6. 检查 C 扩展/NumPy（`memray`/RSS）；
7. 判断是泄漏还是碎片（重启是否恢复）。

**常见修复**：有界缓存、弱引用、清理注册、修复引用链、分片/定期重启。

### 14.6.3 延迟升高（p99 毛刺）

1. 区分是全局变慢还是尾延迟；
2. 看 GC 停顿、日志 I/O、锁等待、连接池等待；
3. 看下游依赖的 p99/错误率；
4. 看队列长度/线程池排队；
5. 看是否有批量任务/定时任务抢占资源；
6. 用追踪定位单请求耗时分布。

**常见修复**：调 GC、异步日志、连接池预热/扩容、超时/重试预算、隔离批量任务、限流。

### 14.6.4 FD 耗尽

1. `psutil.num_fds()` 确认趋势；
2. `/proc/<pid>/fd`/`lsof` 看类型（socket/文件/管道）；
3. 检查连接池是否归还、响应是否关闭；
4. 开发模式 `-X dev` 抓 `ResourceWarning`；
5. 临时提高 rlimit 争取时间，根本修复泄漏。

### 14.6.5 卡死/死锁

1. `py-spy dump --pid PID` 看所有线程栈；
2. `faulthandler.dump_traceback_later()` 定时 dump；
3. 检查锁顺序、线程池嵌套等待、`Future.result()` 互相等待；
4. 检查子进程管道是否写满（`PIPE` 死锁）；
5. 检查数据库/网络是否有无限等待（缺超时）。

### 14.6.6 线程/进程暴涨

1. 看线程数/进程数曲线；
2. 检查是否每请求建线程/进程；
3. 检查池是否有上限、是否复用；
4. 检查递归创建/异常重试导致泄漏；
5. 容器 `pids.max` 是否触顶。

### 14.6.7 启动慢/冷启动高

1. `-X importtime` 找大依赖；
2. 检查模块级初始化/网络调用；
3. 检查镜像大小/拉取时间；
4. 预编译/精简依赖/延迟导入；
5. Serverless 检查内存/并发配置。

## 14.7 基准回归与容量复测

- 把关键路径基准放进 CI（`pyperf`/`pytest-benchmark`），设置退化阈值；
- 记录 Python 版本、依赖版本、硬件/容器规格；
- 每次大版本升级、依赖升级、架构调整后复测；
- 压测要覆盖真实请求组合、数据规模、错误路径；
- 关注 p99 与资源曲线，而不只是平均 QPS；
- 容量规划留余量，定期演练过载/降级。

## 14.8 成本优化

| 手段 | 收益 | 注意 |
| :--- | :--- | :--- |
| 右规格（right-sizing） | 降 CPU/内存成本 | 留足余量，避免频繁 OOM |
| 批处理/合并请求 | 降单位成本 | 增加延迟 |
| 缓存 | 降下游/计算成本 | 一致性/内存成本 |
| 压缩 | 降存储/带宽 | CPU 成本 |
| 异步/池化 | 提高资源利用率 | 复杂度/尾延迟 |
| 自动扩缩容 | 按需付费 | 冷启动/抖动 |
| 选用更轻依赖 | 降启动/内存 | 功能/生态权衡 |
| 升级 Python | 免费性能 | 回归成本 |

## 14.9 本章要点与坑清单

**要点**

- 可观测性 = metrics + logs + traces；按需 profiling。
- 指标要少而准，避免高基数标签。
- 日志参数化、异步化、采样、轮转；敏感信息脱敏。
- 生产剖析优先采样工具（py-spy），cProfile 只在预发短时使用。
- 故障排查按"资源-症状-工具-修复"形成 playbook。
- 基准回归和容量复测要制度化。

**坑**

- 生产直接开 cProfile/line_profiler，拖慢服务。
- 日志里打大对象/密钥/高基数 ID。
- 指标标签带用户 ID，监控系统被打爆。
- 只看平均延迟，不看 p99。
- 故障时盲目重启，不保留现场（栈/内存快照/日志）。

---

# 第 15 章 反模式、决策树、速查表与检查清单

## 15.1 30 个性能/资源反模式

| # | 反模式 | 问题 | 正确方向 |
| :--- | :--- | :--- | :--- |
| 1 | 不测量就优化 | 优化错地方，增加复杂度 | 先 profiler/基准 |
| 2 | 用单次运行下结论 | 噪声/预热/缓存影响 | 多次重复、看分布 |
| 3 | 用 `time.time()` 测耗时 | 系统时钟可能跳变 | `perf_counter` |
| 4 | 压测等响应再发下一个 | 协调遗漏，p99 偏乐观 | 开环压测/校正 |
| 5 | 在 list 里循环 `in` | O(n²) | set/dict |
| 6 | 用 list 当队列 `pop(0)` | O(n²) | deque |
| 7 | 循环里 `s += part` | 可能 O(n²) | `"".join` |
| 8 | 大文件 `readlines()` | 内存 O(n) | 迭代/分块 |
| 9 | 每行 `flush`/`fsync` | 系统调用爆炸 | 批量/缓冲 |
| 10 | 循环里重复 `re.compile` | 重复编译/缓存查找 | 预编译 |
| 11 | 用异常做常见分支 | 抛异常很贵 | `get`/返回值 |
| 12 | 热循环里打日志（f-string） | 关闭级别也求值 | 参数化/级别判断 |
| 13 | 无界缓存 | 内存持续上涨 | 有界 LRU/TTL |
| 14 | `lru_cache` 缓存实例方法 | `self` 无法回收 | 模块级函数/弱引用 |
| 15 | 缓存可变对象并返回 | 调用方污染缓存 | 返回副本/不可变 |
| 16 | `deepcopy` 大对象 | CPU/内存翻倍 | 共享/结构化复制 |
| 17 | 对少量对象用 `__slots__` | 收益可忽略 | 先测量 |
| 18 | 无脑 `gc.disable()` | 循环垃圾堆积 | 保留 GC/按需调参 |
| 19 | 频繁 `gc.collect()` | 全量扫描停顿 | 明确时机 |
| 20 | CPU 密集开多线程 | GIL 下无法并行 | 多进程/扩展 |
| 21 | 每请求建线程/进程 | 创建/销毁成本高 | 有界池 |
| 22 | 无限制 `gather`/提交任务 | 内存/FD/下游被打爆 | 信号量/有界队列 |
| 23 | 协程里调用同步阻塞库 | 卡住事件循环 | `to_thread`/异步库 |
| 24 | 无超时 | 无限等待/级联失败 | 分层超时 |
| 25 | 用 `Future.result(timeout)` 以为能停止任务 | 任务继续跑 | 协作式取消/进程终止 |
| 26 | `Popen` 只 `wait()` 不读 `PIPE` | 管道写满死锁 | `communicate()` |
| 27 | 依赖 `__del__` 释放资源 | 时机不确定/不执行 | `with`/`finally` |
| 28 | 模块顶层建连接/加载模型 | 启动慢/副作用 | 延迟初始化 |
| 29 | 容器里按 `os.cpu_count()` 定并发 | 看到的是宿主核数 | `os.process_cpu_count()`/cgroup |
| 30 | 生产开 cProfile | 开销大、扭曲性能 | py-spy/采样 |

## 15.2 决策树合集

### 15.2.1 该不该优化

```text
有明确的性能/资源问题吗？
├── 否 → 先保证正确性/可维护性，不要提前优化
└── 是 → 有可复现的基线吗？
    ├── 否 → 先建立基线（指标、数据、环境）
    └── 是 → 热点占比大吗？（Amdahl）
        ├── 否 → 可能不值得；除非影响 p99/资源上限
        └── 是 → 进入瓶颈分类
```

### 15.2.2 CPU 瓶颈

```text
CPU 高且 wall ≈ cpu？
├── 是 → 热点在 Python 代码还是 C 扩展？
│   ├── Python → 算法/数据结构 → 内置/推导式 → 向量化 → 进程/JIT/扩展
│   └── C 扩展 → 换更高效 API/减少调用/换库/并行
└── 否 → 可能是 I/O/锁/GC，转对应章节
```

### 15.2.3 内存瓶颈

```text
RSS 高/持续上涨？
├── 是泄漏吗？（长时间不回落、重启恢复）
│   ├── 是 → tracemalloc/对象统计 → 修引用链
│   └── 否 → 峰值/碎片/缓存
│       ├── 峰值 → 流式/分块/减少拷贝
│       ├── 碎片 → 分配器/重启/隔离
│       └── 缓存 → 有界/TTL/弱引用
└── 否 → 不是内存问题
```

### 15.2.4 I/O 瓶颈

```text
wall >> cpu？
├── 是 → 文件/数据库/网络？
│   ├── 文件 → 缓冲/分块/mmap/批量
│   ├── 数据库 → 批量/事务/索引/连接池
│   └── 网络 → 连接复用/异步/压缩/就近
└── 否 → 回到 CPU/内存
```

### 15.2.5 并发模型

```text
任务主要是 CPU 还是 I/O？
├── CPU → 进程池/原生扩展/向量化（线程无效）
└── I/O → 并发量？
    ├── 低/中 → 线程池
    ├── 高（数千连接）→ asyncio
    └── 需要共享大内存 → 线程 + 锁/共享内存
```

### 15.2.6 缓存

```text
结果可复用且计算/查询昂贵？
├── 否 → 不缓存
└── 是 → 数据能接受多旧？
    ├── 强一致 → 不缓存/短 TTL + 主动失效
    ├── 秒级 → TTL + single-flight
    ├── 分钟级 → 本地/TTL/Redis
    └── 可最终一致 → 多级缓存 + 异步刷新
```

## 15.3 检查清单

### 15.3.1 设计阶段

- [ ] 明确性能目标（延迟/吞吐/资源上限），而不是"越快越好"。
- [ ] 估算数据规模与增长趋势，选择合适算法/存储。
- [ ] 明确并发模型与下游承载能力。
- [ ] 为缓存/队列/连接池设定上限与失效策略。
- [ ] 设计超时、重试、熔断、降级、背压。
- [ ] 规划可观测性（指标/日志/追踪）。

### 15.3.2 编码阶段

- [ ] 热路径避免 O(n²)、重复计算、无界内存。
- [ ] 文件/socket/连接/锁用 `with` 或 `finally`。
- [ ] 子进程用 `communicate()` + 超时。
- [ ] 日志参数化，不在热循环里打昂贵日志。
- [ ] 异常不用于常见控制流。
- [ ] 缓存键可哈希、结果不可变/不共享可变状态。
- [ ] 线程/进程池有界，任务可取消/可超时。

### 15.3.3 代码评审

- [ ] 新增缓存是否有上限/失效/击穿保护？
- [ ] 新增线程/进程/连接是否有配额？
- [ ] 新增 I/O 是否设置超时？
- [ ] 新增循环是否可能 O(n²)？
- [ ] 新增全局/类级容器是否可能无限增长？
- [ ] 是否正确处理异常路径的资源释放？
- [ ] 是否有对应的基准/压测/监控？

### 15.3.4 上线前

- [ ] 压测覆盖真实请求组合与数据规模。
- [ ] 观察 p99、错误率、CPU、RSS、FD、线程、GC。
- [ ] 配置资源限额（容器/rlimit/池大小）。
- [ ] 配置超时/重试/熔断/限流。
- [ ] 准备降级/回滚方案。
- [ ] 确认日志/追踪不会拖垮性能或泄露敏感信息。

### 15.3.5 线上巡检

- [ ] CPU/内存/FD/线程曲线是否平稳？
- [ ] GC 次数/停顿是否影响 p99？
- [ ] 缓存命中率/内存是否合理？
- [ ] 连接池等待/超时是否上升？
- [ ] 队列是否持续增长？
- [ ] 错误率/重试次数是否异常？
- [ ] 依赖版本/解释器版本是否有已知性能问题？

## 15.4 默认值与常用 API 速查

### 15.4.1 运行时默认值（本机 CPython 3.13.5，仅作参照）

| 项 | 值/接口 | 说明 |
| :--- | :--- | :--- |
| 线程切换间隔 | `sys.getswitchinterval()` → `0.005` | 5ms |
| GC 阈值 | `gc.get_threshold()` → `(2000, 10, 10)` | 3.12 为 `(700, 10, 10)` |
| 默认缓冲 | `io.DEFAULT_BUFFER_SIZE` → `8192` | 文本/二进制默认 |
| 递归限制 | `sys.getrecursionlimit()` → `1000` | 不是资源管理万能药 |
| int 转 str 位数 | `sys.get_int_max_str_digits()` → `4300` | 3.11+ 安全限制 |
| 线程池默认 | `min(32, (os.process_cpu_count() or 1) + 4)` | 3.13；3.8–3.12 用 `os.cpu_count()` |
| 进程池默认 | `os.process_cpu_count()` | 3.13；Windows 上限 61 |
| 空 list 大小 | `sys.getsizeof([])` → `56` | 仅对象本身 |
| 空 dict 大小 | `sys.getsizeof({})` → `64` | 仅对象本身 |
| 空 str 大小 | `sys.getsizeof("")` → `41` | 3.12/3.13 本机 |

> 这些数字会随版本/平台/编译选项变化，**只用于建立量级直觉**，生产判断以运行时读取为准。

### 15.4.2 常用接口

| 目的 | 接口 |
| :--- | :--- |
| 墙钟计时 | `time.perf_counter()` |
| CPU 计时 | `time.process_time()` |
| 微基准 | `timeit.timeit`、`python -m timeit` |
| 函数剖析 | `cProfile` + `pstats` |
| 采样剖析 | `py-spy`、`pyinstrument` |
| 内存跟踪 | `tracemalloc` |
| 进程内存 | `psutil.Process().memory_info()` |
| GC | `gc.get_stats()`、`gc.collect()`、`gc.freeze()` |
| 线程/进程池 | `concurrent.futures` |
| 异步 | `asyncio`、`TaskGroup`、`asyncio.timeout` |
| 子进程 | `subprocess.run`、`Popen.communicate` |
| 资源限制 | `resource.setrlimit`（Unix） |
| 卡死诊断 | `faulthandler.dump_traceback_later` |
| 导入耗时 | `-X importtime` |
| 开发模式 | `-X dev` |
| 零拷贝 | `memoryview`、`mmap`、`socket.sendfile` |

## 15.5 学习路径（建议 4 周）

| 阶段 | 目标 | 行动 |
| :--- | :--- | :--- |
| 第 1 周 | 建立测量习惯 | 读第 1–2 章；用 `timeit`/`cProfile`/`tracemalloc` 分析自己的一个小项目 |
| 第 2 周 | 掌握计算优化 | 读第 3–7 章；把一个 Python 循环改成向量化/内置函数；做题库第二部分 |
| 第 3 周 | 掌握内存与 I/O | 读第 8–9 章；用 `tracemalloc` 找一次内存增长；用分块改写大文件处理 |
| 第 4 周 | 掌握并发与资源治理 | 读第 10–12 章；给一个服务加超时/限流/背压；读第 13–15 章并做完整题库 |

**练习建议**：

1. 选一个你熟悉的脚本，记录基线（耗时、RSS、FD）；
2. 用 profiler 找前 3 个热点；
3. 只做收益最大的 1–2 个改动；
4. 复测并记录；
5. 写出"为什么快/为什么没快"的结论——这比记住技巧更重要。

## 15.6 最后一句话

**性能优化的终点不是"用了多少技巧"，而是"用最少的复杂度，把资源花在真正影响目标的地方"。** 先测量、再优化、设上限、留余量、可观测——这五件事做到，Python 的性能与资源管理就掌握了 80%。

---

# 附录 A 版本差异速查

| 版本 | 与性能/资源相关的特性 | 备注 |
| :--- | :--- | :--- |
| 3.8 | `functools.cached_property`；`multiprocessing.shared_memory`；pickle protocol 5 | 共享内存/零拷贝基础 |
| 3.9 | `functools.cache`；`asyncio.to_thread`；dict `\|`/`\|=`；`str.removeprefix/removesuffix` | 缓存/线程卸载更方便 |
| 3.10 | `dataclass(slots=True)`；`contextlib.aclosing`；`itertools.pairwise`；`zip(strict=True)` | 省内存/资源管理/遍历 |
| 3.11 | PEP 659 自适应解释器；零成本异常；`TaskGroup`；`asyncio.timeout`；`hashlib.file_digest`；`ExceptionGroup` | 纯 Python 提速、结构化并发 |
| 3.12 | PEP 709 推导式内联；`sys.monitoring`；`itertools.batched`；类型参数语法 | 推导式更快、低开销监控 |
| 3.13 | 实验性 JIT；free-threaded 构建；`os.process_cpu_count()`；`sys._is_gil_enabled()`；GC 阈值调整；`PythonFinalizationError` | 实验特性谨慎上生产 |
| 3.14+ | 持续演进（free-threading/GC/JIT 等） | 以 release notes 与实测为准 |

---

# 附录 B 常用命令与代码片段

## B.1 命令行

```bash
# 微基准
python -m timeit -s "data = list(range(1000))" "sum(data)"

# 函数级剖析
python -m cProfile -o profile.prof app.py
python -m pstats profile.prof

# 导入耗时
python -X importtime -c "import app"

# 开发模式（ResourceWarning/调试分配器等）
python -X dev app.py

# 预编译
python -m compileall -q app/

# 采样剖析（需安装 py-spy）
py-spy top --pid 12345
py-spy record -o flame.svg --pid 12345 --duration 30
py-spy dump --pid 12345
```

## B.2 计时与剖析模板

```python
import cProfile
import pstats
import time

def timed(label, fn, *args, **kwargs):
    start = time.perf_counter()
    result = fn(*args, **kwargs)
    elapsed = time.perf_counter() - start
    print(f"{label}: {elapsed:.6f}s")
    return result

def profile(fn, *args, **kwargs):
    profiler = cProfile.Profile()
    profiler.enable()
    try:
        return fn(*args, **kwargs)
    finally:
        profiler.disable()
        pstats.Stats(profiler).sort_stats("cumulative").print_stats(20)
```

## B.3 内存快照模板

```python
import tracemalloc

def diff_snapshots(before, after, limit=10):
    stats = after.compare_to(before, "lineno")
    for stat in stats[:limit]:
        print(stat)

tracemalloc.start(25)
base = tracemalloc.take_snapshot()
# ... run workload ...
cur = tracemalloc.take_snapshot()
diff_snapshots(base, cur)
print("traced:", tracemalloc.get_traced_memory())
tracemalloc.stop()
```

## B.4 资源监控模板

```python
import os
import psutil

proc = psutil.Process(os.getpid())

def snapshot():
    mem = proc.memory_info()
    return {
        "rss_mib": round(mem.rss / 1024 / 1024, 1),
        "vms_mib": round(mem.vms / 1024 / 1024, 1),
        "threads": proc.num_threads(),
        "fds": proc.num_fds() if hasattr(proc, "num_fds") else None,
        "cpu_percent": proc.cpu_percent(interval=0.1),
    }

print(snapshot())
```

---

# 附录 C 参考资料

## C.1 Python 官方文档

- [timeit](https://docs.python.org/3/library/timeit.html)
- [cProfile / profile](https://docs.python.org/3/library/profile.html)
- [tracemalloc](https://docs.python.org/3/library/tracemalloc.html)
- [gc](https://docs.python.org/3/library/gc.html)
- [sys](https://docs.python.org/3/library/sys.html)
- [resource](https://docs.python.org/3/library/resource.html)
- [concurrent.futures](https://docs.python.org/3/library/concurrent.futures.html)
- [asyncio](https://docs.python.org/3/library/asyncio.html)
- [multiprocessing](https://docs.python.org/3/library/multiprocessing.html)
- [subprocess](https://docs.python.org/3/library/subprocess.html)
- [mmap](https://docs.python.org/3/library/mmap.html)
- [sqlite3](https://docs.python.org/3/library/sqlite3.html)
- [functools](https://docs.python.org/3/library/functools.html)
- [Python 3.13 更新说明](https://docs.python.org/3/whatsnew/3.13.html)
- [Python 性能分析 HOWTO](https://docs.python.org/3/howto/profile.html)

## C.2 PEP 与设计文档

- [PEP 659 – Specializing Adaptive Interpreter](https://peps.python.org/pep-0659/)
- [PEP 669 – Low Impact Monitoring](https://peps.python.org/pep-0669/)
- [PEP 703 – Making the Global Interpreter Lock Optional](https://peps.python.org/pep-0703/)
- [PEP 709 – Inlined comprehensions](https://peps.python.org/pep-0709/)
- [PEP 744 – JIT Compilation](https://peps.python.org/pep-0744/)
- [CPython 垃圾回收设计文档](https://github.com/python/cpython/blob/main/InternalDocs/garbage_collector.md)

## C.3 工具文档

- [py-spy](https://github.com/benfred/py-spy)
- [pyperf](https://github.com/psf/pyperf)
- [pytest-benchmark](https://pytest-benchmark.readthedocs.io/)
- [memory_profiler](https://github.com/pythonprofilers/memory_profiler)
- [memray](https://github.com/bloomberg/memray)
- [scalene](https://github.com/plasma-umass/scalene)
- [pyinstrument](https://github.com/joerick/pyinstrument)
- [psutil](https://psutil.readthedocs.io/)
- [NumPy 性能建议](https://numpy.org/doc/stable/user/performance.html)
- [Cython 文档](https://cython.readthedocs.io/)
- [Numba 文档](https://numba.readthedocs.io/)
- [PyO3 文档](https://pyo3.rs/)

---

> **配套题库**：同目录《Python性能与资源管理选择题_含答案解析.md》，共 80 题，覆盖本文全部章节。建议按"读一章 → 做对应题目 → 回看坑清单"的节奏使用。

