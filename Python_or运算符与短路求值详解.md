# 从 `x = 100 or 200` 说起：Python 的 `or`、真值与短路求值

> **直接答案：`x` 的值是 `100`，类型是 `int`（不是 `True`）。**
>
> 原因：`or` 先检查左操作数 `100` 的真值，发现为真，就**把这个操作数本身原样返回**；右边的 `200` 根本没有机会被选中。
>
> 更关键的一点：`or` 不是"逻辑门"，不负责把结果转成 `True/False`；它是一个"**返回操作数**"的运算符。`x = 100 or 200` 里，`x` 拿到的是整数 `100`。

**实测环境**：Python 3.13.5（Windows，CPython）。下文带 `# ->` 的关键输出均在该解释器上实际运行验证（对象地址、长 repr 之类为简写）。

---

## 0. 先建立知识地图

一行 `x = 100 or 200`，可以牵出 Python 的十个知识面：

```text
x = 100 or 200
├── 语法层：or 是关键字 / 布尔运算符，不是函数，不能被重载
├── 语义层：a or b 返回操作数本身，而不是 bool
├── 真值层：什么算"真"？bool() 的规则、__bool__ / __len__
├── 短路层：b 只在 a 为假时才求值（副作用、异常、性能）
├── 编译层：AST（BoolOp）→ 常量折叠 → 字节码跳转
├── 类型层：结果类型是 int；bool 是 int 的子类
├── 优先级层：or < and < not < 比较 < 位运算 < 算术
├── 惯用法层：默认值、回退链、守卫、缓存、可选回调
├── 陷阱层：0/""/[] 等合法假值被"兜底"掉、对象别名、缓存失效、NumPy
└── 跨语言层：C / Java / JavaScript / SQL 的 || 语义对比
```

---

## 1. 答案与逐项验证

```python
x = 100 or 200

print(x)            # -> 100
print(type(x))      # -> <class 'int'>
print(bool(x))      # -> True
print(x + 1)        # -> 101
print(x == 200)     # -> False
```

| 问题 | 答案 | 原因 |
| --- | --- | --- |
| `x` 的值 | `100` | `100` 为真，`or` 直接返回左操作数 |
| `x` 的类型 | `int` | `or` 返回操作数本身，不做 `bool()` 转换 |
| `bool(x)` | `True` | 非零整数为真 |
| `x + 1` | `101` | 拿到的是整数，可直接参与算术 |
| `200` 去哪了 | 未被选中 | 短路求值；常量折叠后甚至不会出现在字节码里 |
| 如果问 `x == 200` | `False` | 值是 100 |
| 如果问 `x is 100` | 本机恰好 `True`，但**不要这样写** | 依赖小整数缓存与常量复用，是实现细节 |

> **关于 `x is 100`**：`is` 比较的是对象身份（内存地址），不是值。CPython 会缓存 `-5 ~ 256` 的小整数，且常量折叠会让 `x` 与字面量 `100` 指向同一个常量对象，所以这里"碰巧"是 `True`。换一种 Python 实现（如 PyPy），或换一个代码上下文，结果就可能不同；而且 Python 3.8+ 对"用 `is` 比较字面量"会给出 `SyntaxWarning`。判断值请一律用 `==`。

---

## 2. 语法层：`or` 是关键字，不是函数

`or` 属于 Python 的**布尔运算符**（boolean operators），只有三个：`and`、`or`、`not`。

```python
import keyword
print("or" in keyword.kwlist)   # -> True
```

由此带来几个直接结论：

1. 不能写成函数调用：

   ```python
   or(100, 200)
   # SyntaxError: invalid syntax
   ```

2. 不能作为变量名赋值：

   ```python
   or = 1
   # SyntaxError: invalid syntax
   ```

3. 不能被重载。给类定义 `__or__` 只会影响按位或 `|`，不会影响 `or`：

   ```python
   class HasOr:
       def __or__(self, other):
           return "overloaded"

   o = HasOr()
   print(o | 1)            # -> overloaded（走 __or__）
   print(o or "fallback")  # -> <HasOr object>（or 不看 __or__，只看真值）
   ```

   也就是说：`or` / `and` 的行为只由操作数的**真值**决定，没有 `__or__`、`__and__` 这样的重载钩子。想影响 `or` 的结果，只能控制对象的"真值"（见第 5 节）。

4. 没有 `or=` / `and=` 这种增强赋值。`|=` 存在，但它是**按位或赋值**，不是逻辑或：

   ```python
   x = 0
   x or= 1     # SyntaxError: invalid syntax
   x |= 1      # 按位或赋值，x 变成 1
   s = {1}
   s |= {2}    # 集合的 |= 是原地并集，s 变成 {1, 2}
   ```

---

## 3. 语义层：`a or b` 的精确规则

### 3.1 一句话定义

```text
求值 a：
    若 bool(a) 为真 → 返回 a 本身
    否则           → 求值 b，返回 b 本身
```

用伪代码写就是：

