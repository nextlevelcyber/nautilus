# NautilusTrader 作为加密团队回测基础设施的适配性评估

研究对象：本仓库 `develop` 分支（`version.json:4` 标注 `v2.0.0rc8`），以仓库内代码与文档为一手来源。
引用格式为仓库相对路径加行号。凡非源码或文档直接陈述、由作者推理得出的结论，均标注（推断）。

## 1. 结论摘要

**总体判断**：NautilusTrader 适合作为"执行敏感型策略"（做市、跨所套利的执行腿）的**回测引擎内核**，
但它刻意不提供平台层能力（UI、分布式编排、超参优化、数据质量平台），这些需要团队自建
（`ROADMAP.md:65-70`）。对因子研发与市场数据评估两类工作负载，它只能充当数据访问层或策略验证阶段，
不能替代向量化研究栈和数据平台（推断）。

**最重要的五项发现**：

1. **撮合保真度高，但只建模了订单命令的入站延迟。** `LatencyModel` 只有 insert/update/delete 三个回调
   （`crates/execution/src/models/latency.rs:31-75`），回测中仅作用于排队的交易命令
   （`crates/backtest/src/exchange.rs:909-966`）。行情到策略的延迟、交易所回报（ack/fill）到策略的延迟都没有模型；
   sandbox 文档明确写明只覆盖入站一腿（`docs/concepts/backtesting/execution-flow.md:121-122`）。
2. **历史盘口不可变，不建模自身冲击。** 模拟成交不会修改历史订单簿（`docs/concepts/backtesting/fill-prices-and-matching.md:3-5`）。
   `liquidity_consumption` 默认关闭，此时同一档挂单量可被多张模拟订单重复使用（`docs/concepts/backtesting/fill-models.md:31-36`）。
   "用户订单交互、持久化盘口变化"还列在路线图里（`ROADMAP.md:49`）。
3. **跨所资金划转、提币和链上结算没有建模**：每个场所各自拥有独立的起始余额与账户
   （`docs/concepts/backtesting/accounts-and-margin.md:36-61`）。回测 crate 中搜不到 transfer/withdraw/deposit 相关逻辑，
   只有 Rust 侧的 `SimulatedExchange::adjust_account(Money)`（`crates/backtest/src/exchange.rs:683-685`）
   和返回 `Money` 调整量的仿真模块（`docs/concepts/backtesting/simulation-modules.md:19-29`）可以用来近似。
4. **数据接入方面**：Databento、Tardis 有官方加载器，CoinAPI 也有加载器（beta）。**Pyth 没有任何适配器**，
   仓库中只有 Polymarket 自定义数据里的一个 `source` 字符串值 `pyth`（`crates/adapters/polymarket/src/data_types.rs:36-37`）。
   Pyth 数据需要做成 `CustomData`，而 `BacktestNode` 的数据配置拒绝自定义数据类型（`crates/backtest/src/config.rs:892-894`），
   因此只能走低层 `BacktestEngine`。
5. **发布状态与扩展方式**：v2 仍是 RC（`python/pyproject.toml:3`，classifier 为 Beta，见 `python/pyproject.toml:8`），
   每个 RC 都带大量破坏性变更（rc8 未发布版本的 Breaking Changes 段有 62 条，见 `RELEASES.md:41-105`）。
   并行回测只能靠多进程（`docs/concepts/architecture.md:712-729`），一个 `BacktestNode` 只接受一个运行配置
   （`crates/backtest/src/node.rs:356-363`）。

**各场景结论**：做市为较强，跨所套利为较强（仅限执行腿），延迟评估为部分，因子研发为弱，市场数据评估为弱。

## 2. 场景逐项核实

### 2.1 做市（Market making）

**NautilusTrader 提供的能力**

- **L2/L3 盘口回放**：场所 `book_type` 可选 `L1_MBP`、`L2_MBP`、`L3_MBO`（`docs/concepts/backtesting/data-and-venues.md:55-59`），
  `OrderBookDelta(s)` 在 L2/L3 下驱动盘口（`docs/concepts/backtesting/data-and-venues.md:71-78`）。
  `BacktestNode` 默认按 `F_LAST` 把增量合成完整事件后再撮合（`docs/concepts/backtesting/apis-and-runs.md:159-172`）。
- **队列位置跟踪**：需同时设置 `queue_position=True` 和 `trade_execution=True`（`docs/concepts/backtesting/trade-execution.md:115-138`）。
  订单受理时快照同价位的显示量，之后由对手方向的成交逐步消减（`docs/concepts/backtesting/trade-execution.md:171-184`）。
  L3 下按逐笔订单 ID 推进队列（`docs/concepts/backtesting/trade-execution.md:198-205`）；L1 下有基于报价的近似规则
  （`docs/concepts/backtesting/trade-execution.md:210-220`）。
