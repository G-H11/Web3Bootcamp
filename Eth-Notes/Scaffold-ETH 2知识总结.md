# Scaffold-ETH 2知识总结

### **一、环境搭建与项目初始化**

**1、创建项目：**`npx create-eth@latest`

- **输入项目名称**，例如`my-dapp`。
- **选择智能合约框架**：通常选择`Hardhat`（生态成熟）或`Foundry`（测试速度快）。
- 工具会自动完成项目创建和所有依赖的安装。

### **二、核心项目目录结构**

| **路径** | **主要用途** | **关键内容说明** |
| --- | --- | --- |
| **项目根目录** | 项目管理与脚本 | `package.json`：定义了项目的启动、构建、部署等所有脚本命令，是操作入口。 |
| **`packages/hardhat/`** | **智能合约开发环境** | 包含以太坊开发的所有后端部分。 |
| ├─ `contracts/` | **存放Solidity合约源码** | 你的业务合约（如`YourContract.sol`）和外部依赖（如`@openzeppelin`）都在此。 |
| ├─ `deploy/` | **部署脚本目录** | `00_deploy_your_contract.js` 等脚本，定义了合约的部署逻辑和初始化参数。 |
| ├─ `test/` | **合约测试文件** | 为合约编写的单元测试和集成测试文件（使用Hardhat/Foundry）。 |
| └─ `ignition/` (或 `deployments/`) | **部署产出目录** | 存放部署后生成的`artifacts`（合约接口描述）和`deployedAddresses.json`（记录各网络合约地址）。 |
| **`packages/nextjs/`** | **前端DApp应用** | 基于Next.js构建的前端界面。 |
| ├─ `app/` | **Next.js App Router核心** | 存放页面（`page.tsx`）、布局（`layout.tsx`）和API路由。 |
| ├─ `components/` | **可复用UI组件库** | `scaffold-eth/`下提供开箱即用的组件，如`Address`、`Balance`、`Input`等。 |
| ├─ `hooks/` | **自定义React Hooks** | `scaffold-eth/`下的核心交互钩子，如`useScaffoldContractRead/Write`，封装了与合约交互的逻辑。 |
| ├─ `utils/` | **工具函数与配置** | `scaffold.config.ts`（**前端核心配置文件**）和`notification.tsx`（交易通知）等。 |
| └─ `contracts/` | **已部署合约的引用** | 由框架自动生成的`deployedContracts.ts`，**前端通过此文件获取当前网络的合约地址和ABI**，**切勿手动修改**。 |

### **三、交互钩子解析**

| **类别** | **钩子名称** | **核心用途** | **是否消耗 Gas** |
| --- | --- | --- | --- |
| **合约交互** | `useScaffoldContractRead` | 读取合约数据/视图函数 | 否 |
|  | `useScaffoldContractWrite` | 发送交易，修改合约状态 | 是 |
|  | `useDeployedContractInfo` | 获取合约的地址与 ABI | 否 |
| **事件监听** | `useScaffoldEventSubscriber` | 订阅并实时响应合约事件 | 否 |
|  | `useScaffoldWatchContractEvent` | 另一种事件监听方式（可手动控制） | 否 |
| **状态与账户** | `useScaffoldEthPrice` | 获取 ETH 当前价格（USD） | 否 |
|  | `useAccountBalance` | 获取指定地址的 ETH 余额 | 否 |
|  | `useNetworkColor` | 获取当前链的标识颜色 | 否 |
| **交易增强** | `useTransactor` | 增强交易体验（通知、确认） | - |

### **1、合约交互钩子**

**`useScaffoldContractRead`：读取数据**

这是最常用的只读钩子，用于调用合约中 `view` 或 `pure` 函数。

**使用场景**：获取用户余额、DAO提案列表、NFT的总供应量等。**返回的 `data` 已经过解析**（例如 `BigInt` 被转换为 `number`）。

**`useScaffoldContractWrite`：发送交易**

用于发送交易、修改链上状态的钩子。

**状态说明**：

- `isLoading`: 用户未签名，交易在钱包待处理。
- `isMining`: 交易已签名，正在链上打包。
- 交易成功或失败会**自动触发全局通知**。

**`useDeployedContractInfo`：获取合约元数据**

这个钩子会自动根据当前连接的网络（`targetNetworks`），从 `deployedContracts.ts` 中读取对应合约的地址和 ABI。**通常用作其他钩子（如 `useScaffoldContractRead`）的内部依赖，你很少需要直接使用它**。

### **2、事件监听钩子**

**`useScaffoldEventSubscriber`：自动订阅事件**

最常用的事件监听方式，组件挂载时自动订阅，卸载时自动清理。

**`useScaffoldWatchContractEvent`：手动控制监听**

与 `useScaffoldEventSubscriber` 功能类似，但提供更多手动控制选项，监听逻辑更灵活。

### **3、状态与账户信息钩子**

**`useAccountBalance`：获取 ETH 余额**

**`useScaffoldEthPrice`：获取 ETH 价格**

此钩子从 CoinGecko API 获取价格，**注意有速率限制**，生产环境建议使用自己的 API Key。

**`useNetworkColor`：获取网络颜色**

用于 UI 标识，根据链 ID 返回预设颜色（如主网：绿色，Goerli：蓝色）。

### **4、交易增强钩子**

**`useTransactor`：包装交易执行器**

这是一个**高阶工具**，用于包装 `writeAsync` 或任何返回 Promise 的交易函数，为其**添加统一的 UI 反馈**（如交易等待、成功/失败的通知）。

### **四、总结**

**1、理解目录结构**：明确合约在哪里写、前端在哪里改、配置在哪里调。

**2、遵循工作流**：从`contracts/`编写合约 → `yarn deploy`部署 → 在`app/page.tsx`中通过Hooks调用。

**3、善用内置组件**：优先使用`components/scaffold-eth/`中的组件（如`AddressInput`）构建UI，它们已与Hooks深度集成。

**4、修改配置**：通过`scaffold.config.ts`轻松切换开发网络和调整应用行为。