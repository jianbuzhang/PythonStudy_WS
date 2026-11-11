# Python 类与面向对象（OOP）完全指南

> **定位**：一份面向「考试 + 面试 + 工程实践」三合一的类与面向对象知识手册。
> **覆盖**：类与实例、属性查找顺序、三种方法、封装与 `property`、描述符、`__slots__`、继承与 MRO、多态与抽象基类、魔术方法、元类、`dataclass`/`Enum`、设计模式、易混淆对比表、输出预测题、编程练习（全部附可运行答案）。
> **约定**：语法基于 Python 3.8+，示例输出在 **CPython 3.13** 上逐一实测；`# 输出:` / `# =>` 表示真实运行结果，`# 报错:` 表示会抛出异常。

---

## 目录

- [第 0 章 全景：先把地图装进脑子](#第-0-章-全景先把地图装进脑子)
- [第 1 章 类与对象基础](#第-1-章-类与对象基础)
- [第 2 章 继承、MRO 与多态](#第-2-章-继承mro-与多态)
- [第 3 章 魔术方法（Python 对象模型的精髓）](#第-3-章-魔术方法python-对象模型的精髓)
- [第 4 章 进阶：生命周期、元类、dataclass 与设计模式](#第-4-章-进阶生命周期元类dataclass-与设计模式)
- [第 5 章 易混淆知识大对比（20 组）](#第-5-章-易混淆知识大对比20-组)
- [第 6 章 输出预测题（24 题）](#第-6-章-输出预测题24-题)
- [第 7 章 综合练习题库（24 题）](#第-7-章-综合练习题库24-题)
- [第 8 章 速查清单与口诀](#第-8-章-速查清单与口诀)

---

# 第 0 章 全景：先把地图装进脑子

## 0.1 五个核心心智模型

学 OOP 最怕"背语法"。Python 的面向对象其实只围绕 5 句话展开，后面所有细节都是它们的推论：

| # | 心智模型 | 一句话解释 | 直接推论 |
| --- | --- | --- | --- |
| 1 | **一切皆对象** | 类本身也是对象（`type` 的实例），函数、模块、整数都是对象 | 类可以有属性 `Dog.species`；类可以当参数传递；可以动态给类加方法 |
| 2 | **类是命名空间 + 模具** | 类体是一个执行一次的代码块，里面的名字存进类的 `__dict__` | 类体里 `if/for/def` 都会执行；类属性在类创建时就固定了 |
| 3 | **实例 = 一个 `__dict__` + 一个指向类的指针** | `obj.__dict__` 存实例属性，`type(obj)` 指向类 | 属性查找先看实例字典，再沿类的 MRO 向上找 |
| 4 | **方法是绑定到对象的函数** | 函数是**非数据描述符**，通过实例访问时自动"绑定 self" | `d.bark` 是 bound method；`d.bark = 1` 能遮蔽方法 |
| 5 | **继承决定"是什么"，组合决定"有什么"** | 继承复用接口与行为，组合复用实现 | 优先组合；继承只为多态和模板扩展 |

> 💡 **一句话总纲**：Python 的面向对象 = **命名空间查找（MRO）+ 描述符协议 + 魔术方法**。把这三件事吃透，90% 的"诡异"行为都能自己推出来。

## 0.2 知识树

```
面向对象
├── 基础
│   ├── 类 / 实例 / type
│   ├── __init__ / __new__ / self
│   ├── 类属性 vs 实例属性（可变共享陷阱）
│   └── 实例方法 / classmethod / staticmethod
├── 封装
│   ├── _ 约定 / __ 名称改写（name mangling）
│   ├── @property（校验、只读、计算属性）
│   ├── 描述符协议（__get__ / __set__ / __set_name__）
│   └── __slots__（省内存、禁动态属性）
├── 继承
│   ├── 重写 / super()
│   ├── 多继承 / C3 线性化 / MRO
│   ├── 多态 / 鸭子类型 / ABC / Protocol
│   └── 继承 vs 组合
├── 对象模型
│   ├── 魔术方法（表示、比较、容器、运算符、可调用、上下文）
│   ├── 属性访问钩子（__getattr__ / __getattribute__ / __setattr__）
│   ├── 生命周期（引用计数、__del__、GC）
│   └── 元类 / 类装饰器 / __init_subclass__
└── 现代工具箱
    ├── dataclass / NamedTuple / Enum / TypedDict
    └── 设计模式：单例、工厂、观察者、策略
```

## 0.3 三个层次的学习目标

| 层次 | 要求 | 对应章节 |
| --- | --- | --- |
| **会写** | 能定义类、写构造器、继承、重写方法 | 第 1、2 章 |
| **写对** | 知道属性从哪来、`super()` 走到哪、`__eq__` 与 `__hash__` 的联动 | 第 1、2、5 章 |
| **写妙** | 会用魔术方法、描述符、`dataclass`、元类，能识别并避开陷阱 | 第 3、4 章 |

---

# 第 1 章 类与对象基础

## 1.1 从定义到实例化

```python
class Dog:
    species = "Canis familiaris"          # 类属性：所有实例共享

    def __init__(self, name: str, age: int):
        self.name = name                  # 实例属性：每个实例独有
        self.age = age

    def bark(self) -> str:                # 实例方法
        return f"{self.name}: 汪！"


d = Dog("旺财", 3)
print(d.name, d.age, Dog.species)         # 输出: 旺财 3 Canis familiaris
print(d.bark())                           # 输出: 旺财: 汪！
print(type(d))                            # 输出: <class '__main__.Dog'>
print(type(Dog))                          # 输出: <class 'type'>
print(Dog.__mro__)                        # 输出: (<class '__main__.Dog'>, <class 'object'>)
```

**三个必须知道的时机**

1. **类体在类创建时执行一次**（不是每次实例化）。所以 `class` 体里能写 `if`、`for`、甚至 `print`。
2. **`__init__` 在每次实例化时执行**，负责给实例挂属性。
3. **类属性在类创建后仍可修改**，且立刻对所有"没覆盖它"的实例生效。

```python
print("bark" in Dog.__dict__)   # 输出: True   —— 方法存在类的命名空间里
print("name" in Dog.__dict__)   # 输出: False  —— 实例属性不在类里
```

> ⚠️ **考点**：`Dog.__dict__` 里存的是**函数对象**；`d.bark` 每次访问都会"临时"生成一个绑定方法对象（bound method），所以 `d.bark is d.bark` 为 `False`。

---

## 1.2 `self` 与绑定方法

### self 不是关键字，只是第一个参数的惯例名字

```python
class Counter:
    def __init__(this, start=0):      # 用 this 也完全合法，但不推荐
        this.value = start

    def add(self, n=1):
        self.value += n
        return self.value


c = Counter(10)
print(type(Counter.add))              # 输出: <class 'function'>
print(type(c.add))                    # 输出: <class 'method'>
print(c.add.__func__ is Counter.add)  # 输出: True   —— 绑定方法包装了原函数
print(c.add.__self__ is c)            # 输出: True   —— 绑定方法记住了实例
print(Counter.add(c, 5))              # 输出: 15     —— 显式传 self 等价于 c.add(5)
print(c.add(2))                       # 输出: 17
```

### 方法其实是"非数据描述符"

函数对象实现了 `__get__`，所以放在类里之后，通过实例访问会变成绑定方法。理解这一点，就能解释下面的现象：

```python
c.add = lambda n=1: 999      # 给实例挂一个同名属性
print(c.add())               # 输出: 999  —— 实例字典遮蔽了类中的函数
```

这是"函数是非数据描述符"的直接后果：**实例 `__dict__` 的优先级高于非数据描述符**（详见 [1.8 描述符](#18-描述符property-的底层原理)）。

> 🔄 **举一反三**
> 1. 为什么 `print(d.bark)` 显示 `<bound method Dog.bark of <Dog object ...>>`？—— 因为 `__repr__` 里带上了 `__self__`。
> 2. 把方法存进变量 `f = d.bark` 再调用 `f()`，`self` 还在吗？—— 在，绑定关系已经固化在 `f` 里。
> 3. `Dog.bark()` 直接调用为什么报错？—— 少传了 `self`，报 `TypeError: bark() missing 1 required positional argument: 'self'`。

---

## 1.3 `__init__` 与 `__new__`：谁先谁后

### 实例化的完整流程

```
Dog("旺财", 3)
   │
   ├─ 1. type.__call__(Dog, "旺财", 3)          # 类是可调用对象，由元类 type 的 __call__ 接管
   ├─ 2. Dog.__new__(Dog, "旺财", 3)            # 真正"造对象"，返回空实例
   ├─ 3. Dog.__init__(实例, "旺财", 3)          # 给实例初始化，返回值必须是 None
   └─ 4. 返回实例
```

```python
class Probe:
    def __new__(cls, *args, **kwargs):
        print(f"1) __new__  cls={cls.__name__}, args={args}")
        return super().__new__(cls)          # 必须返回一个实例

    def __init__(self, value):
        print(f"2) __init__ value={value}")
        self.value = value


p = Probe(42)
# 输出:
# 1) __new__  cls=Probe, args=(42,)
# 2) __init__ value=42
```

### 关键规则（考试高频）

| 规则 | 说明 |
| --- | --- |
| `__new__` 是**隐式静态方法** | 第一个参数是 `cls`，不需要 `@staticmethod` 装饰 |
| `__init__` 返回非 `None` | 抛 `TypeError: __init__() should return None` |
| `__new__` 返回的**不是 cls 的实例** | `__init__` **不会被调用** |
| 想改不可变类型（`int`/`str`/`tuple`） | 必须重写 `__new__`，因为 `__init__` 改不动 |

```python
class Bad:
    def __init__(self):
        return 1
# Bad()  # 报错: TypeError: __init__() should return None, not 'int'
```

```python
class OnlyInt:
    """__new__ 返回了别的类型 → __init__ 被跳过"""
    def __new__(cls, kind):
        if kind == "int":
            return 42                    # 不是 cls 的实例
        return super().__new__(cls)

    def __init__(self, kind):
        print("__init__ 执行了")

print(OnlyInt("int"))    # 输出: 42      （没有打印"__init__ 执行了"）
print(OnlyInt("obj"))    # 输出: __init__ 执行了
                         #       <__main__.OnlyInt object at 0x...>
```

### 应用：单例模式的最简写法

```python
class Singleton:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

a, b = Singleton(), Singleton()
print(a is b)     # 输出: True
```

> ⚠️ 这个写法在**被继承时有坑**：`_instance` 是类属性，子类会共享父类的实例。更严谨的做法是 `cls.__dict__.get("_instance")` 或使用元类，见 [4.2](#42-元类类装饰器与-__init_subclass__)。

> 🔄 **单例的 4 种写法举一反三**
> 1. **模块级变量**（最 Pythonic）：`# singleton.py` 里直接 `instance = Config()`。
> 2. **`__new__`**：上面这种，适合单类。
> 3. **装饰器**：闭包缓存实例。
> 4. **元类**：`__call__` 里缓存，天然支持继承。

---

## 1.4 类属性 vs 实例属性

### 读取与写入的规则完全不同

| 操作 | 规则 |
| --- | --- |
| **读** `obj.attr` | 先找 `obj.__dict__` → 再沿 `type(obj).__mro__` 找类属性 |
| **写** `obj.attr = v`（普通情况） | **永远写在 `obj.__dict__`**，不会改类属性 |
| **修改类属性** `Cls.attr = v` | 改的是类，所有未覆盖它的实例都会看到新值 |
| **原地修改** `obj.list.append(x)` | 若 `list` 来自类属性，改的是**同一个共享对象** |

```python
class Config:
    retries = 3
    hosts = []

c1, c2 = Config(), Config()

c1.retries = 5                      # 写：只写进 c1.__dict__
print(Config.retries, c1.retries, c2.retries)   # 输出: 3 5 3

c1.hosts.append("h1")               # 读 + 原地改：改的是类属性本身
print(Config.hosts, c2.hosts, c1.hosts)          # 输出: ['h1'] ['h1'] ['h1']
```

### 三个经典陷阱

**陷阱 1：可变类属性被所有实例共享**

```python
class Student:
    courses = []                    # 错：所有学生共用一份列表

    def add(self, course):
        self.courses.append(course)

s1, s2 = Student(), Student()
s1.add("数学")
print(s2.courses)                   # 输出: ['数学']  ← 串数据了
```

正确写法：在 `__init__` 里创建，或用 `field(default_factory=list)`（dataclass）。

**陷阱 2：`+=` 对不可变类属性是"读 + 写实例"**

```python
class C:
    n = 0

c = C()
c.n += 1          # 等价于 c.n = c.n + 1：先读类属性 0，再把 1 写进实例
print(C.n, c.n)   # 输出: 0 1
```

**陷阱 3：可变默认参数（方法同样中招）**

```python
class Basket:
    def add(self, item, bucket=[]):     # 默认值只在函数定义时创建一次
        bucket.append(item)
        return bucket

print(Basket().add("苹果"))             # 输出: ['苹果']
print(Basket().add("香蕉"))             # 输出: ['苹果', '香蕉']  ← 坑
```

修复：`def add(self, item, bucket=None): bucket = [] if bucket is None else bucket`。

> 💡 **判断口诀**：**读靠 MRO，写进实例**；**可变对象跟"谁持有"走**。

---

## 1.5 实例方法 / 类方法 / 静态方法

```python
class Temperature:
    unit = "C"

    def __init__(self, celsius):
        self.celsius = celsius

    # ① 实例方法：需要 self，可读写实例与类状态
    def to_fahrenheit(self):
        return self.celsius * 9 / 5 + 32

    # ② 类方法：第一个参数是 cls，天然多态（cls 会变成子类）
    @classmethod
    def from_fahrenheit(cls, f):
        return cls((f - 32) * 5 / 9)

    # ② 类方法：操作类状态
    @classmethod
    def set_unit(cls, unit):
        cls.unit = unit

    # ③ 静态方法：与 self/cls 都无关，只是"挂在类名下的工具函数"
    @staticmethod
    def is_valid(celsius):
        return -273.15 <= celsius <= 1e6


t = Temperature.from_fahrenheit(212)
print(t.celsius)                        # 输出: 100.0
print(type(t).__name__)                 # 输出: Temperature

class Kelvin(Temperature):
    unit = "K"

print(type(Kelvin.from_fahrenheit(212)).__name__)   # 输出: Kelvin  ← 继承时 cls 自动变子类
print(Temperature.is_valid(20))                     # 输出: True
```

### 三种方法的绑定结果对照表

| 访问方式 | 得到什么 | `type(...)` |
| --- | --- | --- |
| `Temperature.to_fahrenheit` | 普通函数 | `<class 'function'>` |
| `t.to_fahrenheit` | 绑定到 `t` 的方法 | `<class 'method'>` |
| `Temperature.from_fahrenheit` | 绑定到**类**的方法 | `<class 'method'>` |
| `t.from_fahrenheit` | 绑定到**类**的方法（不是实例！） | `<class 'method'>` |
| `Temperature.is_valid` | 普通函数 | `<class 'function'>` |
| `t.is_valid` | 普通函数 | `<class 'function'>` |

```python
print(Temperature.from_fahrenheit.__self__ is Temperature)   # 输出: True
print(t.from_fahrenheit.__self__ is Temperature)             # 输出: True
```

### 选择建议

| 场景 | 选谁 | 例子 |
| --- | --- | --- |
| 需要读写实例属性 | 实例方法 | `account.deposit(100)` |
| 需要"备用构造器"、或要操作类状态 | `@classmethod` | `datetime.fromtimestamp()`、`dict.fromkeys()` |
| 与类和实例都无关的纯工具 | `@staticmethod` | 参数校验、格式转换 |
| 只是工具、不需要类名做命名空间 | 模块级函数 | 更简洁，优先考虑 |

> ⚠️ **易混点**：`classmethod` 里写 `cls(...)` 而不是 `Temperature(...)`，才能在子类调用时返回子类实例——这是"类方法支持多态"的关键。

> 🔄 **举一反三**：`dict.fromkeys()`、`datetime.fromtimestamp()`、`str.maketrans()` 都是类方法；`str.isdigit` 用法类似静态方法（但它是 C 实现）。想想 `int.from_bytes()` 为什么用类方法？——因为它要支持 `bool`/`IntEnum` 等子类。

---

## 1.6 访问控制：`_` 与 `__`（name mangling）

Python 没有真正的私有，只有**约定**和**名称改写**。

| 写法 | 名称 | 语义 | 运行时效果 |
| --- | --- | --- | --- |
| `name` | 公开 | 随便用 | 无 |
| `_name` | 受保护（约定） | "内部使用，别碰" | **无任何强制**，只是给人和 IDE 看 |
| `__name` | 私有（约定 + 改写） | 类内部实现细节 | 被改写为 `_ClassName__name` |
| `__name__` | 魔术名 | 协议钩子 | **不改写**（首尾都是双下划线） |

```python
class Account:
    def __init__(self, balance):
        self.__balance = balance            # 实际存成 _Account__balance

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("金额必须为正")
        self.__balance += amount

    def get_balance(self):
        return self.__balance


acc = Account(100)
acc.deposit(50)
print(acc.get_balance())      # 输出: 150
print(acc.__dict__)           # 输出: {'_Account__balance': 150}
print(hasattr(acc, "__balance"))       # 输出: False
print(hasattr(acc, "_Account__balance"))  # 输出: True  ← 改写后可访问，所以不是真私有
```

### 名称改写的三个细节

1. **只发生在类体内**（包括方法体里的 `self.__x`），类体外写 `acc.__balance` 不会被改写，直接 `AttributeError`。
2. **子类不共享私有属性**：子类里的 `self.__secret` 会改写成 `_子类名__secret`，和父类的 `_父类名__secret` 是两个不同的属性。
3. **改写只针对"至多一个尾下划线"**：`__x`、`__x_` 会改写；`__x__` 不改写。

```python
class Account:
    def __init__(self):
        self.__secret = "账户内部"

    def get(self):
        return self.__secret


class Vip(Account):
    def __init__(self):
        super().__init__()
        self.__secret = "VIP 自己的"     # 变成 _Vip__secret

    def get_vip(self):
        return self.__secret


v = Vip()
print(v.get())        # 输出: 账户内部
print(v.get_vip())    # 输出: VIP 自己的
print(v.__dict__)     # 输出: {'_Account__secret': '账户内部', '_Vip__secret': 'VIP 自己的'}
```

> ⚠️ **面试常问**：如何真正禁止外部访问？—— Python 做不到强制私有；如果需要"只能读不能写"，用 `@property` 只给 getter；如果需要"禁止动态加属性"，用 `__slots__`。名称改写只是为了**避免继承时的命名冲突**，不是安全机制。

---

## 1.7 `property`：把方法用成属性

`property` 是最常用的"受控属性"工具：对外看起来是属性，内部走函数，可以加校验、计算、缓存、日志。

```python
class Circle:
    PI = 3.1415926

    def __init__(self, radius):
        self.radius = radius            # 赋值时触发 setter

    @property
    def radius(self):
        return self._radius             # 内部用 _radius 存，避免无限递归

    @radius.setter
    def radius(self, value):
        if not isinstance(value, (int, float)):
            raise TypeError("半径必须是数字")
        if value <= 0:
            raise ValueError("半径必须为正数")
        self._radius = value

    @property
    def area(self):                     # 只读计算属性
        return Circle.PI * self._radius ** 2


c = Circle(2)
print(c.radius, round(c.area, 2))       # 输出: 2 12.57
c.radius = 3
print(c.radius, round(c.area, 2))       # 输出: 3 28.27

try:
    c.radius = -1
except ValueError as e:
    print("ValueError:", e)             # 输出: ValueError: 半径必须为正数

try:
    c.area = 100                        # area 没有 setter
except AttributeError as e:
    print("只读属性赋值失败:", type(e).__name__)   # 输出: 只读属性赋值失败: AttributeError
```

### 关键点

| 要点 | 说明 |
| --- | --- |
| 内部存储名要用 `_radius` | 若 getter/setter 里写 `self.radius`，会无限递归直到 `RecursionError` |
| 只给 getter = 只读属性 | 赋值会 `AttributeError: property 'area' of 'Circle' object has no setter` |
| 必须先有 `@property` 再有 `@x.setter` | setter 装饰器依赖前一个 property 对象 |
| `property` 是**数据描述符** | 优先级高于实例 `__dict__`，所以 `c.__dict__["radius"] = 999` 也挡不住 setter |
| 校验失败抛 `ValueError` 比 `assert` 好 | `assert` 在 `-O` 模式下会被移除 |

### 举一反三：property 的 4 种用法

```python
class User:
    def __init__(self, first, last):
        self.first, self.last = first, last

    # ① 计算属性
    @property
    def full_name(self):
        return f"{self.first} {self.last}"

    # ② 只读属性
    @property
    def upper_name(self):
        return self.full_name.upper()

    # ③ 带校验的 setter（同上例）

    # ④ 缓存（Python 3.8+）
    @property
    def expensive(self):
        if not hasattr(self, "_cache"):
            self._cache = sum(range(10 ** 6))
        return self._cache

    # ⑤ 3.8 起还有 functools.cached_property（非数据描述符，可被实例字典覆盖）
```

> 💡 **`cached_property` vs `property`**：前者只在第一次计算并写入实例 `__dict__`，之后直接读字典（更快）；后者每次访问都执行函数。前者要求类有 `__dict__`（`__slots__` 类不能用）。

---

## 1.8 描述符：`property` 的底层原理

**描述符 = 定义了 `__get__` / `__set__` / `__delete__` 中任意一个的类**，它的实例作为**类属性**存在时，会接管对它的访问。

| 类型 | 实现的方法 | 例子 | 与实例字典的优先级 |
| --- | --- | --- | --- |
| **数据描述符** | 有 `__set__` 或 `__delete__` | `property`、`__slots__` 的成员描述符 | **高于**实例 `__dict__` |
| **非数据描述符** | 只有 `__get__` | 函数、`staticmethod`、`classmethod`、`cached_property` | **低于**实例 `__dict__` |

### 完整属性查找顺序（背下来）

```
obj.attr 的查找顺序：
1. type(obj).__mro__ 中找到的数据描述符（有 __set__/__delete__）→ 调用 __get__
2. obj.__dict__["attr"]
3. type(obj).__mro__ 中找到的普通类属性 / 非数据描述符 → 调用 __get__
4. type(obj).__getattr__(obj, "attr")（若定义）
5. 抛 AttributeError
```

```python
class DataDesc:
    def __get__(self, obj, owner=None):
        return "数据描述符胜出"
    def __set__(self, obj, value):
        pass

class NonDataDesc:
    def __get__(self, obj, owner=None):
        return "非数据描述符"

class Demo:
    data = DataDesc()
    nondata = NonDataDesc()


d = Demo()
d.__dict__["data"] = "实例字典"
d.__dict__["nondata"] = "实例字典"
print(d.data)       # 输出: 数据描述符胜出
print(d.nondata)    # 输出: 实例字典
```

### 实战：类型校验描述符（可复用的"带校验字段"）

```python
class Positive:
    """把任意属性变成"必须为正数"的字段"""

    def __set_name__(self, owner, name):     # 类创建时自动调用
        self.storage = "_" + name

    def __get__(self, obj, objtype=None):
        if obj is None:                      # 通过类访问：Order.price
            return self
        return getattr(obj, self.storage)

    def __set__(self, obj, value):
        if value <= 0:
            raise ValueError(f"{self.storage} 必须为正数")
        setattr(obj, self.storage, value)


class Order:
    price = Positive()
    qty = Positive()

    def __init__(self, price, qty):
        self.price = price
        self.qty = qty

    def total(self):
        return self.price * self.qty


o = Order(10, 3)
print(o.total(), o.price)        # 输出: 30 10
try:
    o.qty = 0
except ValueError as e:
    print("ValueError:", e)      # 输出: ValueError: _qty 必须为正数
```

> 💡 `__set_name__` 是 Python 3.6+ 的钩子：类创建时，每个描述符都会收到"我被赋给了哪个属性名"，因此可以自动生成存储名。`dataclass`、`attrs`、SQLAlchemy 都重度使用描述符。

> 🔄 **举一反三**：把 `Positive` 改成 `Typed(int)`、`Range(0, 100)`、`Repeated`（自动去重）——同一套协议可以造出无数"字段校验器"，这就是 ORM 框架的底层原理。

---

## 1.9 `__slots__`：省内存与禁止动态属性

默认每个实例都有一个 `__dict__`（哈希表），属性多时内存开销大。`__slots__` 把属性改成**固定偏移的槽位**，并在类里生成**成员描述符**。

```python
class Point:
    __slots__ = ("x", "y")

    def __init__(self, x, y):
        self.x, self.y = x, y


p = Point(1, 2)
print(p.x, p.y)                  # 输出: 1 2
print(hasattr(p, "__dict__"))    # 输出: False
try:
    p.z = 3
except AttributeError:
    print("AttributeError: 不允许新增属性")
```

### 继承时的行为

```python
class Point3D(Point):
    __slots__ = ("z",)           # 子类追加槽位

    def __init__(self, x, y, z):
        super().__init__(x, y)
        self.z = z


p3 = Point3D(1, 2, 3)
print(p3.x, p3.y, p3.z)          # 输出: 1 2 3
print(hasattr(p3, "__dict__"))   # 输出: False
```

| 注意点 | 说明 |
| --- | --- |
| 子类不写 `__slots__` | 子类实例又会获得 `__dict__`，父类的省内存效果被抵消 |
| 槽位名与类属性同名 | 类创建时抛 `ValueError: 'x' in __slots__ conflicts with class variable` |
| 想支持弱引用 | 要加 `"__weakref__"`（父类已有则不用） |
| 想支持动态属性 | 加 `"__dict__"`（等于放弃限制） |
| 与 `@property` 冲突 | 不能给同名槽位再定义 property；但可以槽位存 `_x`、property 暴露 `x` |
| 省内存幅度 | 属性少、实例多时非常可观（百万实例可省上百 MB） |

```python
try:
    class Bad:
        __slots__ = ("x",)
        x = 1
except ValueError as e:
    print("ValueError:", e)   # 输出: ValueError: 'x' in __slots__ conflicts with class variable
```

---

## 1.10 第 1 章易错清单

| # | 易错点 | 正确认知 |
| --- | --- | --- |
| 1 | 以为类属性改一个实例会同步 | 读共享、写各自；可变对象原地改才共享 |
| 2 | 用可变对象做默认参数 | 默认值只创建一次，会跨调用累积 |
| 3 | 以为 `__init__` 负责创建对象 | 创建是 `__new__` 的事，`__init__` 只做初始化 |
| 4 | 以为 `__` 是真正的私有 | 只是名称改写 `_类名__属性`，仍可访问 |
| 5 | 在 property 里写 `self.radius` | 无限递归，内部要用 `self._radius` |
| 6 | 以为 `self` 是关键字 | 只是第一个参数的惯例名 |
| 7 | 以为 `d.bark` 只是一个函数 | 是临时生成的绑定方法，每次访问都不相等 |
| 8 | `__slots__` 子类不声明 | 子类会重新拥有 `__dict__`，限制失效 |
| 9 | 类方法里写死类名 | 用 `cls(...)` 才能让子类构造出子类实例 |
| 10 | 静态方法里访问 `self` | `@staticmethod` 没有 `self`/`cls`，会 `NameError` |

---

# 第 2 章 继承、MRO 与多态

## 2.1 单继承：重写与 `super()`

```python
class Animal:
    def __init__(self, name):
        self.name = name

    def speak(self):
        return "..."

    def __repr__(self):
        return f"{type(self).__name__}({self.name!r})"


class Dog(Animal):
    def __init__(self, name, breed):
        super().__init__(name)          # 先让父类完成自己的初始化
        self.breed = breed

    def speak(self):                    # 重写（override）
        return f"{self.name}: 汪汪！"


class Cat(Animal):
    def speak(self):
        return f"{self.name}: 喵~"


animals = [Dog("旺财", "中华田园犬"), Cat("咪咪")]
for a in animals:
    print(a.speak())
# 输出:
# 旺财: 汪汪！
# 咪咪: 喵~

print(isinstance(animals[0], Animal))   # 输出: True
print(issubclass(Dog, Animal))          # 输出: True
print(animals)                          # 输出: [Dog('旺财'), Cat('咪咪')]
```

### 重写的两种姿势

| 姿势 | 写法 | 场景 |
| --- | --- | --- |
| **完全替换** | 直接重新实现，不调用父类 | 父类行为已不适用 |
| **扩展** | `super().method(...)` 后再追加 | 父类逻辑仍需要，只想补充 |

```python
class Logger:
    def log(self, msg):
        return f"[LOG] {msg}"


class TimestampLogger(Logger):
    def log(self, msg):
        return f"[{__import__('datetime').datetime.now():%H:%M:%S}] " + super().log(msg)
```

### `super()` 到底是什么

**`super()` 不是"父类"**，而是"`type(self).__mro__` 中排在当前类**之后**的那个类"。单继承时看起来像父类，多继承时就会"跳着走"。

| 写法 | 含义 | 建议 |
| --- | --- | --- |
| `super().method()` | 零参形式，依赖编译器注入的 `__class__` 单元格 | ✅ 首选 |
| `super(CurrentClass, self).method()` | 显式从 `CurrentClass` 之后开始找 | 只在需要跳过某层时用 |
| `BaseClass.method(self)` | 硬编码父类，**破坏 MRO**，菱形继承会被调用多次 | ❌ 避免 |

> ⚠️ 零参 `super()` 只能用在**类体内定义的方法**里；在普通函数、lambda、嵌套函数里会报 `RuntimeError: super(): no arguments`。

---

## 2.2 多继承与 MRO（C3 线性化）

```python
class A:
    def hello(self):
        return "A"


class B(A):
    def hello(self):
        return "B -> " + super().hello()


class C(A):
    def hello(self):
        return "C -> " + super().hello()


class D(B, C):
    def hello(self):
        return "D -> " + super().hello()


print([cls.__name__ for cls in D.__mro__])   # 输出: ['D', 'B', 'C', 'A', 'object']
print(D().hello())                            # 输出: D -> B -> C -> A
```

### 菱形继承的执行顺序

```python
class A:
    def f(self):
        print("A.f")


class B(A):
    def f(self):
        print("B.f")
        super().f()


class C(A):
    def f(self):
        print("C.f")
        super().f()


class D(B, C):
    def f(self):
        print("D.f")
        super().f()


D().f()
# 输出:
# D.f
# B.f
# C.f
# A.f        ← 每个类只执行一次，且严格按 MRO 的顺序
```

**结论**：在协作式多继承（每个类都调用 `super()`）中，方法沿着 MRO 链**依次**执行，形成一条"责任链"，而不是每个父类各调一次。

### C3 线性化的两条直觉规则

1. **子类永远排在父类前面**（单调性）。
2. **声明顺序 `class D(B, C)` 中，B 排在 C 前面**（局部优先级）。

C3 就是"在所有父类的 MRO 中，每次选一个没有被其他候选'挡在前面'的类"。冲突时 Python 直接拒绝创建类：

```python
class X:
    pass


class Y(X):
    pass


try:
    class Z(X, Y):      # X 要求在 Y 前，但 Y 是 X 的子类，必须排在 X 前 → 矛盾
        pass
except TypeError as e:
    print("TypeError:", e)
# 输出: TypeError: Cannot create a consistent method resolution order (MRO) for bases X, Y
```

### MRO 相关的三个 API

```python
print(D.__mro__)        # 元组：D, B, C, A, object
print(D.mro())          # 列表：同上（这是方法）
print(D.__bases__)      # 直接父类： (B, C)
print(D.__base__)       # 第一个直接父类：B
```

### 举一反三：手推 MRO

```python
class O: pass
class A(O): pass
class B(O): pass
class C(O): pass
class D(A, B): pass
class E(B, C): pass
class F(D, E): pass

print([c.__name__ for c in F.__mro__])
# 输出: ['F', 'D', 'A', 'E', 'B', 'C', 'O', 'object']
```

> 💡 记忆技巧：把每个类的 MRO 写成链，合并时保持"子类在前、声明顺序在左、父类在后"，遇到冲突就说明继承设计有问题。

---

## 2.3 多态：同一接口，不同实现

**多态 = 调用方只依赖"能做什么"，不关心"是什么"。** Python 的多态是**运行时**的，不需要接口声明。

```python
class Payment:
    def pay(self, amount):
        raise NotImplementedError


class Alipay(Payment):
    def pay(self, amount):
        return f"支付宝支付 {amount} 元"


class WechatPay(Payment):
    def pay(self, amount):
        return f"微信支付 {amount} 元"


def checkout(payment, amount):
    return payment.pay(amount)          # 只依赖 pay 这个"接口"


for p in (Alipay(), WechatPay()):
    print(checkout(p, 99))
# 输出:
# 支付宝支付 99 元
# 微信支付 99 元
```

### 鸭子类型（duck typing）

"如果它走起来像鸭子、叫起来像鸭子，那它就是鸭子。"——**不检查类型，只检查行为**。

```python
class Duck:
    def quack(self):
        return "嘎嘎"


class Person:
    def quack(self):                    # 没有继承任何基类
        return "我在学鸭子叫"


def make_it_quack(thing):
    return thing.quack()


print(make_it_quack(Duck()))            # 输出: 嘎嘎
print(make_it_quack(Person()))          # 输出: 我在学鸭子叫
# make_it_quack(42)  # 报错: AttributeError: 'int' object has no attribute 'quack'
```

**优点**：灵活、解耦。**缺点**：错误发现晚（运行到才炸），IDE/类型检查器难以推断。

### 多态的三种实现方式

| 方式 | 机制 | 检查时机 |
| --- | --- | --- |
| 继承 + 重写 | 子类覆盖父类方法 | 运行时 |
| 鸭子类型 | 只要有同名方法就行 | 运行时（可能很晚） |
| 抽象基类 / Protocol | 强制实现接口 | 实例化时（ABC）/ 静态检查（Protocol） |

---

## 2.4 抽象基类（ABC）

抽象基类用于**定义接口契约**：不能实例化，子类必须实现所有抽象方法。

```python
from abc import ABC, abstractmethod


class Shape(ABC):
    @abstractmethod
    def area(self):
        """子类必须实现"""

    def describe(self):                 # 抽象基类也可以有具体方法
        return f"{type(self).__name__}: 面积 {self.area():.2f}"


class Square(Shape):
    def __init__(self, side):
        self.side = side

    def area(self):
        return self.side ** 2


print(Square(3).describe())             # 输出: Square: 面积 9.00

try:
    Shape()
except TypeError as e:
    print("TypeError:", e)
# 输出: TypeError: Can't instantiate abstract class Shape without an implementation for abstract method 'area'

class Incomplete(Shape):                # 没实现 area
    pass

try:
    Incomplete()
except TypeError as e:
    print("TypeError:", e)
# 输出: TypeError: Can't instantiate abstract class Incomplete without an implementation for abstract method 'area'
```

### 抽象方法的三个细节

1. **抽象方法可以有函数体**，子类可以 `super().area()` 复用。
2. **子类只要还有一个抽象方法没实现，就不能实例化**（包括继承来的抽象方法）。
3. **抽象属性/类方法/静态方法**也能抽象，注意装饰器顺序：

```python
class Base(ABC):
    @property
    @abstractmethod
    def name(self):                     # 顺序：@property 在最外层
        ...

    @classmethod
    @abstractmethod
    def create(cls):
        ...

    @staticmethod
    @abstractmethod
    def helper():
        ...
```

### 虚拟子类：`register()`

```python
from abc import ABC


class Quacker(ABC):
    @abstractmethod
    def quack(self): ...


class RubberDuck:                       # 并没有继承 Quacker
    def quack(self):
        return "嘎"


Quacker.register(RubberDuck)            # 注册为"虚拟子类"
print(issubclass(RubberDuck, Quacker))  # 输出: True
print(isinstance(RubberDuck(), Quacker))# 输出: True
# 注意：register 不做任何方法检查，Quacker() 依然不能实例化

try:
    Quacker()
except TypeError as e:
    print("TypeError:", type(e).__name__)   # 输出: TypeError: TypeError
```

### 标准库中的 ABC

| ABC | 位置 | 需要实现 |
| --- | --- | --- |
| `Sequence` | `collections.abc` | `__getitem__`、`__len__`，免费送 `__contains__`、`index`、`count`、`__iter__` 等 |
| `MutableMapping` | `collections.abc` | `__getitem__`、`__setitem__`、`__delitem__`、`__iter__`、`__len__` |
| `Iterable` / `Iterator` | `collections.abc` | `__iter__` / `__next__` |
| `Hashable` | `collections.abc` | `__hash__` |
| `Callable` | `collections.abc` | `__call__` |

> 💡 **实战价值**：继承 `Sequence` 只写两个方法，就能白拿一整套序列行为，还能通过 `isinstance(x, Sequence)` 判断；这是"继承复用接口 + 混入实现"的典范。

---

## 2.5 `Protocol`：结构化类型（静态鸭子类型）

```python
from typing import Protocol, runtime_checkable


@runtime_checkable
class Speaker(Protocol):
    def speak(self) -> str: ...


class Robot:
    def speak(self) -> str:
        return "beep"


class Human:
    def speak(self) -> str:
        return "你好"


def greet(s: Speaker) -> str:           # 类型检查器按"结构"判断
    return s.speak()


print(greet(Robot()))                   # 输出: beep
print(isinstance(Robot(), Speaker))     # 输出: True   （runtime_checkable 提供）
print(isinstance(42, Speaker))          # 输出: False
```

| 对比项 | 抽象基类 ABC | Protocol |
| --- | --- | --- |
| 是否要求显式继承 | 是（或用 `register`） | **否**，结构匹配即可 |
| 检查时机 | 实例化时（运行时） | 类型检查器（静态），`runtime_checkable` 仅查属性名 |
| 能否提供默认实现 | 能（混入方法） | 不能 |
| 适合 | 框架内部强约束 | 描述"第三方/已有类"的接口 |
| 运行时 `isinstance` | 精确 | 只检查方法/属性是否存在，不检查签名 |

---

## 2.6 继承 vs 组合

### 先看一个"错误继承"的经典案例

```python
class Stack(list):                      # 错：栈不是列表
    def push(self, item):
        self.append(item)


s = Stack()
s.push(1)
s.push(2)
s.insert(0, 999)                        # 完全合法，但破坏了栈的语义
s.sort()                                # 还有 20 多个 list 方法被暴露
print(s)                                # 输出: [999, 1, 2]
```

**问题**：继承会把父类的**全部公开 API** 都变成子类的 API。如果子类并不满足"is-a"关系，就会出现语义污染。

### 组合：has-a 关系，委托实现

```python
class Stack:
    def __init__(self):
        self._items = []                # 组合：内部持有一个列表

    def push(self, item):
        self._items.append(item)

    def pop(self):
        if not self._items:
            raise IndexError("栈为空")
        return self._items.pop()

    def __len__(self):
        return len(self._items)

    def __bool__(self):
        return bool(self._items)


s = Stack()
s.push(1)
s.push(2)
print(s.pop(), len(s), bool(s))         # 输出: 2 1 True
# s.insert(0, 999)   # 报错: AttributeError: 'Stack' object has no attribute 'insert'
```

### 怎么选？

| 判断问题 | 选继承 | 选组合 |
| --- | --- | --- |
| 子类是否**是一种**父类（is-a）？ | ✅ | |
| 子类是否需要**被当作**父类使用（多态传参）？ | ✅ | |
| 只是想**复用几行代码**？ | | ✅ |
| 父类的部分方法对子类**不适用**？ | | ✅ |
| 需要在运行时**替换**行为？ | | ✅（依赖注入） |
| 需要继承多个来源的能力？ | ✅（Mixin） | 组合多个对象 |

> 💡 **设计原则**：**优先组合，继承只为多态和模板扩展**。判断标准不是"能不能继承"，而是"子类是否承诺遵守父类的全部契约"（里氏替换原则）。

### Mixin：多继承的正当用法

Mixin 是"只提供一小块能力、不表示 is-a 关系"的类，通常以 `Mixin` 结尾、不单独实例化。

```python
import json


class JsonMixin:
    def to_json(self):
        return json.dumps(self.__dict__, ensure_ascii=False, default=str)


class TimestampMixin:
    def created_text(self):
        return "刚刚"


class User(JsonMixin, TimestampMixin):
    def __init__(self, name, age):
        self.name = name
        self.age = age


u = User("小明", 18)
print(u.to_json())          # 输出: {"name": "小明", "age": 18}
print(u.created_text())     # 输出: 刚刚
```

### 继承内置类型：`__missing__` 的正确姿势

```python
class Counter(dict):
    def __missing__(self, key):
        return 0


c = Counter()
c["apple"] += 1                 # 读不存在的键触发 __missing__ → 0，再写回
print(c)                        # 输出: {'apple': 1}
print(c["banana"])              # 输出: 0   ← 纯读取不会写入字典
print(c)                        # 输出: {'apple': 1}
```

---

## 2.7 继承中的高频陷阱

### 陷阱 1：子类 `__init__` 忘记调用 `super().__init__()`

```python
class Parent:
    def __init__(self):
        self.ready = True


class Child(Parent):
    def __init__(self):
        self.name = "child"         # 忘了 super().__init__()


try:
    print(Child().ready)
except AttributeError:
    print("AttributeError: 父类的初始化被跳过了")
```

### 陷阱 2：多继承中 `super().__init__` 的参数链断裂

协作式多继承要求每一层都"接受并转发"参数，否则链会断：

```python
class A:
    def __init__(self, **kwargs):
        super().__init__(**kwargs)


class B(A):
    def __init__(self, b, **kwargs):
        self.b = b
        super().__init__(**kwargs)


class C(A):
    def __init__(self, c, **kwargs):
        self.c = c
        super().__init__(**kwargs)


class D(B, C):
    def __init__(self, b, c):
        super().__init__(b=b, c=c)


d = D(1, 2)
print(d.b, d.c)                 # 输出: 1 2
```

### 陷阱 3：用 `type()` 判断类型，漏掉子类

```python
class MyList(list):
    pass

x = MyList()
print(type(x) is list)          # 输出: False
print(isinstance(x, list))      # 输出: True   ← 判断类型优先用 isinstance
```

### 陷阱 4：子类重写方法时改变了语义

```python
class Bird:
    def fly(self):
        return "我在飞"


class Penguin(Bird):
    def fly(self):
        raise NotImplementedError("企鹅不会飞")   # 违背里氏替换：父类能做的事，子类做不了
```

更好的设计：把"会飞"抽成独立接口（`Flyable`），让 `Penguin` 不实现它。

### 陷阱 5：在父类 `__init__` 里调用会被子类重写的方法

```python
class Base:
    def __init__(self):
        self.setup()                # 危险：此时子类属性可能还没初始化

    def setup(self):
        print("Base.setup")


class Child(Base):
    def __init__(self):
        super().__init__()
        self.data = [1, 2, 3]

    def setup(self):
        print("Child.setup, data =", getattr(self, "data", "还没初始化"))


Child()
# 输出: Child.setup, data = 还没初始化
```

> 💡 这是模板方法模式的经典坑：父类调用钩子方法本身没问题，但要保证**钩子不依赖子类的初始化顺序**，或把钩子调用放到明确的 `run()` 里。

---

## 2.8 第 2 章易错清单

| # | 易错点 | 正确认知 |
| --- | --- | --- |
| 1 | 以为 `super()` 就是父类 | 是 MRO 中的下一个类 |
| 2 | 多继承里显式 `Base.method(self)` | 会破坏协作链、重复调用，应用 `super()` |
| 3 | 子类 `__init__` 不调 `super().__init__()` | 父类状态丢失 |
| 4 | 以为多继承按"从左到右依次执行" | 按 C3 MRO，且协作式只执行一次 |
| 5 | 用继承复用代码但 is-a 不成立 | 用组合，避免 API 污染 |
| 6 | ABC 子类漏实现抽象方法 | 实例化时报 `TypeError`，且继承来的抽象方法也算 |
| 7 | 以为 `register()` 会检查方法 | 不会，只是让 `isinstance/issubclass` 返回 `True` |
| 8 | `@property` + `@abstractmethod` 顺序写反 | `@property` 必须在外层 |
| 9 | 用 `type(x) is Base` 判断 | 用 `isinstance`，兼容子类与虚拟子类 |
| 10 | 在父类构造器里调用可被重写的钩子 | 子类字段可能尚未初始化 |

---

# 第 3 章 魔术方法（Python 对象模型的精髓）

> 魔术方法（dunder method，双下划线方法）是 Python 让**自定义对象**接入语言内置语法的钩子：`print`、`==`、`[]`、`for`、`with`、`+`、`()` 背后都是它们在起作用。

## 3.0 一张总表

| 分类 | 魔术方法 | 触发方式 |
| --- | --- | --- |
| 构造/销毁 | `__new__` `__init__` `__del__` | 实例化 / 回收 |
| 字符串表示 | `__repr__` `__str__` `__format__` `__bytes__` | `repr()` `str()` `f-string` `bytes()` |
| 比较 | `__eq__` `__ne__` `__lt__` `__le__` `__gt__` `__ge__` `__hash__` | `==` `!=` `<` `hash()` |
| 容器 | `__len__` `__getitem__` `__setitem__` `__delitem__` `__contains__` `__iter__` `__next__` `__reversed__` | `len()` `[]` `in` `for` |
| 算术 | `__add__` `__sub__` `__mul__` `__truediv__` `__floordiv__` `__mod__` `__pow__` 及 `__r*__`、`__i*__` | `+ - * / // % **` |
| 一元 | `__neg__` `__pos__` `__abs__` `__invert__` | `-x` `+x` `abs(x)` `~x` |
| 转换 | `__bool__` `__int__` `__float__` `__index__` | `bool()` `int()` `float()` `x[obj]` |
| 可调用 | `__call__` | `obj()` |
| 上下文 | `__enter__` `__exit__` | `with obj:` |
| 属性访问 | `__getattr__` `__getattribute__` `__setattr__` `__delattr__` | `obj.x` / `obj.x = v` |
| 拷贝 | `__copy__` `__deepcopy__` | `copy.copy()` `copy.deepcopy()` |

> ⚠️ **第一铁律：魔术方法是"隐式查找"的——解释器直接到 `type(obj)` 上找，不经过实例 `__dict__`。**

```python
class Bad:
    pass


b = Bad()
b.__len__ = lambda: 3               # 挂在实例上，没用
try:
    print(len(b))
except TypeError as e:
    print("TypeError:", e)          # 输出: TypeError: object of type 'Bad' has no len()
```

---

## 3.1 对象表示：`__str__`、`__repr__`、`__format__`

```python
class Money:
    def __init__(self, amount):
        self.amount = amount

    def __repr__(self):                     # 给开发者看：无歧义、最好能 eval
        return f"Money({self.amount!r})"

    def __str__(self):                      # 给用户看：可读
        return f"￥{self.amount}"


m = Money(9.9)
print(m)                # 输出: ￥9.9          ← print 用 __str__
print(str(m))           # 输出: ￥9.9
print(repr(m))          # 输出: Money(9.9)
print([m])              # 输出: [Money(9.9)]   ← 容器用 __repr__
print(f"{m}")           # 输出: ￥9.9          ← f-string 用 __format__（默认走 __str__）
print(f"{m!r}")         # 输出: Money(9.9)
print("%s | %r" % (m, m))  # 输出: ￥9.9 | Money(9.9)
```

| 对比项 | `__str__` | `__repr__` |
| --- | --- | --- |
| 目标读者 | 最终用户 | 开发者 / 调试 |
| 目标 | 可读 | 精确、无歧义，理想情况 `eval(repr(x)) == x` |
| 默认实现 | `object.__str__` 会退化为调用 `__repr__` | `<类名 object at 0x地址>` |
| 用于 | `print`、`str()`、`f"{x}"`、`"{}".format(x)` | `repr()`、容器显示、交互式回显、日志调试 |
| 建议 | 可以只写 `__repr__` | **至少写 `__repr__`**（`__str__` 可省略） |

### 自定义 `__format__`

```python
class Temperature:
    def __init__(self, celsius):
        self.celsius = celsius

    def __str__(self):
        return f"{self.celsius}°C"

    def __format__(self, spec):
        if spec == "F":
            return f"{self.celsius * 9 / 5 + 32:.1f}°F"
        return format(str(self), spec)      # 其它格式串退化为对字符串格式化


print(f"{Temperature(100)}")        # 输出: 100°C
print(f"{Temperature(100):F}")      # 输出: 212.0°F
print(f"[{Temperature(100):>10}]")  # 输出: [     100°C]
```

> 🔄 **举一反三**：`datetime`、`Decimal`、`pathlib.Path` 都实现了 `__format__`；写库时如果对象有"多种展示格式"，就实现它，而不是加一堆 `to_xxx()` 方法。

---

## 3.2 比较与哈希：`__eq__`、`__hash__`、`__lt__`

### 值相等：重写 `__eq__`

```python
class Point:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __eq__(self, other):
        if not isinstance(other, Point):
            return NotImplemented          # 不是"不相等"，而是"我不负责比较"
        return (self.x, self.y) == (other.x, other.y)

    def __hash__(self):
        return hash((self.x, self.y))

    def __repr__(self):
        return f"Point({self.x}, {self.y})"


p1, p2 = Point(1, 2), Point(1, 2)
print(p1 == p2)                 # 输出: True
print(p1 is p2)                 # 输出: False
print(p1 == (1, 2))             # 输出: False   ← 双方都返回 NotImplemented，最终退化为 is
print(len({p1, p2}))            # 输出: 1       ← 相等且哈希相同，集合去重
print({p1: "A"}[p2])            # 输出: A       ← 可以作字典键
```

### `__eq__` 与 `__hash__` 的强制联动

| 你定义了什么 | `__hash__` 会怎样 |
| --- | --- |
| 什么都没定义 | 继承 `object.__hash__`（按 `id` 计算），相等即同一对象 |
| 只定义 `__eq__` | **`__hash__` 被自动置为 `None`** → 实例不可哈希 |
| 定义 `__eq__` + `__hash__` | 正常，但必须保证"相等则哈希相同" |
| 只定义 `__hash__` | 可以，但相等语义没变（仍是身份比较） |
| `@dataclass(frozen=True)` | 自动生成 `__eq__` + `__hash__` |
| `@dataclass`（默认 `eq=True`） | 生成 `__eq__`，`__hash__` 为 `None` |

```python
class MutablePoint:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __eq__(self, other):
        return isinstance(other, MutablePoint) and (self.x, self.y) == (other.x, other.y)


print(MutablePoint.__hash__ is None)    # 输出: True
try:
    hash(MutablePoint(1, 2))
except TypeError as e:
    print("TypeError:", e)              # 输出: TypeError: unhashable type: 'MutablePoint'
```

**不可变对象才应该可哈希**：如果对象放进 `set`/`dict` 后还能改字段，哈希值会变，容器就"找不到"它了。

```python
class MutablePoint:
    def __init__(self, x): self.x = x
    def __eq__(self, o): return isinstance(o, MutablePoint) and self.x == o.x
    def __hash__(self): return hash(self.x)

p = MutablePoint(1)
s = {p}
p.x = 2
print(p in s)      # 输出: False  ← 哈希变了，集合里那个"旧哈希"的位置找不到了
```

### `NotImplemented` vs `NotImplementedError`

| 名称 | 用途 | 用法 |
| --- | --- | --- |
| `NotImplemented` | **值**（单例），表示"这个类型我不处理，请对方或解释器回退" | `return NotImplemented` |
| `NotImplementedError` | **异常**，表示"这个抽象方法子类必须实现" | `raise NotImplementedError` |

### 排序：`total_ordering` 省事

```python
from functools import total_ordering


@total_ordering
class Version:
    def __init__(self, major, minor):
        self.major, self.minor = major, minor

    def __eq__(self, other):
        if not isinstance(other, Version):
            return NotImplemented
        return (self.major, self.minor) == (other.major, other.minor)

    def __lt__(self, other):            # 只需实现 __eq__ + 一个比较
        if not isinstance(other, Version):
            return NotImplemented
        return (self.major, self.minor) < (other.major, other.minor)

    def __repr__(self):
        return f"Version({self.major}, {self.minor})"


print(sorted([Version(1, 10), Version(1, 2), Version(2, 0)]))
# 输出: [Version(1, 2), Version(1, 10), Version(2, 0)]
print(Version(2, 0) >= Version(1, 10))     # 输出: True（total_ordering 自动补全）
```

> 💡 **注意**：`__ne__` 默认由 `__eq__` 取反，不用自己写；`__lt__` 等返回 `NotImplemented` 时，Python 会尝试反向比较（`other.__gt__(self)`）。

---

## 3.3 容器协议：让对象支持 `len`、`[]`、`in`、`for`

```python
class Playlist:
    def __init__(self, songs):
        self._songs = list(songs)

    def __len__(self):                      # len(p)
        return len(self._songs)

    def __getitem__(self, index):           # p[i]、p[1:3]
        return self._songs[index]

    def __setitem__(self, index, value):    # p[i] = x
        self._songs[index] = value

    def __delitem__(self, index):           # del p[i]
        del self._songs[index]

    def __contains__(self, item):           # x in p
        return item in self._songs

    def __iter__(self):                     # for x in p
        return iter(self._songs)

    def __reversed__(self):                 # reversed(p)
        return reversed(self._songs)

    def __repr__(self):
        return f"Playlist({self._songs!r})"


p = Playlist(["晴天", "稻香", "七里香"])
print(len(p))                       # 输出: 3
print(p[1], p[-1])                  # 输出: 稻香 七里香
print(p[0:2])                       # 输出: ['晴天', '稻香']
p[0] = "夜曲"
print("夜曲" in p, "晴天" in p)      # 输出: True False
print([s for s in p])               # 输出: ['夜曲', '稻香', '七里香']
print(list(reversed(p)))            # 输出: ['七里香', '稻香', '夜曲']
del p[0]
print(p)                            # 输出: Playlist(['稻香', '七里香'])
```

### 只实现 `__getitem__` 也能迭代（旧协议）

当类没有 `__iter__` 时，`for` 会退化为"从 0 开始不断调用 `__getitem__(i)`，直到 `IndexError`"：

```python
class Squares:
    def __getitem__(self, i):
        if i >= 5:
            raise IndexError
        return i * i


print(list(Squares()))     # 输出: [0, 1, 4, 9, 16]
print(9 in Squares())      # 输出: True     ← in 没有 __contains__ 时也走这条件协议
```

> ⚠️ 这个旧协议**只对整数索引**生效，且会无限尝试下去（必须抛 `IndexError` 结束），性能也差。现代代码请实现 `__iter__`。

### `__len__` 的硬性要求

```python
class Weird:
    def __len__(self):
        return True          # bool 是 int 的子类，但不行


try:
    len(Weird())
except TypeError as e:
    print("TypeError:", e)   # 输出: TypeError: 'bool' object cannot be interpreted as an integer
```

`__len__` 必须返回**非负整数**，否则 `ValueError: __len__() should return >= 0`。

### 迭代器协议：`__iter__` + `__next__`

```python
class Countdown:
    def __init__(self, start):
        self.current = start

    def __iter__(self):
        return self                      # 自己是迭代器

    def __next__(self):
        if self.current <= 0:
            raise StopIteration          # 结束信号
        self.current -= 1
        return self.current + 1


print(list(Countdown(3)))    # 输出: [3, 2, 1]
```

| 概念 | 需要实现 | 特点 |
| --- | --- | --- |
| 可迭代对象 Iterable | `__iter__` | 可被 `for`、`list()` 使用，可重复迭代 |
| 迭代器 Iterator | `__iter__` + `__next__` | 一次性，耗尽后 `StopIteration` |

> 💡 **坑**：如果 `__iter__` 返回 `self`（如上例），这个对象就是"一次性"的，第二次 `for` 什么也拿不到。可迭代对象应返回一个**新的迭代器**。

---

## 3.4 运算符重载

```python
class Vector:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __add__(self, other):                   # v + w
        if not isinstance(other, Vector):
            return NotImplemented
        return Vector(self.x + other.x, self.y + other.y)

    def __radd__(self, other):                  # 0 + v（sum() 的起点是 0）
        if other == 0:
            return self
        return NotImplemented

    def __mul__(self, k):                       # v * 2
        if not isinstance(k, (int, float)):
            return NotImplemented
        return Vector(self.x * k, self.y * k)

    __rmul__ = __mul__                          # 2 * v

    def __neg__(self):                          # -v
        return Vector(-self.x, -self.y)

    def __abs__(self):                          # abs(v)
        return (self.x ** 2 + self.y ** 2) ** 0.5

    def __eq__(self, other):
        return isinstance(other, Vector) and (self.x, self.y) == (other.x, other.y)

    def __repr__(self):
        return f"Vector({self.x}, {self.y})"


v, w = Vector(1, 2), Vector(3, 4)
print(v + w)            # 输出: Vector(4, 6)
print(v * 2, 2 * v)     # 输出: Vector(2, 4) Vector(2, 4)
print(-v)               # 输出: Vector(-1, -2)
print(abs(Vector(3, 4)))            # 输出: 5.0
print(sum([v, w]))                  # 输出: Vector(4, 6)  ← 借助 __radd__ 处理起点 0
print(v + 1)                        # 报错: TypeError: unsupported operand type(s) for +: 'Vector' and 'int'
```

### `__add__` / `__radd__` / `__iadd__` 的执行顺序

```
a + b
 ├─ 1. type(a).__add__(a, b)          # 若返回非 NotImplemented → 用它
 ├─ 2. type(b).__radd__(b, a)         # 反向
 └─ 3. 都失败 → TypeError

a += b
 ├─ 1. type(a).__iadd__(a, b)         # 原地修改，应返回 self
 ├─ 2. 没有 __iadd__ → 退化为 a = a + b（用 __add__，产生新对象）
 └─ 3. 都没有 → TypeError
```

```python
class Acc:
    def __init__(self, value):
        self.value = value

    def __iadd__(self, other):
        self.value += other
        return self                     # 必须返回对象本身

    def __repr__(self):
        return f"Acc({self.value})"


a = Acc(1)
b = a
a += 10                                 # 原地修改
print(a, b, a is b)                     # 输出: Acc(11) Acc(11) True


class ImmutableAcc:
    def __init__(self, value):
        self.value = value

    def __add__(self, other):
        return ImmutableAcc(self.value + other)

    def __repr__(self):
        return f"ImmutableAcc({self.value})"


x = ImmutableAcc(1)
y = x
x += 10                                 # 退化为 x = x + 10，产生新对象
print(x, y, x is y)                     # 输出: ImmutableAcc(11) ImmutableAcc(1) False
```

### 可变对象的 `+=` 陷阱

```python
t = ([1], 2)
try:
    t[0] += [2]        # 先把 [1] 原地 extend 成 [1, 2]，然后对元组赋值时失败
except TypeError as e:
    print("TypeError:", e)     # 输出: TypeError: 'tuple' object does not support item assignment
print(t)                       # 输出: ([1, 2], 2)  ← 元组"没变"，但里面的列表被改了
```

> 🔄 **举一反三**：为什么 `[1] + [2]` 不修改原列表，而 `l += [2]` 会？—— 列表实现了 `__iadd__`（等价 `extend`），`+` 用 `__add__` 返回新列表。

---

## 3.5 `__call__`：让实例像函数一样调用

```python
class Multiplier:
    def __init__(self, k):
        self.k = k

    def __call__(self, x):
        return self.k * x


double = Multiplier(2)
print(double(21))               # 输出: 42
print(callable(double))         # 输出: True
```

### 实战：用类写装饰器（需要保存状态）

```python
class CountCalls:
    def __init__(self, func):
        self.func = func
        self.calls = 0

    def __call__(self, *args, **kwargs):
        self.calls += 1
        return self.func(*args, **kwargs)


@CountCalls
def add(a, b):
    return a + b


print(add(1, 2), add(3, 4), add.calls)   # 输出: 3 7 2
```

> 💡 与闭包装饰器相比，类装饰器**状态存在实例属性里**，更直观；记得用 `functools.wraps(self.func)` 保留元数据。

---

## 3.6 上下文管理器：`with` 语句

```python
class IgnoreErrors:
    def __init__(self, verbose=True):
        self.verbose = verbose

    def __enter__(self):
        print("进入上下文")
        return self                     # as 拿到的对象

    def __exit__(self, exc_type, exc_value, traceback):
        if exc_type is not None:
            if self.verbose:
                print(f"捕获 {exc_type.__name__}: {exc_value}，已忽略")
            return True                 # True = 吞掉异常；False/None = 继续抛出
        print("正常退出")
        return False


with IgnoreErrors() as ctx:
    print("执行中")
    raise ValueError("出错了")

print("with 之后的代码继续执行")
# 输出:
# 进入上下文
# 执行中
# 捕获 ValueError: 出错了，已忽略
# with 之后的代码继续执行
```

### `__exit__` 的返回值决定异常命运

| `__exit__` 返回 | 异常已发生 | 行为 |
| --- | --- | --- |
| `True` | 是 | **吞掉**异常，`with` 之后的代码继续执行 |
| `False` / `None` | 是 | 异常继续向上传播 |
| 任意 | 否 | 无影响 |

### 用 `@contextmanager` 更简洁

```python
from contextlib import contextmanager


@contextmanager
def tag(name):
    print(f"<{name}>")
    try:
        yield name
    finally:
        print(f"</{name}>")             # 无论是否异常都执行


with tag("div") as t:
    print("内容:", t)
# 输出:
# <div>
# 内容: div
# </div>
```

> 💡 **对比**：`try/finally` 管"一定要执行的清理"；上下文管理器管"成对的资源获取与释放"（文件、锁、数据库事务、临时目录、计时器）。标准库 `contextlib.suppress`、`redirect_stdout`、`closing` 都是现成工具。

---

## 3.7 属性访问钩子：`__getattr__` / `__getattribute__` / `__setattr__`

### `__getattr__`：兜底（只在正常查找失败时调用）

```python
class LazyConfig:
    def __getattr__(self, name):
        if name.startswith("_"):
            raise AttributeError(name)      # 保护内部属性，避免递归
        print(f"懒加载 {name}")
        value = name.upper()
        setattr(self, name, value)          # 写入实例字典，下一次直接命中
        return value


c = LazyConfig()
print(c.host)            # 输出: 懒加载 host
                         #       HOST
print(c.host)            # 输出: HOST          （已写入实例 __dict__，不再触发）
print(hasattr(c, "nope2"))   # 输出: 懒加载 nope2
                             #       True
```

### `__getattribute__`：拦截**所有**属性访问（危险，慎用）

```python
class Secret:
    def __getattribute__(self, name):
        if name == "password":
            return "******"
        return object.__getattribute__(self, name)   # 必须显式走 object，否则无限递归


s = Secret()
s.password = "123456"
print(s.password)        # 输出: ******
print(s.__class__.__name__)   # 输出: Secret
```

### `__setattr__`：拦截所有赋值

```python
class Frozen:
    ALLOWED = {"x", "y"}

    def __setattr__(self, name, value):
        if name not in Frozen.ALLOWED:
            raise AttributeError(f"不允许设置属性 {name}")
        object.__setattr__(self, name, value)     # 绕过自己，避免递归


f = Frozen()
f.x = 1
print(f.x)               # 输出: 1
try:
    f.z = 2
except AttributeError as e:
    print("AttributeError:", e)   # 输出: AttributeError: 不允许设置属性 z
```

### 三者的区别

| 钩子 | 调用时机 | 典型用途 |
| --- | --- | --- |
| `__getattr__` | **正常查找失败后**才调用 | 懒加载、代理、兼容旧属性名 |
| `__getattribute__` | **每次** `obj.attr` 都调用 | 权限控制、日志（易死循环，慎用） |
| `__setattr__` | **每次** `obj.attr = v` 都调用 | 只读对象、校验、自动通知 |

> ⚠️ 在 `__getattribute__` / `__setattr__` 内部一定要用 `object.__getattribute__(self, name)` / `object.__setattr__(self, name, value)`（或 `super()`），否则会无限递归。

---

## 3.8 布尔、数值转换

```python
class Box:
    def __init__(self, items):
        self.items = list(items)

    def __len__(self):
        return len(self.items)

    def __bool__(self):
        return bool(self.items)          # 优先级高于 __len__


print(bool(Box([])), bool(Box([1])))     # 输出: False True
```

| 场景 | 优先级 |
| --- | --- |
| `bool(x)` | `__bool__` → `__len__` → 默认 `True` |
| `int(x)` | `__int__` → `__index__` → `__trunc__` |
| 切片 / `bin()` / `hex()` 参数 | 需要 `__index__`（必须返回 int） |
| `float(x)` | `__float__` |

> 💡 实现 `__index__` 而不是 `__int__`，才能让对象用在 `range()`、列表下标、`bin()` 等**要求真整数**的位置。

---

## 3.9 生命周期：`__del__` 与引用计数

```python
import sys


class Resource:
    def __init__(self, name):
        self.name = name

    def __del__(self):
        print(f"释放 {self.name}")


r = Resource("文件句柄")
print(sys.getrefcount(r))    # 输出: 2（r + getrefcount 的临时参数）
del r                        # 输出: 释放 文件句柄
```

**必须知道的限制**

1. `__del__` **不是** C++ 的析构函数，不保证及时、不保证一定被调用（解释器退出、循环引用）。
2. 不要依赖 `__del__` 释放关键资源 → 用 `with` / `try-finally` / `contextlib.closing`。
3. 循环引用中的对象即使有 `__del__`，现代 CPython（3.4+）也能回收，但顺序不确定。
4. 在 `__del__` 里抛异常会被忽略（打印到 stderr），不要写复杂逻辑。

---

## 3.10 拷贝：`__copy__` 与 `__deepcopy__`

```python
import copy


class Node:
    def __init__(self, value, children=None):
        self.value = value
        self.children = children or []


n = Node(1, [Node(2)])
shallow = copy.copy(n)
deep = copy.deepcopy(n)

print(shallow is n)                     # 输出: False
print(shallow.children is n.children)   # 输出: True   ← 浅拷贝：内层共享
print(deep.children is n.children)      # 输出: False  ← 深拷贝：完全独立
print(deep.children[0] is n.children[0])# 输出: False
```

自定义拷贝行为：

```python
class Cached:
    def __init__(self, data):
        self.data = data
        self._cache = {}

    def __copy__(self):
        new = type(self)(self.data)     # 故意不复制缓存
        return new

    def __deepcopy__(self, memo):
        import copy as _copy
        new = type(self)(_copy.deepcopy(self.data, memo))
        memo[id(self)] = new            # 登记到 memo，防止循环引用
        return new
```

---

## 3.11 第 3 章易错清单

| # | 易错点 | 正确认知 |
| --- | --- | --- |
| 1 | 以为在实例上挂 `__len__` 就能用 | 魔术方法只在**类型**上查找 |
| 2 | 只写 `__str__` 不写 `__repr__` | 容器/调试时显示难看的默认地址 |
| 3 | 定义 `__eq__` 后忘了 `__hash__` | 实例变不可哈希，不能进 `set`/做字典键 |
| 4 | `__eq__` 里直接 `return False` 处理异类 | 应返回 `NotImplemented`，让反向比较有机会 |
| 5 | 把 `NotImplemented` 与 `NotImplementedError` 混用 | 前者是值，后者是异常 |
| 6 | `__iadd__` 不返回 `self` | `a += b` 后 `a` 变成 `None` |
| 7 | `__exit__` 返回 `True` 想"不吞异常" | 恰好相反：`True` 才吞 |
| 8 | `__getattribute__` 里写 `self.x` | 无限递归，必须走 `object.__getattribute__` |
| 9 | 在 `__del__` 里释放关键资源 | 不可靠，用 `with` |
| 10 | 浅拷贝后修改嵌套对象 | 内层仍是共享的，需要 `deepcopy` |
| 11 | `__iter__` 返回 `self` 却希望可重复迭代 | 迭代器是一次性的 |
| 12 | `__len__` 返回 `bool` 或负数 | 必须是 `>= 0` 的 `int` |

---

# 第 4 章 进阶：生命周期、元类、dataclass 与设计模式

## 4.1 对象生命周期与垃圾回收

### 引用计数是主力，循环 GC 是补丁

| 机制 | 作用 | 触发时机 |
| --- | --- | --- |
| **引用计数** | 每个对象记录被引用次数，归零立即释放 | 实时 |
| **循环垃圾回收（gc）** | 找出"互相引用但整体已不可达"的环 | 阈值触发 / `gc.collect()` |
| `__del__` | 对象被回收前的钩子 | 不可靠，别依赖 |

```python
import gc
import weakref


class Node:
    def __init__(self, name):
        self.name = name
        self.peer = None

    def __repr__(self):
        return f"Node({self.name})"


a, b = Node("a"), Node("b")
a.peer, b.peer = b, a               # 循环引用：引用计数永远不为 0
ref_a = weakref.ref(a)              # 弱引用：不增加引用计数

del a, b
print("删除变量后:", ref_a())        # 输出: 删除变量后: Node(a)
print("GC 回收对象数:", gc.collect() > 0)   # 输出: GC 回收对象数: True
print("回收后:", ref_a())            # 输出: 回收后: None
```

### 弱引用（`weakref`）的用途

- 缓存：不希望缓存把对象"钉死"在内存里（`WeakValueDictionary`）。
- 观察者模式：订阅者被销毁后自动从列表消失（`WeakSet`）。
- 父子引用防环：父 → 子强引用，子 → 父弱引用。

```python
import weakref


class Subscriber:
    pass


s = Subscriber()
ref = weakref.ref(s)
print(ref() is s)       # 输出: True
del s
print(ref())            # 输出: None
```

> ⚠️ 有 `__slots__` 的类默认不支持弱引用，需要把 `"__weakref__"` 加入 `__slots__`。

---

## 4.2 元类、类装饰器与 `__init_subclass__`

### 类本身也是对象

```python
class Foo:
    x = 1


print(type(Foo))                # 输出: <class 'type'>
print(isinstance(Foo, type))    # 输出: True

# 用 type 动态创建类：type(类名, 父类元组, 命名空间字典)
Bar = type("Bar", (object,), {"x": 2, "hello": lambda self: "hi"})
print(Bar.__name__, Bar.x, Bar().hello())   # 输出: Bar 2 hi

# 等价写法
class Bar2:
    x = 2
    def hello(self):
        return "hi"
```

### 类创建的 6 个步骤

```
1. 解析基类，确定元类（metaclass）
2. 元类的 __prepare__ 返回"类命名空间"（通常是个 dict）
3. 执行类体代码，把 def/赋值塞进命名空间
4. 元类/type 的 __new__ 创建类对象 —— 此时触发所有描述符的 __set_name__
5. 调用父类的 __init_subclass__（子类注册钩子）
6. 应用类装饰器（如果有）
```

### 方式一：`__init_subclass__`（最简单，3.6+）

```python
class Plugin:
    registry = []

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        Plugin.registry.append(cls.__name__)


class ImagePlugin(Plugin):
    pass


class VideoPlugin(Plugin):
    pass


print(Plugin.registry)      # 输出: ['ImagePlugin', 'VideoPlugin']
```

> 💡 `__init_subclass__` 是隐式类方法，不需要 `@classmethod`；它是"注册子类"最轻量的钩子，Django、pytest 插件系统等都在用类似机制。

### 方式二：类装饰器

```python
def add_repr(cls):
    def __repr__(self):
        attrs = ", ".join(f"{k}={v!r}" for k, v in vars(self).items())
        return f"{cls.__name__}({attrs})"

    cls.__repr__ = __repr__
    return cls


@add_repr
class User:
    def __init__(self, name, age):
        self.name = name
        self.age = age


print(User("小明", 18))     # 输出: User(name='小明', age=18)
```

### 方式三：元类（最强，也最重）

元类是"创建类的类"，可以拦截类的创建、修改类属性、控制实例化。

```python
class Meta(type):
    def __new__(mcls, name, bases, namespace):
        print(f"正在创建类: {name}")
        return super().__new__(mcls, name, bases, namespace)


class Foo(metaclass=Meta):
    pass


class Bar(Foo):                     # 子类自动继承元类
    pass
# 输出:
# 正在创建类: Foo
# 正在创建类: Bar
```

### 元类的经典应用：单例（天然支持继承）

```python
class SingletonMeta(type):
    _instances = {}

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]


class Config(metaclass=SingletonMeta):
    pass


class Database(metaclass=SingletonMeta):
    pass


print(Config() is Config())          # 输出: True
print(Config() is Database())        # 输出: False  ← 每个类各自一个实例
```

### 元类的经典应用：强制类属性

```python
class RequireDoc(type):
    def __new__(mcls, name, bases, namespace):
        if not namespace.get("__doc__"):
            raise TypeError(f"类 {name} 必须写文档字符串")
        return super().__new__(mcls, name, bases, namespace)


class Good(metaclass=RequireDoc):
    """有文档。"""


try:
    class Bad(metaclass=RequireDoc):
        pass
except TypeError as e:
    print("TypeError:", e)          # 输出: TypeError: 类 Bad 必须写文档字符串
```

### 元类冲突

```python
class MetaA(type):
    pass


class MetaB(type):
    pass


class A(metaclass=MetaA):
    pass


try:
    class B(A, metaclass=MetaB):
        pass
except TypeError as e:
    print("TypeError:", e)
# 输出: TypeError: metaclass conflict: the metaclass of a derived class must be a (non-strict) subclass of the metaclasses of all its bases
```

### 三者怎么选

| 需求 | 首选 |
| --- | --- |
| 子类创建时做登记 / 校验参数 | `__init_subclass__` |
| 修改类的属性、给类加方法、包装类 | 类装饰器 |
| 控制"类如何被创建/实例化"、框架级协议 | 元类 |

> 💡 **经验法则**：**能用 `__init_subclass__` 就不用元类；能用类装饰器就不用元类**。元类一旦被继承会污染整个类族，且与 `abc.ABCMeta` 等容易冲突（需继承 `ABCMeta`）。

---

## 4.3 `dataclass`、`NamedTuple`、`Enum`、`TypedDict`

### `@dataclass`：自动生成样板代码

```python
from dataclasses import dataclass, field, asdict


@dataclass
class Student:
    name: str
    age: int = 18
    scores: list = field(default_factory=list)   # 可变默认值必须用 factory

    def __post_init__(self):                     # 初始化后钩子：适合校验/派生字段
        if self.age < 0:
            raise ValueError("年龄不能为负")
        self.scores = list(self.scores)


s = Student("小明", scores=[90])
print(s)                                       # 输出: Student(name='小明', age=18, scores=[90])
print(s == Student("小明", 18, [90]))           # 输出: True
print(asdict(s))                               # 输出: {'name': '小明', 'age': 18, 'scores': [90]}
```

`@dataclass` 自动生成：`__init__`、`__repr__`、`__eq__`。常用参数：

| 参数 | 作用 |
| --- | --- |
| `frozen=True` | 不可变；自动生成 `__hash__`，可做字典键 |
| `order=True` | 生成 `< <= > >=`（按字段顺序比较） |
| `slots=True` | 3.10+，自动生成 `__slots__` |
| `kw_only=True` | 3.10+，所有字段只能关键字传参（解决继承默认值顺序问题） |
| `eq=False` | 不生成 `__eq__`，保留身份比较 |
| `repr=False` | 不生成 `__repr__`（适合含密码等敏感字段） |

```python
@dataclass(frozen=True)
class Point:
    x: int
    y: int


p = Point(1, 2)
print(hash(p) == hash(Point(1, 2)))    # 输出: True
try:
    p.x = 10
except Exception as e:
    print(type(e).__name__)            # 输出: FrozenInstanceError
```

**继承时的坑**：父类有默认值字段时，子类不能添加无默认值字段：

```python
@dataclass
class Base:
    x: int = 0


try:
    @dataclass
    class Child(Base):
        y: int                         # 无默认值字段排在默认值字段之后
except TypeError as e:
    print("TypeError:", e)
# 输出: TypeError: non-default argument 'y' follows default argument 'x'

# 解决：给 y 也加默认值，或使用 kw_only=True
@dataclass(kw_only=True)
class Child2(Base):
    y: int
```

### `NamedTuple`：不可变的轻量记录

```python
from typing import NamedTuple


class Point(NamedTuple):
    x: int
    y: int


p = Point(1, 2)
print(p, p.x, p[0])             # 输出: Point(x=1, y=2) 1 1
print(p._replace(x=10))         # 输出: Point(x=10, y=2)
print(p._asdict())              # 输出: {'x': 1, 'y': 2}
print(isinstance(p, tuple))     # 输出: True
```

### `Enum`：具名常量

```python
from enum import Enum, IntEnum, auto, unique


class Color(Enum):
    RED = 1
    GREEN = 2
    BLUE = 3


print(Color.RED)                    # 输出: Color.RED
print(Color.RED.value)              # 输出: 1
print(Color(1))                     # 输出: Color.RED
print([c.name for c in Color])      # 输出: ['RED', 'GREEN', 'BLUE']
print(Color.RED == 1)               # 输出: False（普通 Enum 不等于值）
print(Color.RED is Color(1))        # 输出: True （成员是单例）


class Status(Enum):
    OK = 200
    SUCCESS = 200                   # 别名：同值即同一个成员


print(Status.SUCCESS is Status.OK)  # 输出: True
print(list(Status))                 # 输出: [<Status.OK: 200>]   ← 迭代不含别名
print(list(Status.__members__))     # 输出: ['OK', 'SUCCESS']     ← 包含别名


class Code(IntEnum):
    OK = 200
    NOT_FOUND = 404


print(Code.OK == 200)               # 输出: True   ← IntEnum 可与整数比较
print(Code.OK + 1)                  # 输出: 201


class Auto(Enum):
    A = auto()
    B = auto()


print(Auto.A.value, Auto.B.value)   # 输出: 1 2
```

### `TypedDict`：给字典加"类型说明"

```python
from typing import TypedDict


class Movie(TypedDict):
    title: str
    year: int


m: Movie = {"title": "流浪地球", "year": 2019}
print(m["title"])           # 输出: 流浪地球
# 注意：TypedDict 只是给类型检查器看的，运行时仍是普通 dict，不做校验
```

### 五种"数据类"对比

| 特性 | 普通类 | `@dataclass` | `NamedTuple` | `Enum` | `TypedDict` |
| --- | --- | --- | --- | --- | --- |
| 自动 `__init__` | ❌ | ✅ | ✅ | ❌ | ❌ |
| 自动 `__repr__` | ❌ | ✅ | ✅ | ✅（`Color.RED`） | ❌ |
| 自动 `__eq__` | ❌ | ✅ | ✅（元组比较） | ✅（身份） | ❌ |
| 可变 | ✅ | ✅（可 `frozen`） | ❌ 不可变 | ❌ 不可变 | ✅ |
| 可哈希 | 默认按 id | 默认不可哈希；frozen 可 | ✅ | ✅ | — |
| 是否元组 | ❌ | ❌ | ✅（可解包、可索引） | ❌ | ❌ |
| 运行时校验 | 自定义 | `__post_init__` | 类型提示不校验 | 值校验 | ❌ 不校验 |
| 适合 | 行为丰富的对象 | 数据载体（DTO） | 轻量不可变记录 | 有限常量集合 | 字典结构标注 |

---

## 4.4 常用设计模式（Pythonic 实现）

> Python 的设计模式往往比 Java 简洁：一等函数、模块、鸭子类型可以省掉大量"模式类"。

### 单例（Singleton）

```python
# 方式 1：模块级实例（最 Pythonic）
# config.py
# class _Config: ...
# config = _Config()

# 方式 2：装饰器
def singleton(cls):
    instances = {}

    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return get_instance


@singleton
class AppConfig:
    def __init__(self):
        self.debug = False


print(AppConfig() is AppConfig())    # 输出: True
```

### 工厂（Factory）：把 `if/elif` 关进工厂里

```python
class Alipay:
    def pay(self, amount):
        return f"支付宝支付 {amount} 元"


class Wechat:
    def pay(self, amount):
        return f"微信支付 {amount} 元"


class PaymentFactory:
    _methods = {"alipay": Alipay, "wechat": Wechat}

    @classmethod
    def create(cls, name):
        try:
            return cls._methods[name]()
        except KeyError:
            raise ValueError(f"不支持的支付方式: {name}") from None


print(PaymentFactory.create("alipay").pay(100))    # 输出: 支付宝支付 100 元
try:
    PaymentFactory.create("paypal")
except ValueError as e:
    print("ValueError:", e)                         # 输出: ValueError: 不支持的支付方式: paypal
```

> 🔄 **举一反三**：工厂的注册表也可以是 `__init_subclass__` 自动收集的（插件化），此时新增支付方式**不需要改工厂代码**（开闭原则）。

### 策略（Strategy）：把算法变成可替换对象

```python
def normal(price):
    return price


def discount(price):
    return price * 0.9


class Order:
    def __init__(self, price, strategy=normal):
        self.price = price
        self.strategy = strategy          # 依赖注入

    def total(self):
        return self.strategy(self.price)


print(Order(100).total())                 # 输出: 100
print(Order(100, discount).total())       # 输出: 90.0
```

### 观察者（Observer）：一对多通知

```python
class Subject:
    def __init__(self):
        self._observers = []

    def subscribe(self, callback):
        self._observers.append(callback)

    def notify(self, event):
        for callback in self._observers:
            callback(event)


log = []
subject = Subject()
subject.subscribe(lambda e: log.append(f"邮件:{e}"))
subject.subscribe(lambda e: log.append(f"短信:{e}"))
subject.notify("订单已支付")
print(log)      # 输出: ['邮件:订单已支付', '短信:订单已支付']
```

### 模板方法（Template Method）：父类定骨架，子类填步骤

```python
from abc import ABC, abstractmethod


class DataExporter(ABC):
    def export(self):                     # 模板方法：固定流程
        data = self.fetch()
        self.validate(data)
        return self.format(data)

    @abstractmethod
    def fetch(self):
        ...

    def validate(self, data):             # 可选覆盖的钩子
        if not data:
            raise ValueError("没有数据")

    @abstractmethod
    def format(self, data):
        ...


class CSVExporter(DataExporter):
    def fetch(self):
        return [("a", 1), ("b", 2)]

    def format(self, data):
        return "\n".join(f"{k},{v}" for k, v in data)


print(CSVExporter().export())
# 输出:
# a,1
# b,2
```

---

## 4.5 SOLID 五原则与常见坏味道

| 原则 | 含义 | 反例 | 正例 |
| --- | --- | --- | --- |
| **S** 单一职责 | 一个类只有一个变化原因 | `User` 类同时管数据库、发邮件、生成报表 | 拆成 `User`、`UserRepository`、`Mailer` |
| **O** 开闭原则 | 对扩展开放，对修改关闭 | `if type == "alipay": ... elif type == "wechat": ...` | 策略/工厂 + 注册表 |
| **L** 里氏替换 | 子类必须能替换父类且不破坏语义 | `Penguin(Bird)` 重写 `fly()` 抛异常 | 抽出 `Flyable` 接口，企鹅不实现 |
| **I** 接口隔离 | 接口要小而专 | 一个巨大的 `Animal` 抽象类要求实现 20 个方法 | 多个小 `Protocol` / Mixin |
| **D** 依赖倒置 | 依赖抽象，不依赖具体实现 | `Order` 内部 `self.pay = Alipay()` | 构造器注入 `Payment` 抽象 |

### 常见面向对象坏味道

| 坏味道 | 表现 | 修法 |
| --- | --- | --- |
| 上帝类 | 一个类几千行、几十个方法 | 按职责拆分 |
| 深继承树 | `A → B → C → D → E` | 用组合 + 少量 Mixin |
| `isinstance` 长链 | `if isinstance(x, A): ... elif isinstance(x, B): ...` | 多态 / 注册表 |
| 可变类属性当共享状态 | 所有实例串数据 | `__init__` 里初始化 |
| 在 `__init__` 里做 IO | 对象创建慢、难测试 | 依赖注入，单独 `load()` |
| 暴露所有属性 | 外部随意改内部状态 | `@property` / 只读 |
| 继承只为复用几行代码 | 子类被迫继承无关 API | 组合 |
| 用 `type(x) == T` 判断 | 子类失效 | `isinstance` |

---

## 4.6 第 4 章易错清单

| # | 易错点 | 正确认知 |
| --- | --- | --- |
| 1 | 依赖 `__del__` 释放资源 | 用 `with` / `try-finally` |
| 2 | 认为引用计数能解决循环引用 | 需要 gc；可用弱引用断环 |
| 3 | `@dataclass` 用可变对象做默认值 | 必须 `field(default_factory=list)` |
| 4 | `frozen=True` 却想改字段 | 会抛 `FrozenInstanceError` |
| 5 | dataclass 子类加无默认值字段 | 报错，用默认值或 `kw_only=True` |
| 6 | 把 `Enum` 成员当普通整数比较 | 普通 `Enum` 不等于值；需要比较用 `IntEnum` 或 `.value` |
| 7 | 元类冲突 | 子类元类必须是所有父类元类的子类 |
| 8 | 给 `NamedTuple` 加方法 | 可以加方法，但不能加新字段 |
| 9 | 以为 `TypedDict` 会运行时校验 | 只是类型标注 |
| 10 | 用元类做本可以类装饰器做的事 | 优先更轻的方案 |

---