```python
v = 求值(a)          # 只求值一次
if bool(v):
    result = v
else:
    result = 求值(b)
```

这条规则包含两个要点：

- **返回值**：结果一定是 `a` 或 `b` **本身**（同一个对象），而不是 `bool(a or b)`；
- **短路**：只有 `a` 为假时，才会去求值 `b`；`a` 为真时 `b` 完全不被触碰。

> 不要把上面的伪代码机械地改写成 `a if bool(a) else b`：那样 `a` 会在源码里出现两次，若 `a` 是函数调用或带副作用的属性，求值次数就变了（详见 10.3 节）。

### 3.2 真值表

| `a` 的真值 | `a or b` 的结果 | 说明 |
| --- | --- | --- |
| 真 | `a` 本身 | 不转换、不拷贝，原样返回 |
| 假 | `b` 本身 | 即使 `b` 也是假值，也返回 `b` |

```python
print(100 or 200)          # -> 100    （a 真，返回 a）
print(0 or 200)            # -> 200    （a 假，返回 b）
print(0 or 0)              # -> 0      （两个都假，返回 b）
print("" or "default")     # -> default
print([] or [1, 2])        # -> [1, 2]
print(None or False)       # -> False  （b 是假值也照返）
```

### 3.3 多个操作数：`a or b or c`

`or` 是**左结合**的，但因为短路，实际行为可以概括为：

> **从左到右逐个求真值，返回第一个真值；如果全是假值，返回最后一个操作数。**

```python
print(None or 0 or "" or [])     # -> []      （全是假值，返回最后一个）
print(0 or "" or "last")         # -> last
print(0 or 1 or 2)               # -> 1       （第一个真值）
```

### 3.4 返回的是"同一个对象"

```python
a = [1]
b = a or [2]
print(b is a)       # -> True，b 和 a 是同一个列表，不是拷贝

a = []
b = a or [1, 2]
c = a or [1, 2]
print(a is b)       # -> False
print(b is c)       # -> False   （每次求值都会新建一个列表字面量）
```

这一点在可变对象上会变成真实 bug，见第 11 节。

---

## 4. 类型层：结果是 `int`，`bool` 只是 `int` 的子类

```python
x = 100 or 200
print(type(x))                  # -> <class 'int'>
print(isinstance(x, int))       # -> True
print(type(True or False))      # -> <class 'bool'>    （两个 bool 操作数，返回 bool）
print(type(0 or 3.5))           # -> <class 'float'>   （返回 b 的类型）
print(type(3 or 4.0))           # -> <class 'int'>     （返回 a 的类型，不会"升级"成 float）
```

几个容易混淆的事实：

```python
print(isinstance(True, int))    # -> True   bool 是 int 的子类
print(True + True)              # -> 2
print(True == 1)                # -> True
print(True is 1)                # -> False  身份不同
```

因此：

- `x = 100 or 200` 得到的是整数 `100`，可以直接 `x + 1`、`x >> 1`；
- 如果业务上需要布尔值，必须显式写 `bool(100 or 200)` 或 `x = True`；
- 类型注解里，`100 or 200` 的类型就是 `int`，不是 `bool`。

---

## 5. 真值层：到底什么算"真"？

`or` 判断左操作数时，调用的是它的**真值**（truth value），等价于 `bool(a)`。

### 5.1 `bool()` 的判定顺序

对任意对象 `obj`，`bool(obj)` 依次尝试：

1. 定义了 `__bool__`：调用它，**必须返回 bool**，否则 `TypeError`；
2. 没有 `__bool__` 但定义了 `__len__`：调用它，**返回 0 为假、非 0 为真**，返回值必须是 `>= 0` 的整数；
3. 两个都没定义：默认为**真**。

### 5.2 假值清单（falsy values）

| 表达式 | `bool()` | 类别 |
| --- | --- | --- |
| `False` | `False` | 布尔 |
| `None` | `False` | 空值 |
| `0` | `False` | 整数零 |
| `0.0` / `-0.0` | `False` | 浮点零（注意负零也是假） |
| `0j` | `False` | 复数零 |
| `Decimal(0)` | `False` | 十进制零 |
| `Fraction(0, 1)` | `False` | 分数零 |
| `""` | `False` | 空字符串 |
| `b""` | `False` | 空字节串 |
| `[]` | `False` | 空列表 |
| `()` | `False` | 空元组 |
| `{}` | `False` | 空字典 |
| `set()` / `frozenset()` | `False` | 空集合 |
| `range(0)` | `False` | 空 range |

**口诀：假值 = 零、空、`None`、`False`。**

### 5.3 反直觉的真值

