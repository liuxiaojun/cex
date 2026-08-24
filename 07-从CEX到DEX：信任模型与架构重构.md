# 第 7 节 | 从 CEX 到 DEX：信任模型与架构重构

> **核心命题：** DEX 不是"去掉交易所" — 是把 CEX 承担的信任重新分配给合约、链和用户。三种架构模式代表三种不同的分配方式，本节重点拆解其中最接近 CEX 的一种：链下运行一个完整的 CEX，链上用数学证明验证结果。

---

## 本节要回答的问题

| 序号 | 问题 |
|------|------|
| 一 | 从 CEX 到 DEX，信任模型发生了什么根本变化？Orderbook DEX 有哪几种架构模式？ |
| 二 | 链下撮合 + 链上结算的完整运行流程是什么？从连接钱包、充值、交易到提现？ |
| 三 | CEX 的服务在"链下撮合 + 链上结算"模式中如何映射？哪些不变、哪些新增？ZK Proof 证明了什么？ |
| 四 | 电路（Circuit）、Batch、State Root 各自的角色是什么？ |
| 五 | 用户退出权如何保障？Escape Hatch 和 Data Availability 的关系是什么？ |

---

## 一、信任模型与架构模式总览

**开场衔接：** 前五节讲完了 CEX 的五大系统。从这节课开始进入 DEX。

---

### 两种信任假设

| | CEX | DEX |
|--|-----|-----|
| **信任对象** | 交易所（机构） | 代码（智能合约） |
| **资金控制** | 交易所代管 | 用户自持 / 链上合约锁定 |
| **余额可信度** | 交易所告诉你有多少钱（你无法验证） | 链上状态公开，任何人可验证 |
| **服务运行位置** | 全部在交易所服务器上 | 分散在合约、链、Operator、用户身上 |

根本区别不是"有没有中心"，而是"你在信任谁"：CEX 信任一个机构；DEX 信任一套代码。

---

### FTX — 当"信任交易所"的假设失效

**FTX（2022.11）：** 全球第二大交易所。用户余额显示正常，但交易所挪用资金给关联公司 → 挤兑时无法兑付 → 破产。CEX 模型中 Account Service 余额 ≠ 链上真实资产，这个等式依赖交易所自律。DEX 模型中，这个等式由代码强制执行。

> **判断点：** 从 CEX 到 DEX，不是"去掉中心化"，而是把原本由交易所一家承担的信任，分散到合约、链、用户自己身上。每一次分散都有代价。

---

### 三种架构模式总览

> 不同的 Orderbook DEX 对"撮合放在哪里"和"如何保证撮合结果可信"做出了不同选择。架构模式决定了 CEX 的服务如何重新分布。

| 架构模式 | 代表项目 | 撮合位置 | 信任机制 | "可信"的含义 | 性能 | 去中心化程度 |
|----------|----------|---------|----------|-------------|------|------------|
| A. 全链上 | EtherDelta（早期）, Kuru Exchange（Monad） | 链上 | 链上直接执行 | 所有操作在链上，合约逐笔执行。早期受限于链性能，高性能链上正在重新验证 | 低→待验证 | 最高 |
| B. Off-chain Matching + On-chain Settlement | dYdX v3 (StarkEx) | 链下 Operator | Validity Proof (ZK) | Operator 批量提交状态变更 + STARK Proof → L1 合约验证。信任数学 — 不合法的状态变更无法生成有效 Proof | 高 | 中 |
| C. App-chain（撮合嵌入共识）| dYdX v4, Hyperliquid | 链上（验证者节点内）| 共识机制 | 撮合作为出块的一部分，多验证者共识确认。信任共识 — 等同于信任链本身 | 中--高 | 高 |

---

**Mode A — 全链上方案（简要）**

```
用户 → 直接提交链上交易（下单/撤单/吃单 = 各一笔链上交易，付 Gas）
    ↓
链上合约：维护 Orderbook + 撮合 + 直接结算
没有 Operator，所有 CEX 服务要么消失，要么全部由合约承担
```

