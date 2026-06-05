---
title: "Java 反射原理详解"
description: "深入理解 Java 反射机制的核心原理与使用"
date: 2026-06-05T10:00:00+08:00
slug: "java-reflection"
tags: ["Java", "反射"]
categories: ["技术"]
---

## 什么是反射

**反射（Reflection）** 是 Java 语言提供的一种运行时自省能力，允许程序在运行时获取任意类的内部信息，并直接操作对象的属性、方法和构造器。

## 核心原理

Java 反射的核心在于 `Class` 对象。每个类在 JVM 中只有一个对应的 `Class` 实例，它包含了该类的完整结构信息：

### 类加载过程

1. **加载**：JVM 将 `.class` 文件读入内存，生成 `Class` 对象
2. **链接**：验证字节码合法性，分配静态存储空间
3. **初始化**：执行静态代码块和静态变量赋值

### 获取 Class 对象的三种方式

```java
// 方式一：Class.forName() — 最常用
Class<?> clazz = Class.forName("com.example.User");

// 方式二：类名.class
Class<?> clazz = User.class;

// 方式三：对象.getClass()
User user = new User();
Class<?> clazz = user.getClass();
```

## 常用操作

### 获取构造器并创建实例

```java
Class<?> clazz = Class.forName("com.example.User");
Constructor<?> constructor = clazz.getDeclaredConstructor(String.class, int.class);
constructor.setAccessible(true); // 访问私有构造器
Object obj = constructor.newInstance("Alice", 25);
```

### 调用方法

```java
Class<?> clazz = obj.getClass();
Method method = clazz.getDeclaredMethod("setName", String.class);
method.invoke(obj, "Bob");
```

### 访问字段

```java
Field field = clazz.getDeclaredField("name");
field.setAccessible(true);
field.set(obj, "Charlie");
```

## 性能影响

反射比直接调用慢的原因：

1. **类型检查**：每次调用都要校验类型安全
2. **装箱拆箱**：基本类型与包装类的转换开销
3. **安全检测**：`setAccessible` 需要绕过访问权限检查

优化建议：对频繁调用的反射方法使用 `setAccessible(true)` 并缓存 `Method`/`Field` 对象。

## 应用场景

- **框架开发**：Spring 的 IoC 容器、MyBatis 的 ORM 映射
- **注解处理**：运行时解析 `@Autowired`、`@Override` 等注解
- **动态代理**：JDK 动态代理基于反射实现
- **IDE 工具**：代码补全、类型检查等功能

## 总结

反射是 Java 动态特性的基石，虽然牺牲了一定的性能，但为框架和工具提供了极大的灵活性。理解反射原理对深入掌握 Java 生态至关重要。
