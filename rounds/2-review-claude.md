# Review — Claude (Reviewer)

## Review Summary
架构分层清晰、市场抽象方向正确，但在回测正确性保障（look-ahead bias、survivorship bias）和关键模块的详细设计（向量化/事件驱动统一、多币种记账流、公司行为处理）上存在明显空白，需要在 V2 中补齐。

## Dimension Reviews

### Architecture Soundness
六层架构整体合理，职责划分清晰。"统一回测内核 + 市场规则层"的核心抽象方向正确——策略不感知市场差异，规则由引擎注入。

优点：
- 数据标准化层将市场差异隔离在策略之外，正确。
- MarketRuleProvider 接口设计（validate_order / calculate_fee / can_sell_today 等）粒度合理。
- 向量化 + 事件驱动双模式并存是务实的选择。

问题：
- 双模式"共享账户模型"只提了一句，但这是架构中最难的部分之一。向量化模式产出目标权重/仓位，事件驱动模式逐笔处理订单，两者的账户状态如何对齐？这需要显式设计。
- 六层中 Layer F（任务编排）出现在总体架构里，但 Phase 1 不需要，建议在架构图中标注为 Phase 2 扩展层，避免给首版实现增加心智负担。
- 缺少数据流图。各层之间的调用关系和数据流向只能从文字推断，建议补充一张组件交互图。

### Technical Feasibility
技术选型（Python + Parquet + DuckDB + FastAPI）成熟可行，Rust/Go 预留升级路径务实。

问题：
- 多币种记账设计不够。§4.3 提到"基础记账币种 + 子账户原币种 + 每日汇率折算"，但没有说明记账流：买入港股时 CNY→HKD 的兑换是显式建模还是隐式折算？持仓市值和已实现 P&L 分别按什么币种计算？这直接影响净值曲线的正确性。
- 公司行为处理复杂度被低估。§5.1 定义了 CorporateAction 实体，但没有设计处理流水线：分红是现金分红还是再投资？拆股/合股后如何调整历史持仓成本？三个市场的除权除息规则差异很大（A 股登记日 vs 美股 ex-date），需要显式建模。
- Bar 模型（§5.1）缺少 `adj_factor` 或复权标记字段。复权逻辑是在数据层预处理还是在引擎运行时动态计算？这影响存储设计和计算性能。

### Missed Risks
以下关键风险在方案中完全未提及：

1. **Look-ahead bias 防护**：回测正确性的第一道防线。策略在时间 T 只能访问 T 及之前的数据，但方案没有讨论数据访问控制机制。向量化模式尤其容易引入 look-ahead（整个 DataFrame 一次性传入），需要显式设计点-in-time 数据视图。

2. **Survivorship bias**：方案未讨论已退市股票、历史指数成分变化的处理。如果回测 universe 只包含当前存续的股票，结果会系统性偏高。需要设计 point-in-time universe 机制。

3. **跨市场时间对齐**：A 股交易时段（UTC+8 09:30-15:00）、港股（UTC+8 09:30-16:00）、美股（UTC-5 09:30-16:00，有夏令时）。如果策略同时交易多个市场，bar 如何对齐？是按各市场本地时间独立驱动，还是统一到 UTC 时间轴？方案没有讨论。

4. **数据质量处理**：停牌时的 bar 是缺失还是填充前值？如果某只股票在回测中间段缺少数据，引擎如何处理？当前只有 `suspend_flag` 标记，缺少引擎层面的处理策略。

5. **订单 ID 生成与幂等性**：批量回测场景下如何保证订单 ID 全局唯一？参数扫描多个 run 并行时如何隔离？

### Alternative Approaches

1. **Build vs Extend**：方案选择从零构建。但 zipline-reloaded（Quantopian 遗产）和 backtrader 已有成熟的事件驱动引擎。可以评估是否在已有框架上扩展多市场层，能显著降低 Phase 1 工作量——尤其是账户模型、订单生命周期这类已被充分验证的部分。

2. **市场规则 DSL**：当前方案为每个市场写代码实现（validate_order 等）。大部分规则（费率、税率、最小交易单位）是纯数据，可以用声明式配置（YAML/JSON）+ 轻量解释器处理，只有少数复杂规则（涨跌停判定、港股多层费用）用代码。方案在 §6.3 提了配置驱动但没有上升到架构层面。

3. **时序数据库替代方案**：当前选择 PostgreSQL + Parquet + DuckDB 三件套。如果未来需要实时流入新数据并支持回测 + 监控双用，可以考虑 TimescaleDB（PostgreSQL 扩展），在保留 SQL 接口的同时获得时序优化，减少技术栈组件数。

## Specific Suggestions

