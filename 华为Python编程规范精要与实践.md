# 华为 Python 编程规范：精要、示例与实践指南

> **文档定位**：面向华为/昇腾技术生态的 Python 工程实践整理。以华为《开放可编程开发指南》**13.3 Python 开发规范**为骨架，融合昇腾社区《python 语言编程指导（建议稿）》《python 语言安全编程指导》，并对照 PEP 8 给出「华为口径 vs 业界口径」的差异说明。
>
> **适用版本**：Python 3.8 / 3.9 / 3.10 / 3.11（规范原文适用 3.8–3.11；本文示例使用 Python 3 语法）。涉及 Python 2 特有规则的条目会**显式标注版本语境**。
>
> **配套材料**：同目录下《华为Python编程规范-选择题.md》共 **62 题**（单选 22、多选 31、情景单选 9），答案与解析默认折叠，用于自测与巩固。

---

## 阅读指南

| 符号 | 含义 |
| :--: | :--- |
| 🔴 **规则（must）** | 编程时**必须**遵守，属于评审/红线检查项，违反即视为缺陷 |
| 🟡 **建议（should）** | 编程时**应该**遵守，允许有充分理由的例外，但例外会破坏一致性 |
| 💡 **理由** | 规范背后的原因。理解原因才能正确判断例外 |
| ✅ **Good** / ❌ **Bad** | 正例 / 反例代码 |
| 📌 **易错点** | 实际评审中最常踩的坑 |
| 🔗 **出处** | 对应规则编号：HW = 华为 13.3 节；AS = 昇腾风格规范；SEC = 昇腾安全规范；PEP8 = 业界通用 |

**约定与效力**：

1. **规则**与**建议**都必须理解其规定的原因，并努力遵守；例外应当是极少的。
2. 在不违背总体原则、经过充分考虑且有充足理由时，可以适当违背规范；但**例外破坏一致性**，评审时需要显式说明。
3. **风格一致性优先**：修改或适配外部开源代码、第三方代码时，应遵守该代码库已有的规范，保持风格统一，不要"顺手格式化"拖入无关改动。
4. 规范不是完美的；禁止某些特性可能影响个别场景的实现效率，但规范的目的是"让大多数程序员获得更大的整体收益"。

---

## 目录

