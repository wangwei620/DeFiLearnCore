# Uniswap V4 代码实现分析

## 🏗️ 核心合约架构

### 1. PoolManager 合约

PoolManager 是 Uniswap V4 的核心管理合约，负责池子的创建、管理和交易执行。

```solidity
contract PoolManager {
    // 池子映射
    mapping(PoolId => Pool) public pools;
    
    // 事件
    event PoolInitialized(
        PoolId indexed poolId,
        Currency indexed currency0,
        Currency indexed currency1,
        uint24 fee,
        int24 tickSpacing,
        IHooks hooks
    );
    
    event Swap(
        PoolId indexed poolId,
        address indexed sender,
        address indexed recipient,
        int256 amount0,
        int256 amount1,
        uint160 sqrtPriceX96,
        uint128 liquidity,
        int24 tick,
        uint24 fee
    );
    
    // 核心函数
    function initialize(
        PoolKey memory key,
        uint160 sqrtPriceX96,
        bytes calldata hookData
    ) external returns (PoolId poolId) {
        poolId = getPoolId(key);
        
        // 检查池子是否已存在
        require(pools[poolId] == address(0), "Pool already exists");
        
        // 创建新池子
        Pool pool = new Pool(poolId);
        pools[poolId] = pool;
        
        // 初始化池子
        pool.initialize(sqrtPriceX96);
        
        // 调用 Hook
        if (key.hooks != address(0)) {
            key.hooks.beforeInitialize(address(this), key, sqrtPriceX96, hookData);
            key.hooks.afterInitialize(address(this), key, sqrtPriceX96, pool.tick(), hookData);
        }
        
        emit PoolInitialized(poolId, key.currency0, key.currency1, key.fee, key.tickSpacing, key.hooks);
    }
    
    function swap(
        PoolKey memory key,
        IPoolManager.SwapParams memory params,
        bytes calldata hookData
    ) external returns (BalanceDelta delta) {
        PoolId poolId = getPoolId(key);
        Pool pool = pools[poolId];
        
        // 调用 Hook
        if (key.hooks != address(0)) {
            key.hooks.beforeSwap(address(this), key, params, hookData);
        }
        
        // 执行交换
        delta = pool.swap(params);
        
        // 调用 Hook
        if (key.hooks != address(0)) {
            key.hooks.afterSwap(address(this), key, params, delta, hookData);
        }
        
        // 转移代币
        _accountDelta(key.currency0, params.recipient, delta.amount0());
        _accountDelta(key.currency1, params.recipient, delta.amount1());
        
        emit Swap(poolId, params.sender, params.recipient, delta.amount0(), delta.amount1(), pool.sqrtPriceX96(), pool.liquidity(), pool.tick(), key.fee);
    }
}
```

### 2. Pool 合约

Pool 合约是单例池的核心实现，包含所有的池子逻辑。

