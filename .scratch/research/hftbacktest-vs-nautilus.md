# hftbacktest 与 NautilusTrader 对比：加密团队回测基础设施选型

研究对象与引用约定：

- **hftbacktest**：本地浅克隆 `/home/user/nextlevelcyber/hftbacktest`（远端为 fork `nextlevelcyber/hftbacktest`），
  以仓库内代码、文档、notebook 为一手来源。引用写作 `hftbacktest:<仓库相对路径>:<行号>`；
  notebook 的行号是 `.ipynb` JSON 文件中的行号。
- **NautilusTrader**：以已完成的基线评估 `.scratch/research/nautilus-backtest-infra-fit.md` 为准，
  写作"基线 §x"。只有在对比需要时才重新打开源码，写作 `nautilus:<路径>:<行号>`。
- 凡非源码或文档直接陈述、由作者推理得出的结论，标注（推断）。未找到的内容写"未找到"并说明搜索范围。
- 评级沿用基线：强 / 较强 / 部分 / 弱。

## 1. 结论摘要

**两者定位不同，不是同一层级的替代品。** hftbacktest 是一个"单机、tick 级、延迟与队列敏感"的微观结构回测器，
附带最小化的实盘机器人；NautilusTrader 是一个"多场所、带账户与组合、回测与实盘同构"的交易引擎。
团队的五类工作负载分别落在两者的强项上，没有一方能单独覆盖。

**逐场景胜出方**：

- **做市：hftbacktest 胜（强 vs 较强）。** 决定性差异是延迟建模：hftbacktest 用 `exch_ts`/`local_ts` 两个时间戳把同一份行情
  分别喂给"交易所侧"和"本地侧"，从而建模行情延迟，并把下单延迟拆成入站和回报两腿
  （`hftbacktest:hftbacktest/src/backtest/models/latency.rs:13-20`），还能按历史实测延迟插值。
  Nautilus 只延迟入站命令（基线 §2.1）。队列模型方面 hftbacktest 提供 6 种 L2 模型（1 种保守模型加 5 种概率模型）和 L3 FIFO，可调参以贴合实盘。
- **跨所套利：Nautilus 胜（较强 vs 部分）。** hftbacktest 的引擎支持多资产多交易所同场回测，但每个资产的账户状态相互独立、
  余额从 0 起算，没有保证金、资金费结算、跨所余额（`hftbacktest:hftbacktest/src/backtest/state.rs:22-46`），
  官方示例也全部是"单资产 + 多进程"。Nautilus 有多场所账户、资金费结算和组合视图（基线 §2.2）。
- **延迟评估：hftbacktest 明显胜（强 vs 部分）。** 行情延迟、入站延迟、回报延迟三者都可建模、可替换为实测数据、可平移扫描，
  并有"用行情延迟合成下单延迟"的工具（`hftbacktest:py-hftbacktest/hftbacktest/data/utils/feed_order_latency.py:45-99`）。
- **因子研发：两者都弱。** 都没有向量化或横截面因子 API。hftbacktest 的 IC 与 `[时间 × 品种]` 矩阵研究只存在于标注
  "DRAFT VERSION" 的教程 notebook 中（`hftbacktest:examples/Pricing Framework.ipynb:12`），不是库功能。
- **市场数据评估：hftbacktest 略胜（部分 vs 弱）。** 它自带采集器，会记录本地接收时间并在序列断档时补拉快照，
  还有时间戳校正与事件排序校验工具。但两者都没有数据质量报告，场所一致性对比、供应商对比都要自建。

**总体建议（推断）**：现实答案是**组合使用，但以 Nautilus 为主平台**。

- Nautilus 承担多场所策略、组合与账户、跨所套利、实盘部署，以及团队共享的数据目录（catalog）。
- hftbacktest 作为"微观结构实验室"，专门用于做市报价与队列研究、延迟敏感性评估、行情时序质量评估。
- 不建议以 hftbacktest 作为唯一平台：它的实盘路径只对 Binance Futures 做过测试网验证
  （`hftbacktest:connector/README.md:8-15`），账务用 `f64` 且没有组合层，核心代码由单一作者维护（见 §3.10）。
- 组合的代价是两套数据格式（npz 事件数组 vs Parquet catalog）和两套策略代码（numba 函数 vs Nautilus 策略类），
  策略不能直接移植。因此组合方案的关键投入是**统一的原始数据层**：保留交易所时间与本地接收时间两个时间戳，
  再分别生成两种回测格式（见 §5）。

## 2. 逐场景对比表

| 场景         | Nautilus                    | hftbacktest | 决定性差异                                                                                                                                                                                                                                                                                                                                                                                                                                                      | 更适合                                     |
| ------------ | --------------------------- | ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| 做市         | 较强（基线 §2.1）           | 强          | hftbacktest 交易所侧按 `exch_ts`、本地侧按 `local_ts` 消费同一事件（`hftbacktest:hftbacktest/src/backtest/proc/nopartialfillexchange.rs:525-527`、`hftbacktest:hftbacktest/src/backtest/proc/local.rs:284-286`），下单延迟分入站与回报两腿（`hftbacktest:hftbacktest/src/backtest/order.rs:94`、`hftbacktest:hftbacktest/src/backtest/order.rs:143`）；Nautilus 只延迟入站命令（`nautilus:crates/execution/src/models/latency.rs:31-75`）。两者都不建模自身冲击 | hftbacktest                                |
| 跨所套利     | 较强，仅执行腿（基线 §2.2） | 部分        | hftbacktest 每个资产独立 `StateValues`，无保证金、资金费、跨所余额（`hftbacktest:hftbacktest/src/backtest/state.rs:22-46`），无跨市场示例（`hftbacktest:ROADMAP.md:68`）；Nautilus 有场所级账户、资金费结算与组合敞口（基线 §2.2）                                                                                                                                                                                                                              | Nautilus；信号时序研究可借助 hftbacktest   |
| 延迟评估     | 部分（基线 §2.3）           | 强          | hftbacktest 的 `IntpOrderLatency` 按实测请求、撮合、回报三时间戳插值（`hftbacktest:hftbacktest/src/backtest/models/latency.rs:57-273`），`latency_offset` 可平移行情延迟（`hftbacktest:hftbacktest/src/backtest/data/reader.rs:466-494`）；Nautilus 无行情与回报延迟（基线 §2.3）                                                                                                                                                                               | hftbacktest                                |
| 因子研发     | 弱（基线 §2.4）             | 弱          | 两者都无因子库 API；hftbacktest 的 IC 计算是 notebook 内手写函数（`hftbacktest:examples/Pricing Framework.ipynb:2153`）；Nautilus 可把 catalog 查询为 pandas/Polars（基线 §2.4）                                                                                                                                                                                                                                                                                | 两者都不适合，另建研究栈                   |
| 市场数据评估 | 弱（基线 §2.5）             | 部分        | hftbacktest 采集器按 `pu` 检测断档并补拉快照（`hftbacktest:collector/src/binancefuturesum/mod.rs:33-73`），有时间戳校正与排序校验（`hftbacktest:py-hftbacktest/hftbacktest/data/validation.py:15-152`）；Nautilus 只有 catalog 覆盖区间与盘口完整性等原语（基线 §2.5）                                                                                                                                                                                          | hftbacktest（仅限 HFT 行情的时序质量评估） |

