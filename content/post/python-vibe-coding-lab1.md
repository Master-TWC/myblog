---
title: "Python Vibe Coding 实验一：Excel 报表脚本与 FastAPI 记账接口"
description: "60 分钟限时实操的两道题：用 openpyxl 把脏数据清洗成可交付报表，用 FastAPI 写一个不会算错一分钱的记账接口，附真实运行截图与踩坑记录"
date: 2026-11-22T15:10:00+08:00
slug: "python-vibe-coding-lab1"
tags: ["Python", "FastAPI", "Excel", "Vibe Coding", "实验记录"]
categories: ["技术"]
---

## 写在前面

这学期有一门「Python Vibe Coding」的实操考核，60 分钟限时，全程用自然语言指挥 AI 写代码，最后按 **结果、过程、验证、风险、复盘** 五个维度打分。

两道题都不难，难的是「验收」：

- **考题一**：给一份有缺失值、负数量、重复行的销售明细表，清洗后生成含四张工作表的月度报表，**不允许改动原始文件**，异常输入要给可读的中文提示；
- **考题二**：用 FastAPI 写一个记账接口，支持新增、条件分页查询、区间汇总，参数非法要返回 4xx + 结构化错误，**并发写入不能出数据错误**。

真正会翻车的地方，一个在「一分钱对不对得上」，一个在「边界输入会不会裸崩」。下面是两个工程的实际做法、运行结果和踩坑记录。

---

## 一、考题一：一条命令生成月度报表

工程在 `lab1/excel核对`，只依赖 openpyxl，三个脚本合计 1284 行（有效代码 1036 行）：

```text
excel核对/
├─ make_report.py       主程序：清洗 → 统计 → 写出四表   542 行
├─ verify_report.py     独立核对：反向校验报表            244 行
├─ gen_mock_data.py     样例数据生成器                    250 行
├─ data/
│  ├─ sales_raw_2024_01..12.xlsx   12 个月样例原始数据
│  ├─ expected_stats.json          人工核算出的期望值
│  └─ bad/                         5 份异常输入样本
└─ out/                            生成的 report_*.xlsx
```

运行就一行：

```bash
python make_report.py data\sales_raw_2024_01.xlsx   # 单月
python make_report.py data                          # 批量：整个目录
```

### 1.1 最容易错的地方：金额必须按「分」算

如果直接用 float 累加，97 行明细的合计可能和「按销售员分组」的合计差 1 分钱——而验收标准恰好是「四路合计完全相等」。解决办法是把金额统一换算成整数分：

```python
def amount_cents(qty, price):
    """数量 × 单价 -> 金额（单位：分）。Decimal 保证两位小数取整稳定。"""
    if qty is None or price is None:
        return 0
    value = Decimal(str(qty)) * Decimal(str(price))
    return int(value.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP) * 100)
```

注意这里先 `str()` 再转 `Decimal`：`Decimal(0.1)` 会得到 `0.1000000000000000055…`，而 `Decimal("0.1")` 才是精确的 0.1。这一个细节决定了在 `.005` 的边界上会不会取整取错方向。

### 1.2 三步清洗，顺序不能换

清洗有三条规则，执行顺序是有讲究的——先剔缺失，再剔非法数量，最后去重：

```python
# 规则 1：关键字段缺失 / 无法解析
if problems:
    result.drop_missing += 1
    continue

# 规则 2：数量 <= 0
if qty <= 0:
    result.drop_qty += 1
    continue

# 规则 3：全字段去重（文本字段此前已 strip）
key = (date, text_values["销售员"], text_values["产品"],
       round(qty, 6), round(price, 6), text_values["地区"])
if key in seen:
    result.drop_dup += 1
    continue
seen[key] = excel_row
```

如果先做去重，两条残缺的行可能因为「长得一样」被合并，导致最终统计的剔除原因失真；而「重复」的判定必须建立在文本已去空格的基础上，否则 `" 张三"` 和 `"张三"` 会被当成两条不同记录，全字段去重就形同虚设。去重键里对数字做了 `round(qty, 6)`，是因为 Excel 里同一个数可能存成 `3` 和 `3.0000000000000004`。

### 1.3 原始文件只读 + 统一错误出口

题目明确要求不得修改原始文件，仅靠「读进来算完不保存」还不够，所以加载时强制只读、取缓存值，写出前再比对一次输入输出路径：

```python
wb = openpyxl.load_workbook(path, read_only=True, data_only=True)
```

所有「可预期的输入问题」都抛同一个自定义异常，主流程捕获后打印中文提示，并用退出码区分问题性质：

```python
class UserError(Exception):
    """可预期的输入问题：给出可读提示，退出码 2。"""

try:
    process_file(src, dst)
    ok += 1
except UserError as exc:
    failures.append((src, str(exc)))
    print("× 处理失败：%s" % exc, file=sys.stderr)

return 2 if failures else 0
```