```solidity
contract Pool {
    // 池子标识
    PoolId public immutable poolId;
    
    // 池子状态
    Slot0 public slot0;
    
    // 流动性信息
    mapping(int24 => Info) public ticks;
    
    // 位置信息
    mapping(bytes32 => Position.Info) public positions;
    
    // 锁定状态
    modifier lock() {
        require(slot0.unlocked, "LOK");
        slot0.unlocked = false;
        _;
        slot0.unlocked = true;
    }
    
    // 初始化函数
    function initialize(uint160 sqrtPriceX96) external {
        require(slot0.sqrtPriceX96 == 0, "Already initialized");
        
        slot0.sqrtPriceX96 = sqrtPriceX96;
        slot0.tick = getTickAtSqrtRatio(sqrtPriceX96);
        slot0.unlocked = true;
    }
    
    // 交换函数
    function swap(
        IPoolManager.SwapParams memory params
    ) external lock returns (BalanceDelta delta) {
        // 验证参数
        require(params.amountSpecified != 0, "Zero amount");
        require(params.sqrtPriceLimitX96 != 0, "Zero price limit");
        
        // 执行交换逻辑
        SwapState memory state = SwapState({
            sqrtPriceX96: slot0.sqrtPriceX96,
            tick: slot0.tick,
            liquidity: liquidity,
            amountSpecifiedRemaining: params.amountSpecified,
            amountCalculated: 0
        });
        
        // 计算交换结果
        while (state.amountSpecifiedRemaining != 0 && state.sqrtPriceX96 != params.sqrtPriceLimitX96) {
            // 计算下一个 tick
            int24 nextTick = getNextTick(state.tick, params.zeroForOne);
            
            // 计算交换数量
            uint160 sqrtPriceNextX96 = getSqrtRatioAtTick(nextTick);
            
            // 检查价格限制
            if (params.zeroForOne) {
                if (sqrtPriceNextX96 < params.sqrtPriceLimitX96) {
                    sqrtPriceNextX96 = params.sqrtPriceLimitX96;
                }
            } else {
                if (sqrtPriceNextX96 > params.sqrtPriceLimitX96) {
                    sqrtPriceNextX96 = params.sqrtPriceLimitX96;
                }
            }
            
            // 计算交换数量
            uint256 amountIn = computeSwapStep(
                state.sqrtPriceX96,
                sqrtPriceNextX96,
                state.liquidity,
                state.amountSpecifiedRemaining,
                params.zeroForOne
            );
            
            // 更新状态
            state.amountSpecifiedRemaining -= amountIn;
            state.amountCalculated += amountIn;
            state.sqrtPriceX96 = sqrtPriceNextX96;
            
            // 更新 tick
            if (state.sqrtPriceX96 == getSqrtRatioAtTick(nextTick)) {
                state.tick = nextTick;
                updateTick(nextTick);
            }
        }
        
        // 更新池子状态
        slot0.sqrtPriceX96 = state.sqrtPriceX96;
        slot0.tick = state.tick;
        
        // 计算代币数量
        if (params.zeroForOne) {
            delta = BalanceDelta.wrap(-int256(state.amountCalculated));
        } else {
            delta = BalanceDelta.wrap(int256(state.amountCalculated));
        }
        
        return delta;
    }
}
```

### 3. Router 合约

Router 合约提供用户友好的接口，处理复杂的交易路径。

```solidity
contract Router {
    IPoolManager public immutable poolManager;
    
    constructor(IPoolManager _poolManager) {
        poolManager = _poolManager;
    }
    
    // 单跳交换
    function exactInputSingle(
        ExactInputSingleParams calldata params
    ) external payable returns (uint256 amountOut) {
        // 验证参数
        require(params.deadline >= block.timestamp, "Expired");
        require(params.amountIn > 0, "Zero amount");
        
        // 准备交换参数
        IPoolManager.SwapParams memory swapParams = IPoolManager.SwapParams({
            sender: address(this),
            recipient: params.recipient,
            amountSpecified: int256(params.amountIn),
            zeroForOne: params.zeroForOne,
            sqrtPriceLimitX96: params.sqrtPriceLimitX96,
            hookData: params.hookData
        });
        
        // 执行交换
        BalanceDelta delta = poolManager.swap(params.poolKey, swapParams);
        
        // 计算输出数量
        amountOut = params.zeroForOne ? uint256(-delta.amount1()) : uint256(-delta.amount0());
        
        // 验证滑点
        require(amountOut >= params.amountOutMinimum, "Too little received");
    }
    
    // 多跳交换
    function exactInput(
        ExactInputParams calldata params
    ) external payable returns (uint256 amountOut) {
        // 验证参数
        require(params.deadline >= block.timestamp, "Expired");
        require(params.amountIn > 0, "Zero amount");
        
        // 解析路径
        bytes memory path = params.path;
        uint256 amountIn = params.amountIn;
        
        while (path.length > 0) {
            // 解析下一个池子
            PoolKey memory poolKey = abi.decode(path, (PoolKey));
            
            // 准备交换参数
            IPoolManager.SwapParams memory swapParams = IPoolManager.SwapParams({
                sender: address(this),
                recipient: path.length > 0 ? address(this) : params.recipient,
                amountSpecified: int256(amountIn),
                zeroForOne: poolKey.currency0 < poolKey.currency1,
                sqrtPriceLimitX96: 0,
                hookData: ""
            });
            
            // 执行交换
            BalanceDelta delta = poolManager.swap(poolKey, swapParams);
            
            // 更新输入数量
            amountIn = poolKey.currency0 < poolKey.currency1 ? 
                uint256(-delta.amount1()) : uint256(-delta.amount0());
        }
        
        amountOut = amountIn;
        
        // 验证滑点
        require(amountOut >= params.amountOutMinimum, "Too little received");
    }
}
```

## 🔧 关键算法实现

### 1. 价格计算算法

