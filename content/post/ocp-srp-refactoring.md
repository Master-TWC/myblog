---
title: "开闭原则与单一职责原则重构实战"
description: "用两个经典案例理解 OCP 和 SRP 及其重构思路"
date: 2026-05-1T17:00:00+08:00
slug: "ocp-srp-refactoring"
tags: ["Java", "设计原则", "重构"]
categories: ["技术"]
---

## 开闭原则（OCP）

> **对扩展开放，对修改关闭。**
> 
> 软件实体（类、模块、函数）应该通过扩展来实现变化，而不是修改已有的代码。

## 单一职责原则（SRP）

> **一个类只应有一个引起它变化的原因。**
>
> 每个类只负责一个明确的职责，避免"万能类"的出现。

---

两个原则常常一起出现：违反了 SRP 的类往往也难以遵循 OCP。下面通过两个案例逐步重构。

## 案例一：订单折扣系统

### 混乱版本

```java
public class OrderService {
    
    public double calculateDiscount(String userLevel, double amount) {
        if ("vip".equals(userLevel)) {
            return amount * 0.8;          // VIP 8 折
        } else if ("gold".equals(userLevel)) {
            return amount * 0.7;          // 黄金 7 折
        } else if ("normal".equals(userLevel)) {
            return amount * 0.95;         // 普通 95 折
        }
        return amount;
    }
}
```

**问题分析：**

1. **违反 SRP**：`OrderService` 既处理订单又处理折扣逻辑
2. **违反 OCP**：新增用户等级需要修改 `calculateDiscount` 方法

### 第一次重构 — 分离职责

```java
// 独立的折扣策略接口
public interface DiscountStrategy {
    double discount(double amount);
}

public class VipDiscount implements DiscountStrategy {
    @Override
    public double discount(double amount) {
        return amount * 0.8;
    }
}

public class GoldDiscount implements DiscountStrategy {
    @Override
    public double discount(double amount) {
        return amount * 0.7;
    }
}

public class NormalDiscount implements DiscountStrategy {
    @Override
    public double discount(double amount) {
        return amount * 0.95;
    }
}
```

```java
public class OrderService {
    
    private DiscountStrategy discountStrategy;
    
    public OrderService(DiscountStrategy discountStrategy) {
        this.discountStrategy = discountStrategy;
    }
    
    public double calculateDiscount(double amount) {
        return discountStrategy.discount(amount);
    }
}
```

**效果：** 新增等级时只需新建一个类实现 `DiscountStrategy`，`OrderService` 不再需要修改 — 符合 OCP。

### 第二次重构 — 工厂 + 配置

```java
@Component
public class DiscountStrategyFactory {
    
    private Map<String, DiscountStrategy> strategyMap = new HashMap<>();
    
    public DiscountStrategyFactory() {
        strategyMap.put("vip", new VipDiscount());
        strategyMap.put("gold", new GoldDiscount());
        strategyMap.put("normal", new NormalDiscount());
    }
    
    public DiscountStrategy getStrategy(String userLevel) {
        DiscountStrategy strategy = strategyMap.get(userLevel);
        if (strategy == null) {
            throw new IllegalArgumentException("未知等级: " + userLevel);
        }
        return strategy;
    }
}
```

```java
public class OrderService {
    
    private DiscountStrategyFactory factory;
    
    public OrderService(DiscountStrategyFactory factory) {
        this.factory = factory;
    }
    
    public double calculateDiscount(String userLevel, double amount) {
        DiscountStrategy strategy = factory.getStrategy(userLevel);
        return strategy.discount(amount);
    }
}
```

现在新增等级只需添加策略类并注册到工厂，现有代码零修改。

---

## 案例二：报表导出系统

### 混乱版本

```java
public class ReportService {
    
    public void exportReport(String format) {
        // 1. 查询数据
        List<Data> data = queryFromDatabase();
        
        // 2. 格式化数据
        String content;
        if ("pdf".equals(format)) {
            content = formatPdf(data);
        } else if ("excel".equals(format)) {
            content = formatExcel(data);
        } else if ("csv".equals(format)) {
            content = formatCsv(data);
        } else {
            throw new IllegalArgumentException("不支持的格式");
        }
        
        // 3. 保存文件
        saveToFile(content, "report." + format);
        
        // 4. 记录日志
        log("导出成功，格式: " + format);
    }
    
    private List<Data> queryFromDatabase() { /* ... */ return null; }
    private String formatPdf(List<Data> data) { /* ... */ return null; }
    private String formatExcel(List<Data> data) { /* ... */ return null; }
    private String formatCsv(List<Data> data) { /* ... */ return null; }
    private void saveToFile(String content, String fileName) { /* ... */ }
    private void log(String msg) { /* ... */ }
}
```