- [第 0 章 总览：华为 Python 规范的体系与效力](#第-0-章-总览华为-python-规范的体系与效力)
- [第 1 章 排版与格式](#第-1-章-排版与格式)
- [第 2 章 导入（Imports）](#第-2-章-导入imports)
- [第 3 章 注释与文档字符串](#第-3-章-注释与文档字符串)
- [第 4 章 命名规范](#第-4-章-命名规范)
- [第 5 章 编码实践](#第-5-章-编码实践)
- [第 6 章 异常处理](#第-6-章-异常处理)
- [第 7 章 并发与并行](#第-7-章-并发与并行)
- [第 8 章 安全编码红线](#第-8-章-安全编码红线)
- [第 9 章 测试与工程实践](#第-9-章-测试与工程实践)
- [第 10 章 工具链与自动化](#第-10-章-工具链与自动化)
- [第 11 章 常见反模式 Top 20](#第-11-章-常见反模式-top-20)
- [附录 A 速查表](#附录-a-速查表)
- [附录 B 提交前自检清单](#附录-b-提交前自检清单)
- [附录 C 参考来源](#附录-c-参考来源)

---

## 第 0 章 总览：华为 Python 规范的体系与效力

### 0.1 规范体系：一份骨架 + 两套细化

华为内部的 Python 编码要求通常由两条线组成，对外公开可查的材料呈现为以下三类文档，本文把它们合成一套可执行的清单：

| 文档 | 定位 | 覆盖内容 | 本文引用代号 |
| :--- | :--- | :--- | :---: |
| 华为《开放可编程开发指南》13.3 Python 开发规范 | **骨架**：条款式规则，与产品开发红线挂钩 | 排版、注释、命名、编码、异常、并发 | `HW 13.3.x` |
| 昇腾社区《python 语言编程指导（建议稿）》 | **细化**：在骨架基础上补齐行宽、命名总表、推导式、类型检查、测试用例等 | 排版、注释、命名、编码、异常、测试用例 | `AS x.x` |
| 昇腾社区《python 语言安全编程指导（建议稿）》 | **安全红线**：安全编码专项 | 数据校验、异常、文件、序列化、运行环境、密码学 | `SEC x.x` |

> 三份文档的口径基本一致，细节处互为补充。例如"一行只写一条语句"只在 HW 中作为规则出现；"推导式不超过 3 个 for"只在 AS 中出现；"禁止对不可信数据 unpickle"只在 SEC 中出现。**工程上应当取三者的并集**。

### 0.2 一张表看懂：华为口径 vs PEP 8

| 维度 | 华为/昇腾口径 | PEP 8 口径 | 差异说明 |
| :--- | :--- | :--- | :--- |
| 缩进 | 4 个空格，**禁止** Tab | 4 个空格 | 一致 |
| 空格/Tab 混用 | **禁止** | 禁止 | 一致 |
| 行宽 | **≤ 120 字符**（AS 规则 1.4，原文"小于 120"） | ≤ 79 字符 | 华为/昇腾放宽到 120，便于工程实践 |
| 一行多语句 | **禁止**（HW 13.3.1.3 规则） | 不推荐 | 华为上升为"规则" |
| 空行 | 独立程序块之间、变量说明之后**必须**加空行 | 顶层定义间 2 行、方法间 1 行 | 华为强调"程序块之间"，更贴近可读性 |
| 导入 | 每个模块**单独一行**；**禁止** `import *` | 每个导入一行；避免 `import *` | 昇腾规范把 `import *` 上升为禁止（AS 规则 1.7） |
| 编码 | 文件必须是 UTF-8，可加 `# coding: utf-8` | 默认 UTF-8 | 一致 |
| 命名 | 模块/函数/变量 `lower_with_under`；类 `CapWords`；常量 `CAPS_WITH_UNDER` | 同 | 一致 |
| 与 None 比较 | 用 `is` / `is not` | 用 `is` / `is not` | 一致 |
| 类型检查 | 用 `isinstance`，不用 `type` | 用 `isinstance` | 一致 |
| 裸 `except:` | **禁止**；也避免直接用 `Exception` | 不推荐 | 华为更严格 |
| 异常捕获范围 | try 只包必须就地处理的语句，异常向外透传 | 类似 | 一致 |
| 注释量 | 有效注释（含 docstring）**建议 ≥ 20%** | 无量化要求 | 华为有量化指标 |
| 文档字符串 | 类/公共函数 docstring 写在声明下一行、缩进 4 空格；多行时结尾 `"""` 自成一行 | 同 | 一致 |

### 0.3 学习路径建议

```text
① 排版/命名/注释（第 1–4 章）      —— 先让代码"长得对"，评审零低级问题
        ↓
② 编码实践/异常（第 5–6 章）        —— 让代码"行为对"，可维护、可调试
        ↓
③ 并发/安全（第 7–8 章）            —— 让代码"跑得稳、扛得住攻击"
        ↓
④ 测试/工具链（第 9–10 章）         —— 让规范"落得下去"，靠工具而不是靠记忆
```

### 0.4 全文高频术语

| 术语 | 含义 |
| :--- | :--- |
| **信任边界** | 数据从"不可信"区域进入"可信"区域的分界线。Web 请求参数、网络报文、用户输入、外部文件都来自信任边界之外 |
| **白名单校验** | 只接受"已知合法"的数据（正向校验），优于"拒绝已知非法"的黑名单 |
| **名字修饰（name mangling）** | `__name` 形式的成员被解释器改名为 `_ClassName__name`，用于避免继承时的名字冲突，**不是权限控制** |
| **LEGB** | 名字解析顺序：Local → Enclosed（闭包）→ Global → Builtin |
| **可变默认参数** | `def f(items=[])` 中的 `[]` 只在函数定义时求值一次，会被所有调用共享 |
| **GIL** | 全局解释器锁。CPython 中同一时刻只有一个线程执行 Python 字节码，因此多线程不能加速 CPU 密集计算 |

---

## 第 1 章 排版与格式

> 排版的目标不是"好看"，而是让代码在**评审、调试、diff、多人协作**中降低理解成本。本章所有条款都是低成本、高收益的，应在提交前由格式化工具兜底。

### 1.1 缩进：4 个空格，禁止 Tab

🔴 **规则**：程序块采用缩进风格编写，缩进的空格数为 **4 个**（HW 13.3.1.1、AS 规则 1.1）。
🔴 **规则**：**禁止混合使用空格（space）和跳格（Tab）**；已有的混合代码要全部转换为空格（AS 规则 1.2）。

💡 **理由**：不同编辑器/查看器的 Tab 宽度不一致，混用会让代码在不同环境下"看起来"层级不同，甚至引发 `TabError`（Python 3 严禁缩进中混用空格与 Tab）。

```python
# ✅ Good：统一 4 个空格
def process_request(request):
    if request.is_valid():
        data = request.parse()
        return data
    return None
```

```text
# ❌ Bad 示意：缩进用了 Tab，且与空格混用（肉眼难以分辨，运行时报 TabError）
# 下面用 [TAB] 标记 Tab 字符；真实代码中必须全部替换为空格
def process_request(request):
[TAB]if request.is_valid():
[TAB]    data = request.parse()
        return data
    return None
```

> 注意：上面这段"Bad"示例因缩进非法，**不能直接运行**，只用于展示"Tab 与空格混用"的样子。真实文件中请用编辑器的"显示空白字符"确认。

🔗 **工具配置**：PyCharm 关闭 *Smart tabs*；VS Code 设置 `"editor.insertSpaces": true, "editor.tabSize": 4`；提交前可用 `python -m tabnanny` + 文件路径 检查混用。

📌 **易错点**：
- PyCharm 的 *Smart tabs* 会"空格不够就用 Tab 补"，要关闭：`Settings → Editor → Code Style → Python → Tabs and Indents`，取消勾选 *Smart tabs*（AS 规则 1.2 说明给出的设置路径）。
- 从网页/聊天工具复制代码时经常带入 NBSP（不换行空格）或 Tab，提交前用编辑器"显示空白字符"检查一遍。

### 1.2 行宽：不超过 120 字符，优先括号续行

🔴 **规则**：一行长度**小于 120 个字符**（AS 规则 1.4）。
🟡 **建议**：较长的语句、表达式或参数要分成多行书写；**首选使用括号**（`()`、`[]`、`{}`）内的隐式续行，其次才用反斜杠 `\` 显式续行。
🟡 **建议**：长表达式要在**低优先级操作符处**划分新行；操作符统一放在新行行首或原行行尾（全项目保持一致）；续行要适当缩进，使排版整齐。

> **口径说明（重要）**：PEP 8 的传统要求是 79 字符，Python 标准库按 80 字符排版。昇腾规范 1.4 的表述是"行宽小于 120 字符，与 Python 标准库（80 字符）看齐"——其**实际要求是 120**，括号中的 80 是对业界惯例的说明；昇腾规范 1.4 的说明也建议把 PyCharm 的 *Hard wrap at* 设为 **120**。
>
> 工程上的正确理解是：**以 120 为上限**，并保持团队内部一致——不要一部分文件按 80、一部分按 120，否则 diff 与评审都会被格式噪声淹没。

```python
# ✅ Good：括号内隐式续行，操作符放在行首，续行缩进对齐
if (width == 0
        and height == 0
        and color == "red"
        and emphasis == "strong"
        and highlight > 100):
    render()
```

```python
# ✅ Good：函数参数多行，每个参数一行、以逗号结尾（便于 diff）
def create_connection(
    host,
    port,
    user,
    password,
    timeout=30,
    retries=3,
):
    ...
```

```text
# ❌ Bad 示意：一行过长；且用反斜杠续行（可读性、可维护性都差）
if width == 0 and height == 0 and color == "red" and emphasis == "strong" and highlight > 100:   # 超过 120 字符
    render()
```

> 反斜杠续行**语法上合法**（`result = a + b + \` 换行后接 `c` 可以运行），但反斜杠后一旦多了空格就会变成 `SyntaxError`，而且对格式化工具与 diff 都不友好。**优先用括号**：

```python
# ✅ Good：括号内隐式续行，无需反斜杠
result = (
    first_value
    + second_value
    + third_value
)
```

📌 **易错点**：
- **反斜杠后不能有任何字符（包括空格和注释）**，否则续行失效。
- 括号内的最后一个元素后面加逗号是合法的（trailing comma），对 diff 友好，Black 等格式化器也默认这样处理。

### 1.3 空行：独立程序块之间、变量说明之后必须加空行

🔴 **规则**：相对独立的程序块之间、变量说明之后**必须**加空行（HW 13.3.1.4、AS 规则 1.3）。

💡 **理由**：空行是"视觉分段"，它把"一段逻辑"和"另一段逻辑"隔开，显著提升可读性。没有空行的代码像一堵墙。

```python
# ✅ Good
if len(device_name) < MAX_NAME_LEN:
    logger.warning("device name too short")

writer = LogWriter()
writer.start()
```

```python
# ❌ Bad：两个独立程序块、变量声明后都没有空行，全挤在一起
if len(device_name) < MAX_NAME_LEN:
    logger.warning("device name too short")
writer = LogWriter()
writer.start()
```

🔗 **补充（PEP 8 口径，工程上同样推荐）**：顶层函数/类定义之间空 **2 行**；类内方法之间空 **1 行**；函数内部用空行分隔逻辑段落（节制使用）。

### 1.4 空格：双目操作符前后加空格

🔴 **规则**：在两个以上的关键字、变量、常量进行对等操作时，它们之间的操作符**前后要加空格**（HW 13.3.1.5）。

细化要求：

| 场景 | 正确写法 | 错误写法 |
| :--- | :--- | :--- |
| 逗号、分号 | `print(a, b, c)` | `print(a,b , c)` |
| 比较操作符 | `if current_time >= MAX_TIME_VALUE:` | `if current_time>= MAX_TIME_VALUE:` |
| 赋值操作符 | `a = b + c`、`a += 2` | `a=b+ c`、`a+=2` |
| 算术操作符 | `a = b + c` | `a=b+c`（可读性差） |
| 逻辑操作符 | `if a and b or c:` | `if a  and  b  or  c:`（多余空格让视觉割裂；不要为"对齐"连敲空格） |
| 函数调用/关键字参数 | `func(a, b, key=value)` | `func(a,b,key = value)` |
| 切片 | `items[1:3]`、`items[::2]` | `items[1 : 3]`（PEP 8：切片冒号两侧不加空格） |
| 默认参数 | `def f(a, b=1):` | `def f(a, b = 1):` |

🟡 **建议**：给操作符留空格时**不要连续留一个以上空格**；在超长语句中，"整体清晰、局部紧凑"可接受。

🟡 **`*`、`**` 的特例**：作为操作符时前后可以加空格；但若与**更低优先级的操作符**同时使用且**不涉及括号**，建议前后不加空格（HW 13.3.1.5）：

```text
# ✅ Good
a = b * c
a = c ** b
x = x*2 - 1      # * 与更低优先级的 - 混用、无括号：* 前后不加空格，突出优先级

# ❌ Bad：空格使用不一致，视觉上割裂
print(a,b , c)
a=b+ c
a+=2
if current_time>= MAX_TIME_VALUE:   # 示意：此处省略分支体
```

> 说明：这段是"空格写法对照"，不是可直接执行的完整代码（最后一行 `if` 没有分支体），因此放在 `text` 代码块中展示。

📌 **易错点**：
- 只给 `+` 加空格、不给 `=` 加，属于"局部规范"，评审会打回；应整行统一。
- 不要用空格做"对齐美感"（如把 `=` 排成列），diff 会产生大量噪声，格式化工具也会改回去。

### 1.5 一行只写一条语句

🔴 **规则**：**一行只写一条语句**，不允许把多个短语句写在一行中（HW 13.3.1.3）。

💡 **理由**：多条语句挤在一行，**调试时无法单步执行**、断点无法精确定位，异常堆栈的行号也会指向"一行里的多条语句"。

```python
# ❌ Bad：一行多条语句
a = 1; b = 2
if a == 1: b = 3
for i in range(3): print(i)

# ✅ Good
a = 1
b = 2

if a == 1:
    b = 3

for i in range(3):
    print(i)
```

📌 **易错点**：`if x: return` 这种"单行 if"虽然语法合法、在代码高尔夫里很常见，但在华为规范下属于"一行多条语句"，评审不通过。

### 1.6 文件编码：UTF-8

🔴 **规则**：Python 文件**必须使用 UTF-8 编码**；文件头可以添加编码声明 `# coding: utf-8`（HW 13.3.1.2）。

💡 **理由**：Python 3 源码默认就是 UTF-8；显式声明是给"历史环境、跨平台工具链、含中文注释的文件"兜底，避免在某些编辑器/CI 环境下按 GBK 解码导致乱码或 `SyntaxError`。

```python
# ✅ Good：文件头（可选但推荐，尤其是含中文注释/字符串时）
# coding: utf-8
"""订单处理模块。

功能：提供订单创建、支付、取消的领域逻辑。
版权信息：华为技术有限公司，版权所有 (C) 2024
"""

import logging
```

📌 **易错点**：文件里出现中文注释后，被 Windows 默认 ANSI 编码重新保存，是"本地好好的、CI 编译报错"的高发原因。提交前确认编辑器右下角显示 `UTF-8`（无 BOM）。

### 1.7 第 1 章速记

```text
4 空格缩进、不用 Tab、不混用
UTF-8（可写 # coding: utf-8）
一行一条语句（调试可单步）
独立程序块 / 变量说明后加空行
双目操作符前后加空格；逗号后加空格；切片冒号不加空格
* 与更低优先级运算符连用不加空格：x = x*2 - 1
行宽 ≤ 120，优先括号续行，操作符放行首或行尾（团队统一）
```

---

## 第 2 章 导入（Imports）

### 2.1 每个模块单独一行

🔴 **规则**：加载模块必须**分开、每个模块占一行**（HW 13.3.1.6、AS 规则 1.6）。

💡 **理由**：单独一行让程序依赖更清晰，代码评审、依赖分析、`grep` 定位都更容易；一行导入多个模块会让"到底用了谁"变得模糊。

```python
# ✅ Good
import sys
import os

# ✅ Good：同一模块内的多个符号可以在同一行导入
from sys import stdin, stdout

# ❌ Bad：一行导入多个模块
import sys, os
```

📌 **易错点**：`import sys, os` 是很多老代码的风格，评审时会被打回；`from sys import stdin, stdout` **不算**违规（规则只约束"模块"分行，不约束"同一模块的符号"）。

### 2.2 禁止 `from xxx import *`

🔴 **规则**：**禁止**使用 `from xxx import *` 的方式导入某个模块的所有成员（AS 规则 1.7；华为 13.3 节未单列此条，工程上按同一红线执行）。

💡 **理由**：`import *` 会把对方模块中所有**非下划线开头**的名字挨个赋给当前作用域的同名变量。如果当前作用域已有同名变量，会**静默覆盖**；冲突后极难定位。它还会让静态检查工具（flake8 的 F403/F405）失效，让"这个名字从哪来"无法回答。

```python
# ❌ Bad：覆盖了本模块的 process 函数，且无法从代码看出 process 来自哪里
from utils import *
from handlers import *

def process(data):          # 若 utils/handlers 中也有 process，此处的定义会覆盖导入进来的同名对象；读代码看不出这个函数到底来自哪里
    ...
```

```python
# ✅ Good：显式导入所需符号
from utils import normalize_path
from handlers import handle_request

def process(data):
    ...
```

📌 **易错点**：
- 需要 `import *` 的典型"借口"是"符号太多"——正确做法是导入模块本身 `import utils`，然后用 `utils.xxx` 限定，或显式列出符号。
- `from typing import *` 同样禁止。

### 2.3 导入的位置：模块 docstring 之后、全局变量之前

🔴 **规则**：导入部分（imports）置于**模块注释和文档字符串之后、模块全局变量和常量声明之前**（HW 13.3.1.7）。

💡 **理由**：使文件结构固定为"文件说明 → 依赖 → 常量/全局变量 → 代码"，读者一眼就能看清"这个文件依赖什么、暴露什么"。

```python
# ✅ Good：标准文件骨架
# coding: utf-8
"""设备管理模块。

功能：封装设备注册、心跳、下线流程。
版权信息：华为技术有限公司，版权所有 (C) 2024
"""

import logging
import os

from device.driver import DeviceDriver     # 本地/项目内模块

LOGGER = logging.getLogger(__name__)
MAX_RETRY = 3
device_cache = {}
```

### 2.4 导入分组与顺序（工程推荐）

🔴 规则只要求"位置正确、每模块一行"，但工程上推荐按以下**三段式分组**，组间空一行，组内按字母序（`isort` 默认行为）：

```python
# ✅ Good：标准库 → 第三方库 → 项目内模块
import logging
import os
from typing import Optional

import requests
from pydantic import BaseModel

from myproject.common import config
from myproject.utils import retry
```

| 组 | 内容 | 示例 |
| :--- | :--- | :--- |
| 标准库 | Python 自带 | `os`、`sys`、`json`、`logging`、`typing` |
| 第三方库 | 通过 pip 安装 | `requests`、`numpy`、`pydantic` |
| 项目内模块 | 本仓库 | `myproject.xxx` |

📌 **易错点**：
- 相对导入（`from . import x`）与绝对导入混用，团队应先约定一种；大项目通常推荐绝对导入。
- 不要在函数内部做"为了省事"的导入（循环依赖除外）；函数内导入会掩盖依赖关系，也让启动期错误延后暴露。**例外**：为规避循环导入、或延迟加载重量级可选依赖时，可局部导入，并加注释说明原因。

### 2.5 第 2 章速记

```text
每个模块一行 import；同一模块的多个符号可以同行
禁止 from xxx import *
导入位置：模块 docstring 之后、全局变量/常量之前
分组顺序：标准库 → 第三方 → 项目内，组间空一行（isort 可自动做）
```

---

## 第 3 章 注释与文档字符串

### 3.1 原则：为什么注释、注释多少

🟡 **建议**：注释和文档字符串必须**有助于对程序的阅读理解**；描述必须**准确、易懂、简洁、没有二义性**（AS 第 2 节·注释原则）。
🟡 **建议**：有效注释量（包括文档字符串）**建议在 20% 以上**。
🟡 **建议**：
1. 避免在注释和文档字符串中使用缩写；如必须使用，要有必要的说明。
2. **修改代码时，始终优先更新相应的注释/文档字符串**，保证注释与代码一致。
3. 有含义的变量，如果不能"自注释"，则需要添加必要的注释。
4. 全局变量**建议**添加详细注释，包括功能、取值范围、哪些函数会修改它、存取时的注意事项。

💡 **理由**：Python 是动态语言，没有类型信息，IDE 无法给出足够提示。没有注释的动态语言代码非常难理解。注释的价值在于说明"**为什么**"（意图、约束、坑），而不是复述"**是什么**"（代码本身已经说了）。

```python
# ❌ Bad：注释复述代码，零信息量
i = i + 1          # i 加 1
name = "Tom"       # 把 name 赋值为 Tom

# ✅ Good：注释说明"为什么"和"坑"
retry_count += 1   # 设备侧限流窗口为 60s，此处最多重试 3 次
```

```python
# ✅ Good：全局变量带详细注释
# 设备缓存：key 为设备序列号（str，24 位十六进制），value 为 DeviceStatus 对象。
# 仅由 DeviceRegistry.refresh() 在持有 registry_lock 时写入；
# 读取方必须通过 get_device() 访问，禁止直接遍历，避免读到刷新中的中间状态。
_DEVICE_CACHE = {}
```

📌 **易错点**：
- "代码改了、注释没改"比没有注释更危险——它会主动误导后面的人。评审时应把"注释与代码一致性"当作检查项。
- 注释里出现无法解释的缩写（`cfg`、`mgr`、`tmp`），新人维护成本极高。

### 3.2 模块文档字符串：文件顶部、import 之前

🔴 **规则**：模块文档字符串写在**文件顶部、导入（import）部分之前**，**不需要缩进**；内容应包含**功能描述和版权声明**（HW 13.3.2.4）。

```python
# ✅ Good
"""设备管理模块。

功能：封装设备注册、心跳保活、下线清理的完整生命周期。
版权信息：华为技术有限公司，版权所有 (C) 2024
"""

import logging
```

📌 **易错点**：模块 docstring 必须在**任何语句之前**（包括 `import`、`# coding` 注释除外）。如果把它放到 `import` 之后，它就不再是模块 docstring，`help(module)` 和 `__doc__` 都取不到。

### 3.3 类和接口的文档字符串

🔴 **规则**：类/接口的文档字符串写在**类声明（`class ClassName:`）所在行的下一行**，并**向后缩进 4 个空格**（HW 13.3.2.1）。

💡 内容可选择包括（但不限于）：功能描述、接口清单等。功能描述除了描述类/接口功能，还要写明**与其他类/接口之间的关系**；接口清单列出该类的接口方法描述。

```python
# ✅ Good
class DeviceAdapter:
    """设备适配器，屏蔽不同厂商设备的协议差异。

    功能描述：
        将 NETCONF / SNMP / CLI 三类驱动统一为 DeviceAdapter 接口，
        上层业务只依赖本接口，不感知具体协议。
    接口：
        connect()      建立连接
        disconnect()   释放连接
        get_status()   查询设备状态
    关系：
        由 DeviceAdapterFactory 根据设备型号创建，生命周期与 DeviceManager 一致。
    """
```

📌 **易错点**：docstring 和 `class` 声明之间不能有空行（Python 语法上 docstring 必须是类体的第一条语句）；缩进必须是类体缩进（4 空格），而不是顶格。

### 3.4 公共函数的文档字符串

🔴 **规则**：公共函数的文档字符串写在**函数声明（`def FunctionName(self):`）所在行的下一行**，并**向后缩进 4 个空格**（HW 13.3.2.2）。

💡 内容可选择包括（但不限于）：**功能描述、输入参数、输出参数、返回值、调用关系（函数、表）、异常描述**等。**异常描述除描述函数内部抛出的异常外，还必须说明异常的含义及什么条件下抛出该异常。**

```python
# ✅ Good：公共函数的标准 docstring
def load_batch(fpath, batch_size=100):
    """从文件加载一批数据。

    功能描述：
        按 batch_size 逐行读取文本文件，跳过空行，返回字符串列表。
    参数：
        fpath (str)：文件路径，必须是 UTF-8 编码的普通文件。
        batch_size (int)：单批最大读取行数，取值 1–10000，默认 100。
    返回值：
        list[str]：读取到的非空行；文件为空时返回 []。
    异常描述：
        FileNotFoundError：fpath 指向的文件不存在时抛出。
        PermissionError：当前进程无读权限时抛出。
        UnicodeDecodeError：文件不是 UTF-8 编码时抛出。
    调用关系：
        由 DataPipeline.run() 调用；内部调用 open()。
    """
```

### 3.5 多行文档字符串：结尾 `"""` 自成一行

🔴 **规则**：文档字符串**多于一行时，末尾的 `"""` 要自成一行**；对于只有一行的文档字符串，把 `"""` 放在同一行没问题（HW 13.3.2.5）。

```python
# ✅ Good：多行，结尾 """ 独占一行
"""
Return a foobang.

Optional plotz says to frobnicate the bizbaz first.
"""

# ✅ Good：单行，可以同行闭合
"""API for interacting with the volume manager."""
```

```python
# ❌ Bad：多行 docstring 的结尾 """ 没有自成一行
"""Return a foobang
Optional plotz says to frobnicate the bizbaz first."""
```

### 3.6 注释的缩进、位置与行内注释

🔴 **规则**：公共属性的注释写在**属性声明的上方**，与声明**保持同样的缩进**。行内注释应以 `#` 和**一个空格**作为开始，与后面的文字注释以一个空格隔开（HW 13.3.2.3）。
🔴 **规则**：注释必须与其描述的代码**保持同样的缩进**，并放在其**上方相邻位置**（HW 13.3.2.6）。
🟡 **建议**：行内注释（语句末尾的注释）要**少用**。

```python
# ❌ Bad：注释顶格，与代码缩进不一致；# 后没有空格
if image_service is not None:
# Deletes the image if it is in queued or saving state
    self._delete_image(context, image_meta["id"], image_service)

# ✅ Good：注释与代码同缩进，位于代码上方相邻行；# 后有空格
if image_service is not None:
    # Deletes the image if it is in queued or saving state.
    self._delete_image(context, image_meta["id"], image_service)

# ✅ Good：属性注释写在声明上方，保持同样缩进；行内注释 # + 一个空格
class DeviceConfig:
    # 心跳周期（秒），取值 5–300，由配置中心下发
    heartbeat_interval = 30

    def __init__(self, name):
        self.name = name  # 设备唯一名称，创建后不可变
```

📌 **易错点**：
- `#注释`（`#` 后无空格）是常见低级问题，flake8 的 E262/E265 会报。
- 注释放在代码**下方**或跨了多行代码，都属于"位置错误"。
- 行内注释不要把行撑过 120 字符。

### 3.7 注释的反模式

| 反模式 | 问题 | 正确做法 |
| :--- | :--- | :--- |
| 注释复述代码 | 零信息量，还会随代码过时 | 解释"为什么这样做/有什么约束" |
| 注释掉的代码 | Python 中可被轻易恢复，安全红线禁止；也是"僵尸代码" | **彻底删除**，版本库有历史（SEC 6.8） |
| 过时的注释 | 主动误导，比没有更糟 | 改代码必须同步改注释 |
| 无意义缩写 | 只有作者看得懂 | 用完整单词，确需缩写要说明 |
| 大段 ASCII 分隔线 | 噪声，diff 冲突高发 | 用空行和函数/类边界表达结构 |
| TODO 没有负责人/单号 | 永远没人处理 | `# TODO(zhangsan, ISSUE-123): ...` |

### 3.8 第 3 章速记

```text
有效注释（含 docstring）建议 ≥ 20%
模块 docstring 在文件顶部 import 之前，不缩进，含功能+版权
类 docstring 在 class 下一行、缩进 4 空格，含功能/接口清单/关系
公共函数 docstring 在 def 下一行，含功能/参数/返回值/异常（异常要说明何时抛）
公共属性注释写在声明上方、同缩进
多行 docstring 结尾 """ 自成一行；单行可同行闭合
注释与代码同缩进、放上方相邻行；# 后有一个空格；行内注释少用
改代码必须同步改注释；禁止注释掉代码
```

---

## 第 4 章 命名规范

> 命名是"最便宜也最贵"的注释：一个好名字抵消一段解释，一个坏名字制造长期误解。华为规范的核心是"**按类型选风格**"——先看清这个名字是模块、类、函数、变量还是常量，再套用对应风格。

### 4.1 命名总表（必须背下来的一张表）

| 类型（Type） | 公共（Public） | 内部（Internal） |
| :--- | :--- | :--- |
| 模块（Modules） | `lower_with_under` | `_lower_with_under` |
| 包（Packages） | `lower_with_under` | 不涉及 |
| 类（Classes） | `CapWords` | `CapWords` |
| 异常（Exceptions） | `CapWords` | 不涉及 |
| 函数/方法（Functions/Methods） | `lower_with_under()` | `_lower_with_under()` |
| 全局/类常量（Global/Class Constants） | `CAPS_WITH_UNDER` | `_CAPS_WITH_UNDER` |
| 全局/类变量（Global/Class Variables） | `lower_with_under` | `lower_with_under` |
| 实例变量（Instance Variables） | `lower_with_under` | `_lower_with_under` 或 `__lower_with_under`（需名字修饰时） |
| 函数/方法参数（Parameters） | `lower_with_under` | `_lower_with_under` 或 `__lower_with_under`（需名字修饰时） |
| 局部变量（Local Variables） | `lower_with_under` | `_lower_with_under` 或 `__lower_with_under`（需名字修饰时） |

**缩写约定**：

- `lower_with_under`：小写字母 + 下划线，如 `sample_module`
- `CapWords`：大写字母开头的大驼峰，如 `SampleClass`
- `CAPS_WITH_UNDER`：大写字母 + 下划线，如 `MAX_RETRY`
- 单下划线开头：暗示**仅供内部使用**，外部不应访问
- 双下划线开头：会被解释器**改名**，用于避免派生类成员重名

🔗 **出处**：HW 13.3.3.1–13.3.3.5；AS 第 3 节命名规范表。

### 4.2 包 / 模块：`lower_with_under`

🔴 **规则**：包（Package）、模块（Module）名使用**意义完整的英文描述**，采用**小写加下划线**（`lower_with_under`）的风格命名（HW 13.3.3.1）。

💡 **理由**：模块名本质上是"被 import 的名字"。历史上有大量 `CapWords.py` 的模块，但如今不鼓励——如果模块名碰巧和类名一致（`import SampleClass` 然后 `SampleClass()`），读者会分不清拿到的是模块还是类。

```python
# ✅ Good：文件名 device_manager.py、包名 protocol_adapters/
from sample_package import sample_module
from sample_module import SampleClass

# ❌ Bad：文件名 DeviceManager.py、DeviceManager 包
from DeviceManager import DeviceManager   # 模块与类同名，读者困惑
```

### 4.3 类 / 异常：`CapWords`

🔴 **规则**：类（Class）名使用意义完整的英文描述，采用**大写字母开头的单词（CapWords）**风格命名（HW 13.3.3.2）。异常类是类，同样用 `CapWords`。

```python
# ✅ Good
class SampleClass:
    pass


class DeviceConnectionError(ConnectionError):
    """设备连接失败。"""


# ❌ Bad
class sample_class:      # 应为 SampleClass
    pass


class device_connection_error(Exception):   # 异常类应为 DeviceConnectionError
    pass
```

📌 **易错点**：`CapWords` 里不要混入下划线（`Device_Manager` 是错误写法），也不要全大写（`DEVICEMANAGER` 是常量的风格）。

### 4.4 函数 / 方法 / 参数：`lower_with_under`

🔴 **规则**：函数（Function）、方法（Method）、函数参数（Function Parameters）名使用意义完整的英文描述，采用**小写加下划线**（`lower_with_under`）风格命名（HW 13.3.3.3）。
🔴 **规则**：模块内部使用的函数用**单下划线（`_`）开头**，表示函数是 protected 的（使用 `from module1 import *` 时不会包含）。

```python
# ✅ Good
def sample_public_function(sample_parameter):
    pass


def _sample_internal_function(sample_parameter):
    """模块内部函数，仅本模块使用。"""
    pass


class SampleClass:
    def sample_member_method(self, sample_parameter):
        pass

    def _sample_private_method(self):
        """类内部方法，外部不应调用。"""
        pass
```

📌 **易错点**：
- 函数名用驼峰（`def getDeviceInfo()`）——违反规范，应为 `get_device_info()`。
- 参数名用单字母（`def f(x)` 在数学函数里可接受，业务函数里应写 `def f(timeout)`）。
- 私有函数用了双下划线 `__func`——函数级别的名字修饰会带来 `_Class__func` 的访问麻烦，通常**单下划线足够**。

### 4.5 变量与常量：一个"大小写"的区分

🔴 **规则**：变量（variable）采用**小写加下划线**（`lower_with_under`）风格命名；常量（constant）采用**大写加下划线**（`CAPS_WITH_UNDER`）风格命名（HW 13.3.3.4）。

```python
# ✅ Good
sample_global_variable = 0          # 全局变量：小写+下划线
M_SAMPLE_GLOBAL_CONSTANT = 0        # 全局常量：大写+下划线


class SampleClass:
    SAMPLE_CLASS_CONSTANT = 0       # 类常量：大写+下划线

    def sample_member_method(self, sample_parameter):
        sample_function_variable = 0        # 局部变量
        sample_instant_variable = SampleClass()
```

```python
# ❌ Bad：常量写成小写、变量写成大写，类型与风格对不上
max_retry = 3                # 若它是"常量"，应为 MAX_RETRY
TIMEOUT = 30                 # 若它是"可变的全局变量"，应为 timeout
```

📌 **易错点**：**"常量"在 Python 中只是约定**（没有 `const` 关键字）。判断标准是"**约定俗成不修改**"：配置项上限、协议名、状态码枚举值 → 常量；运行期会被赋值的全局状态 → 变量。团队应对"什么算常量"达成一致，否则会出现 `MAX_RETRY` 与 `max_retry` 并存。

### 4.6 私有成员：单下划线 vs 双下划线

🔴 **规则**：类或对象的私有成员一般用**单下划线 `_` 开头**；对于**需要被继承的基类成员**，如果想要防止与派生类成员重名，可用**双下划线 `__` 开头**（HW 13.3.3.5）。

💡 **理由**：Python 没有严格的私有权限控制。业界约定俗成用单下划线表示"仅供内部使用"；双下划线开头的成员会被解释器**自动改名**（在名字前加上类名，如 `_Mapping__update`），用于**防止继承场景中的名字冲突**，**并不具备权限控制作用**——外部仍然可以用 `_类名__成员` 访问。因此双下划线只在"确实需要避免名字冲突"时使用（例如设计为被继承的工具基类）。

```python
# ✅ Good：单下划线表示内部使用
class MyClass:
    def my_func(self):
        self._member = 1   # 单下划线：暗示仅供类内部使用

    def _my_private_func(self):
        """单下划线：暗示仅供内部调用。"""
        pass


# ✅ Good：双下划线用于"基类成员防重名"
class Mapping:
    def __init__(self, iterable):
        self.items_list = []
        self.__update(iterable)   # 实际调用 _Mapping__update

    def update(self, iterable):
        for item in iterable:
            self.items_list.append(item)

    __update = update   # 作为 update 的私有副本，不会与派生类成员重名


class MappingSubclass(Mapping):
    def update(self, keys, values):
        """覆盖基类同名方法，参数个数不同，但不影响基类 __init__。"""
        for item in zip(keys, values):
            self.items_list.append(item)

    __update = update   # 实际为 _MappingSubclass__update，不与基类成员重名
```

```python
# 🔍 双下划线"不是私有"的验证
class Account:
    def __init__(self):
        self.__balance = 100


account = Account()
# print(account.__balance)        # AttributeError
print(account._Account__balance)  # 100 —— 改名后仍可访问，证明它不是权限控制
```

📌 **易错点**：
- 把双下划线当"私有"来防外部访问——防不住，只是改名。
- 在**函数**上滥用双下划线（`def __helper(self)`）——会导致子类无法方便地覆盖/调用，通常单下划线更合适。
- 单下划线开头的名字在 `from module import *` 时不会被导出，但在 `from module import _name` 时仍然可以显式导入。

### 4.7 命名要有明确含义

🟡 **建议**：变量命名要有明确含义，使用完整的单词或大家可以理解的缩写，**避免在循环控制变量中使用单个无意义字符**，比如 `i, j, k`（AS 建议 3.1）。

```python
# ❌ Bad：i/j/k 无意义，读者要反推含义
for i in devices:
    for j in i.ports:
        j.enable()

# ✅ Good：名字即文档
for device in devices:
    for port in device.ports:
        port.enable()

# ✅ 可接受：数学/矩阵场景中 i/j/k 是领域惯例
for i in range(rows):
    for j in range(cols):
        matrix[i][j] = 0
```

📌 **易错点**：`data`、`info`、`temp`、`obj` 这类"万能名"是另一种"无意义"——它们不说明内容；应改为 `device_config`、`order_items`、`raw_response` 等。

### 4.8 命名一致性：同类概念用同类名字

🟡 **建议**：同一系统中的相同概念，保持相同的名字（YANG 规范中也有"命名风格统一"的同类要求，思维相通）。

| 场景 | 一致（推荐） | 不一致（应避免） |
| :--- | :--- | :--- |
| 获取单条 vs 多条 | `get_device()` / `get_devices()` | `get_device()` / `fetch_device_list()` |
| 布尔 | `is_enabled`、`has_permission`、`can_retry` | `enabled_flag`、`permission`、`retry` |
| 集合 | `devices`、`ports`、`items`（复数） | `device_list`、`ports_array` |
| 回调 | `on_success`、`on_error` | `success_callback`、`errorHandler` |

📌 **易错点**：布尔变量不要用否定式（`not_valid = True` 会让 `if not not_valid` 这种表达式出现）；应写 `is_valid`。

### 4.9 第 4 章速记

```text
模块/包：lower_with_under       类/异常：CapWords
函数/方法/参数/变量：lower_with_under
常量：CAPS_WITH_UNDER（全局 M_ 前缀是常见团队约定，非强制）
内部：_lower_with_under；私有成员单下划线；仅"继承防重名"才用双下划线
双下划线 = 名字修饰（_类名__成员），不是权限控制
避免 i/j/k 与 data/info/temp 等无意义名；同类概念同名
```

---

## 第 5 章 编码实践

### 5.1 避免在无关变量/概念之间重用名字

🔴 **规则**：避免在无关的变量或无关的概念之间重用名字，避免因重名而导致的意外赋值和错误引用（HW 13.3.4.1）。

💡 **理由**：Python 的函数/类定义和 C 语言不同，**函数/类定义语句实际上是给一个名字赋值**。因此重复定义一个函数/类的名字**不会报错**，后定义的会覆盖前面的。这很容易掩盖编码问题，让同一个名字在不同执行阶段具有不同含义。名字解析遵循 **LEGB** 顺序（Local → Enclosed → Global → Builtin），内层定义会覆盖外层同名变量；代码修改时同名变量极易导致错误引用。

```python
# ❌ Bad：同名函数被覆盖，前面的定义"悄悄消失"
def process(data):
    return data.strip()

def process(data):          # 覆盖了上面的 process，且没有任何警告
    return data.upper()

# ❌ Bad：循环变量与业务变量重名
user = load_user()
for user in users:          # 覆盖了上面的 user，循环结束后 user 是最后一个元素
    ...
print(user.name)            # 期望是"当前登录用户"，实际是"最后一个用户"
```

```python
# ✅ Good：名字各自表达含义
def normalize_text(data):
    return data.strip()


def process_user(user):
    ...


current_user = load_user()
for item in users:
    process_user(item)
```

```python
# ❌ Bad：遮蔽内置名，后续代码行为改变
list = [1, 2, 3]        # 覆盖内置 list
type = "device"         # 覆盖内置 type
id = 1001               # 覆盖内置 id
```

📌 **易错点**：
- 覆盖内置名（`list`、`dict`、`type`、`id`、`input`、`filter`）最隐蔽——当前函数可能不报错，但调到别处的 `list(...)` 才炸。
- 闭包里对同名变量的赋值会产生 `UnboundLocalError`，或让外层变量意外改变。

### 5.2 变量的类型在生命周期内保持稳定

🔴 **规则**：避免变量在其生命周期内的对象类型发生变化（AS 规则 4.5）。

💡 **理由**：Python 是动态类型语言，允许变量被赋值为不同类型的对象，但这么做可能导致运行时错误；而且变量的上下文语义会变化，增加复杂度，难以调试和维护，**也不会带来任何性能提升**。

```python
# ❌ Bad：同一个变量先当 list 用、再当 str 用、又当 dict 用
result = []
result = "done"          # 类型变化
result = {"code": 0}     # 又变
```

```python
# ✅ Good：用不同名字承载不同语义
items = []
status_text = "done"
response = {"code": 0}
```

### 5.3 与 None 比较用 `is` / `is not`

🔴 **规则**：与 None 作比较要使用 `is` 或 `is not`，不要使用等号 `==`（AS 规则 4.1、PEP 8）。

💡 **理由**：`is` 判断是否指向**同一个对象**（比较 `id`）；`==` 会调用 `__eq__` 判断**值是否等价**。一个自定义了 `__eq__` 且总返回 `True` 的对象，会让 `== None` 得到 `True`，而 `is None` 依然是 `False`。None 是单例，用 `is` 既正确又更快。

```python
# ✅ Good
if value is None:
    value = default

if value is not None:
    use(value)
```

```python
# ❌ Bad
if value == None:
    ...

# ❌ Bad：判断一个列表是否为空，不要用 is []（is 比较的是同一个对象，永远为 False）
def sample_sort_list(sample_inst):
    if sample_inst is []:      # 永远不成立
        return
    sample_inst.sort()

fake_list = (2, 3, 1, 4)
sample_sort_list(fake_list)    # AttributeError: 'tuple' object has no attribute 'sort'
```

```python
# ✅ Good：空容器判断用真值，类型检查用 isinstance
def sample_sort_list(sample_inst):
    if not isinstance(sample_inst, list):
        raise TypeError(
            "sample_sort_list para type error: %r" % type(sample_inst)
        )
    if not sample_inst:        # 空列表
        return
    sample_inst.sort()
```

📌 **易错点**：`if not value:` 与 `if value is None:` **语义不同**！前者对 `0`、`""`、`[]`、`{}`、`False` 都成立。当"0/空串"是合法值时，必须用 `is None`。

### 5.4 类型检查用 `isinstance`，不用 `type`

🟡 **建议**：传递实例类型参数后，函数内应使用 `isinstance` 函数进行参数检查，**不要使用 `type`**（AS 建议 4.2、规则 4.1 的延伸）。

💡 **理由**：`isinstance(obj, Base)` 会考虑**继承关系**（子类实例也是父类实例），而 `type(obj) == Base` 只认"精确类型"，会拒绝子类；`isinstance` 还支持元组形式一次检查多个类型。使用 `type(x) is X` 的写法几乎总是错的。

```python
# ❌ Bad
if type(obj) == list:
    ...

if type(value) == int:      # bool 是 int 的子类，这条判断会漏掉 True/False
    ...

# ✅ Good
if isinstance(obj, list):
    ...

if isinstance(value, (int, float)) and not isinstance(value, bool):
    ...

# ✅ 仅当确实需要"精确类型"时才用 type（极少见，需注释说明）
if type(obj) is Device:
    ...
```

📌 **易错点**：
- `isinstance(True, int)` 是 `True`（`bool` 是 `int` 的子类），做数值校验时要显式排除 `bool`。
- `isinstance(x, str)` 对 `bytes` 为 `False`；在 Python 3 中 `str` 与 `bytes` 严格区分，跨版本代码尤其要注意。

### 5.5 用推导式代替重复逻辑，但可读性优先

🟡 **建议**：使用推导式（comprehension）代替重复的逻辑操作构造序列；但推导式必须考虑可读性，**不在一个推导式中使用 3 个（含）以上的 `for` 语句**（AS 建议 4.3 的说明为 "3 个以上"，工程上按"达到 3 层就不再用推导式"执行）。

```python
# ❌ Bad：本可以一行推导
odd_num_list = []
for i in range(100):
    if i % 2 == 1:
        odd_num_list.append(i)

# ✅ Good
odd_num_list = [i for i in range(100) if i % 2 == 1]
```

```python
# ✅ Good：两层推导仍可读（笛卡尔积）
pairs = [(i, j) for i in range(3) for j in range(3)]

# ❌ Bad：三层以上 for 的推导式，可读性极差
result = [
    (i, j, k)
    for i in range(10) for j in range(10) for k in range(10)
    if i + j + k > 15
]
```

📌 **易错点**：
- 推导式里的 `for` 顺序与嵌套 `for` **顺序一致**（从左到右、从外到内）。
- 推导式只适合"构造序列"；如果有副作用（打印、写库、调用远程接口），**不要**塞进推导式——这违背推导式的语义。
- 生成器表达式 `(x for x in ...)` 适合"只消费一次的大数据"，但注意它只能遍历一次。

### 5.6 不需要访问实例的方法：用 `@staticmethod` / `@classmethod`

🔴 **规则**：类的方法不需访问实例时，根据具体场景选择使用 `@staticmethod` 或 `@classmethod` 进行装饰（HW 13.3.4.2）。

💡 **理由**：一般的类方法要接收 `self` 参数表示实例，但有些方法不需要访问实例、不需要访问任何成员，或只需要访问**类自身**的成员。Python 3 允许直接定义不含 `self` 参数的方法并且允许不通过实例调用，但**一旦通过实例调用就会因参数不匹配而报错**。加 `@staticmethod` 可以让解释器明确"此方法不需要 self"，提前拦截问题，可读性也更好。

```python
# ❌ Bad：缺 self 的普通方法，通过实例调用时报错
class Util:
    def parse(value):          # 无 self
        return value.strip()


Util.parse(" a ")       # OK：通过类调用
Util().parse(" a ")     # TypeError: Util.parse() takes 1 positional argument but 2 were given（3.10+ 显示限定名）
```

```python
# ✅ Good：静态方法（不需要访问实例/类状态）
class PathUtil:
    @staticmethod
    def normalize(path):
        return path.replace("\\", "/").strip("/")


# ✅ Good：类方法（需要访问/构造类本身，如工厂方法）
class Device:
    def __init__(self, model, ip):
        self.model = model
        self.ip = ip

    @classmethod
    def from_config(cls, config):
        """工厂方法：从配置字典构造实例，子类调用时返回子类实例。"""
        return cls(config["model"], config["ip"])
```

| 装饰器 | 第一个参数 | 何时使用 |
| :--- | :--- | :--- |
| 普通方法 | `self` | 需要读写实例状态 |
| `@classmethod` | `cls` | 需要访问类属性、需要工厂方法、需要"知道自己是哪个类" |
| `@staticmethod` | 无 | 纯工具函数，放在类里只是"命名空间归属" |

📌 **易错点**：`@classmethod` 里误用 `self` 命名第一个参数（语义上应是 `cls`）；`@staticmethod` 里又想访问实例属性（做不到，因为它没有 `self`）。

### 5.7 类型注解：让接口自解释

🟡 **建议**：函数接口定义入参，可明确限定类型，防止误用（AS 建议 4.6）。

💡 **理由**：Python 没有运行期的类型信息，IDE 无法给出足够提示。接口加上类型注解后，**调用方传参错误更容易被发现**，IDE/`mypy` 可以做静态检查，docstring 也能少写一半。

```python
# ✅ Good：Python 3.9+ 可用内置泛型；3.8 用 typing.List/Dict 或 from __future__ import annotations
from typing import Optional


def find_device(serial: str, timeout: int = 30) -> Optional[dict]:
    """按序列号查询设备；不存在时返回 None。"""
    ...


def retry(times: int, delay: float = 1.0) -> "Callable[[Callable], Callable]":
    ...
```

```python
# ✅ Good：复杂结构用 TypedDict / dataclass 表达，而不是裸 dict
from dataclasses import dataclass


@dataclass(frozen=True)
class DeviceConfig:
    serial: str
    ip: str
    port: int = 830
```

📌 **易错点**：
- 注解**不是运行时强制**，`f("abc")` 传给 `def f(x: int)` 不会报错；要真正校验，需在函数入口用 `isinstance` 检查或引入 pydantic 等校验库。
- 循环引用类型用字符串（`"Device"`）或 `from __future__ import annotations` 延迟求值。

### 5.8 资源管理：用 `with` 或 `finally`

🔴 **规则**：使用 `try...except...` 结构对代码作保护时，需要在异常后使用 `finally` 结构保证操作对象的释放，或者使用 `with` 方式的语法结构（HW 13.3.5.1、AS 规则 5.1）。

```python
# ❌ Bad：异常时文件句柄泄漏
f = open("data.txt")
data = f.read()          # 若这里抛异常，f 永远不会被 close
f.close()

# ✅ Good：with 自动释放（首选）
with open("data.txt", encoding="utf-8") as f:
    data = f.read()

# ✅ Good：必须手动管理时，用 finally 兜底
f = open("data.txt", encoding="utf-8")
try:
    data = f.read()
finally:
    f.close()
```

📌 **易错点**：`with` 可以一次管理多个资源：`with open(a) as fa, open(b) as fb:`；注意顺序与释放顺序相反。自定义资源要实现上下文管理器协议（`__enter__`/`__exit__`）或使用 `contextlib.contextmanager`。

### 5.9 其它常用编码约定

| 场景 | ❌ Bad | ✅ Good | 说明 |
| :--- | :--- | :--- | :--- |
| 遍历下标 | `for i in range(len(items)): print(items[i])` | `for item in items:` 或 `for i, item in enumerate(items):` | 直接迭代 + `enumerate` 更 Pythonic |
| 并行遍历 | `for i in range(len(a)): print(a[i], b[i])` | `for x, y in zip(a, b):` | 注意 `zip` 以最短为准；需要严格等长用 `zip(a, b, strict=True)`（3.10+） |
| 字符串格式化 | `"%s=%s" % (k, v)` / `"{}={}".format(k, v)` | `f"{k}={v}"` | f-string 可读性最好；日志场景用 `logger.info("%s=%s", k, v)` 延迟格式化 |
| 判空 | `if len(items) > 0:` | `if items:` | 容器真值判断 |
| 交换变量 | `t = a; a = b; b = t` | `a, b = b, a` | 元组解包 |
| 单例 None 判断 | `if x == None:` | `if x is None:` | 见 5.3 |
| 文件编码 | `open(path)` | `open(path, encoding="utf-8")` | 显式编码，避免平台默认编码差异 |
| 路径拼接 | `path + "/" + name` | `os.path.join(path, name)` 或 `pathlib.Path(path) / name` | 跨平台 |

### 5.10 第 5 章速记

```text
不重用名字、不遮蔽内置名；函数/类定义即赋值，后定义覆盖前定义（LEGB）
变量生命周期内类型稳定
None 比较用 is / is not；类型检查用 isinstance
推导式代替重复逻辑，但达到 3 层 for 就改用普通循环，且有副作用时不用
不需访问实例 → @staticmethod；需访问类 → @classmethod
接口加类型注解（不是运行期强制）
资源释放用 with / finally
for + enumerate / zip 代替 range(len(...))
```

---

## 第 6 章 异常处理

> 异常处理的目标是"**让错误在正确的地方被正确处理**"：不吞、不漏、不泄密、不破坏对象状态。华为规范在这一章的要求比一般团队更严格，且和安全编码强相关。

### 6.1 不要使用裸 `except:` 捕获所有异常

🔴 **规则**：不要使用 `except:` 语句来捕获所有异常（HW 13.3.5.2）。

💡 **理由**：Python 在异常方面非常"宽容"，`except:` 会捕获**包括 `KeyboardInterrupt`、`SystemExit`、`GeneratorExit` 在内的任何东西**，甚至会把"语法错误级别的编程错误"一起吞掉。使用裸 `except:` 很容易隐藏真正的 bug。**应当明确期望处理的异常**。`Exception` 类是大多数运行时异常的基类，**一般也应避免在 `except` 语句中直接使用**。通常：

- `try` 只应当包含**必须要在当前位置处理异常**的语句；
- `except` 只捕获**必须处理**的异常；
- 对于其它预料之外的异常，让上层函数捕获，或者透传到程序外部，**充分暴露问题**。

```python
# ❌ Bad：裸 except，连 Ctrl+C 和 SystemExit 都吞掉，bug 被隐藏
try:
    config = load_config(path)
except:
    config = {}

# ❌ Bad：直接用 Exception，范围仍然过宽
try:
    config = load_config(path)
except Exception:
    config = {}
```

```python
# ✅ Good：只捕获"确实知道如何处理"的异常
try:
    with open(path, encoding="utf-8") as f:
        config = json.load(f)
except FileNotFoundError:
    logger.warning("config not found: %s, use default", path)
    config = DEFAULT_CONFIG
except json.JSONDecodeError as exc:
    logger.error("invalid config json: %s", path)
    raise ConfigError(f"invalid config: {path}") from exc
# 其它异常（PermissionError、MemoryError……）继续向上抛
```

📌 **易错点**：
- `except:` 之后又 `pass`——这是最严重的"吞异常"写法。
- `except Exception` 用于"兜底 + 记录 + 重抛"是可接受的（如 Web 框架中间件），但**必须重抛或返回明确的错误响应**，不能静默继续。
- 捕获范围要按"异常类型"精确收窄，而不是按"代码块多大"来收窄。

### 6.2 try 的范围要最小

🔴 **规则**（承 6.1 的延伸）：`try` 只应包含必须要在当前位置处理异常的语句；`except` 只捕获必须处理的异常。

```python
# ❌ Bad：try 包住了整个业务，任何异常都被"处理"成 return None
def get_user_age(user_id):
    try:
        user = db.query_user(user_id)
        age = calculate_age(user.birthday)
        send_metrics(user_id, age)
        return age
    except Exception:
        return None

# ✅ Good：只包"预期可能失败"的那一步
def get_user_age(user_id):
    user = db.query_user(user_id)        # 查不到由上层处理
    try:
        birthday = user.birthday
    except AttributeError:
        logger.warning("user %s has no birthday", user_id)
        return None
    age = calculate_age(birthday)
    send_metrics(user_id, age)           # 埋点失败不应影响主流程？由上层决定
    return age
```

### 6.3 `raise` 必须带异常对象；裸 `raise` 只用于重抛

🔴 **规则**：不在 `except` 分支里面的 `raise` 都必须带异常（HW 13.3.5.3）。

💡 **理由**：`raise` 关键字单独使用（裸 `raise`）只能出现在 `try...except` 语句中，用于**重新抛出 `except` 抓住的异常**。在 `except` 之外写裸 `raise` 会抛 `RuntimeError: No active exception to reraise`（注意 Python 实际消息是 `reraise`，无连字符）。

```python
# ❌ Bad：except 外裸 raise
def validate(value):
    if value < 0:
        raise            # RuntimeError: No active exception to reraise
```

```python
# ✅ Good：带异常对象与明确消息
def validate(value):
    if value < 0:
        raise ValueError(f"value must be non-negative, got {value}")
```

```python
# ✅ Good：裸 raise 用于"记录后原样重抛"
try:
    process()
except TimeoutError:
    logger.warning("timeout, will retry")
    raise                # 保留原始堆栈，交给上层
```

```python
# ✅ Good：转换异常时用 raise ... from 保留因果链
try:
    payload = json.loads(raw)
except json.JSONDecodeError as exc:
    raise ProtocolError("device returned invalid json") from exc
```

📌 **易错点**：
- `raise ValueError`（类而不是实例）虽然合法，但丢失了消息；应 `raise ValueError("...")`。
- 用 `raise ... from None` 会**隐藏**原始异常，除非确实不想暴露底层细节（安全场景可能有意为之），否则不要用。
- 自定义异常要继承 `Exception`（或更具体的基类），不要继承 `BaseException`，否则 `except Exception` 之类的常规捕获会漏掉它。

### 6.4 不要抑制或忽略已检查异常

🔴 **规则**：不要抑制或者忽略已检查异常（SEC 2.1）。

💡 **理由**：编码人员常常用空 `except` 或无意义的 `except` 块来抑制异常。每个 `except` 块都应确保程序只会在继续有效的情况下才继续运行。因此 `except` 块必须**要么从异常中恢复，要么重新抛出适合当前上下文的另一个异常**，让最近的外层 `try` 去恢复。异常会打断应用原本预期的控制流——`try` 块中异常点之后的语句都不会执行。

```python
# ❌ Bad：打印堆栈后继续跑，等价于抑制异常
try:
    result = risky_operation()
except Exception:
    import traceback
    traceback.print_exc()      # 打印了，但程序继续，如同没发生异常
```

```python
# ✅ Good 1：从异常中恢复（要求用户重试直到成功）
valid_flag = False
while not valid_flag:
    try:
        path = input("file path: ")
        with open(path, encoding="utf-8") as f:
            data = f.read()
        valid_flag = True
    except FileNotFoundError:
        logger.warning("file not found, please retry: %s", path)
```

```python
# ✅ Good 2：过滤敏感信息后转为对外的异常（安全场景）
class SafeExceptionReporter(Exception):
    """对外暴露的异常：内部细节记录到日志，对外只返回脱敏消息。"""

    def __init__(self, internal_msg: str, public_msg: str):
        super().__init__(public_msg)
        self.internal_msg = internal_msg

    def log(self):
        logger.error("internal error: %s", self.internal_msg)
```

**例外情况（可以抑制异常的场景）**：

1. **资源释放失败不影响后续行为**时（关闭文件、socket、线程等），可以净化并记录日志后忽略——但要考虑资源耗尽问题。
2. **在当前抽象层次无法从异常中恢复**时，不应在本层处理，而应抛出合适的异常让更高层处理（通常就是"不写 catch，让它冒泡"）。

### 6.5 异常中不得泄露敏感信息

🔴 **规则**：禁止在异常中泄露敏感信息（SEC 2.2）。

💡 **理由**：敏感数据范围应基于应用场景与威胁分析确定，典型包括**口令、银行账号、个人信息、通讯记录、密钥**等。异常传递时若未过滤敏感信息，常导致信息泄露，帮助攻击者发起进一步攻击。**不只是异常文本消息，异常本身的类型也可能泄露敏感信息**——例如 `IOError` 会透露文件系统结构，异常的"有/无"可以推断"文件是否存在"。

```python
# ❌ Bad：直接把用户输入拼进异常，且暴露文件是否存在
def open_user_file(path):
    try:
        return open(path).read()
    except OSError as exc:
        raise RuntimeError(f"open {path} failed: {exc}")


# ✅ Good：先标准化 + 限制目录，对外只给统一错误
import os

SAFE_BASE_DIR = os.path.realpath("/home/python")


def open_user_file(path):
    try:
        real_path = os.path.realpath(path)
    except OSError:
        raise ValueError("invalid file") from None
    # 必须用 commonpath（或带 os.sep 的前缀）判断，防止 /home/python-evil 绕过
    if os.path.commonpath([SAFE_BASE_DIR, real_path]) != SAFE_BASE_DIR:
        raise ValueError("invalid file")
    try:
        with open(real_path, encoding="utf-8") as f:
            return f.read()
    except OSError:
        # 不区分"不存在/无权限/是目录"，避免信息泄露
        raise ValueError("invalid file") from None
```

📌 **易错点**：堆栈信息（traceback）在 Web 场景直接返回给前端是大事故；日志里也可能带出敏感字段（见第 8 章）。

### 6.6 方法发生异常时要恢复到之前的对象状态

🔴 **规则**：方法发生异常时要恢复到之前的对象状态（SEC 2.3）。

💡 **理由**：发生异常时，对象一般需要（关键安全对象则**必须**）维持状态一致性。常用手段：

1. **输入校验**（如校验方法调用参数）；
2. **调整逻辑顺序**，使可能发生异常的代码在对象被修改之前执行；
3. 业务操作失败时**回滚**；
4. 对**临时副本**操作，成功后才提交到原始对象；
5. **避免修改对象状态**。

```python
# ❌ Bad：先改状态、再校验；异常时状态没恢复，后续调用结果错误
PADDING = 2


class Dimensions:
    def __init__(self, length, width, height):
        self.length = length
        self.width = width
        self.height = height

    def get_volume_package(self, weight):
        self.length += PADDING
        self.width += PADDING
        self.height += PADDING
        try:
            self.validate(weight)
            return self.length * self.width * self.height
        except ValueError:
            return -1          # BUG：异常时没有回滚 padding

    def validate(self, weight):
        if weight > 20:
            raise ValueError("weight too large")
```

```python
# ✅ Good 1：finally 中回滚（无论是否异常都恢复）
PADDING = 2


class Dimensions:
    def __init__(self, length, width, height):
        self.length = length
        self.width = width
        self.height = height

    def get_volume_package(self, weight):
        self.length += PADDING
        self.width += PADDING
        self.height += PADDING
        try:
            self.validate(weight)
            return self.length * self.width * self.height
        finally:
            self.length -= PADDING
            self.width -= PADDING
            self.height -= PADDING

    def validate(self, weight):
        if weight > 20:
            raise ValueError("weight too large")
```

```python
# ✅ Good 2（最推荐）：先校验、再修改；完全不修改对象，用局部值计算
PADDING = 2


class Dimensions:
    def __init__(self, length, width, height):
        self.length = length
        self.width = width
        self.height = height

    def get_volume_package(self, weight):
        self.validate(weight)
        return (
            (self.length + PADDING)
            * (self.width + PADDING)
            * (self.height + PADDING)
        )

    def validate(self, weight):
        if weight > 20:
            raise ValueError("weight too large")
```

📌 **易错点**：在 `except` 里 `return -1` 而不是抛异常/回滚，会把"状态已污染"的问题延迟到下一次调用才暴露，极难排查。

### 6.7 异常设计的最佳实践

| 实践 | 说明 |
| :--- | :--- |
| 自定义异常继承 `Exception` | 不要继承 `BaseException`；业务异常可继承一个项目基类 `AppError` |
| 异常层次要有意义 | `DeviceTimeoutError(AppError)`、`DeviceAuthError(AppError)`，便于上层按类捕获 |
| 异常消息带上下文、但不带敏感数据 | `f"device {serial} connect failed"`，不要带 `password` |
| 底层异常要"转译" | 把 `requests.Timeout` 转成 `DeviceTimeoutError`，避免底层库泄漏到业务层 |
| 用 `raise ... from exc` | 保留因果链，排查时能看到原始异常 |
| 不要用异常做正常控制流 | 可预判的分支用 `if`；异常用于"异常"情况 |
| EAFP 优于 LBYL | Python 风格：先做再捕获（`try: d[k]`），而不是先检查再访问（`if k in d`），后者有竞态 |

### 6.8 第 6 章速记

```text
禁止裸 except:；也避免直接用 Exception
try 范围最小化；只捕获必须处理的异常，其它透传
except 外 raise 必带异常对象；裸 raise 只用于重抛
不要抑制异常（空 except / 只打印堆栈后继续）
异常不得泄露敏感信息（消息 + 类型 + 是否存在）
异常时恢复对象状态：先校验、finally 回滚、副本提交、不改状态
raise ... from exc 保留因果链；自定义异常继承 Exception
```

---

## 第 7 章 并发与并行

### 7.1 多线程适合阻塞式 I/O，不适合并行计算

🔴 **规则**：**多线程适用于阻塞式 I/O 场景，不适用于并行计算场景**（HW 13.3.6.1）。

💡 **理由**：CPython 有 **GIL（全局解释器锁）**，同一时刻只有一个线程能执行 Python 字节码。因此：

- **阻塞式 I/O**（网络请求、文件读写、数据库查询、等待子进程）期间线程会**释放 GIL**，多线程可以显著提高吞吐 —— 这是线程的用武之地；
- **CPU 密集计算**（加解密、图像处理、大量循环）无法靠多线程并行，线程切换反而增加开销 —— 应使用**多进程**（`multiprocessing` / `ProcessPoolExecutor`）。

```python
# ✅ Good：I/O 密集 —— 线程池并发发起网络请求
from concurrent.futures import ThreadPoolExecutor


def fetch_all(device_ips):
    with ThreadPoolExecutor(max_workers=8) as pool:
        return list(pool.map(fetch_status, device_ips))


def fetch_status(ip):
    """阻塞式 I/O：建议设置超时，避免线程被永久占用。"""
    return requests.get(f"http://{ip}/status", timeout=3).json()
```

```python
# ✅ Good：CPU 密集 —— 进程池并行计算
from concurrent.futures import ProcessPoolExecutor


def hash_all(passwords):
    with ProcessPoolExecutor() as pool:
        return list(pool.map(expensive_hash, passwords))
```

```python
# ❌ Bad：用多线程跑 CPU 密集计算，受 GIL 限制，几乎无加速
from threading import Thread

threads = [Thread(target=expensive_hash, args=(p,)) for p in passwords]
for t in threads:
    t.start()
for t in threads:
    t.join()
```

### 7.2 并发模型怎么选

| 场景 | 推荐方案 | 说明 |
| :--- | :--- | :--- |
| 少量并发、阻塞 I/O | `ThreadPoolExecutor` | 简单直接，注意限制 `max_workers` 和超时 |
| 大量并发、阻塞 I/O | `asyncio` + `aiohttp` 等 | 单线程事件循环，避免线程切换开销；**不要在协程里做阻塞调用** |
| CPU 密集 | `ProcessPoolExecutor` / `multiprocessing` | 绕过 GIL；注意进程间数据序列化成本 |
| 混合场景 | 分层：I/O 用异步/线程，CPU 段交给进程池 | 避免"线程池里再起线程池"导致资源爆炸 |

```python
# ✅ Good：asyncio 并发（注意：await 的必须是异步 API）
import asyncio
import aiohttp


async def fetch(session, url):
    async with session.get(url, timeout=3) as resp:
        return await resp.text()


async def main(urls):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in urls]
        return await asyncio.gather(*tasks, return_exceptions=True)
```

📌 **易错点**：
- 在 `async def` 里调用 `requests.get()`（阻塞）会让整个事件循环卡住；应使用异步库或 `asyncio.to_thread()`。
- 线程池 `max_workers` 无上限会造成连接数爆炸、内存上涨；要结合下游承载能力设置。
- 多线程共享可变状态必须加锁；但**锁的粒度要小**，避免死锁（固定加锁顺序、用超时 `lock.acquire(timeout=...)`）。

### 7.3 第 7 章速记

```text
线程 = 阻塞 I/O；进程 = CPU 密集；asyncio = 高并发 I/O
GIL 决定线程不能加速 CPU 计算
协程里禁止阻塞调用
线程池设 max_workers 与超时；共享状态加锁且锁粒度小
```

---

## 第 8 章 安全编码红线

> 本章对应华为/昇腾《python 语言安全编程指导》。风格问题影响"好不好维护"，安全问题影响"会不会出事"。**安全条款是规则（must），没有"风格偏好"空间**。本章每条都给出「风险 → 反例 → 正例」，并在章末给出危险函数清单与审计清单。

### 8.1 总原则：信任边界与数据校验

🔴 **规则 8.1.1**：**校验跨信任边界传递的不可信数据**（SEC 1.1）。

💡 **理由**：程序可能接收来自用户、网络连接或其他来源的不可信数据，并把数据跨信任边界传到目标系统（浏览器、数据库、文件系统、操作系统）。目标系统无法区分"畸形的不可信数据"，未校验的数据可能引发注入攻击。**校验必须在信任边界以内进行**（例如 Web 应用必须在服务端校验，不能只靠前端 JS）。数据校验分**输入校验**（外部传入）和**输出校验**（传出到外部）。

**四种校验策略**（任何时候尽可能使用"接受已知合法数据"的白名单策略）：

| 策略 | 说明 | 强度 |
| :--- | :--- | :--- |
| ✅ **白名单校验**（接受已知好的数据） | 检查数据是否属于严格约束的、已知可接受的合法集合 | 最强，首选 |
| ⚠️ 黑名单校验（拒绝已知坏的数据） | 维护"已知非法字符/模式"列表；攻击手法演进后列表会过时 | 弱，需持续维护 |
| ✅ 白名单净化 | 删除/编码/替换所有不属于合法字符集的字符，再使用 | 强，适合无法穷举合法值的场景 |
| ⚠️ 黑名单净化 | 剔除或转换某些危险字符（如删除引号、转 HTML 实体） | 弱 |

```python
# ✅ Good：白名单校验（只允许字母、数字、下划线）
import re

NAME_PATTERN = re.compile(r"^[0-9A-Za-z_]+$")


def validate_name(name: str) -> str:
    if not NAME_PATTERN.match(name):
        raise ValueError("invalid name")
    return name
```

```python
# ✅ Good：白名单净化（只保留数字，构造"干净"的电话号码）
def normalize_phone(raw: str) -> str:
    digits = "".join(ch for ch in raw if ch.isdigit())
    if not (7 <= len(digits) <= 15):
        raise ValueError("invalid phone")
    return digits
```

```python
# ⚠️ 黑名单净化：能挡一部分，但不要作为唯一防线
def remove_javascript(text: str) -> str:
    return re.sub(r"javascript", "", text, flags=re.IGNORECASE)
```

📌 **易错点**：
- 只在前端做校验（浏览器可绕过），服务端不校验。
- 用"长度/非空"校验代替"合法性"校验——长度合法不代表内容安全。
- 校验后的数据在系统内部又发生"二次拼接"（如拼进 SQL、命令行），此时需要**对应输出环节的防护**，不能指望输入校验包打天下。

### 8.2 SQL 注入：必须使用参数化查询

🔴 **规则 8.2.1**：**禁止直接使用不可信数据来拼接 SQL 语句**（SEC 1.2）。

💡 **理由**：SQL 注入是指原始 SQL 查询被动态改成与程序预期完全不同的查询，可能导致信息泄露或数据被篡改。防止方式主要有两类：**参数化查询**（优先）和**对不可信数据校验**。参数化查询还能改进数据库访问性能（数据库会缓存查询计划）。Python 的 DB-API 规范（psycopg2、sqlite3、pyodbc、zxJDBC 等）的 `execute()` 支持参数传入，从库函数层面解决注入问题。

```python
# ❌ Bad：字符串拼接 / % 格式化拼 SQL
sql = "select id, type, name from xl_bugs where id = %s and type = %s" % (bug_id, bug_type)
cur.execute(sql)

# ❌ Bad：f-string 拼 SQL（同样可注入）
cur.execute(f"select * from users where name = '{name}'")

# ✅ Good：参数化查询，参数作为 execute 的第二个参数传入
args = (bug_id, bug_type)
cur.execute(
    "select id, type, name from xl_bugs where id = %s and type = %s",
    args,
)
```

```python
# ❌ Bad：字符型字段手工拼接，输入 ' or 1=1 -- 即可绕过
def gen_sql_str(name_str):
    return "select * from stu where name ='%s'" % name_str
# 输入 "' or 1=1 --" 得到：
# select * from stu where name ='' or 1=1 --'


# ✅ Good：参数化（推荐）
def query_student(cur, name):
    cur.execute("select * from stu where name = %s", (name,))
    return cur.fetchall()
```

```python
# 如果确实无法参数化（极少数场景：表名/列名等标识符），必须白名单校验
ALLOWED_COLUMNS = {"id", "name", "type"}


def query_by_column(cur, column, value):
    if column not in ALLOWED_COLUMNS:      # 标识符不能参数化，只能白名单
        raise ValueError("invalid column")
    cur.execute(f"select {column} from xl_bugs where id = %s", (value,))
```

📌 **易错点**：
- 参数化查询**不要给参数加引号**：`cur.execute("... where name = '%s'", (name,))` 是错的；占位符本身会处理引号。
- 不同驱动的占位符不同：`sqlite3`/`psycopg2` 用 `%s` 或 `?`（按驱动），`pyodbc` 用 `?`；要按驱动文档来。
- 表名、列名、`ORDER BY` 字段**不能参数化**，只能用白名单枚举。

### 8.3 路径遍历：先标准化，再验证

🔴 **规则 8.3.1**：**验证路径之前应该先将其标准化**（SEC 1.3）。

💡 **理由**：绝对/相对路径中可能包含符号链接（软链接）、硬链接、快捷方式、影子文件、别名、连接文件，验证前必须完整解析这些链接；路径中还可能包含 `.`（当前目录）、`..`（上级目录），以及操作系统/文件系统特有的命名约定。同一文件可用多种路径名引用，攻击者会利用**目录遍历**（`../../../etc/passwd`）或**等价路径**绕过检查。`os.path.abspath()` 只做"拼成绝对路径"，**不解析符号链接**；`os.path.realpath()` 会解析所有别名、快捷方式、符号链接，并移除 `..`。

```python
# ❌ Bad：abspath 不解析符号链接，攻击者可构造软链接指向敏感文件
import os

def read_file_bad(user_path):
    path = os.path.abspath(user_path)      # 不解析符号链接
    if not path.startswith("/home/python/"):
        raise ValueError("invalid path")
    return open(path).read()
```

```python
# ✅ Good：realpath 标准化后再验证，并限制在安全目录内
import os

SAFE_BASE_DIR = os.path.realpath("/home/python/")


def read_user_file(user_path):
    try:
        real_path = os.path.realpath(user_path)     # 解析符号链接、移除 ..
    except OSError:
        raise ValueError("invalid path")
    # 必须加 os.sep，避免 /home/python-evil 绕过前缀检查
    if not real_path.startswith(SAFE_BASE_DIR + os.sep):
        raise ValueError("invalid path")
    with open(real_path, encoding="utf-8") as f:
        return f.read()
```

```python
# ✅ 更稳妥：Python 3.5+ 可用 os.path.commonpath 判断"是否在目录内"
def is_inside(base_dir, target_path):
    base = os.path.realpath(base_dir)
    target = os.path.realpath(target_path)
    return os.path.commonpath([base, target]) == base
```

📌 **易错点**：
- 前缀判断漏掉路径分隔符：`/home/python-evil/xxx` 会通过 `startswith("/home/python")` 检查。
- `os.path.abspath` 与 `os.path.realpath` 混用——**验证一律用 `realpath`**。
- 校验路径与打开文件之间存在**时间竞争窗口**（TOCTOU）：校验通过后文件可能被改；缓解方式是"在安全目录内校验"，或使用 `O_NOFOLLOW`、文件描述符方式操作。

### 8.4 代码执行：禁止 `eval` / `exec` 执行不可信代码

🔴 **规则 8.4.1**：**禁止使用 `eval` 和 `exec` 执行不可信代码**（SEC 1.4）。

💡 **理由**：`eval(expression[, globals[, locals]])` 把字符串当 Python 表达式求值；`exec` 把字符串当 Python 代码执行。两者都能执行任意 Python 代码，若引入外部输入且缺少校验，可导致用户访问代码命名空间中的变量（信息泄露），`eval` 还能用 `__import__` 导入任意模块。

```python
# ❌ Bad：用 eval 解析用户输入（猜数字游戏）
from random import randint

num = randint(1, 100)
while True:
    guess = eval(input("Your guess> "))   # 输入 num 直接"猜中"；输入 __import__('os') 可执行代码
```

```python
# ✅ Good：不需要求值时，按字符串处理
num = randint(1, 100)
while True:
    guess = input("Your guess> ")
    if guess != str(num):
        print("Nope,", guess, "is wrong.")
    else:
        print("You win!")
        break
```

```python
# ✅ 如果确实需要"安全地"解析字面量结构（如配置），用 ast.literal_eval
import ast

value = ast.literal_eval('{"timeout": 30, "retries": 3}')   # 只支持字面量
```

📌 **易错点**：
- `ast.literal_eval` 只适合"字面量"（数字、字符串、元组、列表、字典、布尔、None），**不能**执行函数调用/表达式；它比 `eval` 安全，但不是"万能解析器"。
- 用正则"黑名单"过滤 `eval` 的输入不可靠（总有绕过方式），根本方法是**不使用 `eval`/`exec`**。

### 8.5 输入函数：注意版本语境

🔴 **规则 8.5.1（Python 2 语境）**：**禁止使用 `input` 函数**，需要用户输入时使用 `raw_input`（SEC 1.5）。

💡 **理由**：在 **Python 2** 中，`input(x)` 等价于 `eval(raw_input(x))`——它会把用户输入当作 Python 表达式求值，可导致用户非法访问程序命名空间中的变量。`raw_input()` 则只读取一行并按字符串返回。

> ⚠️ **版本语境必须说清楚**：在 **Python 3** 中，`input()` 返回字符串（等价于 Py2 的 `raw_input`），本身**不会**求值，因此"禁止 input()"这条规则**不适用于 Python 3**。但 `eval(input())` 在 Python 3 中**仍然危险**，同样禁止。

```python
# ❌ Bad（Python 2 语境）：input() 会求值
secret = "42"
value = input("Answer to everything is ? ")    # 输入 dir() 可列出命名空间；输入 secret 可得 42
```

```python
# ✅ Good（Python 2 语境）
value = raw_input("Answer to everything is ? ")
```

```python
# ✅ Good（Python 3）：input() 返回 str；需要数值就显式转换并校验
raw = input("timeout (seconds): ")
try:
    timeout = int(raw)
except ValueError:
    raise ValueError("timeout must be an integer")
if not 1 <= timeout <= 300:
    raise ValueError("timeout out of range")
```

📌 **易错点**：把 Python 2 的结论照搬到 Python 3，是面试/评审中的高频误判。**判断标准**：是否对输入做了 `eval`/`exec` 等求值操作，而不是看函数名叫什么。

### 8.6 命令注入：禁止用不可信输入调用 OS 命令解析器

🔴 **规则 8.6.1**：**禁止调用 OS 命令解析器执行命令或运行程序，防止命令注入**（SEC 1.6）。

💡 **理由**：使用未经校验的不可信输入作为系统命令的参数或命令的一部分，可能导致命令注入。命令会以与 Python 应用相同的特权级别执行，相当于给攻击者一个 shell。`os.system`、`os.popen` 会调用新进程，如果命令来自外部输入，就可能产生命令/参数注入。要求：

1. 命令执行的字符串**不要去拼接输入的参数**；如果必须拼接，要对输入参数做**白名单过滤**；
2. 对传入参数做**类型校验**（如整数数据强制 `int()` 转换）；
3. 保证格式化字符串正确，例如 `int` 类型拼接用 `%d` 而不是 `%s`。

```python
# ❌ Bad：直接拼接用户输入
import os
import sys

os.system("ls " + sys.argv[1])
# 攻击者输入 ". && echo bad"，实际执行 ls . 和 echo bad 两条命令
```

```python
# ✅ Good：优先使用标准库 API，根本不需要起 shell
import os
import sys

print(os.listdir(sys.argv[1]))
```

```python
# ✅ Good：确实需要子进程时，用列表参数 + shell=False，禁止拼接
import re
import subprocess


def ping_host(host):
    # host 必须来自白名单/严格正则校验；还要拒绝以 "-" 开头的值，
    # 否则 "-f" 之类会被 ping 当作命令行选项（参数注入的残余面）
    if not re.match(r"^[0-9A-Za-z][0-9A-Za-z.\-]*$", host):
        raise ValueError("invalid host")
    result = subprocess.run(
        ["ping", "-c", "1", host],      # 列表参数，不经过 shell
        capture_output=True,
        text=True,
        timeout=5,
        check=False,
    )
    return result.returncode
```

📌 **易错点**：
- `subprocess.run(cmd, shell=True)` 等价于把字符串交给 shell，注入风险与 `os.system` 相同；只有确实需要 shell 特性（管道、通配符）时才用，且必须白名单校验。
- 用 `shlex.quote()` 处理参数是"缓解"而不是"根治"；根治是不用 shell、用列表参数。
- 整数格式化用 `%d` 而非 `%s`：`%s` 会让 `"1 or 1=1"` 这类字符串原样进入拼串。

### 8.7 临时文件：禁止 `mktemp`，用完及时删除，创建即定权限

🔴 **规则 8.7.1**：**禁止使用 `mktemp` 创建临时文件**（SEC 3.1）。

💡 **理由**：`mktemp` 返回的临时文件名包含进程 ID，**很容易被猜测**，攻击者可利用它执行**符号链接攻击**，覆盖任意文件，导致拒绝服务。Python 2.3 之后该函数已被废弃。替代函数：`NamedTemporaryFile()`、`mkstemp()`、`mkdtemp()`——新函数生成的文件名不再包含 PID，而是六个随机字符。

```python
# ❌ Bad
import tempfile

temp_file = tempfile.mktemp()
```

```python
# ✅ Good
import os
import tempfile

fd, path = tempfile.mkstemp()          # 原子创建、名字随机
try:
    with os.fdopen(fd, "w", encoding="utf-8") as f:
        f.write("data")
finally:
    os.remove(path)
```

🔴 **规则 8.7.2**：**临时文件使用完毕应及时删除**（SEC 3.2）。

💡 **理由**：全局可写目录（`/tmp`、`/var/tmp`、`C:\TEMP`）中的文件可能被本机攻击者利用；及时删除有助于回收文件名和存储资源。每个程序都有责任确保删除已用完的临时文件。

```python
# ✅ Good：delete=True 在关闭时自动删除
import tempfile

with tempfile.NamedTemporaryFile(
    mode="w", encoding="utf-8", delete=True
) as tmp:
    tmp.write("data")
    tmp.flush()
    # 在这里读取/使用 tmp.name 指向的文件内容
```

🔴 **规则 8.7.3**：**在多用户系统中创建文件时指定合适的访问许可**（SEC 3.3）。

💡 **理由**：文件创建时未做足够限制，攻击者可能在程序修改权限之前读取/修改它。内置 `open()` 无法显式指定权限；应使用 `os.open` 的 `mode` 参数或创建后立即 `os.chmod`。

```python
# ❌ Bad：权限取决于 umask，可能被同机其他用户读到
with open("secret.tmp", "w") as f:
    f.write("sensitive")

# ✅ Good：创建时就指定 0600
import os

fd = os.open("secret.tmp", os.O_WRONLY | os.O_CREAT | os.O_EXCL, 0o600)
with os.fdopen(fd, "w", encoding="utf-8") as f:
    f.write("sensitive")

# ✅ Good：或创建后立即收紧权限（无创建时指定权限的 API 时使用）
with open("secret2.tmp", "w", encoding="utf-8") as f:
    os.chmod("secret2.tmp", 0o600)
    f.write("sensitive")
```

> **取舍说明**：`chmod` 与 `open` 之间仍存在极短窗口；能使用 `os.open(..., 0o600)`（上一种写法）时应优先，`chmod` 仅用于无法在创建时指定权限的场景。

📌 **易错点**：
- 权限常量用八进制 `0o600`（Python 3 写法），不要写 `0600`（Python 3 中是语法错误）。
- **例外**：文件创建在"安全目录"中，且对非受信用户不可读时，可以按默认权限创建。

### 8.8 反序列化：不可信数据禁止 `pickle` / `shelve`

🔴 **规则 8.8.1**：**禁止对不可信来源的数据进行 unpickle 和 load shelf**（SEC 4.1）。

💡 **理由**：`pickle` 在**反序列化时就可以执行任意代码**（通过 `__reduce__` 等机制），Python 官方文档明确说明它**不提供安全性保证**。`shelve` 依赖 `pickle`，从不可信来源 `load` 同样不安全。

```python
# ❌ Bad：反序列化来自网络/用户上传的数据
import pickle

data = pickle.loads(request.body)      # 攻击者可构造 payload 执行任意命令
```

```python
# ✅ Good：跨信任边界用"纯数据"格式 + 严格校验
import json

try:
    data = json.loads(request.body)
except json.JSONDecodeError as exc:
    raise ValueError("invalid request body") from exc
if not isinstance(data, dict) or set(data) - {"name", "age"}:
    raise ValueError("unexpected fields")
```

📌 **易错点**：`pickle` 只适合"自己写入、自己读取、全程不离开信任边界"的场景（如本机缓存）；即便如此，也要考虑缓存文件被篡改的风险。

### 8.9 敏感数据：加密、签名、脱敏与不硬编码

🔴 **规则 8.9.1**：**将敏感对象发送出信任区域前进行签名并加密**（SEC 4.2）。

💡 **理由**：敏感数据传输过程中要防止窃取和恶意篡改。以下场景需要密封/数字签名：① 序列化或传输敏感数据；② 没有使用 SSL 之类的安全通道；③ 敏感数据需长久保存。加密算法选择参考公司《密码算法应用规范》。

**例外**：为已加密对象签名在特定场景是合理的（验证真实性）；签名/加密只对**必须跨信任边界**的对象是必需的，始终在信任边界内的对象不需要。

🔴 **规则 8.9.2**：**禁止序列化未加密的敏感数据**（SEC 4.3）。

💡 **理由**：序列化只保存对象状态，**不提供安全机制**。可访问序列化数据的攻击者可获取敏感信息、推断实现细节，也可恶意修改数据。因此序列化信息中不应包括**密钥、数字证书**，以及那些在序列化时引用敏感数据的类。

```python
# ❌ Bad：把坐标等敏感字段一并序列化
class GPSLocation:
    def __init__(self, x, y, location_id):
        self.x = x
        self.y = y
        self.location_id = location_id


# ✅ Good：用 __getstate__ 排除敏感字段
class GPSLocation:
    def __init__(self, x, y, location_id):
        self.x = x
        self.y = y
        self.location_id = location_id

    def __getstate__(self):
        state = dict(self.__dict__)
        del state["x"]
        del state["y"]
        return state
```

🔴 **规则 8.9.3**：**禁止将敏感信息硬编码在程序中**（SEC 6.4）。

💡 **理由**：把口令、加密密钥硬编码在代码里，任何能访问 `pyc` 的人（**不需要反编译**，用字符串提取即可）都能拿到。应使用配置中心、密钥管理服务、环境变量（配合最小权限）等方式。

```python
# ❌ Bad
class DefaultPassword:
    DEFAULT_PASSWORD = "123456"


# ✅ Good：从受保护的配置/密钥管理系统读取，且不落日志
import os

password = os.environ["DB_PASSWORD"]     # 由部署平台注入，源码中不出现明文
```

🔴 **规则 8.9.4**：**禁止在日志中保存口令、密钥等敏感信息**（SEC 6.1）。

💡 口令包括**明文口令和密文口令**。若因特殊原因必须打印，用**固定长度的星号**代替。

```python
# ❌ Bad
logger.info("login user=%s password=%s", username, password)

# ✅ Good
logger.info("login user=%s password=%s", username, "*" * 8)
```

🔴 **规则 8.9.5**：**禁止使用私有或者弱加密算法**（SEC 6.2）。

💡 禁止 DES、MD5 等；应使用经过验证的、安全的、公开的加密算法（如 AES-256-GCM、SHA-256 及以上、RSA-2048+、ECC P-256+）。算法选型遵循公司《密码算法应用规范》。

```python
# ❌ Bad：MD5 做口令哈希 / DES 加密
import hashlib
digest = hashlib.md5(password.encode()).hexdigest()

# ✅ Good：加盐 + 慢哈希（口令场景用 bcrypt/scrypt/argon2，而不是裸 SHA-256）
import secrets
import hashlib

salt = secrets.token_bytes(16)
digest = hashlib.pbkdf2_hmac("sha256", password.encode(), salt, 600_000)
```

🔴 **规则 8.9.6**：**基于哈希算法的口令安全存储必须加入盐值（salt）**（SEC 6.3）。

💡 **理由**：不加盐时 `密文 = H(明文)`，攻击者拿到密文且知道算法后可**字典攻击**。加盐后 `密文 = H(明文 + 盐值)`，盐值随机，相同口令得到不同密文，大幅增加破解难度。生产环境推荐直接用 `bcrypt`、`argon2-cffi` 等专用库（自带盐与工作因子）。

```python
# ✅ Good：bcrypt（自动加盐、可调 cost）
import bcrypt

hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))
if bcrypt.checkpw(password.encode(), hashed):
    ...
```

### 8.10 随机数：安全场景不能用 `random`

🔴 **规则 8.10.1**：**使用安全随机数**（SEC 6.5）。

💡 **理由**：`random` 模块实现的是**伪随机数生成器**（可用于模拟、抽样、游戏），它**不能用于安全加密目的**。口令、令牌、会话 ID、盐值、验证码等必须使用密码学安全的随机源：Python 3.6+ 用 `secrets` 模块；需要 `SystemRandom` 时用 `random.SystemRandom`（底层调用 OS 的 CSPRNG）；类 Unix 下也可读 `/dev/urandom`（`/dev/random` 可能阻塞，工程上更推荐 `secrets`）。

```python
# ❌ Bad：伪随机数用于安全场景
import random

token = random.randint(0, 100)
reset_code = "".join(random.choice("0123456789") for _ in range(6))

# ✅ Good：secrets 模块
import secrets

token = secrets.token_urlsafe(32)          # 会话令牌/重置令牌
reset_code = "".join(secrets.choice("0123456789") for _ in range(6))
salt = secrets.token_bytes(16)

# ✅ 也可以用 SystemRandom（接口与 random 一致）
import random

secure_random = random.SystemRandom()
value = secure_random.randint(0, 100)
```

📌 **易错点**：`random.seed(42)` 之后序列可复现——这在测试里是优点，在安全场景里是灾难。

### 8.11 传输安全：敏感数据用 TLS

🔴 **规则 8.11.1**：**使用 SSLSocket 代替 Socket 来进行安全数据交互**（SEC 6.6）。

💡 **理由**：在不安全通道传输敏感数据时，必须使用 SSL/TLS，而不是裸 socket。TLS 提供：**完整性**（防篡改）、**认证**（对端身份）、**保密性**（加密，防窃听）。

```python
# ❌ Bad：裸 socket 明文传输
import socket

with socket.create_connection(("example.com", 443)) as sock:
    sock.sendall(b"password=123456")
```

```python
# ✅ Good：用 ssl 包装（现代写法；ssl.wrap_socket 已废弃）
import socket
import ssl

context = ssl.create_default_context()          # 默认校验证书与主机名
with socket.create_connection(("example.com", 443)) as sock:
    with context.wrap_socket(sock, server_hostname="example.com") as tls:
        tls.sendall(b"password=123456")
```

📌 **易错点**：
- `ssl._create_unverified_context()` / `verify_mode = CERT_NONE` 会关闭证书校验，等于没有认证，**生产禁用**。
- 例外：数据不敏感；数据已恰当加密；网络路径没有越出信任边界（两端都在可信内网且整网可信）——这三种情况裸 socket 可接受。

### 8.12 运行环境与代码清理

🔴 **规则 8.12.1**：**生产代码不能包含任何调试入口点**（SEC 5.1）。

💡 为调试/测试添加的调测代码若被带进生产，等于给系统留了"后门入口"。**禁止**：可由外部触发或默认开启的隐藏调试入口（如 `if request.args.get("debug"): ...`）、硬编码测试账号、`pdb.set_trace()`、未删除的 `/debug` 路由。调测能力应通过配置开关控制且**默认关闭**，生产构建中不打包（见下方 ✅ Good）。

```python
# ❌ Bad：调试后门
@app.route("/admin")
def admin():
    if request.args.get("debug") == "1":     # 任何人带参数即可绕过鉴权
        return dump_all_users()
    return require_admin()
```

```python
# ✅ Good：调测能力通过配置开关控制且默认关闭，生产构建中不打包
if settings.ENABLE_PROFILER:          # 由部署配置控制，生产为 False
    enable_profiler(app)
```

🔴 **规则 8.12.2**：**禁止通过注释的方式删除无用的功能代码**（SEC 6.8）。

💡 **理由**：Python 是解释执行的语言，即使是 `pyc` 也能被反编译。注释掉的功能/接口可能被"别有用心的人"恢复启用；尤其是接口函数，如果没有彻底删除，可能在不知情时被启用本应屏蔽的功能。因此**不使用的功能、模块、函数、变量必须在代码中彻底删除**。版本库会保留历史，不需要靠注释"留档"。

```python
# ❌ Bad：注释掉的功能接口，等于把"待启用"的后门留在代码里
if option == "load":
    LoadCmd(option, sys.argv[2:3][0])
# elif option == "active":
#     ActiveCmd(sys.argv[2:3][0])
# elif option == "inactive":
#     InActiveCmd(sys.argv[2:3][0])
```

```python
# ✅ Good：不需要就删除；需要保留能力就用配置/特性开关显式管理
if option == "load":
    LoadCmd(option, sys.argv[2:3][0])
else:
    Loginfo("Command %s is unknown" % sys.argv[1])
```

🔴 **规则 8.12.3**：**对子类继承的变量要做显式定义和赋初值**（SEC 6.7）。

💡 **理由**：Python 中类变量以字典处理并遵循 MRO。子类**没有定义**的属性会引用基类的属性值；基类属性值变化时，子类读到的值也随之变化。对"子类应当有自己取值"的类变量，必须在子类中显式定义，避免与基类联动。

```python
# ❌ Bad：C 没有定义 x，读 C.x 实际读到 A.x；A.x 一改，C.x 跟着变
class A:
    x = 1


class B(A):
    pass


class C(A):
    pass


B.x = 2
print(A.x, B.x, C.x)      # 1 2 1
A.x = 3
print(A.x, B.x, C.x)      # 3 2 3  ← C.x 跟着变了
```

```python
# ✅ Good：子类显式定义，互不影响
class A:
    x = 1


class B(A):
    x = 2


class C(A):
    x = -1


A.x = 3
print(A.x, B.x, C.x)      # 3 2 -1
```

🟡 **建议 8.12.4**：**慎用 `__del__` 方法**（SEC 建议 6.9）。

💡 **理由**：对象引用计数归零时会自动调用 `__del__`；但存在循环引用时对象可能无法销毁；程序结束时解释器按 `locals` 字典顺序回收，而不是按创建顺序，`__del__` 中访问全局数据/其他模块可能引发 `NameError`/`AttributeError`。不能保证 `__del__` 在程序结束时一定执行，**重要清理应显式做**。

```python
# ✅ Good：显式清理，或用 atexit 注册
import atexit


connection = open_connection("deaddot.com")


def cleanup():
    print("Going away...")
    close_connection(connection)


atexit.register(cleanup)
```

### 8.13 危险函数清单（评审必查）

| 函数/模块 | 风险 | 说明 |
| :--- | :---: | :--- |
| `eval()` / `exec()` | 🔴 极高 | 可执行任意 Python 代码 |
| `input()`（**仅 Python 2**） | 🔴 极高 | 等价于 `eval(raw_input())`；Python 3 无此问题 |
| `os.system()` / `os.popen()` | 🔴 极高 | 命令注入 |
| `pickle.load()` / `pickle.loads()` | 🔴 极高 | 反序列化可导致任意代码执行 |
| `shelve.open()` | 🟠 高 | 依赖 pickle |
| `tempfile.mktemp()` | 🟠 高 | 已废弃，符号链接攻击/竞争条件 |
| `random.random()` / `random.randint()` | 🟡 中 | 伪随机，安全场景禁用（用 `secrets`） |
| `subprocess.*(..., shell=True)` | 🟠 高 | 需避免拼接不可信输入 |
| `open(path)` 不校验路径 | 🟡 中 | 需 `realpath` + 目录限制 |
| `os.path.abspath()` 用于安全校验 | 🟡 中 | 不解析符号链接，应使用 `realpath()` |
| `yaml.load()`（未指定 Loader） | 🟠 高 | 可构造对象导致代码执行；用 `yaml.safe_load()` |
| `assert` 做权限/参数校验 | 🟡 中 | `python -O` 会移除 assert，校验失效 |

### 8.14 安全编码审计清单

**数据校验**
- [ ] 跨信任边界的数据是否校验？是否优先白名单？
- [ ] SQL 是否参数化？是否还有 f-string/% 拼接？
- [ ] 文件路径是否 `realpath` 标准化并限制在安全目录？
- [ ] 是否使用 `eval`/`exec` 处理不可信输入？
- [ ] 是否把不可信输入拼接进 `os.system`/`os.popen`/`shell=True`？

**异常行为**
- [ ] 是否存在抑制/忽略异常（空 except、只打印后继续）？
- [ ] 异常消息/类型是否泄露敏感信息？
- [ ] 异常发生时对象状态是否恢复？

**文件操作**
- [ ] 是否使用 `mktemp`？
- [ ] 临时文件是否及时删除？
- [ ] 创建文件是否指定合适权限（0600）？

**序列化**
- [ ] 是否对不可信数据 `unpickle` / `load shelf`？
- [ ] 敏感数据序列化前是否加密或排除（`__getstate__`）？

**敏感信息与密码学**
- [ ] 日志是否包含口令/密钥？
- [ ] 是否硬编码敏感信息？
- [ ] 是否使用 DES/MD5 等弱算法？口令是否加盐？
- [ ] 安全场景是否使用 `secrets`/`SystemRandom`？

**网络与运行环境**
- [ ] 敏感数据传输是否 TLS？是否关闭了证书校验？
- [ ] 生产代码是否含调试入口点？
- [ ] 是否用注释删除无用代码？
- [ ] 子类继承的类变量是否显式定义？

### 8.15 第 8 章速记

```text
数据校验：白名单优先，校验在信任边界内
SQL：参数化，绝不拼串（表名/列名只能白名单）
路径：realpath 标准化 + 目录限制（startswith 要带分隔符）
代码执行：禁 eval/exec（用 ast.literal_eval 处理字面量）
命令：禁 os.system/os.popen 拼输入；subprocess 用列表参数 shell=False
临时文件：禁 mktemp；及时删除；0600 权限
反序列化：不可信数据禁 pickle/shelve，用 json
敏感信息：不硬编码、不写日志、出信任边界要加密签名
密码学：禁 DES/MD5；口令加盐慢哈希；安全随机数用 secrets
传输：TLS；生产禁调试入口；禁注释代码；子类类变量显式赋值
```

---

## 第 9 章 测试与工程实践

### 9.1 可变默认参数：用 `None`，不要用 `[]` / `{}`

🔴 **规则**：函数参数中的**可变**参数不要使用默认值，在定义时使用 `None`（AS 规则 6.1）。

💡 **理由**：参数的默认值在**方法定义被执行时**就已经设定，**只会被设定一次**；函数定义后每次调用都复用同一个"预计算"的默认对象。当默认值是可变对象（list/dict/set）时，函数体内对它的修改会**累积影响下一次调用**。

```python
# ❌ Bad：items 是同一个 list，跨调用累积
def add_item(item, items=[]):
    items.append(item)
    return items


print(add_item("a"))    # ['a']
print(add_item("b"))    # ['a', 'b']  ← 本意是 ['b']
```

```python
# ✅ Good：默认 None，函数内新建（注意不要写 items = items or []，空列表会被替换）
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

```python
# ✅ Good（Python 3.5+）：确实需要"空容器默认值"时可考虑 dataclasses.field(default_factory)
from dataclasses import dataclass, field


@dataclass
class Batch:
    items: list = field(default_factory=list)
```

📌 **易错点**：
- `if not items: items = []` 会把调用方传入的**空列表**替换成新列表，违反"调用方传入的对象应被使用"的预期；必须用 `is None` 判断。
- 不可变默认值（`int`、`str`、`tuple`、`None`）没有这个问题。

### 9.2 测试用例：禁止用注释"停用"

🔴 **规则**：严禁使用**注释行等形式**仅使功能或测试用例失效；测试用例应该使用**用例级别控制**的方式不被调度，或者**直接删除**无效代码（AS 规则 6.2）。

💡 **理由**：注释掉的用例会持续腐烂（依赖过期、被人遗忘），也无法被任何工具统计；而"级别控制"（如 pytest marker / unittest skip）保留可追溯性，还能被 CI 统一报告。

```python
# ❌ Bad：注释掉失效用例
# def test_login_with_expired_token():
#     assert login(expired_token) is False

# ✅ Good：显式标记跳过，并说明原因与追踪单号
import os

import pytest


@pytest.mark.skip(reason="依赖的测试环境下线，见 ISSUE-1234")
def test_login_with_expired_token():
    assert login(expired_token) is False


# ✅ Good：条件跳过
@pytest.mark.skipif(not os.getenv("INTEGRATION"), reason="需要集成环境")
def test_device_connect():
    ...
```

### 9.3 测试设计要点

| 要点 | 说明 |
| :--- | :--- |
| 命名 | `test_` + 被测行为 + 场景 + 期望，如 `test_parse_config_missing_key_raises` |
| 隔离 | 每个用例独立，不依赖执行顺序、不共享可变全局状态 |
| 外部依赖 | 网络/数据库/时间/随机数用 mock/patch/fixture 替换，保证可重复 |
| 断言 | 一个用例聚焦一个行为；断言要具体（断言异常类型与消息，而不只是"抛了异常"） |
| 覆盖 | 正常路径 + 边界值 + 异常路径（空、None、超长、超范围、非法类型） |
| 不测实现细节 | 测"行为/契约"，避免重构时大量用例失效 |
| 失败信息 | 断言消息要能直接定位问题（带上实际值与期望值） |

```python
# ✅ Good：pytest 风格
import pytest


def test_load_batch_skips_blank_lines(tmp_path):
    path = tmp_path / "data.txt"
    path.write_text("a\n\nb\n", encoding="utf-8")

    assert load_batch(str(path)) == ["a", "b"]


def test_load_batch_missing_file_raises(tmp_path):
    with pytest.raises(FileNotFoundError):
        load_batch(str(tmp_path / "not_exist.txt"))
```

### 9.4 质量门禁：让规范可执行

规范如果只靠"评审时人肉检查"，一定会退化。工程上应配置**三层门禁**：

| 层级 | 手段 | 拦截问题 |
| :--- | :--- | :--- |
| 编辑器/保存时 | Black、isort、flake8 插件 | 格式、导入顺序、低级风格 |
| 提交前 | pre-commit 钩子 | 未格式化代码、明显 lint 错误、密钥泄露扫描 |
| CI | flake8 + pylint + mypy + bandit + 单测覆盖率 | 规范、类型、安全、回归 |

### 9.5 第 9 章速记

```text
可变默认参数用 None，函数内初始化
禁止注释停用测试；用 skip / 用例级别控制或删除
用例：独立、可重复、断言具体、覆盖边界与异常
规范落地靠门禁：编辑器 → pre-commit → CI
```

---

## 第 10 章 工具链与自动化

### 10.1 工具分工

| 工具 | 定位 | 能查什么 | 不能查什么 |
| :--- | :--- | :--- | :--- |
| **flake8** | 轻量 lint 聚合器 | PEP 8 风格（E/W）、未使用导入/变量（F）、复杂度（C） | 深层设计问题、安全问题不全面 |
| **pylint** | 重量级静态分析 | 命名规范、代码异味、重复代码、复杂度、未使用代码、部分错误模式 | 运行期行为、业务语义 |
| **black** | 格式化器（"不妥协"） | 统一缩进、行宽、引号、换行风格（自动改） | 不检查逻辑问题 |
| **isort** | 导入排序 | 导入分组与字母序 | 导入位置之外的规范 |
| **mypy** | 静态类型检查 | 类型注解一致性、`None` 处理、接口误用 | 无注解代码覆盖有限 |
| **bandit** | 安全静态扫描 | `eval`、`os.system`、`pickle`、硬编码口令等危险模式 | 业务逻辑漏洞 |
| **pre-commit** | 提交前钩子编排 | 把以上工具串起来，提交即拦截 | 不能替代 CI（可被 `--no-verify` 绕过） |
| **pytest + coverage** | 单测与覆盖率 | 回归、覆盖盲区 | 测试质量本身 |

💡 **关键认知**：**工具只能把"可机械判定"的规范自动化**。"注释是否解释了为什么""异常是否在正确层次处理""这个 try 是否包得太大"——仍然需要评审者的判断。规范意识 + 工具兜底 + 代码评审，三者缺一不可。

### 10.2 推荐配置（可直接落地）

**`pyproject.toml`（black / isort / mypy）**

```toml
[tool.black]
line-length = 120
target-version = ["py38", "py39", "py310", "py311"]

[tool.isort]
profile = "black"
line_length = 120
known_first_party = ["myproject"]

[tool.mypy]
python_version = "3.11"
warn_unused_ignores = true
warn_redundant_casts = true
disallow_untyped_defs = false
ignore_missing_imports = true
```

**`.flake8`（与 black 对齐）**

```ini
[flake8]
max-line-length = 120
extend-ignore = E203, W503
exclude = .git,__pycache__,build,dist,.venv
max-complexity = 15
per-file-ignores =
    __init__.py: F401
```

> 说明：`E203`（切片冒号前的空格）与 `W503`（二元运算符前换行）与 black 的格式化结果冲突，业界常见做法是忽略；`max-complexity` 用 McCabe 圈复杂度做"函数别写太长太绕"的量化门禁。

**`.pre-commit-config.yaml`**

```yaml
repos:
  - repo: https://github.com/psf/black
    rev: 24.8.0
    hooks: [{id: black}]
  - repo: https://github.com/PyCQA/isort
    rev: 5.13.2
    hooks: [{id: isort}]
  - repo: https://github.com/PyCQA/flake8
    rev: 7.1.1
    hooks: [{id: flake8}]
  - repo: https://github.com/PyCQA/bandit
    rev: 1.7.9
    hooks: [{id: bandit, args: ["-c", "pyproject.toml"]}]
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks: [{id: gitleaks}]
```

📌 **易错点**：
- 工具版本不固定（不锁 `rev`）会导致"昨天能过、今天全红"。
- 只加 flake8 不配 black：风格问题永远修不完；**格式化交给 black，语义问题交给 flake8/pylint**，分工明确。
- 把所有规则一次性全开会淹没在历史告警里；应**增量治理**（先对新增文件严，存量逐步清）。

### 10.3 第 10 章速记

```text
格式化：black（行宽与团队一致，华为口径 120）
导入：isort（profile=black，三段分组）
风格/错误：flake8（E/W/F/C，max-complexity）
深度静态：pylint；类型：mypy；安全：bandit；密钥：gitleaks
门禁：pre-commit（提交前）+ CI（合并前）
工具管"可机械判定"的，评审管"语义与设计"
```

---

## 第 11 章 常见反模式 Top 20

> 下表是评审中出现频率最高的 20 个问题，按"发现成本低、危害大"排序，可直接作为 code review checklist。

| # | 反模式 | 为什么错 | 正确做法 | 规范条目 |
| :-: | :--- | :--- | :--- | :--- |
| 1 | `except:` / `except Exception:` 包住大段逻辑 | 吞掉所有错误，隐藏 bug，影响退出信号 | 精确捕获预期异常，其它透传 | HW 13.3.5.2 |
| 2 | 空 `except: pass` | 故障被完全静默 | 记录 + 恢复/重抛 | SEC 2.1 |
| 3 | `except` 里 `return None` 不恢复状态 | 对象状态被污染，延迟爆炸 | 先校验、finally 回滚、副本提交 | SEC 2.3 |
| 4 | `def f(x=[])` / `def f(x={})` | 可变默认值跨调用共享 | `None` + 函数内初始化 | AS 规则 6.1 |
| 5 | `if x == None:` | `__eq__` 可被欺骗 | `is None` | AS 规则 4.1 |
| 6 | `type(x) == list` | 拒绝子类实例 | `isinstance(x, list)` | AS 建议 4.2 |
| 7 | `from xxx import *` | 静默覆盖、依赖不清、静态检查失效 | 显式导入或 `import xxx` | AS 规则 1.7 |
| 8 | `import sys, os` | 依赖不清晰 | 每模块一行 | HW 13.3.1.6 |
| 9 | `a = 1; b = 2` / `if x: return` | 无法单步调试 | 一行一条语句 | HW 13.3.1.3 |
| 10 | SQL 字符串拼接 / f-string 拼 SQL | SQL 注入 | 参数化查询 | SEC 1.2 |
| 11 | `eval()` / `exec()` 处理外部输入 | 任意代码执行 | 禁 eval/exec；`ast.literal_eval` | SEC 1.4 |
| 12 | `os.system("ls " + arg)` | 命令注入 | 标准库 API 或列表参数 `shell=False` | SEC 1.6 |
| 13 | `os.path.abspath` 做安全校验 | 不解析符号链接，可被绕过 | `os.path.realpath` + 目录限制 | SEC 1.3 |
| 14 | `pickle.loads(外部数据)` | 反序列化即代码执行 | 用 `json` 等纯数据格式 | SEC 4.1 |
| 15 | `random` 生成令牌/验证码 | 伪随机可预测 | `secrets` / `SystemRandom` | SEC 6.5 |
| 16 | 口令/密钥硬编码、写日志 | 源码/pyc 可提取，日志扩散 | 密钥管理服务；日志脱敏 | SEC 6.1/6.4 |
| 17 | `hashlib.md5(password)` 不加盐 | 彩虹表/字典攻击 | 加盐 + bcrypt/argon2/pbkdf2 | SEC 6.2/6.3 |
| 18 | 注释掉代码 / 保留调试后门 | 可被恢复启用，安全红线 | 彻底删除；调测开关默认关闭 | SEC 5.1/6.8 |
| 19 | `__del__` 里做关键清理 | 不保证执行、顺序不确定 | 显式清理 / `atexit` / `with` | SEC 6.9 |
| 20 | 注释与代码不一致 | 主动误导，比没注释更糟 | 改代码同步改注释 | AS 第 2 节 |

---

## 附录 A 速查表

### A.1 命名速查

| 对象 | 风格 | 示例 |
| :--- | :--- | :--- |
| 包 / 模块 | `lower_with_under` | `device_manager.py`、`protocol_adapters/` |
| 内部模块 | `_lower_with_under` | `_compat.py` |
| 类 / 异常 | `CapWords` | `DeviceAdapter`、`DeviceTimeoutError` |
| 函数 / 方法 | `lower_with_under()` | `get_device_status()` |
| 内部函数 | `_lower_with_under()` | `_parse_header()` |
| 参数 / 局部变量 / 实例变量 | `lower_with_under` | `timeout`、`device_id` |
| 全局变量 / 类变量 | `lower_with_under` | `device_cache`、`default_port` |
| 常量（全局/类） | `CAPS_WITH_UNDER` | `MAX_RETRY`、`DEFAULT_TIMEOUT` |
| 私有成员 | `_` 前缀 | `self._cache` |
| 继承防重名成员 | `__` 前缀（名字修饰） | `self.__update` → `_Class__update` |

### A.2 排版速查

```text
缩进          4 空格，禁 Tab，禁混用
编码          UTF-8，可写 # coding: utf-8
行宽          ≤ 120 字符（AS 规则 1.4 原文：小于 120）
续行          括号优先，反斜杠次之；操作符放行首/行尾（统一）
一行语句      只写一条
空行          独立程序块之间、变量说明之后必须加
空格          双目操作符前后；逗号后；切片冒号两侧不加
特例          x = x*2 - 1（* 与低优先级运算符连用不加空格）
导入          每模块一行；禁 import *；位置在 docstring 后、全局变量前
```

### A.3 异常处理速查

```text
try 最小化；只捕获必须处理的异常；其它透传
禁 except:；避免直接用 Exception
except 外 raise 必带异常；裸 raise 只用于重抛
retry 用有限循环 + 精确捕获；禁止 "打印堆栈后继续"
对外异常脱敏（消息 + 类型 + 是否存在）
raise ... from exc 保留因果链
异常时恢复对象状态（先校验 / finally / 副本 / 不改状态）
资源释放优先 with，其次 finally
```

### A.4 安全红线速查

```text
不可信数据：白名单校验
SQL：参数化；标识符白名单
路径：realpath + 目录限制（带分隔符）
代码执行：禁 eval/exec
命令：禁 os.system/os.popen 拼接；subprocess 列表参数 shell=False
临时文件：禁 mktemp；用完删除；0600
反序列化：禁 pickle/shelve 处理不可信数据
敏感信息：不硬编码、不进日志、出边界必加密签名
密码学：禁 DES/MD5；口令加盐慢哈希；随机数用 secrets
传输：TLS；生产无调试入口；禁注释代码
```

### A.5 文件模板

```python
# coding: utf-8
"""模块名与一句话功能说明。

功能：描述本模块解决的业务问题。
版权信息：华为技术有限公司，版权所有 (C) 2024
"""

import logging
import os
from typing import Optional

from myproject.common import config

LOGGER = logging.getLogger(__name__)

MAX_RETRY = 3              # 常量：大写+下划线
device_cache = {}          # 全局变量：小写+下划线，建议详细注释


class DeviceManager:
    """设备管理器，负责设备生命周期管理。

    功能描述：
        ...
    接口：
        register() / unregister() / get_status()
    """

    def __init__(self, config_path: str):
        self.config_path = config_path

    def register(self, serial: str) -> None:
        """注册设备。

        参数：
            serial (str)：设备序列号，24 位十六进制字符串。
        异常描述：
            ValueError：serial 格式非法时抛出。
        """
        ...
```

---

## 附录 B 提交前自检清单

**排版与格式**
- [ ] 4 空格缩进、无 Tab、无混用
- [ ] 文件 UTF-8（含中文时确认无 BOM）
- [ ] 每行 ≤ 120 字符；续行用括号、操作符位置统一
- [ ] 一行一条语句
- [ ] 独立程序块之间、变量说明之后有空行
- [ ] 双目操作符空格一致；逗号后空格；切片冒号不加空格

**导入**
- [ ] 每个模块单独一行
- [ ] 无 `from xxx import *`
- [ ] 导入位于模块 docstring 之后、全局变量之前
- [ ] 分组顺序：标准库 → 第三方 → 项目内

**注释与文档字符串**
- [ ] 模块 docstring（功能 + 版权）在文件顶部
- [ ] 类/公共函数 docstring 在声明下一行、缩进 4 空格
- [ ] 多行 docstring 结尾 `"""` 自成一行
- [ ] 注释与代码同缩进、位于上方相邻行；`# ` 有空格
- [ ] 无注释掉的代码、无过时注释、无未说明缩写

**命名**
- [ ] 模块/函数/变量 `lower_with_under`；类/异常 `CapWords`；常量 `CAPS_WITH_UNDER`
- [ ] 内部函数/成员用 `_` 前缀；未滥用 `__`
- [ ] 无 `i/j/k` 等无意义循环变量；无 `data/info/temp` 万能名
- [ ] 未遮蔽内置名（`list`、`type`、`id`、`input`…）

**编码**
- [ ] `None` 比较用 `is` / `is not`
- [ ] 类型检查用 `isinstance`
- [ ] 推导式不超过 3 个 `for`，无副作用
- [ ] 不需访问实例的方法用 `@staticmethod`/`@classmethod`
- [ ] 变量生命周期内类型稳定
- [ ] 公共接口有类型注解（推荐）

**异常**
- [ ] 无裸 `except:`；未直接用 `Exception` 兜底
- [ ] `try` 范围最小；异常向外透传
- [ ] `except` 外 `raise` 带异常对象
- [ ] 异常消息无敏感信息
- [ ] 异常时对象状态一致（`finally` / 先校验）
- [ ] 资源释放用 `with` / `finally`

**安全**
- [ ] 外部输入经白名单校验
- [ ] SQL 参数化；路径 `realpath` + 目录限制
- [ ] 无 `eval`/`exec` 处理外部输入；无 `os.system` 拼接
- [ ] 临时文件 `mkstemp`/`NamedTemporaryFile` 且及时删除；权限 0600
- [ ] 不可信数据不做 `pickle`/`shelve`
- [ ] 无硬编码敏感信息、日志无敏感信息
- [ ] 口令加盐 + 慢哈希；无 DES/MD5
- [ ] 安全随机数用 `secrets`
- [ ] 敏感传输用 TLS 且校验证书
- [ ] 生产代码无调试入口点；无用代码已删除

**测试与工程**
- [ ] 无可变默认参数
- [ ] 无注释掉的测试用例
- [ ] 新增逻辑有对应测试；边界/异常路径覆盖

---

## 附录 C 参考来源

1. 华为《开放可编程开发指南》（DevZone，V100R023C00，2023-08-08）第 13.3 节「Python 开发规范」（排版、注释、命名、编码、异常处理、并发与并行）与第 13.4 节「其他注意事项」。  
   <https://devzone.huawei.com/cn/enterprise/aoc/V100R023C00/pdf/%E5%BC%80%E6%94%BE%E5%8F%AF%E7%BC%96%E7%A8%8B%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%97.pdf>
2. 昇腾社区《python 语言编程指导（建议稿）》——排版、注释、命名、编码、异常处理、测试用例。  
   <https://raw.gitcode.com/AmmarDab3an/community/raw/master/docs/contributor/Ascend-python-coding-style-guide.md>
3. 昇腾社区《python 语言安全编程指导（建议稿）》——数据校验、异常行为、文件操作、序列化、运行环境、密码学等安全编码规则。  
   <https://raw.gitcode.com/McFlurry100/community/raw/master/docs/contributor/Ascend-python-secure-coding-guide.md>
4. 昇腾 MindSDK 参考应用《Python 安全编码规范（审计参考）》——危险函数清单与审计检查清单。  
   <https://raw.gitcode.com/Ascend/mindsdk-referenceapps/raw/master/AgentSDK/agent_skills/skills/sdk-security-audit/references/security-issues-python.md>
5. Python 官方文档：PEP 8《Style Guide for Python Code》、`pickle` 模块安全警告、`secrets` 模块、`ssl` 模块。  
   <https://peps.python.org/pep-0008/> ｜ <https://docs.python.org/3/library/pickle.html> ｜ <https://docs.python.org/3/library/secrets.html>
6. 华为云 CodeArts Check 产品文档（代码检查规则集与规范落地思路）。

> **免责说明**：本文为依据公开资料整理的学习与工程参考，条款解释以华为/昇腾正式发布的最新规范为准；不同产品线的规范可能存在差异，落地时请以所在团队的规范库与评审标准为准。