- `0`：成功
- `2`：输入数据有问题（找数据，不用改代码）
- `1`：程序内部异常

这样批量跑 12 个月时，人或者 CI 只看退出码就知道该找谁。

### 1.4 跑出来的结果

下面是脚本真实运行时的控制台输出，剔除的每一行都标了行号和原因：

![make_report.py 处理 2024-01 的控制台输出](/myblog/images/lab1/01-terminal-run.png)

*图 1：112 行原始数据 → 剔除 15 行（缺失 7、数量 ≤ 0 共 3、重复 5）→ 有效 97 行*

| 指标 | 数值 |
|---|---|
| 原始数据行数 | 112 |
| 剔除：关键字段缺失或无法解析 | 7 |
| 剔除：数量 ≤ 0 | 3 |
| 剔除：全字段重复 | 5 |
| 清洗后有效明细 | 97 |
| 总销售数量 | 424 |
| 月度总销售额 | **840,235.50 元** |
| 销售员 / 地区 / 产品 | 5 人 / 5 个 / 5 种 |

生成的报表有四个工作表，「总览」页把 20 项指标和校验结果一次列全，打开第一眼就能看出四个数对不对得上：

![报表总览工作表](/myblog/images/lab1/02-excel-overview.jpg)

*图 2：总览页——原始 112 行、剔除 15 行、有效 97 行、总额 840,235.50 元，「校验：销售额合计」显示四路合计一致*

![按销售员汇总工作表](/myblog/images/lab1/03-excel-by-sales.jpg)

*图 3：按销售员汇总，附明细行数、销售数量、金额占比与合计行*

![明细清洗后工作表](/myblog/images/lab1/04-excel-detail.jpg)

*图 4：明细清洗后——冻结首行、开启自动筛选，金额列 `#,##0.00`*

### 1.5 五份异常样本：不崩溃，说人话

专门做了 `data/bad/` 目录，按「程序最可能在哪儿断」设计了五份样本：

| 输入样本 | 特征 | 程序提示 | 退出码 |
|---|---|---|---|
| 空文件.xlsx | 0 字节 | 文件是 0 字节（空文件），没有任何内容 | 2 |
| 仅表头.xlsx | 只有一行表头 | 工作表《Sheet1》只有表头、没有数据行 | 2 |
| 缺字段.xlsx | 表头只有 4 列 | 缺少必需字段：日期、销售员、产品、数量、单价、地区（并回显实际读到的表头） | 2 |
| 全脏数据.xlsx | 5 行全不合格 | 清洗后没有有效数据，无法生成报表（附各类剔除行数） | 2 |
| 单价为零.xlsx | 1 行单价为 0 | **正常生成报表**，同时提示「有 1 行单价 ≤ 0 已按规则保留，请与业务确认」 | 0 |

最后一份是特意留的对照组：题目只要求剔除数量 ≤ 0，没有要求剔除单价为 0。如果把它也当脏数据删掉，报表看起来「很干净」，但统计口径已经悄悄变了——**这类过度清洗比崩溃更难发现**。

### 1.6 独立核对：12 份 × 79 项 = 948 项全过

「统计值与人工核对一致」不能靠肉眼扫一遍。所以另写了 `verify_report.py`，从另一个方向验证：不复用 `make_report.py` 的任何统计函数，而是打开生成的报表重新算一遍，再和人工核算后固化的 `expected_stats.json` 逐项比对。

```bash
python verify_report.py out --expect data/expected_stats.json
# ===== 核对结束：12/12 通过 =====
# 每份报表：79 项检查，0 项失败
```

检查分五类：结构（四个表齐全、列名正确）、明细（行数与逐行金额）、分组（按销售员/按地区的行数、数量、金额）、一致性（四路合计相等）、对账（与人工核算值一致）。一年十二个月都不出错，比单看一个月更能说明清洗逻辑的稳定性。

---

## 二、考题二：FastAPI 记账接口

接口按三层组织：`models` 负责「什么样的数据合法」，`store` 负责「数据放哪儿、怎么保证并发安全」，`main` 负责「请求怎么进来、错误怎么出去」。合计 1212 行（有效代码 913 行）。

```text
C 模型层 models.py（Pydantic 校验，127 行）
      ↓
C 存储层 store.py（内存 + 线程锁，129 行）
      ↓
C 路由层 main.py（5 个端点 + 4 个全局异常处理器，200 行）
```

### 2.1 金额：对外 Decimal，对内整数分

```python
def decimal_to_cents(value: Decimal) -> int:
    return int((value * 100).to_integral_value(rounding=ROUND_HALF_UP))

def cents_to_decimal(cents: int) -> Decimal:
    return (Decimal(int(cents)) / 100).quantize(CENT)   # CENT = Decimal("0.01")
```

