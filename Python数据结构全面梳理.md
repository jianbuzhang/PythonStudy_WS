# Python 数据结构全面梳理：list / tuple / dict / set 与 collections

> **文档定位**：一份可以直接当"考前讲义 + 日常速查手册"用的 Python 容器知识梳理。覆盖 4 个内置容器（`list`、`tuple`、`dict`、`set`）以及标准库 `collections` 中的常用容器，重点讲清 **接口怎么用、底层什么代价、哪里容易错、彼此怎么选**。
>
> **适用版本**：以 CPython **3.8 – 3.13** 为基准，示例在 3.13 上验证。涉及 3.9/3.10+ 才有的特性会在文中显式标注"3.9+ / 3.10+"；3.7 之前"dict 无序"的旧结论不再适用，文中会单独提醒。
>
> **配套题库**：同目录《Python数据结构选择题220题_含答案解析.md》，共 220 道单选，答案与解析默认折叠，建议读完一章做对应部分。
>
> **使用建议**：第一遍通读第 1、7 章建立全局观；第二遍按需精读第 2–6 章；考前只看每章末尾的"坑清单"和第 7 章的对比表。

---

## 目录

- 第 1 章 总览与心智模型：四个维度、复杂度、协议
- 第 2 章 list：动态数组的全部细节
- 第 3 章 tuple：不可变序列与被低估的用途
- 第 4 章 dict：哈希表、视图与合并
- 第 5 章 set 与 frozenset：集合运算与去重
- 第 6 章 collections：Counter / defaultdict / OrderedDict / deque / namedtuple / ChainMap / User*
- 第 7 章 横向对比与 30 组易混淆点
- 第 8 章 易错输出题自测（附折叠答案）
- 第 9 章 学习路径与考前清单

---

# 第 1 章 总览与心智模型

## 1.1 一张表看清 4 个内置容器

| 容器 | 字面量 | 可变性 | 有序性 | 可哈希 | 元素重复 | 访问方式 | 典型用途 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `list` | `[1, 2]` | 可变 | 有序（插入序） | ❌ 不可哈希 | 允许 | 下标 / 切片 | 有序集合、栈、队列（不推荐）、批量处理 |
| `tuple` | `(1, 2)` | 不可变 | 有序 | ✅ 元素全可哈希时 | 允许 | 下标 / 切片 | 固定记录、多返回值、dict 键、常量表 |
| `dict` | `{'a': 1}` | 可变 | 有序（3.7+ 保证插入序） | ❌ 不可哈希 | 键唯一 | 键访问 | 映射、计数、缓存、结构化数据 |
| `set` | `{1, 2}` | 可变 | **无序**（迭代顺序不可依赖） | ❌ 不可哈希 | 自动去重 | 只能遍历 / `in` | 去重、成员判断、集合运算 |
| `frozenset` | `frozenset({1, 2})` | 不可变 | 无序 | ✅ 可哈希 | 自动去重 | 同 set | 可哈希的集合、dict 键、set 的元素 |
| `str` | `'abc'` | 不可变 | 有序 | ✅ | 字符可重复 | 下标 / 切片 | 文本 |
| `bytes` | `b'abc'` | 不可变 | 有序 | ✅ | 同上 | 下标 / 切片 | 二进制 |

**三条最容易记混的结论**：

1. **`{}` 是空字典，不是空集合**；空集合必须写 `set()`。
2. **"有序"对 dict 指插入顺序**，不是自动排序；想按 key 排序要显式 `sorted(d.items())`。
3. **tuple 的"不可变"指元组持有的引用不能换**，但如果元素本身是 list，list 内部仍可改。

## 1.2 四个维度：判断该用谁

面对一个需求，按下面 4 个问题依次追问，基本就能选对容器：

```text
需要"键 -> 值"的映射吗？
├── 是 → dict（需要默认值 → defaultdict；需要计数 → Counter；需要顺序敏感 → OrderedDict）
└── 否 → 只需要"元素是否存在"或去重吗？
    ├── 是 → set（需要不可变/可哈希 → frozenset）
    └── 否 → 需要保持顺序、按下标访问吗？
        ├── 是 → 元素会变吗？
        │   ├── 会 → list（频繁两端增删 → deque）
        │   └── 不会 → tuple（字段有名字 → namedtuple）
        └── 否 → 先想清楚是否真的不需要顺序，多数场景仍建议 list
```

**一个常被忽略的点**：`dict` 的键集合在语义上就是一个 `set`。所以"去重且保序"最常用的写法是 `list(dict.fromkeys(items))`——利用 dict 键唯一 + 插入有序，一趟 O(n)。

## 1.3 可变性、有序性、可哈希性到底是什么

### 1.3.1 可变（mutable）与不可变（immutable）

- **可变对象**：创建后可以原地修改，`id()` 不变。典型：`list`、`dict`、`set`、`bytearray`、`deque`、`Counter`、`defaultdict`、`OrderedDict`。
- **不可变对象**：任何"修改"都会产生新对象，原名绑定到新对象（`id` 变化）。典型：`int`、`float`、`bool`、`str`、`bytes`、`tuple`、`frozenset`、`namedtuple`。

```python
a = [1, 2]
print(id(a))          # 例如 140... 
a.append(3)
print(id(a))          # 不变：原地修改

s = "ab"
print(id(s))
s += "c"
print(id(s))          # 变了：str 不可变，+= 生成新对象
```

> ⚠️ 注意：`tuple` 的不可变是**浅层**的。`t = ([1], [2])` 中 `t[0].append(9)` 完全合法，因为变的是 list 对象，不是 tuple 的槽位。

### 1.3.2 "有序"的三种不同含义

| 说法 | 含义 | 例子 |
| :--- | :--- | :--- |
| 插入有序 | 迭代顺序 = 元素被加入的顺序 | `dict`（3.7+）、`list`、`deque`、`OrderedDict` |
| 排序有序 | 元素按大小/`key` 排好 | `sorted()` 的结果、`heapq` 内部 |
| 无序 | 迭代顺序由哈希决定，跨进程/跨版本可能不同 | `set`、`frozenset` |

`set` 的无序**不是随机**：同一次运行内，同样的插入顺序通常得到同样的迭代顺序；但这是实现细节，**绝不能依赖**。需要稳定顺序时，用 `sorted(s)` 或"去重保序"写法。

### 1.3.3 可哈希（hashable）的准确含义

一个对象可哈希，当且仅当：

1. 它有 `__hash__` 方法且返回值不为 `None`（`__hash__ = None` 表示显式禁用哈希）；
2. 它的哈希值在对象生命周期内**不变**；
3. 满足契约：`a == b` 蕴含 `hash(a) == hash(b)`（反过来不成立，哈希冲突是正常的）。

**哪些可哈希？**

- ✅ `int`、`float`、`bool`、`complex`、`str`、`bytes`、`None`、`frozenset`、元素全可哈希的 `tuple`；
- ✅ 默认情况下的自定义类实例（按 `id` 计算哈希）；
- ❌ `list`、`dict`、`set`、`bytearray`（"可变 + 实现了 `__eq__`"的类型通常不可哈希）；
- ⚠️ 定义了 `__eq__` 但没定义 `__hash__` 的类：`__hash__` 自动变成 `None`，实例不可哈希。

```python
hash((1, 2))          # OK
hash((1, [2]))        # TypeError: unhashable type: 'list'
{(1, 2): "ok"}        # 元组可作键（元素都可哈希时）
{[1, 2]: "no"}        # TypeError: unhashable type: 'list'
{frozenset({1, 2}): "ok"}   # frozenset 可作键
```

**为什么可变容器不能当 dict 键 / set 元素？** 哈希表按哈希值定位元素；如果键的内容能变，哈希值就变了，之后再也找不到它，哈希表的正确性被破坏。所以语言层面直接禁止。

**一个反直觉但重要的细节**：`1`、`1.0`、`True` 的哈希值相同且互相 `==`，因此在 dict / set 中**被视为同一个键**：

```python
d = {1: 'a'}
d[1.0] = 'b'
d[True] = 'c'
print(d)          # {1: 'c'} —— 键始终是第一次插入的 1
print(len({1, 1.0, True}))   # 1
```

## 1.4 时间复杂度速查表（平均情况）

| 操作 | list | tuple | dict | set | deque |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 按下标 `[]` | O(1) | O(1) | — | — | 两端 O(1)，中间 O(n) |
| `append` / `appendleft` | 尾部 O(1) 摊还 | — | — | — | O(1) |
| `insert(0, x)` | O(n) | — | — | — | O(n) |
| `pop()` 尾部 | O(1) | — | — | — | O(1) |
| `pop(0)` | O(n) | — | — | — | `popleft()` O(1) |
| 按值删除 `remove` | O(n) | — | — | O(1) 平均 | O(n) |
| 成员判断 `x in c` | **O(n)** | O(n) | **O(1)** 平均 | **O(1)** 平均 | O(n) |
| 按 key 取值 | — | — | O(1) 平均 | — | — |
| 遍历 | O(n) | O(n) | O(n) | O(n) | O(n) |
| 切片 | O(k) | O(k) | — | — | ❌ 不支持 |
| 排序 | O(n log n) | — | — | — | — |
| `len` | O(1) | O(1) | O(1) | O(1) | O(1) |
| 复制（浅） | O(n) | O(1)（同一对象） | O(n) | O(n) | O(n) |

> 说明：dict / set 的 O(1) 是**平均**复杂度；极端哈希冲突下最坏 O(n)。tuple 的 `t[:]` 不复制，直接返回同一对象（不可变所以安全）；`tuple(t)` 对 tuple 也返回同一对象。

**由复杂度直接推出的工程结论**：

- 在 list 里做 `x in big_list` 循环 → O(n²)，改成 set 往往立刻快几个数量级；
- 用 list 当队列 `pop(0)` → O(n²)，改用 `collections.deque`；
- 在 list 中间频繁插删 → O(n²)，考虑 deque / 链表结构 / 批量重建；
- 计数用 `dict.get` 手写没问题，但 `Counter` 更快更清楚。

## 1.5 协议视角：内置容器为什么能通用

Python 不靠继承强制接口，而靠"协议"（duck typing）。理解这几个协议，就能理解为什么 `for x in 任意容器`、`x in 容器`、`len(容器)` 都能工作。

| 协议 / ABC | 必须实现 | 免费获得的能力 | 典型实现 |
| :--- | :--- | :--- | :--- |
| 可迭代 `Iterable` | `__iter__` | `for`、解包、`list()`、`sum()` 等 | list、dict、set、str、文件、生成器 |
| 迭代器 `Iterator` | `__next__`（+ `__iter__` 返回自身） | `next()`、一次性消费 | `iter(list)`、生成器 |
| 容器 `Container` | `__contains__` | `in` | 所有容器 |
| 序列 `Sequence` | `__len__` + `__getitem__` | `in`、`index()`、`count()`、`reversed()`、迭代 | list、tuple、str、range |
| 可变序列 `MutableSequence` | 再加 `__setitem__`、`__delitem__`、`insert` | `append`、`extend`、`pop`、`remove`、`__iadd__` | list、UserList |
| 映射 `Mapping` | `__getitem__`、`__iter__`、`__len__` | `get`、`keys`、`items`、`values`、`==` | dict、MappingProxyType |
| 可变映射 `MutableMapping` | 再加 `__setitem__`、`__delitem__` | `pop`、`popitem`、`clear`、`update`、`setdefault` | dict、UserDict |
| 集合 `Set` | `__contains__`、`__iter__`、`__len__` | `&`、`\|`、`-`、`^`、比较运算、`isdisjoint` | set、frozenset |
| 可变集合 `MutableSet` | 再加 `add`、`discard` | `remove`、`pop`、`clear`、`&=` 等原地运算 | set |

> 这些 ABC 都在 `collections.abc` 中。判断"是不是序列"时，用 `isinstance(x, collections.abc.Sequence)` 比 `isinstance(x, (list, tuple))` 更通用（注意 `str` 也是 Sequence）。

```python
from collections.abc import Sequence, Mapping, Set

print(isinstance([1, 2], Sequence))     # True
print(isinstance((1, 2), Sequence))     # True
print(isinstance("ab", Sequence))       # True（str 也是序列）
print(isinstance({1: 2}, Mapping))      # True
print(isinstance({1, 2}, Set))          # True
print(isinstance(frozenset(), Set))     # True
```

## 1.6 容器选型的 12 个典型场景

| 场景 | 首选 | 理由 / 备注 |
| :--- | :--- | :--- |
| 保存一批有序、会增删的元素 | `list` | 下标访问 O(1)，尾部增删快 |
| 固定字段的一条记录 | `tuple` / `namedtuple` | 不可变、可哈希、可解包 |
| 函数返回多个值 | `tuple` | 本质就是返回一个元组 |
| 键值配置、缓存 | `dict` | 平均 O(1) 查找 |
| 需要"键不存在时自动给默认值" | `defaultdict` | 比 `setdefault` 更简洁 |
| 统计词频 / 票数 | `Counter` | 自带计数与 Top-N |
| 去重、快速判断存在性 | `set` | 平均 O(1) `in` |
| 去重且保持原顺序 | `list(dict.fromkeys(seq))` | 一趟完成，O(n) |
| 频繁从头部增删（队列/滑动窗口） | `deque` | 两端 O(1) |
| 需要按插入顺序做 LRU / 移到末尾 | `OrderedDict` | `move_to_end`、`popitem(last=)` |
| 多层配置覆盖查找 | `ChainMap` | 从前往后逐层查找 |
| 继承并定制 dict / list 行为 | `UserDict` / `UserList` | 避免直接继承内置类型的坑 |

## 1.7 本章小结

- 先问"映射、集合、序列"哪一类，再问"可变吗、需要顺序吗、需要哈希吗"。
- dict / set 的 O(1) 来自哈希表；list 的 `in` 是 O(n)，这是最常见的性能事故来源。
- `{}` 是 dict；空 set 用 `set()`；dict 的"有序"是插入序。
- tuple 不可变是浅层的；可哈希的前提是"内容不可变 + `__hash__` 不为 None"。
- 1 / 1.0 / True 在 dict 和 set 中是同一个键。

---

# 第 2 章 list：动态数组的全部细节

`list` 是 Python 里使用频率最高的容器：有序、可变、允许重复、按下标 O(1) 访问。它的底层是**动态数组**（连续存放 `PyObject*` 指针），这个事实决定了它所有性能特征：尾部操作快、头部/中间插删慢、`in` 是线性扫描。

## 2.1 创建 list 的 8 种方式

```python
[]                          # 空列表
[1, 2, 3]                   # 字面量
list()                      # 空列表
list((1, 2, 3))             # 从任意可迭代对象转换
list("abc")                 # ['a', 'b', 'c'] —— 按字符拆开
list(range(5))              # [0, 1, 2, 3, 4]
list({"a": 1, "b": 2})      # ['a', 'b'] —— 对 dict 迭代得到的是键
[x * x for x in range(5)]   # 列表推导式
[0] * 5                     # [0, 0, 0, 0, 0]
```

**易错点**：

```python
list(123)        # ❌ TypeError: 'int' object is not iterable
list("hello")    # ⚠️ ['h','e','l','l','o']，不是 ['hello']
list({"a": 1})   # ⚠️ 得到键，不是键值对；要键值对用 list(d.items())
```

## 2.2 索引与切片

### 2.2.1 下标访问

```python
a = [10, 20, 30, 40]
a[0]    # 10
a[-1]   # 40（最后一个）
a[-4]   # 10（第一个）
a[4]    # ❌ IndexError: list index out of range
a[1.0]  # ❌ TypeError: list indices must be integers or slices, not float
a[True] # 20 —— bool 是 int 的子类，True == 1，等价于 a[1]（不推荐这样写）
```

- 负索引从 `-1` 开始，`-len(a)` 是第一个；再小就 `IndexError`。
- 下标必须是整数（`int` / `bool`）或切片；**切片不会因为越界报错**。

### 2.2.2 切片：`a[start:stop:step]`

规则：**左闭右开**，`start` 默认 0，`stop` 默认 `len(a)`，`step` 默认 1，负步长时默认方向反转。