- **流动性消耗**：`liquidity_consumption=True` 时按价位记录已消耗量，新的盘口更新到达后重置
  （`docs/concepts/backtesting/fill-prices-and-matching.md:153-155`）。
- **成交模型**：内置 11 种（`docs/concepts/backtesting/fill-models.md:38-52`），也支持 Python 自定义模型，
  可返回合成订单簿（`docs/concepts/backtesting/fill-models.md:130-160`）。
- **费用、保证金与资金费**：`fee_model` 必须显式配置（`crates/backtest/src/config.rs:288-294`）。
  `FundingRateUpdate` 在资金费结算点结算到持仓（`docs/concepts/backtesting/accounts-and-margin.md:6-32`）。
  支持强平（`crates/backtest/src/config.rs:361-367`）。
- **自有订单簿**：`OwnOrderBook` 跟踪自身挂单，可从公开盘口中扣除（`docs/concepts/order_book.md:184-198`）。
- **示例**：现成教程覆盖网格做市（`docs/tutorials/grid_market_maker_dydx.md:1-12`）与盘口回测
  （`docs/tutorials/backtest_orderbook_binance.py:1-9`）。

**缺失或受限之处**

- **不建模市场冲击与他人反应**：历史盘口不可变（`docs/concepts/backtesting/fill-prices-and-matching.md:125-129`），
  相关能力仍在路线图上（`ROADMAP.md:49`）。
- **队列跟踪的局限**：只支持 `LIMIT` 单，每张单独立估计，无法看到隐藏单（`docs/concepts/backtesting/trade-execution.md:222-227`）。
  `NO_AGGRESSOR` 成交会同时消减两侧队列，结果偏乐观（`docs/concepts/backtesting/trade-execution.md:229-232`）。
- **延迟只覆盖命令入站一腿**，详见 2.3 节。
- **大量成交累积到同一持仓时 v2 反而更慢**：4,000 笔成交累积持仓的场景，v2 比 v1 慢 63.9%
  （`crates/backtest/benches/BENCHMARKS.md:890`），剖析显示 64% 的 CPU 耗在完整 `Position` 克隆上
  （`crates/backtest/benches/BENCHMARKS.md:938`）。做市策略的成交笔数通常很多，这一点直接相关（推断）。
- **加密 CEX 很少提供 L3 数据**：Tardis 的 `book_change` 只映射为 `OrderBookDeltas`（`docs/integrations/tardis.md:44-46`），
  多数 CEX 只能做到 L2 队列估计（推断）。

**结论：较强。**

| 初步判断             | 核实结果 | 依据                                                                                                       |
| -------------------- | -------- | ---------------------------------------------------------------------------------------------------------- |
| L2/L3 增量回放       | 已证实   | `docs/concepts/backtesting/data-and-venues.md:55-78`                                                       |
| 队列位置跟踪         | 已证实   | `docs/concepts/backtesting/trade-execution.md:115-232`，但只支持 LIMIT，且依赖成交数据                     |
| `LatencyModel`       | 需修正   | 只延迟入站命令，不覆盖行情延迟和回报延迟（`crates/execution/src/models/latency.rs:31-75`）                 |
| 可配置成交模型       | 已证实   | `docs/concepts/backtesting/fill-models.md:38-160`                                                          |
| 保真度取决于数据粒度 | 已证实   | `docs/concepts/backtesting/data-and-venues.md:27-43`、`docs/concepts/backtesting/data-and-venues.md:80-81` |
| 适配度为"强"         | 需修正   | 下调为较强，原因是无自身冲击建模、延迟单腿、高成交量时的性能退化                                           |

### 2.2 跨所套利（Cross-venue arbitrage）

**NautilusTrader 提供的能力**

- **单引擎内多场所**：同一回测可以同时跑多个场所和多个策略（`docs/concepts/overview.md:37-40`），
  `BacktestRunConfig` 接受多个 `BacktestVenueConfig`（`docs/concepts/backtesting/apis-and-runs.md:96-103`）。
- **场所级配置**：`latency_model`、`fill_model`、`fee_model` 都是场所级字段（`crates/backtest/src/config.rs:288-298`），
  因此可以给每个场所配置不同的延迟。
- **资金费**：`FundingRateUpdate` 数据类型与结算机制见 `docs/concepts/backtesting/accounts-and-margin.md:6-32`。
  `BacktestDataConfig` 支持加载 `FundingRateUpdate`、`MarkPriceUpdate`、`IndexPriceUpdate`（`docs/concepts/data/catalog.md:243-246`）。
- **组合视图**：可以按场所和账户查询敞口与权益（`docs/concepts/portfolio.md:25-61`、`docs/concepts/portfolio.md:109-121`）。
- **同时间戳的级联下单**：例如在 `on_order_filled` 中对冲，命令可在同一时间戳内结算
  （`docs/concepts/backtesting/execution-flow.md:13-23`）。