| 表达式 | `bool()` | 说明 |
| --- | --- | --- |
| `-1` | `True` | 负数为真 |
| `0.1` | `True` | 非零浮点为真 |
| `"0"` / `"False"` / `" "` | `True` | 非空字符串一律为真，"假"字面量也是真 |
| `[0]` / `(0,)` / `{0}` | `True` | 容器非空即为真，哪怕里面装的是 0 |
| `float("nan")` | `True` | NaN 不是 0，为真 |
| `float("inf")` | `True` | 无穷大为真 |
| 函数 / 类 / 模块 / 生成器 | `True` | 没有定义真值方法的对象默认为真 |
| 自定义对象（无 `__bool__`、无 `__len__`） | `True` | 默认为真 |

```python
for v in [False, None, 0, 0.0, 0j, "", b"", [], (), {}, set(), range(0)]:
    assert not v, repr(v)

for v in [-1, 0.1, "0", "False", [0], (0,), {0}, float("nan")]:
    assert bool(v), repr(v)
```

> **易错点**：`if x:` 为假，不代表 `x` 一定等于 `None`，也不代表 `x` 一定等于 `0`。它可能是 `""`、`[]`、`{}`、`False`、`0.0`、`-0.0`、`Decimal(0)`……这就是"假值兜底"会踩坑的根源。

---

## 6. 编译层：AST、常量折叠与字节码

### 6.1 AST：`or` 是一个 `BoolOp`

```python
import ast

tree = ast.parse("100 or 200", mode="eval")
print(ast.dump(tree, indent=2))
```

输出（节选）：

```text
Expression(
  body=BoolOp(
    op=Or(),
    values=[
      Constant(value=100),
      Constant(value=200)]))
```

这说明 `or` 在语法树上不是函数调用，而是 `BoolOp`（布尔运算）节点。

### 6.2 常量折叠：`100 or 200` 在编译期就变成 `100`

```python
import dis

dis.dis(compile("x = 100 or 200", "<s>", "exec"))
```

Python 3.13.5 实际输出：

```text
  1           LOAD_CONST               0 (100)
              STORE_NAME               0 (x)
              RETURN_CONST             1 (None)
```

可以看到：字节码里只剩 `LOAD_CONST 100`，`200` 已经消失。原因是两个操作数都是常量，CPython 编译器在 AST 优化阶段直接算出了结果（常量折叠）。

```python
dis.dis(compile("x = 0 or 200", "<s>", "exec"))
# -> LOAD_CONST 200
```

> 注意：这是**实现细节**，不是语言规范。即使编译器不折叠，运行结果也一样。它带来的一个"考试意义"是：问"`200` 有没有被求值"时，标准回答是"**短路语义决定它不会被选中**"；而在 CPython 里更极端——它压根没进字节码。

### 6.3 运行期字节码：跳转结构

当操作数不是常量（例如函数参数）时，才能看到真正的短路跳转：

```python
def pick(a, b):
    return a or b

import dis
dis.dis(pick)
```

Python 3.13.5 实际输出（省略首行 `RESUME`；首列行号随代码所在位置变化）：

```text
  4           LOAD_FAST                0 (a)
              COPY                     1
              TO_BOOL
              POP_JUMP_IF_TRUE         2 (to L1)
              POP_TOP
              LOAD_FAST                1 (b)
      L1:     RETURN_VALUE
```

逐条解释：

| 指令 | 作用 |
| --- | --- |
| `LOAD_FAST a` | 把 `a` 压栈 |
| `COPY 1` | 复制栈顶，留一份用于"真时直接返回" |
| `TO_BOOL` | 计算 `a` 的真值（CPython 3.13 中有这条指令；更早版本通常在跳转指令内部判真值） |
| `POP_JUMP_IF_TRUE` | 为真则跳到 `L1`，栈顶的 `a` 留着作为返回值 |
| `POP_TOP` | 为假则丢掉 `a` |
| `LOAD_FAST b` | 加载 `b` |
| `L1: RETURN_VALUE` | 返回栈顶（真时是 `a`，假时是 `b`） |

结构可以概括为：

```text
求值 a → 真？→ 是：返回 a（b 不求值）
              → 否：求值 b → 返回 b
```

> 不同 Python 版本的指令名会变化（例如 3.12 及更早的指令序列与 3.13 不完全相同，不一定出现 `TO_BOOL`），但"复制 / 判真 / 跳转 / 否则加载 b"的结构一致。**语义稳定，字节码易变**，学习时抓结构即可。

---

## 7. 短路层：副作用、异常与性能

### 7.1 右边可能根本不被执行

```python
def f():
    print("f 被调用")
    return 0

def g():
    print("g 被调用")
    return "g"

print(f() or g())
# 输出：
# f 被调用
# g 被调用
# g
```

```python
def h():
    print("h 被调用")
    return 100

def k():
    print("k 被调用")
    return 200

print(h() or k())
# 输出：
# h 被调用
# 100        ← k() 没有被调用
```

### 7.2 短路可以"挡住异常"

```python
print(1 or 1 / 0)     # -> 1       右边不执行，不会除零
print(0 and 1 / 0)    # -> 0       and 在左边为假时也短路
print(0 or 1 / 0)     # -> ZeroDivisionError: division by zero
```