核心优势：没有任何链下角色，完全透明，最去中心化。

**但在主流 L1 上走不通：** TPS 有限（无法处理每秒数千次挂撤单）、Gas 成本（做市商不可接受）、确认延迟（幽灵订单）。

**案例：** EtherDelta（2017）— 所有操作链上执行。幽灵订单 + Gas 浪费 + 体验不可接受。但前端被黑时合约资金不受影响 — 证明了链上合约的安全性。

**高性能链正在让全链上方案重新变得可行：** Monad（并行 EVM, 10,000+ TPS）、MegaETH（亚毫秒延迟, 100,000+ TPS）、Sei（内置 Orderbook 模块）。Kuru Exchange（Monad, Paradigm 领投 $11.6M）正在验证高性能链能否让全链上 CLOB 真正可用。

---

**Mode C — App-chain 方案（一句话）**

撮合嵌入验证者共识，不再需要单一 Operator，也不需要 ZK Proof — 信任来自多数验证者诚实。代表项目：dYdX v4、Hyperliquid。

> **预告第 9 节展开。** Mode C 涉及共识机制、Bridge、价格投票等独立话题，将在后续课程详细拆解。

---

**本节聚焦 Mode B** — Off-chain Matching + On-chain Settlement。正因为全链上方案在主流 L1 上面临严重性能瓶颈，Mode B 用"链下运行 + 链上验证"的方式，在性能和安全之间找到了一个平衡点。

---

## 二、Mode B 完整运行流程

> Mode B 的本质：链下运行一个完整的 CEX（Operator），链上用数学证明验证结果。以 dYdX v3 / StarkEx 为例，按用户交互流程完整展开。

---

### 首次连接 — 密钥派生

```
用户连接 MetaMask
    ↓ MetaMask 弹出，签名一条消息（不是链上交易，不花 Gas）
    ↓
浏览器用这个签名 → 派生出 STARK 密钥对（StarkEx 体系的密钥，非以太坊密钥）
    ├── STARK 私钥 → 存储在浏览器 localStorage
    └── STARK 公钥 → 注册到 Operator，绑定用户的以太坊地址

此后下单签名用 STARK 私钥，MetaMask 不再弹出
这是一个关键的 UX 设计 — 如果每笔下单都弹钱包，体验和全链上模式一样差
```

---

### 充值 — 资金进入 L1 合约

```
用户在前端选择充值
    ↓ MetaMask 弹出，签名链上交易（付 Gas）
    ↓
以太坊 L1 交易：用户资金 → 转入 StarkEx Vault 合约
    ↓
Operator 监听到链上充值事件 → 链下 Account 余额 +N
```

> **单链限制：** StarkEx 合约部署在 Ethereum L1，用户充提只走 Ethereum 主网。其他链的资产须先跨桥至 Ethereum。

---

### 交易 — 全在链下完成

```
用户下单
    ↓ 浏览器用 STARK 私钥自动签名（无 MetaMask 弹出，无 Gas）
    ↓
Operator（链下）
    ├── 验证 STARK 签名 → 确认是该用户的订单
    ├── 维护 Orderbook（全在内存中，和 CEX 一样）
    └── 执行撮合 → 生成成交记录
          ↓
    成交记录在链下累积（不立即上链）
```

---

### 批量结算 — Proof 上链

```
Operator 积累一批成交（主网活跃期平均 2–10 分钟一个 Batch，低峰更久，高峰更频繁）
    ↓
汇总状态变更：
    用户 A: USDC -100, ETH +0.05
    用户 B: USDC +100, ETH -0.05
    ...（可能包含数百到数千笔）
    ↓
提交给 StarkEx Service（链下服务，维护 Balances Tree 和 Orders Tree）
    ↓
StarkEx Service → SHARP（Shared Prover，共享证明服务）
    SHARP 不是 dYdX 独享的 — dYdX、Immutable X、Sorare 等共用同一个 Prover
    多个应用的批次可以合并成一个 Proof → 分摊 L1 验证的 Gas 成本
    ↓
SHARP 生成 STARK Proof → 提交到 L1
    ↓
L1 Verifier 合约验证 Proof
    ├── ❌ 验证失败 → 拒绝整批（Operator 无法伪造）
    └── ✅ 验证通过 → 将有效证明的哈希写入 Fact Registry（链上注册表）
    ↓
StarkEx Contract（dYdX 的链上合约）查询 Fact Registry
    → 确认这批状态变更对应的 Proof 已通过验证
    → 更新 Vault 中所有用户余额

关键设计：
    → 验证 Proof 的成本远低于重新执行所有交易
    → SHARP 共享模式进一步降低单个应用的 Gas 成本
    → Fact Registry 解耦了"验证"和"状态更新" —
      Verifier 只管验证，应用合约自行决定何时读取结果并更新状态
```