```solidity
library PriceMath {
    // 从 tick 计算价格
    function getSqrtRatioAtTick(int24 tick) internal pure returns (uint160) {
        require(tick >= MIN_TICK && tick <= MAX_TICK, "Tick out of range");
        
        uint256 absTick = tick < 0 ? uint256(-int256(tick)) : uint256(int256(tick));
        require(absTick <= uint256(uint24(MAX_TICK)), "Tick out of range");
        
        // 使用查找表计算价格
        uint256 ratio = absTick & 0x1 != 0 ? 0xfffcb933bd6fad37aa2d162d1a594001 : 0x100000000000000000000000000000000;
        if (absTick & 0x2 != 0) ratio = (ratio * 0xfff97272373d413259a46990580e213a) >> 128;
        if (absTick & 0x4 != 0) ratio = (ratio * 0xfff2e50f5f656932ef12357cf3c7fdcc) >> 128;
        if (absTick & 0x8 != 0) ratio = (ratio * 0xffe5caca7e10e4e61c3624eaa0941cd0) >> 128;
        if (absTick & 0x10 != 0) ratio = (ratio * 0xffcb9843d60f6159c9db58835c926644) >> 128;
        if (absTick & 0x20 != 0) ratio = (ratio * 0xff973b41fa98c081472e6896dfb254c0) >> 128;
        if (absTick & 0x40 != 0) ratio = (ratio * 0xff2ea16466c96a3843ec78b326b52861) >> 128;
        if (absTick & 0x80 != 0) ratio = (ratio * 0xfe5dee046a99a2a811c461f1969c3053) >> 128;
        if (absTick & 0x100 != 0) ratio = (ratio * 0xfcbe86c7900a88aedcffc83b479aa3a4) >> 128;
        if (absTick & 0x200 != 0) ratio = (ratio * 0xf987a7253ac413176f2b074cf7815e54) >> 128;
        if (absTick & 0x400 != 0) ratio = (ratio * 0xf3392b0822b70005940c7a398e4b70f3) >> 128;
        if (absTick & 0x800 != 0) ratio = (ratio * 0xe7159475a2c29b7443b29c7fa6e889d9) >> 128;
        if (absTick & 0x1000 != 0) ratio = (ratio * 0xd097f3bdfd2022b8845ad8f792aa5825) >> 128;
        if (absTick & 0x2000 != 0) ratio = (ratio * 0xa9f746462d870fdf8a65dc1f90e061e5) >> 128;
        if (absTick & 0x4000 != 0) ratio = (ratio * 0x70d869a1562d1a594001) >> 128;
        if (absTick & 0x8000 != 0) ratio = (ratio * 0x31be135f97d08fd981231505542fcfa6) >> 128;
        if (absTick & 0x10000 != 0) ratio = (ratio * 0x9aa508b5b7a84e1c677de54f3e99bc9) >> 128;
        if (absTick & 0x20000 != 0) ratio = (ratio * 0x5d6af8dedb81196699c329225ee604) >> 128;
        if (absTick & 0x40000 != 0) ratio = (ratio * 0x2216e584f5fa1ea926041bedfe98) >> 128;
        if (absTick & 0x80000 != 0) ratio = (ratio * 0x48a170391f7dc42444e8fa2) >> 128;
        
        if (tick > 0) ratio = type(uint256).max / ratio;
        
        return uint160((ratio >> 32) + (ratio % (1 << 32) == 0 ? 0 : 1));
    }
    
    // 从价格计算 tick
    function getTickAtSqrtRatio(uint160 sqrtPriceX96) internal pure returns (int24) {
        require(sqrtPriceX96 >= MIN_SQRT_RATIO && sqrtPriceX96 <= MAX_SQRT_RATIO, "Price out of range");
        
        uint256 ratio = uint256(sqrtPriceX96) << 32;
        
        uint256 r = ratio;
        uint256 msb = 0;
        
        assembly {
            let f := shl(7, gt(r, 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF))
            msb := or(msb, f)
            r := shr(f, r)
        }
        assembly {
            let f := shl(6, gt(r, 0xFFFFFFFFFFFFFFFF))
            msb := or(msb, f)
            r := shr(f, r)
        }
        assembly {
            let f := shl(5, gt(r, 0xFFFFFFFF))
            msb := or(msb, f)
            r := shr(f, r)
        }
        assembly {
            let f := shl(4, gt(r, 0xFFFF))
            msb := or(msb, f)
            r := shr(f, r)
        }
        assembly {
            let f := shl(3, gt(r, 0xFF))
            msb := or(msb, f)
            r := shr(f, r)
        }
        assembly {
            let f := shl(2, gt(r, 0xF))
            msb := or(msb, f)
            r := shr(f, r)
        }
        assembly {
            let f := shl(1, gt(r, 0x3))
            msb := or(msb, f)
            r := shr(f, r)
        }
        assembly {
            let f := gt(r, 0x1)
            msb := or(msb, f)
        }
        
        if (msb >= 128) r = ratio >> (msb - 127);
        else r = ratio << (127 - msb);
        
        int256 log_2 = (int256(msb) - 128) << 64;
        
        assembly {
            r := shr(127, mul(r, r))
            let f := shr(128, r)
            log_2 := or(log_2, shl(63, f))
            r := shr(f, r)
        }
        assembly {
            r := shr(127, mul(r, r))
            let f := shr(128, r)
            log_2 := or(log_2, shl(62, f))
            r := shr(f, r)
        }
        assembly {
            r := shr(127, mul(r, r))
            let f := shr(128, r)
            log_2 := or(log_2, shl(61, f))
            r := shr(f, r)
        }
        assembly {
            r := shr(127, mul(r, r))
            let f := shr(128, r)
            log_2 := or(log_2, shl(60, f))
            r := shr(f, r)
        }
        assembly {
            r := shr(127, mul(r, r))
            let f := shr(128, r)
            log_2 := or(log_2, shl(59, f))
            r := shr(f, r)
        }
        assembly {
            r := shr(127, mul(r, r))
            let f := shr(128, r)
            log_2 := or(log_2, shl(58, f))
            r := shr(f, r)
        }
        assembly {
            r := shr(127, mul(r, r))
            let f := shr(128, r)
            log_2 := or(log_2, shl(57, f))
            r := shr(f, r)
        }
        assembly {
            r := shr(127, mul(r, r))
            let f := shr(128, r)
            log_2 := or(log_2, shl(56, f))
            r := shr(f, r)
        }
        assembly {
            r := shr(127, mul(r, r))
            let f := shr(128, r)
            log_2 := or(log_2, shl(55, f))
            r := shr(f, r)
        }
        assembly {
            r := shr(127, mul(r, r))
            let f := shr(128, r)
            log_2 := or(log_2, shl(54, f))
            r := shr(f, r)
        }
        assembly {
            r := shr(127, mul(r, r))
            let f := shr(128, r)
            log_2 := or(log_2, shl(53, f))
            r := shr(f, r)
        }
        assembly {
            r := shr(127, mul(r, r))
            let f := shr(128, r)
            log_2 := or(log_2, shl(52, f))
            r := shr(f, r)
        }
        assembly {
            r := shr(127, mul(r, r))
            let f := shr(128, r)
            log_2 := or(log_2, shl(51, f))
            r := shr(f, r)
        }
        assembly {
            r := shr(127, mul(r, r))
            let f := shr(128, r)
            log_2 := or(log_2, shl(50, f))
        }
        
        int256 log_sqrt10001 = log_2 * 255738958999603826347141;
        
        int24 tickLow = int24((log_sqrt10001 - 3402992956809132418596140100660247210) >> 128);
        int24 tickHi = int24((log_sqrt10001 + 291339464771989622907027621153398088495) >> 128);
        
        tick = tickLow == tickHi ? tickLow : getSqrtRatioAtTick(tickHi) <= sqrtPriceX96 ? tickHi : tickLow;
    }
}
```