这是"用 `or` / `and` 做保护"的原理：

```python
if denominator and total / denominator > 0.5:
    ...

if user and user.profile and user.profile.email:
    ...
```

### 7.3 左操作数只求值一次

`a or b` 中 `a` 只求值一次（字节码先 `LOAD` 再 `COPY`）。所以即使 `a` 是带副作用的属性或函数调用，也不会重复执行：

```python
calls = 0

def get():
    global calls
    calls += 1
    return 0

print(get() or "fallback")   # -> fallback
print(calls)                 # -> 1，只调用了一次
```

### 7.4 性能

`or` 的短路本身就是一种优化手段：能提前确定结果时，右边的昂贵计算（数据库查询、网络请求、大循环）不会发生。但不要为了"微性能"牺牲可读性；`or` 首先是语义工具。

---

## 8. 优先级与结合性

### 8.1 相关运算符的优先级（从低到高）

| 优先级 | 运算符 | 示例 |
| --- | --- | --- |
| 最低 | `lambda` | `lambda: 100 or 200` 是函数，调用后得 `100` |
| ↓ | 条件表达式 `if-else` | `0 or 1 if 0 else 2` 解析为 `(0 or 1) if 0 else 2` |
| ↓ | `or` | `1 or 0 and 0` 解析为 `1 or (0 and 0)` |
| ↓ | `and` | `1 and 0 or 2` 解析为 `(1 and 0) or 2` |
| ↓ | `not` | `not 0 or 1` 解析为 `(not 0) or 1` |
| ↓ | 比较 `in / is / < / ==` 等 | `1 < 2 or 3 < 2` 解析为 `(1 < 2) or (3 < 2)` |
| ↓ | `\|`（按位或） | `1 \| 2 or 3` 先算 `1 \| 2` |
| ↓ | `^`、`&`、移位、`+ -`、`* / // %`、一元、`**` | 算术与位运算优先级更高 |

> 表格只列了与 `or` 相关的关键层级；完整优先级表以官方文档 "Operator precedence" 为准。

### 8.2 用实例验证

```python
print(1 or 0 and 0)        # -> 1    等价于 1 or (0 and 0) = 1 or 0 = 1
print((1 or 0) and 0)      # -> 0    手动加括号后结果不同
print(100 and 0 or 200)    # -> 200  等价于 (100 and 0) or 200 = 0 or 200
print(not 0 or 1)          # -> True 等价于 (not 0) or 1 = True or 1 = True
print(2 or 3 > 1)          # -> 2    等价于 2 or (3 > 1)
print(0 or 1 if 0 else 2)  # -> 2    等价于 (0 or 1) if 0 else 2
print(1 if 0 else 2 or 3)  # -> 2    等价于 1 if 0 else (2 or 3)
print((lambda: 100 or 200)())  # -> 100  lambda 体包含整个 or 表达式
```

### 8.3 结合性

- `or` 左结合：`a or b or c` 即 `(a or b) or c`；
- `and` 左结合：`a and b and c` 即 `(a and b) and c`；
- `not` 是一元运算符，只作用于紧邻的操作数：`not a or b` 是 `(not a) or b`，不是 `not (a or b)`。

> **写代码建议**：只要读者需要停顿 0.5 秒才能判断优先级，就加括号。`a or b and c` 这种表达式在 code review 里几乎总会被要求加括号。

---

## 9. 对偶：`and` 与 `not`

### 9.1 `and` 的规则

```text
求值 a：
    若 bool(a) 为假 → 返回 a 本身
    否则           → 求值 b，返回 b 本身
```

> **`and`：遇假即返，全真返末；`or`：遇真即返，全假返末。**

| `a` 的真值 | `a and b` 的结果 |
| --- | --- |
| 真 | `b`（继续看右边） |
| 假 | `a`（短路，右边不求值） |

```python
print(100 and 200)     # -> 200
print(0 and 200)       # -> 0
print([] and [1])      # -> []
print(100 and 0 and 999)  # -> 0   （第一个假值）
print(1 and "a" and [1])  # -> [1] （全真，返回最后一个）
```

### 9.2 `not` 一定返回 `bool`

```python
print(not 100)      # -> False
print(not 0)        # -> True
print(not [])       # -> True
print(type(not 100))# -> <class 'bool'>
```

三个运算符的返回类型对比：

| 表达式 | 返回 | 类型 |
| --- | --- | --- |
| `100 or 200` | `100` | `int` |
| `100 and 200` | `200` | `int` |
| `not 100` | `False` | `bool` |

---

## 10. 与相似写法辨析（高频混淆点）

### 10.1 `or` vs 按位或 `|`

```python
print(100 or 200)     # -> 100
print(100 | 200)      # -> 236
print(True | False)   # -> True
print(type(True | False))  # -> <class 'bool'>
```

为什么 `100 | 200 == 236`？

```text
 100 = 0b0110_0100
 200 = 0b1100_1000
 -------------------
  |  = 0b1110_1100 = 236
```