**缺失或受限之处**

- **跨所划转、提币、充值、链上确认、结算周期均未建模**：`crates/backtest/src` 中搜不到 transfer/withdraw/deposit。
  每个场所的起始余额相互独立（`docs/concepts/backtesting/accounts-and-margin.md:36-61`）。
  Rust 侧的 `adjust_account`（`crates/backtest/src/exchange.rs:683-685`）没有出现在 Python `BacktestEngine` 的方法列表里
  （`crates/backtest/src/python/engine.rs:99-770`）。用自定义 `SimulationModule` 生成 `Money` 调整来近似划转是可行的
  （推断，依据 `docs/concepts/backtesting/simulation-modules.md:19-29`）。但模块只拿到单个场所的只读快照
  （`docs/concepts/backtesting/simulation-modules.md:39-51`），两腿划转要靠两个模块协同实现（推断）。
- **资金费数据的入库路径有缺口**：
  - Tardis Machine 回放写 catalog 时不写资金费、标记价和指数价（`docs/integrations/tardis.md:59-60`）。
  - Python `ParquetDataCatalog` 的类型化写入方法里没有资金费（`crates/persistence/src/python/backend/parquet/catalog.rs:232-515`），
    Python `StreamingWriter` 也不接受它（`crates/persistence/src/python/backend/writer.rs:141-189`）。
  - 可行路径有三条：
    - Tardis CSV 加载器 `load_tardis_funding_rates`（`crates/adapters/tardis/src/csv/load.rs:748`）。
    - 直接 `BacktestEngine.add_data` 加入数据（`crates/backtest/src/python/engine.rs:1615-1616`）。
    - Rust 侧写入，见 Rust 流式写入的资金费往返测试（`crates/persistence/src/backend/parquet/writer.rs:1147-1181`）。
- **没有现成的套利示例或教程**：在 `docs/`、`examples/` 中搜索 arbitrage、cross-venue，未找到。

**结论：较强**（仅限执行腿与价差、资金费收益；资金与库存约束层面为"部分"）。

| 初步判断               | 核实结果 | 依据                                                                                 |
| ---------------------- | -------- | ------------------------------------------------------------------------------------ |
| 单引擎多场所           | 已证实   | `docs/concepts/overview.md:40`、`docs/concepts/backtesting/apis-and-runs.md:96-103`  |
| 场所级延迟             | 已证实   | `crates/backtest/src/config.rs:298`；同样只覆盖入站命令                              |
| 资金费数据类型         | 已证实   | `docs/concepts/backtesting/accounts-and-margin.md:6-32`；Python 入库路径缺失写入方法 |
| 划转、提币、结算"未知" | 已否定   | 已确认未建模，见上文                                                                 |

### 2.3 延迟评估（Latency evaluation）

**NautilusTrader 提供的能力**

- **回测侧**：
  - 自定义 Python `LatencyModel` 按命令返回纳秒延迟，可以用 `command.client_order_id` 查表回放实测延迟
    （`docs/concepts/behavioral_models.md:89-137`）。
  - 到达时间为 `command.ts_init + delay`（`docs/concepts/behavioral_models.md:135-137`）。
  - 延迟命令按结算点释放（`docs/concepts/backtesting/execution-flow.md:97-112`）。
  - 因此可以做延迟参数扫描。
- **实盘侧时间戳**：
  - 数据与事件都带 `ts_event` 与 `ts_init`（`docs/concepts/data/index.md:453-477`、`docs/concepts/events/index.md:80-91`）。
  - 订单报表含 `ts_init` 与 `ts_last`（`docs/concepts/reports.md:63-64`），成交报表含 `ts_event` 与 `ts_init`（`docs/concepts/reports.md:114-115`）。
  - 文档说明 `ts_init - ts_event` 只有在时钟同步时才代表延迟（`docs/concepts/data/index.md:479-483`）。
- **Rust 实盘运行器指标**：`LiveNodeHandle::metrics_snapshot()` 提供分发速率、队列深度、利用率等
  （`docs/concepts/live.md:262-333`）。这些都是进程内指标，不含网络往返时间。
- **Sandbox**：真实行情加模拟撮合，可挂 `latency_model`（`docs/concepts/backtesting/execution-flow.md:114-142`）。
- **数据源时间戳**：Tardis 映射为 `ts_event = timestamp`（交易所时间）、`ts_init = local_timestamp`（采集端接收时间）
  （`crates/adapters/tardis/src/csv/load.rs:222-223`）。因此可以估计"交易所到采集端"的延迟，但仍有时钟偏差的问题。

**缺失或受限之处**