### 2. 流动性计算算法

```solidity
library LiquidityMath {
    // 计算流动性
    function getLiquidityForAmount0(
        uint160 sqrtPriceAX96,
        uint160 sqrtPriceBX96,
        uint256 amount0
    ) internal pure returns (uint128) {
        require(sqrtPriceAX96 < sqrtPriceBX96, "Invalid price range");
        
        uint256 intermediate = FullMath.mulDiv(sqrtPriceBX96 - sqrtPriceAX96, Q96, sqrtPriceAX96);
        return uint128(FullMath.mulDiv(amount0, intermediate, Q96));
    }
    
    function getLiquidityForAmount1(
        uint160 sqrtPriceAX96,
        uint160 sqrtPriceBX96,
        uint256 amount1
    ) internal pure returns (uint128) {
        require(sqrtPriceAX96 < sqrtPriceBX96, "Invalid price range");
        
        return uint128(FullMath.mulDiv(amount1, Q96, sqrtPriceBX96 - sqrtPriceAX96));
    }
    
    // 计算代币数量
    function getAmount0ForLiquidity(
        uint160 sqrtPriceAX96,
        uint160 sqrtPriceBX96,
        uint128 liquidity
    ) internal pure returns (uint256) {
        require(sqrtPriceAX96 < sqrtPriceBX96, "Invalid price range");
        
        uint256 intermediate = FullMath.mulDiv(sqrtPriceBX96 - sqrtPriceAX96, Q96, sqrtPriceBX96);
        return FullMath.mulDiv(liquidity, intermediate, Q96);
    }
    
    function getAmount1ForLiquidity(
        uint160 sqrtPriceAX96,
        uint160 sqrtPriceBX96,
        uint128 liquidity
    ) internal pure returns (uint256) {
        require(sqrtPriceAX96 < sqrtPriceBX96, "Invalid price range");
        
        return FullMath.mulDiv(liquidity, sqrtPriceBX96 - sqrtPriceAX96, Q96);
    }
}
```

