---
title: "继承 Thread 与实现 Runnable 的区别"
description: "深入对比 Java 中创建线程的两种方式"
date: 2026-06-11T14:00:00+08:00
slug: "thread-vs-runnable"
tags: ["Java", "多线程"]
categories: ["技术"]
---

## 两种创建方式

Java 提供两种方式来实现多线程：

### 继承 Thread 类

```java
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("线程执行中: " + getName());
    }
}

// 使用
MyThread t = new MyThread();
t.start();
```

### 实现 Runnable 接口

```java
class MyTask implements Runnable {
    @Override
    public void run() {
        System.out.println("任务执行中");
    }
}

// 使用
Thread t = new Thread(new MyTask());
t.start();
```

## 核心区别

### 1. 单继承限制

Java 只允许单继承。继承 `Thread` 意味着该类无法再继承其他类，而 `Runnable` 是一个接口，实现它不影响继承其它类：

```java
// ❌ 无法继承其他类
class MyThread extends Thread {
    // 不能 extends 其他类了
}

// ✅ 还可以继承其他类
class MyTask extends BaseClass implements Runnable {
    @Override
    public void run() { }
}
```

### 2. 逻辑与线程分离

`Runnable` 将**任务逻辑**与**线程机制**解耦：

- `Runnable` 只描述"要做什么"
- `Thread` 控制"怎么运行"

```java
// 同一个任务可以被多个线程执行
Runnable task = () -> System.out.println("任务执行");

new Thread(task).start();
new Thread(task).start();  // 复用同一任务
```

而继承 `Thread` 时，每个线程都是独立的对象，无法直接共享任务逻辑。

### 3. 资源复用

实现 `Runnable` 更适合多线程共享同一资源（如卖票场景）：

```java
// Runnable 方式 — 共享 tickets
class TicketTask implements Runnable {
    private int tickets = 10;
    @Override
    public void run() {
        while (tickets > 0) {
            System.out.println(Thread.currentThread().getName() + " 卖出: " + tickets--);
        }
    }
}

// 三个线程共享同一个 task
TicketTask task = new TicketTask();
new Thread(task, "窗口1").start();
new Thread(task, "窗口2").start();
new Thread(task, "窗口3").start();
```

继承 `Thread` 无法天然实现这种共享，除非使用 `static` 变量。

### 4. 灵活性

`Runnable` 可以配合线程池使用，`Thread` 则不能：

```java
// ✅ Runnable 可提交给线程池
ExecutorService executor = Executors.newFixedThreadPool(4);
executor.submit(() -> System.out.println("任务"));
executor.shutdown();

// Thread 需要手动 start()，无法与线程池协作
```

### 5. 函数式编程

`Runnable` 是函数式接口（只有一个抽象方法），可以使用 Lambda 简洁地创建：

```java
// Lambda 写法 — 非常简洁
new Thread(() -> System.out.println("任务")).start();
```

继承 `Thread` 必须创建子类或匿名内部类，代码更冗长。

## 总结

| 对比维度 | 继承 Thread | 实现 Runnable |
|---------|------------|--------------|
| 继承约束 | 无法再继承其他类 | 不影响继承 |
| 任务复用 | ❌ 每个线程独立 | ✅ 可共享 |
| 线程池 | ❌ 不适用 | ✅ 完美支持 |
| Lambda | ❌ 需要子类 | ✅ 支持 |
| 耦合度 | 高（任务与线程耦合） | 低（关注点分离） |

**优先选择实现 `Runnable`**（或 `Callable`），仅在需要重写 `Thread` 的其它方法（如 `interrupt()`）时才考虑继承。

> 在 Java 8+ 的项目中，通常使用 `Runnable` + Lambda 或 `ExecutorService`，这样更灵活、更符合现代 Java 编程风格。
