这段代码是 Python 生成器与迭代器嵌套的经典考题，我们一步一步拆开看。

---

## 一、代码输出是什么？

```python
def func():
    yield (x for x in range(3))

for x in func():
    print(x)
```

**输出结果：**
```
<generator object func.<locals>.<genexpr> at 0x...>
```

（地址每次运行不同）

👉 **打印的不是 0 1 2，而是一个生成器对象。**

---

## 二、为什么会这样？（核心分析）

### 1️⃣ `func()` 是一个生成器函数
```python
def func():
    yield (x for x in range(3))
```

- `yield` 的存在 → `func()` 是**生成器函数**
- 调用 `func()` 时，**不会执行函数体**，而是返回一个生成器对象
- 每次迭代 `func()`，才执行一次 `yield`

### 2️⃣ `yield` 后面是一个生成器表达式
```python
(x for x in range(3))
```

这是一个**生成器表达式**，返回的是：
> 一个 generator object，而不是 list

### 3️⃣ `for x in func()` 发生了什么？
```python
for x in func():
    print(x)
```

- `func()` 返回一个生成器
- `for` 循环只迭代一次
- 第一次 `next(func())`：
  - 执行 `yield (x for x in range(3))`
  - `yield` 产出的是**生成器表达式本身**
- `x` 就是这个生成器对象
- `print(x)` → 打印生成器对象

✅ **所以只输出一行：生成器对象**

---

## 三、考察知识点（重点）

### ✅ 1. 生成器函数 vs 生成器表达式
| 类型 | 示例 | 返回 |
|----|----|----|
| 生成器函数 | `def f(): yield ...` | 生成器对象 |
| 生成器表达式 | `(x for x in range(3))` | 生成器对象 |

### ✅ 2. `yield` 产出的是“值”，不是执行结果
```python
yield expr
```
- `expr` 是什么，`yield` 就产出什么
- 这里 `expr` 是一个生成器表达式 → 产出生成器对象

### ✅ 3. `for` 迭代生成器函数时
- 每次 `next()` 执行到 `yield`
- `yield` 返回的值，就是 `for` 循环变量的值

### ✅ 4. 生成器是惰性求值的
```python
(x for x in range(3))
```
- 不会立即生成 `[0,1,2]`
- 只有在迭代它时才会逐个产出值

---

## 四、举一反三（常见变形题）

### 🔹 变形 1：想输出 0 1 2 怎么办？

#### ✅ 方法 1：在循环里再迭代一次
```python
def func():
    yield (x for x in range(3))

for gen in func():
    for x in gen:
        print(x)
```

输出：
```
0
1
2
```

---

#### ✅ 方法 2：直接 `yield from`
```python
def func():
    yield from (x for x in range(3))

for x in func():
    print(x)
```

输出：
```
0
1
2
```

✅ **这是面试/考试最爱考的改法**

---

### 🔹 变形 2：如果 `yield` 改成 `return` 呢？
```python
def func():
    return (x for x in range(3))

print(func())
```

输出：
```
<generator object func.<locals>.<genexpr> at 0x...>
```

- `return` 直接返回生成器表达式
- 函数不是生成器函数

---

### 🔹 变形 3：如果改成列表推导式呢？
```python
def func():
    yield [x for x in range(3)]

for x in func():
    print(x)
```

输出：
```
[0, 1, 2]
```

✅ 因为 `yield` 产出的是 list

---

### 🔹 变形 4：多层 `yield`
```python
def func():
    yield (i for i in range(3))

g = func()
gen = next(g)
print(list(gen))
```

输出：
```
[0, 1, 2]
```

---

## 五、一句话总结（面试版）

> `yield` 后面如果是生成器表达式，产出的是生成器对象本身，而不是其迭代结果；  
> 若想逐层展开，应使用 `yield from` 或在外部再迭代一次。

---

如果你愿意，我可以再帮你出 **3 道同难度生成器面试题**，让你一次吃透这个考点。