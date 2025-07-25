# Uniswap V4 代码实例与最佳实践

本文档提供Uniswap V4的实际代码示例和开发最佳实践，帮助开发者快速上手并避免常见陷阱。

## 目录
1. [基础设置与环境](#基础设置与环境)
2. [基本交换实现](#基本交换实现)
3. [流动性管理](#流动性管理)
4. [Hook开发实例](#hook开发实例)
5. [高级应用模式](#高级应用模式)
6. [安全最佳实践](#安全最佳实践)
7. [Gas优化技巧](#gas优化技巧)
8. [测试策略](#测试策略)

## 基础设置与环境

### 项目依赖配置

```json
// package.json
{
  "dependencies": {
    "@uniswap/v4-core": "latest",
    "@openzeppelin/contracts": "^4.9.0",
    "forge-std": "^1.7.0"
  }
}
```

```toml
# foundry.toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
remappings = [
    "v4-core/=lib/v4-core/src/",
    "@openzeppelin/=lib/openzeppelin-contracts/"
]

[rpc_endpoints]
mainnet = "${MAINNET_RPC_URL}"
sepolia = "${SEPOLIA_RPC_URL}"
```

### 基础合约结构

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IPoolManager} from "v4-core/interfaces/IPoolManager.sol";
import {IUnlockCallback} from "v4-core/interfaces/callback/IUnlockCallback.sol";
import {PoolKey} from "v4-core/types/PoolKey.sol";
import {BalanceDelta} from "v4-core/types/BalanceDelta.sol";
import {Currency, CurrencyLibrary} from "v4-core/types/Currency.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract V4Integration is IUnlockCallback {
    using CurrencyLibrary for Currency;
    
    IPoolManager public immutable poolManager;
    
    constructor(IPoolManager _poolManager) {
        poolManager = _poolManager;
    }
    
    modifier onlyPoolManager() {
        require(msg.sender == address(poolManager), "Not authorized");
        _;
    }
    
    function unlockCallback(bytes calldata data) 
        external 
        onlyPoolManager 
        returns (bytes memory) 
    {
        // 解码数据并执行相应操作
        return "";
    }
    
    // 辅助函数：安全的代币转账
    function _settle(Currency currency, uint256 amount) internal {
        if (currency.isNative()) {
            poolManager.settle{value: amount}();
        } else {
            IERC20(Currency.unwrap(currency)).transferFrom(
                msg.sender, 
                address(poolManager), 
                amount
            );
            poolManager.settle();
        }
    }
    
    // 辅助函数：安全的代币提取
    function _take(Currency currency, address to, uint256 amount) internal {
        poolManager.take(currency, to, amount);
    }
}
```

## 基本交换实现

### 简单交换器

```solidity
contract SimpleSwapper is V4Integration {
    struct SwapParams {
        PoolKey key;
        bool zeroForOne;
        int256 amountSpecified;
        uint160 sqrtPriceLimitX96;
    }
    
    constructor(IPoolManager _poolManager) V4Integration(_poolManager) {}
    
    function swap(SwapParams calldata params) external payable {
        bytes memory data = abi.encode(params);
        poolManager.unlock(data);
    }
    
    function unlockCallback(bytes calldata data) 
        external 
        override 
        onlyPoolManager 
        returns (bytes memory) 
    {
        SwapParams memory params = abi.decode(data, (SwapParams));
        
        BalanceDelta delta = poolManager.swap(
            params.key,
            IPoolManager.SwapParams({
                zeroForOne: params.zeroForOne,
                amountSpecified: params.amountSpecified,
                sqrtPriceLimitX96: params.sqrtPriceLimitX96
            }),
            ""
        );
        
        // 处理输入代币
        if (params.zeroForOne) {
            if (delta.amount0() > 0) {
                _settle(params.key.currency0, uint256(uint128(delta.amount0())));
            }
            if (delta.amount1() < 0) {
                _take(params.key.currency1, msg.sender, uint256(uint128(-delta.amount1())));
            }
        } else {
            if (delta.amount1() > 0) {
                _settle(params.key.currency1, uint256(uint128(delta.amount1())));
            }
            if (delta.amount0() < 0) {
                _take(params.key.currency0, msg.sender, uint256(uint128(-delta.amount0())));
            }
        }
        
        return "";
    }
}
```

### 带滑点保护的交换器

```solidity
contract SlippageProtectedSwapper is V4Integration {
    error SlippageExceeded(uint256 expected, uint256 actual);
    
    struct ExactInputParams {
        PoolKey key;
        bool zeroForOne;
        uint256 amountIn;
        uint256 amountOutMinimum;
        uint160 sqrtPriceLimitX96;
    }
    
    function exactInputSingle(ExactInputParams calldata params) 
        external 
        payable 
        returns (uint256 amountOut) 
    {
        bytes memory data = abi.encode(params, true); // true表示exactInput
        bytes memory result = poolManager.unlock(data);
        amountOut = abi.decode(result, (uint256));
        
        if (amountOut < params.amountOutMinimum) {
            revert SlippageExceeded(params.amountOutMinimum, amountOut);
        }
    }
    
    function unlockCallback(bytes calldata data) 
        external 
        override 
        onlyPoolManager 
        returns (bytes memory) 
    {
        (ExactInputParams memory params, bool isExactInput) = 
            abi.decode(data, (ExactInputParams, bool));
        
        BalanceDelta delta = poolManager.swap(
            params.key,
            IPoolManager.SwapParams({
                zeroForOne: params.zeroForOne,
                amountSpecified: int256(params.amountIn),
                sqrtPriceLimitX96: params.sqrtPriceLimitX96
            }),
            ""
        );
        
        uint256 amountOut;
        
        if (params.zeroForOne) {
            _settle(params.key.currency0, params.amountIn);
            amountOut = uint256(uint128(-delta.amount1()));
            _take(params.key.currency1, msg.sender, amountOut);
        } else {
            _settle(params.key.currency1, params.amountIn);
            amountOut = uint256(uint128(-delta.amount0()));
            _take(params.key.currency0, msg.sender, amountOut);
        }
        
        return abi.encode(amountOut);
    }
}
```

## 流动性管理

### 全范围流动性提供者

```solidity
contract FullRangeLiquidityProvider is V4Integration {
    using TickMath for int24;
    
    struct MintParams {
        PoolKey key;
        uint256 amount0Desired;
        uint256 amount1Desired;
        uint256 amount0Min;
        uint256 amount1Min;
        address recipient;
        uint256 deadline;
    }
    
    mapping(bytes32 => uint256) public userLiquidity;
    
    function addLiquidity(MintParams calldata params) 
        external 
        payable 
        returns (uint128 liquidity, uint256 amount0, uint256 amount1) 
    {
        require(block.timestamp <= params.deadline, "Deadline exceeded");
        
        bytes memory data = abi.encode(params);
        bytes memory result = poolManager.unlock(data);
        (liquidity, amount0, amount1) = abi.decode(result, (uint128, uint256, uint256));
        
        require(amount0 >= params.amount0Min, "Amount0 too low");
        require(amount1 >= params.amount1Min, "Amount1 too low");
    }
    
    function unlockCallback(bytes calldata data) 
        external 
        override 
        onlyPoolManager 
        returns (bytes memory) 
    {
        MintParams memory params = abi.decode(data, (MintParams));
        
        // 使用全价格范围
        (BalanceDelta delta, BalanceDelta fees) = poolManager.modifyLiquidity(
            params.key,
            IPoolManager.ModifyLiquidityParams({
                tickLower: TickMath.MIN_TICK,
                tickUpper: TickMath.MAX_TICK,
                liquidityDelta: int256(_calculateLiquidityDelta(params)),
                salt: bytes32(0)
            }),
            ""
        );
        
        uint256 amount0 = uint256(uint128(delta.amount0()));
        uint256 amount1 = uint256(uint128(delta.amount1()));
        
        // 支付代币
        if (amount0 > 0) _settle(params.key.currency0, amount0);
        if (amount1 > 0) _settle(params.key.currency1, amount1);
        
        // 记录用户流动性
        bytes32 positionKey = keccak256(abi.encode(
            params.recipient,
            TickMath.MIN_TICK,
            TickMath.MAX_TICK,
            bytes32(0)
        ));
        userLiquidity[positionKey] += uint256(int256(delta.amount0() + delta.amount1()));
        
        return abi.encode(uint128(amount0 + amount1), amount0, amount1);
    }
    
    function _calculateLiquidityDelta(MintParams memory params) 
        private 
        view 
        returns (uint128) 
    {
        // 简化实现：基于输入代币数量计算流动性
        return uint128(params.amount0Desired + params.amount1Desired);
    }
}
```

### 集中流动性策略

```solidity
contract ConcentratedLiquidityStrategy is V4Integration {
    struct PositionInfo {
        uint128 liquidity;
        int24 tickLower;
        int24 tickUpper;
        uint256 feeGrowthInside0LastX128;
        uint256 feeGrowthInside1LastX128;
        uint128 tokensOwed0;
        uint128 tokensOwed1;
    }
    
    mapping(address => mapping(bytes32 => PositionInfo)) public positions;
    
    function createPosition(
        PoolKey memory key,
        int24 tickLower,
        int24 tickUpper,
        uint128 liquidity
    ) external returns (bytes32 positionId) {
        positionId = keccak256(abi.encode(msg.sender, tickLower, tickUpper, block.timestamp));
        
        bytes memory data = abi.encode(key, tickLower, tickUpper, int256(uint256(liquidity)));
        poolManager.unlock(data);
        
        positions[msg.sender][positionId] = PositionInfo({
            liquidity: liquidity,
            tickLower: tickLower,
            tickUpper: tickUpper,
            feeGrowthInside0LastX128: 0,
            feeGrowthInside1LastX128: 0,
            tokensOwed0: 0,
            tokensOwed1: 0
        });
    }
    
    function rebalancePosition(
        bytes32 positionId,
        PoolKey memory key,
        int24 newTickLower,
        int24 newTickUpper
    ) external {
        PositionInfo storage position = positions[msg.sender][positionId];
        require(position.liquidity > 0, "Position not found");
        
        // 移除当前流动性
        _removeLiquidity(key, position);
        
        // 在新范围添加流动性
        position.tickLower = newTickLower;
        position.tickUpper = newTickUpper;
        _addLiquidity(key, position);
    }
    
    function _removeLiquidity(PoolKey memory key, PositionInfo storage position) private {
        bytes memory data = abi.encode(
            key, 
            position.tickLower, 
            position.tickUpper, 
            -int256(uint256(position.liquidity))
        );
        poolManager.unlock(data);
    }
    
    function _addLiquidity(PoolKey memory key, PositionInfo storage position) private {
        bytes memory data = abi.encode(
            key, 
            position.tickLower, 
            position.tickUpper, 
            int256(uint256(position.liquidity))
        );
        poolManager.unlock(data);
    }
}
```

## Hook开发实例

### 动态手续费Hook

```solidity
import {BaseHook} from "v4-core/BaseHook.sol";
import {Hooks} from "v4-core/libraries/Hooks.sol";
import {IPoolManager} from "v4-core/interfaces/IPoolManager.sol";
import {PoolKey} from "v4-core/types/PoolKey.sol";
import {BeforeSwapDelta, BeforeSwapDeltaLibrary} from "v4-core/types/BeforeSwapDelta.sol";

contract DynamicFeeHook is BaseHook {
    using BeforeSwapDeltaLibrary for BeforeSwapDelta;
    
    // 最小和最大手续费 (万分比)
    uint24 public constant MIN_FEE = 500;   // 0.05%
    uint24 public constant MAX_FEE = 10000; // 1%
    
    // 波动性计算的时间窗口
    uint256 public constant VOLATILITY_WINDOW = 1 hours;
    
    struct PoolState {
        uint256 lastUpdateTime;
        uint160 lastSqrtPriceX96;
        uint256 priceChangeSum;
        uint24 currentFee;
    }
    
    mapping(PoolId => PoolState) public poolStates;
    
    constructor(IPoolManager _poolManager) BaseHook(_poolManager) {}
    
    function getHookPermissions() public pure override returns (Hooks.Permissions memory) {
        return Hooks.Permissions({
            beforeInitialize: false,
            afterInitialize: true,
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
    
    function afterInitialize(
        address,
        PoolKey calldata key,
        uint160 sqrtPriceX96,
        int24
    ) external override returns (bytes4) {
        PoolId poolId = key.toId();
        poolStates[poolId] = PoolState({
            lastUpdateTime: block.timestamp,
            lastSqrtPriceX96: sqrtPriceX96,
            priceChangeSum: 0,
            currentFee: MIN_FEE
        });
        
        return BaseHook.afterInitialize.selector;
    }
    
    function beforeSwap(
        address,
        PoolKey calldata key,
        IPoolManager.SwapParams calldata,
        bytes calldata
    ) external override returns (bytes4, BeforeSwapDelta, uint24) {
        PoolId poolId = key.toId();
        PoolState storage state = poolStates[poolId];
        
        // 更新波动性和动态手续费
        uint24 newFee = _updateFeeBasedOnVolatility(poolId, state);
        
        return (
            BaseHook.beforeSwap.selector,
            BeforeSwapDeltaLibrary.ZERO_DELTA,
            newFee
        );
    }
    
    function _updateFeeBasedOnVolatility(
        PoolId poolId,
        PoolState storage state
    ) private returns (uint24) {
        // 获取当前价格
        (uint160 currentSqrtPriceX96,,,) = poolManager.getSlot0(poolId);
        
        uint256 timeDelta = block.timestamp - state.lastUpdateTime;
        
        if (timeDelta > 0 && state.lastSqrtPriceX96 > 0) {
            // 计算价格变化率
            uint256 priceChange = _abs(
                int256(uint256(currentSqrtPriceX96)) - 
                int256(uint256(state.lastSqrtPriceX96))
            );
            
            uint256 priceChangeRate = (priceChange * 1e18) / uint256(state.lastSqrtPriceX96);
            
            // 更新累计价格变化
            if (timeDelta < VOLATILITY_WINDOW) {
                state.priceChangeSum += priceChangeRate;
            } else {
                state.priceChangeSum = priceChangeRate;
            }
            
            // 基于波动性计算新手续费
            uint256 volatility = state.priceChangeSum;
            uint24 newFee = MIN_FEE + uint24((volatility * (MAX_FEE - MIN_FEE)) / 1e18);
            
            if (newFee > MAX_FEE) newFee = MAX_FEE;
            
            state.currentFee = newFee;
            state.lastUpdateTime = block.timestamp;
            state.lastSqrtPriceX96 = currentSqrtPriceX96;
        }
        
        return state.currentFee;
    }
    
    function _abs(int256 x) private pure returns (uint256) {
        return x >= 0 ? uint256(x) : uint256(-x);
    }
}
```

### 限价单Hook

```solidity
contract LimitOrderHook is BaseHook {
    using TickMath for int24;
    
    struct LimitOrder {
        address owner;
        int24 tick;
        bool zeroForOne;
        uint128 amount;
        bool filled;
    }
    
    mapping(PoolId => mapping(int24 => LimitOrder[])) public limitOrders;
    mapping(address => uint256[]) public userOrders;
    
    uint256 private nextOrderId;
    
    constructor(IPoolManager _poolManager) BaseHook(_poolManager) {}
    
    function getHookPermissions() public pure override returns (Hooks.Permissions memory) {
        return Hooks.Permissions({
            beforeInitialize: false,
            afterInitialize: false,
            beforeAddLiquidity: false,
            afterAddLiquidity: false,
            beforeRemoveLiquidity: false,
            afterRemoveLiquidity: false,
            beforeSwap: false,
            afterSwap: true,
            beforeDonate: false,
            afterDonate: false,
            beforeSwapReturnDelta: false,
            afterSwapReturnDelta: true,
            afterAddLiquidityReturnDelta: false,
            afterRemoveLiquidityReturnDelta: false
        });
    }
    
    function placeLimitOrder(
        PoolKey calldata key,
        int24 tick,
        bool zeroForOne,
        uint128 amount
    ) external returns (uint256 orderId) {
        orderId = nextOrderId++;
        
        limitOrders[key.toId()][tick].push(LimitOrder({
            owner: msg.sender,
            tick: tick,
            zeroForOne: zeroForOne,
            amount: amount,
            filled: false
        }));
        
        userOrders[msg.sender].push(orderId);
        
        // 收取代币
        Currency inputCurrency = zeroForOne ? key.currency0 : key.currency1;
        IERC20(Currency.unwrap(inputCurrency)).transferFrom(
            msg.sender,
            address(this),
            amount
        );
    }
    
    function afterSwap(
        address,
        PoolKey calldata key,
        IPoolManager.SwapParams calldata params,
        BalanceDelta delta,
        bytes calldata
    ) external override returns (bytes4, BalanceDelta) {
        // 获取交换后的当前tick
        (,int24 currentTick,,) = poolManager.getSlot0(key.toId());
        
        // 检查并执行可以触发的限价单
        BalanceDelta hookDelta = _executeLimitOrders(key, currentTick, params.zeroForOne);
        
        return (BaseHook.afterSwap.selector, hookDelta);
    }
    
    function _executeLimitOrders(
        PoolKey calldata key,
        int24 currentTick,
        bool swapDirection
    ) private returns (BalanceDelta hookDelta) {
        PoolId poolId = key.toId();
        
        // 检查可执行的限价单范围
        int24 startTick = swapDirection ? currentTick - 100 : currentTick;
        int24 endTick = swapDirection ? currentTick : currentTick + 100;
        
        for (int24 tick = startTick; tick <= endTick; tick++) {
            LimitOrder[] storage orders = limitOrders[poolId][tick];
            
            for (uint256 i = 0; i < orders.length; i++) {
                if (!orders[i].filled && _shouldExecuteOrder(orders[i], currentTick)) {
                    _executeOrder(key, orders[i]);
                    orders[i].filled = true;
                }
            }
        }
        
        return toBalanceDelta(0, 0); // 简化实现
    }
    
    function _shouldExecuteOrder(
        LimitOrder memory order,
        int24 currentTick
    ) private pure returns (bool) {
        if (order.zeroForOne) {
            return currentTick <= order.tick;
        } else {
            return currentTick >= order.tick;
        }
    }
    
    function _executeOrder(PoolKey calldata key, LimitOrder memory order) private {
        // 执行限价单交换
        // 这里需要调用poolManager.swap()来执行实际的交换
        // 简化实现，实际需要更复杂的逻辑
    }
}
```

## 高级应用模式

### 多池路由器

```solidity
contract MultiPoolRouter is V4Integration {
    struct ExactInputParams {
        bytes path;        // 编码的路径：token0, fee, token1, fee, token2...
        uint256 amountIn;
        uint256 amountOutMinimum;
    }
    
    function exactInput(ExactInputParams calldata params) 
        external 
        payable 
        returns (uint256 amountOut) 
    {
        bytes memory data = abi.encode(params);
        bytes memory result = poolManager.unlock(data);
        amountOut = abi.decode(result, (uint256));
        
        require(amountOut >= params.amountOutMinimum, "Insufficient output");
    }
    
    function unlockCallback(bytes calldata data) 
        external 
        override 
        onlyPoolManager 
        returns (bytes memory) 
    {
        ExactInputParams memory params = abi.decode(data, (ExactInputParams));
        
        uint256 amountOut = _performSwaps(params.path, params.amountIn);
        return abi.encode(amountOut);
    }
    
    function _performSwaps(bytes memory path, uint256 amountIn) 
        private 
        returns (uint256 amountOut) 
    {
        uint256 pathLength = path.length;
        require(pathLength >= 43, "Invalid path"); // 至少包含两个token和一个fee
        
        amountOut = amountIn;
        
        for (uint256 i = 0; i < pathLength; i += 43) { // 20 + 3 + 20 = 43字节每跳
            if (i + 43 > pathLength) break;
            
            // 解析路径
            (Currency currencyIn, uint24 fee, Currency currencyOut) = _parsePath(path, i);
            
            // 构造PoolKey
            PoolKey memory key = PoolKey({
                currency0: currencyIn < currencyOut ? currencyIn : currencyOut,
                currency1: currencyIn < currencyOut ? currencyOut : currencyIn,
                fee: fee,
                tickSpacing: _getTickSpacing(fee),
                hooks: IHooks(address(0))
            });
            
            bool zeroForOne = currencyIn < currencyOut;
            
            // 执行交换
            BalanceDelta delta = poolManager.swap(
                key,
                IPoolManager.SwapParams({
                    zeroForOne: zeroForOne,
                    amountSpecified: int256(amountOut),
                    sqrtPriceLimitX96: 0
                }),
                ""
            );
            
            // 处理代币转账
            if (i == 0) {
                // 第一次交换，用户支付输入代币
                _settle(currencyIn, amountOut);
            }
            
            if (i + 43 >= pathLength) {
                // 最后一次交换，用户收到输出代币
                amountOut = uint256(uint128(zeroForOne ? -delta.amount1() : -delta.amount0()));
                _take(currencyOut, msg.sender, amountOut);
            } else {
                // 中间交换，继续到下一跳
                amountOut = uint256(uint128(zeroForOne ? -delta.amount1() : -delta.amount0()));
            }
        }
    }
    
    function _parsePath(bytes memory path, uint256 offset) 
        private 
        pure 
        returns (Currency currencyIn, uint24 fee, Currency currencyOut) 
    {
        // 从path中解析token地址和fee
        // 简化实现
        assembly {
            currencyIn := mload(add(add(path, 0x20), offset))
            fee := mload(add(add(path, 0x20), add(offset, 20)))
            currencyOut := mload(add(add(path, 0x20), add(offset, 23)))
        }
    }
    
    function _getTickSpacing(uint24 fee) private pure returns (int24) {
        if (fee == 500) return 10;
        if (fee == 3000) return 60;
        if (fee == 10000) return 200;
        return 60; // 默认值
    }
}
```

### 套利机器人

```solidity
contract ArbitrageBot is V4Integration {
    struct ArbitrageParams {
        PoolKey[] keys;
        uint256 amountIn;
        uint256 minProfit;
    }
    
    function executeArbitrage(ArbitrageParams calldata params) 
        external 
        returns (uint256 profit) 
    {
        bytes memory data = abi.encode(params);
        bytes memory result = poolManager.unlock(data);
        profit = abi.decode(result, (uint256));
        
        require(profit >= params.minProfit, "Insufficient profit");
    }
    
    function unlockCallback(bytes calldata data) 
        external 
        override 
        onlyPoolManager 
        returns (bytes memory) 
    {
        ArbitrageParams memory params = abi.decode(data, (ArbitrageParams));
        
        uint256 currentAmount = params.amountIn;
        Currency currentCurrency = params.keys[0].currency0;
        
        // 执行循环套利
        for (uint256 i = 0; i < params.keys.length; i++) {
            PoolKey memory key = params.keys[i];
            bool zeroForOne = currentCurrency == key.currency0;
            
            BalanceDelta delta = poolManager.swap(
                key,
                IPoolManager.SwapParams({
                    zeroForOne: zeroForOne,
                    amountSpecified: int256(currentAmount),
                    sqrtPriceLimitX96: 0
                }),
                ""
            );
            
            if (i == 0) {
                // 第一次交换，需要支付输入代币
                _settle(currentCurrency, currentAmount);
            }
            
            // 更新下一轮的输入
            currentAmount = uint256(uint128(zeroForOne ? -delta.amount1() : -delta.amount0()));
            currentCurrency = zeroForOne ? key.currency1 : key.currency0;
        }
        
        // 计算利润
        uint256 profit = currentAmount > params.amountIn ? 
            currentAmount - params.amountIn : 0;
            
        // 提取最终代币
        _take(currentCurrency, msg.sender, currentAmount);
        
        return abi.encode(profit);
    }
}
```

## 安全最佳实践

### 重入保护

```solidity
contract SecureV4Integration is V4Integration, ReentrancyGuard {
    using SafeERC20 for IERC20;
    
    modifier onlyEOA() {
        require(tx.origin == msg.sender, "Only EOA allowed");
        _;
    }
    
    modifier validDeadline(uint256 deadline) {
        require(block.timestamp <= deadline, "Transaction expired");
        _;
    }
    
    // 所有外部函数都应用重入保护
    function secureSwap(
        PoolKey calldata key,
        IPoolManager.SwapParams calldata params,
        uint256 deadline
    ) external payable nonReentrant onlyEOA validDeadline(deadline) {
        // 交换逻辑
    }
}
```

### 参数验证

```solidity
contract ValidatedV4Integration is V4Integration {
    using SafeERC20 for IERC20;
    
    error InvalidPoolKey();
    error InvalidAmount();
    error InvalidSlippage();
    
    function _validatePoolKey(PoolKey memory key) internal pure {
        if (Currency.unwrap(key.currency0) >= Currency.unwrap(key.currency1)) {
            revert InvalidPoolKey();
        }
        if (key.fee > 1000000) { // 100%
            revert InvalidPoolKey();
        }
    }
    
    function _validateAmount(uint256 amount) internal pure {
        if (amount == 0) revert InvalidAmount();
    }
    
    function _validateSlippage(uint256 minOut, uint256 actualOut) internal pure {
        if (actualOut < minOut) revert InvalidSlippage();
    }
    
    function safeSwap(
        PoolKey calldata key,
        IPoolManager.SwapParams calldata params,
        uint256 minAmountOut
    ) external payable {
        _validatePoolKey(key);
        _validateAmount(uint256(params.amountSpecified));
        
        // 执行交换
        bytes memory data = abi.encode(key, params, minAmountOut);
        bytes memory result = poolManager.unlock(data);
        uint256 amountOut = abi.decode(result, (uint256));
        
        _validateSlippage(minAmountOut, amountOut);
    }
}
```

## Gas优化技巧

### 批量操作

```solidity
contract BatchV4Operations is V4Integration {
    struct BatchSwapParams {
        PoolKey[] keys;
        IPoolManager.SwapParams[] swapParams;
        uint256[] minAmountsOut;
    }
    
    function batchSwap(BatchSwapParams calldata params) 
        external 
        payable 
        returns (uint256[] memory amountsOut) 
    {
        require(
            params.keys.length == params.swapParams.length &&
            params.swapParams.length == params.minAmountsOut.length,
            "Array length mismatch"
        );
        
        bytes memory data = abi.encode(params);
        bytes memory result = poolManager.unlock(data);
        amountsOut = abi.decode(result, (uint256[]));
    }
    
    function unlockCallback(bytes calldata data) 
        external 
        override 
        onlyPoolManager 
        returns (bytes memory) 
    {
        BatchSwapParams memory params = abi.decode(data, (BatchSwapParams));
        uint256[] memory amountsOut = new uint256[](params.keys.length);
        
        for (uint256 i = 0; i < params.keys.length; i++) {
            BalanceDelta delta = poolManager.swap(
                params.keys[i],
                params.swapParams[i],
                ""
            );
            
            // 计算输出金额
            amountsOut[i] = params.swapParams[i].zeroForOne ?
                uint256(uint128(-delta.amount1())) :
                uint256(uint128(-delta.amount0()));
                
            // 处理代币转账
            _handleTokenTransfers(params.keys[i], params.swapParams[i], delta);
        }
        
        return abi.encode(amountsOut);
    }
    
    function _handleTokenTransfers(
        PoolKey memory key,
        IPoolManager.SwapParams memory swapParams,
        BalanceDelta delta
    ) private {
        if (swapParams.zeroForOne) {
            if (delta.amount0() > 0) {
                _settle(key.currency0, uint256(uint128(delta.amount0())));
            }
            if (delta.amount1() < 0) {
                _take(key.currency1, msg.sender, uint256(uint128(-delta.amount1())));
            }
        } else {
            if (delta.amount1() > 0) {
                _settle(key.currency1, uint256(uint128(delta.amount1())));
            }
            if (delta.amount0() < 0) {
                _take(key.currency0, msg.sender, uint256(uint128(-delta.amount0())));
            }
        }
    }
}
```

## 测试策略

### 单元测试示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import {IPoolManager} from "v4-core/interfaces/IPoolManager.sol";
import {PoolManager} from "v4-core/PoolManager.sol";
import {PoolKey} from "v4-core/types/PoolKey.sol";
import {Currency, CurrencyLibrary} from "v4-core/types/Currency.sol";
import {MockERC20} from "solmate/test/utils/mocks/MockERC20.sol";

contract V4IntegrationTest is Test {
    using CurrencyLibrary for Currency;
    
    PoolManager poolManager;
    MockERC20 token0;
    MockERC20 token1;
    PoolKey key;
    
    address user = makeAddr("user");
    
    function setUp() public {
        // 部署合约
        poolManager = new PoolManager();
        token0 = new MockERC20("Token0", "T0", 18);
        token1 = new MockERC20("Token1", "T1", 18);
        
        // 确保token0 < token1
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
        
        // 初始化池
        poolManager.initialize(key, SQRT_PRICE_1_1, "");
        
        // 给用户代币
        token0.mint(user, 1000 ether);
        token1.mint(user, 1000 ether);
    }
    
    function testBasicSwap() public {
        vm.startPrank(user);
        
        // 授权
        token0.approve(address(poolManager), type(uint256).max);
        token1.approve(address(poolManager), type(uint256).max);
        
        // 执行交换
        SimpleSwapper swapper = new SimpleSwapper(poolManager);
        
        uint256 amountIn = 1 ether;
        uint256 balanceBefore = token1.balanceOf(user);
        
        swapper.swap(SimpleSwapper.SwapParams({
            key: key,
            zeroForOne: true,
            amountSpecified: int256(amountIn),
            sqrtPriceLimitX96: 0
        }));
        
        uint256 balanceAfter = token1.balanceOf(user);
        
        // 验证结果
        assertGt(balanceAfter, balanceBefore, "Should receive token1");
        
        vm.stopPrank();
    }
    
    function testSlippageProtection() public {
        vm.startPrank(user);
        
        SlippageProtectedSwapper swapper = new SlippageProtectedSwapper(poolManager);
        
        // 设置过高的最小输出，应该revert
        vm.expectRevert();
        swapper.exactInputSingle(SlippageProtectedSwapper.ExactInputParams({
            key: key,
            zeroForOne: true,
            amountIn: 1 ether,
            amountOutMinimum: 1000 ether, // 不现实的高期望
            sqrtPriceLimitX96: 0
        }));
        
        vm.stopPrank();
    }
    
    // 模糊测试
    function testFuzzSwap(uint256 amountIn) public {
        amountIn = bound(amountIn, 1e6, 100 ether); // 限制范围
        
        vm.startPrank(user);
        
        // 确保用户有足够代币
        if (token0.balanceOf(user) < amountIn) {
            token0.mint(user, amountIn);
        }
        
        SimpleSwapper swapper = new SimpleSwapper(poolManager);
        
        uint256 balanceBefore = token1.balanceOf(user);
        
        swapper.swap(SimpleSwapper.SwapParams({
            key: key,
            zeroForOne: true,
            amountSpecified: int256(amountIn),
            sqrtPriceLimitX96: 0
        }));
        
        uint256 balanceAfter = token1.balanceOf(user);
        
        assertGt(balanceAfter, balanceBefore, "Should receive some token1");
        
        vm.stopPrank();
    }
}
```

### 集成测试

```solidity
contract V4IntegrationIntegrationTest is Test {
    function testMultiPoolArbitrage() public {
        // 创建多个池
        // 设置价格差异
        // 执行套利
        // 验证盈利
    }
    
    function testHookInteraction() public {
        // 部署自定义Hook
        // 创建带Hook的池
        // 测试Hook行为
        // 验证状态变化
    }
    
    function testLiquidityProvision() public {
        // 添加流动性
        // 执行交换
        // 移除流动性
        // 验证手续费收益
    }
}
```

## 总结

这些代码实例和最佳实践覆盖了Uniswap V4开发的主要场景：

1. **基础集成**：正确的合约结构和安全模式
2. **交换操作**：从简单到复杂的交换实现
3. **流动性管理**：全范围和集中流动性策略
4. **Hook开发**：动态手续费和限价单等高级功能
5. **高级应用**：多池路由和套利机器人
6. **安全实践**：重入保护、参数验证、错误处理
7. **性能优化**：批量操作和Gas优化
8. **测试策略**：单元测试、集成测试、模糊测试

关键要点：
- 始终实现`IUnlockCallback`接口
- 正确处理Delta余额管理
- 实施适当的安全检查
- 考虑Gas优化
- 充分测试边界情况

这些模式和实践将帮助开发者构建安全、高效的Uniswap V4集成应用。