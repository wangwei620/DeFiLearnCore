# Uniswap V4 快速入门指南

欢迎来到Uniswap V4的世界！本指南将帮助您在最短时间内理解并开始使用Uniswap V4。

## 🚀 5分钟快速理解

### V4的核心创新

```
传统V3：每个池 = 一个合约
V4革新：所有池 = 一个PoolManager合约 + Hook系统
```

**三大突破**：
1. **Singleton架构** - 大幅降低Gas成本
2. **Hook系统** - 无限扩展可能性  
3. **解锁-回调模式** - 灵活的操作组合

### 关键概念速览

| 概念 | 简单理解 | 作用 |
|------|----------|------|
| PoolManager | 总管家 | 管理所有池子的状态 |
| Hook | 插件系统 | 在操作前后插入自定义逻辑 |
| Delta | 记账本 | 跟踪每笔操作的代币变化 |
| unlock/callback | 事务包装器 | 确保操作的原子性 |

## 📋 环境准备（10分钟）

### 1. 创建项目

```bash
# 创建新项目
mkdir my-v4-project && cd my-v4-project

# 初始化Foundry项目
forge init --no-git

# 安装V4核心依赖
forge install Uniswap/v4-core
forge install OpenZeppelin/openzeppelin-contracts
```

### 2. 配置foundry.toml

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
remappings = [
    "v4-core/=lib/v4-core/src/",
    "@openzeppelin/=lib/openzeppelin-contracts/"
]
solc_version = "0.8.26"
evm_version = "cancun"

[rpc_endpoints]
mainnet = "${MAINNET_RPC_URL}"
sepolia = "${SEPOLIA_RPC_URL}"
```

### 3. 基础合约模板

```solidity
// src/V4Integration.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IPoolManager} from "v4-core/interfaces/IPoolManager.sol";
import {IUnlockCallback} from "v4-core/interfaces/callback/IUnlockCallback.sol";
import {PoolKey} from "v4-core/types/PoolKey.sol";
import {BalanceDelta} from "v4-core/types/BalanceDelta.sol";
import {Currency, CurrencyLibrary} from "v4-core/types/Currency.sol";