```python
a = [0, 1, 2, 3, 4, 5]

a[1:4]      # [1, 2, 3]
a[:3]       # [0, 1, 2]
a[3:]       # [3, 4, 5]
a[:]        # [0, 1, 2, 3, 4, 5]  浅拷贝
a[::2]      # [0, 2, 4]
a[1::2]     # [1, 3, 5]
a[::-1]     # [5, 4, 3, 2, 1]     反转（返回新列表）
a[4:1:-1]   # [4, 3, 2]
a[-3:-1]    # [3, 4]
a[100:200]  # []  —— 越界不报错，自动截断
a[-100:2]   # [0, 1]
a[::0]      # ❌ ValueError: slice step cannot be zero
```

**切片的三条重要性质**：

1. **越界不报错**：起止位置会被夹到有效范围，结果是空列表或部分列表。
2. **切片返回新列表**（浅拷贝）：修改切片结果不影响原列表，但切片里的嵌套对象仍是同一批对象。
3. **对 list 切片 = O(k)**，k 是结果长度；`a[:]` 是常用的整体浅拷贝写法。

### 2.2.3 切片赋值：可以改变长度

```python
a = [1, 2, 3, 4, 5]
a[1:3] = [20, 30, 40]     # 替换 2 个元素为 3 个
print(a)                   # [1, 20, 30, 40, 4, 5]

a[1:3] = []                # 等价于删除这两个位置
print(a)                   # [1, 40, 4, 5]

a[1:1] = [99]              # 在位置 1 插入
print(a)                   # [1, 99, 40, 4, 5]

a[len(a):] = [7]           # 等价于 append
print(a)                   # [1, 99, 40, 4, 5, 7]

a[::2] = [0, 0, 0]         # 带步长的切片赋值：右侧长度必须与切片长度一致
print(a)                   # [0, 99, 0, 4, 0, 7]
a[::2] = [1]               # ❌ ValueError: attempt to assign sequence of size 1 to extended slice of size 3
```

> 记忆：**普通切片赋值长度自由（可增可删），扩展切片（带 step）赋值长度必须严格相等**。

### 2.2.4 `del` 删除

```python
a = [0, 1, 2, 3, 4]
del a[2]        # [0, 1, 3, 4]
del a[1:3]      # [0, 4]
del a[::2]      # 删除偶数位
```

`del` 是按位置删除，`remove` 是按值删除，`pop` 是删除并返回，三者不要混。

## 2.3 方法总表（返回值是重点）

| 方法 | 作用 | 返回值 | 异常 / 备注 |
| :--- | :--- | :--- | :--- |
| `append(x)` | 尾部追加一个元素 | `None` | O(1) 摊还 |
| `extend(iterable)` | 尾部追加一批元素 | `None` | 接受任意可迭代对象，O(k) |
| `insert(i, x)` | 在 i 前插入 | `None` | 越界自动夹到两端；O(n) |
| `remove(x)` | 删除第一个等于 x 的元素 | `None` | 不存在 → `ValueError` |
| `pop([i])` | 删除并返回位置 i 的元素 | 被删元素 | 默认 `-1`；空列表/越界 → `IndexError` |
| `clear()` | 清空 | `None` | — |
| `index(x[, start[, stop]])` | 找第一个等于 x 的位置 | `int` | 找不到 → `ValueError` |
| `count(x)` | 统计等于 x 的个数 | `int` | O(n) |
| `sort(*, key=None, reverse=False)` | **原地**排序 | `None` | 元素不可比较 → `TypeError` |
| `reverse()` | **原地**反转 | `None` | — |
| `copy()` | 浅拷贝 | 新 `list` | 等价于 `a[:]` |

**一句话总结**：所有"原地修改"的方法（`append`、`extend`、`insert`、`remove`、`sort`、`reverse`、`clear`）都返回 `None`。这制造了 Python 最经典的错误之一：

```python
a = [3, 1, 2]
a = a.sort()          # ❌ a 变成 None！sort 是原地排序
print(a)              # None
```

正确写法：`a.sort()` 单独一行；或者 `a = sorted(a)`。

## 2.4 增删改查的细节

### 2.4.1 添加

```python
a = [1, 2]
a.append([3, 4])      # [1, 2, [3, 4]] —— 把整个列表当一个元素
a.extend([3, 4])      # [1, 2, [3, 4], 3, 4] —— 拆开追加
a.extend("ab")        # 追加 'a'、'b' 两个字符
a += [5]              # 等价于 extend，但要求右边是可迭代对象
```

- `append` vs `extend`：**加一个 vs 加一批**。
- `extend` 接受任何可迭代对象（tuple、str、dict、生成器、set 都行）；`append` 只加一个对象。
- `a += b` 本质调用 `__iadd__`，对 list 是**原地扩展**；`a = a + b` 会新建列表。

### 2.4.2 插入

```python
a = [1, 2, 3]
a.insert(1, 99)     # [1, 99, 2, 3]
a.insert(0, 0)      # [0, 1, 99, 2, 3]
a.insert(100, 7)    # 越界 → 追加到末尾
a.insert(-1, 8)     # 注意：插在最后一个元素之前 → [0, 1, 99, 2, 3, 8, 7]
```

`insert` 的索引语义是"插入后新元素位于该索引处"；负索引会被换算到 `len + i` 后夹紧，所以 `insert(-1, x)` 是**倒数第二的位置**，不是末尾。

### 2.4.3 删除

```python
a = ['a', 'b', 'c', 'b']

a.remove('b')       # 删除第一个 'b' → ['a', 'c', 'b']
a.remove('z')       # ❌ ValueError: list.remove(x): x not in list
a.pop()             # 'b'，列表变 ['a', 'c']
a.pop(0)            # 'a'
a.pop(10)           # ❌ IndexError
[].pop()            # ❌ IndexError: pop from empty list
a.clear()           # []

del a[0]            # 按位置删
del a[0:2]          # 按切片删
```

- `remove(x)` 按**值**删第一个，O(n)，找不到抛 `ValueError`。
- `pop(i)` 按**位置**删并返回，O(n)（删尾部除外），越界抛 `IndexError`。
- `del a[i]` 按位置删，不返回值；`del a[i:j]` 删一段。
- 三个方法复杂度都是 O(n)，因为要移动后续元素。

### 2.4.4 查找

```python
a = [10, 20, 30, 20]
a.index(20)              # 1
a.index(20, 2)           # 3（从下标 2 开始找）
a.index(99)              # ❌ ValueError
a.count(20)              # 2
20 in a                  # True（O(n)）
a.index(20)              # O(n)
```

`index` 和 `count` 都按 `==` 比较；`in` 也是 O(n)。如果要在循环里频繁判断存在性，换成 set。

### 2.4.5 反转与复制

```python
a = [1, 2, 3]
a.reverse()          # 原地 → None，a 变成 [3, 2, 1]
b = a[::-1]          # 返回新列表，a 不变
c = a.copy()         # 浅拷贝
d = a[:]             # 浅拷贝
e = list(a)          # 浅拷贝
```

`a[::-1]`、`reversed(a)`、`a.reverse()` 的区别：

| 写法 | 是否原地 | 返回 | 复杂度 |
| :--- | :--- | :--- | :--- |
| `a.reverse()` | 是 | `None` | O(n) |
| `a[::-1]` | 否 | 新 list | O(n) |
| `reversed(a)` | 否 | 反向迭代器 | O(1) 创建，遍历 O(n) |

## 2.5 排序：`sort` 与 `sorted`

### 2.5.1 基本用法

```python
a = [3, 1, 2]
a.sort()                 # 原地，a = [1, 2, 3]
b = sorted(a)            # 返回新列表
b = sorted(a, reverse=True)   # 降序
c = sorted("cab")        # ['a', 'b', 'c'] —— 任何可迭代对象都能排

words = ["Apple", "banana", "Cherry"]
sorted(words, key=str.lower)  # ['Apple', 'banana', 'Cherry']
sorted(words, key=len)        # 按长度
```

- `sort` 只能用于 list，原地，返回 `None`。
- `sorted` 接受任意可迭代对象，返回新 list。
- `key` 是**关键字参数**，位置参数写法 `sorted(a, len)` 在 Python 3 会报 `TypeError`（`sorted` 的 `key` 只能按关键字传）。
- **稳定性**：相等 key 的元素保持原相对顺序；`reverse=True` 也保持稳定性（不是简单地把结果倒过来）。

```python
items = [("b", 2), ("a", 2), ("c", 1)]
sorted(items, key=lambda t: t[1])
# [('c', 1), ('b', 2), ('a', 2)] —— 2 分组的原顺序 b 在 a 前
sorted(items, key=lambda t: t[1], reverse=True)
# [('b', 2), ('a', 2), ('c', 1)] —— 2 分组仍然 b 在 a 前
```

### 2.5.2 多关键字与混合升降序

```python
students = [("Tom", 90), ("Amy", 90), ("Bob", 85)]

# 分数降序、同分按名字升序：先按次要关键字排一次，再按主要关键字稳定排序
students.sort(key=lambda s: s[0])          # 先名字升序
students.sort(key=lambda s: s[1], reverse=True)  # 再分数降序（稳定）

# 数值型混合升降序：用负数把"降序"变成"升序"
data = [(3, 1), (1, 2), (3, 0)]
data.sort(key=lambda t: (-t[0], t[1]))     # 第一列降序、第二列升序

# 复杂比较逻辑：functools.cmp_to_key
from functools import cmp_to_key
sorted([3, 1, 2], key=cmp_to_key(lambda x, y: y - x))   # [3, 2, 1]
```

### 2.5.3 排序会失败的场景

```python
sorted([1, "a"])            # ❌ TypeError: '<' not supported between instances of 'str' and 'int'
sorted([None, 1])           # ❌ TypeError
sorted([[1, 2], [3]])       # ✅ 列表之间可以字典序比较
sorted([(1, "a"), (1, 2)])  # ❌ 第二个元素 str 与 int 不能比较
```

排序依赖元素之间可比较（`<`）。混合类型要么统一转换，要么用 `key` 映射成统一类型。

## 2.6 列表推导式

### 2.6.1 基本形式

```python
[x * x for x in range(5)]              # [0, 1, 4, 9, 16]
[x for x in range(10) if x % 2 == 0]   # 带过滤
[(x, y) for x in range(2) for y in range(2)]  # 嵌套循环
# [(0,0), (0,1), (1,0), (1,1)]
[[0] * 3 for _ in range(3)]            # 3x3 矩阵（正确写法）
[x if x > 0 else 0 for x in [-1, 1]]   # 条件表达式（注意 if 的位置不同）
```

**注意 `if` 的两种位置**：

- `[x for x in it if cond]`：**过滤**，不满足的不要；
- `[x if cond else y for x in it]`：**三元表达式**，每个元素都要，产出 x 或 y。

### 2.6.2 嵌套推导式的顺序

```python
matrix = [[1, 2], [3, 4]]
flat = [n for row in matrix for n in row]   # [1, 2, 3, 4]
# 等价于：
# for row in matrix:
#     for n in row:
#         flat.append(n)
```

顺序是"**从外到内，从左到右**"，和写普通嵌套 for 的顺序一致。

### 2.6.3 推导式的作用域（Python 3）

```python
i = 100
result = [i for i in range(3)]
print(i)        # 100 —— 推导式有独立作用域，不会泄漏循环变量

# 生成器表达式同理，而且是惰性的
gen = (x for x in range(3))
print(type(gen))    # <class 'generator'>
print(list(gen))    # [0, 1, 2]
print(list(gen))    # [] —— 生成器只能消费一次
```

> Python 2 的列表推导式会泄漏循环变量，Python 3 不会。考试如果遇到"推导式变量泄漏"的说法，默认按 Python 3 判断。

## 2.7 底层结构：为什么这些操作是 O(1) / O(n)

- CPython 的 list 是一个**连续数组**，保存指向对象的指针，外加 `ob_size`（长度）和已分配容量。
- `append` 在容量足够时 O(1)；容量不足时按约 1.125 倍扩容并整体拷贝，但**摊还**到每次是 O(1)。
- `insert(i, x)` / `pop(i)` / `remove` 需要把 i 之后的元素整体后移/前移，O(n)。
- `a[i]` 直接按偏移寻址，O(1)，且缓存局部性好，遍历很快。
- 每次 `append` 可能触发扩容，因此"已知长度"时用推导式或 `list(range(n))` 通常比循环 append 更快（推导式还省掉方法查找）。

```python
import sys

print(sys.getsizeof([]))        # 56（64 位 CPython 3.13，具体值随版本/平台变化）
print(sys.getsizeof([1, 2, 3])) # 88
a = []
for i in range(1000):
    a.append(i)
print(sys.getsizeof(a))         # 通常大于 1000 * 8，因为预留了容量
```

> 不要用 `sys.getsizeof` 计算"列表内容占用的总内存"——它只算 list 自身（指针数组），不含元素对象。元素是共享引用，不是副本。

## 2.8 拷贝：赋值、浅拷贝、深拷贝

```python
a = [[1], [2]]

b = a                 # ① 赋值：同一个对象，两个名字
c = a.copy()          # ② 浅拷贝
d = a[:]              # ② 浅拷贝
e = list(a)           # ② 浅拷贝

a[0].append(9)
print(b)      # [[1, 9], [2]]  跟着变
print(c)      # [[1, 9], [2]]  外层独立，内层共享 → 也变
print(a is b) # True
print(a is c) # False
```

| 方式 | 新对象？ | 内层元素 | 修改外层（如 `append`） | 修改内层（如 `a[0].append`） |
| :--- | :--- | :--- | :--- | :--- |
| `b = a` | 否 | 共享 | 互相影响 | 互相影响 |
| `copy()` / `a[:]` / `list(a)` | 是 | 共享 | 互不影响 | 互相影响 |
| `copy.deepcopy(a)` | 是 | 递归复制 | 互不影响 | 互不影响 |

二维列表的"独立"写法：

```python
grid = [[0] * 3 for _ in range(3)]   # ✅ 每行独立
grid = [[0] * 3] * 3                 # ❌ 三行是同一个 list
grid[0][0] = 1
# 错误写法的结果：[[1,0,0],[1,0,0],[1,0,0]]
```

## 2.9 常见陷阱清单（考前必看）

1. **`a = a.append(x)`** → `a` 变成 `None`。所有原地方法都返回 `None`。
2. **`a = a.sort()`** → 同上。要返回值用 `sorted(a)`。
3. **`[[0] * 3] * 3`** → 三行共享同一个内层列表。
4. **`a = [1, 2]; b = a; b += [3]`** → `a` 也变成 `[1, 2, 3]`（`+=` 原地扩展）。
5. **函数里 `def f(lst): lst += [1]`** → 会修改调用者的列表；想不改用 `lst = lst + [1]` 或 `lst.copy()`。
6. **边遍历边 `remove`** → 跳过元素：
   ```python
   a = [1, 2, 2, 3]
   for x in a:
       if x == 2:
           a.remove(x)      # 结果 [1, 2, 3]，漏删一个
   # 正确：a = [x for x in a if x != 2]
   ```
7. **`for i in range(len(a))` 中删除元素** → 索引错位；倒序遍历或重建列表。
8. **`a.remove(x)` 找不到抛 `ValueError`**，不是静默失败；不确定时先 `if x in a`。
9. **`a.index(x)` 找不到抛 `ValueError`**；`a.find()` 是字符串方法，list 没有。
10. **`a.pop()` 空列表抛 `IndexError`**；`a.pop(0)` 是 O(n)。
11. **`a[::2] = [...]` 右侧长度必须匹配**，否则 `ValueError`。
12. **`a[::0]` 抛 `ValueError`**（步长不能为 0）。
13. **`[1, 2] == (1, 2)` 是 `False`**：list 与 tuple 是不同类型，不相等。
14. **`sorted()` 不会修改原列表**；`a.sort()` 会，且返回 `None`。
15. **`sum(["a", "b"])` 抛 `TypeError`**（sum 从 0 开始加，字符串不能用 `+` 与 0 相加）；拼接字符串用 `"".join(...)`。
16. **`min([])` / `max([])` 抛 `ValueError`**；`sum([])` 返回 0；`any([])` 是 False、`all([])` 是 True。
17. **`list("abc")` 拆成字符**，不是把整个字符串当一个元素；要 `["abc"]` 直接用字面量。
18. **`a * -1` 得到空列表**；`a * 0` 也是空列表，不报错。
19. **`a[1:3] = "xy"`** 合法（字符串是可迭代对象，被拆成 'x'、'y'），容易与"元素类型必须一致"的直觉冲突。
20. **`in` 对 list 是 O(n)**：`if x in big_list` 放进循环前先考虑 set。
21. **`a += "abc"`** 会追加三个字符（`+=` 对 list 等价 extend），不是追加一个字符串。
22. **`bool([]) is False`**、`if not a` 判断空列表；不要写 `if a == []`（能工作但不 Pythonic，且对 numpy 数组会出错）。
23. **`a.copy()` 只复制一层**：嵌套 list/dict 仍共享。
24. **`a.sort(key=...)` 的 key 只调用一次/元素**，不要在 key 里做有副作用的事。