---

### 提现 — 三条路径

```
正常提现（Slow Withdrawal）：
    用户请求提现 → Operator 在下一批结算中包含 → Batch + Proof 上链验证
    → L1 Vault 释放资金 → 用户 claim
    耗时：数小时（等 Batch 结算）

快速提现（Fast Withdrawal）：
    用户发起快速提现 → 链下匹配流动性提供者（LP）
    → LP 在 L1 立即把资金打给用户
    → Batch 结算时，StarkEx 合约将用户的资金转给 LP 偿还
    本质：Conditional Transfer — 用户付手续费换即时到账，LP 赚手续费
    耗时：即时（LP 垫付）

强制提现（Forced Withdrawal — Operator 拒绝/宕机时）：
    用户直接在 L1 调用 forcedWithdrawalRequest()（MetaMask 签名，付 Gas）
    → Operator 必须在 ~7 天内处理
    → 超时 → 任何人可调用 freeze()
    → 冻结后用户提交 Merkle Proof 直接从 Vault 提取
    耗时：数天（Grace Period）
```

---

### 三种签名的区别

| 签名方式 | 用什么签 | 什么时候用 | 是否上链 | 是否弹 MetaMask |
|---------|---------|----------|---------|---------------|
| 以太坊签名（链上交易） | 以太坊私钥 | 充值、提现、Force Withdrawal | 是（付 Gas） | 是 |
| 以太坊签名（链下消息） | 以太坊私钥 | 首次连接，派生 STARK 密钥 | 否 | 是（仅一次） |
| STARK 签名 | STARK 私钥（浏览器本地） | 每笔下单、撤单 | 否 | 否 |

---

## 三、组件总览：CEX → Mode B 的服务映射

> Mode B 的本质：一个 CEX + 链上结算层。大部分服务仍在 Operator 链下运行，和 CEX 几乎一样。链上只承担两件事：**资金托管**（Vault）和**最终状态验证**（STARK Proof）。

---

### 链下组件 — 与 CEX 基本一致（Operator 内部运行）

| 组件 | 职责 | 与 CEX 的差异 |
|------|------|-------------|
| API Gateway | 接收用户请求，鉴权，限流，路由 | 验证 STARK 签名而非传统 API Key |
| Order Service | 订单参数校验，创建订单 | 验证用户签名的链下订单 |
| Risk Service（Pre-trade） | 下单前风控检查 | 与 CEX 基本一致，Operator 链下执行 |
| Risk Service（In-trade） | 持续监控仓位风险 | 与 CEX 基本一致，Operator 链下持续执行 |
| Matching Engine | 撮合，输出成交记录 | 与 CEX 完全一致 |
| Account Service | 用户余额、仓位的实时管理 | 链下维护实时版本，链上只有最终状态 |
| Position Service | 管理仓位（方向、数量、入场价） | 与 CEX 一致 |
| Mark Price Service | 计算标记价格（用于盈亏计算和清算判断） | 与 CEX 完全一致 — Operator 是链下服务器，直接调外部交易所 API 聚合价格，和 CEX 一样 |
| Liquidation Engine | 执行强制平仓 | 与 CEX 基本一致，Operator 内部执行 |
| Insurance Fund | 穿仓损失缓冲池 | 基金余额是链下状态树的一部分（通过 ZK Proof 验证）；链上可查、透明。耗尽后触发 ADL |
| Funding Service | 计算并结算 Funding Rate | 与 CEX 基本一致，Operator 计算后链下结算 |
| Clearing Service | 成交结算 | 链下即时结算，链上批量确认 |
| Ledger Service | 逐笔交易记录 | 链下记录完整流水，链上只有 State Root |
| Market Data Service | 行情生成与推送（成交、Orderbook、K 线） | 与 CEX 一致 — 链上无逐笔成交，所有行情来自链下 |
| Chain Monitor | 监听 L1 充值事件 → 链下记账；处理链下提现请求 | CEX Wallet Service 的简化版 — 无 per-user 地址，无冷热钱包 |