### 3. 交换计算算法

```solidity
library SwapMath {
    // 计算交换步骤
    function computeSwapStep(
        uint160 sqrtPriceCurrentX96,
        uint160 sqrtPriceTargetX96,
        uint128 liquidity,
        uint256 amountRemaining,
        bool zeroForOne
    ) internal pure returns (uint256 amountIn, uint256 amountOut, uint160 sqrtPriceNextX96) {
        bool exactIn = amountRemaining >= 0;
        
        if (exactIn) {
            if (zeroForOne) {
                amountIn = amountRemaining;
                amountOut = getAmount1Delta(sqrtPriceTargetX96, sqrtPriceCurrentX96, liquidity, false);
            } else {
                amountIn = amountRemaining;
                amountOut = getAmount0Delta(sqrtPriceCurrentX96, sqrtPriceTargetX96, liquidity, false);
            }
        } else {
            if (zeroForOne) {
                amountOut = uint256(-amountRemaining);
                amountIn = getAmount0Delta(sqrtPriceTargetX96, sqrtPriceCurrentX96, liquidity, true);
            } else {
                amountOut = uint256(-amountRemaining);
                amountIn = getAmount1Delta(sqrtPriceCurrentX96, sqrtPriceTargetX96, liquidity, true);
            }
        }
        
        sqrtPriceNextX96 = sqrtPriceTargetX96;
    }
    
    // 计算代币数量变化
    function getAmount0Delta(
        uint160 sqrtPriceAX96,
        uint160 sqrtPriceBX96,
        uint128 liquidity,
        bool roundUp
    ) internal pure returns (uint256) {
        require(sqrtPriceAX96 < sqrtPriceBX96, "Invalid price range");
        
        uint256 numerator1 = uint256(liquidity) << 96;
        uint256 numerator2 = sqrtPriceBX96 - sqrtPriceAX96;
        
        if (roundUp) {
            return FullMath.mulDivRoundingUp(numerator1, numerator2, sqrtPriceBX96);
        } else {
            return FullMath.mulDiv(numerator1, numerator2, sqrtPriceBX96);
        }
    }
    
    function getAmount1Delta(
        uint160 sqrtPriceAX96,
        uint160 sqrtPriceBX96,
        uint128 liquidity,
        bool roundUp
    ) internal pure returns (uint256) {
        require(sqrtPriceAX96 < sqrtPriceBX96, "Invalid price range");
        
        if (roundUp) {
            return FullMath.mulDivRoundingUp(liquidity, sqrtPriceBX96 - sqrtPriceAX96, Q96);
        } else {
            return FullMath.mulDiv(liquidity, sqrtPriceBX96 - sqrtPriceAX96, Q96);
        }
    }
}
```

## 🔐 安全实现

### 1. 重入保护

```solidity
contract ReentrancyGuard {
    uint256 private constant _NOT_ENTERED = 1;
    uint256 private constant _ENTERED = 2;
    
    uint256 private _status;
    
    constructor() {
        _status = _NOT_ENTERED;
    }
    
    modifier nonReentrant() {
        require(_status != _ENTERED, "ReentrancyGuard: reentrant call");
        
        _status = _ENTERED;
        _;
        _status = _NOT_ENTERED;
    }
}
```

### 2. 访问控制

