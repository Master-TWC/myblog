---
title: "Python 实验：本地文档智能审查与摘要"
description: "零第三方依赖解析 .docx：8 条规则体检 + 关键词抽取 + 抽取式摘要，顺手给出 72/100 的质量分。"
date: 2026-11-18T20:10:00+08:00
slug: "python-homework-01-doc-review"
tags: ["Python", "文本处理", "NLP", "实验记录"]
categories: ["技术"]
---

# Python 实验：本地文档智能审查与摘要

> 实验目标：不联网、不调大模型，纯本地用 Python 把一份 Word 文档读进来，做三件事——
> **体检（找问题）→ 抽词（看重点）→ 摘要（给结论）**。

## 一、需求拆解

老师给的题目是"本地文档智能审查与摘要"，拆开就是四个可验证的输出：

| 模块 | 输入 | 输出 | 验收方式 |
| --- | --- | --- | --- |
| 文档解析 | `实验报告.docx` | 纯文本段落 + 句列表 | 段落数、字数对得上 |
| 规则审查 | 段落/句子 | 问题清单（类型 + 位置 + 建议） | 人工抽查 10 条命中 |
| 关键词 | 全文 | Top10 高频实词 | 读起来像"内容词" |
| 摘要 | 句子集合 | 5~8 句摘要 | 能独立读懂，且不改原句 |

关键约束：**不能用 `python-docx`**（实验环境里没装），得自己从 `.docx` 里把文字挖出来。

## 二、.docx 其实是个 zip

Word 文档本质是 ZIP 包，正文在 `word/document.xml`，用标准库就能开：

```python
import zipfile
from xml.etree import ElementTree as ET

NS = "{http://schemas.openxmlformats.org/wordprocessingml/2006/main}"

def read_docx_paragraphs(path):
    with zipfile.ZipFile(path) as zf:
        xml = zf.read("word/document.xml")
    root = ET.fromstring(xml)
    paras = []
    for p in root.iter(f"{NS}p"):
        # 表格里的段落跳过：w:tbl 里的文字对"正文体检"是噪声
        if any(anc.tag == f"{NS}tbl" for anc in p.iterancestors()):
            continue
        text = "".join(t.text or "" for t in p.iter(f"{NS}t"))
        if text.strip():
            paras.append(text.strip())
    return paras
```

> 踩坑：一开始没排除表格，33 张表格里的短句全被当成"短段落"，问题数直接飙到 **722 条**。加上 `iterancestors()` 过滤后回到 12 条——**规则要作用在正确的对象上**。

## 三、8 条审查规则

规则不求多，求"看一眼就知道怎么改"：

| # | 规则 | 级别 | 判定 |
| --- | --- | --- | --- |
| 1 | 超长句（>60 字） | 中 | 逐句列出前 8 条，带证据 |
| 2 | 重复标点 / 中文里的半角标点 | 低 | `。。`、`，，`、`中文,中文` |
| 3 | 口语词 | 低 | 然后/就是/那个/基本/其实/大概… |
| 4 | 连续空白或全角空格 | 低 | 两个以上空格或 `\u3000` |
| 5 | 中英文之间未加空格 | 低 | 「用Streamlit搭建」 |
| 6 | 短段落（<12 字） | 低 | 多为标题或碎行 |
| 7 | 相邻段落完全重复 | 中 | 上下段一模一样 |
| 8 | 表格数量 | 提示 | 只统计不扣分（报告里表格多是加分项） |

实现上统一成"规则函数往 `Review.issues` 里塞问题对象"，主流程只负责收集与计分：

```python
@dataclass
class Issue:
    rule: str
    level: str      # 高 / 中 / 低
    where: str      # 第几段第几句
    evidence: str

@property
def score(self) -> int:               # Review.score
    penalty = {"高": 6, "中": 3, "低": 1}
    lost = sum(penalty.get(i.level, 1) for i in self.issues)
    return max(5, 100 - lost)         # 扣分制，最低 5 分
```

## 四、关键词：分词 + 包含关系去重

没有 jieba 的情况下，用"2~4 字 n-gram + 停用词 + 包含关系去重"也能出效果：