## 3. 关键维度对比

### 3.1 订单簿与队列建模

- **盘口重建**：两者都支持 L2（按价位）与 L3（逐笔订单）。hftbacktest 有 `HashMapMarketDepth`、`BTreeMarketDepth`、
  `ROIVectorMarketDepth` 与 `FuseMarketDepth`。其中 HashMap 实现在丢包时能"自然刷新"最优价，
  BTree 实现在丢包时可能出现交叉盘（`hftbacktest:hftbacktest/src/depth/hashmapmarketdepth.rs:10-20`、
  `hftbacktest:hftbacktest/src/depth/btreemarketdepth.rs:18-22`）。Nautilus 见基线 §2.1。
- **L2 队列模型**：hftbacktest 提供 `RiskAdverseQueueModel` 与 `ProbQueueModel`。后者配 5 种概率函数（Power、Power2、Power3、Log、Log2），
  可调参数 `n` 用于贴合实盘（`hftbacktest:hftbacktest/src/backtest/models/queue.rs:42-330`）。
  Nautilus 只有一种确定性规则：受理时快照显示量，之后由对手方成交消减（基线 §2.1）。
- **L3**：hftbacktest 有 `L3FIFOQueueModel`，它明确假设所有交易所都按价格-时间 FIFO 撮合
  （`hftbacktest:hftbacktest/src/backtest/models/queue.rs:473-479`），且 Python 侧不允许 L3 与部分成交交易所组合
  （`hftbacktest:py-hftbacktest/src/lib.rs:520-525`）。Nautilus 的 L3 按订单 ID 推进队列（基线 §2.1）。
- **自身冲击**：两者都不建模。hftbacktest 文档明确说明"你的订单不能改变模拟市场"
  （`hftbacktest:docs/order_fill.rst:8-11`）；L3 下回测订单只进入队列模型，不写入盘口（推断，依据
  `hftbacktest:hftbacktest/src/backtest/models/queue.rs:624` 中 `_depth` 未被使用）。Nautilus 见基线 §1 第 2 条。
- **小结**：做市队列研究 hftbacktest 更灵活（多模型、可校准）；Nautilus 有可选的 `liquidity_consumption`，
  在"同一档被多张模拟单重复消耗"问题上更保守（基线 §1 第 2 条）。

### 3.2 延迟建模（行情 vs 下单，入站 vs 回报）

| 延迟分量                   | Nautilus                                   | hftbacktest                                                                                                                                                      |
| -------------------------- | ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 行情延迟（交易所到本地）   | 不建模，数据按 `ts_init` 回放（基线 §2.3） | 建模：同一事件在交易所侧按 `exch_ts`、本地侧按 `local_ts` 生效（`hftbacktest:docs/latency_models.rst:12-16`）                                                    |
| 下单入站（本地到撮合）     | `LatencyModel` 三个命令回调（基线 §2.3）   | `LatencyModel::entry`（`hftbacktest:hftbacktest/src/backtest/models/latency.rs:15-16`）                                                                          |
| 回报（撮合到本地，含成交） | 不建模（基线 §2.3）                        | `LatencyModel::response`，成交回报同样受此延迟（`hftbacktest:docs/latency_models.rst:22-25`）                                                                    |
| 实测延迟回放               | 自定义 Python 模型按订单查表（基线 §2.3）  | `IntpOrderLatency` 按 `req_ts`/`exch_ts`/`resp_ts` 线性插值（`hftbacktest:hftbacktest/src/backtest/models/latency.rs:57-273`）                                   |
| 交易所拒单                 | 未找到                                     | `exch_ts <= 0` 视为拒单，返回负延迟（`hftbacktest:hftbacktest/src/backtest/models/latency.rs:195-215`）                                                          |
| 按下单、改单、撤单区分     | 三个回调可分别返回（基线 §2.3）            | 不支持，路线图未完成（`hftbacktest:ROADMAP.md:22`）；回调能拿到 `&Order`，Rust 自定义模型理论上可区分（推断）                                                    |
| 采集点平移                 | 未找到                                     | 行情 `latency_offset`（`hftbacktest:py-hftbacktest/src/lib.rs:169-179`）；下单延迟同样可平移（`hftbacktest:hftbacktest/src/backtest/models/latency.rs:276-295`） |

两个共同限制：

- hftbacktest 假设请求与回报保持顺序，不模拟 REST 请求乱序到达（`hftbacktest:hftbacktest/src/backtest/order.rs:30-33`）。
- hftbacktest 的 Python 侧只能在 `ConstantLatency` 与 `IntpOrderLatency` 两种模型中选择
  （`hftbacktest:py-hftbacktest/src/lib.rs:72-81`）。自定义模型只能用 Rust 写，或者在 Python 里生成延迟数组后交给
  `IntpOrderLatency`（`hftbacktest:py-hftbacktest/src/lib.rs:310`）。

### 3.3 多场所

