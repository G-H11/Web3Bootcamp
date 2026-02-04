# 基础概念了解回顾——Week1Day3

## 一、EVM 与 Gas 机制

### 核心结论

EVM（以太坊虚拟机）是以太坊的执行引擎，负责解析执行智能合约字节码，维护链上状态；Gas 机制通过计算资源计价与限额防止网络滥用，是保障网络安全、效率与经济可持续性的核心，两者共同支撑以太坊 “世界计算机” 的稳定运行。

### 关键知识点

**1、EVM 核心构成**

**①全局状态（World State）：**记录所有账户的 nonce、balance、codeHash、storageRoot，以 Merkle-Patricia Trie 组织。

**②执行环境：**栈（Stack，深度上限 1024，用于算术运算）、内存（Memory，临时数据存储，执行后销毁）、存储（Storage，永久状态存储，Gas 成本最高）。

**③操作码（Opcodes）：**EVM 指令集，如 ADD（加法）、SSTORE（写入存储）、CALL（外部调用），每条指令有固定 Gas 成本。

**④执行上下文：**包含 msg.sender（调用者）、msg.value（附带 ETH）、gasLeft（剩余 Gas）等，决定合约执行权限与资源限额。

**2、EVM 执行流程**

**①部署阶段：**EOA 发起创建交易→EVM 执行 init code→初始化 storage→写入 runtime code→生成合约账户。

**②调用阶段：**接收 calldata（函数选择器 + 参数）→解析函数→执行字节码→操作栈 / 内存 / 存储→触发事件→返回结果。

**③异常处理：**执行失败（如 Gas 耗尽、revert）时，状态全部回滚，已消耗 Gas 不退还，防止资源滥用。

**3、Gas 机制细节**

**①Gas 定义：**计算资源计量单位，用于衡量交易 / 合约调用的资源消耗，避免无限循环与 DoS 攻击。

**②费用构成（EIP-1559 后）：**实际费用 = GasUsed×(BaseFee+PriorityFee)，BaseFee 由协议自动调整并销毁，PriorityFee 为验证者小费。

**③单位换算：**1 ETH=10¹⁸ Wei，1 Gwei=10⁹ Wei，Gas 价格通常以 Gwei/Gas 表示，如 20 Gwei/Gas 即 20×10⁹ Wei/Gas。

**④opcode Gas 成本：**算术运算（如 ADD）约 3 Gas，内存操作（如 MLOAD）约 3-300 Gas，存储写入（SSTORE）约 20000 Gas，外部调用（CALL）约 700 Gas 起步。

**4、Gas 优化策略**

**①存储优化：**合并存储写入（先在内存计算再批量写入）、打包小变量到同一存储槽（如 uint128+uint64+bool）、用事件替代非关键存储。

**②数据位置：**外部函数参数用 calldata（只读不复制），重复读取的 storage 变量缓存到 memory。

**③代码优化：**使用 constant/immutable 变量（不占存储）、避免无上限循环、采用库合约复用逻辑、开启编译器优化（Solc optimizer）。

**④链上链下分工：**将遍历、统计等重计算移至链下，链上仅验证结果（如 Merkle 证明）。

**⑤网络选择：**优先在 L2（如 Arbitrum、Optimism）部署，利用 blob 交易降低数据成本。

**5、关键升级影响**

**①London 升级（EIP-1559）：**引入 BaseFee 销毁机制，费用可预测性提升，ETH 具备通缩潜力。

**②Dencun 升级（EIP-4844）：**引入 blob 交易，降低 L2 数据存储成本，间接减少 L2 合约调用 Gas 费。

**③EIP-6780：**限制 SELFDESTRUCT 功能，仅在合约创建同交易中生效，避免状态滥用。

## 二、共识机制与生态展望

### 核心结论

以太坊历经 PoW 到 PoS 的共识转型，通过合并、Dencun 等升级构建 “Rollup 为中心” 的扩容路线，生态聚焦 DeFi、NFT、Layer2 三大核心赛道，未来将通过 Danksharding、Verkle 树进一步提升扩展性，ETH 经济模型趋向通缩，机构参与度与跨链互操作性持续提升。

### 关键知识点

**1、共识机制演进**

**①PoW（工作量证明）：**早期共识，矿工通过算力竞争出块，能耗高，2022 年合并后停用，Ethash 算法为内存密集型，抑制 ASIC 垄断。

**②The Merge（合并）：**2022 年 9 月实施，将执行层（主网）与共识层（信标链）合并，PoS 取代 PoW，能耗降 99.95%，为扩容铺路。

**③PoS（权益证明）：**验证者质押≥32 ETH 参与出块，通过 Slashing 惩罚双签、环绕投票等作恶行为，奖励包括共识层 ETH 增发 + 执行层小费 + MEV。

**④最终性（Finality）：**通过 Casper FFG 机制，区块经足够验证者投票后达成最终确认，不可回滚。

**2、扩容路线图**

**①Rollup 为中心：**Optimistic Rollup（乐观式，如 Optimism）与 ZK Rollup（零知识，如 zkSync）承接大部分交易，L1 负责结算与数据可用性。

**②Proto-Danksharding（EIP-4844）：**2024 年 Dencun 升级引入，支持 blob 交易，为 L2 提供廉价临时数据空间，L2 交易费降低 90%。

**③Danksharding：**未来升级方向，引入数据可用性采样（DAS），验证者随机抽样验证 blob 数据，无需下载全部数据，进一步提升 L2 数据容量。

**④Verkle 树：**替换当前的 Merkle-Patricia Trie，缩小状态证明体积，支撑无状态客户端，降低节点存储压力。

**3、生态核心亮点**

**①DeFi：**流动质押（Lido）、再质押（EigenLayer）、RWA（国债代币化）成为增长动力，BlackRock 等传统机构入局发行链上基金。

**②NFT：**从 PFP 投机转向实用场景，包括游戏资产（Axie Infinity）、品牌会员（奢侈品牌数字藏品）、凭证化资产（POAP 出席凭证）。

**③Layer2：**Arbitrum、Optimism、Base 等 Rollup 日活与交易量持续增长，Dencun 升级后费用进一步降低，成为应用部署首选。

**④社交与 DAO：**去中心化社交协议（Lens Protocol、Farcaster）、DAO 治理（Uniswap DAO）、公共物品资助（Gitcoin Grants）生态成熟。

**4、ETH 经济模型**

**①发行机制：**PoS 下年增发约 60-70 万 ETH，通胀率 0.5% 左右，远低于 PoW 时代的 4-4.6%。

**②销毁机制：**EIP-1559 下 BaseFee 销毁，高链上活动时销毁量超过发行量，呈现净通缩。

**③价值支撑：**Gas 费收入、质押收益、生态应用需求构成 ETH 核心价值，机构质押与 RWA 上链提升长期需求。

**5、未来发展方向**

**①技术升级：**Pectra 升级（2025 年）将推进账户抽象、EOF（EVM 对象格式）、验证者质押上限提升（从 32 ETH 到 2048 ETH）。

**②生态扩展：**跨链互操作性（LayerZero、CCIP）、隐私保护（零知识证明）、AI+DeFi（智能策略代理）成为创新方向。

**③治理与合规：**EIP 流程优化、机构合规工具完善、RWA 监管框架落地，推动以太坊生态规范化。