## 2.10 list 与其他"有序序列"的取舍

| 需求 | 推荐 | 原因 |
| :--- | :--- | :--- |
| 按下标随机访问、尾部增删 | `list` | 数组，O(1) 下标 |
| 频繁在两端增删（队列、滑动窗口） | `collections.deque` | 两端 O(1) |
| 只存数字且在意内存 | `array.array("i")` / `numpy.ndarray` | 紧凑存储、向量化 |
| 固定记录、不可变 | `tuple` / `namedtuple` | 可哈希、语义清晰 |
| 只读、超大、切片多 | `tuple` | 省内存；切片返回自身，不复制 |
| 需要"去重 + 保持顺序" | `dict.fromkeys` + `list` | O(n) 且保序 |

> 经验法则：**默认用 list**；一旦发现"从头部 pop"或"中间频繁插删"，就考虑 deque 或换数据结构，而不是硬扛 O(n²)。

---

# 第 3 章 tuple：不可变序列与被低估的用途

`tuple` 常被当成"不能改的 list"，但它真正的价值是：**可哈希、可解包、语义上是一条固定记录**。函数返回多值、dict 的 `items()`、`zip()`、`enumerate()`、`str.partition()` 产出的都是元组。

## 3.1 创建：括号不重要，逗号才重要

```python
t = (1, 2, 3)
t = 1, 2, 3            # 括号可以省略（逗号才是元组的语法核心）
t = ()                 # 空元组
t = (1,)               # 单元素元组：必须有逗号
t = 1,                 # 同样是单元素元组
t = tuple()            # 空元组
t = tuple([1, 2])      # (1, 2)
t = tuple("abc")       # ('a', 'b', 'c')
t = tuple(range(3))    # (0, 1, 2)
```

### 3.1.1 最容易错的两个写法

```python
a = (1)        # ❌ 这不是元组，就是整数 1（括号只是分组）
print(type(a)) # <class 'int'>

b = (1,)       # ✅ 单元素元组
print(type(b)) # <class 'tuple'>
print(len(b))  # 1

c = (1, 2) * 0  # 空元组 ()
d = (1, 2) * 2  # (1, 2, 1, 2)
```

**记忆**：`()` 是空元组，`(x)` 是 `x`，`(x,)` 才是单元素元组。写单元素元组时永远保留那个逗号。

## 3.2 "不可变"到底是什么意思

```python
t = (1, 2, 3)
t[0] = 9          # ❌ TypeError: 'tuple' object does not support item assignment
del t[0]          # ❌ TypeError
t.append(4)       # ❌ AttributeError: 'tuple' object has no attribute 'append'
```

元组不可变，指的是**元组持有的引用（槽位）不能替换、不能增删**。但元素指向的对象本身可能是可变的：

```python
t = ([1], [2])
t[0].append(9)    # ✅ 合法，变的是 list
print(t)          # ([1, 9], [2])

t = ("abc", [1])
t[0] += "d"       # ❌ TypeError：str 不可变，等价于给元组槽位重新赋值
```

### 3.2.1 经典陷阱：`t[0] += [3]` 既报错又"改成功"

```python
t = ([1], 2)
try:
    t[0] += [3]
except TypeError as e:
    print("TypeError:", e)

print(t)          # ([1, 3], 2) —— 列表已经被原地修改了！
```

原因：`t[0] += [3]` 分两步——先对 `t[0]` 执行 `list.__iadd__`（**原地修改成功**），再试图把结果写回 `t[0]`（元组不允许赋值，抛 `TypeError`）。所以异常发生了，但副作用已经产生。

**结论**：不要用"元组里套 list"来假装不可变；要真正的不可变，元组里也只能放不可变对象。

### 3.2.2 `+=` 对元组是"重新绑定"

```python
t = (1, 2)
print(id(t))
t += (3,)         # 创建新元组，名字 t 重新绑定
print(t, id(t))   # (1, 2, 3)，id 变了
```

同理 `t = t + (3,)` 也是新建。对不可变对象做"修改"，本质都是"新建 + 重新绑定"。

## 3.3 可哈希性：什么时候能当 dict 键

元组可哈希的**前提是所有元素都可哈希**：

```python
hash((1, 2))          # ✅
hash((1, "a"))        # ✅
hash((1, [2]))        # ❌ TypeError: unhashable type: 'list'
hash((1, {"a": 1}))   # ❌ unhashable type: 'dict'

d = {(1, 2): "point"}           # ✅
d[(1, [2])] = "bad"             # ❌ TypeError
s = {(1, 2), (3, 4)}            # ✅ 元组可作 set 元素
```

**应用场景**：

- 二维坐标、区间 `(start, end)` 作缓存键；
- `(user_id, item_id)` 作去重键；
- `functools.lru_cache` 的内部键就是参数元组（因此参数必须可哈希）。

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)
# 缓存键就是 (n,)，n 可哈希
```

> 注意：`1` 与 `1.0` 相等且哈希相同，所以 `(1, 2)` 与 `(1.0, 2.0)` 也是相等的元组，作为键会命中同一个位置。

```python
print((1, 2) == (1.0, 2.0))          # True
print(hash((1, 2)) == hash((1.0, 2.0)))  # True
print({(1, 2): "a"}[(1.0, 2.0)])     # a
```

## 3.4 解包：tuple 最常用的能力

### 3.4.1 基本解包

```python
point = (3, 5)
x, y = point            # x=3, y=5
a, b = b, a             # 交换（右侧先求值成元组，再解包）

for i, ch in enumerate("ab"):
    print(i, ch)

name, _, domain = "user@example.com".partition("@")
# partition 返回三元组 ('user', '@', 'example.com')
```

变量个数必须与元素个数一致，否则：

```python
a, b = (1, 2, 3)   # ❌ ValueError: too many values to unpack
a, b, c = (1, 2)   # ❌ ValueError: not enough values to unpack
```

### 3.4.2 星号解包（一个变量收集剩余元素）

```python
first, *rest = [1, 2, 3, 4]
# first=1, rest=[2, 3, 4]（注意：rest 是 list，不是 tuple）

*init, last = (1, 2, 3, 4)
# init=[1, 2, 3], last=4

a, *mid, b = range(5)
# a=0, mid=[1, 2, 3], b=4

a, *b, c, *d = range(5)   # ❌ SyntaxError：一个赋值目标里最多一个 *
```

### 3.4.3 解包在函数调用与字面量中

```python
def f(a, b, c):
    return a + b + c

args = (1, 2, 3)
f(*args)          # 6 —— 把元组拆成位置参数

print(*args)      # 1 2 3
print([0, *args, 4])       # [0, 1, 2, 3, 4]
print({"x": 1, **{"y": 2}})  # {'x': 1, 'y': 2}

record = ("Tom", 20)
name, age = record         # 最常用的"记录"用法
```

### 3.4.4 嵌套解包

```python
data = (1, (2, 3))
a, (b, c) = data      # a=1, b=2, c=3

pairs = [((1, 2), "a"), ((3, 4), "b")]
for (x, y), label in pairs:
    print(x, y, label)
```

## 3.5 元组的其他操作

```python
t = (1, 2, 2, 3)

t[0]           # 1
t[-1]          # 3
t[1:3]         # (2, 2) —— 切片返回 tuple
t[::-1]        # (3, 2, 2, 1)
len(t)         # 4
t.count(2)     # 2
t.index(2)     # 1
2 in t         # True（O(n)）
min(t), max(t), sum(t)   # 1 3 8

t + (4, 5)     # (1, 2, 2, 3, 4, 5)
t * 2          # 重复
t + [4]        # ❌ TypeError：tuple 只能和 tuple 拼接

sorted(t)      # [1, 2, 2, 3] —— 返回 list！
tuple(sorted(t))  # (1, 2, 2, 3) —— 要元组需显式转
```

### 3.5.1 元组比较是"字典序"

```python
(1, 2) < (1, 3)      # True：先比第一个，相同再比第二个
(1, 2) < (1, 2, 0)   # True：前缀相同，短的更小
(2,) < (10,)         # True：2 < 10
(1, "a") < (1, 2)    # ❌ TypeError：str 与 int 不能比较
```

### 3.5.2 切片与转换的特殊行为

```python
t = (1, 2, 3)
print(t[:] is t)         # True —— 元组切片返回自身（不可变，无需复制）
print(tuple(t) is t)     # True —— tuple(tuple) 返回同一对象
print(tuple([1, 2]))     # (1, 2)

print((), )              # ()
print(tuple())           # ()
```

> 因为不可变，`t[:]`、`tuple(t)` 可以安全地返回原对象，这是 tuple 相对 list 的一个性能优势（大元组切片不复制）。

## 3.6 性能与内存：tuple 比 list 轻

```python
import sys
print(sys.getsizeof([1, 2, 3]))   # 88
print(sys.getsizeof((1, 2, 3)))   # 64
```

- tuple 不需要"预留容量"字段（不可变，长度固定），对象头更小。
- 创建 tuple 通常比 list 快；常量元组还会被编译器**折叠**：

```python
import dis
def f():
    return (1, 2, 3)
# 字节码里直接是常量 (1, 2, 3)，运行时无需重新构造
```

- 空元组 `()` 是**单例**，`() is ()` 为 True。
- **但**：不要为了省内存把"会变的有序集合"塞进 tuple；tuple 的优势是语义（不可变、可哈希、记录），不是一点点内存。

### 3.6.1 什么时候用 tuple，什么时候用 list

| 场景 | 选择 | 理由 |
| :--- | :--- | :--- |
| 一条固定记录：`("Tom", 20, "CS")` | `tuple` | 字段固定、可解包、可哈希 |
| 同类型元素的可变集合 | `list` | 需要增删改、排序 |
| 函数返回多个值 | `tuple` | 天然支持解包 |
| 字典的复合键 | `tuple` | 可哈希（元素都可哈希时） |
| 需要排序/去重后修改 | `list` | tuple 没有 `sort`/`append` |
| 作为 set 元素或 dict 键 | `tuple` / `frozenset` | 不可变且可哈希 |

## 3.7 tuple 相关陷阱清单

1. **`(1)` 是 int，`(1,)` 才是 tuple**。
2. **`t[0] = x` 抛 `TypeError`**；`t += (x,)` 是新建对象，不是原地修改。
3. **元组里的 list 可以改**；`t[0] += [3]` 会先改列表再抛 `TypeError`（副作用已发生）。
4. **含不可哈希元素的元组不能作 dict 键 / set 元素**（如 `(1, [2])`）。
5. **`t[0:2]` 返回 tuple，不是 list**；`list(t[0:2])` 才转列表。
6. **`sorted(t)` 返回 list**，不是 tuple；`t.sort()` 不存在（`AttributeError`）。
7. **`t + [1]` 抛 `TypeError`**，tuple 只能与 tuple 相加；`t + (1,)` 才对。
8. **`tuple("abc")` 拆成字符**。
9. **解包数量必须匹配**，否则 `ValueError`；不确定长度时用 `*rest`。
10. **星号解包收集到的是 list**，不是 tuple。
11. **`t = 1,` 是元组**；但 `return 1,` 也返回元组，别误以为返回了 int。
12. **元组不是"只读列表"的同义词**：需要 `.index()`、`.count()` 之外的操作时，它没有。
13. **`hash(t)` 对相同内容稳定**，但不要依赖具体哈希值（受 `PYTHONHASHSEED` 影响，且跨版本可能变）。
14. **`tuple` 与 `list` 比较永远 `False`**（`[1,2] == (1,2)` 为 False），即使内容"看起来一样"。
15. **大元组切片不复制**（返回自身），所以不要通过切片"拷贝"元组——本来就不需要。
16. **`()` 是单例**，`() is ()` 为 True；但不要用 `is` 比较元组内容。
17. **`tuple` 可以当 `namedtuple` 的基类**，但普通 tuple 没有字段名，可读性差。
18. **`*` 解包函数参数时，位置参数个数必须匹配**，否则 `TypeError`。

---

# 第 4 章 dict：哈希表、视图与合并

`dict` 是 Python 的映射类型：键唯一、按键 O(1) 平均查找、**3.7 起保证插入顺序**。它几乎是 Python 程序里最重要的数据结构——模块命名空间、对象属性、JSON、缓存、计数，底层都离不开 dict 或它的变体。

## 4.1 创建 dict 的 8 种方式

```python
{}                                   # 空字典
{"a": 1, "b": 2}                     # 字面量
dict()                               # 空字典
dict(a=1, b=2)                       # 关键字形式：键必须是合法标识符（字符串）
dict([("a", 1), ("b", 2)])           # 由二元组序列构造
dict(zip(["a", "b"], [1, 2]))        # 由两个序列 zip 构造
dict.fromkeys(["a", "b"], 0)         # 所有键共享同一个值 → {'a': 0, 'b': 0}
{k: v * 2 for k, v in {"a": 1, "b": 2}.items()}   # 字典推导式
{**{"a": 1}, "b": 2}                 # 解包合并
```

**易错点**：

```python
dict(1=2)            # ❌ SyntaxError：关键字必须像标识符
dict([("a", 1), "b"])  # ❌ ValueError：每个元素必须恰好两个
dict({"a": 1}, b=2)  # ✅ {'a': 1, 'b': 2}
```

## 4.2 键的规则：可哈希、唯一、相等即同一键

### 4.2.1 键必须可哈希

```python
d = {}
d["a"] = 1
d[1] = 2
d[(1, 2)] = 3        # ✅ 元组可作键（元素都可哈希）
d[frozenset({1})] = 4  # ✅
d[[1, 2]] = 5        # ❌ TypeError: unhashable type: 'list'
d[{"x": 1}] = 6      # ❌ unhashable type: 'dict'
d[{1, 2}] = 7        # ❌ unhashable type: 'set'
d[(1, [2])] = 8      # ❌ 元组里含 list → 不可哈希
```

### 4.2.2 `1`、`1.0`、`True` 是同一个键

```python
d = {}
d[1] = "int"
d[1.0] = "float"
d[True] = "bool"
print(d)          # {1: 'bool'} —— 键是第一次插入的 1
print(len(d))     # 1
```

原因：`hash(1) == hash(1.0) == hash(True)` 且 `1 == 1.0 == True`。dict 先按哈希定位，再用 `==` 比较，所以三者互相覆盖。

### 4.2.3 一个更隐蔽的细节：NaN 与"同一对象"

```python
nan = float("nan")
d = {nan: 1}
print(d[nan])          # 1 —— 同一个对象，dict 用身份（is）短路判断
print(d[float("nan")]) # ❌ KeyError —— 不同 NaN 对象不相等
```

`nan != nan`，所以两个不同的 NaN 对象不会被认为是同一个键；但用**同一个 NaN 对象**访问是可以命中的，因为 CPython 的查找会先比较对象身份。这个行为属于实现细节，实际代码中不要用 NaN 当键。

### 4.2.4 自定义对象作键：`__hash__` 与 `__eq__` 必须一致

```python
class User:
    def __init__(self, uid):
        self.uid = uid
    def __hash__(self):
        return hash(self.uid)
    def __eq__(self, other):
        return isinstance(other, User) and self.uid == other.uid