`|` 是按位运算，对两个整数的二进制位逐位取或；`or` 是逻辑运算，只看真值并返回操作数。两者完全不是一回事：

| 表达式 | 结果 | 类型 | 操作 |
| --- | --- | --- | --- |
| `100 or 200` | `100` | `int` | 逻辑或，短路 |
| `100 \| 200` | `236` | `int` | 按位或 |
| `True or False` | `True` | `bool` | 逻辑或 |
| `True \| False` | `True` | `bool` | 按位或（bool 的位运算） |

### 10.2 `a or b` vs `bool(a or b)`

```python
print(100 or 200)          # -> 100      （值上下文：拿操作数）
print(bool(100 or 200))    # -> True     （布尔上下文：显式转 bool）
```

需要布尔值时显式转换；不要指望 `or` 自动给你 `True/False`。

### 10.3 `a or b` vs 条件表达式

```python
# a 是普通变量时，两种写法结果相同
x = a or b
x = a if a else b
```

差异主要在可读性、意图表达，以及**求值次数**：

- `a or b` 简短，适合"回退/默认值"；
- `a if a else b` 更直白，适合强调"二选一"，但 `a` 在源码里出现两次：条件判断一次，条件为真、取结果时再一次。`a` 是普通变量时无感；如果 `a` 是函数调用或带副作用的属性，就会被执行两次。

想让 `a` 只求值一次，用海象运算符（3.8+）：

```python
x = v if (v := get_value()) else default
```

> 同理，`a if a is not None else b` 在 `a` 是变量时是稳妥写法；如果 `a` 是带副作用的表达式，也应改用海象运算符先存一次。

### 10.4 `or` vs `any()`

```python
print(any([0, "", []]))    # -> False
print(any([0, "", [1]]))   # -> True
print(type(any([0, 1])))   # -> <class 'bool'>
```

`any()` 是函数，**永远返回 bool**，并且可以接收可迭代对象；`or` 是运算符，返回操作数，只能写固定数量的操作数。

| 需求 | 选择 |
| --- | --- |
| 固定几个值的回退 | `a or b or c` |
| 判断"一组值里有没有真值" | `any(iterable)` |
| 判断"一组值是否全为真" | `all(iterable)` |

### 10.5 `a or b` vs `a if a is not None else b`

这是最重要的辨析：

- `a or b`：`a` 是**任意假值**（`0`、`""`、`[]`、`False`、`None`……）都会回退到 `b`；
- `a if a is not None else b`：只有 `a is None` 才回退，`0`、`""`、`[]` 都被保留。

```python
a = 0
print(a or 42)                          # -> 42   （0 被当成"没有值"）
print(a if a is not None else 42)       # -> 0    （0 是合法值，保留）
```

```python
a = ""
print(a or "default")                   # -> default
print(a if a is not None else "default")# -> ""       （空串被保留）
```

> **选型口诀**：如果 `0`、`""`、`[]` 是合法业务值，就**不要用 `or` 兜底**，用 `is None` 判断。

### 10.6 `or` vs SQL / JavaScript 的 `||`

- Python 的 `or` 与 JavaScript 的 `||` 很像：都返回操作数、都短路；
- 但两门语言的**真值规则不同**（见第 13 节），不能直接照搬；
- SQL 的 `OR` 是三值逻辑：`NULL` 参与运算时，结果可能是 `NULL` 也可能已被确定（如 `TRUE OR NULL` 为 `TRUE`），与 Python 的 `None` 语义不同。

---

## 11. 举一反三：惯用法与反模式

### 11.1 惯用法一：配置 / 环境变量回退

```python
import os

port = int(os.getenv("PORT") or 8000)
debug = (os.getenv("DEBUG") or "").lower() in {"1", "true", "yes"}
```

`os.getenv` 在变量不存在时返回 `None`，变量为空字符串时返回 `""`，两者都是假值，于是回退到默认值。这是 `or` 最经典的用法。

### 11.2 惯用法二：用户输入兜底

```python
name = input("请输入名字：").strip() or "匿名用户"
```

空输入（`""`）会回退到默认昵称。

### 11.3 惯用法三：多级回退链

```python
value = cli_arg or env_value or config_value or DEFAULT
```

从左到右取第一个非假值；如果都为假，取 `DEFAULT`。适合"命令行 > 环境变量 > 配置文件 > 默认值"的优先级链。

> 但要注意：如果某一级的"合法值"可能是 `0`、`""`、`[]`、`False`，这条链会把它跳过。此时应改写成显式的 `is not None` 判断，或使用哨兵对象。

### 11.4 惯用法四：延迟初始化 / 可选依赖

```python
client = client or build_client()
handler = handler or default_handler
```

只有在"当前没有可用对象"时才构造，避免重复开销。前提是对象为真值；如果对象可能合法地为空容器，同样有坑。

### 11.5 惯用法五：条件守卫（`and` 的舞台）

```python
if user and user.profile and user.profile.email:
    send_mail(user.profile.email)
```

