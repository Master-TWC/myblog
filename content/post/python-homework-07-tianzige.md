---
title: "Python 学习笔记：从「田字格绘制」看函数与代码复用的艺术"
description: "同一个田字格写三遍：把 7 行几何细节从循环体里抽出来之后，新增一种格子样式不再需要改调用处。"
date: 2026-11-24T21:30:00+08:00
slug: "python-homework-07-tianzige"
tags: ["Python", "Turtle", "重构", "学习笔记"]
categories: ["技术"]
---

# Python 学习笔记：从「田字格绘制」看函数与代码复用的艺术

> 这篇不是实验报告，是一篇学习笔记。
> 起因很简单：用 turtle 画一个田字格，画第二个格子时我复制粘贴了一遍，画到第九个时我意识到——
> **我复制粘贴的不是"代码"，是"没被命名的概念"。**

## 一、v1：先把图形画出来

田字格 = 一个方框 + 横竖两条中线。最直白的写法：

```python
for row in range(3):
    for col in range(3):
        x, y = x0 + col * size, y0 - row * size
        pen.penup(); pen.goto(x, y); pen.pendown()
        pen.goto(x + size, y)          # 上边
        pen.goto(x + size, y - size)   # 右边
        pen.goto(x, y - size)          # 下边
        pen.goto(x, y)                 # 左边
        pen.goto(x + size, y - size)   # 想从右下角起笔画横中线……
        pen.goto(x, y)                 # ……结果连成了对角线
```

能跑，但有两个问题。第一，循环体里塞了 **7 行几何细节**，意图被细节淹没了：读代码的人得先在脑子里把 7 次 `goto` 重放一遍，才能看出"哦，这是画一个格子"。

第二，图就是画歪的——**"方框"和"十字"共用同一支笔、中间没抬笔**，所以本该是横竖中线的两笔变成了对角线：

![v1 版本：细节直接写在循环里](/myblog/images/homework/07-tianzige/v1-naive.png)

*每个格子里那条斜线，就是"没抬笔"的证据*

## 二、v2：把"格子"命名成函数

只要给概念起个名字，两个函数就够了：

```python
def draw_square(pen, x, y, size):
    """画一个正方形框：把「抬笔到位 → 走四条边」封装起来。"""
    pen.penup(); pen.goto(x, y); pen.pendown()
    for _ in range(4):
        pen.forward(size)
        pen.right(90)

def draw_cross(pen, x, y, size):
    """画十字中线：田字格的「田」就靠这两笔。"""
    pen.penup(); pen.goto(x + size / 2, y); pen.pendown()
    pen.goto(x + size / 2, y - size)
    pen.penup(); pen.goto(x, y - size / 2); pen.pendown()
    pen.goto(x + size, y - size / 2)
```

循环体一下从 7 行变成 2 行：

```python
for row in range(3):
    for col in range(3):
        x, y = x0 + col * size, y0 - row * size
        draw_square(pen, x, y, size)
        draw_cross(pen, x, y, size)
```

![v2 版本：田字格 3×3](/myblog/images/homework/07-tianzige/tian-3x3.png)

*同样的 3×3 田字格，循环体只剩两次调用*

**注意这里发生了什么**：代码总行数其实**变多了**（多了两个 `def` 和注释）。抽函数从来不是"为了让代码变短"，而是**为了让读代码的人一次只面对一个层次**——看主程序时读到"画方框 + 画十字"，看函数时再关心几何细节。

## 三、v3：把"样式"变成参数

美术课不止田字格，还有米字格（加两条对角线）和回宫格（内嵌小方框）。如果按 v1 的思路，每加一种就得再抄 7 行；按 v2 的思路，再加一个函数；按 v3 的思路——**只加一个分支**：