```solidity
contract AccessControl {
    mapping(bytes32 => mapping(address => bool)) private _roles;
    
    bytes32 public constant DEFAULT_ADMIN_ROLE = 0x00;
    
    modifier onlyRole(bytes32 role) {
        require(hasRole(role, msg.sender), "AccessControl: access denied");
        _;
    }
    
    function hasRole(bytes32 role, address account) public view returns (bool) {
        return _roles[role][account];
    }
    
    function grantRole(bytes32 role, address account) external onlyRole(DEFAULT_ADMIN_ROLE) {
        _roles[role][account] = true;
        emit RoleGranted(role, account, msg.sender);
    }
    
    function revokeRole(bytes32 role, address account) external onlyRole(DEFAULT_ADMIN_ROLE) {
        _roles[role][account] = false;
        emit RoleRevoked(role, account, msg.sender);
    }
}
```

### 3. 数学安全

```solidity
library SafeMath {
    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        uint256 c = a + b;
        require(c >= a, "SafeMath: addition overflow");
        return c;
    }
    
    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b <= a, "SafeMath: subtraction overflow");
        return a - b;
    }
    
    function mul(uint256 a, uint256 b) internal pure returns (uint256) {
        if (a == 0) return 0;
        uint256 c = a * b;
        require(c / a == b, "SafeMath: multiplication overflow");
        return c;
    }
    
    function div(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b > 0, "SafeMath: division by zero");
        return a / b;
    }
}
```

## 📊 性能优化

### 1. Gas 优化

```solidity
contract GasOptimized {
    // 使用紧凑的数据结构
    struct CompactData {
        uint128 value1;
        uint128 value2;
    }
    
    // 批量操作
    function batchProcess(uint256[] calldata data) external {
        uint256 length = data.length;
        for (uint256 i = 0; i < length;) {
            processItem(data[i]);
            unchecked { ++i; }
        }
    }
    
    // 使用 unchecked 优化
    function optimizedMath(uint256 a, uint256 b) internal pure returns (uint256) {
        unchecked {
            return a + b;
        }
    }
}
```

### 2. 存储优化

```solidity
contract StorageOptimized {
    // 使用位操作压缩存储
    struct PackedData {
        uint128 value1;
        uint64 value2;
        uint64 value3;
    }
    
    // 使用映射优化存储
    mapping(bytes32 => uint256) private _storage;
    
    function setValue(bytes32 key, uint256 value) external {
        _storage[key] = value;
    }
    
    function getValue(bytes32 key) external view returns (uint256) {
        return _storage[key];
    }
}
```

## 🔮 扩展实现

### 1. Hook 集成

```solidity
contract HookIntegration {
    IHooks public hooks;
    
    function executeWithHook(
        bytes calldata hookData
    ) external {
        if (address(hooks) != address(0)) {
            hooks.beforeExecute(hookData);
        }
        
        // 执行核心逻辑
        executeCore();
        
        if (address(hooks) != address(0)) {
            hooks.afterExecute(hookData);
        }
    }
}
```

### 2. 事件系统

```solidity
contract EventSystem {
    event PoolCreated(
        address indexed pool,
        address indexed token0,
        address indexed token1,
        uint24 fee
    );
    
    event SwapExecuted(
        address indexed pool,
        address indexed sender,
        address indexed recipient,
        uint256 amount0In,
        uint256 amount1In,
        uint256 amount0Out,
        uint256 amount1Out
    );
    
    function emitPoolCreated(address pool, address token0, address token1, uint24 fee) external {
        emit PoolCreated(pool, token0, token1, fee);
    }
    
    function emitSwapExecuted(
        address pool,
        address sender,
        address recipient,
        uint256 amount0In,
        uint256 amount1In,
        uint256 amount0Out,
        uint256 amount1Out
    ) external {
        emit SwapExecuted(pool, sender, recipient, amount0In, amount1In, amount0Out, amount1Out);
    }
}
```

### 3. 升级机制

```solidity
contract Upgradeable {
    address public implementation;
    address public admin;
    
    modifier onlyAdmin() {
        require(msg.sender == admin, "Not admin");
        _;
    }
    
    function upgrade(address newImplementation) external onlyAdmin {
        require(newImplementation != address(0), "Invalid implementation");
        implementation = newImplementation;
        emit Upgraded(newImplementation);
    }
    
    fallback() external {
        address impl = implementation;
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
    
    event Upgraded(address indexed implementation);
}
```