> 以上组件和 CEX 几乎一样 — 这就是"Mode B = 一个 CEX + 链上结算层"。

### 链下组件 — Mode B 新增（链上结算相关）

| 组件 | 职责 |
|------|------|
| State Manager | 维护完整的链下状态（Balances Tree + Orders Tree + 全局 Funding 指数等） |
| Batch Assembler | 将 N 笔交易（含充提）打包成一个 Batch |
| Prover Service | 为每个 Batch 生成 ZK Proof（STARK 证明），包含执行电路（Circuit） |
| DA 组件（链下部分） | **Validium**：Data Availability Committee（DAC）存储数据并签名；**外部 DA 链**：发布到 Celestia / EigenDA 等独立 DA 层 |

> CEX 不需要 State Manager、Batch Assembler、Prover — 这些是 Mode B 为了向 L1 证明执行正确性而新增的组件。

### 链上合约（L1）

| 合约 | 职责 |
|------|------|
| Verifier Contract | 验证 Operator 提交的 ZK Proof |
| State Contract | 存储 State Root（balances_tree_root + orders_tree_root） |
| DA 相关 | **ZK-Rollup**：要求 Batch 附带 State Diff（L1 链上存储）；**Validium**：验证 DAC 签名；**外部 DA 链**：验证 DA 层的数据承诺（Data Commitment） |
| Deposit / Withdrawal Contract | 接收用户充值（锁定资产）；Batch 验证通过后释放提现资产供用户 claim |
| Escape Hatch | Forced Withdrawal — 用户绕过 Operator 直接从 L1 提款的最终保障 |

### 变化总结（CEX → Mode B）

| 类别 | 涉及的服务 | 说明 |
|------|-----------|------|
| **消失** | User Service | 无注册，用户 = 钱包地址 |
| **Operator 链下运行** | API Gateway, Order, Risk, Matching, Market Data, Position, Mark Price, Liquidation, Insurance Fund, Funding | 和 CEX 几乎一样 |
| **大幅简化** | Wallet Service → Chain Monitor | 无 per-user 地址，无冷热钱包 |
| **拆为两层** | Account, Clearing, Ledger | 链下实时 + 链上最终状态 |
| **Mode B 新增** | 链下：State Manager, Batch Assembler, Prover, DA 组件 | CEX 没有的 |
| **链上新增** | Verifier, State Contract, Deposit/Withdrawal, Escape Hatch | 资金托管 + 状态验证 + 退出权 |

> **关键变化不是"服务移到链上"，而是资金的最终控制权在链上。** Operator 链下运行的服务和 CEX 几乎一样；真正的区别在于：用户资金始终锁在 L1 合约中，Operator 只能"排列"交易，不能"偷"资金。

---

## 四、ZK Proof — 从链下执行到链上验证

> Mode B 的核心逻辑：Operator 在链下执行所有交易，然后用数学证明向 L1 证明执行结果是正确的。L1 不需要重新执行任何一笔交易 — 只需要验证证明。

---

### 完整流程

```
充值：用户资产 → L1 Deposit Contract 锁定 → Chain Monitor 监听 → 链下记入余额
交易：用户下单 → Operator 撮合 → State Manager 更新状态（余额、仓位、nonce）
提现：用户请求 → State Manager 更新余额 → 等 Batch + Proof 验证后在 L1 claim
批量上链：累积 N 笔 → 打包 Batch → Prover 生成 Proof → L1 Verifier 验证
    → 有效 → State Contract 更新 State Root → 提现可 claim
    → 无效 → 拒绝整批
```