```python
def draw_grid(pen, rows=3, cols=3, size=110, style="tian"):
    x0 = -(cols * size) / 2
    y0 = (rows * size) / 2
    for row in range(rows):
        for col in range(cols):
            x, y = x0 + col * size, y0 - row * size
            draw_square(pen, x, y, size)          # 复用的还是 v2 的函数
            if style == "hui":
                draw_square(pen, x + size * 0.22, y - size * 0.22, size * 0.56)
                continue
            draw_cross(pen, x, y, size)
            if style == "mi":
                pen.penup(); pen.goto(x, y); pen.pendown()
                pen.goto(x + size, y - size)
                pen.penup(); pen.goto(x + size, y); pen.pendown()
                pen.goto(x, y - size)
```

于是调用方变成了"声明意图"：

```python
draw_grid(pen, 3, 3, size=110, style="tian")   # 田字格
draw_grid(pen, 3, 3, size=110, style="mi")     # 米字格
draw_grid(pen, 2, 2, size=150, style="hui")    # 回宫格
```

![米字格（style="mi"）](/myblog/images/homework/07-tianzige/mi-3x3.png)

*同一个函数，只改了 style 参数：3×3 米字格*

## 四、三种写法的真实对比

![三种写法的复用成本对比](/myblog/images/homework/07-tianzige/reuse-compare.png)

*蓝：画一个格子循环体要写几行；橙：新增一种样式要改几行*

| 版本 | 画一个格子 | 新增一种样式 | 关键改动 |
| --- | --- | --- | --- |
| v1 复制粘贴 | 7 行（全写在循环体里） | 7 行（再抄一遍） | 四条边 + 两条中线 |
| v2 抽函数 | 2 行（两次调用） | 2 行（调两次别的函数） | `draw_square` / `draw_cross` |
| v3 加参数 | 2 行（一次调用） | 3 行（加一个 `if` 分支） | `style` 三种取值 |

**几何细节一行都没少**——7 行 `goto`/`forward` 依然在 `draw_square` 和 `draw_cross` 里。变的只有一件事：**同样的细节，你写几遍**。

## 五、三个能带走的判断

### 1. "写两遍"就该抽函数了吗？

我的判断标准不是次数，而是**这个概念会不会在别处被再次提起**。画格子只出现一次，`draw_square` 也值得抽——因为"方框"在后来的回宫格里又用了一次；而如果把"设置画笔颜色"单独抽一个函数，就没意义，因为它只有一行、也不会被复用。

> 规则：**抽的是概念，不是行数**。三行能说清一件事，就抽；一行说不出概念，就别抽。

### 2. 参数化到什么程度为止？

`draw_grid` 有 5 个参数（行列、大小、样式、还有隐式的画笔）。再加"圆角""虚线""背景色"，参数会爆炸。这时候正确的做法不是继续加参数，而是**换数据结构**：把样式做成配置字典或类。

```python
STYLES = {
    "tian": {"cross": True, "diagonal": False, "inner": False},
    "mi":   {"cross": True, "diagonal": True,  "inner": False},
    "hui":  {"cross": False, "diagonal": False, "inner": True},
}
```

参数超过 4~5 个，通常说明"这个函数在同时做几件事"。

### 3. 什么时候该停止抽象？

这个实验里我没把"画格子"做成类，也没做插件系统。理由：**当前需求只有三种样式、一个页面**。为了"以后可能加样式"提前设计三层继承，是最常见的过度设计。

> 一个够用的判据：**当你第二次因为同一个原因改这段代码时，再重构**。

## 六、小结

| 阶段 | 代码形态 | 学到的东西 |
| --- | --- | --- |
| v1 | 循环体 7 行细节 | 能跑 ≠ 可读 |
| v2 | 7 行细节 → 2 行调用 | 函数是"给概念命名" |
| v3 | 调用也被参数决定 | 参数化的边界是"意图是否还看得见" |

最后一句总结：**函数不是把代码变短的工具，是把"意图"和"实现"分层的工具**。画格子这件事本身很小，但"什么时候抽、抽到什么程度、什么时候停"这三个判断，才是真正可迁移的部分。
