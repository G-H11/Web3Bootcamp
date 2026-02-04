# Hardhat & Foundry 智能合约测试总结

## 一、Hardhat

Hardhat 是基于 Node.js 的以太坊智能合约开发框架，其测试体系以**模块化、可扩展、贴近生产环境**为核心，支持单元测试、集成测试、主网分叉测试等全维度测试场景，工程化结构覆盖 “配置 - 编码 - 执行 - 分析” 全流程，是目前 Web3 开发中最主流的测试工程化方案。

### 1、测试文件命名规则

- 单元测试：`[合约名].test.ts`（如`MyERC20.test.ts`）；
- 集成测试：`[场景名]-integration.test.ts`（如`token-dao-integration.test.ts`）；
- 分叉测试：`[场景名]-fork.test.ts`（如`uniswap-fork.test.ts`）。

### 2、测试用例结构（Mocha+Chai）

每个测试文件遵循 “前置准备→测试用例→后置清理” 的结构，核心模板如下：

```
import { ethers } from "hardhat";
import { expect } from "chai";
import { MyERC20 } from "../typechain-types/contracts/token/MyERC20"; // TypeChain生成的类型
import { deployMyERC20 } from "../utils/deploy-helpers"; // 复用部署函数

// 测试套件（对应单个合约）
describe("MyERC20 (Unit Tests)", function () {
// 全局变量
let myERC20: MyERC20;
let deployer: SignerWithAddress;
let user1: SignerWithAddress;
let user2: SignerWithAddress;
const TOTAL_SUPPLY = ethers.parseEther("1000000"); // 总供应量

// 前置准备（每个测试用例执行前运行）
beforeEach(async function () {
// 1. 获取测试账户
[deployer, user1, user2] = await ethers.getSigners();
// 2. 部署合约
myERC20 = await deployMyERC20(deployer, TOTAL_SUPPLY);
});

// 测试用例1：
it("should set correct total supply after deployment", async function () {
const totalSupply = await myERC20.totalSupply();
expect(totalSupply).to.equal(TOTAL_SUPPLY);
});

// 测试用例2：
it("should mint all supply to deployer", async function () {
const deployerBalance = await myERC20.balanceOf(deployer.address);
expect(deployerBalance).to.equal(TOTAL_SUPPLY);
});

// 测试用例3：
it("should transfer tokens between users", async function () {
const transferAmount = ethers.parseEther("1000");
// 部署者转账给user1
await myERC20.connect(deployer).transfer(user1.address, transferAmount);
// 验证余额
expect(await myERC20.balanceOf(user1.address)).to.equal(transferAmount);
expect(await myERC20.balanceOf(deployer.address)).to.equal(
TOTAL_SUPPLY - transferAmount
);
});

// 测试用例4：
it("should revert when transferring more than balance", async function () {
const invalidAmount = ethers.parseEther("1000001"); // 超过总供应量
// 验证交易回滚，并包含指定错误信息
await expect(
myERC20.connect(deployer).transfer(user1.address, invalidAmount)
).to.be.revertedWithCustomError(myERC20, "InsufficientBalance");
});
});
```

总之，Hardhat 测试工程化遵循 “目录对齐、测试分层、配置统一” 的原则，`test`目录按 “单元→集成→分叉” 划分，与`contracts`一一对应。用例遵循原子性、独立性原则，覆盖正常 / 异常 / 边界场景，通过 TypeChain 实现类型安全。

## 二、Foundry

Foundry 是基于 Rust 开发的高性能智能合约开发测试框架，核心优势在于**Solidity 原生测试（无需 JS/TS 中转）、极速模糊测试、轻量主网分叉、精准 Gas 分析**，其测试体系以 “贴近 EVM、高性能、原生安全” 为核心，工程化结构覆盖 “配置 - 编码 - 执行 - 分析” 全流程，是 Web3 进阶开发中主流的测试工程化方案。

### 1、测试文件命名与结构规则

- 文件名：`[合约名/场景名].t.sol`（如`MyERC20.t.sol`、`UniswapIntegration.t.sol`）；
- 测试合约：每个测试文件对应一个 / 多个测试合约，命名为`[被测合约名]Test`（如`MyERC20Test`）；
- 测试函数：必须以`test`开头（如`testTotalSupplyAfterDeployment`），失败场景以`testFail`开头或用`vm.expectRevert`验证。

### 2、测试合约核心模板（Solidity 原生）

Foundry 测试合约需继承`Test`合约（Foundry 内置，提供 Cheatcodes 和断言），核心结构如下：

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.20;

// 导入Foundry核心测试合约
import "forge-std/Test.sol";
// 导入被测合约
import "../../src/token/MyERC20.sol";
// 导入测试工具
import "../utils/TestSetup.sol";

// 测试合约
contract MyERC20Test is Test {
// 1. 全局变量
MyERC20 public myERC20;
address public deployer = makeAddr("deployer"); // 生成测试账户
address public user1 = makeAddr("user1");
address public user2 = makeAddr("user2");
uint256 public constant TOTAL_SUPPLY = 1_000_000 ether;
// 2. 前置准备（每个测试用例执行前运行，类似Hardhat的beforeEach）
function setUp() public {
    // 给测试账户转账ETH（用于支付Gas）
    vm.deal(deployer, 100 ether);
    vm.deal(user1, 100 ether);
    vm.deal(user2, 100 ether);

    // 切换到deployer账户部署合约（模拟真实部署）
    vm.prank(deployer);
    myERC20 = new MyERC20(TOTAL_SUPPLY);
}

// 3. 基础测试用例：
function testTotalSupplyAfterDeployment() public view {
    // Foundry内置断言（无需Chai，原生Solidity）
    assertEq(myERC20.totalSupply(), TOTAL_SUPPLY);
}

// 4. 基础测试用例：
function testMintAllSupplyToDeployer() public view {
    assertEq(myERC20.balanceOf(deployer), TOTAL_SUPPLY);
}

// 5. 正常流程测试：
function testTransferTokensBetweenUsers() public {
    uint256 transferAmount = 1000 ether;

    // 模拟deployer发起转账（vm.prank切换调用者）
    vm.prank(deployer);
    myERC20.transfer(user1, transferAmount);

    // 验证余额
    assertEq(myERC20.balanceOf(user1), transferAmount);
    assertEq(myERC20.balanceOf(deployer), TOTAL_SUPPLY - transferAmount);
}

// 6. 异常流程测试：转账金额超过余额
function testRevertWhenTransferMoreThanBalance() public {
    uint256 invalidAmount = 1_000_001 ether;
    // 预期回滚，并验证自定义错误
    vm.expectRevert(abi.encodeWithSignature("InsufficientBalance(uint256,uint256)", TOTAL_SUPPLY, invalidAmount));
    vm.prank(deployer);
    myERC20.transfer(user1, invalidAmount);
}
}

```

Foundry 测试工程化遵循 “Solidity 原生、约定优于配置” 的原则，目录按 “单元→集成→分叉” 划分，Cheatcodes 是 Foundry 测试的灵魂，通过`vm.*`系列函数操控 EVM 环境、模拟链上行为。Cheatcodes（`vm.*`）操控 EVM 环境，原生模糊测试 / 不变式测试可自动发现边界漏洞，测试速度远快于 Hardhat。