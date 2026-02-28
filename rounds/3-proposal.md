## Project Brief
- **Original requirement**: 设计一个量化回测工具，支持 A 股、港股、美股三大市场，能够统一完成策略回测、结果评估与后续扩展。
- **Key constraints**: 需要兼容多市场交易规则差异；回测结果必须可复现；应支持从单机研究到多策略批量回测；优先保证回测正确性与数据偏差防护，再追求更高性能；MVP 阶段避免引入过重的平台化复杂度。
- **Excluded approaches**: 不采用仅支持单市场的简化方案；不把市场规则硬编码到策略层；不强制向量化与事件驱动共用同一套内部状态机；不在首版直接做生产级 OMS/EMS 或 Tick 级高频模拟。
- **Core decisions**: 采用“标准化数据层 + 双回测内核 + 统一结果契约 + 市场规则引擎”的架构；将 look-ahead bias、survivorship bias、防错时序建模纳入核心设计；以配置驱动承载大部分市场差异；MVP 继续使用 Python + Parquet + DuckDB + PostgreSQL，任务编排层明确降级到 Phase 2。

# 多市场量化回测工具设计方案 V2

## 1. 目标与边界

### 1.1 产品目标
构建一套面向研究和策略验证的量化回测工具，满足：

1. 支持 A 股、港股、美股的股票与 ETF 历史回测。
2. 支持日频与分钟频，且保证时间、费用、公司行为口径一致。
3. 支持统一策略接口，使同一策略可在不同市场复用。
4. 支持单次回测、参数扫描、结果归档与复现实验。
5. 支持多币种账户和统一净值折算。

### 1.2 首版明确边界
首版不做：

1. Tick 级撮合和盘口级市场冲击模拟。
2. 实盘下单、实时风控、OMS/EMS。
3. 期货、期权、融券、保证金融资。
4. 多租户权限体系和复杂集群调度。

## 2. 设计原则

### 2.1 正确性优先
系统必须先确保：

1. 没有 look-ahead bias。
2. 具备 survivorship bias 防护。
3. 账户、持仓、费用、公司行为处理可审计。
4. 任何回测结果可以通过版本快照复现。

### 2.2 架构原则
- 策略统一，市场差异隔离。
- 内核解耦，结果口径统一。
- 配置优先，代码兜底。
- MVP 轻量，后续按热点演进。

## 3. 总体架构

### 3.1 六层结构

1. 数据接入层（Data Connectors）
2. 数据标准化层（Canonical Data Model）
3. 数据访问控制层（Point-in-Time Data Access）
4. 回测执行层（Dual Engine Runtime）
5. 分析与报告层（Analytics & Reporting）
6. 任务编排与服务层（Orchestration & API, Phase 2+）

### 3.2 高层数据流

1. 外部数据源接入原始行情、主数据、公司行为、汇率、日历。
2. 数据标准化层将其转换为统一模型，并持久化为 Parquet + 元数据索引。
3. 数据访问控制层按回测时间点生成 point-in-time 视图，防止未来数据泄漏。
4. 回测执行层选择向量化引擎或事件驱动引擎执行。
5. 两类引擎都输出统一的 `BacktestResult` 契约。
6. 分析层基于 `BacktestResult` 生成指标、图表、报告。

### 3.3 组件交互原则
- 引擎内部状态允许不同实现。
- 分析层只依赖统一结果模型，不依赖引擎内部对象。
- 市场规则层在订单校验、费用结算、交收限制上被执行层调用。

## 4. 核心架构决策

### 4.1 双引擎，不共享内部状态机
V1 中“向量化与事件驱动共享账户模型”的描述过于粗糙，V2 明确修正：

1. 向量化引擎使用矩阵/列式数据结构，以目标仓位和批量换仓计算为主。
2. 事件驱动引擎使用显式订单、成交、持仓事件流，以精细撮合为主。
3. 两者不共享内部 `Account` 状态机，不强制共用同一条对象更新路径。
4. 两者必须输出同一格式的 `BacktestResult`，包括统一的 `fills`、`positions`、`cash_ledger`、`equity_curve`、`metrics_input`。

这样可以避免为了“统一”牺牲向量化性能，同时保持结果可比较、可复用。

