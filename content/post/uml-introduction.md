---
title: "UML 类图详解"
description: "UML 类图的核心元素与关系表示"
date: 2026-05-19T16:00:00+08:00
slug: "uml-class-diagram"
tags: ["UML", "设计"]
categories: ["技术"]
---

## 什么是 UML

**UML（Unified Modeling Language，统一建模语言）** 是一种用于软件系统可视化的标准建模语言。它通过图形化的方式描述系统的结构、行为和交互，是软件开发中沟通设计的通用语言。

## 类图核心元素

### 类的表示

一个类用矩形表示，分为三个区域：类名、属性、方法：

```
┌──────────────────────┐
│        Person         │  ← 类名（居中、加粗）
├──────────────────────┤
│ - name: String        │  ← 属性
│ - age: int            │
│ # id: String          │
│ + address: String     │
├──────────────────────┤
│ + getName(): String   │  ← 方法
│ + setName(n: String)  │
│ # validate(): boolean │
│ - format(): void      │
└──────────────────────┘
```

**可见性符号：**

| 符号 | 可见性 | 含义 |
|-----|-------|------|
| `+` | public | 对外可见 |
| `-` | private | 仅类内部可见 |
| `#` | protected | 子类可见 |
| `~` | package | 同包可见 |

### 抽象类与接口

```
┌──────«interface»──────┐      ┌───«abstract class»───┐
│      Drawable         │      │      Shape            │
├───────────────────────┤      ├──────────────────────┤
│ + draw(): void        │      │ - color: String       │
│ + resize(r: double)   │      ├──────────────────────┤
└───────────────────────┘      │ + abstract area():…  │
                               │ + getColor(): String  │
                               └──────────────────────┘
```

接口使用 `«interface»` 构造型标记，抽象类使用 `«abstract class»` 或斜体类名。

## 类之间的关系

### 1. 泛化（Generalization）— 继承

实线空心三角箭头，子类指向父类：

```
    ┌──────────┐
    │  Animal  │
    └────┬─────┘
         △
        / \
    ┌───┘ └───┐
    │          │
┌───────┐ ┌───────┐
│  Dog  │ │  Cat  │
└───────┘ └───────┘
```

代码体现：`class Dog extends Animal`

### 2. 实现（Realization）— 接口

虚线空心三角箭头，实现类指向接口：

```
  «interface»
  ┌──────────┐
  │  Flyable │
  └────┬─────┘
       △
    ⋮  │  ⋮
┌───────┴───────┐
│    Bird        │
└────────────────┘
```

代码体现：`class Bird implements Flyable`

### 3. 关联（Association）

实线箭头，表示"拥有"关系：

```
┌────────┐     ┌────────┐
│ Teacher├────►│ Student│
└────────┘     └────────┘
```

代码体现：`class Teacher { List<Student> students; }`

### 4. 聚合（Aggregation）— 弱拥有

空心菱形 + 实线，表示整体与部分的关系，部分可脱离整体存在：

```
┌──────────┐          ┌────────┐
│  School  │◇────────│ Teacher│
└──────────┘          └────────┘
```

学校停办后，老师依然存在。代码体现：

```java
class School {
    private List<Teacher> teachers;  // 老师由外部传入
    public School(List<Teacher> teachers) {
        this.teachers = teachers;
    }
}
```

### 5. 组合（Composition）— 强拥有

实心菱形 + 实线，部分的生命周期依赖于整体：

```
┌─────────┐          ┌──────┐
│  House  │◆────────│ Room │
└─────────┘          └──────┘
```

房子拆除后，房间也不复存在。代码体现：

```java
class House {
    private List<Room> rooms;
    public House() {
        rooms = new ArrayList<>();  // 房间由 House 内部创建
        rooms.add(new Room());
    }
}
```

### 6. 依赖（Dependency）

虚线箭头，表示临时使用关系：

```
┌────────┐     ┌──────────┐
│  Car   │……> │  GasStation│
└────────┘     └──────────┘
```

代码体现（方法参数、局部变量、静态方法调用）：

```java
class Car {
    public void refuel(GasStation station) {  // 方法参数
        station.pump();
    }
}
```

## 多重性标识

| 表示 | 含义 |
|------|------|
| `1` | 恰好一个 |
| `0..1` | 零或一个 |
| `*` | 零到多个 |
| `1..*` | 一到多个 |
| `0..5` | 零到五个 |

示例：

```
┌────────┐            ┌────────┐
│ Order  │1         1..*│ Order  │
│ 订单   │───────────│ 订单项 │
└────────┘  contains  └────────┘
```

## 综合示例

一个简单的订单系统类图：

```
┌──────────┐          ┌──────────────┐
│  User    │          │    Order     │
├──────────┤1      1..*├──────────────┤
│ - id:long│─────────►│ - id: long   │
│ - name   │          │ - total:double│
├──────────┤          │ - status     │
│+login()  │          ├──────────────┤
│+logout() │          │+pay(): void  │
└──────────┘          │+cancel():void│
                      └──────┬───────┘
                             │1
                             │
                      ┌──────┴───────┐
                      │  OrderItem   │
                 1..* ├──────────────┤
                      │ - qty: int   │
                      │ - price:double
                      ├──────────────┤
                      │+subtotal():  │
                      └──────┬───────┘
                             │1
                             │
                      ┌──────┴───────┐
                      │   Product    │
                      ├──────────────┤
                      │ - name       │
                      │ - price:double│
                      └──────────────┘
```

## 常用的 UML 工具

- **PlantUML** — 文本驱动，代码生成 UML
- **draw.io** — 免费在线绘图
- **StarUML** — 开源桌面工具
- **IntelliJ IDEA** — 内置类图查看功能
- **Mermaid** — Markdown 原生支持，适合嵌入文档

> UML 类图的核心价值在于**沟通**而非"画图"。保持简单清晰，不必追求 100% 符合规范，让团队成员能快速理解设计意图才是关键。