contract MyV4Contract is IUnlockCallback {
    using CurrencyLibrary for Currency;
    
    IPoolManager public immutable poolManager;
    
    constructor(IPoolManager _poolManager) {
        poolManager = _poolManager;
    }
    
    modifier onlyPoolManager() {
        require(msg.sender == address(poolManager));
        _;
    }
    
    function unlockCallback(bytes calldata data) 
        external 
        onlyPoolManager 
        returns (bytes memory) 
    {
        // 在这里实现您的逻辑
        return "";
    }
}
```

## 🏊‍♂️ 第一次交换（15分钟）

### 最简单的交换器

```solidity
// src/SimpleSwapper.sol
contract SimpleSwapper is MyV4Contract {
    struct SwapData {
        PoolKey key;
        bool zeroForOne;
        int256 amountSpecified;
    }
    
    function swap(
        PoolKey memory key,
        bool zeroForOne,
        int256 amountSpecified
    ) external {
        SwapData memory data = SwapData(key, zeroForOne, amountSpecified);
        poolManager.unlock(abi.encode(data));
    }
    
    function unlockCallback(bytes calldata data) 
        external 
        override 
        onlyPoolManager 
        returns (bytes memory) 
    {
        SwapData memory swapData = abi.decode(data, (SwapData));
        
        // 执行交换
        BalanceDelta delta = poolManager.swap(
            swapData.key,
            IPoolManager.SwapParams({
                zeroForOne: swapData.zeroForOne,
                amountSpecified: swapData.amountSpecified,
                sqrtPriceLimitX96: 0
            }),
            ""
        );
        
        // 处理代币转账（简化版）
        _handleDelta(swapData.key, delta, swapData.zeroForOne);
        
        return "";
    }
    
    function _handleDelta(PoolKey memory key, BalanceDelta delta, bool zeroForOne) private {
        if (zeroForOne) {
            // 输入token0，输出token1
            if (delta.amount0() > 0) {
                // 向池子转入token0
                _settle(key.currency0, uint128(delta.amount0()));
            }
            if (delta.amount1() < 0) {
                // 从池子提取token1
                _take(key.currency1, uint128(-delta.amount1()));
            }
        } else {
            // 输入token1，输出token0
            if (delta.amount1() > 0) {
                _settle(key.currency1, uint128(delta.amount1()));
            }
            if (delta.amount0() < 0) {
                _take(key.currency0, uint128(-delta.amount0()));
            }
        }
    }
    
    function _settle(Currency currency, uint128 amount) private {
        // 简化实现：假设代币已授权
        poolManager.settle();
    }
    
    function _take(Currency currency, uint128 amount) private {
        poolManager.take(currency, msg.sender, amount);
    }
}
```

### 测试交换器

```solidity
// test/SimpleSwapper.t.sol
contract SimpleSwapperTest is Test {
    PoolManager poolManager;
    SimpleSwapper swapper;
    MockERC20 token0;
    MockERC20 token1;
    PoolKey key;
    
    function setUp() public {
        poolManager = new PoolManager();
        swapper = new SimpleSwapper(poolManager);
        
        // 创建测试代币
        token0 = new MockERC20("Token0", "T0", 18);
        token1 = new MockERC20("Token1", "T1", 18);
        
        // 确保正确排序
        if (address(token0) > address(token1)) {
            (token0, token1) = (token1, token0);
        }
        
        // 创建池
        key = PoolKey({
            currency0: Currency.wrap(address(token0)),
            currency1: Currency.wrap(address(token1)),
            fee: 3000,
            tickSpacing: 60,
            hooks: IHooks(address(0))
        });
        
        // 初始化池（需要先添加流动性）
        poolManager.initialize(key, SQRT_PRICE_1_1, "");
    }
    
    function testSwap() public {
        // 给用户代币并授权
        token0.mint(address(this), 1000 ether);
        token0.approve(address(poolManager), type(uint256).max);
        
        // 执行交换
        swapper.swap(key, true, 1 ether);
        
        // 验证结果
        assertGt(token1.balanceOf(address(this)), 0);
    }
}
```

## 🎣 第一个Hook（30分钟）

### 简单计数Hook

```solidity
// src/CounterHook.sol
import {BaseHook} from "v4-core/BaseHook.sol";
import {Hooks} from "v4-core/libraries/Hooks.sol";

contract CounterHook is BaseHook {
    uint256 public swapCount;
    
    constructor(IPoolManager _poolManager) BaseHook(_poolManager) {}
    
    function getHookPermissions() public pure override returns (Hooks.Permissions memory) {
        return Hooks.Permissions({
            beforeInitialize: false,
            afterInitialize: false,
            beforeAddLiquidity: false,
            afterAddLiquidity: false,
            beforeRemoveLiquidity: false,
            afterRemoveLiquidity: false,
            beforeSwap: true,
            afterSwap: false,
            beforeDonate: false,
            afterDonate: false,
            beforeSwapReturnDelta: false,
            afterSwapReturnDelta: false,
            afterAddLiquidityReturnDelta: false,
            afterRemoveLiquidityReturnDelta: false
        });
    }
    
    function beforeSwap(
        address,
        PoolKey calldata,
        IPoolManager.SwapParams calldata,
        bytes calldata
    ) external override returns (bytes4, BeforeSwapDelta, uint24) {
        swapCount++;
        return (BaseHook.beforeSwap.selector, BeforeSwapDeltaLibrary.ZERO_DELTA, 0);
    }
}
```

### Hook地址计算

Hook的功能由其部署地址决定。您需要使用CREATE2来获得特定的地址：

```solidity
// script/DeployHook.s.sol
contract DeployHook is Script {
    function run() external {
        uint160 flags = uint160(
            Hooks.BEFORE_SWAP_FLAG
        );
        
        // 计算需要的salt
        (address hookAddress, bytes32 salt) = HookMiner.find(
            CREATE2_DEPLOYER,
            flags,
            type(CounterHook).creationCode,
            abi.encode(address(poolManager))
        );
        
        // 部署Hook
        vm.startBroadcast();
        CounterHook hook = new CounterHook{salt: salt}(poolManager);
        vm.stopBroadcast();
        
        require(address(hook) == hookAddress, "Hook address mismatch");
    }
}
```

## 🏗️ 常见开发模式

### 1. 数据编码模式

```solidity
// 在unlock中传递复杂数据
struct CallbackData {
    address sender;
    PoolKey key;
    IPoolManager.SwapParams params;
    uint256 deadline;
}