### 4.2 统一结果契约
定义标准结果对象 `BacktestResult`：

- `run_metadata`
- `time_axis`
- `orders`
- `fills`
- `position_snapshots`
- `cash_ledger`
- `equity_curve`
- `drawdown_curve`
- `fx_snapshots`
- `market_breakdown`
- `warnings`
- `config_snapshot`

统一契约是双引擎对齐的唯一硬约束。

### 4.3 Phase 切分
- Phase 1：仅实现数据、规则、双引擎、分析层。
- Phase 2：增加任务编排、参数扫描、服务 API。
- Phase 3：增加分布式扩展、缓存复用、团队平台能力。

## 5. 数据接入与标准化

### 5.1 接入数据类型
- 历史行情：日线、分钟线
- 证券主数据：代码、市场、板块、上市状态
- 公司行为：现金分红、送股、拆股、合股、退市
- 交易日历：节假日、半日市、临时停牌
- 汇率：USD/CNY、HKD/CNY 及可扩展币对
- 基准和指数成分：用于基准比较与 survivorship bias 控制

### 5.2 标准化数据模型

#### Instrument
- `instrument_id`
- `symbol`
- `market` (`CN`, `HK`, `US`)
- `asset_type`
- `currency`
- `lot_size`
- `timezone`
- `listing_date`
- `delisting_date`
- `trade_status`

#### Bar
- `instrument_id`
- `timestamp_utc`
- `market_local_timestamp`
- `open`
- `high`
- `low`
- `close`
- `volume`
- `turnover`
- `suspend_flag`
- `adjustment_mode` (`raw`, `forward`, `backward`)

#### CorporateAction
- `action_id`
- `instrument_id`
- `action_type`
- `effective_date`
- `record_date`
- `cash_amount`
- `share_ratio`
- `split_ratio`

#### FXRate
- `base_currency`
- `quote_currency`
- `timestamp_utc`
- `rate`

#### IndexComposition
- `index_id`
- `instrument_id`
- `effective_from`
- `effective_to`

### 5.3 存储选型
- 元数据、任务、配置：PostgreSQL
- 历史行情与公司行为：Parquet
- 研究内查询与中间分析：DuckDB / Polars
- 报告与图表产物：对象存储或本地产物目录

### 5.4 数据加载与内存控制
为解决海量分钟线和跨市场数据的内存问题，V2 增加明确的数据加载策略：

1. 默认按日期块或月份块分片加载，不允许全量无界载入。
2. 向量化引擎优先依赖 DuckDB/Polars 进行列式扫描与过滤。
3. 事件驱动引擎使用流式迭代器按时间推进，只保留必要窗口。
4. 提供内存上限配置，超限时自动降级为 chunked 模式。
5. 对参数扫描重复使用的行情切片提供只读缓存句柄，避免重复反序列化。

## 6. 偏差防护与时间模型

### 6.1 Look-Ahead Bias 防护
这是 V2 的核心新增能力。

#### 数据访问规则
1. 任一时间点 `T`，策略只能访问 `<= T` 的数据。
2. 向量化引擎只暴露 expanding window / rolling window 视图，不允许策略拿到完整未来区间。
3. 事件驱动引擎的 `on_bar(context, data)` 仅注入当前 bar 与历史缓存。
4. 公司行为、指数成分、财务数据均按生效时间点注入。

#### 工程实现
- 引入 `PointInTimeView` 抽象，所有策略读数据必须走该接口。
- 对常见危险操作（一次性请求全周期因子矩阵）在研究模式下给出显式警告。

### 6.2 Survivorship Bias 防护

1. Universe 构建必须基于历史时点，而不是当前存续证券列表。
2. `Instrument` 使用 `listing_date` / `delisting_date` 控制证券在某日是否可交易。
3. 指数/板块策略通过 `IndexComposition` 历史表查询当日成分。
4. 默认在报告中标注 universe 是否启用 point-in-time 模式。

### 6.3 跨市场时间模型
三个市场不应被强行对齐到相同本地 bar。

设计方案：