- **Nautilus**：一个引擎内多场所、多账户，场所级 `latency_model`/`fill_model`/`fee_model`，资金费结算，组合敞口（基线 §2.2）。
- **hftbacktest**：
  - `Backtest` 支持多资产、多交易所，每个资产可配不同的队列模型和资产类型
    （`hftbacktest:hftbacktest/src/backtest/mod.rs:601-608`）。
  - 所有资产的交易所侧、本地侧数据和订单事件放进同一个 `EventSet`，每步取最早时间戳，因此跨场所在同一条时间轴上同步
    （`hftbacktest:hftbacktest/src/backtest/evs.rs:42-63`）。
  - 但所有资产必须共用同一种盘口实现类型 `MD`（`hftbacktest:hftbacktest/src/backtest/mod.rs:604-608`）。
  - 每个资产的 `State` 独立，余额、持仓、费用都从 0 起算（`hftbacktest:hftbacktest/src/backtest/state.rs:22-35`）。
    没有共享资金池、保证金、强平或资金费结算；在 `hftbacktest/src` 与 Python 包中搜索 funding、margin、liquidat、transfer，
    只在 Binance 转换器（把资金费率作为自定义事件 ID 102 写入，`hftbacktest:py-hftbacktest/hftbacktest/data/utils/binancefutures.py:51-58`）
    和统计指标 `MaxLeverage` 中出现。
  - Python 示例全部是单资产回测，多资产靠 `multiprocessing` 并行跑独立回测，再在 notebook 里手工加总权益
    （`hftbacktest:examples/Making Multiple Markets.ipynb:327`）。跨市场做市示例仍在路线图上（`hftbacktest:ROADMAP.md:68`）。

### 3.4 数据格式与接入（CEX / Databento / Pyth）

| 来源              | Nautilus                                                                          | hftbacktest                                                                                                                                                                                                                                                                  |
| ----------------- | --------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 存储格式          | Parquet catalog，支持 S3/GCS/Azure（基线 §2.4、§4.1）                             | 8 字段 numpy 结构化数组，存为 `.npz`；每行 64 字节（`hftbacktest:docs/data.rst:10-20`、`hftbacktest:hftbacktest/src/types.rs:311-331`）；可选读取 `s3://`（`hftbacktest:hftbacktest/src/backtest/data/npy/mod.rs:299-316`）；Parquet 尚未支持（`hftbacktest:ROADMAP.md:23`） |
| 自有 CEX 原始数据 | 通用 CSV 加载器、wrangler、Binance 深度 CSV（基线 §4.1）                          | 需转成 `ev/exch_ts/local_ts/px/qty/order_id/ival/fval` 并打上 `EXCH_EVENT`/`LOCAL_EVENT` 标志；可复用 `correct_local_timestamp`、`correct_event_order`、`validate_event_order`（`hftbacktest:py-hftbacktest/hftbacktest/data/validation.py:15-152`）                         |
| Tardis            | CSV 加载与 Machine 回放（基线 §4.3）                                              | `tardis.convert` 支持 trades、`incremental_book_L2`、`book_ticker`（`hftbacktest:py-hftbacktest/hftbacktest/data/utils/tardis.py:56-112`），另有 `convert_fuse`（`hftbacktest:py-hftbacktest/hftbacktest/data/utils/tardis.py:472`）                                         |
| Databento         | 覆盖 MBO、MBP-1、MBP-10、TBBO、OHLCV 等；`ts_recv` 映射为 `ts_event`（基线 §4.2） | 只支持 MBO（`hftbacktest:py-hftbacktest/hftbacktest/data/utils/databento.py:45-51`）；`ts_event` 映射为 `exch_ts`，`ts_recv` 映射为 `local_ts`，两个时间都保留（`hftbacktest:py-hftbacktest/hftbacktest/data/utils/databento.py:73-75`）                                     |
| Pyth              | 无适配器，需做成 `CustomData`（基线 §4.4）                                        | 未找到（在整个仓库搜索 `pyth`，排除 python 后无结果）；只能用 notebook 演示的"本地时间点查找数组"方式接入（`hftbacktest:examples/Integrating Custom Data.ipynb:22`）                                                                                                         |

### 3.5 实盘路径

- **Nautilus**：17 个官方适配器，其中加密场所有 Binance、Bybit、Coinbase、Deribit、dYdX、Hyperliquid、Kraken、OKX 等
  （`nautilus:ADAPTERS.md:79-97`）；有执行对账（`nautilus:docs/concepts/live.md:244`）；回测与实盘使用同一套策略代码
  （`nautilus:docs/concepts/overview.md:15`）。
- **hftbacktest**：
  - Rust 侧有 `Bot` trait，回测与实盘共用同一份策略代码（`hftbacktest:hftbacktest/src/types.rs:778`、
    `hftbacktest:hftbacktest/examples/algo.rs:5-21`）。
  - 实盘连接器只覆盖 Binance Futures（仅测试网验证）和 Bybit（开发中）（`hftbacktest:connector/README.md:8-15`）；
    代码里另有 Binance Spot（`hftbacktest:connector/Cargo.toml:6-9`）。
  - 机器人与连接器通过 iceoryx2 共享内存通信，必须在同一台机器上（`hftbacktest:connector/README.md:41`）。
  - 连接器启动时撤销全部挂单以获得干净状态（`hftbacktest:connector/src/binancefutures/user_data_stream.rs:129-140`），
    未找到持仓、挂单的对账逻辑。
  - Python 实盘绑定存在，但 PyPI 构建默认不启用 `live` 特性（`hftbacktest:py-hftbacktest/Cargo.toml:10-12`），
    README 标注实盘"仅 Rust"（`hftbacktest:README.rst:21`），Python 实盘列在路线图未完成项（`hftbacktest:ROADMAP.md:9`）。

### 3.6 分析、报表与 UI

- **Nautilus**：34 个统计量、报表 DataFrame、自包含 Plotly HTML tearsheet；UI 明确不在范围内（基线 §2.4、§5.3）。
- **hftbacktest**：
  - Python `stats` 模块提供 17 个指标，包括 `SR`、`Sortino`、`MaxDrawdown`、`ReturnOverMDD`、`MaxLeverage`
    （`hftbacktest:py-hftbacktest/hftbacktest/stats/__init__.py:1-50`）。
  - 年化默认按 252 天，文档提示加密可改为 365（`hftbacktest:py-hftbacktest/hftbacktest/stats/metrics.py:84-93`）。
  - 记录器只按固定间隔采样每个资产的价格、持仓、余额、费用、成交次数（`hftbacktest:py-hftbacktest/hftbacktest/recorder.py:19-37`、
    `hftbacktest:py-hftbacktest/hftbacktest/types.py:113-125`），不是逐笔成交日志。
  - 绘图用 matplotlib 或 holoviews（`hftbacktest:py-hftbacktest/hftbacktest/stats/stats.py:83-99`）。
  - 没有 UI，也没有组合层报表；编排类工作（中心编排应用等）在路线图未完成项中（`hftbacktest:ROADMAP.md:58-61`）。