**问题分析：**

1. **违反 SRP**：数据查询、格式化、文件保存、日志记录全在一个类里
2. **违反 OCP**：新增格式需要修改 `exportReport` 方法

### 重构 — 职责分离

```java
// 1. 数据查询 — 单独的数据访问层
public class ReportDataFetcher {
    public List<Data> fetch() {
        System.out.println("查询报表数据...");
        return new ArrayList<>();
    }
}

// 2. 格式化 — 策略接口
public interface ReportFormatter {
    String format(List<Data> data);
    String getExtension();
}

public class PdfFormatter implements ReportFormatter {
    @Override
    public String format(List<Data> data) {
        return "PDF 格式内容";
    }
    @Override
    public String getExtension() { return "pdf"; }
}

public class ExcelFormatter implements ReportFormatter {
    @Override
    public String format(List<Data> data) {
        return "Excel 格式内容";
    }
    @Override
    public String getExtension() { return "xlsx"; }
}

public class CsvFormatter implements ReportFormatter {
    @Override
    public String format(List<Data> data) {
        return "CSV 格式内容";
    }
    @Override
    public String getExtension() { return "csv"; }
}

// 3. 文件保存 — 独立的输出层
public class FileExporter {
    public void save(String content, String fileName) {
        System.out.println("保存文件: " + fileName);
    }
}

// 4. 日志 — 独立的日志服务
public class ReportLogger {
    public void logExport(String format, boolean success) {
        System.out.println("导出" + (success ? "成功" : "失败")
                         + "，格式: " + format + "，时间: " + LocalDateTime.now());
    }
}
```

```java
// 组装 — ReportService 只负责编排
public class ReportService {
    
    private final ReportDataFetcher dataFetcher;
    private final ReportFormatter formatter;
    private final FileExporter fileExporter;
    private final ReportLogger logger;
    
    public ReportService(ReportDataFetcher dataFetcher,
                         ReportFormatter formatter,
                         FileExporter fileExporter,
                         ReportLogger logger) {
        this.dataFetcher = dataFetcher;
        this.formatter = formatter;
        this.fileExporter = fileExporter;
        this.logger = logger;
    }
    
    public void exportReport() {
        try {
            List<Data> data = dataFetcher.fetch();
            String content = formatter.format(data);
            fileExporter.save(content, "report." + formatter.getExtension());
            logger.logExport(formatter.getExtension(), true);
        } catch (Exception e) {
            logger.logExport(formatter.getExtension(), false);
            throw e;
        }
    }
}
```

```java
// 使用时
ReportFormatter formatter = new PdfFormatter();
ReportService service = new ReportService(
    new ReportDataFetcher(),
    formatter,
    new FileExporter(),
    new ReportLogger()
);
service.exportReport();
```

现在新增导出格式只需新建一个 `ReportFormatter` 实现类，其它所有类都不需要修改。

---

## 重构要点总结

| 原则 | 判断标准 | 典型坏味道 | 重构手法 |
|------|---------|-----------|---------|
| SRP | 一个类的职责能否用一句话说清？ | 类名带"And"、"Manager"、"Util" | 提取类/接口 |
| OCP | 加新功能是否需要改已有代码？ | 大量的 if-else / switch | 策略模式、模板方法 |

**判断违反 SRP 的快速测试：**

> 如果一个类因为多个不同的原因被修改，它就违反了 SRP。

举例：`OrderService` 因为"新增用户等级"和"修改折扣算法"两个原因都要改，说明它至少承担了两项职责。

**判断违反 OCP 的快速测试：**

> 加新功能时，试着问自己：我需要改哪些已有的类？

如果答案是"只需要加新类"，说明符合 OCP；如果答案是"还要改某个已有的类"，那就是 OCP 被违反了。

> 设计原则不是教条，需要在灵活性和复杂度之间权衡。对于确定不会再变的逻辑，写死的 if-else 可能比过度设计更可取。**关键是识别那些"一定会变"和"很可能变"的部分，在合适的地方留好扩展点。**