1. 原始数据保留各市场本地时间。
2. 标准层统一额外存储 `timestamp_utc`。
3. 引擎调度以 UTC 时间轴推进。
4. 某市场未开盘时，该市场不会产生新 bar，但策略仍可读取“截至当前 UTC 时刻最近可用数据”。
5. 策略层通过 `context.get_latest(symbol)` 获取各标的最近有效快照，避免错误的横向同步。

这样能避免用美股收盘数据错误影响 A 股盘中决策。

## 7. 回测执行层

### 7.1 向量化引擎
适用场景：
- 日频选股
- 轮动
- 资产配置
- 参数扫描

特点：
- 输入是时间序列矩阵和目标权重信号。
- 通过再平衡逻辑生成归一化交易需求。
- 用简化成交假设快速计算资金曲线。

### 7.2 事件驱动引擎
适用场景：
- 分钟级择时
- 更复杂的订单生命周期
- 需要部分成交、挂单、撤单模拟

特点：
- 显式时间推进
- 订单状态机清晰
- 更适合表达细粒度交易约束

### 7.3 双引擎一致性策略
同一策略在两种模式下不要求逐笔完全一致，但必须满足：

1. 在相同成交假设、相同时间粒度下，结果应落在可接受误差范围内。
2. 差异必须能被解释为撮合精度、滑点模型或时间分辨率不同。
3. 报告层必须暴露 `engine_mode` 和 `matching_model`。

### 7.4 策略接口

#### 研究统一接口
- `initialize(context)`
- `before_trading(context)`
- `on_bar(context, data)`
- `after_trading(context)`

#### 扩展接口
- `on_order(context, order_event)`
- `on_fill(context, fill_event)`

向量化引擎可只实现研究统一接口；事件驱动引擎可额外启用扩展接口。

### 7.5 订单与撮合
订单类型：
- Market
- Limit
- TargetPosition
- TWAP/VWAP（仅在分钟级近似模拟）

撮合分层：

1. Level 1：下一根 bar 开盘/收盘成交。
2. Level 2：bar 区间成交 + 滑点。
3. Level 3：成交量约束 + 部分成交。

限制说明：
- TWAP/VWAP 在仅有分钟线时属于近似模拟，报告必须标注“非真实执行质量评估”。

## 8. 市场规则引擎

### 8.1 规则分层
大部分规则用配置表达，复杂规则用代码实现。

配置型规则：
- 费率
- 税率
- lot size
- 默认交收周期
- 节假日

代码型规则：
- A 股涨跌停
- A 股 T+1 卖出限制
- 港股多层费用叠加
- 特殊停牌/熔断处理

### 8.2 标准接口
定义 `MarketRuleProvider`：

- `validate_order(order, portfolio_state)`
- `normalize_quantity(order)`
- `price_limit_check(order, market_snapshot)`
- `calculate_fee(fill)`
- `calculate_tax(fill)`
- `settlement_rule(fill)`
- `can_sell_today(position, as_of_time)`
- `handle_suspension(order_or_position, as_of_time)`

### 8.3 各市场关键差异

#### A 股
- 股票默认 T+1 卖出
- 卖出收印花税
- 涨跌停限制显式校验
- 停牌较常见，挂单和估值必须有默认规则

#### 港股
- T+0
- 按手数下单
- 多项费用叠加，费率配置复杂
- 流动性分层明显，滑点应更保守

#### 美股
- T+0 视为可当日卖出
- 支持碎股可配置开关
- 无涨跌停，但需处理熔断/停牌事件
- 券商费用差异大，采用账户模板配置

## 9. 多币种账务模型

### 9.1 核心原则
账务设计不再只停留在“每日折算”描述，V2 明确为双层账务：

1. 原币种子账本：记录各币种真实现金、费用、已实现盈亏。
2. 组合主账本：按主币种（默认 CNY）汇总折算净值和风险指标。

### 9.2 记账流

#### 买入流程
1. 策略提交订单。
2. 若账户无足够目标币种现金，可按配置触发自动换汇。
3. 换汇生成一条 `FXLedgerEntry`，记录汇率、点差、换汇成本。
4. 成交后在原币种账本扣减现金，在持仓账本增加头寸。

#### 卖出流程
1. 成交后先在原币种账本记入卖出所得。
2. 税费、佣金也在原币种账本落账。
3. 是否自动换回主币种由组合配置决定。