- **结论**：两者都需要自建 Web 管理界面；Nautilus 的报表与结果对象更完整。

### 3.7 Python 与 Rust 的易用性

- **hftbacktest**：
  - Python 策略必须写成 Numba `@njit` 函数（`hftbacktest:README.rst:15`、`hftbacktest:README.rst:86-93`）。
    导入库和首次编译策略都需要 JIT 时间（`hftbacktest:docs/jit_compilation_overhead.rst:4-8`）。
  - Numba 只支持 Python 的子集，团队现有的 pandas/对象式代码不能直接放进策略循环（推断）。
  - 队列模型、交易所模型、延迟模型的自定义只能用 Rust 实现（`hftbacktest:docs/order_fill.rst:172-179`）；
    Python 只能从固定枚举中选择（`hftbacktest:py-hftbacktest/src/lib.rs:72-105`）。
  - Python 默认配置：`LogProbQueueModel2`、`NoPartialFillExchange`、零延迟、零费用
    （`hftbacktest:py-hftbacktest/src/lib.rs:134-157`）。零延迟默认值容易造成过于乐观的结果（推断）。
- **Nautilus**：Python 策略是普通 Python 类，回调同步执行；Python 可自定义 `LatencyModel` 与成交模型（基线 §2.3、§5.4）。
- **小结**：一般研究员上手 Nautilus 更容易；需要改撮合或队列假设时，两者都要写 Rust 或 Python 扩展，但 Nautilus 的 Python 扩展点更多。

### 3.8 性能

- **hftbacktest**：仓库内没有正式基准，只有 notebook 中的 `%%time` 输出，硬件未说明：
  - BTCUSDT 一个月（2023 年 5 月）全量 L2 回放加订单簿不平衡做市策略，墙钟约 33 分钟
    （`hftbacktest:examples/Market Making with Alpha - Order Book Imbalance.ipynb:241`、`hftbacktest:examples/Market Making with Alpha - Order Book Imbalance.ipynb:257`）。
  - BTCUSDT 一天，完整回测 1 分 49 秒；"加速回测"（不建模队列与回报延迟、预计算成交条件）为 416 毫秒，约快 260 倍
    （`hftbacktest:examples/Accelerated Backtesting.ipynb:13`、`hftbacktest:examples/Accelerated Backtesting.ipynb:1233`、
    `hftbacktest:examples/Accelerated Backtesting.ipynb:1353`）。加速回测是 notebook 里手写的 numba 代码，不是库功能。
  - 调度器每步对 `4 × 资产数` 个时间戳做线性扫描（`hftbacktest:hftbacktest/src/backtest/evs.rs:42-63`），
    资产数很多时可能成为瓶颈（推断）。
- **Nautilus**：原生基准只覆盖单场所 bar 加报价，没有多场所 L2 吞吐数据；大量成交累积持仓时 v2 比 v1 慢（基线 §5.4）。
- **结论**：两者都没有可直接比较的、与团队负载相同的公开数字，需要用自有数据实测（见 §6）。

### 3.9 许可证

- **hftbacktest**：MIT（`hftbacktest:LICENSE:1-3`、`hftbacktest:hftbacktest/Cargo.toml:6`），可自由 fork、修改、闭源内部使用。
- **Nautilus**：LGPL-3.0-only，对外分发修改版需开放修改部分源码，商标受限，贡献需签 CLA（基线 §5.2）。
- **结论**：许可证层面 hftbacktest 风险更低。

### 3.10 成熟度与维护风险

| 信号       | Nautilus                                      | hftbacktest                                                                                                                                                                                                                                         |
| ---------- | --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 版本       | v2 为 `2.0.0rc8`，Beta（基线 §5.1）           | Rust crate `0.9.4`（`hftbacktest:hftbacktest/Cargo.toml:3`）；Python 包 `2.4.4` 标注 Production/Stable（`hftbacktest:py-hftbacktest/pyproject.toml:11`、`hftbacktest:py-hftbacktest/pyproject.toml:23`）                                            |
| 稳定性声明 | 每个 RC 都有大量破坏性变更（基线 §5.1）       | Rust README 称"处于初始开发阶段，可能不经通知做破坏性变更"，实盘"未经全面测试"（`hftbacktest:hftbacktest/README.md:5-6`）                                                                                                                           |
| 作者       | 多维护者的组织项目（推断）                    | Cargo 与 PyPI 元数据只有一位作者 nkaz001（`hftbacktest:hftbacktest/Cargo.toml:5`、`hftbacktest:py-hftbacktest/pyproject.toml:9`）；路线图只致谢一位外部贡献者（`hftbacktest:ROADMAP.md:21`）                                                        |
| 测试与 CI  | 完整 CI 与 pre-commit（推断，基于本仓库结构） | Rust 核心约 22 个单元测试（在 `hftbacktest/src` 中统计 `#[test]`），Python 测试 1 个文件 77 行；CI 只有 `codeql.yml`、`release-python.yml`、`stale.yml`，无测试工作流；测试覆盖与 CI 列为未完成（`hftbacktest:ROADMAP.md:55-56`）                   |
| 代码规模   | 大型多 crate 工作区                           | 核心 crate 约 1.4 万行 Rust，连接器约 7.6 千行（`wc -l` 统计）                                                                                                                                                                                      |
| 文档       | 概念文档、教程、集成文档齐全（基线）          | Read the Docs 站点加 21 个教程 notebook（`hftbacktest:README.rst:181-203`），API 参考较薄                                                                                                                                                           |
| fork 状态  | 不适用                                        | 远端为 `nextlevelcyber/hftbacktest`，但唯一可见提交作者为 nkaz001，日期 2025-12-23；README 与 Cargo 链接全部指向 `nkaz001/hftbacktest`（`hftbacktest:README.rst:70`、`hftbacktest:hftbacktest/Cargo.toml:8`）。浅克隆，无法判断 fork 是否有独有改动 |

**数值精度**：hftbacktest 的价格、数量、余额、费用全部是 `f64`（`hftbacktest:hftbacktest/src/types.rs:314-331`、
`hftbacktest:hftbacktest/src/backtest/state.rs:38-46`），盘口内部按 `round(px / tick_size)` 量化为 tick
（`hftbacktest:hftbacktest/src/depth/hashmapmarketdepth.rs:92-93`）。Nautilus 使用 64 位或 128 位定点数
（`nautilus:docs/concepts/overview.md:243`）。对研究型回测问题不大；用于对账级的 PnL 与费用核算时，Nautilus 更合适（推断）。