---

### Proof 证明了什么

```
f(旧 State Root, 这批交易) = 新 State Root
```

即：**给定旧状态和这批交易，新状态是唯一正确的结果。**

这意味着：

- Operator **不可能伪造**交易结果（数学上不可能生成一个"谎言"的有效证明）
- L1 **不需要重新执行**任何交易 — 只需验证 Proof
- 验证比生成快几个数量级 — 这个不对称性是整个模型成立的前提

---

### 生成 vs 验证的不对称性

| | 生成 Proof | 验证 Proof |
|---|-----------|-----------|
| 执行者 | Operator 的 Prover Service | L1 上的 Verifier Contract |
| 耗时 | 主网平均 2–10 分钟 | 一笔 L1 交易（毫秒级） |
| 计算量 | 极大（专用硬件） | 极小（链上合约即可） |
| 成本 | Operator 承担 | Gas 费（一笔 L1 交易的 Gas） |

> 这个不对称性是关键：Proof 的生成很慢、很贵，但验证很快、很便宜。如果反过来（验证也很慢），整个模型就不成立了。

### 电路（Circuit）：链下逻辑的密码学形式

电路是交易规则的密码学编码，定义什么样的状态转换是"合法的"（余额守恒、余额非负、nonce 递增、价格优先撮合等）。电路在部署时确定，编译进 Verifier Contract。Prover 必须按电路规则生成 Proof，Verifier 按电路规则验证。链下逻辑偏离电路 → Proof 无法生成。

> 电路把"信任 Operator 的行为"转化为"信任数学的正确性"。只要电路正确，Operator 就无法作弊。

### Batch 机制与设计取舍

Proof 生成成本高且耗时长，所以批量处理：一个 Batch 包含数百到数千笔交易，但只需一个 Proof。Batch 越大 → 分摊成本越低，但用户等待 L1 最终确认的时间越长。

> 链下撮合是即时的（用户"感觉"交易已成功），但 L1 最终确认需要等 Batch + Proof 验证。

---

### State Root — 不是一棵树，而是一个二元组

dYdX v3 的 State Root 实际上是 **(balances_tree_root, orders_tree_root)**，两个根一起存入 StarkEx 合约，每次 Batch 验证通过后同时替换。

```
            State（存储在 L1 StarkEx 合约）
           /                                \
Balances Tree Root                 Orders Tree Root
（高度 31 Merkle 树）              （防重放 Merkle 树）
    |                                    |
叶子 = Vault（按 vaultId 索引）     叶子 ID = 请求哈希
├── starkKey                       value = 已执行数量
├── collateralAmount               → 防止重复执行或超量成交
└── 合成资产条目（assetId, amount, cachedFundingIndex）
```

- **Balances Tree**：一个用户可以有多个 Vault（不是"一个叶子 = 一个用户"）
- **Orders Tree**：防重放 — 新撮合时检查已执行数量是否用完
- **链下状态还包含**：全局 Funding 指数、系统时间戳、价格字典（Oracle Price Tick）

**关键属性：** 压缩性（两个 Root 都是固定长度哈希值）、可验证性（任何用户可用 Merkle Proof 证明自己的 Vault 状态）、防篡改（任何 Vault 变化都会导致 Root 变化）。

State Root 存储在 L1 上 — 这是用户退出权的技术基础。

---

### 关键判断：Operator 不可伪造，但可审查

| 行为 | 能否做到 | 约束机制 |
|------|----------|----------|
| 正常撮合、排序 | 可以 — 核心职责 | -- |
| 拒绝某用户的订单（审查）| 可以做到 | 这就是中心化风险 — 用户感知为"被拒绝服务" |
| 偷取用户资产 | 做不到 | 链上合约控制资金，Operator 没有私钥 |
| 伪造成交 | 做不到 | Proof 验证失败 → L1 拒绝 |
| 操纵交易顺序（抢跑）| 有可能 | Sequencer 控制排序，存在 MEV 风险 |
| 停止服务 | 可以做到 | 用户需要退出机制（Escape Hatch） |