- **回测不建模行情延迟和回报延迟**：
  - 数据按 `ts_init` 稳定排序回放（`docs/concepts/data/index.md:487-492`），策略在同一时间戳内收到数据
    （`docs/concepts/backtesting/execution-flow.md:13-23`）。
  - 交易所生成的事件不延迟。sandbox 有明文说明（`docs/concepts/backtesting/execution-flow.md:121-122`）。
    回测侧的 `LatencyModel` trait 只有三个命令方法（`crates/execution/src/models/latency.rs:31-75`），
    据此推断回测中同样不延迟交易所回报（推断）。
- **Databento 丢失撮合引擎时间**：报价和成交类 schema 把 Databento 的 `ts_recv` 映射为 Nautilus 的 `ts_event`
  （`docs/integrations/databento.md:540-543`），原始撮合引擎时间 `ts_event` 和 `ts_in_delta` 不进入核心数据类型（推断）。
- **没有面向网络链路的延迟测量或导出工具**：搜索 `crates/**/Cargo.toml` 中的 prometheus、opentelemetry，未找到。
  `metrics_snapshot` 的 Python 暴露也未找到，仅在 `crates/live/src/python/node.rs:4377` 的测试中出现。

**结论：部分。**

| 初步判断                     | 核实结果 | 依据                                                                                     |
| ---------------------------- | -------- | ---------------------------------------------------------------------------------------- |
| 可扫描模拟延迟               | 已证实   | `docs/concepts/behavioral_models.md:89-137`                                              |
| 真实链路延迟属于实盘测量问题 | 已证实   | `docs/concepts/data/index.md:479-483`；没有专门的测量工具                                |
| 存在 `ts_event` 和 `ts_init` | 已证实   | `docs/concepts/data/index.md:453-511`                                                    |
| 延迟和指标工具               | 需修正   | 仅有 Rust 运行器进程内指标（`docs/concepts/live.md:262-333`），没有网络 RTT 或直方图工具 |

### 2.4 因子研发（Factor development）

**NautilusTrader 提供的能力**

- **流式指标**：41 个指标实现文件（`crates/indicators/src/{average,book,momentum,ratio,support,volatility}`），
  统一 `Indicator` trait，逐条更新（`crates/indicators/src/indicator.rs:28-69`）。
- **Catalog 查询到 DataFrame**：
  - `query_catalog` 返回 PyArrow、pandas 或 Polars（`python/nautilus_trader/persistence/catalog_to_df.py:62-110`）。
  - 查询支持 DataFusion SQL `where_clause` 与时间范围（`docs/concepts/data/catalog.md:375-398`）。
  - Catalog 可放在 S3、GCS 或 Azure（`docs/concepts/data/catalog.md:36-73`）。
- **自定义数据**：因子信号可以注册为 `CustomData`，写入 catalog 并路由给策略（`docs/concepts/custom_data.md:359-377`）。
- **绩效统计**：`crates/analysis/src/statistics` 下有 34 个统计量。

**缺失或受限之处**

- **引擎是事件驱动的**：没有向量化或横截面 API，也没有 IC、分组回测等因子评估功能，搜索未找到。
  项目自己把"向量化研究"定位为它要取代的另一套做法（`docs/concepts/overview.md:46-55`）。
- **`BacktestNode` 不能直接加载自定义数据（因子）**：配置校验见 `crates/backtest/src/config.rs:892-894`，
  迁移文档也有说明（`MIGRATION_V2.md:1245-1246`），只能用 `BacktestEngine.add_data`
  （`crates/backtest/src/python/engine.rs:1631-1632`）。
- **自定义统计量只能在 `BacktestEngine` 上注册**（`MIGRATION_V2.md:1247-1248`）。
- **Sharpe 默认按 252 天年化**（`crates/analysis/src/statistics/sharpe_ratio.rs:46-55`），7x24 的加密市场需要自行调整（推断）。
- **超参优化、AI/ML 工具明确不在项目范围内**（`ROADMAP.md:69`）。

**结论：弱**（作为因子研究引擎）。适合放在"因子到策略"的事件驱动验证阶段（推断）。

| 初步判断             | 核实结果 | 依据                                                                                              |
| -------------------- | -------- | ------------------------------------------------------------------------------------------------- |
| 事件驱动、非向量化   | 已证实   | `docs/concepts/overview.md:46-55`                                                                 |
| 存在指标             | 已证实   | `crates/indicators/src/indicator.rs:28-69`                                                        |
| 支持自定义数据       | 已证实   | `docs/concepts/custom_data.md:13-23`；但 `BacktestNode` 不支持                                    |
| Catalog 查询能力     | 需修正   | 比预想强：可直接输出 pandas/Polars（`python/nautilus_trader/persistence/catalog_to_df.py:62-73`） |
| 批量或向量化分析 API | 已证实   | 不存在，搜索未找到                                                                                |