逐级判空，任何一级为假就短路，不会触发 `AttributeError`。这里用 `and` 比嵌套 `if` 更紧凑。

### 11.6 惯用法六：异常/除零保护

```python
if denominator and numerator / denominator > 0.5:
    ...
```

`denominator` 为 `0` 时短路，避免 `ZeroDivisionError`。

### 11.7 反模式一：用 `or` 兜底合法假值

```python
# 需求：缓存没命中才加载
data = cache.get(key) or load_from_db(key)

# 如果缓存里合法地存了 0、""、[]、False，就会被误判为"没命中"，反复回源
```

正确写法：

```python
if key in cache:
    data = cache[key]
else:
    data = load_from_db(key)
    cache[key] = data
```

或者用哨兵对象区分"缺失"与"假值"：

```python
MISSING = object()
data = cache.get(key, MISSING)
if data is MISSING:
    data = load_from_db(key)
```

### 11.8 反模式二：函数默认参数用 `or`

```python
def add(item, bucket=None):
    bucket = bucket or []      # 调用方传 [] 时也会被替换成新列表
    bucket.append(item)
    return bucket
```

如果调用方期望"传进来的空列表被原地修改"，`bucket or []` 会破坏这个契约（`bucket` 被换成新对象，原列表不变）。更稳妥：

```python
def add(item, bucket=None):
    if bucket is None:
        bucket = []
    bucket.append(item)
    return bucket
```

> 顺带复习：**不要用可变对象做默认参数**（`def f(x=[])`），这是另一个经典坑，和 `or` 无关，但经常一起考。

### 11.9 反模式三：可变对象别名扩散

```python
a = [1]
b = a or [2]     # b is a，不是拷贝
b.append(3)
print(a)         # -> [1, 3]    a 被意外修改
```

`or` 返回的是原对象，不做拷贝。需要独立副本时用 `copy.copy` / `copy.deepcopy` / 切片。

### 11.10 反模式四：生成器/迭代器永远为真

```python
gen = (x for x in [])
value = gen or [1, 2]     # -> <generator ...>，永远拿不到 [1, 2]
```

生成器对象没有 `__len__`，真值恒为 `True`，不存在"空生成器为假"这回事。判断生成器是否为空，应先用 `list()` 物化或使用 `next(gen, default)`。

### 11.11 反模式五：优先级误读

```python
if a or b == c:
    ...
```

这解析为 `a or (b == c)`，不是 `(a or b) == c`。如果意图是后者，必须加括号。

```python
if (a or b) == c:
    ...
```

---

## 12. 自定义对象的真值：`__bool__` 与 `__len__`

```python
class Flag:
    def __init__(self, on):
        self.on = on

    def __bool__(self):
        return self.on


class Bag:
    def __init__(self, items):
        self.items = items

    def __len__(self):
        return len(self.items)


print(Flag(False) or "fallback")   # -> fallback
print(Flag(True) or "fallback")    # -> <Flag object>
print(Bag([]) or "fallback")       # -> fallback
print(Bag([1]) or "fallback")      # -> <Bag object>
```

规则细节：

1. `__bool__` 优先于 `__len__`：

   ```python
   class Both:
       def __bool__(self):
           return False

       def __len__(self):
           return 3

   print(bool(Both()))   # -> False（__bool__ 说了算）
   ```

2. `__bool__` 必须返回 `bool`，返回 `1` 会报错：

   ```python
   class BadBool:
       def __bool__(self):
           return 1

   bool(BadBool())
   # TypeError: __bool__ should return bool, returned int
   ```

3. `__len__` 必须返回 `>= 0` 的整数，返回 `-1` 会报错：

   ```python
   class BadLen:
       def __len__(self):
           return -1

   bool(BadLen())
   # ValueError: __len__() should return >= 0
   ```

4. 两个方法都没定义的对象默认为真：

   ```python
   class Plain:
       pass

   print(bool(Plain()))   # -> True
   ```

> **NumPy / pandas 特别提醒**：数组和 Series 没有"空/非空"这种真值语义，`if arr:` 的行为是"整体真值判断"；多元素数组会直接抛出 `ValueError: The truth value of an array with more than one element is ambiguous`，`arr or fallback` 同样会炸。数组场景请用 `arr.size`、`arr is not None` 或 `np.any` / `np.all` 显式判断。（本环境未安装 NumPy，此为 NumPy 的公开行为，未在本机实测。）

---

## 13. 跨语言对比：`||` 到底返回什么？

| 语言 | 表达式 | 结果 | 说明 |
| --- | --- | --- | --- |
| Python | `100 or 200` | `100` | 返回操作数；假值规则见第 5 节 |
| C / C++ | `100 \|\| 200` | `1` | 逻辑运算符返回 `int`（0 或 1），短路 |
| Java | `100 \|\| 200` | 编译错误 | 操作数必须是 `boolean`，无"真值"概念 |
| JavaScript | `100 \|\| 200` | `100` | 返回操作数，类似 Python；但真值规则不同 |
| SQL | `NULL OR TRUE` | `TRUE` | 三值逻辑；`NULL OR FALSE` 得 `NULL` |
| Shell | `cmd1 \|\| cmd2` | 退出状态 | `cmd1` 失败（非 0）才执行 `cmd2` |