d = {User(1): "Tom"}
print(d[User(1)])   # Tom —— 相等且哈希一致 → 命中
```

如果只定义 `__eq__` 不定义 `__hash__`，Python 会把 `__hash__` 设为 `None`，实例**不可哈希**：

```python
class Bad:
    def __eq__(self, other):
        return True
hash(Bad())    # ❌ TypeError: unhashable type: 'Bad'
```

**契约**：`a == b` ⇒ `hash(a) == hash(b)`。违反会导致"明明相等的键却查不到"。

## 4.3 访问：`[]`、`get`、`setdefault` 的三条路

```python
d = {"a": 1}

d["a"]                 # 1
d["b"]                 # ❌ KeyError: 'b'

d.get("a")             # 1
d.get("b")             # None —— 不存在也不报错
d.get("b", 0)          # 0 —— 指定默认值
print(d)               # {'a': 1} —— get 不会插入

d.setdefault("a", 100) # 1 —— 键已存在，返回原值，不修改
d.setdefault("b", 100) # 100 —— 键不存在，插入并返回
print(d)               # {'a': 1, 'b': 100}
```

| 写法 | 键不存在时 | 是否插入 | 适用场景 |
| :--- | :--- | :--- | :--- |
| `d[k]` | 抛 `KeyError` | — | 确定键存在 |
| `d.get(k)` | 返回 `None` | 否 | 可选读取 |
| `d.get(k, default)` | 返回 default | 否 | 带默认值读取 |
| `d.setdefault(k, default)` | 插入 default 并返回 | 是 | 初始化嵌套结构 |
| `defaultdict(factory)[k]` | 调用 factory 插入并返回 | 是 | 分组、计数（第 6 章） |

**关键区别**：`get` 只读；`setdefault` 会写；`defaultdict` 连 `d[k]` 这种读操作都会写。

### 4.3.1 嵌套字典的经典写法

```python
# 用 setdefault 构建"键 -> 列表"
groups = {}
for name, dept in [("Tom", "CS"), ("Amy", "Math"), ("Bob", "CS")]:
    groups.setdefault(dept, []).append(name)
print(groups)   # {'CS': ['Tom', 'Bob'], 'Math': ['Amy']}

# 等价的 defaultdict 写法
from collections import defaultdict
groups = defaultdict(list)
for name, dept in [("Tom", "CS"), ("Amy", "Math"), ("Bob", "CS")]:
    groups[dept].append(name)
```

> 注意 `d.get(dept, []).append(name)` 是**错误**的：它往一个临时空列表里 append，结果直接丢弃，字典不会更新。

### 4.3.2 `setdefault` 的默认值总会被求值

```python
def expensive():
    print("被调用了")
    return []

d = {"k": 1}
d.setdefault("k", expensive())   # 即使 k 已存在，expensive() 也会执行！
```

`defaultdict(list)` 的工厂函数只有在键缺失时才调用，这是它相对 `setdefault` 的一个实际优势。

## 4.4 增、删、改、合并

### 4.4.1 新增 / 修改

```python
d = {"a": 1}
d["b"] = 2            # 新增
d["a"] = 10           # 修改：位置不变
print(d)              # {'a': 10, 'b': 2}

d.update({"c": 3})    # 批量更新
d.update([("d", 4)])  # 接受二元组序列
d.update(e=5)         # 关键字形式
```

**更新已有键不会改变插入顺序**：

```python
d = {"a": 1, "b": 2}
d["a"] = 99
print(list(d))        # ['a', 'b'] —— a 仍在最前
```

**删除后重新插入会排到末尾**：

```python
d = {"a": 1, "b": 2}
del d["a"]
d["a"] = 3
print(list(d))        # ['b', 'a']
```

### 4.4.2 删除

```python
d = {"a": 1, "b": 2, "c": 3}

d.pop("a")            # 1，字典剩 {'b': 2, 'c': 3}
d.pop("z")            # ❌ KeyError
d.pop("z", 0)         # 0 —— 带默认值不报错
del d["b"]            # 按 key 删除；不存在 → KeyError
d.popitem()           # ('c', 3) —— 弹出最后一个（LIFO）
d.clear()             # {}
```

- `pop(key[, default])` 返回被删的值；
- `del d[key]` 不返回值，键不存在抛 `KeyError`；
- `popitem()` 在 3.7+ 弹出**最后插入**的键值对（LIFO），返回 `(key, value)`；空字典抛 `KeyError`。

### 4.4.3 合并：四种写法与优先级

```python
a = {"x": 1, "y": 2}
b = {"y": 20, "z": 30}

# ① 解包合并（3.5+）：后面的覆盖前面的
c = {**a, **b}        # {'x': 1, 'y': 20, 'z': 30}

# ② | 运算符（3.9+）：右操作数优先
c = a | b             # {'x': 1, 'y': 20, 'z': 30}

# ③ |= 原地合并（3.9+）：等价于 a.update(b)
a |= b                # a 变成 {'x': 1, 'y': 20, 'z': 30}

# ④ update：原地修改，返回 None
a.update(b)
```

| 写法 | 是否原地 | 返回值 | 冲突时谁赢 | 版本 |
| :--- | :--- | :--- | :--- | :--- |
| `{**a, **b}` | 否 | 新 dict | 后者 | 3.5+ |
| `a \| b` | 否 | 新 dict | 后者 | 3.9+ |
| `a \|= b` | 是 | 语句无值（表达式层返回 `a` 自身） | 后者 | 3.9+ |
| `a.update(b)` | 是 | `None` | 后者 | 一直都有 |

**合并顺序决定覆盖关系**，这是考试高频点：`{**a, **b}` 中 `b` 覆盖 `a`。

### 4.4.4 `fromkeys` 的共享值陷阱

```python
d = dict.fromkeys(["a", "b"], [])
print(d)                 # {'a': [], 'b': []}
d["a"].append(1)
print(d)                 # {'a': [1], 'b': [1]} —— 两个键指向同一个列表！

# 正确写法：每个键一个独立列表
d = {k: [] for k in ["a", "b"]}
```

`fromkeys` 只是把**同一个值对象**赋给所有键；值不可变时没问题（如 `0`、`None`），可变时就会共享。

## 4.5 视图：`keys()` / `values()` / `items()`

```python
d = {"a": 1, "b": 2}

print(d.keys())     # dict_keys(['a', 'b'])
print(d.values())   # dict_values([1, 2])
print(d.items())    # dict_items([('a', 1), ('b', 2)])
```

### 4.5.1 视图是"动态窗口"，不是快照

```python
ks = d.keys()
print(list(ks))     # ['a', 'b']
d["c"] = 3
print(list(ks))     # ['a', 'b', 'c'] —— 视图跟着字典变
```

### 4.5.2 视图的特性

| 特性 | keys | values | items |
| :--- | :---: | :---: | :---: |
| 可迭代 | ✅ | ✅ | ✅ |
| 支持 `len` | ✅ | ✅ | ✅ |
| 支持 `in` | ✅（O(1)） | ✅（O(n)） | ✅ |
| 可下标 `[0]` | ❌ | ❌ | ❌ |
| 支持集合运算 `& \| - ^` | ✅ | ❌ | ✅ |
| 动态反映字典变化 | ✅ | ✅ | ✅ |

```python
d = {"a": 1, "b": 2}
d.keys() & {"b", "c"}       # {'b'}
d.items() & {("a", 1)}      # {('a', 1)}
d.values() & {1}            # ❌ TypeError: unsupported operand type(s) for &: 'dict_values' and 'set'
```

> `d.keys()` 之所以能做集合运算，是因为键的集合语义天然支持；`values()` 的值可能重复且不可哈希，所以不支持。

### 4.5.3 遍历的三种常用姿势

```python
d = {"a": 1, "b": 2}

for k in d:                 # 只遍历键
    print(k)

for k, v in d.items():      # 同时要键和值（推荐）
    print(k, v)

for v in d.values():        # 只要值
    print(v)

for k in sorted(d):         # 按键排序后遍历
    print(k)
```

## 4.6 顺序：插入有序，但不是"排序"

- Python 3.7+ 语言规范保证 dict 迭代顺序 = **插入顺序**（3.6 是 CPython 实现细节）。
- 修改已有键**不改变**位置；删除再插入会移到末尾。
- "有序"不等于"自动排序"：要按 key 排序，用 `sorted(d)` 或 `sorted(d.items())`。

```python
d = {"banana": 2, "apple": 1}
print(list(d))                        # ['banana', 'apple'] —— 插入序
print(sorted(d))                      # ['apple', 'banana'] —— 字母序
print(sorted(d.items(), key=lambda kv: kv[1]))  # 按值排序
print(dict(sorted(d.items())))        # 重建一个按 key 排序的 dict
```

`reversed(d)`（3.8+）按插入顺序反向迭代键：

```python
print(list(reversed({"a": 1, "b": 2})))   # ['b', 'a']
```

## 4.7 遍历时修改字典：什么时候报错

```python
d = {"a": 1, "b": 2}

for k in d:
    d[k] = d[k] + 1      # ✅ 修改已有键的值：合法

for k in d:
    d["c"] = 3           # ❌ RuntimeError: dictionary changed size during iteration

for k in d:
    del d[k]             # ❌ RuntimeError
```

**规则**：遍历过程中不能改变字典的**大小**（增/删键），但修改已有键的值是允许的。
需要增删时，先复制键：`for k in list(d): ...`，或先收集要删的键再统一删。

## 4.8 底层：哈希表为什么快

- dict 底层是**开放寻址的哈希表**：用 `hash(key)` 定位槽位，冲突时按探测序列找下一个空位。
- 平均查找/插入/删除 O(1)；极端冲突时最坏 O(n)。
- 装载因子过高时扩容（rehash），扩容是 O(n)，但摊还到每次插入仍是 O(1)。
- 字符串的哈希值会被缓存，重复哈希同一个 str 很快。
- dict 的内存开销比 list 大：每个键值对要存哈希、键指针、值指针；空 dict 也占几十字节。

```python
import sys
print(sys.getsizeof({}))            # 64（3.13，具体值随版本变化）
print(sys.getsizeof({"a": 1}))      # 184 左右
print(sys.getsizeof({"a": 1, "b": 2}))
```

**工程含义**：

- 用 dict 做"存在性判断"和"按键取值"非常快；
- 但 dict 比 list/set 更占内存，键很多时要注意；
- 不要依赖 `hash()` 的具体数值，它可能随 `PYTHONHASHSEED` 变化（str/bytes 的哈希默认随机化）。

## 4.9 dict 相关陷阱清单

1. **`d["missing"]` 抛 `KeyError`**；不确定时用 `get`。
2. **`d.get(k, [])` 返回的列表是临时的**，append 不会写回字典。
3. **`d.setdefault(k, expensive())` 的默认值总会被求值**，即使键已存在。
4. **`dict.fromkeys(keys, [])` 所有键共享同一个列表**。
5. **遍历时增删键抛 `RuntimeError`**；改值可以。
6. **`1`、`1.0`、`True` 是同一个键**，容易造成"键数量对不上"。
7. **list/dict/set 不能作键**；含 list 的 tuple 也不能。
8. **定义了 `__eq__` 却没定义 `__hash__` 的对象不可哈希**。
9. **`d.copy()` 是浅拷贝**，嵌套 dict/list 仍共享。
10. **`d.pop(k)` 键不存在抛 `KeyError`**；给默认值才安全。
11. **`d.popitem()` 在 3.7+ 是 LIFO**（弹最后一个），不是随机。
12. **`d.items()` 返回视图，不是列表**；要列表用 `list(d.items())`。
13. **`values()` 不能做集合运算**；`keys()`、`items()` 可以。
14. **`d.keys()` 不能下标**（`d.keys()[0]` 抛 `TypeError`）。
15. **`{**a, **b}` 中 b 覆盖 a**；顺序写反结果不同。
16. **`d["x"] += 1` 在键不存在时抛 `KeyError`**，不是从 0 开始。
17. **dict 有序 ≠ 排序**；按 key 排序要 `sorted`。
18. **删除再插入会改变位置**（排到末尾），可能影响依赖顺序的逻辑。
19. **`d == OrderedDict(...)` 与两个 OrderedDict 比较的规则不同**（详见第 6 章）。
20. **`dict` 的 `in` 只检查键**，不检查值；检查值要 `x in d.values()`（O(n)）。
21. **`d1 | d2` 需要 3.9+**，旧版本用 `{**d1, **d2}` 或 `update`。
22. **`for k, v in d:` 是错的**，必须 `d.items()`；直接遍历 dict 得到的是键。
23. **`dict` 的键顺序在 3.6 是 CPython 实现细节，3.7+ 才是语言保证**；跨版本兼容代码不要依赖 3.5 及更早的顺序。
24. **`json.dumps` 默认不保证键顺序**（除非 `sort_keys` 或依赖 dict 插入序），不要把它当排序工具。

## 4.10 dict 家族的选择

| 容器 | 与 dict 的关系 | 额外能力 | 典型场景 |
| :--- | :--- | :--- | :--- |
| `dict` | 本体 | 插入有序、`\|` 合并 | 通用映射 |
| `defaultdict` | dict 子类 | 缺失键自动生成默认值 | 分组、计数、图邻接表 |
| `Counter` | dict 子类 | 计数、`most_common`、算术 | 词频、投票、Top-N |
| `OrderedDict` | dict 子类 | `move_to_end`、顺序敏感的相等性 | LRU、需要显式顺序操作 |
| `ChainMap` | 非 dict 子类（Mapping） | 多层映射叠加查找 | 多层配置、作用域链 |
| `MappingProxyType` | 只读视图 | 禁止修改 | 对外暴露只读配置 |

> 3.7+ 普通 dict 也有序了，但 `OrderedDict` 仍有存在价值：`move_to_end()`、`popitem(last=False)`、两个 OrderedDict 比较时**顺序敏感**。详见第 6 章。

---

# 第 5 章 set 与 frozenset：集合运算与去重

`set` 是**可变、无序、元素唯一**的哈希集合，平均 O(1) 完成成员判断和增删。它最常被用来做三件事：去重、成员判断、集合运算（并/交/差/对称差）。`frozenset` 是它的不可变、可哈希版本。

## 5.1 创建：空集合必须用 `set()`

```python
s = set()                 # ✅ 空集合
s = {}                    # ❌ 这是空字典，不是空集合！
print(type({}))           # <class 'dict'>

s = {1, 2, 3}             # 字面量
s = set([1, 1, 2, 3])     # {1, 2, 3} —— 自动去重
s = set("hello")          # {'h', 'e', 'l', 'o'} —— 按字符拆分
s = set(range(3))         # {0, 1, 2}
s = {x * 2 for x in range(3)}   # {0, 2, 4} —— 集合推导式
s = frozenset({1, 2})     # 不可变集合
```

> **高频易错**：`s = {}` 得到的是 dict；想建空集合写 `set()`。如果对 `{}` 调用 `.add()`，会得到 `AttributeError: 'dict' object has no attribute 'add'`。

## 5.2 元素规则：必须可哈希

```python
s = {1, "a", (1, 2), frozenset({3})}   # ✅
s = {[1, 2]}          # ❌ TypeError: unhashable type: 'list'
s = {{"a": 1}}        # ❌ unhashable type: 'dict'
s = {{1, 2}}          # ❌ unhashable type: 'set'
s = {(1, [2])}        # ❌ 元组含 list → 不可哈希

s = set()
s.add([1])            # ❌ TypeError
s.add((1, 2))         # ✅
```

**去重原理**：set 内部是哈希表；插入元素时先算 `hash`，再用 `==` 判断是否已存在。所以：

```python
{1, 1.0, True}        # {1} —— 三者相等且哈希相同
len({1, 1.0, True})   # 1
```

**`frozenset` 可以放进 set、可以作 dict 键**，因为它不可变且可哈希：

```python
s = {frozenset({1, 2}), frozenset({3})}
d = {frozenset({1, 2}): "pair"}
```

### 5.2.1 NaN 的特殊情况

```python
nan = float("nan")
print(nan == nan)           # False
print(nan in {nan})         # True —— 同一个对象，身份短路
print(float("nan") in {nan})# False —— 不同对象且不相等
len({nan, nan})             # 1（同一个对象）
len({float("nan"), float("nan")})  # 2（两个不同对象，互不相等）
```

结论：**不要用 NaN 做集合元素做去重**，行为反直觉且依赖对象身份。

## 5.3 增删改查

```python
s = {1, 2, 3}