### 2.5 市场数据评估（数据质量与场所数据评估）

**NautilusTrader 提供的能力（多为校验"原语"，而非工具）**

- **Catalog 覆盖区间**：`get_intervals`、`get_missing_intervals_for_request`、`query_first_timestamp`、`query_last_timestamp`
  （`python/nautilus_trader/persistence/__init__.pyi:387-408`、`crates/persistence/src/python/backend/parquet/catalog.rs:1359-1460`）。
- **写入保护**：默认拒绝时间重叠的写入（`docs/concepts/data/catalog.md:188-191`），有文件名修复与合并操作
  （`docs/concepts/data/catalog.md:410-470`）。
- **盘口完整性**：`book_check_integrity` 检查交叉盘口与层级约束，但需要显式调用（`docs/concepts/order_book.md:127-139`）。
  乱序增量会被应用而不是拒绝，只记录警告（`docs/concepts/order_book.md:145-160`）。
- **回放期校验**：
  - 精度不匹配时告警并跳过，连续 20 次后报错（`docs/concepts/backtesting/fill-prices-and-matching.md:212-225`）。
  - 增量事件缺少 `F_LAST` 时运行失败（`docs/concepts/backtesting/apis-and-runs.md:178-182`）。
  - Tardis 加载器会跳过首个快照之前的行（`docs/integrations/tardis.md:500-502`）。
- **微观结构指标**：`SpreadAnalyzer`（`crates/indicators/src/ratio/spread_analyzer.rs:36`）、
  `BookImbalanceRatio`（`crates/indicators/src/book/imbalance.rs:32`）。
- **盘口分析方法**：`get_avg_px_for_quantity`、`simulate_fills` 等（`docs/concepts/order_book.md:102-125`）。

**缺失或受限之处**

- 没有数据质量报告或作业，搜索未找到。覆盖区间基于文件名中的时间范围（`docs/concepts/data/catalog.md:157-161`），
  文件内部的缺口检测、序列号断档统计、跨场所一致性对比都需要自建（推断）。
- 时间时区处理有已知差异：DataFusion 时区库与核心库版本不一致（`docs/concepts/data/catalog.md:400-408`）。

**结论：弱。**

| 初步判断                   | 核实结果 | 依据                                                               |
| -------------------------- | -------- | ------------------------------------------------------------------ |
| 属于数据平台工作，适配度弱 | 已证实   | 只有校验原语，没有质量工具                                         |
| Parquet catalog            | 已证实   | `docs/concepts/data/catalog.md:1-18`                               |
| 存在数据校验或完整性检查   | 需修正   | 存在若干原语（盘口完整性、重叠写入、覆盖区间），但不是数据质量工具 |

## 3. 场景 × 问题 × 数据粒度 × 保真度矩阵

| 场景         | 回测能回答的问题                       | 所需数据粒度                                        | 保真度                                        | 回测无法回答的问题                               |
| ------------ | -------------------------------------- | --------------------------------------------------- | --------------------------------------------- | ------------------------------------------------ |
| 做市         | 报价策略的成交率、库存、价差收益和费用 | L2 增量加逐笔成交，最好有 L3（MBO）                 | L3 加成交时高；L2 为队列估计；L1 依赖概率模型 | 自身挂单对他人行为的影响、隐藏单、真实优先级规则 |
| 跨所套利     | 价差捕获、对冲腿滑点、资金费收益       | 各场所 L1/L2 加成交加资金费，时间轴统一             | 执行腿中高；资金层低                          | 划转、提币耗时与费用，交易所风控，各所时钟偏差   |
| 延迟评估     | 策略收益对下单延迟的敏感度             | 带 `ts_event` 和 `ts_init` 的逐笔数据加实测延迟分布 | 只覆盖入站命令                                | 行情延迟、回报延迟、真实链路抖动与丢包           |
| 因子研发     | 因子转为策略后的事件驱动表现           | Bar 或逐笔数据，因子以 `CustomData` 形式提供        | 对执行层高；对因子统计无支持                  | IC、衰减、横截面分组、大规模参数搜索             |
| 市场数据评估 | 数据能否回放、盘口是否完整、覆盖区间   | 原始逐笔或盘口数据                                  | 仅有原语                                      | 场所间数据一致性、序列缺口统计、数据供应商对比   |

补充说明：

- 数据粒度与 `book_type` 必须一致。Nautilus 不能从低粒度数据生成 L2/L3（`docs/concepts/backtesting/data-and-venues.md:80-81`）。
- 使用 Bar 数据时，无法确定 bar 内价格顺序、价差、深度和队列位置（`docs/concepts/backtesting/data-and-venues.md:41-43`）。
- 交易所打印的成交只能证明那一刻存在流动性（`docs/concepts/backtesting/fill-prices-and-matching.md:192-193`）。
- 成交模型与延迟模型都不决定实盘的成交（`docs/concepts/behavioral_models.md:36-38`）。