### Suggestion 1: 补充 look-ahead bias 防护机制设计
- **Priority**: high
- **Issue**: 方案完全未讨论 look-ahead bias 防护，这是回测工具正确性的基石。
- **Suggestion**: 在 §4（核心模块设计）新增"数据访问控制"小节，设计 point-in-time 数据视图。向量化模式应使用 expanding window（而非全量 DataFrame）；事件驱动模式应确保 on_bar 回调只能访问当前及历史数据。
- **Rationale**: 无 look-ahead 防护的回测结果不可信，会导致策略上线后表现远逊于回测。

### Suggestion 2: 补充 survivorship bias 处理机制
- **Priority**: high
- **Issue**: 未讨论退市股票和历史指数成分的处理。
- **Suggestion**: 在 §5 或 §6 新增"Point-in-time Universe"设计。Instrument 表需要包含 listing_date 和 delisting_date（已有），但还需要设计 IndexComposition 历史表，支持按日期查询历史成分股。
- **Rationale**: 不处理 survivorship bias，选股类策略的回测结果会系统性虚高 2-5% 年化。

### Suggestion 3: 显式设计向量化/事件驱动模式的账户统一方案
- **Priority**: high
- **Issue**: §3.2C 提到"二者共享账户模型"但未给出设计。
- **Suggestion**: 增加一个小节说明：(a) 向量化模式的目标仓位如何转化为订单序列；(b) 两种模式的 Fill → Position → Account 更新是否走同一条路径；(c) 同一策略用两种模式回测时结果是否应完全一致，还是允许在撮合精度上有可接受差异。
- **Rationale**: 这是架构中最容易出现口径不一致的地方。如果两种模式对同一策略产生不同结果，用户将无法信任任何一种。

### Suggestion 4: 设计跨市场时间对齐机制
- **Priority**: medium
- **Issue**: 三个市场在不同时区和交易时段运行，方案未讨论 bar 对齐策略。
- **Suggestion**: 在 §6 新增"跨市场时间模型"小节。建议方案：每个市场独立维护本地时间轴的 bar 序列，回测引擎按 UTC 统一调度，策略通过 context 分别访问各市场最新可用 bar。不要试图对齐不同市场的 bar 到同一时间戳。
- **Rationale**: 错误的时间对齐会引入隐蔽的 look-ahead bias（如用美股收盘数据影响 A 股盘中决策）。

### Suggestion 5: 设计公司行为处理流水线
- **Priority**: medium
- **Issue**: CorporateAction 只定义了实体，缺少处理流水线设计。
- **Suggestion**: 在 §4 或 §5 新增"Corporate Action Processing"小节，至少覆盖：(a) 现金分红 vs 股票分红的账户处理；(b) 拆股/合股后历史持仓成本和数量的调整；(c) 复权因子的计算时机（预处理 vs 运行时）；(d) 三个市场除权除息规则差异的显式对比表。
- **Rationale**: 公司行为处理不正确会导致价格序列断裂和 P&L 计算错误，而且这类 bug 非常隐蔽。

### Suggestion 6: 明确缺失数据/停牌的引擎处理策略
- **Priority**: medium
- **Issue**: Bar 模型有 suspend_flag 但引擎如何处理未定义。
- **Suggestion**: 定义引擎对以下情况的默认行为：(a) 停牌时的持仓估值（用最后成交价 vs 标记为不可交易）；(b) 数据缺失时是跳过该 bar 还是填充；(c) 停牌期间的挂单如何处理（自动撤单 vs 挂至复牌）。
- **Rationale**: A 股停牌现象远比美股常见，不处理会导致回测中大量隐性错误。

### Suggestion 7: 将任务编排层标注为 Phase 2
- **Priority**: low
- **Issue**: Layer F（任务编排与服务层）出现在总体架构中，但 Phase 1 不需要。
- **Suggestion**: 在 §3.1 的六层列表中将第 6 层标注为"Phase 2+"，在 §3.2F 开头加一句"本层在 Phase 1 不实现，列于此处用于架构预留"。
- **Rationale**: 减少首版实现的范围蔓延风险，让团队聚焦核心引擎正确性。

### Suggestion 8: 评估基于已有框架扩展的可行性
- **Priority**: low
- **Issue**: 方案选择完全自研，但市面上有成熟的开源回测框架。
- **Suggestion**: 在 §7 或单独一节，简要评估 zipline-reloaded / backtrader / vnpy 作为基础框架的优劣，给出选择自研的显式理由（或考虑部分复用）。
- **Rationale**: 如果已有框架能覆盖 60-70% 的需求，自研的 ROI 需要被论证。即使最终选择自研，这个评估也能增强方案的说服力。