s.add(4)          # {1, 2, 3, 4}
s.remove(4)       # {1, 2, 3}；不存在 → KeyError
s.discard(99)     # 不存在也不报错
s.pop()           # 随机弹出一个元素（不可指定）；空集合 → KeyError
s.clear()         # set()
```

| 方法 | 作用 | 元素不存在时 | 返回值 |
| :--- | :--- | :--- | :--- |
| `add(x)` | 加入一个元素 | 已存在则无操作 | `None` |
| `remove(x)` | 删除指定元素 | **`KeyError`** | `None` |
| `discard(x)` | 删除指定元素 | 静默忽略 | `None` |
| `pop()` | 弹出一个元素 | 空集合 **`KeyError`** | 被弹出的元素 |
| `clear()` | 清空 | — | `None` |

**注意**：`pop()` 不能指定索引（set 无序、无下标），弹出的是哪个元素由实现决定。需要"删最小/最大"时用 `min(s)` / `max(s)` 再 `remove`（O(n)），或改用 `heapq`。

### 5.3.1 访问与遍历

```python
s = {3, 1, 2}

s[0]           # ❌ TypeError: 'set' object is not subscriptable
s[0:2]         # ❌ TypeError

for x in s:    # ✅ 只能遍历
    print(x)   # 顺序不保证是 3,1,2

print(2 in s)  # True，平均 O(1)
print(len(s))  # 3
print(sorted(s))   # [1, 2, 3] —— 排序后得到 list
print(list(s))     # 顺序任意
```

**set 无序的具体含义**：迭代顺序由哈希决定，不能假设它等于插入顺序、也不保证跨进程一致。需要顺序时显式 `sorted()`，或者用"去重保序"方案。

## 5.4 集合运算：并、交、差、对称差

```python
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}

a | b     # {1, 2, 3, 4, 5, 6}   并集 union
a & b     # {3, 4}               交集 intersection
a - b     # {1, 2}               差集 difference（在 a 不在 b）
a ^ b     # {1, 2, 5, 6}         对称差 symmetric_difference（只在其中一个）
```

### 5.4.1 运算符 vs 方法：返回值与原地修改

| 运算符 | 方法 | 是否原地 | 运算符表达式的值 | 方法返回值 |
| :--- | :--- | :--- | :--- | :--- |
| `a \| b` | `a.union(b)` | 否 | 新集合 | 新集合 |
| `a & b` | `a.intersection(b)` | 否 | 新集合 | 新集合 |
| `a - b` | `a.difference(b)` | 否 | 新集合 | 新集合 |
| `a ^ b` | `a.symmetric_difference(b)` | 否 | 新集合 | 新集合 |
| `a \|= b` | `a.update(b)` | **是** | `a` 自身（语句本身无值） | `None` |
| `a &= b` | `a.intersection_update(b)` | **是** | `a` 自身（语句本身无值） | `None` |
| `a -= b` | `a.difference_update(b)` | **是** | `a` 自身（语句本身无值） | `None` |
| `a ^= b` | `a.symmetric_difference_update(b)` | **是** | `a` 自身（语句本身无值） | `None` |

```python
a = {1, 2, 3}
b = {3, 4}

c = a | b            # c 是新集合，a 不变
a |= b               # a 变成 {1, 2, 3, 4}，原地修改
a.update(b)          # 同上；返回 None

x = {1, 2}
print(x.union({3}))  # {1, 2, 3}；x 不变
print(x)             # {1, 2}
```

**方法可以接受任意可迭代对象，运算符要求两边都是集合类型**：

```python
a = {1, 2, 3}
a.union([3, 4])          # ✅ {1, 2, 3, 4}
a | [3, 4]               # ❌ TypeError: unsupported operand type(s) for |: 'set' and 'list'

a.intersection([2, 3])   # ✅
a.difference([1])        # ✅

# 方法还能一次传多个
a.union([4], [5])        # ✅ {1, 2, 3, 4, 5}
```

### 5.4.2 运算结果类型：看左操作数

```python
print(type({1} | {2}))                 # <class 'set'>
print(type({1} | frozenset({2})))      # <class 'set'>
print(type(frozenset({1}) | {2}))      # <class 'frozenset'>
print(type(frozenset({1}) | frozenset({2})))  # <class 'frozenset'>
```

规则：**结果类型跟随左操作数**。`frozenset` 在左边，结果就是 `frozenset`。

### 5.4.3 子集、超集、不相交

```python
a = {1, 2}
b = {1, 2, 3}

a <= b          # True：子集
a < b           # True：真子集
b >= a          # True：超集
b > a           # True：真超集
a.issubset(b)   # True，参数可以是任意可迭代对象
b.issuperset(a) # True
a.isdisjoint({4, 5})   # True：没有交集
a.isdisjoint({2, 4})   # False
```

- `<=` / `<` 是**子集**语义，不是数值比较；`<` 表示真子集（不相等）。
- `issubset` / `issuperset` 接受任意可迭代对象（如 list），运算符要求集合类型。
- `isdisjoint` 判断"交集为空"，也可接受任意可迭代对象。

```python
{1, 2} <= [1, 2, 3]     # ❌ TypeError（运算符要求集合）
{1, 2}.issubset([1, 2, 3])   # ✅ True
```

### 5.4.4 `frozenset` 的"原地"运算其实是重新绑定

```python
f = frozenset({1, 2})
print(id(f))
f |= {3}            # frozenset 不可变，"|=" 生成新对象并重新绑定
print(f, id(f))     # frozenset({1, 2, 3})，id 变了

s = {1, 2}
print(id(s))
s |= {3}            # set 原地修改
print(s, id(s))     # {1, 2, 3}，id 不变
```

## 5.5 去重与"去重且保序"

```python
nums = [3, 1, 3, 2, 1]

# ① set 去重：快，但丢失顺序
unique = list(set(nums))        # 顺序任意

# ② dict.fromkeys 去重保序（推荐，O(n)）
unique = list(dict.fromkeys(nums))   # [3, 1, 2]

# ③ 手写循环（理解原理用）
seen = set()
unique = []
for x in nums:
    if x not in seen:
        seen.add(x)
        unique.append(x)
```

三种方式的选择：

| 需求 | 写法 | 复杂度 |
| :--- | :--- | :--- |
| 只去重、不在乎顺序 | `list(set(x))` / `set(x)` | O(n) |
| 去重且保持首次出现顺序 | `list(dict.fromkeys(x))` | O(n) |
| 去重且需要自定义"相等"规则 | 手写循环 + `seen` | O(n) |
| 去重后排序 | `sorted(set(x))` | O(n log n) |

> 元素不可哈希（如 list、dict）时不能用 set 去重：要么转成可哈希形式（如 tuple），要么用 `O(n²)` 的循环比较。

## 5.6 性能与内存

| 操作 | 平均复杂度 | 最坏 |
| :--- | :--- | :--- |
| `x in s` | O(1) | O(n) |
| `s.add(x)` | O(1) | O(n)（触发扩容时摊还） |
| `s.remove(x)` / `discard` | O(1) | O(n) |
| `s.pop()` | O(1) | O(n) |
| `a \| b` | O(len(a) + len(b)) | — |
| `a & b` | O(min(len(a), len(b))) | — |
| `a - b` | O(len(a)) | — |
| `a ^ b` | O(len(a) + len(b)) | — |
| `a <= b` | O(len(a)) | — |
| 遍历 | O(n) | — |

```python
import sys
print(sys.getsizeof(set()))       # 216（3.13，空 set 也不小）
print(sys.getsizeof({1, 2, 3}))
```

**内存换时间**：set 比 list 占内存多（哈希表 + 每个元素），但成员判断快得多。数据量小（几十个）时差异可以忽略；上万级循环里的 `in` 判断，set 是质变。

## 5.7 set 相关陷阱清单

1. **`{}` 是空字典**，空集合必须 `set()`。
2. **set 不能下标/切片**：`s[0]` 抛 `TypeError`；需要顺序先 `sorted(s)`。
3. **set 无序**，`list(s)` 的顺序不能依赖，`pop()` 弹出的也不是"第一个"。
4. **`remove` 不存在抛 `KeyError`**，`discard` 才静默。
5. **`pop()` 空集合抛 `KeyError`**。
6. **元素必须可哈希**：list/dict/set 不能放入；含 list 的 tuple 也不行。
7. **`1`、`1.0`、`True` 在 set 中只保留一个**。
8. **NaN 去重行为反直觉**（不同 NaN 对象不相等），不要依赖。
9. **遍历时 `add`/`remove` 抛 `RuntimeError: Set changed size during iteration`**；先 `for x in list(s)`。
10. **`update` / `intersection_update` 等原地方法返回 `None`**，`a = a.update(b)` 会把 a 变成 None。
11. **运算符要求集合类型，方法接受任意可迭代对象**：`s | [1]` 报错，`s.union([1])` 可以。
12. **`|` 与 `|=` 不同**：前者返回新集合，后者原地修改（frozenset 则重新绑定）。
13. **`<=` 是子集不是"小于等于"**；`{1,2} < {1,2}` 为 False（真子集）。
14. **`set` 不可哈希**，不能作 dict 键；需要时用 `frozenset`。
15. **`frozenset` 没有 `add`/`remove`**，只能做运算产生新对象。
16. **`sorted(s)` 返回 list**，不是 set；`s.sort()` 不存在。
17. **`set` 的 `in` 是 O(1) 平均**，但最坏 O(n)；不要用它处理恶意构造的哈希冲突数据。
18. **`set("abc")` 拆成字符**，不是 `{"abc"}`。
19. **集合推导式 `{x for ...}` 得到 set；`{k: v for ...}` 才是 dict**，两者用错会得到意外类型。
20. **`list(set(x))` 不保序**；需要保序用 `dict.fromkeys`。

## 5.8 set 的典型应用

```python
# 1) 快速成员判断
valid = {"GET", "POST", "PUT", "DELETE"}
if method in valid: ...

# 2) 去重
unique_users = set(users)

# 3) 找共同好友 / 差集
common = set(a_friends) & set(b_friends)
only_a = set(a_friends) - set(b_friends)

# 4) 判断两个列表是否有交集
if set(a).isdisjoint(b): ...   # 无交集

# 5) 权限/标签比较
required <= user_perms        # 是否拥有全部所需权限

# 6) 集合运算实现"差异检测"
added = new_keys - old_keys
removed = old_keys - new_keys
```

> 注意第 5 条：`required <= user_perms` 只有在 `user_perms` 是集合类型（`set` / `frozenset`）时才能用运算符；如果是 list，改用 `required.issubset(user_perms)`。

---

# 第 6 章 collections：标准库里的"进阶容器"

`collections` 提供了一批在 dict / list / tuple / set 基础上针对特定场景优化的容器。考试和实际开发中最常用的是 `Counter`、`defaultdict`、`deque`、`namedtuple`，进阶一点是 `OrderedDict`、`ChainMap` 和 `UserDict/UserList/UserString`。

## 6.0 总览：什么时候用哪个

| 容器 | 一句话定位 | 底层/基类 | 最典型场景 |
| :--- | :--- | :--- | :--- |
| `Counter` | 计数器（multiset） | `dict` 子类 | 词频、投票、Top-N |
| `defaultdict` | 带默认值工厂的字典 | `dict` 子类 | 分组、计数、图邻接表 |
| `OrderedDict` | 对顺序有额外操作的字典 | `dict` 子类 | LRU、需要 `move_to_end` |
| `deque` | 双端队列 | 块状双向链表 | 队列、栈、滑动窗口 |
| `namedtuple` | 带字段名的元组 | `tuple` 子类 | 轻量记录、返回值 |
| `ChainMap` | 多层映射叠加 | `MutableMapping` | 多层配置、作用域链 |
| `UserDict` | 可安全继承的字典 | `MutableMapping` | 定制字典行为 |
| `UserList` | 可安全继承的列表 | `MutableSequence` | 定制列表行为 |
| `UserString` | 可安全继承的字符串 | `Sequence` | 定制字符串行为 |

---

## 6.1 Counter：计数器

`Counter` 是 dict 的子类，键是元素、值是计数。它把"统计出现次数"这件事变成了一等公民。

### 6.1.1 创建

```python
from collections import Counter

Counter()                          # 空计数器
Counter("aabbbc")                  # Counter({'b': 3, 'a': 2, 'c': 1})
Counter([1, 2, 2, 3, 3, 3])        # Counter({3: 3, 2: 2, 1: 1})
Counter({"a": 2, "b": 1})          # 从映射创建：值就是计数
Counter(a=2, b=1)                  # 关键字形式
Counter("aab") + Counter("abb")    # 计数相加
```

### 6.1.2 访问不存在的键：返回 0，且不插入

```python
c = Counter("aab")
print(c["a"])     # 2
print(c["z"])     # 0 —— 不抛 KeyError，也不插入
print(c)          # Counter({'a': 2, 'b': 1})，仍然没有 z
print("z" in c)   # False
```

这个特性让计数代码非常简洁：

```python
c = Counter()
for ch in "hello":
    c[ch] += 1        # 缺失键按 0 处理，不会 KeyError
print(c)              # Counter({'l': 2, 'h': 1, 'e': 1, 'o': 1})
```

> 对比普通 dict：`d[ch] += 1` 在键不存在时抛 `KeyError`，必须先 `d[ch] = d.get(ch, 0) + 1`。

### 6.1.3 核心方法

| 方法 | 作用 | 返回值 |
| :--- | :--- | :--- |
| `most_common(n=None)` | 按计数降序取前 n 个 | `list[tuple]`，`n=None` 取全部 |
| `elements()` | 按计数重复元素 | 迭代器（计数 ≤ 0 的元素被忽略） |
| `update(iterable/mapping)` | **累加**计数 | `None` |
| `subtract(iterable/mapping)` | **减去**计数（可为负） | `None` |
| `total()` | 所有计数之和（3.10+） | `int` |
| `clear()` | 清空 | `None` |
| `copy()` | 浅拷贝 | `Counter` |

```python
c = Counter("aabbbc")
print(c.most_common())      # [('b', 3), ('a', 2), ('c', 1)]
print(c.most_common(2))     # [('b', 3), ('a', 2)]
print(c.most_common(0))     # []

print(list(c.elements()))   # ['a', 'a', 'b', 'b', 'b', 'c']
print(c.total())            # 6（3.10+）

c.update("aab")             # 累加
print(c)                    # Counter({'a': 4, 'b': 4, 'c': 1})
c.subtract("b")             # 减去
print(c)                    # Counter({'a': 4, 'b': 3, 'c': 1})
```

**`most_common` 的并列顺序**：计数相同时保持元素第一次出现的顺序（稳定排序）。

```python
Counter("abcabc").most_common()   # [('a', 2), ('b', 2), ('c', 2)]
```

### 6.1.4 `update` 与 `+` 的区别（高频考点）

```python
c = Counter("aab")
c.update("b")        # 计数直接相加：a=2, b=2
print(c)

c2 = Counter("aab")
c3 = c2 + Counter("b")   # 返回新 Counter，a=2, b=2
print(c3)
```

| 写法 | 是否修改原对象 | 计数 ≤ 0 时 |
| :--- | :--- | :--- |
| `c.update(x)` | 是 | 保留负值/零值 |
| `c.subtract(x)` | 是 | 保留负值 |
| `c + d` | 否 | **丢弃计数 ≤ 0 的项** |
| `c - d` | 否 | **只保留正计数** |
| `c & d` | 否 | 取较小值（min） |
| `c \| d` | 否 | 取较大值（max） |
| `+c`（一元） | 否 | 丢弃计数 ≤ 0 的项 |

```python
a = Counter(a=2, b=1)
b = Counter(a=1, b=3)