### 13.1 Python vs JavaScript 的真值差异

| 值 | Python | JavaScript |
| --- | --- | --- |
| `0` / `-0` | 假 | 假 |
| `""` | 假 | 假 |
| `[]` | **假** | **真** |
| `{}` | **假** | **真** |
| `"0"` / `"false"` | 真 | 真 |
| `NaN` | **真** | **假** |
| `null` / `undefined` | `None` 假 | 假 |

这就是为什么"把 JS 的 `a || b` 习惯搬到 Python"时，空数组/空对象的处理会不一样：

```javascript
// JavaScript
[] || [1, 2]      // -> []      （空数组为真）
{} || {a: 1}      // -> {}      （空对象为真）
NaN || 1          // -> 1
```

```python
# Python
[] or [1, 2]      # -> [1, 2]   （空列表为假）
{} or {"a": 1}    # -> {"a": 1} （空字典为假）
float("nan") or 1 # -> nan      （NaN 为真）
```

### 13.2 SQL 三值逻辑速览

| 表达式 | 结果 |
| --- | --- |
| `TRUE OR NULL` | `TRUE` |
| `FALSE OR NULL` | `NULL` |
| `NULL OR NULL` | `NULL` |
| `FALSE AND NULL` | `FALSE` |

SQL 的 `OR` 返回的是布尔/三值结果，不会"返回操作数"，和 Python 的 `or` 语义完全不同。

---

## 14. 同一行代码，改一个符号：变体速查

| 代码 | 结果 | 类型 | 考点 |
| --- | --- | --- | --- |
| `x = 100 or 200` | `100` | `int` | `or` 返回第一个真值 |
| `x = 0 or 200` | `200` | `int` | 左假取右 |
| `x = 100 and 200` | `200` | `int` | `and` 遇真继续，返回右 |
| `x = 0 and 200` | `0` | `int` | `and` 遇假即返 |
| `x = 100 \| 200` | `236` | `int` | 按位或，不是逻辑或 |
| `x = bool(100 or 200)` | `True` | `bool` | 显式布尔化 |
| `x = 100 or 200 or 300` | `100` | `int` | 第一个真值即停 |
| `x = 0 or "" or []` | `[]` | `list` | 全假返回最后一个 |
| `x = 100 if 100 else 200` | `100` | `int` | 条件表达式等价写法 |

---

## 15. 练习题（附答案与解析）

### 15.1 基础题

写出下列表达式的结果和类型：

| 题号 | 表达式 | 结果 | 类型 |
| --- | --- | --- | --- |
| 1 | `100 or 200` | `100` | `int` |
| 2 | `0 or 200` | `200` | `int` |
| 3 | `"" or "0"` | `"0"` | `str` |
| 4 | `"0" or 0` | `"0"` | `str` |
| 5 | `[] or {} or 0` | `0` | `int` |
| 6 | `100 and 200` | `200` | `int` |
| 7 | `0 and 200` | `0` | `int` |
| 8 | `1 or 0 and 0` | `1` | `int` |
| 9 | `not 0 or 1` | `True` | `bool` |
| 10 | `bool(0) or "a"` | `"a"` | `str` |
| 11 | `-0.0 or "zero"` | `"zero"` | `str` |
| 12 | `float("nan") or 1` | `nan` | `float` |
| 13 | `1 or 1 / 0` | `1` | `int` |
| 14 | `0 or 1 / 0` | 抛 `ZeroDivisionError` | — |
| 15 | `x = 100 or 200; x += 1` | `101` | `int` |
| 16 | `x = True or False; type(x)` | `bool` | `type` |

### 15.2 陷阱题

**题 1**：`x = 0 or "" or [] or {} or None`，`x` 是什么？

> 答：`None`。全为假值，返回**最后一个**操作数。类型是 `NoneType`。

**题 2**：`x = "False" or 0`，`x` 是什么？`bool(x)` 呢？

> 答：`x` 是字符串 `"False"`，`bool(x)` 是 `True`。非空字符串一律为真，和字符串内容无关。

**题 3**：

```python
a = [1]
b = a or [2]
c = a or [2]
print(b is a, b is c)
```

> 答：`True True`。`a` 为真，`b = a or [2]` 返回 `a` 本身；同理 `c` 也返回 `a`。所以 `b`、`c`、`a` 是同一个列表对象，`b is a` 与 `b is c` 都为 `True`。
>
> 对比：如果 `a = []`（假值），每次 `or` 都会取右边新创建的列表字面量，`b is c` 才会是 `False`。

**题 4**：

```python
a = []
b = a or [1]
c = a or [1]
print(b is c)
```

