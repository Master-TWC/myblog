---
title: "匿名内部类使用场景"
description: "Java 匿名内部类的常见用法与最佳实践"
date: 2026-06-11T15:00:00+08:00
slug: "anonymous-inner-class"
tags: ["Java", "面向对象"]
categories: ["技术"]
---

## 什么是匿名内部类

**匿名内部类（Anonymous Inner Class）** 是一种没有名字的局部内部类，用于在需要时快速创建某个接口或抽象类的实现。它必须在定义时同时创建实例，且只能使用一次。

```java
// 常规做法：定义一个命名类
class MyTask implements Runnable {
    @Override
    public void run() {
        System.out.println("任务执行");
    }
}
new Thread(new MyTask()).start();

// 匿名内部类：省去命名，直接创建
new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("任务执行");
    }
}).start();
```

## 常见使用场景

### 1. 事件监听（GUI 编程）

匿名内部类最经典的场景是 Swing/Android 的事件处理：

```java
button.addActionListener(new ActionListener() {
    @Override
    public void actionPerformed(ActionEvent e) {
        System.out.println("按钮被点击");
        // 处理点击事件
    }
});
```

每个按钮的点击逻辑各不相同，不值得为每次点击创建一个命名类，匿名内部类在这里恰到好处。

### 2. 线程创建

配合 `Runnable` 或 `Thread` 创建一次性线程：

```java
new Thread(new Runnable() {
    @Override
    public void run() {
        // 后台执行的任务
        downloadFile();
    }
}).start();
```

虽然 Java 8 后 Lambda 更简洁，但在低版本或需要 `this` 指向内部类时，匿名内部类仍有优势。

### 3. 自定义比较器

对集合进行一次性排序时：

```java
List<Person> list = new ArrayList<>();
list.add(new Person("Alice", 25));
list.add(new Person("Bob", 20));

Collections.sort(list, new Comparator<Person>() {
    @Override
    public int compare(Person a, Person b) {
        return Integer.compare(a.getAge(), b.getAge());
    }
});
```

### 4. 策略模式简化

当策略只在一个地方使用时，匿名内部类可以避免接口实现类的扩散：

```java
interface PaymentStrategy {
    void pay(int amount);
}

class Order {
    private PaymentStrategy strategy;
    
    public void setStrategy(PaymentStrategy strategy) {
        this.strategy = strategy;
    }
    
    public void checkout(int amount) {
        strategy.pay(amount);
    }
}

// 使用时直接传入匿名实现
Order order = new Order();
order.setStrategy(new PaymentStrategy() {
    @Override
    public void pay(int amount) {
        System.out.println("微信支付: " + amount + "元");
    }
});
order.checkout(100);
```

### 5. 模板方法模式

父类定义骨架，匿名内部类填充具体实现：

```java
abstract class BaseTask {
    public void execute() {
        long start = System.currentTimeMillis();
        doRun();
        long end = System.currentTimeMillis();
        System.out.println("耗时: " + (end - start) + "ms");
    }
    protected abstract void doRun();
}

// 使用时
new BaseTask() {
    @Override
    protected void doRun() {
        // 具体业务逻辑
    }
}.execute();
```

## 注意事项

### 变量捕获

匿名内部类访问外部局部变量时，变量必须是 `final` 或 effectively final：

```java
public void doSomething() {
    String msg = "Hello";   // effectively final
    int count = 0;          // effectively final
    
    Runnable r = new Runnable() {
        @Override
        public void run() {
            System.out.println(msg);    // ✅ 可以访问
            // count++;                 // ❌ 编译错误
        }
    };
}
```

### this 指向

在匿名内部类中，`this` 指向的是内部类实例本身，而非外部类：

```java
public class Outer {
    private String name = "Outer";
    
    void test() {
        Runnable r = new Runnable() {
            private String name = "Inner";
            
            @Override
            public void run() {
                System.out.println(this.name);      // Inner
                System.out.println(Outer.this.name); // Outer
            }
        };
    }
}
```

## 匿名内部类 vs Lambda

Java 8 引入 Lambda 后，匿名内部类使用场景有所缩减，但 Lambda 不能完全替代它：

| 对比维度 | 匿名内部类 | Lambda |
|---------|-----------|--------|
| 抽象方法数量 | 任意数量 | 只能一个（函数式接口） |
| this 指向 | 内部类实例 | 外部类实例 |
| 语法简洁度 | 较冗长 | 简洁 |
| 反编译 | 生成独立的 class 文件 | invokedynamic |

当接口有多个抽象方法时，必须使用匿名内部类：

```java
// ❌ Lambda 不适用
// button.addKeyListener(e -> { });  // KeyListener 有三个方法

// ✅ 必须使用匿名内部类
button.addKeyListener(new KeyListener() {
    @Override public void keyTyped(KeyEvent e) { }
    @Override public void keyPressed(KeyEvent e) { }
    @Override public void keyReleased(KeyEvent e) { }
});
```

> **优先使用 Lambda**（函数式接口时），当接口有多个抽象方法或需要 distinct `this` 指向时，再考虑匿名内部类。