print(a + b)   # Counter({'b': 4, 'a': 3})
print(a - b)   # Counter({'a': 1}) —— b 的负计数被丢弃
print(a & b)   # Counter({'a': 1, 'b': 1})
print(a | b)   # Counter({'b': 3, 'a': 2})
print(+Counter(a=-1, b=2))   # Counter({'b': 2})
```

> 考试常考：`Counter({'a': 1}) - Counter({'a': 2})` 得到的是**空 Counter**，不是 `{'a': -1}`；要保留负数用 `subtract`。

### 6.1.5 计数为 0 的键与相等性

```python
c = Counter("aab")
c["z"] = 0
print("z" in c)      # True —— 键确实存在
print(c == Counter("aab"))   # 3.10+：True —— 相等性忽略 0 计数
```

在 Python 3.10+ 中，`Counter` 的相等性把"缺失"和"计数为 0"视为相同；3.9 及更早版本按 dict 的相等性比较（上面的例子会是 False）。写兼容代码时不要依赖这个细节。

### 6.1.6 常见坑

1. `c["missing"]` 返回 0 但**不插入**；`c["missing"] += 1` 会插入。
2. `most_common(n)` 返回 list，不是 Counter；`dict(c.most_common(3))` 可转回字典。
3. `elements()` 返回迭代器且**忽略计数 ≤ 0 的项**；`list(c.elements())` 才是列表。
4. `update` 是**累加**，不是替换；想覆盖用 `c[k] = v`。
5. `+`/`-` 会**丢弃非正计数**，与 `update`/`subtract` 不同。
6. Counter 是 dict 子类，`isinstance(c, dict)` 为 True，`json.dumps(c)` 可用。
7. `Counter` 的键仍然必须可哈希。
8. `Counter` 不保证"计数相同"时的顺序是插入序以外的规则（稳定排序，按首次出现）。

---

## 6.2 defaultdict：带默认值工厂的字典

`defaultdict` 在访问不存在的键时，调用 `default_factory` 生成默认值并插入，然后返回它。这是"分组 / 计数 / 邻接表"最顺手的工具。

### 6.2.1 基本用法

```python
from collections import defaultdict

dd = defaultdict(int)          # 默认值 0
dd["a"] += 1
dd["b"] += 1
dd["a"] += 1
print(dd)                      # defaultdict(<class 'int'>, {'a': 2, 'b': 1})

dd = defaultdict(list)         # 默认值 []
dd["fruits"].append("apple")
dd["fruits"].append("banana")
print(dd)                      # defaultdict(<class 'list'>, {'fruits': ['apple', 'banana']})

dd = defaultdict(set)
dd["a"].add(1)
print(dd)                      # defaultdict(<class 'set'>, {'a': {1}})
```

### 6.2.2 分组：与 `setdefault` 对比

```python
pairs = [("Tom", "CS"), ("Amy", "Math"), ("Bob", "CS")]

# defaultdict 写法
groups = defaultdict(list)
for name, dept in pairs:
    groups[dept].append(name)

# setdefault 写法
groups2 = {}
for name, dept in pairs:
    groups2.setdefault(dept, []).append(name)
```

两者结果相同。`defaultdict` 更简洁，但 `setdefault` 的默认值**每次都会求值**，`defaultdict` 的工厂只在缺失时调用。

### 6.2.3 嵌套 defaultdict

```python
graph = defaultdict(lambda: defaultdict(int))
graph["A"]["B"] = 1
graph["A"]["C"] = 2
graph["B"]["A"] = 1
print(graph["A"])       # defaultdict(<class 'int'>, {'B': 1, 'C': 2})
print(dict(graph))      # 转成普通 dict 便于打印/序列化
```

### 6.2.4 关键行为：读取也会创建键

```python
dd = defaultdict(list)
print("x" in dd)      # False
_ = dd["x"]           # 访问即创建！
print("x" in dd)      # True
print(dd)             # defaultdict(<class 'list'>, {'x': []})
```

这带来一个常见 bug：

```python
dd = defaultdict(int)
for k in keys:
    if dd[k] > 0:     # ❌ 即使 k 不存在，也会插入 dd[k] = 0
        ...
# 正确：if dd.get(k, 0) > 0 或 if k in dd and dd[k] > 0
```

| 操作 | 是否创建键 |
| :--- | :--- |
| `dd[k]` | ✅ 创建 |
| `dd[k] += 1` | ✅ 创建 |
| `k in dd` | ❌ |
| `dd.get(k)` | ❌ |
| `dd.setdefault(k, v)` | ✅（键缺失时） |

### 6.2.5 其他细节

- `default_factory` 是属性；可以读取、可以改，也可以在创建后设为 `None`（之后行为退化为普通 dict，缺失键抛 `KeyError`）。
- 工厂必须是**无参可调用对象**：`int`、`list`、`set`、`dict`、`lambda: 0` 等；`defaultdict(0)` 会抛 `TypeError`。
- `defaultdict(lambda: ...)` 不能直接 pickle（lambda 不可序列化）；需要序列化时用模块级函数或 `functools.partial`。
- `copy()` 是浅拷贝；`copy.deepcopy` 深拷贝。
- `defaultdict` 是 dict 子类，`isinstance(dd, dict)` 为 True。

### 6.2.6 常见坑

1. **读操作 `dd[k]` 会创建键**，导致"键数量莫名变多"。
2. 用 `for k in dd` 遍历时不要再用 `dd[k]` 检查存在性（虽然遍历已有键不会新建，但逻辑上容易混）。
3. `defaultdict(int)` 的默认值是 0，不是"自动加 1"；`dd[k] += 1` 才是计数。
4. 工厂函数写错（如 `defaultdict(list())`）会直接报错：要传可调用对象，不是实例。
5. 需要 JSON 输出时先 `dict(dd)`，否则 `json.dumps` 虽能处理 dict 子类，但嵌套 defaultdict 需要递归转换。

---

## 6.3 OrderedDict：对顺序有"额外操作"的字典

Python 3.7+ 普通 dict 已经保证插入顺序，但 `OrderedDict` 仍有不可替代的能力：

1. `move_to_end(key, last=True)`：把键移到末尾（或开头）；
2. `popitem(last=True)`：可指定从头部还是尾部弹出；
3. 两个 `OrderedDict` 比较时**顺序敏感**；
4. `reversed(od)` 反向迭代（普通 dict 3.8+ 也支持）；
5. 语义上显式表达"顺序很重要"。

### 6.3.1 基本用法

```python
from collections import OrderedDict

od = OrderedDict([("a", 1), ("b", 2), ("c", 3)])
print(od)                  # OrderedDict({'a': 1, 'b': 2, 'c': 3})

od["d"] = 4                # 新键加到末尾
od["a"] = 10               # 已有键：位置不变
od.move_to_end("a")        # 把 a 移到末尾
print(od)                  # OrderedDict({'b': 2, 'c': 3, 'd': 4, 'a': 10})
od.move_to_end("a", last=False)   # 移到开头
print(od)                  # OrderedDict({'a': 10, 'b': 2, 'c': 3, 'd': 4})

print(od.popitem(last=True))      # ('d', 4) 弹末尾
print(od.popitem(last=False))     # ('a', 10) 弹开头
```

### 6.3.2 相等性：OrderedDict vs dict（高频考点）

```python
od1 = OrderedDict([("a", 1), ("b", 2)])
od2 = OrderedDict([("b", 2), ("a", 1)])
d = {"b": 2, "a": 1}

print(od1 == od2)   # False —— 两个 OrderedDict 比较，顺序敏感
print(od1 == d)     # True  —— OrderedDict 与普通 dict 比较，忽略顺序
```

规则：**当比较双方都是 OrderedDict 时顺序敏感；只要有一方是普通 dict，就退化为普通 dict 的相等性（忽略顺序）**。

### 6.3.3 用 OrderedDict 实现 LRU 缓存

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = OrderedDict()

    def get(self, key):
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)      # 访问后移到最近使用
        return self.cache[key]

    def put(self, key, value):
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)   # 淘汰最久未使用（头部）
```

> 实际项目中也可以用 `functools.lru_cache`，但面试常要求手写，`OrderedDict` 是最直接的实现。

### 6.3.4 常见坑

1. **`od == other_od` 顺序敏感**，不要以为和 dict 一样。
2. `move_to_end` 的键不存在抛 `KeyError`。
3. `popitem()` 默认 `last=True`（弹末尾）；普通 dict 的 `popitem()` 也是 LIFO，但不能指定方向。
4. `OrderedDict` **不是**"自动排序"的字典，它只维护插入顺序；排序仍要 `sorted`。
5. 3.7+ 普通 dict 有序，所以"需要有序"本身不再是使用 OrderedDict 的理由；理由应该是 `move_to_end` / `popitem(last=False)` / 顺序敏感比较。

---

## 6.4 deque：双端队列

`deque`（double-ended queue）是两端都能 O(1) 增删的容器，底层是分块的双向链表。它同时可以当**栈**和**队列**用，还支持 `maxlen` 实现固定长度窗口。

### 6.4.1 创建与基本操作

```python
from collections import deque

d = deque([1, 2, 3])
d.append(4)          # 右端加入 → deque([1, 2, 3, 4])
d.appendleft(0)      # 左端加入 → deque([0, 1, 2, 3, 4])
d.pop()              # 弹右端 → 4
d.popleft()          # 弹左端 → 0
print(d)             # deque([1, 2, 3])

d.extend([4, 5])     # 右端批量加入
d.extendleft([6, 7]) # 左端批量加入 —— 注意顺序！
print(d)             # deque([7, 6, 1, 2, 3, 4, 5])
```

**`extendleft` 的顺序陷阱**：它等价于对迭代对象逐个 `appendleft`，所以**最终顺序与输入相反**：

```python
d = deque([1, 2, 3])
d.extendleft([4, 5])
print(d)             # deque([5, 4, 1, 2, 3])
```

### 6.4.2 `maxlen`：固定长度窗口

```python
d = deque([1, 2, 3], maxlen=3)
d.append(4)          # 满了 → 自动从左边挤出 1
print(d)             # deque([2, 3, 4], maxlen=3)

d.appendleft(0)      # 从右边挤出 4
print(d)             # deque([0, 2, 3], maxlen=3)

d = deque(maxlen=0)  # 永远为空，append 无效果
d.append(1)
print(d)             # deque([], maxlen=0)
```

`maxlen` 是只读属性，创建后不能改。用它实现"最近 N 条记录 / 滑动窗口"非常方便。

### 6.4.3 旋转与反转

```python
d = deque([1, 2, 3, 4])
d.rotate(1)      # 右旋 1 位 → deque([4, 1, 2, 3])
d.rotate(-2)     # 左旋 2 位 → deque([2, 3, 4, 1])
d.reverse()      # 原地反转 → deque([1, 4, 3, 2])
```

`rotate(n)`：正数向右（末尾到开头），负数向左。注意 `rotate` 是原地操作，返回 `None`。

### 6.4.4 索引、插入与删除

```python
d = deque([1, 2, 3, 4, 5])

d[0]        # 1（两端 O(1)）
d[-1]       # 5
d[2]        # 3（中间 O(n)，因为要遍历块）
d[1:3]      # ❌ TypeError: sequence index must be integer, not 'slice'

d.insert(2, 99)    # 中间插入 O(n)；若 deque 有 maxlen 且已满，insert 会抛 IndexError
d.remove(99)       # 按值删除 O(n)，不存在 → ValueError
d.count(1)         # 计数
d.index(2)         # 查找索引
d.clear()          # 清空
d.copy()           # 浅拷贝
```

### 6.4.5 list 还是 deque？

| 操作 | list | deque |
| :--- | :--- | :--- |
| 尾部 append/pop | O(1) | O(1) |
| 头部 insert(0)/pop(0) | **O(n)** | **O(1)** |
| 中间插入/删除 | O(n) | O(n) |
| 按下标访问 | **O(1)** | 两端 O(1)，中间 O(n) |
| 切片 | ✅ | ❌ |
| 排序 | `sort()` | ❌（要 `sorted(d)`） |
| 内存 | 连续数组，较省 | 分块链表，稍多 |
| 线程安全 | 否 | 单次 append/pop 原子（GIL 下） |

**结论**：需要"队列"（先进先出）或"两端频繁操作"用 deque；需要随机访问、切片、排序用 list。

### 6.4.6 常见坑

1. **`extendleft` 会反转输入顺序**。
2. **`deque` 不支持切片**：`d[1:3]` 抛 `TypeError`。
3. **中间索引是 O(n)**，不要用 deque 做"随机访问密集型"任务。
4. **`maxlen` 不能修改**，且满了以后 append 会**静默挤掉**另一端的元素。
5. **`popleft()` / `pop()` 空队列抛 `IndexError`**。
6. **`rotate` 返回 None**，别写成 `d = d.rotate(1)`。
7. **`deque` 不可哈希**，不能作 dict 键。
8. **`deque` 没有 `sort()`**；需要排序用 `sorted(d)`（返回 list）。
9. **线程安全只针对单个操作**：`if d: d.popleft()` 这种"检查再操作"不是原子的。

---

## 6.5 namedtuple：带字段名的元组

`namedtuple` 生成一个 tuple 的子类，既能像元组一样解包/索引，又能用字段名访问，是"轻量记录"的常用方案。

### 6.5.1 定义与使用

```python
from collections import namedtuple

Point = namedtuple("Point", ["x", "y"])      # 字段名用列表
Point = namedtuple("Point", "x y")           # 也可以空格分隔
Point = namedtuple("Point", "x, y")          # 逗号分隔

p = Point(1, 2)
print(p)          # Point(x=1, y=2)
print(p.x, p.y)   # 1 2
print(p[0], p[1]) # 1 2 —— 仍然是元组
x, y = p          # 可解包
print(len(p))     # 2
print(isinstance(p, tuple))   # True
```

### 6.5.2 常用 API

| API | 作用 | 示例 |
| :--- | :--- | :--- |
| `_fields` | 字段名元组 | `Point._fields` → `('x', 'y')` |
| `_asdict()` | 转成 dict（3.8+ 返回普通 dict） | `p._asdict()` → `{'x': 1, 'y': 2}` |
| `_replace(**kwargs)` | 返回替换字段后的新对象 | `p._replace(x=10)` |
| `_make(iterable)` | 从可迭代对象构造 | `Point._make([1, 2])` |
| `_field_defaults` | 默认值字典 | `{'y': 0}` |
| `defaults` | 定义时指定的默认值 | `namedtuple(..., defaults=[0])` |

```python
Point = namedtuple("Point", "x y", defaults=[0])
p = Point(1)
print(p)                      # Point(x=1, y=0)
print(Point._field_defaults)  # {'y': 0}
print(Point._make([3, 4]))    # Point(x=3, y=4)
print(p._replace(x=9))        # Point(x=9, y=0)
print(p._asdict())            # {'x': 1, 'y': 0}
```

默认值从**最右边的字段**开始生效（和函数默认参数规则一致）。

### 6.5.3 不可变性与"改字段"

```python
p = Point(1, 2)
p.x = 10        # ❌ AttributeError: can't set attribute
p[0] = 10       # ❌ TypeError

p2 = p._replace(x=10)   # ✅ 返回新对象
print(p, p2)            # Point(x=1, y=2) Point(x=10, y=2)
```

### 6.5.4 namedtuple vs 普通 tuple vs dataclass

| 方案 | 字段名 | 可变 | 可哈希 | 适用 |
| :--- | :--- | :--- | :--- | :--- |
| `tuple` | ❌ | ❌ | ✅（元素可哈希时） | 临时多返回值 |
| `namedtuple` | ✅ | ❌ | ✅ | 轻量只读记录 |
| `typing.NamedTuple` | ✅ + 类型注解 | ❌ | ✅ | 需要类型提示的只读记录 |
| `@dataclass` | ✅ | ✅（默认） | ❌（默认 `eq=True` 时不可哈希） | 需要可变、字段多、带方法的记录 |
| `@dataclass(frozen=True)` | ✅ | ❌ | ✅ | 不可变记录 + 类型提示 |

```python
from typing import NamedTuple

class Point(NamedTuple):
    x: int
    y: int = 0

p = Point(1)
print(p, p.x)   # Point(x=1, y=0) 1
```

### 6.5.5 常见坑