```python
ordered = sorted(freq.items(), key=lambda kv: (-kv[1], -len(kv[0])))
for word, count in ordered:
    if count < 3 or word in {"py", "px", "www", "com", "图", "表"}:
        continue
    # 被更长且不低于自己频次的词包含时丢弃：有「销售额」就不保留「售额」
    if any(word != other and word in other and count <= other_count
           for other, other_count in kept):
        continue
    kept.append((word, count))
```

## 五、摘要：给句子打分再按原文顺序输出

抽取式摘要不生成新句子，只排序挑选，安全可控：

```python
for idx, sentence in enumerate(sentences):
    if len(sentence) < 15 or sentence[-1] not in "。！？":
        continue                          # 标题、编号、表格残留不参与
    base = sum(weight[t] for t in set(tokens))
    score = base / (len(sentence) ** 0.5)  # 长句摊薄，避免一句话包场
    score *= 1.05 if idx < n * 0.3 else 1.0
    scored.append((idx, sentence, round(score, 3)))

picked = sorted(scored, key=lambda x: -x[2])[:5]
picked.sort(key=lambda x: x[0])           # 按原文顺序输出，读起来才连贯
```

> 踩坑：不加"结尾必须是句号"这一条时，摘要把"数据预处理""三、图表设计"这类**标题碎片**也抽了出来，拼起来像目录而不是摘要。

## 六、跑一遍真实文档

拿 lab2 的实验报告当样本（`实验2 数据看板_实验报告.docx`）：

```
文档：实验2 数据看板_实验报告.docx
段落 154 个（表格 33 张已跳过），总字符 6,393，中文字数 3,987
切句 858 → 有效句 197，平均句长 32.4 字，最长 130 字
命中问题 12 条，质量分 72 / 100
关键词：数据×32  图表×25  看板×21  代码×20  进阶×15  实验×13  基础×12
```

![文档审查结果三联图](/myblog/images/homework/01-doc-review/review-chart.png)

*左：句长分布（红线是 60 字警戒线）；中：问题类型条数；右：关键词 Top10*

自动摘要输出（5 句，全部是原文原句）：

> 1. 本次实验按《实验2 数据看板的设计与实现》的要求，围绕一个电商销售场景做一块经营大盘……（长句，101 字）
> 2. 所以本报告的顺序是——先讲清楚要回答什么问题，再讲数据怎么洗干净，最后才讲画了什么图。
> 3. ·全看板只用 4 类基础图表 + 2 张表格，不为炫技引入树图、桑基图——信息降噪本身就是设计能力。
> 4. 脏数据不清理，看板就会「漂亮地给出错误结论」，空值检查与口径一致性校验是必需步骤，不是可选项。
> 5. 进阶部分新增的 6 种图形，每一种都对应一个基础图表回答不了的问题，没有为炫技而加。

## 七、质量分怎么来的

不是拍脑袋，是扣分制：

| 级别 | 每条扣分 | 本次命中 |
| --- | --- | --- |
| 高 | 6 | 0 条 |
| 中（超长句） | 3 | 8 条 → -24 |
| 低（标点/口语/空白/短段落） | 1 | 4 条 → -4 |

100 − 28 = **72 分**。分数低不代表写得差，代表"给 AI 读之前得先拆句"——8 条超长句里有 4 条超过 80 字，最长 130 字。

## 八、产出与复盘

```
01_文档审查与摘要/
├── review.py            430 行，零第三方依赖
├── data/                待审文档
└── out/
    ├── 审查报告.md       问题清单 + 关键词 Top20 + 摘要
    ├── 审查图表.png      三联图
    └── 运行日志.txt
```

三条经验：

1. **规则要作用在正确的对象上**：表格里的短句不是"短段落"，过滤前问题数 722 条，过滤后 12 条；
2. **摘要质量取决于"准入条件"**：长度 ≥15 字、结尾是句末标点，这两个门槛比打分公式影响大得多；
3. 纯标准库也能做出可用的文本分析，但**中文词感确实是 n-gram 的天花板**——后面做词云换成 jieba 之后，Top 词立刻从"数字/直接/同一"变成了"经济/治理/科技"这种真正的概念词。

