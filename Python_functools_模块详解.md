# Python functools 模块详解与实战

> **适用版本**：Python 3.9 ~ 3.14（各特性引入/变更版本在文中标注）
> **验证环境**：CPython 3.13.5（Windows），文中示例均已实际运行验证；3.14+ 特性单独标注。
> **说明**：`functools.partial` 常被误拼为 `functools.partical`，正确拼写是 **partial**（意为"部分的"）。

---

## 目录

1. [模块概述](#1-模块概述)
2. [元数据包装：wraps 与 update_wrapper](#2-元数据包装wraps-与-update_wrapper)
3. [函数缓存：lru_cache、cache、cached_property](#3-函数缓存lru_cachecachecached_property)
4. [比较与单分派：total_ordering、singledispatch、singledispatchmethod](#4-比较与单分派total_orderingsingledispatchsingledispatchmethod)
5. [函数适配核心：functools.partial（重点）](#5-函数适配核心functoolspartial重点)
6. [方法版偏函数：functools.partialmethod](#6-方法版偏函数functoolspartialmethod)
7. [折叠聚合：functools.reduce](#7-折叠聚合functoolsreduce)
8. [比较函数转 key：functools.cmp_to_key](#8-比较函数转-keyfunctoolscmp_to_key)
9. [综合实战案例](#9-综合实战案例)
10. [版本变更速查](#10-版本变更速查)
11. [最佳实践与常见陷阱](#11-最佳实践与常见陷阱)
12. [API 速查表](#12-api-速查表)
13. [参考资料](#13-参考资料)

---

## 1. 模块概述

`functools` 是 Python 标准库中处理**高阶函数**（higher-order functions）的模块：里面的工具要么**接收函数作为参数**，要么**返回新的可调用对象**。官方定义是"为高阶函数服务：作用于或返回其他函数的函数。一般而言，任何可调用对象都可以当作函数来使用"。

它解决的问题可以归为五类：

| 分类 | API | 解决什么问题 |
| --- | --- | --- |
| 元数据保留 | `wraps`、`update_wrapper` | 装饰器包装后丢失函数名、文档、签名 |
| 结果缓存 | `lru_cache`、`cache`、`cached_property` | 重复计算开销大 |
| 比较与分派 | `total_ordering`、`singledispatch`、`singledispatchmethod` | 少写样板比较方法、消除长串 `isinstance` 分支 |
| 参数适配 | `partial`、`partialmethod` | 冻结部分参数，生成签名更简单的可调用对象 |
| 折叠与排序 | `reduce`、`cmp_to_key` | 把序列折叠成单值、把旧式比较函数接入 `key=` 接口 |

### 1.1 模块清单（按引入版本）

| API | 类别 | 作用 | 引入版本 |
| --- | --- | --- | --- |
| `partial` | 函数 | 偏函数应用，冻结部分参数/关键字 | 2.5 |
| `partialmethod` | 描述符 | 类中方法版的 `partial` | 3.4 |
| `Placeholder` | 哨兵对象 | `partial` / `partialmethod` 的位置参数占位符 | **3.14** |
| `reduce` | 函数 | 从左到右折叠序列 | 3.0（从内置函数移入） |
| `wraps` | 装饰器 | 复制被包装函数的元数据 | 2.5 |
| `update_wrapper` | 函数 | `wraps` 的底层实现 | 2.5 |
| `lru_cache` | 装饰器 | LRU 缓存 | 3.2 |
| `cache` | 装饰器 | 无容量上限的缓存 | **3.9** |
| `cached_property` | 描述符 | 把方法变成只计算一次的属性 | **3.8** |
| `total_ordering` | 类装饰器 | 补齐全部富比较方法 | 3.2 |
| `singledispatch` | 装饰器 | 单分派泛型函数（按第一个参数类型分派） | 3.4 |
| `singledispatchmethod` | 装饰器 | 方法版单分派 | **3.8** |
| `cmp_to_key` | 函数 | 旧式比较函数 → `key` 函数 | 3.2 |

### 1.2 导入方式

```python
import functools

from functools import (
    partial, partialmethod, reduce,
    wraps, update_wrapper,
    lru_cache, cache, cached_property,
    total_ordering, singledispatch, singledispatchmethod,
    cmp_to_key,
)
```

> 💡 `reduce` 在 Python 3 中已不是内置函数，必须从 `functools` 导入。`functools` 的实现有 C 加速版本（`_functools`），`partial`、`reduce`、`lru_cache` 等性能都很好。

---

## 2. 元数据包装：wraps 与 update_wrapper

### 2.1 为什么需要它

装饰器本质是"用新函数替换旧函数"。如果不做处理，被装饰函数的 `__name__`、`__doc__`、`__module__`、`__annotations__`、签名都会变成包装函数的：

```python
def timer(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@timer
def add(a: int, b: int = 1) -> int:
    """两数相加"""
    return a + b

print(add.__name__)        # wrapper   ← 不是 add
print(add.__doc__)         # None      ← 文档丢了
print(add.__annotations__) # {}        ← 注解丢了
```

这会带来一连串问题：日志里函数名全是 `wrapper`、`help()` 看不到文档、`inspect.signature()` 显示 `(*args, **kwargs)`、Sphinx/pydoc 文档生成错乱、框架（如 FastAPI、pytest、click）依赖签名做参数解析时直接报错。

### 2.2 wraps 做了什么

`functools.wraps(wrapped)` 是 `functools.update_wrapper()` 的便捷形式，默认复制两组内容：

| 常量 | 默认值 | 复制方式 |
| --- | --- | --- |
| `WRAPPER_ASSIGNMENTS` | `('__module__', '__name__', '__qualname__', '__annotations__', '__type_params__', '__doc__')` | 直接赋值 |
| `WRAPPER_UPDATES` | `('__dict__',)` | 用 `dict.update()` 更新 |

此外它还会设置 `wrapper.__wrapped__ = wrapped`，这是 `inspect` 模块识别"包装关系"的约定。

```python
import functools

def timer(func):
    @functools.wraps(func)          # 关键一行
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@timer
def add(a: int, b: int = 1) -> int:
    """两数相加"""
    return a + b

print(add.__name__)          # add
print(add.__doc__)           # 两数相加
print(add.__annotations__)   # {'a': <class 'int'>, 'b': <class 'int'>, 'return': <class 'int'>}
print(add.__wrapped__)       # <function add at 0x...>
print(add(2))                # 3
```

### 2.3 签名保留与 __wrapped__

`inspect.signature()` 默认会顺着 `__wrapped__` 链查找真实签名，所以即使包装函数写的是 `*args, **kwargs`，对外也显示原签名：

```python
import inspect

print(inspect.signature(add))        # (a: int, b: int = 1) -> int
print(inspect.unwrap(add).__name__)  # add（剥掉所有包装层）
```

`__wrapped__` 还被大量库用作"绕过包装"的入口：例如访问 `lru_cache` 包装的原函数、让文档工具找到原始函数。

### 2.4 update_wrapper 的精细控制

需要只复制部分属性时，直接用 `update_wrapper`：

```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    functools.update_wrapper(
        wrapper,
        func,
        assigned=("__name__", "__doc__"),  # 只复制名字和文档
        updated=(),                        # 不合并 __dict__
    )
    return wrapper
```

### 2.5 使用要点

1. **每个装饰器层都要写 `@functools.wraps(func)`**；装饰器叠加时，每层各自包装。
2. `wraps` 不复制 `__defaults__`、`__kwdefaults__`、`__code__`，这些由 `__wrapped__` + `inspect` 机制兜底。
3. `wraps` 也能用于 `partial` 对象等带 `__dict__` 的可调用对象，但注意它会把 `__wrapped__` 指向原函数，从而影响 `inspect.signature` 的结果。
4. 自定义可调用类做装饰器时，用 `update_wrapper(self, func)` 更新实例属性，并重新定义 `__call__`。

---

## 3. 函数缓存：lru_cache、cache、cached_property

### 3.1 lru_cache

`lru_cache` 用"最近最少使用（LRU）"策略缓存函数结果，适合**纯函数**（相同输入永远产生相同输出）以及 I/O 密集、计算昂贵的场景。

#### 3.1.1 基本用法

```python
from functools import lru_cache

@lru_cache(maxsize=128)      # 等价于 @lru_cache()，默认 maxsize=128
def fib(n: int) -> int:
    if n < 2:
        return n
    return fib(n - 1) + fib(n - 2)

print([fib(n) for n in range(10)])
# [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

print(fib.cache_info())
# CacheInfo(hits=..., misses=10, maxsize=128, currsize=10)
```

自 Python 3.8 起，`@lru_cache` 可以不写括号直接使用：`@lru_cache` 等价于 `@lru_cache(maxsize=128)`。

#### 3.1.2 参数详解

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `maxsize` | `128` | 最多缓存条目数；`None` 表示无上限（不淘汰）；`0` 表示不缓存（实测每次调用都是 miss） |
| `typed` | `False` | 为 `True` 时，参数类型不同视为不同调用（如 `1` 与 `1.0` 分开缓存） |
| `user_function` | 无 | 直接传入被装饰函数，支持无括号写法（3.8+） |

选择 `maxsize` 的经验：

- 结果集小且固定（配置、字典查表）→ `None`；
- 结果集可能无限增长（用户 ID、URL）→ 设一个有限值，防止内存泄漏；
- 关注命中率时，用 `cache_info()` 观察 `hits / (hits + misses)`，再调整 `maxsize`。

#### 3.1.3 缓存诊断：cache_info / cache_clear / cache_parameters

```python
@lru_cache(maxsize=2)
def double(x):
    return x * 2

double(1); double(2); double(1)
print(double.cache_info())       # CacheInfo(hits=1, misses=2, maxsize=2, currsize=2)
print(double.cache_parameters()) # {'maxsize': 2, 'typed': False}
double.cache_clear()
print(double.cache_info())       # CacheInfo(hits=0, misses=0, maxsize=2, currsize=0)
```

- `cache_info()` → `CacheInfo(hits, misses, maxsize, currsize)` 具名元组；
- `cache_clear()` → 清空缓存（**测试、配置热更新、手动失效时必须调用**）；
- `cache_parameters()`（3.9+）→ 返回 `maxsize`、`typed` 的字典副本，仅供查看；
- `__wrapped__` → 原始未缓存函数，可用于绕过缓存或重新包装。

#### 3.1.4 缓存键的语义（重要）

`lru_cache` 的缓存键由"位置参数元组 + 关键字参数顺序 + 类型标记"构成，因此：

```python
@lru_cache(maxsize=None)
def f(a=1, b=2):
    return object()

f(a=1, b=2) is f(b=2, a=1)   # False —— 关键字顺序不同，视为不同调用
f(1, 2) is f(a=1, b=2)       # False —— 位置参数与关键字参数形式不同
```

`typed` 的细节：

```python
@lru_cache(maxsize=None)
def g(x, y):
    return object()

g(1, 1) is g(1.0, 1.0)       # True  —— typed=False，1 与 1.0 哈希相等、值相等，合并缓存

@lru_cache(maxsize=None, typed=True)
def h(x, y):
    return object()

h(1, 1) is h(1.0, 1.0)       # False —— typed=True，按类型区分
```

> 官方文档提示：`typed=False` 时"通常"把不同类型视为等价，但个别类型（如 `str` 与 `int`）仍可能分开缓存，不要依赖这个边界行为。

#### 3.1.5 递归与动态规划

`lru_cache` 把指数级递归降为线性：

```python
@lru_cache(maxsize=None)
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)

print(fib(100))  # 354224848179261915075，几乎瞬间完成
```

#### 3.1.6 在实例方法上使用（常见坑）

```python
class Service:
    @lru_cache(maxsize=128)
    def get(self, key):
        return self._load(key)
```

这样写有两个问题：

1. `self` 也是缓存键的一部分，每个实例独立占用缓存，实例多时内存膨胀；
2. 缓存会**强引用 `self`**，实例无法被垃圾回收（缓存不清理就一直泄漏）。

推荐写法：

```python
# 方案一：缓存模块级纯函数，方法只做转发
@lru_cache(maxsize=1024)
def _load_user(user_id: int):
    ...

class UserService:
    def get_user(self, user_id: int):
        return _load_user(user_id)


# 方案二：缓存"与实例无关"的类方法（cls 同样会被缓存，通常无害）
class UserService:
    @classmethod
    @lru_cache(maxsize=1024)
    def load(cls, user_id):
        ...


# 方案三：实例级缓存（需要随实例销毁而销毁时）
class UserService:
    def __init__(self):
        self._cache = {}

    def get_user(self, user_id):
        if user_id not in self._cache:
            self._cache[user_id] = self._load(user_id)
        return self._cache[user_id]


# 方案四：无参派生属性用 cached_property
```

#### 3.1.7 线程安全与副作用

- 缓存结构本身线程安全（不会因并发写坏字典）；
- 但**函数体没有加锁**：两个线程同时未命中同一参数时，函数可能被并发执行多次，后完成的结果覆盖先完成的；
- 因此不要缓存有副作用、返回可变对象、依赖时间/随机数/I-O 状态的函数；
- 缓存会**一直持有参数和返回值引用**，直到被淘汰或 `cache_clear()`。缓存大对象（如 DataFrame、图片）时尤其要控制 `maxsize`。

> ⚠️ 生成器函数、`async def` 协程函数返回的都是"惰性对象/协程"，缓存它们通常没有意义甚至有害。

### 3.2 cache

Python 3.9 新增，等价于 `lru_cache(maxsize=None)`，但因为没有淘汰逻辑，速度更快、内存占用更小：

```python
from functools import cache

@cache
def factorial(n):
    return n * factorial(n - 1) if n else 1

print(factorial(10))  # 3628800
print(factorial.cache_info())  # CacheInfo(hits=..., misses=11, maxsize=None, currsize=11)
```

只要确定缓存条目数量可控，就优先用 `@cache`；需要限制内存时用 `@lru_cache(maxsize=N)`。

### 3.3 cached_property

`cached_property`（3.8+）把"无参方法"变成**首次访问时计算、之后作为普通属性返回**的属性：

```python
import functools

class DataSet:
    def __init__(self, seq):
        self._data = tuple(seq)

    @functools.cached_property
    def total(self):
        print("  (只在第一次访问时计算)")
        return sum(self._data)

d = DataSet([1, 2, 3])
print(d.total)   # 打印计算提示，输出 6
print(d.total)   # 直接返回 6，不再计算
print(d.__dict__)  # {'_data': (1, 2, 3), 'total': 6}
```

#### 3.3.1 与 property、lru_cache 的对比

| 特性 | `@property` | `@cached_property` | `@lru_cache` 修饰方法 |
| --- | --- | --- | --- |
| 计算次数 | 每次访问都计算 | 每个实例只算一次 | 按参数缓存 |
| 存放位置 | 无 | 实例 `__dict__`（同名属性） | 函数级缓存 |
| 可否赋值 | 默认禁止（无 setter） | 可以，赋值后覆盖缓存 | 不涉及 |
| 失效方式 | 无 | `del obj.attr` | `cache_clear()` 全清 |
| 参数 | 可以有 | 只能 `self` | 任意参数 |
| 内存 | 无额外 | 占用实例字典（破坏 PEP 412 key-sharing） | 全局缓存，强引用 self |
| 要求 | 无 | 实例必须有可写 `__dict__` | 参数可哈希 |
| 线程安全 | — | 不保证只算一次（3.12 起移除了内部锁） | 结构安全，函数可能并发执行 |

#### 3.3.2 关键行为

```python
d = DataSet([1, 2, 3])
print(d.total)      # 6
del d.total         # 删除缓存属性
print(d.total)      # 重新计算，仍为 6

d.total = 100       # 允许赋值，覆盖缓存
print(d.total)      # 100
```

注意：`cached_property` 是**非数据描述符**，所以实例属性赋值优先于它；类上仍然能看到描述符对象：

```python
print(type(DataSet.total))  # <class 'functools.cached_property'>
```

#### 3.3.3 限制与坑

1. **实例必须有可写 `__dict__`**：`__slots__` 类、元类实例会报 `TypeError: No '__dict__' attribute ...`；
2. 破坏 PEP 412 的 key-sharing 字典，大量小对象时内存可能变大；
3. 3.12 起移除内部锁，多线程下 getter 可能执行多次（要求幂等）；
4. 只适合"实例创建后基本不变"的派生属性；频繁变更的数据请手动管理缓存；
5. 不支持 `__set_name__` 之外的动态改名——每个属性名必须唯一。

### 3.4 缓存选型速查

| 场景 | 推荐 |
| --- | --- |
| 纯函数、参数可哈希、结果集有限 | `@cache` |
| 同上但需要限制内存 | `@lru_cache(maxsize=N)` |
| 实例的无参派生属性 | `@cached_property` |
| 实例方法带参数、需要独立缓存 | 实例内字典 / 模块级缓存函数 |
| 需要 TTL（过期时间） | 标准库没有，用 `cachetools.TTLCache` 或自己实现 |

---

## 4. 比较与单分派：total_ordering、singledispatch、singledispatchmethod

### 4.1 total_ordering

Python 的富比较有 6 个方法：`__eq__`、`__ne__`、`__lt__`、`__le__`、`__gt__`、`__ge__`。`@total_ordering` 只需你实现 `__eq__` 加任意一个有序比较，就能自动补齐其余方法。

```python
import functools

@functools.total_ordering
class Version:
    def __init__(self, major, minor):
        self.major, self.minor = major, minor

    def __eq__(self, other):
        if not isinstance(other, Version):
            return NotImplemented
        return (self.major, self.minor) == (other.major, other.minor)

    def __lt__(self, other):
        if not isinstance(other, Version):
            return NotImplemented
        return (self.major, self.minor) < (other.major, other.minor)


print(Version(1, 2) < Version(1, 10))   # True
print(Version(2, 0) > Version(1, 9))    # True
print(Version(1, 2) <= Version(1, 2))   # True
print(Version(1, 2) >= Version(1, 2))   # True
print(sorted([Version(2, 0), Version(1, 10)]))
```

要点：

- 必须实现 `__eq__` 和 `__lt__/__le__/__gt__/__ge__` 中的**至少一个**；
- 返回 `NotImplemented` 才能正确触发 Python 的反射比较与 `TypeError`（3.4+ 支持）；
- 只补齐"类及其父类都没有定义"的方法，不会覆盖已有实现；
- 有性能代价（派生方法内部还要调用你实现的方法），热点路径可手写全部 6 个方法。

### 4.2 singledispatch

`@singledispatch`（3.4+）把"按第一个参数类型分派"的 `if/elif isinstance` 链改写为可扩展的泛型函数。新增类型只需 `register`，无需改原函数，符合开闭原则。

```python
from functools import singledispatch

@singledispatch
def describe(obj):
    """默认实现：相当于 object 类型"""
    return f"未知对象: {obj!r}"

@describe.register
def _(obj: int):
    return f"整数: {obj}"

@describe.register
def _(obj: str):
    return f"字符串: {obj!r}"

@describe.register(list)
def _(obj):
    return f"列表，长度 {len(obj)}"

print(describe(42))        # 整数: 42
print(describe("hi"))      # 字符串: 'hi'
print(describe([1, 2, 3])) # 列表，长度 3
print(describe((1, 2)))    # 未知对象: (1, 2)  ← 走默认实现
```

#### 4.2.1 注册方式

```python
# 1) 注解推断类型
@describe.register
def _(obj: int): ...

# 2) 显式传类型（适合 lambda、已有函数、无注解代码）
def handle_complex(obj):
    return "复数"
describe.register(complex, handle_complex)

# 3) 函数式调用（返回值是未装饰的原函数，便于单独测试/pickle）
def handle_bytes(obj):
    return "字节"
returned = describe.register(bytes)(handle_bytes)
print(returned is handle_bytes)  # True
```

#### 4.2.2 分派规则

- 按**第一个参数**的运行时类型，沿 MRO 找最近注册的实现；
- 默认实现注册在 `object` 上，找不到时使用；
- 注册**抽象基类（ABC）**时，虚拟子类也会命中：

```python
from collections.abc import Mapping

@describe.register
def _(obj: Mapping):
    return f"映射，{len(obj)} 个键"

print(describe({"a": 1}))  # 映射，1 个键
```

- 子类会命中父类实现，例如 `bool` 是 `int` 的子类，`describe(True)` 命中 `int` 分支；
- 3.11+ 支持 `typing.Union` / `int | float` 注解：

```python
@describe.register
def _(obj: int | float):
    return "数字"
```

#### 4.2.3 内省接口

| 属性 | 作用 |
| --- | --- |
| `describe.registry` | 只读映射：类型 → 实现函数 |
| `describe.dispatch(T)` | 查询类型 `T` 会分派到哪个实现 |
| `describe.register` | 注册新实现（可作为装饰器） |

```python
print(describe.dispatch(str).__name__)   # 具体实现函数名
print(list(describe.registry))           # [<class 'object'>, <class 'int'>, ...]
```

#### 4.2.4 局限

1. 只按**第一个参数**分派，多参数联合分派需要第三方库（如 `multipledispatch`）或手动实现；
2. 不支持 `list[int]` 这类参数化泛型的运行时分派——请注册 `list`，泛型注解只对类型检查器有意义；
3. 重复注册同一类型会覆盖旧实现；
4. 对性能极敏感的热点代码，`singledispatch` 有额外查找开销；普通业务代码可放心使用。

### 4.3 singledispatchmethod

`@singledispatchmethod`（3.8+）是方法版：**跳过 `self`/`cls`**，按第一个实际参数分派。

```python
from functools import singledispatchmethod

class Negator:
    @singledispatchmethod
    def neg(self, arg):
        raise NotImplementedError(f"不支持的类型: {type(arg).__name__}")

    @neg.register
    def _(self, arg: int):
        return -arg

    @neg.register
    def _(self, arg: bool):
        return not arg

n = Negator()
print(n.neg(5))     # -5
print(n.neg(True))  # False
```

与其他方法装饰器组合时，`singledispatchmethod` 必须是**最外层**：

```python
class Negator:
    @singledispatchmethod
    @classmethod
    def neg(cls, arg):
        raise NotImplementedError

    @neg.register
    @classmethod
    def _(cls, arg: int):
        return -arg

print(Negator.neg(5))  # -5
```

---

## 5. 函数适配核心：functools.partial（重点）

### 5.1 它是什么

`functools.partial(func, *args, **keywords)` 返回一个**新的可调用对象**（partial 对象），调用它时相当于：

```
func(*已冻结的位置参数, *本次调用传入的位置参数, **合并后的关键字参数)
```

也就是"**偏函数应用**"（partial application）：把函数的一部分参数先固定下来，得到签名更简单的可调用对象。

> 偏函数应用 ≠ 柯里化（currying）：柯里化是"每次只接收一个参数，逐层返回新函数"；`partial` 可以一次冻结任意多个参数，且调用时仍可继续传参。

### 5.2 最小示例

```python
from functools import partial

# 经典用法：把 int() 的 base 固定为 2
basetwo = partial(int, base=2)
basetwo.__doc__ = "把二进制字符串转成整数"
print(basetwo("10010"))  # 18

# 冻结位置参数
def power(base, exp):
    return base ** exp

square = partial(power, exp=2)   # exp 是普通参数，可用关键字冻结
cube = partial(power, exp=3)
print(square(5), cube(2))        # 25 8

# 回调：固定上下文参数
def on_click(button_id, event):
    print(f"按钮 {button_id} 被点击: {event}")

handler = partial(on_click, 3)
handler("left-click")  # 按钮 3 被点击: left-click
```

### 5.3 内部结构：三个只读属性

```python
p = partial(power, 2)
print(p.func)      # <function power at 0x...>
print(p.args)      # (2,)
print(p.keywords)  # {}
```

| 属性 | 含义 | 可写性 |
| --- | --- | --- |
| `p.func` | 被包装的可调用对象 | 只读（赋值抛 `AttributeError: readonly attribute`） |
| `p.args` | 冻结的位置参数元组 | 只读 |
| `p.keywords` | 冻结的关键字参数字典 | 属性只读，但**字典本身可变** |

```python
p = partial(int, base=2)
p.keywords["base"] = 8   # 可以改字典内容（不推荐，容易造成隐蔽 bug）
print(p("10"))           # 8
```

partial 对象还支持 `__dict__`（可挂自定义属性）和弱引用，并且可哈希——但**按对象身份比较**，不要用两个"内容相同"的 partial 做等值查找：

```python
p1, p2 = partial(pow, 2), partial(pow, 2)
print(p1 == p2)          # False
print(hash(p1) == hash(p2))  # False（默认对象哈希）
```

### 5.4 参数合并规则

记住三条规则即可：

| 规则 | 说明 |
| --- | --- |
| 位置参数追加 | 新调用的位置参数追加在冻结位置参数**之后** |
| 关键字覆盖 | 新调用的关键字参数**覆盖**同名冻结关键字 |
| 位置/关键字冲突 | 同一参数既被位置传入又被关键字传入 → `TypeError: multiple values` |

```python
def f(a, b, c=0, *, d=0):
    return (a, b, c, d)

p = partial(f, 1)          # 冻结 a=1
print(p(2))                # (1, 2, 0, 0)
print(p(2, 3))             # (1, 2, 3, 0)
print(p(2, c=3))           # (1, 2, 3, 0)
print(p(2, d=9))           # (1, 2, 0, 9)

q = partial(f, 1, c=10)
print(q(2))                # (1, 2, 10, 0)
print(q(2, c=20))          # (1, 2, 20, 0) ← 调用时的关键字覆盖冻结值

partial(f, 1)(a=2)         # TypeError: f() got multiple values for argument 'a'
```

### 5.5 等价实现（帮助理解）

官方文档给出的近似实现：

```python
def partial(func, /, *args, **keywords):
    def newfunc(*more_args, **more_keywords):
        return func(*args, *more_args, **{**keywords, **more_keywords})
    newfunc.func = func
    newfunc.args = args
    newfunc.keywords = keywords
    return newfunc
```

真实实现是 C 版本（`_functools.partial`），额外提供：只读属性、`__dict__`、弱引用、pickle 支持、`inspect.signature` 识别、嵌套扁平化。

### 5.6 嵌套 partial 会自动扁平化

```python
p1 = partial(f, 1)
p2 = partial(p1, 2, d=5)
print(p2.func.__name__)  # f     ← 自动展开到最内层函数
print(p2.args)           # (1, 2)
print(p2.keywords)       # {'d': 5}

p3 = partial(partial(f, c=1), d=2)
print(p3.func.__name__, p3.args, p3.keywords)  # f () {'c': 1, 'd': 2}
```

也就是说，连续用 `partial` 包装不会产生"套娃"调用开销。

### 5.7 位置参数只能从左边冻结：Placeholder（3.14+）

`partial` 只能冻结**前 N 个**位置参数，不能跳过中间参数：

```python
# 意图：把 str.replace 的 old、new 固定住？
p = partial(str.replace, " ", "_")
print(repr(p("  A B  ")))  # ' '  ← 结果完全不对
```

原因：`str.replace(self, old, new, count)` 的第一个参数是 `self`，`partial` 冻结的 `" "` 成了 `self`，`"_"` 成了 `old`，实际调用变成 `" ".replace("_", "  A B  ")`。

Python 3.14 新增 `functools.Placeholder` 哨兵解决"跳过位置参数"的问题：

```python
from functools import partial, Placeholder as _

# 保留前两个位置参数（self, old），只冻结第三个参数（new=""）
remove = partial(str.replace, _, _, "")
msg = "Hello, dear dear world!"
print(remove(msg, " dear"))  # Hello, world!

# 再基于 remove 冻结第二个占位符
remove_dear = partial(remove, _, " dear")
print(remove_dear(msg))      # Hello, world!
```

规则：

1. `Placeholder` 是单例哨兵，写在 `*args` 中表示"这里留空，调用时按顺序填入"；
2. 只要用了 `Placeholder`，**所有占位符都必须在调用时填满**，否则 `TypeError`；
3. `Placeholder` 不能作为关键字参数传入；
4. 对已有 partial 再套 partial 时，新传入的位置参数会填旧占位符；想继续保留占位，就再插一个 `Placeholder`。

> 3.13 及更早版本没有 `Placeholder`，需要这种能力时只能用 `lambda` 或重新定义包装函数。

### 5.8 partial vs lambda vs 闭包 vs 默认参数

```python
def multiply(x, y):
    return x * y

# 1) lambda：最直观
double_l = lambda x: multiply(x, 2)

# 2) 默认参数：函数参数默认值在定义时绑定
def double_d(x, factor=2):
    return multiply(x, factor)

# 3) 闭包：运行时捕获变量
def make_double():
    factor = 2
    def double(x):
        return multiply(x, factor)
    return double

# 4) partial
double_p = partial(multiply, y=2)
```

| 维度 | `partial` | `lambda` | 闭包 | 默认参数 |
| --- | --- | --- | --- | --- |
| 可读性（简单场景） | 好 | 好 | 一般 | 一般 |
| 元数据（`__name__`/`__doc__`） | 无，需 `update_wrapper` | 无 | 有 | 有 |
| 可 pickle | ✅（函数与参数可 pickle 时） | ❌ | ❌ | ✅ |
| 可内省签名 | ✅ `inspect.signature` 可显示剩余参数 | 部分 | ✅ | ✅ |
| 捕获变量时机 | 创建时冻结值 | 调用时读取 | 调用时读取 | 定义时绑定 |
| 额外函数定义 | 不需要 | 需要 | 需要 | 需要 |
| 性能 | C 实现，开销很小 | 创建新函数对象 | 一般 | 无额外开销 |

选型建议：

- **需要传回调、要可 pickle、要签名可读、要复用** → `partial`；
- **一行内一次性使用、逻辑简单** → `lambda`；
- **需要复杂逻辑或多行语句** → 普通函数/闭包；
- **只是给函数加默认值** → 默认参数最直接。

> ⚠️ 循环中创建 `lambda` 的经典 bug（延迟绑定）用 `partial` 可以自然避免：`partial(handler, i)` 在创建时就冻结了 `i` 的当前值。

### 5.9 典型使用场景

#### 场景 1：为回调固定上下文

```python
from functools import partial

def on_click(button_id, event):
    print(f"按钮 {button_id}: {event}")

buttons = {i: partial(on_click, i) for i in range(3)}

for i, handler in buttons.items():
    handler("click")   # 按钮 0: click / 按钮 1: click / 按钮 2: click

# Tkinter 示例：command=partial(handler, idx)
```

#### 场景 2：配合 map / sorted / filter

```python
from functools import partial

print(list(map(partial(pow, 2), [1, 2, 3])))          # [2, 4, 8]
print(sorted(["bbb", "a", "cc"], key=partial(len)))   # ['a', 'cc', 'bbb']
```

#### 场景 3：依赖注入 / 预配置客户端

```python
import json
from functools import partial

dumps_compact = partial(json.dumps, ensure_ascii=False, separators=(",", ":"))

def request(method, url, **kwargs): ...

get = partial(request, "GET")
post_json = partial(request, "POST", headers={"Content-Type": "application/json"})
```

#### 场景 4：装饰器工厂（带参数的装饰器）

```python
import functools
import time

def retry(times=3, delay=0.1, exceptions=(Exception,)):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions:
                    if attempt == times:
                        raise
                    time.sleep(delay)
        return wrapper
    return decorator

retry_network = partial(retry, times=5, delay=1.0, exceptions=(ConnectionError,))

@retry_network
def fetch(url):
    ...
```

#### 场景 5：函数表 / 策略表

```python
from functools import partial

def save_to(format, path, data): ...

handlers = {
    "csv": partial(save_to, "csv"),
    "json": partial(save_to, "json"),
}
handlers["json"]("out", {"a": 1})
```

#### 场景 6：减少重复的长参数列表

```python
from functools import partial

def draw_rect(ctx, x, y, w, h, color, width, dashed):
    ...

draw_box = partial(draw_rect, w=10, h=10, color="black", width=1, dashed=False)
draw_box(ctx, 0, 0)
draw_box(ctx, 20, 20)
```

### 5.10 常见陷阱

#### 陷阱 1：创建时不校验参数

`partial` 创建时**不会**检查参数是否合法，错误要到调用时才暴露：

```python
p = partial(int, base=999)   # 创建成功
p("10")                      # ValueError: int() base must be >= 2 and <= 36, or 0
```

#### 陷阱 2：关键字覆盖是"静默"的

```python
p = partial(power, exp=2)
print(p(3, exp=3))   # 27 —— 调用方无意中覆盖了默认 exp，且不报错
```

如果希望冻结值不可被覆盖，用位置参数或自己包一层校验函数。

#### 陷阱 3：位置参数与关键字冲突

```python
p = partial(f, 1)
p(2, a=3)   # TypeError: f() got multiple values for argument 'a'
```

#### 陷阱 4：没有函数名和文档

```python
p = partial(power, exp=2)
print(p.__name__)  # AttributeError: 'functools.partial' object has no attribute '__name__'
```

需要元数据时：

```python
import functools

def square(x):
    """求平方"""
    return x ** 2

p = functools.partial(power, exp=2)
functools.update_wrapper(p, square)   # 会设置 __wrapped__，inspect.signature 会跟随 square
print(p.__name__, p.__doc__)
```

#### 陷阱 5：类属性里的 partial 不会自动绑定 self

```python
class C:
    def g(self, x, y=0):
        return (x, y)

    h = partial(g, 1)   # 注意：这不是方法！

c = C()
c.h()   # TypeError: C.g() missing 1 required positional argument: 'x'
```

在 Python 3.13 上这一行还会触发 `FutureWarning: functools.partial will be a method descriptor in future Python versions`。类里要预设方法参数，请用 `partialmethod`（见下一节），或显式用 `staticmethod` 包裹。

#### 陷阱 6：pickle 限制

partial 对象可 pickle，前提是 `func` 和参数本身可 pickle。`lambda`、局部函数、绑定方法等通常不可 pickle，导致多进程场景失败：

```python
import pickle
pickle.dumps(partial(lambda x: x, 1))  # PicklingError
```

多进程/分布式场景请用**模块级函数 + partial**。

#### 陷阱 7：签名与类型提示

`inspect.signature(partial(f, 1))` 会正确显示剩余参数：

```python
print(inspect.signature(partial(f, 1)))  # (b, c=0, *, d=0)
```

但类型检查器对 `partial` 的推导有限，复杂场景建议封装成普通函数，或使用 `Callable[..., R]` / `ParamSpec`（3.10+）：

```python
from typing import Callable, ParamSpec, TypeVar

P = ParamSpec("P")
R = TypeVar("R")

def typed_partial(func: Callable[P, R], *args, **kwargs) -> Callable[..., R]:
    return partial(func, *args, **kwargs)
```

### 5.11 源码级理解

CPython 的 `partial` 类型核心逻辑（简化）：

- `partial_new`：若 `func` 本身是 partial，则把它的 `args`、`keywords` 合并进来（扁平化）；
- `partial_call`：`func(*self->args, *args, **(self->keywords | kwargs))`；
- `partial_get`：3.13 起加入 `__get__`，用于未来把它变成方法描述符；
- `tp_repr`、`tp_reduce`：提供可读表示与 pickle 支持。

---

## 6. 方法版偏函数：functools.partialmethod

`functools.partialmethod`（3.4+）是**描述符**，专为"类中方法预设参数"设计：访问 `obj.method` 时返回一个已经绑定 `self` 的 partial 对象。

### 6.1 基本用法

```python
import functools

class Cell:
    def __init__(self):
        self._alive = False

    @property
    def alive(self):
        return self._alive

    def set_state(self, state):
        self._alive = bool(state)

    set_alive = functools.partialmethod(set_state, True)
    set_dead = functools.partialmethod(set_state, False)


c = Cell()
print(c.alive)   # False
c.set_alive()
print(c.alive)   # True
c.set_dead()
print(c.alive)   # False
```

### 6.2 partial 与 partialmethod 的区别

| 对比项 | `partial` | `partialmethod` |
| --- | --- | --- |
| 使用位置 | 普通函数/回调 | **类体**中定义方法 |
| 描述符协议 | 3.13 起有 `__get__`，行为仍在演进 | 完整描述符，自动绑定 `self`/`cls` |
| 访问 `obj.attr` | 返回同一个 partial 对象 | 返回新的、已绑定 self 的 partial |
| 直接调用 | 可调用 | 类属性本身不可直接调用，需通过实例/类访问 |
| 典型用途 | 冻结普通函数参数 | 给方法取别名并预设参数 |

### 6.3 与 staticmethod / classmethod 组合

```python
import functools

class M:
    @staticmethod
    def smul(a, b):
        return a * b

    mul3 = functools.partialmethod(smul, 3)   # 绑定静态方法

    @classmethod
    def cadd(cls, a, b):
        return (cls.__name__, a + b)

    add10 = functools.partialmethod(cadd, 10)  # 绑定类方法

    def inst(self, a, b):
        return (a, b)

    hole = functools.partialmethod(inst, 1)


m = M()
print(m.mul3(4), M.mul3(4))   # 12 12
print(m.add10(5), M.add10(5)) # ('M', 15) ('M', 15)
print(m.hole(2))              # (1, 2) —— self 在最前，随后是预设参数
```

### 6.4 使用要点

- `partialmethod` 的第一个参数可以是描述符（普通函数、`staticmethod`、`classmethod`、`abstractmethod`、另一个 `partialmethod`）；
- 与 `partial` 一样，预设的是**最左边**的参数（在 `self`/`cls` 之后）；
- 要让 `register` 之类的属性正常工作，`partialmethod` 需放在合适位置；组合 `classmethod` 时把 `classmethod` 放在里面；
- 仅支持 Python 3.4+；类属性中的 `partial` 在 3.13 会告警，正是迁移到 `partialmethod` 的信号。

---

## 7. 折叠聚合：functools.reduce

### 7.1 语义

`functools.reduce(function, iterable[, initial])` 用二元函数把序列从左到右折叠成单个值：

```
reduce(f, [a, b, c, d]) = f(f(f(a, b), c), d)
```

- `function(accumulator, element)`：第一个参数是累计值，第二个是当前元素；
- 提供 `initial` 时，从 `initial` 开始，空序列返回 `initial`；
- 不提供 `initial` 时，空序列抛 `TypeError`，单元素序列直接返回该元素；
- Python 3.14 起 `initial` 支持关键字传参：`reduce(f, xs, initial=0)`。

等价实现：

```python
_initial_missing = object()

def reduce(function, iterable, /, initial=_initial_missing):
    it = iter(iterable)
    if initial is _initial_missing:
        value = next(it)          # 空序列 -> TypeError
    else:
        value = initial
    for element in it:
        value = function(value, element)
    return value
```

### 7.2 基础示例

```python
from functools import reduce
import math

# 求和 / 求积
print(reduce(lambda a, b: a + b, [1, 2, 3, 4, 5]))          # 15
print(reduce(lambda a, b: a * b, range(1, 6), 1))           # 120

# 最大公约数
print(reduce(math.gcd, [12, 18, 24]))                       # 6

# 求最值（内置 max 可直接接收可迭代对象，这里仅作演示）
print(reduce(lambda a, b: a if a > b else b, [3, 9, 2]))    # 9

# 空序列 + initial
print(reduce(lambda a, b: a + b, [], 0))                    # 0
```

### 7.3 进阶示例

#### 字典合并

```python
dicts = [{"a": 1}, {"b": 2}, {"a": 3}]
print(reduce(lambda acc, d: acc | d, dicts, {}))   # {'a': 3, 'b': 2}
```

#### 展平嵌套列表（仅演示，实际建议 `itertools.chain.from_iterable`）

```python
print(reduce(lambda acc, xs: acc + xs, [[1, 2], [3], [4, 5]], []))  # [1, 2, 3, 4, 5]
```

#### 函数组合（compose）

```python
def compose(*funcs):
    """compose(f, g)(x) == f(g(x))；从左到右写出执行顺序的逆序"""
    return reduce(lambda f, g: lambda x: g(f(x)), funcs, lambda x: x)

process = compose(str.strip, str.lower, lambda s: s.replace(" ", "-"))
print(process("  Hello World  "))   # hello-world
```

#### 数据管道（pipeline）

```python
from functools import partial, reduce

def replace_ws(text, old=" ", new="_"):
    return text.replace(old, new)

steps = [str.strip, str.lower, partial(replace_ws, new="-")]

def pipeline(value, steps):
    return reduce(lambda acc, step: step(acc), steps, value)

print(pipeline("  Hello World  ", steps))   # hello-world
```

#### 构建配置 / 归并操作

```python
operations = [lambda x: x + 1, lambda x: x * 2, lambda x: x - 3]
print(reduce(lambda acc, op: op(acc), operations, 10))   # ((10+1)*2)-3 = 19
```

### 7.4 什么时候不该用 reduce

| 需求 | 更好的写法 |
| --- | --- |
| 求和 | `sum(xs)` |
| 求积 | `math.prod(xs)`（3.8+） |
| 求最值 | `max(xs)` / `min(xs)` |
| 需要所有中间结果 | `itertools.accumulate(xs, f)` |
| 简单遍历、需要调试/断点 | `for` 循环 |
| 条件短路 | `any` / `all` |
| 展平嵌套 | `itertools.chain.from_iterable` |
| 过滤/映射 | 列表推导式 / `filter` / `map` |
| 分组 | `collections.defaultdict` / `itertools.groupby` |

反模式：

```python
# ❌ 用 reduce 累积列表：O(n²)，可读性差
result = reduce(lambda acc, x: acc + [x * 2], range(1000), [])

# ✅
result = [x * 2 for x in range(1000)]

# ❌ 在 reduce 中做副作用（写文件、修改外部状态）
# ✅ 用普通循环，意图更清晰
```

> `reduce` 的价值在于"二元聚合逻辑本身可以像数据一样传递"（例如从配置、插件、函数表中动态取出），而不是替代所有循环。Guido 当年把 `reduce` 移出内置函数，原因之一就是滥用会降低可读性。

---

## 8. 比较函数转 key：functools.cmp_to_key

### 8.1 背景

Python 3 的排序接口只接受 `key` 函数（返回排序键），而 Python 2 风格是 `cmp(a, b)` 比较函数（返回负数/0/正数）。`cmp_to_key` 把后者适配成前者，主要用于**迁移旧代码**或**比较逻辑难以表达成 key** 的场景。

```python
import functools

def version_cmp(a, b):
    ta = tuple(int(p) for p in a.split("."))
    tb = tuple(int(p) for p in b.split("."))
    return (ta > tb) - (ta < tb)   # 返回 -1 / 0 / 1

versions = ["1.10.0", "1.2.0", "1.9.3", "1.2.10"]
print(sorted(versions, key=functools.cmp_to_key(version_cmp)))
# ['1.2.0', '1.2.10', '1.9.3', '1.10.0']
```

> 这个例子其实可以用 key 直接写：`key=lambda v: tuple(int(p) for p in v.split("."))`。`cmp_to_key` 更适合"比较依赖外部状态、语言环境或多字段混合规则"的情况。

### 8.2 区域感知排序

```python
import functools
import locale

# 区域名称因平台而异：Linux 常见 "zh_CN.UTF-8"，Windows 常见 "Chinese_China.936"。
# 传空字符串表示使用系统当前区域设置。
locale.setlocale(locale.LC_COLLATE, "")

words = ["张三", "李四", "王五"]
print(sorted(words, key=functools.cmp_to_key(locale.strcoll)))
```

`locale.strcoll` 正是"比较函数"签名，`cmp_to_key` 让它能用于 `sorted` 的 `key=`。注意排序结果依赖当前系统区域，跨平台部署时应显式指定并处理 `locale.Error`。

### 8.3 与其他接受 key 的接口配合

`cmp_to_key` 返回的是一个包装类（CPython 中名为 `KeyWrapper`），凡是接受 `key=` 的接口都能用：

```python
import heapq
from functools import cmp_to_key

key = cmp_to_key(lambda a, b: (a > b) - (a < b))
print(heapq.nsmallest(2, [3, 1, 2], key=key))   # [1, 2]
print(min([3, 1, 2], key=key))                  # 1
```

### 8.4 使用要点

- 比较函数必须返回负数 / 0 / 正数（不要只返回 `True/False`，虽然排序可能"碰巧"正确）；
- 比较函数会被调用 O(n log n) 次，且每次都有包装对象开销；能用 key 函数就别用 `cmp_to_key`；
- 包装后只能与同类包装对象比较，与其他类型混排会抛 `TypeError`；
- 使用 `key=` 的排序是**稳定排序**，`cmp_to_key` 不会破坏稳定性。

---

## 9. 综合实战案例

### 9.1 带元数据保留的 retry 装饰器（wraps + partial）

```python
import functools
import time

def retry(times=3, delay=0.1, exceptions=(Exception,)):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as exc:
                    if attempt == times:
                        raise
                    print(f"[retry] {func.__name__} 第 {attempt} 次失败: {exc}")
                    time.sleep(delay)
        return wrapper
    return decorator


@retry(times=3, delay=0.01, exceptions=(ValueError,))
def parse_int(text):
    """把字符串解析为整数"""
    return int(text)


print(parse_int.__name__, parse_int.__doc__)  # parse_int 把字符串解析为整数
print(parse_int("42"))                        # 42
```

### 9.2 singledispatch 替代 if/elif 分支

```python
from functools import singledispatch

@singledispatch
def to_sql_value(value):
    return str(value)

@to_sql_value.register
def _(value: type(None)):
    return "NULL"

@to_sql_value.register
def _(value: bool):
    return "TRUE" if value else "FALSE"

@to_sql_value.register
def _(value: int | float):
    return repr(value)

@to_sql_value.register
def _(value: str):
    return "'" + value.replace("'", "''") + "'"

rows = [None, True, 42, 3.14, "O'Reilly"]
print([to_sql_value(v) for v in rows])
# ['NULL', 'TRUE', '42', '3.14', "'O''Reilly'"]
```

### 9.3 缓存 + 依赖注入

```python
import functools

class PriceService:
    def __init__(self, fetch):
        self._fetch = fetch

    @functools.cache
    def get_price(self, sku):
        return self._fetch(sku)


calls = []

def fake_fetch(sku):
    calls.append(sku)
    return 9.9

service = PriceService(fake_fetch)
print(service.get_price("A-1"))   # 9.9
print(service.get_price("A-1"))   # 9.9（第二次命中缓存）
print(calls)                      # ['A-1']（fetch 只被调用一次）
```

> 注意：`@cache` 修饰实例方法同样会强引用 `self` 并缓存 `self`，仅适合生命周期与进程相同的服务对象。需要随实例销毁时改用实例字典。

### 9.4 管道式数据处理（partial + reduce）

```python
from functools import partial, reduce

def strip_text(text):
    return text.strip()

def lower_text(text):
    return text.lower()

def replace_chars(text, old=" ", new="_"):
    return text.replace(old, new)

def pipeline(value, steps):
    return reduce(lambda acc, step: step(acc), steps, value)

clean = partial(pipeline, steps=[strip_text, lower_text, partial(replace_chars, new="-")])
print(clean("  Hello World  "))   # hello-world
```

---

## 10. 版本变更速查

| 版本 | 变更 |
| --- | --- |
| 2.5 | `partial`、`wraps`、`update_wrapper` 引入 |
| 3.0 | `reduce` 从内置函数移入 `functools` |
| 3.2 | `lru_cache`、`total_ordering`、`cmp_to_key` 引入 |
| 3.3 | `lru_cache` 增加 `typed` 参数 |
| 3.4 | `partialmethod`、`singledispatch` 引入；`total_ordering` 支持返回 `NotImplemented` |
| 3.7 | `singledispatch` 的 `register` 支持类型注解推断 |
| 3.8 | `cached_property`、`singledispatchmethod` 引入；`lru_cache` 可直接无括号使用 |
| 3.9 | `cache` 引入；`lru_cache` 增加 `cache_parameters()` |
| 3.11 | `singledispatch.register` 支持 `typing.Union` 与 `types.UnionType` 注解 |
| 3.12 | `cached_property` 移除内部锁；`WRAPPER_ASSIGNMENTS` 增加 `__type_params__` |
| 3.13 | 类属性中的 `partial` 触发 FutureWarning（未来会变成方法描述符） |
| 3.14 | 新增 `functools.Placeholder`；`reduce` 的 `initial` 支持关键字传参 |
| 3.15 | `singledispatchmethod` 支持非描述符可调用对象 |

> 兼容性提示：写库代码时如果要在 3.8/3.9 上运行，避免使用 `Placeholder`、`str | None` 注解等新语法；`cache` 在 3.8 上不存在，可用 `lru_cache(maxsize=None)` 替代。

---

## 11. 最佳实践与常见陷阱

### 11.1 最佳实践

1. **装饰器必写 `@functools.wraps(func)`**，否则日志、文档、框架参数解析都会出问题。
2. **只缓存纯函数**；缓存有副作用的函数会导致难以复现的 bug。
3. **为缓存提供清理入口**：测试之间调用 `cache_clear()`，避免测试互相污染。
4. **限制 `maxsize`**，或确保输入域有限；`@cache` 无上限，只用于确定不会膨胀的场景。
5. **实例方法缓存优先用实例字典或模块级函数**，避免 `lru_cache` 强引用 `self`。
6. **类中预设方法参数用 `partialmethod`**，不要直接在类体里赋值 `partial`。
7. **`partial` 优先用于"回调 + 上下文"和"预配置"**，不要为了少写一个参数而制造难懂的调用链。
8. **`reduce` 只用于语义自然的二元聚合**；求和用 `sum`，求积用 `math.prod`，要看中间值用 `accumulate`。
9. **类型分派优先 `singledispatch`**，把 `isinstance` 分支变成可注册、可扩展的实现表。
10. **`cmp_to_key` 是迁移工具**，新代码优先写 key 函数。

### 11.2 快速自检清单

| 症状 | 可能原因 | 修复 |
| --- | --- | --- |
| 日志里函数名都是 `wrapper` | 忘记 `wraps` | 加 `@functools.wraps(func)` |
| `signature()` 显示 `*args, **kwargs` | 未设置 `__wrapped__` | 同上 |
| 缓存命中率为 0 | 参数含不可哈希对象 / `self` 不同 / 关键字参数顺序不同 | 统一调用形式 |
| 内存持续增长 | `maxsize=None` 或无界 `cache` | 设置上限并定期清理 |
| 实例无法回收 | `lru_cache` 缓存了方法/`self` | 改用实例字典或模块级函数 |
| 类中 `partial` 调用报缺少参数 | `partial` 不绑定 `self` | 用 `partialmethod` 或 `staticmethod` |
| `reduce` 报 `TypeError: reduce() of empty iterable` | 空序列未给 `initial` | 传 `initial` 或改用 `sum` |
| 多进程下 pickle 失败 | 用了 `lambda`/局部函数 | 改用模块级函数 + `partial` |

---

## 12. API 速查表

```python
import functools

# ---- 元数据 ----
@functools.wraps(func)                      # 复制元数据 + 设置 __wrapped__
functools.update_wrapper(wrapper, wrapped)  # 底层实现
functools.WRAPPER_ASSIGNMENTS               # 默认复制的属性名
functools.WRAPPER_UPDATES                   # 默认更新的属性名

# ---- 缓存 ----
@functools.cache                            # 无界缓存（3.9+）
@functools.lru_cache(maxsize=128, typed=False)
@functools.cached_property                  # 属性级缓存（3.8+）
func.cache_info(); func.cache_clear(); func.cache_parameters()

# ---- 比较与分派 ----
@functools.total_ordering
@functools.singledispatch
@functools.singledispatchmethod             # 3.8+
func.register(Type) / func.dispatch(Type) / func.registry

# ---- 参数适配 ----
p = functools.partial(func, *args, **kwargs)
p.func; p.args; p.keywords
functools.partialmethod(func, *args, **kwargs)   # 类中使用
functools.Placeholder                       # 位置占位符（3.14+）

# ---- 折叠与排序 ----
functools.reduce(func, iterable[, initial])
functools.cmp_to_key(cmp_func)
```

### 最小记忆卡片

```
wraps       → 装饰器不丢名字
lru_cache   → 记住算过的
cache       → 无限记住（3.9+）
cached_property → 属性只算一次
total_ordering  → 一个比较方法补齐六个
singledispatch  → 按第一个参数类型分派
partial     → 先冻结几个参数
partialmethod → 类里冻结方法参数
reduce      → 从左到右折叠成一个值
cmp_to_key  → 老式比较函数接进 key=
```

---

## 13. 参考资料

- Python 官方文档：`functools` — <https://docs.python.org/zh-cn/3/library/functools.html>
- Python 官方文档：`functools.partial` — <https://docs.python.org/zh-cn/3/library/functools.html#functools.partial>
- Python 3.14 新特性（`Placeholder`）— <https://docs.python.org/3.14/whatsnew/3.14.html>
- PEP 443：Single-dispatch generic functions — <https://peps.python.org/pep-0443/>
- Python 排序指南（`cmp_to_key` 与 key 函数）— <https://docs.python.org/zh-cn/3/howto/sorting.html>
- Python 官方 FAQ：如何缓存方法调用 — <https://docs.python.org/zh-cn/3/faq/programming.html#how-do-i-cache-method-calls>
- CPython 源码 `Lib/functools.py` — <https://github.com/python/cpython/blob/main/Lib/functools.py>

---

> **文档信息**：示例在 CPython 3.13.5 上运行验证；`Placeholder` 相关示例需要 Python 3.14+，在 3.13 上会因 `ImportError` 无法执行。