## 4. 数据接入

### 4.1 通用转换路径（自有 CEX 数据）

- **官方流程**：源数据加载器 → 规范化字段、时间戳、精度 → 构造模型对象 → 写入 catalog 或直接加入回测
  （`docs/concepts/data/index.md:596-618`）。
- **先写入合约定义再写行情**（`docs/integrations/databento.md:834-836`）。
  每次写入只能包含一个标识，并且按 `ts_init` 升序（`docs/concepts/data/catalog.md:150-151`）。
- **可用工具**：
  - 通用 CSV/Parquet 加载器 `CSVTickDataLoader`、`CSVBarDataLoader` 等，输出 pandas（`python/nautilus_trader/persistence/loaders.py:26-146`）。
  - Arrow IPC 转换器 `QuoteTickDataWrangler`、`OrderBookDeltaDataWrangler` 等，要求输入已经符合 Nautilus schema
    （`docs/concepts/data/index.md:547-560`）。
  - Binance 深度 CSV 加载器 `load_binance_order_book_deltas`，返回 DataFrame（`python/nautilus_trader/adapters/binance/__init__.pyi:515-517`）。
  - 端到端示例：外部 CSV 写入 catalog，再用 `BacktestNode` 运行（`docs/how_to/loading_external_data.py:1-176`）。
- **精度要求**：价格和数量必须与合约精度一致，否则会被跳过或拒绝
  （`docs/concepts/backtesting/fill-prices-and-matching.md:195-238`）。官方 wheel 默认为 128 位、16 位小数
  （`docs/getting_started/installation.md:442-451`）。
- **CoinAPI（beta）**：支持 S3 CSV flat file 的 Rust/Python 加载器（`docs/integrations/coinapi.md:50`，
  `python/nautilus_trader/adapters/coinapi/__init__.pyi:127-148`）。
- **CEX 适配器以实盘为主**：catalog 回写（`update_catalog`）只适用于"显式报告完成"的客户端，例如 Databento 或 Python 适配器
  （`docs/concepts/data/catalog.md:218-222`）。

### 4.2 Databento

- **加载**：`DatabentoDataLoader` 把 DBN 文件解码为 Nautilus 对象，覆盖 MBO、MBP-1、MBP-10、TBBO、OHLCV、DEFINITION、STATUS 等
  （`docs/integrations/databento.md:80-104`、`docs/integrations/databento.md:809-860`）。
- **建议一次性写入 catalog**：DataFusion 读取 Parquet 比每次解码 DBN 至少快一个数量级（`docs/integrations/databento.md:784-799`）。
- **与加密业务的关系**：Databento 是传统资产数据商（`docs/integrations/databento.md:1-10`），与加密业务相关的主要是
  CME 的 `GLBX.MDP3`（`docs/integrations/databento.md:197`）以及作为信号源的传统资产（推断）。
- **时间戳映射**：`ts_recv` 映射为 `ts_event`（`docs/integrations/databento.md:540-543`），对延迟研究有影响。

### 4.3 Tardis

- **CSV 加载与流式读取**：deltas、snapshot5、snapshot25、quotes、trades、funding rates、options chain
  （`crates/adapters/tardis/src/csv/load.rs:169-787`、`python/nautilus_trader/adapters/tardis/__init__.py:29-35`）。
- **Tardis Machine 回放写入 catalog**：`run_tardis_machine_replay`（`docs/integrations/tardis.md:11-12`），
  但不写入资金费、标记价和指数价（`docs/integrations/tardis.md:59-60`）。
- **不解析的格式**：`book_ticker`、`liquidation`（`docs/integrations/tardis.md:63`）。
- **其他限制**：`TardisDataClient` 不实现数据请求（`docs/integrations/tardis.md:854-858`）；
  不建议使用混合多个合约的 CSV（`docs/integrations/tardis.md:504-512`）。
- **场所覆盖**：`docs/integrations/tardis.md:113-163`。

### 4.4 Pyth

- **核实结果：没有 Pyth 适配器（已证实）。** 在 `crates`、`python`、`docs`、`examples` 中搜索 `pyth`，
  只有 Polymarket PolyBolt 数据类型的 `source` 字段说明（`crates/adapters/polymarket/src/data_types.rs:36-37`、
  `docs/integrations/polymarket.md:2353-2354`），以及 `.gitleaks.toml` 中的一条注释。
  官方适配器清单中也没有 Pyth（`ADAPTERS.md:80-98`、`docs/integrations/index.md:8-30`）。