> 答：`False`。`a` 为假，`b`、`c` 分别取右边新创建的列表字面量，是两个不同对象。

**题 5**：为什么 `x = 100 or 200` 在 `dis` 里看不到 `200`？

> 答：两个操作数都是常量，CPython 在编译期做了常量折叠，直接把结果 `100` 写进字节码。这是实现优化，不影响语言语义；即使不折叠，短路规则也会让 `200` 不被选中。

**题 6**：`if x or y == 0:` 和 `if (x or y) == 0:` 一样吗？

> 答：不一样。前者解析为 `x or (y == 0)`，后者是 `(x or y) == 0`。`==` 的优先级高于 `or`。

**题 7**：`def f(x=None): x = x or []` 有什么风险？

> 答：调用方传入 `[]` 或 `""`、`0` 等合法假值时，会被替换成新的 `[]`，破坏"原地修改"的契约；如果调用方期望 `x` 仍是传入对象，行为会不符合预期。用 `if x is None:` 更稳。

**题 8**：`data = cache.get(key) or load(key)` 在什么情况下会重复回源？

> 答：当缓存中合法地存了 `0`、`""`、`[]`、`False` 等假值时，`or` 会认为"没命中"并再次调用 `load`。应改用 `key in cache` 或哨兵对象。

**题 9**：`gen = (i for i in []); gen or "fallback"` 的结果？

> 答：返回生成器对象本身。生成器恒为真，`"fallback"` 不会被采用。需要判断空否，用 `list(gen)` 或 `next(gen, default)`。

**题 10**：自定义类只实现 `__len__` 返回 0，`obj or "x"` 返回什么？

> 答：返回 `"x"`。没有 `__bool__` 时，`__len__` 返回 0 视为假。

### 15.3 编程题

**题 11**：实现"取第一个非空字符串"，输入可能是 `None`、`""`、空白串。

```python
def first_non_blank(*values, default="未填写"):
    for v in values:
        if v and v.strip():
            return v
    return default
```

> 说明：这里用 `if v and v.strip()` 而不是 `v or ...`，因为 `v or ...` 无法在循环里表达"跳过空串继续看下一个"。`and` 在这里同时完成判空与去空白判断。

**题 12**：用 `or` 写一个"取环境变量，缺失或为空则用默认值"的函数，并说明它与 `os.getenv("X", default)` 的区别。

```python
import os

def env_or(name, default):
    return os.getenv(name) or default
```

> 区别：`os.getenv(name, default)` 只在变量**不存在**时用默认值；变量存在但为空字符串时返回 `""`。而 `env_or` 在变量为空字符串时也会回退到默认值。按业务需求选择。

---

## 16. 一页速查卡

### 16.1 三条核心规则

```text
1. a or b ：a 为真 → 返回 a；a 为假 → 返回 b（b 仅在 a 为假时求值）
2. a and b：a 为真 → 返回 b；a 为假 → 返回 a（b 仅在 a 为真时求值）
3. not a  ：一定返回 bool
```

### 16.2 两个口诀

```text
or ：遇真即返，全假返末
and：遇假即返，全真返末
```

### 16.3 假值清单

```text
False  None  0  0.0  -0.0  0j  Decimal(0)  Fraction(0,1)
""  b""  []  ()  {}  set()  frozenset()  range(0)
```

### 16.4 默认值写法选型

| 需求 | 推荐写法 |
| --- | --- |
| 任意假值都回退 | `a or b` |
| 只有 `None` 回退 | `a if a is not None else b` |
| 多级回退 | `a or b or c or DEFAULT` |
| 需要布尔结果 | `bool(a or b)` |
| 缓存命中判断 | `key in cache` / 哨兵对象，不要用 `or` |
| 判断一组值 | `any(...)` / `all(...)` |

### 16.5 一句话答题模板

> `x = 100 or 200` 中，`100` 是真值，`or` 短路并返回左操作数本身，所以 `x == 100`，类型为 `int`，`bool(x) == True`；`or` 不返回布尔值，也不把结果转换成布尔值。

---

## 17. 延伸阅读（官方文档）

- [Boolean operations — `and`, `or`, `not`](https://docs.python.org/3/reference/expressions.html#boolean-operations)
- [Truth value testing](https://docs.python.org/3/library/stdtypes.html#truth-value-testing)
- [Operator precedence](https://docs.python.org/3/reference/expressions.html#operator-precedence)
- [`object.__bool__` / `object.__len__`](https://docs.python.org/3/reference/datamodel.html#object.__bool__)
- [`dis` — Disassembler for Python bytecode](https://docs.python.org/3/library/dis.html)

---

**最后再回答一次开头的问题：**

```python
x = 100 or 200
# x == 100
# type(x) == int
# bool(x) == True
```

`or` 返回的是**操作数**，不是布尔值；`100` 为真，所以 `x` 就是 `100`。记住这一点，再记住"假值 = 零、空、None、False"，这一类题就不会再错。