## 4. hftbacktest 逐项核实

### 4.1 做市

**提供的能力**

- **盘口重建**：L2 增量（`DEPTH_EVENT`）、清空（`DEPTH_CLEAR_EVENT`）、快照（`DEPTH_SNAPSHOT_EVENT`）、BBO（`DEPTH_BBO_EVENT`），
  以及 L3 的新增、撤单、改单、成交事件（`hftbacktest:hftbacktest/src/types.rs:150-297`）。
- **多源盘口融合**：`FuseMarketDepth` 把更新更快的 `bookTicker` 与较慢的深度流合并。教程指出 Binance `depth@0ms` 仍是聚合推送
  （`hftbacktest:examples/Fusing Depth Data.ipynb:12`）。
- **队列位置模型**：
  - `RiskAdverseQueueModel`：只有同价位成交才推进队列；盘口量减少时只截断为新量
    （`hftbacktest:hftbacktest/src/backtest/models/queue.rs:42-96`）。
  - `ProbQueueModel`：盘口量减少时，按概率把减少量分配到自己前方或后方。
    估计式为 `est_front = front - (1 - p) * chg + min(back - p * chg, 0)`，并先扣除已计入的成交量以免重复计算
    （`hftbacktest:hftbacktest/src/backtest/models/queue.rs:181-207`）。
  - 概率函数 `p = f(back) / (f(back) + f(front))` 等 5 种，`f` 为幂函数或 `log(1 + x)`
    （`hftbacktest:hftbacktest/src/backtest/models/queue.rs:219-330`）。文档给出选择 `f` 的约束：队首时概率为 0，队尾时为 1
    （`hftbacktest:docs/order_fill.rst:163-168`）。
  - `L3FIFOQueueModel`：价格-时间优先，回测订单排在真实订单之间，排在其后的真实订单成交即视为自己已成交
    （`hftbacktest:hftbacktest/src/backtest/models/queue.rs:473-479`）。
- **交易所模型**：
  - `NoPartialFillExchange`（默认）：价格穿越最优价、穿越成交价，或在队首且与成交价相等时，**全量**成交；
    吃单不论数量都按最优价全量成交（`hftbacktest:docs/order_fill.rst:22-45`）。
  - `PartialFillExchange`：队首等价成交时按剩余成交量部分成交；吃单按盘口数量成交，但盘口不因此改变
    （`hftbacktest:docs/order_fill.rst:53-87`）。
  - 时间有效性支持 GTC、GTX（post-only）、FOK、IOC，订单类型支持 Limit 与 Market（`hftbacktest:hftbacktest/src/types.rs:412-445`）。
    在无部分成交模型中，FOK/IOC 吃单也全量成交（`hftbacktest:hftbacktest/src/backtest/proc/nopartialfillexchange.rs:330-339`）。
- **费用**：按成交额、按成交量、按笔三种费率模型，maker/taker 分开，可设负费率（返佣）；另有按方向的费用
  （`hftbacktest:hftbacktest/src/backtest/models/fee.rs:3-153`）。
- **延迟**：见 §3.2 与 §4.3。
- **示例丰富**：GLFT 模型、网格、订单簿不平衡、基差、APT、大 tick 品种队列做市、L3 对比等（`hftbacktest:README.rst:181-203`）。

**缺失或受限之处**

- **不建模市场冲击**：建议从小单量开始，并对照实盘逐步校准（`hftbacktest:docs/debugging_backtesting_and_live_discrepancies.rst:32-36`）。
- **自定义队列模型只能用 Rust 写**（`hftbacktest:docs/order_fill.rst:172-179`）；"更多队列或交易所模型"仍在路线图上（`hftbacktest:ROADMAP.md:17`）。
- **L3 FIFO 假设**不适用于按比例分配（pro-rata）等撮合规则（`hftbacktest:hftbacktest/src/backtest/models/queue.rs:477-479`）；
  收到清空消息时会清掉所有回测订单，可能与交易所实际行为不同（`hftbacktest:hftbacktest/src/backtest/models/queue.rs:455-469`）。
- **无资金费结算、无保证金和强平**（见 §3.3），永续合约做市的持仓成本需要在策略或后处理中自行计算。
- **数值为 `f64`**（见 §3.10）。

**结论：强。**

### 4.2 跨所套利

**提供的能力**

- **多资产多交易所**：同一回测可加入多个资产，各自配置数据、延迟、队列、交易所、费用模型（`hftbacktest:README.rst:20`、
  `hftbacktest:hftbacktest/src/backtest/mod.rs:601-608`）。
- **场所级延迟**：每个资产独立的 `LatencyModel` 和行情 `latency_offset`。`latency_offset` 的设计用途正是"行情在别处采集、策略在另一地运行"
  的跨所回测（`hftbacktest:py-hftbacktest/src/lib.rs:169-179`、`hftbacktest:ROADMAP.md:16`）。
- **跨场所同步**：统一的 `EventSet` 按时间戳取下一个事件（`hftbacktest:hftbacktest/src/backtest/evs.rs:42-63`），
  各场所的本地时间戳需处于同一时钟下（推断）。
- **相关教程**：基差做市使用现货 L1 数据（`hftbacktest:examples/Market Making with Alpha - Basis.ipynb:8`）；
  跨所参数对比（Binance、Bybit、OKX、Hyperliquid 等）是分场所独立回测（`hftbacktest:examples/High-Frequency Grid Trading - Comparison Across Other Exchanges.ipynb`，第 0 至 4 个单元）。

**缺失或受限之处**

- **账户与资金层**：每个资产的余额、持仓独立且从 0 起算，权益按资产计算（`hftbacktest:hftbacktest/src/backtest/state.rs:22-56`、
  `hftbacktest:hftbacktest/src/backtest/assettype.rs:2-7`）。没有共享资金、保证金、资金费、划转、提币建模。
- **没有跨市场示例**（`hftbacktest:ROADMAP.md:68`），Python 示例全部单资产（检索 `examples/*.ipynb` 中的回测构造调用，参数均为 `[asset]`）。
- **按下单、改单、撤单区分延迟**未支持（`hftbacktest:ROADMAP.md:22`）。
- **所有资产必须共用同一盘口类型**（`hftbacktest:hftbacktest/src/backtest/mod.rs:604-608`），因此不能在同一回测中混用 L2 与 L3 场所（推断）。