**CEX vs DEX 的核心区别：** CEX 可以审查 + 停服 + 理论上可以挪用资金。DEX Operator 可以审查 + 停服 + 但不可能挪用资金。"不能偷钱"这一条 = DEX 相对 CEX 的根本性安全提升。

---

## 五、Escape Hatch 与 Data Availability — 用户退出权

> Operator 不可能偷你的钱（数学不允许），但可以不理你（拒绝处理你的交易）。Escape Hatch 就是用户在 Operator 不配合时的最终退出通道。

---

### Forced Withdrawal 完整流程

```
Alice 请求提现被 Operator 拒绝
    → Alice 在 L1 调用 forcedWithdrawalRequest()
    → 等待期（Grace Period，通常数天）
    → Operator 配合 → 正常处理，流程结束
    → Operator 不配合（超时）→ 任何人可调用 freeze() → 合约进入冻结状态
    → 冻结后：所有用户提交 Merkle Proof 证明余额 → 直接从 L1 合约提款
```

**关键点：**

- Forced Withdrawal 是用户的**最终保障** — 即使 Operator 完全消失，用户也能取回资产
- 但代价是：合约被冻结后，整个系统停止运行，所有用户都必须退出
- 这不是一个"方便的功能"，而是一个"核武器" — 正常情况下不应该用到，但它的存在是信任模型的基石

**dYdX v3 (StarkEx) 的具体参数：**

- 用户在以太坊 L1 上直接调用 `forcedWithdrawalRequest()`
- StarkEx Operator 必须在约 7 天内处理
- 如果超时未处理 → 任何人可调用 `freeze()` → 系统进入冻结状态
- 冻结后：用户可以提交 Merkle Proof 证明自己的余额 → 直接从 Vault 提取
- Operator 无法阻止冻结 — 这是写在 L1 合约中的硬约束

---

### Merkle Proof 的前提：Data Availability

用户构造 Merkle Proof 需要整棵状态树的数据。如果数据只存在 Operator 手里 → Operator 消失 → 数据丢失 → Escape Hatch 名存实亡。所以必须有独立于 Operator 的数据存储方案 → 这就是 **Data Availability（数据可用性）**。

---

### 三种 DA 方案对比表

| | ZK-Rollup | 外部 DA 链（Celestia / EigenDA 等） | Validium |
|---|-----------|-------------------------------------|----------|
| 数据位置 | L1 链上（以 calldata 形式提交） | 独立的 DA 链 | 链下数据可用性委员会（DAC） |
| 每个 Batch | Operator 将 State Diff 发布到 L1 | Operator 将数据发布到 DA 链，DA 链的数据承诺提交到 L1 | Operator 将数据发送给 DAC，DAC 签名确认 |
| 成本 | **最高** — 每个 Batch 都需要 L1 Gas | **中等** — DA 链专为数据存储优化，远低于 L1 | **最低** — 数据不上链 |
| DA 保证 | **L1 级别** — L1 始终可用，任何人都能重建完整状态 | **DA 链共识级别** — 依赖 DA 链的安全性和数据采样（DAS）机制 | **信任假设** — 依赖 DAC 成员诚实存储 |
| Escape Hatch | 始终可行 | 取决于 DA 链的可用性 — DA 链宕机或数据丢失则无法退出 | 仅在 DAC 配合时可行 |
| 信任假设 | 仅信任 L1 | 信任 DA 链的共识和验证者 | 信任 DAC 成员不串通 |
| 典型案例 | dYdX v3 | Manta Pacific（迁移至 Celestia DA） | ImmutableX |

> **State Diff：** 每个 Batch 附带"哪些用户的状态发生了变化以及新值是什么"。任何人可以通过重放所有 Batch 的 State Diff，从创世状态重建出完整的当前状态树。