### 9.3 P&L 口径
- 已实现盈亏：先按原币种计算，再按成交/结算汇率折算。
- 未实现盈亏：按最新市价与最新估值汇率折算。
- 报告同时输出原币种视图和主币种汇总视图。

## 10. 公司行为处理流水线

### 10.1 处理原则
公司行为必须在 point-in-time 框架下生效，不能事后整体重写账户。

### 10.2 支持的行为类型
- 现金分红
- 送股/配股
- 拆股
- 合股
- 退市/摘牌

### 10.3 处理流水线

1. 数据层存储原始公司行为事件。
2. 标准化层生成按证券排序的事件时间线。
3. 回测推进到生效日时，由 Corporate Action Processor 执行：
   - 现金分红：增加现金账本；
   - 送股/拆股：调整持仓数量与摊薄成本；
   - 合股：减少持仓数量并重算成本；
   - 退市：按规则转入现金清算或标记为无价值头寸。

### 10.4 复权策略
- 存储层保留原始价格。
- 研究层可按需查询前复权/后复权视图。
- 回测成交和账户核算基于原始价格 + 公司行为事件调整，而不是直接基于“已复权价格”偷懒结算。

这能降低价格连续性与真实账务口径不一致的风险。

## 11. 缺失数据、停牌与异常处理

### 11.1 停牌处理
- 停牌时默认不可新成交。
- 持仓估值默认使用最近可用成交价，并标注估值冻结。
- 未成交挂单默认继续挂起，用户可配置为自动撤单。

### 11.2 缺失数据处理
- 缺少 bar 时默认不前向填充成交价用于撮合。
- 指标计算可选择使用前值填充，但必须与撮合逻辑隔离。
- 数据质量问题写入 `warnings`，并进入报告摘要。

### 11.3 幂等与唯一性
- `run_id` 全局唯一。
- `order_id` 在 `run_id` 作用域内唯一，格式建议为 `run_id + sequence`。
- 参数扫描时各 run 的订单、成交、缓存目录完全隔离。

## 12. 分析与报告层

### 12.1 核心指标
- 累计收益、年化收益、最大回撤
- Sharpe、Sortino、Calmar
- 换手率、胜率、盈亏比
- 分市场收益拆分
- 分币种收益拆分
- 交易成本占比

### 12.2 可信度标记
报告中新增以下质量标记：

- `point_in_time_enabled`
- `survivorship_safe`
- `fx_mode`
- `corporate_action_mode`
- `execution_model`

这类元信息和收益指标一样重要，否则报告难以被正确解读。

## 13. 技术栈与演进

### 13.1 MVP 技术栈
- Python
- FastAPI（仅轻量查询/任务接口，可延后）
- PostgreSQL
- Parquet
- DuckDB / Polars
- Pandas / NumPy / Numba

### 13.2 调度选型
V2 调整建议如下：

1. Phase 1 不引入独立分布式调度框架。
2. 单机参数扫描先用本地多进程执行器。
3. Phase 2 再评估 Ray 作为参数扫描与分布式计算底座。
4. 暂不将 Celery/RQ 作为默认方案，只保留为轻量异步任务备选。

这样既吸收了“Celery 不适合大规模数据计算”的意见，也避免 MVP 过早引入复杂集群依赖。

### 13.3 UI 方案
- Phase 1 以 Python SDK + notebook + HTML 报告为主。
- 内部演示可优先采用 Streamlit 快速搭建分析面板。
- 如需团队级长期产品化，再评估 React 前端。

这比直接承诺前后端分离更符合首版投入产出比。

### 13.4 Build vs Extend 评估
首版建议自研核心抽象，但在局部借鉴成熟框架：

- `backtrader`：适合参考订单生命周期设计。
- `zipline-reloaded`：适合参考资产管线和时间序列处理。
- `vn.py`：适合参考国内市场接口与账户抽象。

最终仍建议自研，因为本项目的核心难点是三市场统一规则、双引擎并存和偏差防护，这些都很难无成本嫁接到现有单市场或单范式框架中。

## 14. 分阶段实施路线

### Phase 1: MVP（4-6 周）