**结论：部分。** 适合研究"跨所信号在真实延迟下能否执行"，不适合评估资金占用、库存与保证金约束。

### 4.3 延迟评估

**提供的能力**

- **三类延迟**：行情延迟、下单入站延迟、下单回报延迟（`hftbacktest:docs/latency_models.rst:7-25`）。
- **所需数据字段**：
  - 行情：每个事件需要 `exch_ts`（交易所时间）与 `local_ts`（本地接收时间），且必须满足 `local_ts > exch_ts`
    （`hftbacktest:docs/data.rst:14-15`、`hftbacktest:docs/data.rst:217-223`）。
  - 下单：`req_ts`、`exch_ts`、`resp_ts` 三列（`hftbacktest:hftbacktest/src/backtest/models/latency.rs:57-69`）。
    文档建议定期提交不会成交的订单来采集（`hftbacktest:docs/latency_models.rst:45-47`）。
- **采集器记录两个时间戳**：采集器在 WebSocket 收到消息时取 `Utc::now()` 作为本地时间
  （`hftbacktest:collector/src/binancefuturesum/http.rs:83`），并写成"纳秒时间戳 + 原始 JSON"的按天 gzip 文件
  （`hftbacktest:collector/src/file.rs:36-50`）；交易所时间保留在原始 JSON 内，由转换器解析（`hftbacktest:py-hftbacktest/hftbacktest/data/utils/binancefutures.py:97-120`）。
- **延迟扫描**：
  - 常数延迟可直接改参数（`hftbacktest:py-hftbacktest/src/lib.rs:271-283`）。
  - `generate_order_latency` 用"行情延迟 × 倍数 + 偏移"合成下单延迟（`hftbacktest:py-hftbacktest/hftbacktest/data/utils/feed_order_latency.py:45-99`）；
    教程用 4 倍和 3 倍行情延迟演示延迟放大对收益的影响（`hftbacktest:examples/Impact of Order Latency.ipynb`）。
  - 行情与下单延迟都可按偏移量平移（见 §3.2）。
  - 文档明确把"人为降低延迟以评估更高等级账户或基础设施投入"列为用途
    （`hftbacktest:docs/debugging_backtesting_and_live_discrepancies.rst:21-24`）。
- **策略可读取当前延迟**：`feed_latency()` 与 `order_latency()` 返回最近一次的时间戳（`hftbacktest:py-hftbacktest/hftbacktest/binding.py:948-985`），
  实盘机器人同样提供（`hftbacktest:hftbacktest/src/live/mod.rs:24-25`）。

**缺失或受限之处**

- **时钟偏差处理粗糙**：`correct_local_timestamp` 对整份文件施加一个统一偏移，使最小延迟不为负（`hftbacktest:py-hftbacktest/hftbacktest/data/validation.py:15-50`），
  无法处理时变的时钟漂移；文档建议用 PTP 改善时钟同步（`hftbacktest:docs/data.rst:219-221`）。
- **时间戳语义因数据源而异**：Tardis 的 Binance Futures 数据用推送时间 `E`，而不是撮合时间 `T`，因此测得的延迟偏小
  （`hftbacktest:py-hftbacktest/hftbacktest/data/utils/tardis.py:67-69`）。交易所官方历史数据没有本地时间戳，只能人为加延迟
  （`hftbacktest:py-hftbacktest/hftbacktest/data/utils/binancehistmktdata.py:146-149`）。
- **不模拟请求乱序到达**（`hftbacktest:hftbacktest/src/backtest/order.rs:30-33`）。
- **没有网络链路测量工具**（RTT 直方图、丢包统计）；搜索采集器与连接器源码未找到（推断这属于实盘监控范畴）。

**结论：强。**

### 4.4 因子研发

**提供的能力**

- **Python 侧数据是 numpy 结构化数组**，可直接用 polars/numpy 处理（`hftbacktest:docs/data.rst:10-20`）；依赖包含 numba、polars
  （`hftbacktest:py-hftbacktest/pyproject.toml:13-19`）。
- **研究模式示例**：
  - "Pricing Framework"（标注 DRAFT）构造 0.1 秒采样的 `[时间 × 品种]` 收益矩阵，计算现货收益、市场回归、VAMP 等信号，
    并在 notebook 中手写 IC 函数（`hftbacktest:examples/Pricing Framework.ipynb:12`、`hftbacktest:examples/Pricing Framework.ipynb:2153`）。
  - "Accelerated Backtesting" 预计算成交条件，单日回测从 1 分 49 秒降到 416 毫秒，适合参数搜索（`hftbacktest:examples/Accelerated Backtesting.ipynb:1353`）。
- **参数搜索**：Rust 示例用 Python 多进程调用回测程序做网格搜索（`hftbacktest:hftbacktest/examples/6_gridsearch.py:1-25`）。

**缺失或受限之处**

- **没有因子库 API**：在 Python 包中搜索 IC、相关系数、横截面分组，未找到；上述研究代码都在 notebook 中。
- **没有数据目录或查询层**：数据按文件路径逐个加载，也没有元数据管理（推断，依据 `BacktestAsset.data()` 只接收文件列表）。
- **策略必须是 numba 函数**（见 §3.7），不适合大规模 ML 或 pandas 风格研究。

**结论：弱。** 比 Nautilus 更贴近 tick 级 HFT 信号研究的写法，但仍需独立的研究栈。

### 4.5 市场数据评估

**提供的能力**

- **采集器**：支持 Binance Spot、Binance USDⓈ-M 与 COIN-M 合约、Bybit、Hyperliquid（`hftbacktest:collector/src/main.rs:38-108`）。
  - Binance 合约流按 `pu` 与上一条 `u` 比对检测断档，断档时限流补拉 REST 快照（`hftbacktest:collector/src/binancefuturesum/mod.rs:33-91`）。
  - 在 Bybit 与 Hyperliquid 采集代码中搜索断档或序列号检测，未找到。
- **校正与校验**：
  - `correct_local_timestamp`：修正负延迟（`hftbacktest:py-hftbacktest/hftbacktest/data/validation.py:15-50`）。
  - `correct_event_order`：交易所时间与本地时间顺序不一致时，把事件拆成交易所侧和本地侧两份，分别排序
    （`hftbacktest:py-hftbacktest/hftbacktest/data/validation.py:53-136`、`hftbacktest:docs/data.rst:204-248`）。
  - `validate_event_order`：校验两侧时间是否单调（`hftbacktest:py-hftbacktest/hftbacktest/data/validation.py:139-152`）。