> **dYdX v3 选择了 ZK-Rollup 模式** — 作为金融应用，用户退出权的绝对保证比成本更重要。

**安全性排序：** ZK-Rollup（L1 DA）> 外部 DA 链 > Validium（DAC）— 安全性递减，成本也递减。

---

### 因果链条与关键判断

```
Data Availability → Merkle Proof → Escape Hatch → 用户退出权
```

没有 DA，State Root 只是一个无法解包的承诺 — 你知道你的钱"在里面"，但你证明不了。

Mode B 的用户退出权是一条完整链条：ZK Proof 保证状态正确 → State Root 在 L1 上可查 → DA 保证数据可重建 → Escape Hatch 保证用户能退出。**任何一环断裂，退出权就失效。**

> DEX 的"去中心化"更准确地说是"可验证 + 可退出"，而不是"没有中心"。评估安全性看两点：(1) Operator 权力边界是否清晰 (2) 用户退出权是否真正可用（含 DA）。

---

## 六、链上与链下状态边界

### 清晰的边界

| 类别 | 状态 | 特点 |
|------|------|------|
| **链上状态** | 用户账户余额（Vault）、已结算仓位、最终成交记录 | 可信、慢、贵 |
| **链下状态** | Orderbook、未成交订单、实时撮合过程 | 快、便宜、需信任 Operator |

### 灰色地带

| 灰色地带 | 问题 | 选择 |
|----------|------|------|
| 挂单冻结的资金 | 链上冻结（贵）还是链下记账（信任 Operator）？ | 链下快但 Operator 可能超额承诺 |
| 未结算的成交 | Operator 撮合完但未上链，算不算成立？ | Operator 宕机时这批成交可能丢失 |
| 链下取消 | 用户发取消但 Operator 在取消前撮合了 | 取决于 Operator 是否诚实处理时序 |

### 判断框架：三个问题

1. 它**必须上链**吗？（涉及资金最终归属 → 必须；只是中间过程 → 可以不上链）
2. 如果不上链，**信任风险**是什么？（Operator 能操纵吗？影响资金还是只影响体验？）
3. 如果上链，**性能代价**是什么？（Gas + 延迟是否可接受？）

> **判断点：** 状态的可信度和更新频率往往矛盾 — 没有"全部上链"或"全部链下"的正确答案。

---

## 本节判断点汇总

1. 从 CEX 到 DEX，不是"去掉中心化"— 而是把信任从交易所转移到合约、链、用户自己身上
2. CEX 的服务不是消失了 — 而是重新分配（消失 / 链下保留 / 拆两层 / 新增链上组件）
3. 在主流 L1 上 Orderbook 完全上链不可行（撮合频率 vs 出块速度的量级差距），但高性能链正在尝试突破
4. 三种架构模式的核心差异是信任机制 — Validity Proof（信数学）、共识（信链）、全链上（信执行）
5. Mode B 的本质是"一个 CEX + 链上结算层"— 大部分服务仍在 Operator 链下，只有资金控制权真正上链
6. ZK Proof 的核心是不对称性：生成慢、验证快。这个不对称性是整个 Mode B 模型成立的前提
7. 电路（Circuit）是链下业务逻辑的密码学形式 — 只要电路正确，Operator 就无法伪造状态
8. Operator 不能偷钱，但能审查和停服 — "不能偷钱"是 DEX 相对 CEX 的根本性安全提升
9. Escape Hatch 解决的是审查问题，不是伪造问题
10. 用户退出权的完整链条：Proof → State Root → Data Availability → Escape Hatch — 任何一环断裂，退出权失效
11. 状态的可信度和更新频率往往矛盾 — 设计必须在两者之间找到平衡
12. DEX 的"去中心化" = 可验证 + 可退出（Force Withdrawal + Data Availability）

---

**下节预告：** 本节讲的 Mode B 架构有一个隐含前提：只考虑了单链场景。但如今很多 Orderbook DEX 支持多条链的充提 — 链下统一账本和链上分散 Vault 之间的再平衡，才是真正的系统瓶颈。→ 第 8 节：多链多币种充提