1. **字段名不能是关键字、不能以下划线开头**（`rename=True` 可自动改名）。
2. **`p.x = ...` 抛 `AttributeError`**；要改字段用 `_replace`。
3. **`p["x"]` 不支持**：它按整数索引，不按字符串索引；要 dict 用 `_asdict()`。
4. **`_asdict()` 是浅拷贝**，嵌套可变对象仍共享。
5. **`_replace` 返回新对象**，原对象不变。
6. **`_make` 参数长度必须匹配字段数**。
7. **`namedtuple` 是 tuple 子类**，`isinstance(p, tuple)` 为 True；与普通 tuple 相等性按元素比较（`Point(1,2) == (1,2)` 为 True）。
8. **默认值从右往左生效**，不能只给左边字段默认值。

---

## 6.6 ChainMap：多层映射叠加

`ChainMap` 把多个 dict 串成一条查找链：先查第一个，找不到再查第二个……写入只作用于第一个。典型用途是"多层配置覆盖"。

### 6.6.1 基本用法

```python
from collections import ChainMap

defaults = {"host": "localhost", "port": 80}
env = {"port": 8080}
cli = {"port": 9000}

config = ChainMap(cli, env, defaults)
print(config["port"])    # 9000（从前往后，第一个命中的赢）
print(config["host"])    # localhost（前面都没有，落到 defaults）
print(config.maps)       # [{'port': 9000}, {'port': 8080}, {'host': 'localhost', 'port': 80}]
```

### 6.6.2 写操作只影响第一个映射

```python
config["debug"] = True
print(config.maps[0])    # {'port': 9000, 'debug': True}
print(config.maps[1])    # {'port': 8080} —— 没变
```

删除同理，只删第一个映射里的键：

```python
cm = ChainMap({"a": 1}, {"b": 2})
print(cm["b"])     # 2（能查到）
del cm["b"]        # ❌ KeyError：删除只作用于第一个映射
```

### 6.6.3 `new_child` 与 `parents`

```python
base = ChainMap({"a": 1}, {"b": 2})

child = base.new_child({"c": 3})
print(child)          # ChainMap({'c': 3}, {'a': 1}, {'b': 2})
print(child.parents)  # ChainMap({'a': 1}, {'b': 2}) —— 去掉第一层

print(len(base))      # 2（去重后的键数）
print("b" in base)    # True
print(dict(base))     # {'a': 1, 'b': 2}
```

- `new_child(m=None)`：在链首插入一个新映射，返回新的 ChainMap；不传参数则插入空 dict。
- `parents`：返回去掉第一层的 ChainMap（原映射对象共享，不是深拷贝）。
- `maps`：底层映射列表，可直接操作（谨慎）。

### 6.6.4 常见坑

1. **写入/删除只作用于第一个映射**，不会"穿透"到后面的层。
2. **`del cm[key]` 对只存在于后面层的键抛 `KeyError`**。
3. `new_child` / `parents` 返回的是**新 ChainMap 对象**，但底层 dict 是共享的，改一个会影响另一个。
4. `len(cm)` 是**去重后的键数**，不是各层长度之和。
5. `ChainMap` 的遍历顺序由底层 `maps` 的扫描顺序决定，不是"自动排序"；需要稳定顺序时用 `sorted(cm)`。
6. `ChainMap` 不是 dict 子类，`isinstance(cm, dict)` 为 False；需要普通 dict 时 `dict(cm)`。

---

## 6.7 UserDict / UserList / UserString：为什么不用直接继承内置类型

### 6.7.1 直接继承 `dict` 的坑

```python
class LoggedDict(dict):
    def __setitem__(self, key, value):
        print(f"set {key}={value}")
        super().__setitem__(key, value)

d = LoggedDict()
d.update({"x": 1})     # ❗ 不会打印任何东西：dict.update 是 C 实现，绕过了 __setitem__
d["y"] = 2             # 会打印
```

内置类型的 C 方法为了性能，通常直接操作内部结构，**不会调用子类重写的方法**。这导致"继承 dict 并重写 `__setitem__`"无法拦截 `update`、`setdefault` 等操作。

### 6.7.2 用 `UserDict` 就安全了

```python
from collections import UserDict

class LoggedDict(UserDict):
    def __setitem__(self, key, value):
        print(f"set {key}={value}")
        super().__setitem__(key, value)

d = LoggedDict()
d.update({"x": 1})   # ✅ 打印 "set x=1"：UserDict.update 内部走 self[k] = v
print(d.data)        # {'x': 1}
```

`UserDict` 内部把数据放在 `self.data`（普通 dict）里，所有方法都用 Python 层实现，因此**子类重写的方法会被正确调用**。

### 6.7.3 三个 User* 容器

| 类 | 内部数据 | 基类 | 用途 |
| :--- | :--- | :--- | :--- |
| `UserDict` | `self.data`（dict） | `MutableMapping` | 定制字典行为、拦截读写 |
| `UserList` | `self.data`（list） | `MutableSequence` | 定制列表行为 |
| `UserString` | `self.data`（str） | `Sequence` | 定制字符串行为 |

```python
from collections import UserList, UserString

class MyList(UserList):
    def append(self, item):
        print("append", item)
        super().append(item)

ml = MyList([1, 2])
ml.append(3)
print(ml, ml.data)    # [1, 2, 3] [1, 2, 3]

s = UserString("hello")
print(s.upper())      # HELLO
print(s.data)         # hello
```

### 6.7.4 常见坑

1. **`UserDict` 不是 `dict` 的子类**：`isinstance(UserDict(), dict)` 为 False；需要原生 dict 用 `dict(ud)` 或 `ud.data`。
2. **`UserList` 不是 `list` 的子类**，但实现了大部分列表协议；`isinstance(ul, list)` 为 False。
3. 直接继承 `dict` / `list` 并重写方法，可能被 C 实现的批量方法绕过；需要拦截时用 User* 容器。
4. `UserString` 的不可变性来自内部 str，但 `.data` 可以直接改（等于破坏封装）。

---

## 6.8 collections 章节的横向速查

| 需求 | 选择 | 一句话示例 |
| :--- | :--- | :--- |
| 统计词频 | `Counter` | `Counter(text.split()).most_common(10)` |
| 缺失键默认 0/[]/set() | `defaultdict` | `defaultdict(list)[k].append(v)` |
| 需要把 key 移到末尾 | `OrderedDict` | `od.move_to_end(k)` |
| 队列 / BFS / 滑动窗口 | `deque` | `q.popleft()` |
| 固定窗口最近 N 个 | `deque(maxlen=N)` | `dq.append(x)` |
| 轻量只读记录 | `namedtuple` | `Point(x, y)` |
| 多层配置覆盖 | `ChainMap` | `ChainMap(cli, env, defaults)` |
| 拦截字典读写 | `UserDict` | `class D(UserDict): ...` |

**最后提醒**：`collections` 的容器都是"针对场景优化"的，不是"更高级所以更好"。默认仍从 list/dict/set 开始，出现明确痛点（计数、缺失键、两端操作、字段可读性）时再换。

---

# 第 7 章 横向对比与 30 组易混淆点

前面按容器逐个讲，这一章把容易混在一起的概念集中对比。**考试丢分大多不是因为不会，而是因为"以为一样"**。

## 7.1 总对比表

| 维度 | `list` | `tuple` | `dict` | `set` | `frozenset` | `deque` |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 可变 | ✅ | ❌ | ✅ | ✅ | ❌ | ✅ |
| 有序 | ✅ 插入序 | ✅ 固定 | ✅ 插入序（3.7+） | ❌ | ❌ | ✅ |
| 可哈希 | ❌ | 元素可哈希时 ✅ | ❌ | ❌ | ✅ | ❌ |
| 允许重复 | ✅ | ✅ | 键唯一 | ❌ | ❌ | ✅ |
| 下标访问 | ✅ O(1) | ✅ O(1) | 按键 O(1) | ❌ | ❌ | 两端 O(1)，中间 O(n) |
| 切片 | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `in` | O(n) | O(n) | O(1) 平均 | O(1) 平均 | O(1) 平均 | O(n) |
| 可作 dict 键 | ❌ | 视元素 | ❌ | ❌ | ✅ | ❌ |
| 内存 | 中 | 较小 | 大 | 大 | 大 | 中 |

| 维度 | `Counter` | `defaultdict` | `OrderedDict` | `namedtuple` | `ChainMap` | `UserDict` |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 基类 | dict | dict | dict | tuple | MutableMapping | MutableMapping |
| 可变 | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ |
| 特殊能力 | 计数 / Top-N | 缺失键默认值 | 顺序操作 | 字段名 | 多层查找 | 安全继承 |
| 典型场景 | 词频 | 分组 | LRU | 记录 | 配置覆盖 | 定制 dict |

## 7.2 30 组易混淆点

### 7.2.1 拷贝与引用（1–6）

**1. `b = a` vs `b = a.copy()`**

```python
a = [1, 2]
b = a
c = a.copy()
a.append(3)
print(b, c)      # [1, 2, 3] [1, 2]
print(a is b, a is c)   # True False
```

赋值只是多一个名字；`copy()`/`[:]`/`list(a)` 才创建新容器。

**2. 浅拷贝 vs 深拷贝**

```python
import copy
a = [[1], [2]]
shallow = copy.copy(a)
deep = copy.deepcopy(a)
a[0].append(9)
print(shallow)   # [[1, 9], [2]] —— 内层共享
print(deep)      # [[1], [2]]    —— 完全独立
```

**3. `a += b` vs `a = a + b`（在函数里尤其危险）**

```python
def f1(lst):
    lst += [99]      # 原地扩展，修改调用者的列表

def f2(lst):
    lst = lst + [99] # 新建列表，只改局部名字

x = [1]; f1(x); print(x)   # [1, 99]
y = [1]; f2(y); print(y)   # [1]
```

**4. `[[0] * 3] * 3` vs `[[0] * 3 for _ in range(3)]`**

```python
bad = [[0] * 3] * 3
bad[0][0] = 1
print(bad)       # [[1, 0, 0], [1, 0, 0], [1, 0, 0]] —— 三行同一个对象

good = [[0] * 3 for _ in range(3)]
good[0][0] = 1
print(good)      # [[1, 0, 0], [0, 0, 0], [0, 0, 0]]
```

**5. 元组"不可变"≠ 内容不可变**

```python
t = ([1], [2])
t[0].append(9)   # ✅ 合法
print(t)         # ([1, 9], [2])
t[0] = [3]       # ❌ TypeError
```

**6. `dict.copy()` 与嵌套字典**

```python
d = {"cfg": {"debug": False}}
d2 = d.copy()
d2["cfg"]["debug"] = True
print(d["cfg"]["debug"])   # True —— 内层共享
```

### 7.2.2 相等、哈希与身份（7–12）

**7. `==` vs `is`**

```python
a = [1, 2]
b = [1, 2]
print(a == b)   # True：值相等
print(a is b)   # False：不是同一个对象
print([] is []) # False
print(None is None)  # True：单例
```

`is` 比较身份（`id`），只应用于 `None`、`True`、`False` 等单例；比较值一律用 `==`。

**8. 小整数缓存与字符串驻留（不要依赖）**

```python
a = int("257"); b = int("257")
print(a is b)     # False：两次调用生成不同对象

a = 257; b = 257
print(a is b)     # True 是常见结果：同一代码块内的常量被复用，但这是实现细节
```

**9. `1`、`1.0`、`True` 是同一个 dict 键 / set 元素**

```python
print({1: "a", True: "b", 1.0: "c"})   # {1: 'c'}
print({1, True, 1.0})                  # {1}
print(1 == 1.0 == True)                # True
```

**10. `hash` 相同不代表相等（哈希冲突）**

```python
print(hash(1) == hash(1.0))   # True：相等对象的哈希必须相同
# 但哈希相同不一定代表相等：dict/set 在哈希命中后还会用 == 复核
```

**11. 可哈希 vs 可变**

- 不可变 ⇒ 通常可哈希，但**不是所有不可变对象都可哈希**（如自定义类可以显式 `__hash__ = None`）。
- 可变 ⇒ 通常不可哈希，但**不是绝对**（例如自定义类可以实现 `__hash__`，只是不该这么做）。
- 准确表述：**可哈希要求哈希值在生命周期内不变，且与 `==` 一致**。

**12. `bool` 是 `int` 的子类**

```python
print(True == 1, True + 1)   # True 2
print(isinstance(True, int)) # True
print([1, 2][True])          # 2（True 当 1 用）
```

### 7.2.3 方法与运算（13–20）

**13. `sort()` vs `sorted()`**

| 写法 | 是否原地 | 返回值 | 适用范围 |
| :--- | :--- | :--- | :--- |
| `a.sort()` | 是 | `None` | 仅 list |
| `sorted(a)` | 否 | 新 list | 任意可迭代对象 |

```python
a = [3, 1, 2]
b = sorted(a)
c = a.sort()
print(a, b, c)   # [1, 2, 3] [1, 2, 3] None
```

**14. `remove` vs `pop` vs `del`**

```python
a = ["x", "y", "z", "y"]
a.remove("y")    # 按值删第一个 → ['x', 'z', 'y']
a.pop()          # 按位置删末尾并返回 → 'y'
del a[0]         # 按位置删，不返回 → ['z']
```

**15. `append` vs `extend` vs `+`**

```python
a = [1]
a.append([2, 3])   # [1, [2, 3]]
a = [1]
a.extend([2, 3])   # [1, 2, 3]
a = [1]
b = a + [2, 3]     # a 不变，b = [1, 2, 3]
```

**16. `d.get(k)` vs `d.setdefault(k, v)` vs `defaultdict[k]`**

| 写法 | 缺失时返回值 | 是否插入 |
| :--- | :--- | :--- |
| `d.get(k)` | `None` | 否 |
| `d.setdefault(k, v)` | `v` | 是 |
| `defaultdict(f)[k]` | `f()` | 是 |

**17. `sorted(d)` vs `sorted(d.items())` vs `sorted(d.values())`**

```python
d = {"b": 1, "a": 2}
print(sorted(d))           # ['a', 'b'] —— 键排序
print(sorted(d.items()))   # [('a', 2), ('b', 1)] —— 按键排序
print(sorted(d.values()))  # [1, 2] —— 值排序
```

**18. `d.keys()` 视图 vs `list(d.keys())`**

```python
d = {"a": 1}
ks = d.keys()
d["b"] = 2
print(list(ks))     # ['a', 'b'] —— 视图动态
# print(ks[0])      # ❌ TypeError：视图不可下标
```

**19. `for k in d` vs `for k, v in d.items()`**

```python
d = {"a": 1}
for k in d:
    print(k)          # 键

for k, v in d.items():
    print(k, v)       # 键值对
```

**20. `+` vs `|`（dict 合并，3.9+）**

```python
a = {"x": 1}; b = {"x": 2, "y": 3}
print(a | b)        # {'x': 2, 'y': 3} —— 右覆盖左
print({**a, **b})   # 同上
# a + b             # ❌ TypeError：dict 不支持 +
```

### 7.2.4 容器选择（21–25）

**21. `list` 当队列 vs `deque` 当队列**

```python
# ❌ list：pop(0) 是 O(n)，循环 n 次 → O(n²)
while lst:
    lst.pop(0)

# ✅ deque：popleft() 是 O(1)
from collections import deque
q = deque(lst)
while q:
    q.popleft()
```

**22. list 的 `in` vs set 的 `in`**

```python
big = list(range(100000))
# if x in big: ...      # O(n)
big_set = set(big)
# if x in big_set: ...  # O(1) 平均
```

**23. 去重：`set` vs `dict.fromkeys`**

```python
nums = [3, 1, 3, 2]
print(list(set(nums)))              # 顺序不保证，例如 [1, 2, 3]
print(list(dict.fromkeys(nums)))    # [3, 1, 2] —— 保序
```

**24. `tuple` vs `list` 的"记录/集合"语义**

```python
record = ("Tom", 20, "CS")   # 字段固定、异质 → tuple
scores = [90, 85, 77]        # 同质、会增删排序 → list
```

**25. `str` 的 `+` 拼接 vs `join`**

```python
# ❌ 循环里 += 字符串：每次新建，可能 O(n²)
s = ""
for w in words:
    s += w

# ✅ 一次性 join：O(n)
s = "".join(words)
```

### 7.2.5 collections 相关（26–30）

**26. `Counter.update` vs `Counter +`**