function complexOperation(CallbackData memory data) external {
    require(block.timestamp <= data.deadline, "Expired");
    poolManager.unlock(abi.encode(data));
}
```

### 2. 错误处理模式

```solidity
contract SafeV4Contract is MyV4Contract {
    error InsufficientOutput(uint256 expected, uint256 actual);
    error Expired();
    
    function safeSwap(
        PoolKey memory key,
        int256 amountIn,
        uint256 minAmountOut,
        uint256 deadline
    ) external {
        if (block.timestamp > deadline) revert Expired();
        
        bytes memory result = poolManager.unlock(
            abi.encode(key, amountIn, minAmountOut)
        );
        
        uint256 amountOut = abi.decode(result, (uint256));
        if (amountOut < minAmountOut) {
            revert InsufficientOutput(minAmountOut, amountOut);
        }
    }
}
```

### 3. 多操作组合模式

```solidity
function swapAndAddLiquidity(
    PoolKey memory key,
    uint256 swapAmount,
    int24 tickLower,
    int24 tickUpper
) external {
    bytes memory data = abi.encode(
        "SWAP_AND_ADD",
        key,
        swapAmount,
        tickLower,
        tickUpper
    );
    poolManager.unlock(data);
}

function unlockCallback(bytes calldata data) external override returns (bytes memory) {
    (string memory operation, PoolKey memory key, uint256 swapAmount, int24 tickLower, int24 tickUpper) = 
        abi.decode(data, (string, PoolKey, uint256, int24, int24));
    
    if (keccak256(bytes(operation)) == keccak256("SWAP_AND_ADD")) {
        // 1. 先执行交换
        BalanceDelta swapDelta = poolManager.swap(key, /* params */, "");
        
        // 2. 再添加流动性
        (BalanceDelta liquidityDelta,) = poolManager.modifyLiquidity(
            key,
            IPoolManager.ModifyLiquidityParams({
                tickLower: tickLower,
                tickUpper: tickUpper,
                liquidityDelta: /* calculate based on swap result */,
                salt: bytes32(0)
            }),
            ""
        );
        
        // 3. 处理最终的代币转账
        _handleCombinedDeltas(key, swapDelta, liquidityDelta);
    }
    
    return "";
}
```

## 🚨 常见陷阱与解决方案

### 陷阱1：忘记处理Delta平衡

```solidity
// ❌ 错误：没有处理delta
function badSwap(PoolKey memory key) external {
    poolManager.unlock("");
}

function unlockCallback(bytes calldata) external override returns (bytes memory) {
    BalanceDelta delta = poolManager.swap(/* ... */);
    // 忘记调用settle/take，会导致CurrencyNotSettled错误
    return "";
}

// ✅ 正确：必须处理所有delta
function goodSwap(PoolKey memory key) external {
    poolManager.unlock("");
}

function unlockCallback(bytes calldata) external override returns (bytes memory) {
    BalanceDelta delta = poolManager.swap(/* ... */);
    
    // 必须处理所有非零delta
    if (delta.amount0() != 0) {
        if (delta.amount0() > 0) {
            _settle(key.currency0, uint128(delta.amount0()));
        } else {
            _take(key.currency0, uint128(-delta.amount0()));
        }
    }
    
    if (delta.amount1() != 0) {
        if (delta.amount1() > 0) {
            _settle(key.currency1, uint128(delta.amount1()));
        } else {
            _take(key.currency1, uint128(-delta.amount1()));
        }
    }
    
    return "";
}
```

### 陷阱2：Hook地址权限不匹配

```solidity
// ❌ 错误：Hook地址的权限位与实际实现不匹配
contract BadHook is BaseHook {
    // 实现了beforeSwap但地址权限位没有设置BEFORE_SWAP_FLAG
    function beforeSwap(...) external override returns (...) {
        // 这个函数永远不会被调用
    }
}