- **可行路径（推断）**：
  - 把 Pyth 的价格、置信区间、发布时间等字段定义为 Python `CustomData`（`docs/concepts/custom_data.md:25-35`），
    写入 catalog 时需满足 schema 与 `ts_init` 升序的要求（`docs/concepts/data/index.md:699-706`）。
  - 回测时通过 `BacktestEngine.add_data` 加入数据（`crates/backtest/src/python/engine.rs:1631-1632`）。
    自定义数据只路由到数据引擎，不参与撮合（`crates/backtest/src/engine.rs:528-529`）。
  - 如果只需要价格，也可以映射到 `IndexPriceUpdate`，但该类型只有 `value` 一个价格字段
    （`docs/concepts/data/index_price_update.md:1-12`）。
- **新官方集成需要先走 RFC 流程**（`ROADMAP.md:94-115`）。

## 5. 风险与限制

### 5.1 发布状态

- v2 为 RC：`2.0.0rc8`（`python/pyproject.toml:3`），classifier 为 Beta（`python/pyproject.toml:8`），
  安装需要加 `--pre`（`MIGRATION_V2.md:22-27`）。
- RC 发布节奏：rc3 于 2026-08-20 发布（`RELEASES.md:1717`），rc7 于 2026-10-15 发布（`RELEASES.md:312`），
  rc8 尚未发布（`RELEASES.md:3`）。
- 破坏性变更多：rc7 有 43 条（`RELEASES.md:340-385`），rc8 有 62 条（`RELEASES.md:41-105`），
  其中包括 `LatencyModel` 签名、`BacktestVenueConfig` 参数顺序等回测接口（`RELEASES.md:41-105`）。
- v1 位于 `develop_v1`，切换后约三个月只接收关键安全补丁（`MIGRATION_V2.md:6-7`）。最后一个 v1 版本 1.231.0
  发布于 2026-08-02（`RELEASES.md:2018-2020`），据此推算 v1 维护期大约在 2026 年 11 月前后结束（推断）。

### 5.2 许可证

- 采用 LGPL-3.0-only（`LICENSE:1-2`、`Cargo.toml:56`、`python/pyproject.toml:21`）。
- 内部使用且不对外分发时，通常不触发源码公开义务；如果对外分发修改过的库，则需按 LGPL 提供修改后的源码
  （外部来源：<https://www.gnu.org/licenses/lgpl-3.0.en.html>）。这里的理解需要法务确认（推断）。
- 项目名称与商标使用受限（`TRADEMARK.md:15-25`），贡献代码需签署 CLA（`README.md:676-677`）。

### 5.3 UI 与平台

- 可视化输出是**自包含 HTML** 文件（Plotly，可交互），没有后端服务（`docs/concepts/visualization.md:3-27`）。
- UI、仪表盘明确不在项目范围内（`ROADMAP.md:67`）；路线图里的可视化工作也限定为"非完整 UI 仪表盘"（`ROADMAP.md:50`）。
- 分布式编排、超参优化同样不在范围内（`ROADMAP.md:68-69`）。
- 这些事项也不允许在官方渠道讨论（`ROADMAP.md:72-77`）。

### 5.4 性能

- **原生基准**：2 万事件的纯回放约 23 ms（`crates/backtest/benches/BENCHMARKS.md:127-132`）。
  测试负载是单场所、1 分钟 bar 加报价（`crates/backtest/benches/BENCHMARKS.md:15-22`）。
- **v2 与 v1 对比**：32 组对比中 v2 有 30 组更快（`crates/backtest/benches/BENCHMARKS.md:916-922`），
  例如 depth10 场景快 90.6%（`crates/backtest/benches/BENCHMARKS.md:884`）。
  但成交累积持仓的大规模场景 v2 更慢（`crates/backtest/benches/BENCHMARKS.md:890`）。
- **未测量的方面**：峰值内存、大规模 catalog 或流式负载、多场所场景（`crates/backtest/benches/BENCHMARKS.md:992-997`）。
  因此对于做市和跨所的真实负载，**没有现成的吞吐数据**。
- **Python 回调同步执行**，必须尽快返回（`docs/concepts/python.md:43-46`）。

### 5.5 并行与可复现性

- **并行**：同一进程内不能并发运行多个 node（`docs/concepts/architecture.md:712-729`、`docs/concepts/python.md:112-116`），
  一个 `BacktestNode` 只接受一个配置（`crates/backtest/src/node.rs:356-363`）。
- **复用已加载数据**：`reset()` 可以复用已加载的数据做参数扫描（`docs/concepts/backtesting/apis-and-runs.md:218-235`、
  `docs/concepts/backtesting/apis-and-runs.md:261-284`）。
