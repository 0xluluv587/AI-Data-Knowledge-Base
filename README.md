# AI-Data-Knowledge-Base[AI 金融数据分析知识库]
本项目旨在构建金融领域AI数据分析的专业知识库，便于AI学习特定领域知识并提供专业回答。当AI回答与基金数据提取、量化策略表现和风险指标分析相关问题时，应首先参考本文档，基于数据结构理解和指标计算标准给出准确回答。如果遇到需要计算本文档里未列出的指标，回答时应注明“此指标未在参考文档里找到公式，仅供参考“

## 📊 目录

1. [数据结构与应用场景](#1-数据结构与应用场景)
2. [基金业绩与风险指标](#2-基金业绩与风险指标)
3. [基金组合(FoF)策略指标](#3-基金组合fof策略指标)
4. [交易执行分析](#4-交易执行分析)

## 1. 数据结构与应用场景 Context for every questio

### 数据库结构
- **服务器**: mcp服务器用于访问starrocks数据库
- **主要数据表**:

  - `fund_nav` 业务系统存储的基金费前净值表，存储产品的费前净值(NAV before fee)和费后净值(NAV after fee)历史数据，包括日常净值、年化收益等信息。是基金产品价值评估和收益计算的核心数据表。字段描述：

| 字段名 | 类型 | 描述 |
|--------|------|------|
| id | int64 | 主键ID |
| created_at | int64 | 创建时间（毫秒时间戳） |
| updated_at | int64 | 更新时间（毫秒时间戳） |
| rcu_id | int64 | 账户ID，关联到RCU账户 |
| day_timestamp | int64 | 日期时间戳，表示该天UTC 0点的时间戳（毫秒） |
| nav | decimal(38,18) | 净值，某天的申购净值/申购价格 |
| before_fee_nav | decimal(38,18) | 费前净值，风控费前的净值 |
| amended_nav | decimal(38,18) | 修正净值，对原始净值进行修正后的值 |
| product_id | varchar(64) | 产品ID |
| time_stamp | int64 | 净值时间戳，精确到净值更新的具体时间点 |
| is_daily_first | tinyint | 是否是当天第一条有效净值（1:是，2:否） |
| product_name | varchar(128) | 产品名称 |
| currency | varchar(128) | 产品币种 |
| snapshot_id | varchar(64) | 快照ID |
| apy_return | decimal(38,18) | 年化收益率 |
| reported_nav | decimal(38,18) | 已报告净值 |
| aum | decimal(38,18) | 资产管理规模（Assets Under Management） |
| is_back_test_data | tinyint | 是否是回测数据（1:是，2:否） |
| has_apy | tinyint | 是否有年化收益率（1:是，2:否） |
| has_return | tinyint | 是否有收益率（1:是，2:否） |
| status | int32 | 净值状态（1:正常，2:异常，3:自动计算关闭，4:延迟显示，5:隐藏） |
| is_settled_nav | tinyint | 是否是结算净值（1:是，2:否），用于业绩提取点 |
| shares | decimal(38,18) | 总份额（劣后份额+优先份额） |
| update_mode | int32 | 更新模式（1:系统更新，2:人工更新） |
| is_adjust | tinyint | 是否需要净值修改（1:需要，2:不需要） |
| apy_days | int64 | APY计算天数 |
| risk_is_suspicious | tinyint | 风控是否可疑 |

  - `product_statistical_data` 产品统计数据表，该表数据会根据净值实时进行更新，记录产品的各种收益率、波动率、人气值等统计数据。字段描述：

| 字段名 | 类型 | 描述 |
|--------|------|------|
| id | int64 | 主键ID |
| created_at | int64 | 创建时间（毫秒时间戳） |
| updated_at | int64 | 更新时间（毫秒时间戳） |
| product_id | varchar(64) | 产品ID，唯一索引 |
| volatility | decimal(38,18) | 波动率，净值更新后同步更新 |
| holdings | int | 持有该产品的人数 |
| x_day_apy | decimal(38,18) | X天APY，X为产品配置的天数 |
| apy_7d | decimal(38,18) | 7天年化收益率 |
| apy_30d | decimal(38,18) | 30天年化收益率 |
| apy_60d | decimal(38,18) | 60天年化收益率 |
| apy_90d | decimal(38,18) | 90天年化收益率 |
| apy_180d | decimal(38,18) | 180天年化收益率 |
| apy1year | decimal(38,18) | 1年年化收益率 |
| inception_to_date_apy | decimal(38,18) | 成立至今年化收益率 |
| apy_ytd | decimal(38,18) | 年初至今年化收益率 |
| return_7d | decimal(38,18) | 7天收益率 |
| return_30d | decimal(38,18) | 30天收益率 |
| return_60d | decimal(38,18) | 60天收益率 |
| return_90d | decimal(38,18) | 90天收益率 |
| return_180d | decimal(38,18) | 180天收益率 |
| return1year | decimal(38,18) | 1年收益率 |
| return_since_inception | decimal(38,18) | 成立至今收益率 |
| ytd_return | decimal(38,18) | 年初至今收益率 |
| maximum_drawdown_7d | decimal(38,18) | 7天最大回撤 |
| maximum_drawdown_30d | decimal(38,18) | 30天最大回撤 |
| maximum_drawdown_60d | decimal(38,18) | 60天最大回撤 |
| maximum_drawdown_90d | decimal(38,18) | 90天最大回撤 |
| maximum_drawdown_180d | decimal(38,18) | 180天最大回撤 |
| maximum_drawdown_1y | decimal(38,18) | 1年最大回撤 |
| maximum_drawdown_ytd | decimal(38,18) | 年初至今最大回撤 |
| maximum_since_inception | decimal(38,18) | 成立至今最大回撤 |
| popularity_value_7d | decimal(38,18) | 7天人气值 |
| popularity_value_30d | decimal(38,18) | 30天人气值 |
| popularity_value_60d | decimal(38,18) | 60天人气值 |
| popularity_value_90d | decimal(38,18) | 90天人气值 |
| popularity_value_180d | decimal(38,18) | 180天人气值 |
| popularity_value_1y | decimal(38,18) | 1年人气值 |
| popularity_value_inception | decimal(38,18) | 成立至今人气值 |
| popularity_value_ytd | decimal(38,18) | 年初至今人气值 |
| quick_tag_ids | varchar(256) | 快速标签ID |
| since_inception_days | int | 成立以来的天数 |
| running_days | int | 产品运行天数 |

  - `product` 产品基础信息表，存储产品的核心信息，如名称、币种、策略、风险等级等。是基金管理系统中的核心表之一。字段描述：

| 字段名 | 类型 | 描述 |
|--------|------|------|
| id | int64 | 主键ID |
| created_at | int64 | 创建时间（毫秒时间戳） |
| updated_at | int64 | 更新时间（毫秒时间戳） |
| deleted_at | datetime | 删除时间，用于软删除 |
| product_id | varchar(64) | 产品ID，唯一索引 |
| product_code | varchar(20) | 产品代码 |
| product_name | varchar(128) | 产品名称 |
| currency | varchar(128) | 产品币种 |
| strategy_id | varchar(64) | 策略ID |
| status | tinyint | 产品状态（1:未上架，2:已上架） |
| is_stop_raise | tinyint | 是否停止募集（1:是，2:否） |
| risk_level | tinyint | 风险等级（1-5级） |
| btc_perf_start_time | int64 | BTC性能开始时间 |
| nav_start_time | int64 | 净值开始时间 |
| listing_time | int64 | 上架时间 |
| is_customized | tinyint | 是否定制化策略（1:是，2:否） |
| is_private_strategy | tinyint | 是否私有策略（1:是，2:否） |
| rcu_id | int64 | 账户ID |
| pb_user_id | varchar(64) | PB用户ID |
| rcs_id | int64 | RCS系统ID |
| ex_account_ids | text | 交易所账户ID列表 |
| fund_manager_id | varchar(64) | 基金经理ID |
| weight | int64 | 权重，用于排序 |
| apy_display_rule | tinyint | APY显示规则（0:成立至今APY，1:预期APY，2:X天APY） |
| number_of_x_day | int | APY计算天数，当display_rule=2时使用 |
| expected_apy | decimal(38,18) | 预期年化收益率 |
| bitcoin_performance | tinyint | 是否显示比特币表现（1:是，2:否） |
| min_subscription_amount | decimal(38,18) | 最低认购金额 |
| user_daily_subscription_limit | decimal(38,18) | 用户每日认购限额 |
| user_subscription_limit | decimal(38,18) | 用户认购总限额 |
| is_launchpad | tinyint | 是否为首发产品（1:是，2:否） |
| min_redemption_shares | decimal(38,18) | 最低赎回份额 |

  - `product_label` 产品标签表，用于存储产品的各种标签信息，如风险类型、投资方向等，帮助用户更好地筛选产品。字段描述：

| 字段名 | 类型 | 描述 |
|--------|------|------|
| id | int64 | 主键ID |
| created_at | int64 | 创建时间（毫秒时间戳） |
| updated_at | int64 | 更新时间（毫秒时间戳） |
| deleted_at | datetime | 删除时间，用于软删除 |
| product_id | varchar(64) | 产品ID，索引 |
| product_label_id | varchar(64) | 产品标签ID，唯一索引 |

  - `product_content` 产品内容表，存储产品的详细描述信息，如策略说明、团队背景、交易场所等多语言内容，丰富产品的展示信息。字段描述：

| 字段名 | 类型 | 描述 |
|--------|------|------|
| id | int64 | 主键ID |
| created_at | int64 | 创建时间（毫秒时间戳） |
| updated_at | int64 | 更新时间（毫秒时间戳） |
| deleted_at | datetime | 删除时间，用于软删除 |
| product_id | varchar(64) | 产品ID，唯一索引 |
| team_background_url | text | 团队背景图片URL |
| broker_page_url | text | 经纪商页面URL |
| performance_display | tinyint | 表现显示类型（1:追踪回报，2:每日收益） |
| performance_display_pnl | tinyint | 是否显示盈亏（1:是，2:否） |
| private_strategy_visibility | tinyint | 私有策略可见性（0:仅代码持有者可见，1:完全可见） |
| currency_positions_analysis | uint32 | 货币持仓分析（位掩码字段） |
| trade_history_positions_analysis | uint32 | 交易历史持仓分析（位掩码字段） |
| investment_history | uint32 | 投资历史（位掩码字段） |
| personal_profile_display | tinyint | 个人简介显示（1:显示，2:不显示） |
| broker_page_redirect | tinyint | 经纪商页面重定向（1:重定向，2:不重定向） |

  - `product_strategy` 产品策略表，定义不同的投资策略类型及其特性，为产品提供策略分类和规则。字段描述：

| 字段名 | 类型 | 描述 |
|--------|------|------|
| id | int64 | 主键ID |
| created_at | int64 | 创建时间（毫秒时间戳） |
| updated_at | int64 | 更新时间（毫秒时间戳） |
| deleted_at | datetime | 删除时间，用于软删除 |
| strategy_id | varchar(64) | 策略ID，唯一索引 |
| name | varchar(128) | 策略名称 |
| weight | int64 | 权重，用于排序 |
| picture | text | 策略图片URL |
| nav_abnormal_ratio | decimal(38,18) | 净值异常比例 |
| up_nav_abnormal_ratio | decimal(38,18) | 上升净值异常比例 |
| down_nav_abnormal_ratio | decimal(38,18) | 下降净值异常比例 |
| is_migrate | tinyint | 是否已迁移（0:否，1:是） |

  - `i18n` 国际化表，存储系统中需要多语言支持的文本内容，如产品名称、描述等，支持系统的国际化和本地化。字段描述：

| 字段名 | 类型 | 描述 |
|--------|------|------|
| id | int64 | 主键ID |
| created_at | int64 | 创建时间（毫秒时间戳） |
| updated_at | int64 | 更新时间（毫秒时间戳） |
| entity_id | varchar(64) | 实体ID，如产品ID、策略ID等 |
| locale_type | uint32 | 本地化类型，标识文本的用途 |
| language | varchar(10) | 语言代码，如zh-CN, en-US等 |
| text | text | 翻译文本内容 |

  - `riskmgt.risk_fund_strategies_info_draft` 包含risk业务系统里策略的小时级费前数据，通常提取天级别数据时应参考每日UTC 00:00的数据(如有）；也包含基金的标签信息
  - `riskmgt.rcu_info_draft` 包含基金名称等信息
    - 主键: rcu_id
    - 主要字段: name, quotes, unit, portfolio_assets(json类型), platform_list(数组类型), label_list(数组类型)
    - 说明:
      - portfolio_assets是json类型，元素格式为"总用户投资资产信息"，如{"USDT": "342061.8490047697274239523487"}，查询时需使用GET_JSON_STRING(portfolio_assets, quotes)
      - platform_list是数组类型，元素格式为"交易所名"，如["okex", "binance"]，查询时需使用以下sql语句：select rcu_id, unnest from riskmgt.rcu_info_draft, unnest(platform_list) as unnest 或可以使用ARRAY_CONTAINS(label_list, '交易所名')
      - label_list是数组类型，元素格式为"基金label信息"，如["public", "absolute_value"]，查询时需使用以下sql语句：select rcu_id, unnest from riskmgt.rcu_info_draft, unnest(label_list) as unnest或可以使用ARRAY_CONTAINS(label_list, 'label名')
  - `riskmgt.risk_fund_strategies_info_draft`
    - 主键: rcu_id + _time
    - 时间字段: _time (注意：不是date)
    - 主要字段: name, quotes, adjusted_nav, gross_leverage_ratio, nav, total_asset_value, portfolio_assets(json类型), platform_list(数组类型), label_list(数组类型)
    - 说明: portfolio_assets(json类型), platform_list(数组类型), label_list(数组类型) 的sql查询方式与 riskmgt.rcu_info_draft 一致
    - 查询示例：查询Long Short策略：
<pre> SELECT * FROM riskmgt.risk_fund_strategies_info_draft WHERE ARRAY_CONTAINS(label_list, 'longshort') OR name LIKE '%Long Short%' OR name LIKE '%多空%' </pre>
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



- **典型应用场景举例**:
- 1) 查询特定类型的基金/策略时需要关联多表进行查询，比如要查询多空类策略：
  select rcus.* from rcus, rcu_label, rcu_to_rcu_label where rcus.id = rcu_to_rcu_label.rcu_id and rcu_to_rcu_label.rcu_label_id = rcu_label.id and rcu_label.`name` = "longshort"
  using lable_list in risk_fund_strategies_info_draft to determining whether a strategy is public or private.
  **策略类型标签映射表**
  | 策略类型 | 可能的标签值 | 常见命名模式 |
  |---------|------------|------------|
  | 多空策略 | "longshort", "Long Short" | product_strategy 或 Lable中包含"多空"、"Long Short"、"long-short" |
  | 套利策略 | "arbitrage", "Arbitrage" |  product_strategy 或 Lable中包含"套利"、"活期套利"、"Arbitrage" |
  | CTA策略 | "CTA", "trend", "cta" |  product_strategy 或 Lable中包含"CTA"、"趋势" |
  | 做市策略 | "market_making", "mm" |  product_strategy 或 Lable中包含"MM"、"做市"、"Market Making" |
  | 主观交易策略 | "Discretionary Trading", "Discretionary" |  product_strategy 或 Lable中包含"主观交易"、"主观"、"Discretionary" |
  | 混合型策略 | "Hybrid" |  product_strategy 或 Lable中包含"Hybrid"、"混合"、"混合型" |
  | 增强型指数策略 | "Enhanced Index", "Index" |  product_strategy 或 Lable中包含"Enhanced"、"指数"、"Index" |
  | 期权策略 | "Options Strategy", "Options" |  product_strategy 或 Lable中包含"Options"、"期权" |
  
- 2) 基于业务系统数据查询某个策略的每日有效NAV数据案例：
  > select fn.* from ( select *, ROW_NUMBER() OVER (PARTITION BY product_id ORDER BY time_stamp DESC) as row_num from fund_two.fund_nav fn where is_daily_first = 1 and status = 1 and not (upper(fn.product_name) like '%TEST%' or fn.product_name like '%测试%') ) fn left join fund_two.product p on fn.product_id = p.product_id where row_num = 1 and p.status != 3
  > 如果要分析费前NAV数据，通常业务系统中的费前 NAV 数据，因为有标记正常/异常、异常阈值的拦截，比riskmgt.risk_fund_strategies_info_draft里的数据更准确可靠一些。必要的情况可以可以两份数据对比参考。
  

- 3) 统计[xxx 指标]从好到差排名N名以内的某类策略的表现和报告：
  > 可优先查看 `product_statistical_data` 产品统计数据表里是否有现成的指标，比如“30天收益率前10名套利策略”，如果有的话直接用业务系统计算好的数据进行排名，效率更高，可以避免重新全量策略便利的工作量。
  > 确定了要分析的策略列表后，再根据需求内容和数据库中的数据进行具体分析
  

## 2. 基金业绩与风险指标
> 如无特别说明，以下指标默认基于业务系统`fund_nav`里 UTC+0 0点的有效费后净值(NAV after fee)计算；
> 而如果提示词里指明希望看策略的费前表现，应该基于业务系统`fund_nav`里 UTC+0 0点的有效的休正后费前净值(amended_nav)计算；
> 有效费后NAV和amended_nav都可参考“典型应用场景举例”中的“2)基于业务系统数据查询某个策略的每日有效NAV数据案例”)

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
| 7天、30天、60天、90天、180天、1年、成立以来 | σ_T = stdev(最近T天的DLR) * sqrt(365) |

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
- 标准统计: 1天、7天、30天、60天、90天、180天、1年、成立以来
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

- **默认时间范围**: 1天、7天、30天、60天、90天、180天、1年、成立以来
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
