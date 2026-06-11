---
title: "Java IO 流详解"
description: "Java IO 体系、字节流与字符流、装饰器模式"
date: 2026-06-11T18:00:00+08:00
slug: "java-io-stream"
tags: ["Java", "IO"]
categories: ["技术"]
---

## IO 流体系概览

Java IO 分为**字节流**和**字符流**两大体系，基类都是抽象类：

```
                    ┌───────────┐
                    │ 字节流     │     字符流
                    │           │
                    │ InputStream│    Reader
                    │    ▲      │      ▲
                    │    │      │      │
                    │ OutputStream│   Writer
                    └───────────┘
```

### 核心基类

| 类别 | 输入 | 输出 |
|------|------|------|
| 字节流 | `InputStream` | `OutputStream` |
| 字符流 | `Reader` | `Writer` |

---

## 字节流

以字节（8 bit）为单位读写，适用于所有文件类型（图片、视频、音频、文本等）。

### FileInputStream / FileOutputStream

```java
// 读取文件
try (FileInputStream fis = new FileInputStream("input.txt")) {
    int b;
    while ((b = fis.read()) != -1) {   // 读取一个字节
        System.out.print((char) b);
    }
}

// 写入文件
try (FileOutputStream fos = new FileOutputStream("output.txt")) {
    fos.write("Hello".getBytes());
}
```

**逐字节读取效率低**，应使用缓冲数组：

```java
try (FileInputStream fis = new FileInputStream("input.txt")) {
    byte[] buffer = new byte[8192];
    int len;
    while ((len = fis.read(buffer)) != -1) {
        // 处理 buffer 中 len 个字节
    }
}
```

### BufferedInputStream / BufferedOutputStream

自带缓冲区，减少底层系统调用，显著提升性能：

```java
try (BufferedInputStream bis = new BufferedInputStream(new FileInputStream("input.txt"));
     BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream("output.txt"))) {
    byte[] buffer = new byte[8192];
    int len;
    while ((len = bis.read(buffer)) != -1) {
        bos.write(buffer, 0, len);
    }
}
```

### DataInputStream / DataOutputStream

读写 Java 基本类型和 String 的字节表示：

```java
try (DataOutputStream dos = new DataOutputStream(new FileOutputStream("data.bin"))) {
    dos.writeInt(42);
    dos.writeUTF("你好");
    dos.writeDouble(3.14);
}

try (DataInputStream dis = new DataInputStream(new FileInputStream("data.bin"))) {
    int i = dis.readInt();        // 42
    String s = dis.readUTF();     // "你好"
    double d = dis.readDouble();  // 3.14
}
```

---

## 字符流

以字符（16 bit）为单位读写，专门处理文本文件，内置编码转换。

### FileReader / FileWriter

```java
// 读取文本
try (FileReader fr = new FileReader("note.txt", StandardCharsets.UTF_8)) {
    char[] buffer = new char[4096];
    int len;
    while ((len = fr.read(buffer)) != -1) {
        System.out.print(new String(buffer, 0, len));
    }
}

// 写入文本
try (FileWriter fw = new FileWriter("note.txt", StandardCharsets.UTF_8)) {
    fw.write("你好，世界");
}
```

### BufferedReader / BufferedWriter

提供 `readLine()` 按行读取，是处理文本最常用的方式：

```java
try (BufferedReader br = new BufferedReader(new FileReader("note.txt"))) {
    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
}

try (BufferedWriter bw = new BufferedWriter(new FileWriter("output.txt"))) {
    bw.write("第一行");
    bw.newLine();
    bw.write("第二行");
}
```

### InputStreamReader / OutputStreamWriter

**字节流通向字符流的桥梁**，指定编码将字节解码为字符：

```java
// 读取指定编码的文件
try (BufferedReader br = new BufferedReader(
        new InputStreamReader(new FileInputStream("data.txt"), "GBK"))) {
    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
}
```

---

## 字节流 vs 字符流

| 对比维度 | 字节流 | 字符流 |
|---------|-------|-------|
| 单位 | byte (8 bit) | char (16 bit) |
| 适用场景 | 图片、视频、音频、压缩包等 | 纯文本文件 |
| 编码处理 | 不处理，原样读写 | 内置编码/解码 |
| 基类 | InputStream / OutputStream | Reader / Writer |

**选择原则：** 非文本文件用字节流，文本文件用字符流（或字符流包装类）。

---

## 装饰器模式

Java IO 广泛使用**装饰器模式**，这也是面试常见考点。流可以多层嵌套，每一层添加一种功能：

```java
// 同时具备缓冲 + 数据读写能力
DataInputStream dis = new DataInputStream(
    new BufferedInputStream(
        new FileInputStream("data.bin")
    )
);
```

核心抽象（Component）：

```
       InputStream (抽象)
      /     |      \
FileInputStream  FilterInputStream  ...
                    |
           BufferedInputStream
           DataInputStream
           PushbackInputStream
```

`FilterInputStream` 是装饰器基类，持有 `InputStream` 引用，子类增强其行为。

---

## 常用流速查表

| 类 | 类型 | 用途 |
|----|------|------|
| FileInputStream | 字节输入 | 读取文件 |
| FileOutputStream | 字节输出 | 写入文件 |
| BufferedInputStream | 字节输入 | 带缓冲的字节读取 |
| BufferedOutputStream | 字节输出 | 带缓冲的字节写入 |
| DataInputStream | 字节输入 | 读取基本类型 + String |
| DataOutputStream | 字节输出 | 写入基本类型 + String |
| ObjectInputStream | 字节输入 | 反序列化对象 |
| ObjectOutputStream | 字节输出 | 序列化对象 |
| FileReader | 字符输入 | 读取文本文件 |
| FileWriter | 字符输出 | 写入文本文件 |
| BufferedReader | 字符输入 | 带缓冲的按行读取 |
| BufferedWriter | 字符输出 | 带缓冲的文本写入 |
| InputStreamReader | 字符输入 | 字节→字符桥梁 |
| OutputStreamWriter | 字符输出 | 字符→字节桥梁 |
| PrintStream | 字节输出 | System.out，格式化输出 |
| PrintWriter | 字符输出 | 打印格式化文本 |

---

## try-with-resources

所有 IO 流都实现了 `AutoCloseable` 接口，应使用 try-with-resources 自动关闭：

```java
// ✅ 正确做法 — 自动关闭
try (BufferedReader br = Files.newBufferedReader(Paths.get("file.txt"))) {
    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
} catch (IOException e) {
    e.printStackTrace();
}

// ❌ 错误做法 — 忘记在 finally 中关闭
```

## 实用工具：Files

Java 7 引入的 `java.nio.file.Files` 类封装了常见 IO 操作，日常开发优先使用：

```java
// 读取所有行
List<String> lines = Files.readAllLines(Paths.get("file.txt"), StandardCharsets.UTF_8);

// 写入
Files.write(Paths.get("out.txt"), "内容".getBytes());

// 复制文件
Files.copy(Paths.get("src.txt"), Paths.get("dst.txt"), StandardCopyOption.REPLACE_EXISTING);

// 遍历目录
Files.walk(Paths.get("./src"))
     .filter(p -> p.toString().endsWith(".java"))
     .forEach(System.out::println);
```

> Java IO 体系的核心是**装饰器模式**：记住四个抽象基类 `InputStream` / `OutputStream` / `Reader` / `Writer`，以及它们各自的文件、缓冲、转换实现，就能自由组合满足各种 IO 需求。日常开发推荐优先使用 `java.nio.file.Files` 工具类，让代码更简洁。
