---
title: "爬虫开发实验报告：基于 Vibe Coding 的 Python 静态网页爬虫"
description: "抓取高校「通知公告」栏目 30 条真实数据：列表页解析、详情页正文长度、CSV 落盘与统计图，外加两个真实报错。"
date: 2026-11-23T21:00:00+08:00
slug: "python-homework-06-crawler"
tags: ["Python", "爬虫", "requests", "BeautifulSoup", "实验记录"]
categories: ["技术"]
---

# 爬虫开发实验报告：基于 Vibe Coding 的 Python 静态网页爬虫

> 实验对象是**静态服务端渲染**的高校通知公告栏目（不是 JS 渲染、不需要浏览器）。
> 目标很朴素：抓 3 页列表 + 30 条详情，落成 CSV，再画个统计图。
> 但真跑起来会遇到两类问题：**HTTP 头编码** 和 **字符集识别**。

## 一、选站与合规

选站标准：结构规整、允许低频访问、不涉及登录与隐私。

```
目标：桂林电子科技大学「通知公告」
列表：https://www.guet.edu.cn/tzgg/list.htm   （第 2 页起 list2.htm、list3.htm …）
详情：https://www.guet.edu.cn/2026/1113/c6360a162429/page.htm
规模：全站 439 条，每页 10 条
```

"礼貌三件套"写死在代码里：

```python
UA = ("Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 "
      "(KHTML, like Gecko) Chrome/126.0 Safari/537.36 (student-lab; low-frequency)")
SESSION.headers.update({"User-Agent": UA, "Accept-Language": "zh-CN,zh;q=0.9"})
...
time.sleep(0.4)      # 每次请求之间固定间隔，不并发
```

## 二、分层：一个函数只干一件事

```python
def fetch(url, retries=3) -> str | None:      # 重试 + 退避 + 字符集纠正
def parse_list(html) -> list[dict]:           # 列表页 → 标题/日期/摘要/链接
def parse_detail(html) -> tuple[int, str]:    # 详情页 → (正文长度, 发布时间)
def plot(rows) -> Path:                       # 月度分布 + 标题热词
```

### 2.1 列表页：`li.news` 就是一行数据

```python
for li in soup.select("li.news"):
    date_node = li.select_one(".news_date")
    link = li.select_one(".news_title a")
    summary_node = li.select_one(".news_text a")
    items.append({
        "标题": (link.get("title") or link.get_text(strip=True)).strip(),
        "日期": date_node.get_text(strip=True),
        "链接": urljoin(BASE, link["href"]),      # 相对路径转绝对
        "摘要": summary_node.get_text(strip=True) if summary_node else "",
        "正文长度": 0,
    })
```

两个细节：`link.get("title")` 优先取 `title` 属性（比可见文字完整）；`urljoin` 处理相对链接，否则详情页请求会 404。

### 2.2 详情页：只要一个数字

正文长度用"去空白后的字符数"，比"段落数"更能反映信息量：

```python
body = soup.select_one(".wp_articlecontent")
body_len = len(re.sub(r"\s+", "", body.get_text())) if body else 0
```

### 2.3 重试与退避

```python
for attempt in range(1, retries + 1):
    try:
        resp = SESSION.get(url, timeout=15)
        resp.raise_for_status()
        if not resp.encoding or resp.encoding.lower() == "iso-8859-1":
            resp.encoding = resp.apparent_encoding
        return resp.text
    except requests.RequestException as exc:
        time.sleep(attempt * 1.2 + random.random())
```

**`apparent_encoding` 这一行是必需的**：服务器没在响应头里声明 charset 时，requests 会退回 ISO-8859-1，中文全部变乱码；用 `apparent_encoding`（chardet 探测）覆盖即可。

## 三、两个真实报错

### 报错 1：UA 里写了中文

```
UnicodeEncodeError: 'latin-1' codec can't encode characters in position 111-119
```

HTTP 头按 RFC 只能用 latin-1 编码，而我第一版 UA 里写了注释 `# 学生实验，低频访问`。改成纯 ASCII 的 `(student-lab; low-frequency)` 即可。

> 这个坑很典型：**注释写在字符串里，就不再是注释**。

### 报错 2：乱码

表现是标题全是 `å…¨å›½â€¦`。原因就是上一节的编码问题——加 `apparent_encoding` 后恢复。

## 四、运行结果

```
[列表 1/3] https://www.guet.edu.cn/tzgg/list.htm      解析 10 条，新增 10 条
[列表 2/3] https://www.guet.edu.cn/tzgg/list2.htm     解析 10 条，新增 10 条
[列表 3/3] https://www.guet.edu.cn/tzgg/list3.htm     解析 10 条，新增 10 条
[汇总] 去重后共 30 条
[详情] 进度 10/30 … 30/30
[输出] CSV：out/通知公告.csv（30 行）
[输出] 图表：out/爬虫统计.png
       正文长度：平均 1385 字，最长 5239 字，最短 234 字
[耗时] 19.8 秒
```

![爬取结果统计](/myblog/images/homework/06-crawler/crawl-stats.png)

*左：按月份分布（2026-04 最多，10 条）；右：标题高频词 Top 12*

CSV 前两行长这样（`utf-8-sig` 编码，Excel 直接打开不乱码）：

| 标题 | 日期 | 正文长度 | 链接 |
| --- | --- | --- | --- |
| 关于开展2027年"挑战杯"大学生课外学术科技作品竞赛校内选拔赛的通知 | 2026-11-13 | 2386 | `.../2026/1113/c6360a162429/page.htm` |
| 桂林电子科技大学2026年校企联聘高层次人才招聘公告 | 2026-11-02 | 5239 | `.../2026/1102/c6360a161894/page.htm` |

## 五、数据说明什么

- **正文长度差异巨大**（234 ~ 5239 字）：招聘公告最长，活动通知中等，简讯最短；
- **月份分布不均**：4 月 10 条、3 月 4 条、8 月只有 1 条——和高校的学期节奏吻合（开学季通知多，寒暑假少）；
- **标题热词**"大学/桂林/电子科技"是校名，真正有信息量的是"招聘/公示/考试/教师"，说明这个栏目以人事与教学类通知为主。

## 六、产出

```
06_静态爬虫/
├── crawl_guet.py
└── out/
    ├── 通知公告.csv       30 行，utf-8-sig
    ├── 爬虫统计.png       月度分布 + 标题热词
    └── 运行日志.txt
```

## 七、小结

| 问题 | 原因 | 解法 |
| --- | --- | --- |
| `UnicodeEncodeError` | 请求头只能 latin-1 | UA 不用中文 |
| 中文乱码 | 响应头没声明 charset | `resp.apparent_encoding` |
| 详情页 404 | 相对链接 | `urljoin(BASE, href)` |
| 被限流 | 请求太密 | 固定 0.4s 间隔 + 单线程 |

三条更通用的经验：

1. **先看 HTML 结构再写选择器**（`li.news` 比"第几个 div"稳定得多）；
2. **每层都做一次"最小可用验证"**：列表能解析 → 详情能取到长度 → CSV 能落盘，一步步来；
3. 爬虫的合规性不是可选项：**低频、声明身份、只抓公开信息**，这三条是底线。