// ✅ 正确：使用CREATE2确保地址匹配权限
// 1. 先确定需要的权限
uint160 flags = Hooks.BEFORE_SWAP_FLAG | Hooks.AFTER_SWAP_FLAG;

// 2. 使用HookMiner找到对应的部署参数
(address expectedAddress, bytes32 salt) = HookMiner.find(
    deployer,
    flags,
    type(GoodHook).creationCode,
    constructorArgs
);

// 3. 使用计算出的salt部署
GoodHook hook = new GoodHook{salt: salt}(poolManager);
require(address(hook) == expectedAddress);
```

### 陷阱3：重入攻击

```solidity
// ❌ 易受攻击：没有重入保护
contract VulnerableContract is MyV4Contract {
    mapping(address => uint256) public balances;
    
    function withdraw() external {
        uint256 amount = balances[msg.sender];
        balances[msg.sender] = 0;
        
        // 恶意合约可以在这里重入
        payable(msg.sender).transfer(amount);
    }
}

// ✅ 安全：使用重入保护
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract SecureContract is MyV4Contract, ReentrancyGuard {
    mapping(address => uint256) public balances;
    
    function withdraw() external nonReentrant {
        uint256 amount = balances[msg.sender];
        balances[msg.sender] = 0;
        payable(msg.sender).transfer(amount);
    }
    
    function secureOperation() external nonReentrant {
        poolManager.unlock("");
    }
}
```

## 📚 学习资源与下一步

### 官方资源
- [Uniswap V4 官方文档](https://docs.uniswap.org/protocol/V4/overview)
- [V4 Core 仓库](https://github.com/Uniswap/v4-core)
- [V4 示例仓库](https://github.com/Uniswap/v4-periphery)

### 推荐学习路径

**第1周：基础掌握**
1. 理解Singleton架构和unlock-callback模式
2. 实现基本的交换和流动性操作
3. 学习Delta管理机制

**第2周：Hook系统**
1. 理解Hook权限系统
2. 开发简单的Hook（计数器、日志记录）
3. 学习CREATE2部署

**第3周：高级应用**
1. 多池路由实现
2. 套利机器人开发
3. 复杂Hook开发（动态费率、限价单）

**第4周：优化与部署**
1. Gas优化技巧
2. 安全审计检查清单
3. 测试网部署和测试

### 实践项目建议

1. **新手项目**：简单的代币交换界面
2. **进阶项目**：具有动态费率的流动性池
3. **高级项目**：跨链套利机器人

### 社区资源
- [Uniswap Discord](https://discord.gg/uniswap)
- [V4 开发者论坛](https://gov.uniswap.org/)
- [示例代码集合](https://github.com/Uniswap/v4-template)

## 🎯 快速检查清单

开始开发前，确保您已经：

- [ ] 理解V4的核心概念（Singleton、Hook、Delta）
- [ ] 设置好开发环境（Foundry + V4依赖）
- [ ] 实现了基本的`IUnlockCallback`接口
- [ ] 知道如何处理Delta平衡
- [ ] 了解Hook权限系统的工作原理

准备生产部署前，确保您已经：

- [ ] 实施了重入保护
- [ ] 添加了适当的参数验证
- [ ] 处理了所有错误情况
- [ ] 编写了全面的测试
- [ ] 进行了Gas优化
- [ ] 考虑了安全审计

---

🎉 **恭喜！** 您现在已经掌握了Uniswap V4的基础知识，可以开始构建下一代DeFi应用了！

记住：V4是一个强大但复杂的系统。从小项目开始，逐步提升复杂度，充分测试每个功能。社区随时准备帮助您解决问题！