- **盘口工具**：`create_last_snapshot` 生成日终快照供次日使用（`hftbacktest:py-hftbacktest/hftbacktest/data/utils/snapshot.py:9-20`）；
  `DiffOrderBookSnapshot` 用于从快照流生成增量（`hftbacktest:py-hftbacktest/hftbacktest/data/utils/difforderbooksnapshot.py:17`）。
- **场所数据对比示例**："Fusing Depth Data" 比较 L2 深度流与 `bookTicker` 的更新频率和领先关系（`hftbacktest:examples/Fusing Depth Data.ipynb:12`）。

**缺失或受限之处**

- **转换器不检查序列号**：Binance 转换器读取 `depthUpdate` 时不使用 `U`/`u`/`pu`（`hftbacktest:py-hftbacktest/hftbacktest/data/utils/binancefutures.py:116-143`），
  断档只能靠采集端补快照。
- **没有数据质量报告**：缺口统计、跨场所一致性、供应商对比均未找到。
- **转换器按文件全量载入内存**，并预分配 `buffer_size` 行（默认 1 亿行，每行 64 字节），大文件需手动调参
  （`hftbacktest:py-hftbacktest/hftbacktest/data/utils/tardis.py:56-62`、`hftbacktest:py-hftbacktest/hftbacktest/data/utils/tardis.py:97-99`）。

**结论：部分。**

### 4.6 数据接入

- **格式**：8 字段结构化数组 `ev, exch_ts, local_ts, px, qty, order_id, ival, fval`，`ev` 是事件类型、方向与
  `EXCH_EVENT`/`LOCAL_EVENT` 的位标志组合（`hftbacktest:docs/data.rst:10-20`、`hftbacktest:hftbacktest/src/types.rs:150-297`）。
  文件为 `.npz`，可从 S3 读取（需 `s3` 特性，Python 包已启用，`hftbacktest:py-hftbacktest/Cargo.toml:16`）。
- **现成转换器**（`hftbacktest:py-hftbacktest/hftbacktest/data/utils/`）：
  - Binance Futures 原始流（含 `markPrice` 与 `bookTicker` 可选自定义事件，`binancefutures.py:20-63`）。
  - Binance 和 Bybit 官方历史数据（无本地时间戳，`binancehistmktdata.py:28-31`、`bybithistmktdata.py:27-30`）。
  - Bybit 原始流（含多深度融合 `convert_fused`，`bybit.py:111`）。
  - Hyperliquid 原始流（`hyperliquid.py:23`）、MEXC 现货（`mexc.py:17`）、Tardis（`tardis.py:56`、`tardis.py:472`）。
  - Databento 仅 MBO（`databento.py:22-51`），依赖可选包 `databento`（`hftbacktest:py-hftbacktest/pyproject.toml:45`）。
  - v1 数据迁移（`migration2.py:29`）。
- **Databento 适配度**：转换器保留 `ts_event` 与 `ts_recv`，L3 教程使用 CME MBO（`hftbacktest:examples/Level-3 Backtesting.ipynb:10`）。
  对团队而言，CME 加密期货 MBO 可以用 L3 FIFO 精确回测，这是 hftbacktest 的一个亮点（推断）。
  局限：`to_df()` 全量载入，且逐行 Python 循环（`hftbacktest:py-hftbacktest/hftbacktest/data/utils/databento.py:56-109`），大文件较慢（推断）。
- **Pyth**：未找到任何支持（全仓库搜索 `pyth`）。可行路径（推断）：
  - Pyth 的 `publish_time` 与本地接收时间天然对应 `exch_ts`/`local_ts`，可以写成本地侧时间点查找数组，
    在 numba 策略中按 `current_timestamp` 查最近值（`hftbacktest:examples/Integrating Custom Data.ipynb:22`）。
  - 也可以编码为自定义 `ev` ID 混入事件流（Binance 转换器使用 100 至 104 的先例，`binancefutures.py:51-63`），
    但 Python 侧未找到读取自定义事件的接口，需要用 Rust 包装 `LocalProcessor`（`hftbacktest:hftbacktest/examples/custom_evhandling.rs:27-29`）。
- **团队自有 CEX 数据需要做的事**：
  1. 每条消息必须有本地接收时间。若历史数据只有交易所时间，延迟研究将失真。
  2. 按场所解析为上述 8 字段，区分快照（先 `DEPTH_CLEAR_EVENT` 再 `DEPTH_SNAPSHOT_EVENT`）与增量。
  3. 成交与由成交引起的深度更新的先后顺序会影响队列估计，Tardis 转换器要求成交文件先于深度文件输入以免重复扣减
     （`hftbacktest:py-hftbacktest/hftbacktest/data/utils/tardis.py:75-80`）。
  4. 运行 `correct_local_timestamp`、`correct_event_order`、`validate_event_order`，按天输出 `.npz`，并用 `create_last_snapshot` 生成日终快照。

### 4.7 实盘

- **架构**：连接器是独立进程，通过 iceoryx2 共享内存服务多个机器人，仅限同机（`hftbacktest:connector/README.md:1-5`、
  `hftbacktest:connector/README.md:41`）；TCP 远程通信在路线图上（`hftbacktest:ROADMAP.md:31`）。
- **场所**：Binance Futures（测试网验证）、Bybit（开发中）（`hftbacktest:connector/README.md:13-16`），代码中另有 Binance Spot
  （`hftbacktest:connector/Cargo.toml:6-9`）。OKX、Hyperliquid、Coinbase、Kraken、Databento 行情均在路线图未完成项中
  （`hftbacktest:ROADMAP.md:43-49`）。
- **同一份策略代码**：Rust 的 `Bot` trait 同时由回测器与实盘机器人实现（`hftbacktest:hftbacktest/src/types.rs:778-783`），
  示例中同一个 `gridtrading` 函数用于回测与实盘（`hftbacktest:hftbacktest/examples/algo.rs:5-21`、`hftbacktest:hftbacktest/examples/gridtrading_live.rs:47-58`）。
- **风险声明**：连接器"风险自负，可能并非所有情况都正常工作"（`hftbacktest:connector/README.md:8-9`）；
  Binance 下单走 REST，WebSocket 下单在路线图上（`hftbacktest:ROADMAP.md:34`）。