这样 `/summary` 的收入合计、支出合计、结余天然满足「收入 − 支出 = 结余」，不会出现 `300.3 - 128.5 = 171.79999999999998`。

校验上还做了一层严格性：金额必须为正、不超过 999,999,999.99、且最多两位小数。最后这条的实现是「取整后必须等于原值」——传 `12.345` 取到分是 `12.35`，与原值不等就直接判非法，比用正则去数字符串更可靠，因为在所有路径上都生效。

### 2.2 并发：一把锁，两个原则

验收要求是「并发写入不产生数据错误」。做法是三条：

1. **分配 ID + 追加记录**必须在临界区内作为一个整体完成，否则两个线程可能拿到同一个 ID；
2. 读取时复制一份快照，之后的过滤、排序、分页都在副本上做，不会边遍历边被修改；
3. 用 `threading.Lock` 而不是 `asyncio.Lock`——端点写成普通 `def`，FastAPI 会把它们放进线程池执行，它们是真的在多线程里跑。

```python
def add(self, payload: RecordCreate) -> RecordOut:
    with self._lock:                 # 临界区：分配 ID + 追加，二者必须原子
        record = _Record(
            id=self._next_id,
            amount_cents=decimal_to_cents(payload.amount),
            type=payload.type,
            category=payload.category,
            date=payload.date,
            created_at=dt.datetime.now(dt.timezone.utc),
        )
        self._next_id += 1
        self._items.append(record)
    return record.to_out()

def snapshot(self) -> List[_Record]:
    """复制一份不可变记录列表，调用方可在锁外安全遍历。"""
    with self._lock:
        return list(self._items)
```

方案的上限也写进了注释和 README：内存存储只在单进程内有效。如果 `uvicorn --workers 4` 起四个进程，每个进程各有一份列表，数据就不共享了——那时必须换 SQLite / PostgreSQL，并把「读-改-写」放进事务。

### 2.3 统一的结构化错误

错误来源很杂：Pydantic 的请求体校验、FastAPI 的查询参数校验、框架层的 404/405、业务代码自己抛的 400。如果每个接口各写各的，一定会有某个角落漏掉，返回的 JSON 形状对不上。所以用四个全局异常处理器把它们收敛成同一个形状：

```python
def error_response(status_code: int, code: str, message: str,
                   details: Optional[List[ErrorDetail]] = None) -> JSONResponse:
    body = ErrorBody(code=code, message=message, details=details or [])
    return JSONResponse(status_code=status_code,
                        content={"error": body.model_dump(mode="json")})

@app.exception_handler(RequestValidationError)
async def _handle_validation_error(_: Request, exc: RequestValidationError) -> JSONResponse:
    details = [_translate_error(err) for err in exc.errors()]
    return error_response(422, "VALIDATION_ERROR", "参数校验失败，请检查请求参数", details)
```

错误体固定为 `{"error": {"code", "message", "details"}}`：`code` 是给程序判断的稳定英文标识，`message` 是给用户看的中文摘要，`details` 是逐字段的细节。

| HTTP | code | 场景 |
|---|---|---|
| 422 | `VALIDATION_ERROR` | 请求体 / 查询参数校验失败 |
| 400 | `INVALID_DATE_RANGE` | `start_date` 晚于 `end_date` |
| 400 | `INVALID_PARAMETER` | 参数组合不合法 |
| 404 | `RECORD_NOT_FOUND` | 指定的流水不存在 |
| 404 | `NOT_FOUND` | 路径不存在 |
| 405 | `METHOD_NOT_ALLOWED` | 方法不支持 |
| 500 | `INTERNAL_ERROR` | 未预期异常 |

还有个容易被忽略的点：**把框架的英文报错翻译成中文**。Pydantic 默认返回 `Input should be greater than 0`，直接透出去等于没做本地化，所以加了一层 `_translate_error`，按错误类型映射成中文，并把字段名里的 `body`、`query` 前缀剥掉。

### 2.4 真实请求与响应

以下都是启动服务后实际抓下来的返回（只调整了换行）：

```http
POST /records        HTTP 201
{"id":1,"amount":128.5,"type":"支出","category":"餐饮","date":"2024-01-05",
 "created_at":"2026-11-22T06:43:06.928636Z"}
```

```http
POST /records        HTTP 422   （金额 = 0）
{"error":{"code":"VALIDATION_ERROR","message":"参数校验失败，请检查请求参数",
 "details":[{"field":"amount","message":"金额必须大于 0","type":"value_error"}]}}
```

```http
GET /summary?start_date=2024-01-01&end_date=2024-12-31      HTTP 200
{"start_date":"2024-01-01","end_date":"2024-12-31",
 "income":300.3,"expense":128.5,"balance":171.8,"record_count":2}
```

