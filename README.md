# AI-Data-Knowledge-Base[AI 金融数据分析知识库]
本项目旨在构建金融领域AI数据分析的专业知识库，便于AI学习特定领域知识并提供专业回答。当AI回答与基金数据提取、量化策略表现和风险指标分析相关问题时，应首先参考本文档，基于数据结构理解和指标计算标准给出准确回答。如果遇到需要计算本文档里未列出的指标，回答时应注明“此指标未在参考文档里找到公式，仅供参考”

## 📊 目录

1. [数据结构与应用场景](#1-数据结构与应用场景)
2. [基金业绩与风险指标](#2-基金业绩与风险指标)
3. [基金组合(FoF)策略指标](#3-基金组合fof策略指标)
4. [交易执行分析](#4-交易执行分析)

## 1. 数据结构与应用场景 Context for every questio

### 数据库结构
- **服务器**: mcp服务器用于访问starrocks数据库
- **主要数据表**:
  - `riskmgt.risk_fund_strategies_info_draft` 包含基金的小时级数据，通常提取天级别数据时应参考每日UTC 00:00的数据(如有）；也包含基金的标签信息
  - `rcu_info_draft` 包含基金名称等信息
  - `rcu_lable` 包含基金标签等信息
- **关键字段**:
  - `adjusted_nav`: 基金净值，也是考虑各种费用、分润和异常情况下的调整后费前净值，用于还原策略的真实表现和盈利能力。
  - risk_fund_strategies.`total_asset_value`:基金的AUM，单位是此基金的计价币种。
  - `gross_leverage_ratio`: 基金杠杆率
  - `rcu_info_draft.name`: 对应rcus.id的基金名称
  - `rcu_lable`,`lable_list`: 用于判断策略是否为公开策略、具体的策略类型等，主要的基金类型包括：
    - 套利策略(Arbitrage/Agile Arbitrage)
    - 主观交易策略(Discretionary Trading)
    - CTA趋势策略(CTA)
    - 混合型策略(Hybrid Strategy)
- using lable_list in risk_fund_strategies_info_draft to determining whether a strategy is public. 
- 查询特定类型的基金/策略时需要关联多表进行查询，比如要查询多空类策略：
  select rcus.* from rcus, rcu_label, rcu_to_rcu_label where rcus.id = rcu_to_rcu_label.rcu_id and rcu_to_rcu_label.rcu_label_id = rcu_label.id and rcu_label.`name` = "longshort"
  **策略类型标签映射表**
  | 策略类型 | 可能的标签值 | 常见命名模式 |
  |---------|------------|------------|
  | 多空策略 | "longshort", "Long Short" | 包含"多空"、"Long Short"、"long-short" |
  | 套利策略 | "arbitrage", "Arbitrage" | 包含"套利"、"Arbitrage" |
  | CTA策略 | "CTA", "trend", "cta" | 包含"CTA"、"趋势" |
  | 做市策略 | "market_making", "mm" | 包含"MM"、"做市"、"Market Making" |

## 2. 基金业绩与风险指标
> 以下指标基于UTC+0 0点的daily NAV计算
### 收益率指标 Return

| 指标 | 计算公式 |
|------|---------|
| 成立以来收益率(Return Since inception) | NAV_T / NAV_1 - 1 |
| 特定时间段收益率(7天/1月/3月/6月/1年) | NAV_T / NAV_T-X - 1 |
| 指定区间收益率 | NAV_end / NAV_begin - 1 |
| 日均收益率(Average Daily Return, ADR) | average(DR_i : DR_i-X+1) |
| 预期年化收益率(Expected Annualized Return) | (1+ADR)^365 - 1 |

**说明**:
- 日收益率(DR_i) = NAV_T / NAV_T-1 - 1 (当T=1时不计算)
- 日对数收益率(DLR_i) = ln(NAV_T / NAV_T-1) (当T=1时不计算)
- 预期年化收益率与日均收益率的时间区间应与收益率指标一致

### 波动率指标(Volatility)

> 运行超过7天后开始计算波动率

| 时间区间 | 计算公式 |
|---------|---------|
| 成立以来/7天/30天/90天/1年 | σ_T = stdev(最近T天的DLR) * sqrt(365) |

**说明**: 当RCU的DLR为0时，计算σ时不需跳过该点

### 夏普比率(Sharpe Ratio)

| 时间区间 | 计算公式 |
|---------|---------|
| 成立以来/7天/30天/90天/1年 | [avg(最近T天的DLR) * 365 - Rf] / [stdev(最近T天的DLR) * sqrt(365)] |

**说明**:
- 标准公式: Sharpe Ratio = E(Ra-Rf) / σ
- 无风险利率Rf默认为0，如需考虑其他值应询问客户
- E(Ra) = avg(最近T天的DLR) * 365
- σ为年化后的波动率

### 索提诺比率(Sortino Ratio)

| 计算公式 | 说明 |
|---------|------|
| E(Ra-Rf) / σd | σd为下行标准差，仅考虑收益率低于无风险利率的部分 |

**处理步骤**:
1. 样本处理: 若DLR < Rf则保留DLR值，否则记为0
2. 年化处理: E(Ra) = avg(最近T天的DLR) * 365, σd = 计算出的标准差 * sqrt(365)
3. 分母n为总收益率个数，非仅小于Rf的收益率个数

### 最大回撤(Max Drawdown)

| 步骤 | 计算方法 |
|------|---------|
| 1 | 记录UTC+0 0:00的NAV |
| 2 | 计算每日回撤: drawdown_T = if [NAV_T / max(NAV_1, NAV_2,...NAV_T-1) - 1 < 0, NAV_T / max(NAV_1, NAV_2,...NAV_T-1) - 1, 0] (T≥2) |
| 3 | 最大回撤(MDD) = abs(min(drawdown_T)) |

### 恢复天数指标(Recover Days)

- **最大恢复天数**: 计算连续为负的drawdown_T的最大天数
- **平均恢复天数**: 连续为负的drawdown_T的平均天数

### 日回撤(daily drawdown)
**日回撤** = abs(NAV_T / NAV_T-1 - 1)

## 3. 基金组合(FoF)策略指标

### 时间加权收益率(TWR)

| 计算公式 | R_twr = (1+R_1)(1+R_2)(1+R_3)...(1+R_n) - 1 |
|---------|-------------------------------------------|

**说明**:
- R_i = (期末资金-(期初资金+流入资金+流出资金)) / (期初资金+流入资金)
- 期初资金 = 当前份额 * NAV (8点的份额和NAV)
- 流入/流出资金 = 流入资金 + 流出资金
  - 流入资金 = 申购金额(含申购手续费)
  - 流出资金 = -[赎回金额(剔除赎回手续费)]
- 流入/流出按照[操作确认时间]统计，使用[确认日期的NAV]计算
- 操作确认时间按UTC+0 00:00:00~23:59:59统计

### 年化收益率(基于TWR)

| 计算公式 | 说明 |
|---------|------|
| (1+R_twr)^(365/周期天数) - 1 | R_twr和周期天数必须来自同一时间窗口 |

**时间窗口选择**:
- 标准统计: 7天、30天、90天、180天、1年
- 今年以来: 最新更新日期 - 1月1日
- 投资以来:
  - 有持有中订单: 最新更新日期 - 第一笔申购订单NAV确认日期(剔除取消订单)
  - 无持有中订单: 最后一笔赎回订单NAV确认日期 - 第一笔申购订单NAV确认日期(剔除取消订单)

## 4. 交易执行分析

> 所有指标均区分期权(category = "option")和非期权(category ≠ "option")

### 换手率指标 (Turnover)

| 指标 | 计算公式 |
|------|---------|
| 换手率(Turnover) | 交易量 / 平均AUM |
| 日均换手率(Daily Turnover) | 换手率 / 区间天数 |

**说明**: 区间天数 = 结束日期 - 开始日期

### Taker/Maker 比率

| 计算公式 | 特殊情况 |
|---------|---------|
| Taker交易量 / Maker交易量 | 当Maker交易量=0:<br>- 若Taker≠0，比率为∞<br>- 若Taker=0，比率为0 |

### 滑点与交易费用

| 指标 | 计算公式 |
|------|---------|
| 平均滑点/交易量 | 总滑点 / 交易量 |
| 平均交易费用/交易量 | 总交易费用 / 交易量 |

**汇总维度**:
- 按交易所(venue)
- 按交易账户(exaccountid)
- 按产品类型(product type)
- 按Taker/Maker
- 按标的资产(underlying)
- 以上维度均细分至8小时级别

### 时间范围选择

- **默认时间范围**: 1天、7天、30天、90天(滚动展示，对应图表)
- **自定义时间范围**: 前端选择时间范围和统计时间节点，区间为[开始时间，结束时间)，对应表格

## 4. 常见问题与解决方案
1. **Q: 无法找到特定标签的策略**
   A: 尝试多种筛选方式:
   - 检查标签拼写和大小写
   - 结合名称关键词筛选
   - 检查标签格式，如ARRAY_CONTAINS需要使用双引号

2. **Q: 收益率计算结果异常**
   A: 检查:
   - 净值数据是否为0或异常值(与前后对比偏差超过30%的值)
   - 时间区间是否有完整数据
   - 考虑使用阈值筛选异常结果(如±50%以上的短期收益)
---

*注意: 本文档未列出的指标，AI在回答时应注明"此指标未在参考文档里找到公式，仅供参考"*