- **成熟度判断（推断）**：适合单人或小团队对单一场所做原型验证；对 USD 5 亿规模的团队，缺少对账、风控、多场所覆盖，不宜作为生产实盘。

### 4.8 工程与风险

- **许可证**：MIT（见 §3.9）。
- **版本与稳定性**：见 §3.10。Python 包的 Production/Stable 分类与 Rust README 的"初始开发阶段"声明相互矛盾
  （`hftbacktest:py-hftbacktest/pyproject.toml:23`、`hftbacktest:hftbacktest/README.md:5-6`）。
- **发布节奏**：浅克隆只有一个提交，无法从仓库判断发布节奏，未找到；仓库有 Python 发布工作流 `release-python.yml`。
- **单人维护风险**：作者元数据只有一人（见 §3.10）。代码量不大且为 MIT，团队有能力自行维护 fork（推断）。
- **文档质量**：概念文档（数据、延迟、成交）短而准确，与源码一致；大量知识在 notebook 中；Rust API 文档依赖 docs.rs。
- **组合、会计、报表**：只有单资产记录和 17 个指标（见 §3.6），没有组合 PnL 归因、逐笔成交日志或 tearsheet。
- **UI**：无。

## 5. 平台层自建工作量对比

| 模块                 | 以 Nautilus 为基础                                         | 以 hftbacktest 为基础                                     | 组合方案的额外成本                                                      |
| -------------------- | ---------------------------------------------------------- | --------------------------------------------------------- | ----------------------------------------------------------------------- |
| 原始数据层           | 加载器写入 catalog；需补 Pyth `CustomData`（基线 §4）      | 每个场所写转换器到 `.npz`，按天切分并生成日终快照（§4.6） | 统一原始层，保留 `exch_ts` 与 `local_ts`，分别导出两种格式              |
| 数据目录与查询       | 已有 Parquet catalog 与 DataFusion 查询（基线 §2.4）       | 无，需自建文件索引和元数据                                | 以 Nautilus catalog 或独立湖仓为准                                      |
| 数据质量             | 自建缺口、序列、跨所一致性作业（基线 §6）                  | 复用采集器断档补快照与排序校验，仍需自建质量报告（§4.5）  | 一套质量作业即可，放在原始层                                            |
| 行情与延迟采集       | 自建本地接收时间埋点、延迟库（基线 §6）                    | 可直接用采集器（仅 5 类场所），需自建下单延迟采集         | 采集器产出同时服务两个引擎                                              |
| 延迟模型             | 需自行扩展到行情延迟与回报延迟（推断：需改 Rust 撮合路径） | 已具备；自定义模型需写 Rust                               | 延迟研究集中在 hftbacktest                                              |
| 账户、保证金、资金费 | 已有；划转、提币需自建 `SimulationModule`（基线 §2.2）     | 全部自建，包括跨资产资金池（§4.2）                        | 放在 Nautilus                                                           |
| 组合与报表           | 有统计量与 tearsheet，需适配 365 天年化（基线 §2.4）       | 只有单资产指标，组合汇总需自建（§3.6）                    | 统一结果模型，两个引擎都导出到同一结果库                                |
| 作业编排与参数扫描   | 多进程或容器调度，`reset()` 复用数据（基线 §6）            | 多进程调度，可借鉴加速回测模式（§4.4）                    | 一个调度器，两类作业模板                                                |
| Web 管理界面         | 全部自建（基线 §5.3）                                      | 全部自建                                                  | 一个界面，两类引擎后端                                                  |
| 实盘                 | 已有多场所适配器与对账（§3.5）                             | 需补齐场所、对账、风控，或另选实盘系统（§4.7）            | 实盘统一走 Nautilus；hftbacktest 的研究结论需在 Nautilus 中重写策略     |
| 策略代码复用         | 回测与实盘同构                                             | Rust 回测与实盘同构；Python 仅回测                        | 两套策略代码，需建立"hftbacktest 研究、Nautilus 实现"的交接与一致性测试 |

**工作量判断（推断）**：

- 只用 Nautilus：缺口集中在"行情延迟与回报延迟"建模，这需要改动撮合核心，风险最高。
- 只用 hftbacktest：缺口集中在账户、组合、实盘，工作量最大。
- 组合方案：核心引擎都不改，新增工作集中在统一数据层、结果库与界面，以及策略从 hftbacktest 到 Nautilus 的移植流程。
  这是三者中风险最可控的方案。

## 6. 待解决问题

需要用团队自有数据动手验证的事项：

1. **队列模型校准**：在团队主要做市品种上，用实盘成交记录对比 `RiskAdverseQueueModel`、各 `ProbQueueModel` 变体与 Nautilus 队列跟踪的成交率差异，确定哪种更贴近实盘。
2. **自有数据的本地时间戳质量**：现有 CEX 历史数据是否带采集端接收时间、时钟同步方式（NTP/PTP）、是否存在时变漂移。
   这决定 hftbacktest 的行情延迟建模是否可信（§4.3）。
3. **吞吐与内存**：以典型规模（例如 10 至 50 个品种、多场所、一个月全量 L2）分别实测两个引擎的墙钟时间与峰值内存。
   两边都没有可比的公开数字（§3.8）。hftbacktest 需重点测 `npz` 全量载入与多资产调度的开销。
4. **多场所单回测**：在 hftbacktest 中把同一品种的两个场所放进一个回测，验证跨所下单的时序与 `latency_offset` 行为是否符合预期，因为官方没有此类示例（§4.2）。
5. **Databento MBO 转换器的规模**：在 CME 加密期货的整日 MBO 文件上测转换耗时与内存（§4.6）。
6. **Pyth 数据形态**：确定使用 Hermes 历史价格还是链上更新，以及是否有团队自己的接收时间，再决定两侧的接入方式（§4.6，基线 §4.4）。
7. **Nautilus 回测中交易所回报是否有延迟**：基线遗留问题（基线 §7 第 1 条），直接影响"做市在 Nautilus 中复现 hftbacktest 结论"的可行性。
8. **hftbacktest 上游活跃度**：本地为浅克隆的 fork，无法判断上游 `nkaz001/hftbacktest` 当前的发布节奏与问题响应速度，需要在 GitHub 上核实。
9. **研究到实盘的一致性**：选一个做市策略，分别在 hftbacktest 与 Nautilus 中实现并用同一份数据回测，量化两者结果差异，作为组合方案交接流程的基准。