1. 建立标准化数据模型与加载器。
2. 实现 point-in-time 数据访问。
3. 实现基础市场规则引擎（A/H/US）。
4. 实现向量化日频回测。
5. 实现统一 `BacktestResult` 与基础报告。

### Phase 1.5: 正确性增强（2-4 周）

1. 增加公司行为处理流水线。
2. 增加多币种账务与换汇流水。
3. 增加停牌/缺失数据默认策略。
4. 建立黄金样本回归测试。

### Phase 2: 扩展执行（4-8 周）

1. 增加分钟级事件驱动引擎。
2. 增加 Level 2 / Level 3 撮合模型。
3. 增加本地参数扫描。
4. 评估并接入 Ray。

### Phase 3: 平台化（按需）

1. 服务 API
2. 任务编排
3. 结果缓存去重
4. 多用户与审计

## 15. 测试与验证策略

### 15.1 回归测试
每个市场至少维护：

1. 费用测试样例
2. T+1 / T+0 规则样例
3. 涨跌停或停牌样例
4. 公司行为样例
5. 多币种折算样例

### 15.2 黄金样本
选取一组已知结果策略：
- 买入持有
- 定投
- 固定调仓
- 跨市场双资产轮动

每次引擎升级必须对比结果漂移。

## 16. 结论

V2 的关键修正是把“多市场支持”从单纯的接口兼容，提升为“正确性受控的统一回测体系”：

1. 用 point-in-time 数据访问杜绝未来函数。
2. 用历史 universe 和指数成分控制 survivorship bias。
3. 用双引擎分离内部状态、统一结果契约兼顾性能与一致性。
4. 用显式多币种账务和公司行为流水保障净值可信。
5. 用 Phase 切分避免首版范围失控。

按此方案推进，首版不仅能支持 A 股、港股、美股回测，而且能让研究结果具备足够的可解释性与工程可演进性。

## Changelog (V1 → V2)
| # | Reviewer Suggestion | Disposition | Rationale |
|---|---|---|---|
| 1 | Antigravity: 不要强制双引擎共享同一套底层账户状态机 | ✅ accepted | 改为“双引擎内部独立实现 + 统一 BacktestResult 契约”，避免为了统一牺牲向量化性能 |
| 2 | Antigravity: 为海量数据补充流式载入与内存控制方案 | ✅ accepted | 增加 chunked loading、流式迭代、内存上限和只读缓存句柄设计，这是可用性的必要前提 |
| 3 | Antigravity: 用 Ray/Dask 替代 Celery | ✅ accepted (phased) | V2 不再把 Celery/RQ 作为默认路线，Phase 2 优先评估 Ray；但 MVP 不直接引入分布式调度以控制复杂度 |
| 4 | Antigravity: 用 Streamlit/Dash 替代 React | ✅ accepted (scoped) | V2 调整为 Phase 1 优先 Python SDK + HTML 报告，内部面板优先 Streamlit，React 仅作为后续产品化选项 |
| 5 | Claude: 增加 look-ahead bias 防护 | ✅ accepted | 新增 PointInTimeView 与数据访问控制层，这是回测可信度基线 |
| 6 | Claude: 增加 survivorship bias 处理 | ✅ accepted | 新增 point-in-time universe 与 IndexComposition 历史成分设计 |
| 7 | Claude: 显式说明双引擎一致性边界 | ✅ accepted | 新增双引擎一致性策略，明确“不要求逐笔完全一致，但要求误差可解释” |
| 8 | Claude: 设计跨市场时间对齐机制 | ✅ accepted | 增加 UTC 调度 + 市场本地时间保留方案，避免跨市场时间错用 |
| 9 | Claude: 增加公司行为处理流水线 | ✅ accepted | 增加现金分红、送股、拆合股、退市的流水化处理与复权策略说明 |
| 10 | Claude: 明确停牌与缺失数据默认处理 | ✅ accepted | 增加停牌估值、挂单策略、缺失数据与 warnings 输出 |
| 11 | Claude: 将任务编排层标注为 Phase 2 | ✅ accepted | 在总体架构中明确第 6 层为 Phase 2+，缩小 MVP 范围 |
| 12 | Claude: 评估基于现有框架扩展 | ✅ accepted | 增加 build vs extend 评估，并给出仍建议自研的理由 |