注意 `300.3 − 128.5 = 171.8` 是**精确相等**的，而不是 `171.79999999999998`——这正是内部用整数分累加的效果。另外，区间颠倒是「参数组合起来才不合法」，Pydantic 逐字段校验发现不了，所以放在路由函数里先判，返回 400 而不是 422，语义更准确。

Swagger UI 里的接口清单：

![FastAPI Swagger 文档](/myblog/images/lab1/05-swagger-docs.png)

*图 5：自动生成的 Swagger 文档——元信息、流水、统计三组接口，中文描述与统一错误模型*

### 2.5 测试：72 个用例 + 真并发压测

```text
........................................................................ [100%]
72 passed, 2 warnings in 46.17s
```

两个 warning 都不影响结论：一个是 Starlette 关于 httpx 用法的弃用提醒，另一个是 Windows 下写 `.pytest_cache` 被拒（目录权限问题），属于环境噪音。

并发用例特意**没有用 TestClient**：它走 ASGI 直连，请求在同一个线程里顺序完成，根本产生不了竞态。要真正验证锁是否有效，必须起一个真实的 uvicorn 服务，再用线程池并发打过去——16 线程 × 25 次 = 400 条写入，ID 无重复、无丢失；另有一组 6 写 + 4 读混合的用例验证读写交叉。

---

## 三、AI 协作：哪些交给它，哪些必须自己拍板

这次是「AI 写初版 + 人工审阅修改」的方式，全程固定五步走：**澄清需求 → 拆成小步 → 每步验证 → Code Review → 复盘**。

交给 AI 划算的部分，都是机械、可验证、错了立刻能发现的活：

- 样板代码：路由骨架、Pydantic 字段声明、openpyxl 的样式代码；
- 格式容错：日期多写法解析、千分位与全角逗号处理；
- 用例铺开：把「金额的十种非法取值」写成参数化列表，它想到的比我全（尤其是布尔和 NaN）；
- 文档骨架：README 结构、错误码矩阵。

必须自己拍板的部分，是「它没有依据判断」的部分：

| 决定点 | 我的选择 | 为什么 |
|---|---|---|
| 单价 ≤ 0 的行 | 保留并告警 | 需求只写了剔除数量 ≤ 0，扩大清洗范围会改变业务口径 |
| 去重前是否 trim | 文本先去首尾空格 | 否则 `" 张三"` 与 `"张三"` 是两条记录，去重形同虚设 |
| 四舍五入时机 | 先相乘再取到分 | 单价是业务原始值，只有成交金额需要落到分 |
| 日期区间 | 闭区间 | 月度报表与「指定日期区间」都按自然日理解 |
| 分页越界页 | 返回 200 + 空 items | 越界是正常翻页结果，不是错误，`total`、`pages` 照实返回 |
| 并发方案 | 内存 + 线程锁，写清边界 | 题目允许内存存储，先用最小方案把问题说透 |

---

## 四、踩过的四个坑

这四个都不是「想出来的」，是跑起来才暴露的：

1. **浮点误差只在合计处暴露。** 单行结果看着都对，直到把 97 行加起来和分组结果比对，才发现差 1 分钱。教训：涉及金额，第一行代码就该决定用整数分。
2. **PowerShell 下中文乱码。** 同样的脚本在 cmd 里正常、在 PowerShell 里变问号，原因是控制台编码不一致，运行前设 `PYTHONIOENCODING=utf-8` 即可。不影响逻辑，但会让人误以为程序出错。
3. **输出文件被 Excel 占用。** 报表还开着就重新运行，写入直接抛 `PermissionError`。把 WinError 32 翻译成「该文件可能正在 Excel/WPS 中打开」之后，问题立刻自解释。
4. **TestClient 测不出并发。** 并发用例最初用 TestClient 写，永远通过——因为请求在同一线程里顺序执行，锁有没有都一样。换成真起 uvicorn 再压，才验证到锁确实在起作用。**测试工具的适用边界，和被测代码一样需要审视。**

---

## 五、复盘

做完这两道题，最深的体会是：**Vibe Coding 的效率优势是真的，但它放大的恰恰是需求不清的代价。**

AI 能在几分钟内写出一个「看起来完整」的报表脚本，也能写出一个「看起来规范」的接口；但如果需求里的口径没定清楚，它会非常自信地把错误的口径实现得非常完整——而且因为代码整洁、注释齐全，这种错误比手写代码里的错误更难被察觉。

所以整个过程里，我花在「想清楚」上的时间明显多于花在「写」上的时间。把需求拆成可判定的验收项、把边界列成清单、把口径取舍写进文档，这些动作在纯手写时代是「好习惯」，在 Vibe Coding 时代变成了必要工序——因为代码的产出速度已经远快于人的审查速度，唯一还能控制质量的地方，就剩需求与验收这两端。