```python
from collections import Counter
c = Counter(a=2)
c.update({"a": 1})       # 原地：a=3
d = c + Counter(a=5)     # 新对象：a=8
```

**27. `defaultdict[k]` vs `defaultdict.get(k)`**

```python
from collections import defaultdict
dd = defaultdict(list)
_ = dd["x"]          # 创建 x: []
print("x" in dd)     # True
dd.get("y")          # 不创建
print("y" in dd)     # False
```

**28. `deque.extendleft` 的顺序**

```python
from collections import deque
d = deque([1, 2, 3])
d.extendleft([4, 5])
print(d)             # deque([5, 4, 1, 2, 3]) —— 输入被反转
```

**29. `OrderedDict` 的相等性**

```python
from collections import OrderedDict
od1 = OrderedDict([("a", 1), ("b", 2)])
od2 = OrderedDict([("b", 2), ("a", 1)])
print(od1 == od2)          # False（两个 OrderedDict，顺序敏感）
print(od1 == {"b": 2, "a": 1})   # True（与普通 dict 比较，忽略顺序）
```

**30. 继承 `dict` vs 继承 `UserDict`**

```python
class D1(dict):
    def __setitem__(self, k, v):
        print("hook")
        super().__setitem__(k, v)

D1().update({"a": 1})    # 不打印：update 是 C 实现，绕过重写

from collections import UserDict
class D2(UserDict):
    def __setitem__(self, k, v):
        print("hook")
        super().__setitem__(k, v)

D2().update({"a": 1})    # 打印 hook：UserDict 走 Python 层方法
```

## 7.3 一句话速记

- **要改 → list / dict / set / deque；要当键 → tuple / frozenset / str。**
- **要顺序 → list / dict；要排序 → `sorted`；要快速查 → dict / set。**
- **拷贝分三层：引用、浅拷贝、深拷贝；集合类型默认只做浅拷贝。**
- **`sort` / `update` / `append` / `add` 这些"动作"通常返回 `None`。**
- **dict 的键和 set 的元素是同一套哈希规则：可哈希、唯一、1 == 1.0 == True。**
- **collection 是"专用工具"：Counter 计数、defaultdict 补默认、deque 两端、namedtuple 命名、ChainMap 分层。**

---

# 第 8 章 易错输出题自测（答案折叠）

> 先自己在纸上写出输出，再点开折叠核对。这些题覆盖了前 7 章最容易错的知识点，和配套题库的"综合部分"互相补充。

### 题目 1

```python
a = [1, 2, 3]
b = a
b += [4]
print(a, a is b)
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
[1, 2, 3, 4] True
```

**解析：** `b = a` 只复制引用；`b += [4]` 对 list 调用 `__iadd__`，等价于 `extend`，**原地修改**，所以 `a` 和 `b` 是同一个对象，都变成 `[1, 2, 3, 4]`。
</details>

### 题目 2

```python
t = ([1], 2)
try:
    t[0] += [3]
except TypeError:
    print("TypeError")
print(t)
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
TypeError
([1, 3], 2)
```

**解析：** `t[0] += [3]` 先原地修改列表（成功），再试图把结果写回 `t[0]`，元组不允许赋值，于是抛 `TypeError`——但副作用已经发生。
</details>

### 题目 3

```python
d = {1: "a"}
d[True] = "b"
d[1.0] = "c"
print(d)
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
{1: 'c'}
```

**解析：** `1 == True == 1.0` 且三者哈希相同，dict 认为它们是同一个键；键保留第一次插入的 `1`，值被最后一次赋值覆盖为 `'c'`。
</details>

### 题目 4

```python
d = dict.fromkeys("ab", [])
d["a"].append(1)
print(d)
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
{'a': [1], 'b': [1]}
```

**解析：** `fromkeys` 把**同一个**列表对象赋给所有键，两个键共享该列表。要独立列表用 `{k: [] for k in "ab"}`。
</details>

### 题目 5

```python
from collections import deque

d = deque([1, 2, 3], maxlen=2)
d.append(4)
d.appendleft(0)
print(d)
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
deque([0, 3], maxlen=2)
```

**解析：** `deque([1,2,3], maxlen=2)` 只保留最后两个 `[2, 3]`；`append(4)` 从左边挤出 2 → `[3, 4]`；`appendleft(0)` 从右边挤出 4 → `[0, 3]`。
</details>

### 题目 6

```python
from collections import Counter

c = Counter("aab")
print(c["z"], "z" in c, c.most_common(1))
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
0 False [('a', 2)]
```

**解析：** `Counter` 访问缺失键返回 `0`，但**不插入**，所以 `"z" in c` 为 False；`most_common(1)` 返回计数最高的一个。
</details>

### 题目 7

```python
from collections import OrderedDict

a = OrderedDict([("a", 1), ("b", 2)])
b = OrderedDict([("b", 2), ("a", 1)])
print(a == b, a == dict(b))
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
False True
```

**解析：** 两个 `OrderedDict` 比较时**顺序敏感**，`a` 与 `b` 顺序不同 → False；与普通 dict 比较时退化为忽略顺序 → True。
</details>

### 题目 8

```python
a = [1, 2, 3, 4, 5]
a[1:3] = [9]
print(a)
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
[1, 9, 4, 5]
```

**解析：** 普通切片赋值长度自由：用 1 个元素替换 2 个元素，后面的元素整体前移。若是 `a[::2] = [9]`（扩展切片）则必须等长，会抛 `ValueError`。
</details>

### 题目 9

```python
x = [[0] * 2] * 2
x[0][0] = 1
print(x)
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
[[1, 0], [1, 0]]
```

**解析：** `[[0] * 2] * 2` 复制的都是**同一个内层列表的引用**，改一行等于改两行。正确写法是 `[[0] * 2 for _ in range(2)]`。
</details>

### 题目 10

```python
import copy

a = [[1], [2]]
b = copy.copy(a)
c = copy.deepcopy(a)
a[0].append(9)
print(b, c)
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
[[1, 9], [2]] [[1], [2]]
```

**解析：** 浅拷贝只复制外层，内层 list 共享，所以 `b` 跟着变；深拷贝递归复制，`c` 独立。
</details>

### 题目 11

```python
t = (1, 2, 3)
print(t[:] is t, tuple(t) is t)
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
True True
```

**解析：** 元组不可变，切片和 `tuple(t)` 都可以安全地返回原对象，不需要复制。对 list 则不同：`a[:]` 是新列表。
</details>

### 题目 12

```python
print(type({1} | frozenset({2})), type(frozenset({1}) | {2}))
print({1, 2} == frozenset({2, 1}))
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
<class 'set'> <class 'frozenset'>
True
```

**解析：** 集合运算结果类型**跟随左操作数**；set 与 frozenset 按元素比较，顺序无关，所以相等。
</details>

### 题目 13

```python
from collections import defaultdict

dd = defaultdict(int)
print(dd["x"], dd, "x" in dd)
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
0 defaultdict(<class 'int'>, {'x': 0}) True
```

**解析：** `defaultdict` 的 `dd[k]` 在键缺失时会调用工厂（`int()` → 0）并**插入**，所以访问后 `"x" in dd` 为 True。
</details>

### 题目 14

```python
from collections import namedtuple

P = namedtuple("P", "x y", defaults=[0])
p = P(1)
print(p, p._asdict(), p._replace(y=2))
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
P(x=1, y=0) {'x': 1, 'y': 0} P(x=1, y=2)
```

**解析：** `defaults=[0]` 从最右边字段开始生效，所以 `y` 默认为 0；`_asdict()` 转 dict；`_replace()` 返回新对象，原对象不变。
</details>

### 题目 15

```python
d = {"a": 1, "b": 2}
for k in d:
    if k == "a":
        del d[k]
print(d)
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
RuntimeError: dictionary changed size during iteration
```

**解析：** 遍历 dict 时增删键会改变大小，CPython 检测到后抛 `RuntimeError`。应先 `for k in list(d)` 或先收集要删的键。
</details>

### 题目 16

```python
items = [("b", 2), ("a", 2), ("c", 1)]
print(sorted(items, key=lambda t: t[1]))
print(sorted(items, key=lambda t: t[1], reverse=True))
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
[('c', 1), ('b', 2), ('a', 2)]
[('b', 2), ('a', 2), ('c', 1)]
```

**解析：** `key` 只取第二个元素比较；排序是**稳定**的，`2` 分组保持原来的 `b` 在 `a` 前。`reverse=True` 也是稳定反转分组，不是简单倒序。
</details>

### 题目 17

```python
a = [1, 2]
a = a.append(3)
print(a)
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
None
```

**解析：** `append` 原地修改并返回 `None`，把返回值赋给 `a` 后 `a` 变成 None。同理 `a = a.sort()`、`a = a.extend(...)` 都是经典错误。
</details>

### 题目 18

```python
print(len({1, True, 1.0, 2}), {1, True, 1.0, 2})
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
2 {1, 2}
```

**解析：** `1`、`True`、`1.0` 三者相等且哈希相同，set 只保留第一个。
</details>

### 题目 19

```python
a = [3, 1, 2]
b = sorted(a)
c = a.sort()
print(a, b, c)
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
[1, 2, 3] [1, 2, 3] None
```

**解析：** `sorted(a)` 返回新列表赋给 `b`；`a.sort()` 原地排序并把 `a` 变成 `[1,2,3]`，返回 `None` 赋给 `c`。注意 `a.sort()` 在 `b = sorted(a)` 之后执行，但两者结果都是升序。
</details>

### 题目 20

```python
d = {"b": 1, "a": 2}
print(list(d), sorted(d), sorted(d.items()))
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
['b', 'a'] ['a', 'b'] [('a', 2), ('b', 1)]
```

**解析：** `list(d)` 按键的**插入顺序**；`sorted(d)` 对键排序；`sorted(d.items())` 按键值对的第一个元素（键）排序。
</details>

### 题目 21

```python
s = "ab"
t = s
s += "c"
print(s, t, s is t)
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
abc ab False
```

**解析：** str 不可变，`s += "c"` 创建新字符串并重新绑定 `s`，`t` 仍指向原来的 `"ab"`。
</details>

### 题目 22

```python
from collections import Counter

c = Counter(a=1)
c.update({"a": 1})
print(c, c + Counter(a=3))
```

<details>
<summary>查看答案与输出</summary>

**输出：**

```text
Counter({'a': 2}) Counter({'a': 5})
```

**解析：** `update` 原地累加计数 → `a=2`；`c + Counter(a=3)` 返回新 Counter，计数相加 → `a=5`，`c` 本身不变。
</details>

---

# 第 9 章 学习路径与考前清单

## 9.1 三阶段学习路径

### 阶段一：建立正确的心智模型（1–2 天）

1. 记住第 1 章的四个维度：**可变性 / 有序性 / 可哈希性 / 唯一性**。
2. 能手写出 4 个容器的 10 种创建方式，并解释 `{}`、`set()`、`(1,)` 的区别。
3. 能背出复杂度表，尤其是：list 的 `in` 是 O(n)，dict/set 是 O(1)；list 头部插删是 O(n)。
4. 能解释"为什么 list 不能当 dict 键，而 tuple 可以（有条件）"。

### 阶段二：把接口用熟（2–3 天）

1. 每个容器的方法至少手敲一遍，重点记住**返回值**（哪些返回 `None`）。
2. 把第 2–6 章的"常见坑"逐条运行验证，观察报错信息。
3. 完成配套题库的 list / tuple / dict / set 四个专项部分，错题回到对应章节。
4. 练习：用 dict 统计、用 set 去重、用 deque 写 BFS、用 Counter 取 Top-N、用 namedtuple 表示记录。

### 阶段三：形成选型直觉（1 天）

1. 看到需求先问"映射、集合还是序列"，再问"是否可变、是否需要顺序、是否要当键"。
2. 把第 7 章的 30 组易混淆点过一遍，能说出每组"错在哪、对在哪"。
3. 限时做综合部分和输出题，训练"看代码出结果"的推演能力。
4. 用 `dis` / `sys.getsizeof` / `timeit` 观察一次底层行为，理解复杂度不是纸上谈兵。

## 9.2 考前 30 分钟速查卡

**创建**

- `[]`、`list()`；`()`、`tuple()`、`(1,)`；`{}`、`dict()`、`dict.fromkeys()`；`set()`（不是 `{}`）；`frozenset()`。
- `{}` 永远是 dict；空 set 只能 `set()`。

**高频方法**

| 容器 | 增 | 删 | 查 | 其他 |
| :--- | :--- | :--- | :--- | :--- |
| list | `append` / `extend` / `insert` | `remove` / `pop` / `del` / `clear` | `index` / `count` / `in` | `sort` / `reverse` / `copy` |
| dict | `d[k]=v` / `update` / `setdefault` | `pop` / `del` / `popitem` / `clear` | `get` / `in` / `keys` / `values` / `items` | `copy` / `\|` / `fromkeys` |
| set | `add` / `update` | `remove` / `discard` / `pop` / `clear` | `in` | `union` / `intersection` / `difference` / `symmetric_difference` |
| deque | `append` / `appendleft` / `extend` / `extendleft` | `pop` / `popleft` / `remove` / `clear` | `index` / `count` / `in` | `rotate` / `reverse` / `maxlen` |

**必背结论**

1. `sort`、`sorted` 之外，几乎所有"原地修改"方法返回 `None`。
2. `a += b` 对 list 是原地扩展；`a = a + b` 新建列表。
3. `t[0] += [3]` 会先改列表再抛 `TypeError`。
4. `1 == 1.0 == True`，哈希相同，在 dict/set 中是同一个键。
5. dict 3.7+ 插入有序，但"有序 ≠ 排序"。
6. 遍历 dict/set 时增删元素 → `RuntimeError`。
7. `fromkeys` 的默认值会被所有键共享（可变对象时是坑）。
8. 浅拷贝只复制一层；深拷贝用 `copy.deepcopy`。
9. `deque.extendleft` 会反转输入顺序。
10. `Counter` 缺失键返回 0；`+`/`-` 丢弃非正计数，`update`/`subtract` 保留。
11. `defaultdict[k]` 会创建键，`get(k)` 不会。
12. 两个 `OrderedDict` 比较顺序敏感；与普通 dict 比较则忽略顺序。
13. `namedtuple` 不可变，改字段用 `_replace`。
14. `set` 无序、不可下标；`sorted(s)` 返回 list。
15. `tuple` 切片、`tuple(t)` 可能返回原对象（不可变，无需复制）。

## 9.3 错题复盘模板

每道错题按下面 4 行记录，比抄答案有效得多：

```text
题目考点：        （例如：dict 的 fromkeys 共享默认值）
我的答案 / 错因：  （例如：以为每个键会新建列表）
正确结论：        （fromkeys 只赋同一个对象；用推导式代替）
反例代码：        （3 行能复现错误的最小代码）
```

## 9.4 配套题库使用建议

《Python数据结构选择题220题_含答案解析.md》按 6 个部分组织：

| 部分 | 主题 | 题量 | 建议用时 |
| :--- | :--- | :---: | :--- |
| 第一部分 | list 专项 | 40 | 30 分钟 |
| 第二部分 | tuple 与序列通用 | 35 | 25 分钟 |
| 第三部分 | dict 专项 | 40 | 30 分钟 |
| 第四部分 | set / frozenset 专项 | 35 | 25 分钟 |
| 第五部分 | collections 专项 | 40 | 35 分钟 |
| 第六部分 | 综合易混淆 | 30 | 30 分钟 |
| **合计** | | **220** | **约 3 小时** |

**刷题方法**：

1. **第一轮**：按部分顺序做，做完一部分立即对答案，错题回到本文对应章节。
2. **第二轮**：只做错题 + 标记为"挑战"的题，重点看"逐项解析"里错误选项为什么错。
3. **第三轮**：随机抽 30 题限时 25 分钟，模拟考试节奏。
4. 看到"以下说法正确的是"时，先排除绝对化表述（"一定""必须""任何情况下"），再验证边界条件。

---

> **最后一句**：数据结构题的本质不是背接口，而是**推演对象的状态**。做题时在草稿上画出"谁和谁是同一个对象、哪一步创建了新对象、哪一步原地修改"，绝大多数易错题都会变得清晰。