- **确定性**：
  - 回放按 `ts_init` 稳定排序（`docs/concepts/data/index.md:487-492`），`TradeId` 是确定性生成的
    （`docs/concepts/backtesting/execution-flow.md:210-228`）。
  - 成交模型不设 `random_seed` 时使用操作系统随机源（`crates/execution/src/models/fill.rs:185-196`、`crates/execution/src/models/fill.rs:236-250`）。
  - 自定义 Python 模型的状态会跨 `reset()` 保留（`docs/concepts/behavioral_models.md:62-64`）。
  - 基准测试中，预加载与新加载两条路径的规范结果摘要一致（`crates/backtest/benches/BENCHMARKS.md:38-43`）。
  - DST 契约面向 madsim 运行时（`docs/concepts/dst.md:127-150`），对回测只约束场所和合约的迭代顺序（`docs/concepts/dst.md:412-422`）。

### 5.6 已知限制

- `BacktestNode` 不支持自定义数据、DataFrame 和下载引擎（`MIGRATION_V2.md:1245-1246`）。
- 不能为 `BacktestNode` 注册自定义统计量（`MIGRATION_V2.md:1247-1248`）。
- 同一个 catalog 实例上不能重叠查询和写入（`docs/concepts/data/catalog.md:153-155`）。
- v1 的 catalog 需要迁移后才能使用（`MIGRATION_V2.md:895-904`）。

## 6. 平台层需要自建的部分

| 模块              | 需要自建的内容                       | NautilusTrader 可依托的能力                                                                                                                                                                                                                             |
| ----------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 作业编排          | 任务队列、进程池或容器调度、失败重试 | 一进程一 node（`docs/concepts/architecture.md:728`）；`BacktestRunConfig` 作为声明式作业（`docs/concepts/backtesting/apis-and-runs.md:96-103`）                                                                                                         |
| 参数扫描          | 网格、随机或贝叶斯搜索，结果汇总     | `reset()` 复用已加载数据（`docs/concepts/backtesting/apis-and-runs.md:261-284`）；`change_fill_model`（`docs/concepts/backtesting/fill-models.md:132-133`）                                                                                             |
| 结果存储          | 指标库、实验追踪、运行元数据         | `BacktestResult` 的统计量与收益序列（`crates/backtest/src/python/result.rs:111-160`）；报表 DataFrame（`docs/concepts/reports.md:277-300`）；`StreamingConfig` 写入 catalog 并可用 `read_backtest` 读取（`docs/how_to/stream_parquet_catalog.md:1-34`） |
| Web 管理和后台 UI | 作业提交、结果浏览、权限、审计       | 嵌入 tearsheet HTML（`docs/concepts/visualization.md:15-17`）；`run_async()` 可挂载到 ASGI（`docs/concepts/python.md:50-60`）                                                                                                                           |
| 数据平台          | 入库流水线、数据质量作业、数据血缘   | S3 catalog（`docs/concepts/data/catalog.md:36-73`）；覆盖区间 API；`query_catalog`                                                                                                                                                                      |
| 场所建模扩展      | 划转、提币、交易所风控规则           | 自定义 `SimulationModule`（`docs/concepts/backtesting/simulation-modules.md:19-51`）                                                                                                                                                                    |
| 延迟数据采集      | 实盘延迟埋点、时钟同步、延迟分布库   | 订单和成交时间戳报表；自定义 `LatencyModel` 查表回放（`docs/concepts/behavioral_models.md:131-133`）                                                                                                                                                    |
| 可复现性          | 记录 seed、wheel 版本、配置哈希      | 成交模型 `random_seed`；确定性 `TradeId`                                                                                                                                                                                                                |

## 7. 待解决问题

1. **回测中交易所回报（ack/fill）到策略是否存在任何延迟。** 代码中只看到命令入站延迟，sandbox 文档明确说回报不延迟。
   回测侧没有找到同样明文的说明，需要读完 `crates/backtest/src/exchange.rs` 中的事件分发路径，或用最小用例实测确认。
2. **多场所、长时段 L2 回放的真实吞吐与内存占用。** 基准明确未覆盖这些（`crates/backtest/benches/BENCHMARKS.md:992-997`），需要用团队自己的数据实测。
3. **大量成交累积持仓的性能退化在 rc8 之后是否已修复。** `RELEASES.md` 中没有找到相关修复条目。
4. **Python 端如何把 `FundingRateUpdate` 写入 catalog。** 没有找到类型化写入方法，可能需要走 Rust 或等待新接口。
5. **v2 正式版（非 RC）的发布时间与 API 冻结时间点。** 仓库中没有日期，路线图只提到"稳定 API"目标（`ROADMAP.md:132-137`）。
   外部的 v2 路线图 issue（`MIGRATION_V2.md:1261`）没有核实。
6. **LGPL 在团队部署形态下的具体义务**，例如内部 SaaS 或给其他实体使用，需要法务意见。
7. **Pyth 数据的具体形态。** 无论是 Hermes 历史价格还是链上更新，都需要先确定字段与时间语义，才能设计 `CustomData` schema。
