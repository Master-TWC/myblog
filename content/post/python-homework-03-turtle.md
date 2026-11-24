---
title: "Python 实验：Turtle 自动轨迹绘制彩色图形"
description: "螺旋、分形树、曼陀罗三种轨迹，一次开窗全部画完并自动截图——顺带解决 DPI 缩放导致的截图偏移。"
date: 2026-11-19T19:30:00+08:00
slug: "python-homework-03-turtle"
tags: ["Python", "Turtle", "绘图", "实验记录"]
categories: ["技术"]
---

# Python 实验：Turtle 自动轨迹绘制彩色图形

> 题目要求"自动轨迹绘制彩色图形"。我把它理解成三件事：
> **轨迹由公式生成（不是手写坐标）→ 颜色随参数连续变化 → 画完自动截图存档**。
> 最后选了三个图案：阿基米德螺旋、递归分形树、12 瓣曼陀罗。

## 一、整体结构

`turtle` 是单画笔的串行绘图，所以要同时画三个图、还要"每个图单独截图"，必须解决两个问题：**窗口复用**和**截图时机**。

```python
class Canvas:
    """一个窗口、一支画笔，画完一张截图一张。"""
    def __init__(self, width=940, height=940):
        self.screen = turtle.Screen()
        self.screen.setup(width, height, 60, 40)
        self.screen.bgcolor("#0B1020")
        self.screen.setworldcoordinates(-450, -450, 450, 450)   # 原点居中
        self.screen.tracer(0, 0)                                 # 关掉逐笔刷新
        self.pen = turtle.Turtle(visible=False)
        self.pen.speed(0)

    def reset(self, title):
        self.screen.title(title)
        self.pen.clear()
        self.pen.penup(); self.pen.home(); self.pen.pendown()

    def snapshot(self, name):
        self.screen.update()
        root = self.screen.getcanvas().winfo_toplevel()
        root.attributes("-topmost", True)
        time.sleep(0.35)
        box = (root.winfo_rootx(), root.winfo_rooty(),
               root.winfo_rootx() + root.winfo_width(),
               root.winfo_rooty() + root.winfo_height())
        ImageGrab.grab(bbox=box, all_screens=True).save(OUT / name)
```

## 二、三个图形三种思路

### 2.1 阿基米德螺旋：公式决定轨迹

`r = a·θ`，每步前进一点点、同时转一个小角度，半径自然增长：

```python
for i in range(470):
    pen.color(hue_color(i / 470))       # 色相跟着进度走
    pen.circle(0.9 * i, 8)              # 半径越来越大 → 螺旋张开
```

![螺旋完成图](/myblog/images/homework/03-turtle/spiral.png)

*470 步、半径 0.9i，色相从 0 走到 1，形成一圈彩虹*

为了体现"画的过程"，脚本还在 25% / 50% / 75% 时各截一张：

![螺旋画到一半](/myblog/images/homework/03-turtle/spiral-50.png)

*第 235 步（50%）时的状态：螺旋已经成形，还没收尾*

### 2.2 递归分形树：把"树"写成"树的子树"

一棵树的定义是递归的——树干上长出两棵小一号的树：

```python
def branch(pen, length, depth):
    if depth == 0:
        return
    pen.pensize(max(0.6, depth * 0.9))
    pen.color(hue_color(0.75 - depth * 0.07))
    pen.forward(length)
    pen.left(28)
    branch(pen, length * 0.72, depth - 1)   # 左子树
    pen.right(56)
    branch(pen, length * 0.72, depth - 1)   # 右子树
    pen.left(28)
    pen.backward(length)                    # 回到分叉点
```

参数：`length=110`、`depth=8`、每层 ×0.72、夹角 ±28°，起点 `(0, -400)`。

![递归分形树](/myblog/images/homework/03-turtle/tree.png)

*8 层递归，枝干颜色从洋红渐变到青绿，末端自然收细*

> 递归绘图必须"画出去再走回来"（`backward(length)`），否则画笔位置会一层层漂移，树就歪了。

### 2.3 曼陀罗：一片花瓣旋转 12 次

一片花瓣用两段同半径圆弧拼成（`circle(r,60) → left(120) → circle(r,60)`），天然轴对称；把它绕中心旋转 12 次，就是曼陀罗：

```python
for layer, (radius, width, offset) in enumerate([(330, 1.9, 0), (250, 1.6, 18), (175, 1.3, 36)]):
    pen.pensize(width)
    for k in range(12):
        pen.penup(); pen.goto(0, 0)
        pen.setheading(k * 30 + offset)      # 每片都从圆心重新出发
        pen.pendown()
        pen.pencolor(hue_color(layer * 0.18 + k / 12 * 0.62))
        pen.circle(radius, 60)
        pen.left(120)
        pen.circle(radius, 60)
```

![曼陀罗](/myblog/images/homework/03-turtle/mandala.png)

*三层花瓣（330/250/175），偏移 0/18/36°，形成交错叠压的层次*

> 第一版我在同一个 `for k` 里连续画、只改 `setheading`，结果花瓣彼此不共圆心，图形是散的。**每片花瓣都必须"回到圆心 + 重新摆角"再落笔**，对称性才有保证。

## 三、两个真正的坑

### 坑 1：`Screen()` 只能建一次

画完第一个图 `screen.bye()` 之后，第二次 `turtle.Screen()` 直接抛：

```
turtle.Terminator
```

原因是 Tk 的主窗口被销毁、turtle 的全局 `_screen` 还指着旧对象。改法很土但有效：**只建一个 Canvas，每次画前 `reset()`（清画布 + 回原点 + 摆正角度）**。

### 坑 2：截图整体偏移 138 像素

第一张截图里，本该在正中的原点跑到了 `(608, 647)`（画布 940×1000）。查下来是 **Windows 的 DPI 缩放**：系统按 125% 渲染，`ImageGrab` 拿到的是物理像素，而窗口坐标是逻辑像素。

修复只用一行，放在 import 之后：

```python
try:
    ctypes.windll.shcore.SetProcessDpiAwareness(2)   # 2 = PER_MONITOR_DPI_AWARE
except Exception:
    pass
```

声明之后原点回到 `(473, 503)`，截图和画布对齐了。

## 四、运行结果

```
[spiral]  已截图：螺旋_进度25%.png（940×940）
          已截图：螺旋_进度50%.png（940×940）
          已截图：螺旋_进度75%.png（940×940）
          已截图：螺旋_完成.png（940×940）
[tree]    已截图：分形树_完成.png（940×940）
[mandala] 已截图：曼陀罗_完成.png（940×940）

共 6 张截图，总耗时约 6 秒
```

## 五、小结

| 图形 | 轨迹来源 | 学到的点 |
| --- | --- | --- |
| 螺旋 | `r = a·θ` | 用公式生成坐标，颜色跟随进度 |
| 分形树 | 递归 `branch()` | 递归要能"走回来"，否则位置漂移 |
| 曼陀罗 | 单片花瓣 + 旋转 | 对称图形靠"回到圆心重摆角" |

三条可复用的经验：

1. **`tracer(0, 0)` + 手动 `update()`** 能让几百步的图形瞬间出结果，否则要等十几秒；
2. 截图前必须 `topmost` + `sleep(0.35)`，否则抓到的是被别的窗口遮挡的画面；
3. 涉及 GUI 截图的功能，**先问一句"系统缩放是多少"**——DPI 是这类 bug 的头号来源。
