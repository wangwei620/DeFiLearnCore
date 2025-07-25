# Uniswap V4 核心流程图

本文档包含Uniswap V4各个核心操作的详细流程图，帮助开发者理解系统的工作原理。

## 目录
1. [系统架构概览](#系统架构概览)
2. [池初始化流程](#池初始化流程)
3. [流动性操作流程](#流动性操作流程)
4. [交换执行流程](#交换执行流程)
5. [Hook系统执行流程](#hook系统执行流程)
6. [Delta管理流程](#delta管理流程)
7. [安全检查流程](#安全检查流程)

## 系统架构概览

```mermaid
graph TB
    subgraph "用户层"
        User[用户/DApp]
        Router[路由合约]
    end
    
    subgraph "Uniswap V4 核心"
        PM[PoolManager]
        subgraph "Pool状态"
            P1[Pool 1]
            P2[Pool 2]
            P3[Pool N]
        end
        subgraph "系统组件"
            Hooks[Hook系统]
            Delta[Delta管理]
            Fee[费用管理]
            Lock[锁定机制]
        end
    end
    
    subgraph "外部合约"
        ERC20_1[ERC20 Token A]
        ERC20_2[ERC20 Token B]
        HookContract[自定义Hook合约]
    end
    
    User --> Router
    Router --> PM
    PM --> P1
    PM --> P2
    PM --> P3
    PM --> Hooks
    PM --> Delta
    PM --> Fee
    PM --> Lock
    Hooks --> HookContract
    PM --> ERC20_1
    PM --> ERC20_2
```

## 池初始化流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant PM as PoolManager
    participant Hook as Hook合约
    participant Pool as Pool状态
    
    User->>PM: initialize(poolKey, sqrtPriceX96)
    
    Note over PM: 验证PoolKey参数
    PM->>PM: 检查货币排序
    PM->>PM: 验证tickSpacing
    PM->>PM: 验证手续费
    
    Note over PM: Hook前置检查
    alt Hook存在且有beforeInitialize
        PM->>Hook: beforeInitialize(sender, key, sqrtPriceX96)
        Hook-->>PM: 返回selector
        PM->>PM: 验证返回值
    end
    
    Note over PM: 初始化Pool状态
    PM->>Pool: 创建新Pool状态
    PM->>Pool: 设置初始价格和tick
    PM->>Pool: 初始化tick bitmap
    
    Note over PM: Hook后置处理
    alt Hook存在且有afterInitialize
        PM->>Hook: afterInitialize(sender, key, sqrtPriceX96, tick)
        Hook-->>PM: 返回selector
        PM->>PM: 验证返回值
    end
    
    PM->>PM: 触发Initialize事件
    PM-->>User: 返回tick
```

## 流动性操作流程

### 添加流动性

```mermaid
sequenceDiagram
    participant User as 用户合约
    participant PM as PoolManager
    participant Hook as Hook合约
    participant Pool as Pool库
    participant Position as Position库
    
    User->>PM: unlock(callback_data)
    PM->>PM: 设置解锁状态
    PM->>User: unlockCallback(callback_data)
    
    User->>PM: modifyLiquidity(key, params, hookData)
    
    Note over PM: Hook前置处理
    alt Hook存在且有beforeAddLiquidity
        PM->>Hook: beforeAddLiquidity(sender, key, params, hookData)
        Hook-->>PM: 返回selector
    end
    
    Note over PM: 流动性计算
    PM->>Pool: 计算流动性变化
    Pool->>Pool: 检查tick范围有效性
    Pool->>Position: 获取当前position状态
    
    Note over Pool: 更新tick信息
    Pool->>Pool: 更新tickLower信息
    Pool->>Pool: 更新tickUpper信息
    Pool->>Pool: 更新tick bitmap
    
    Note over Pool: 更新position
    Pool->>Position: 更新position流动性
    Pool->>Position: 计算手续费增长
    
    Note over PM: Hook后置处理
    alt Hook存在且有afterAddLiquidity
        PM->>Hook: afterAddLiquidity(sender, key, params, delta, fees, hookData)
        Hook-->>PM: 返回(selector, hookDelta)
    end
    
    PM->>PM: 更新Delta余额
    PM->>PM: 触发ModifyLiquidity事件
    PM-->>User: 返回(delta, feesAccrued)
    
    User->>PM: settle/take操作结算代币
    PM->>PM: 检查Delta平衡
    PM-->>User: unlockCallback返回值
    PM->>PM: 解除锁定状态
```

### 移除流动性

```mermaid
flowchart TD
    A[用户调用unlock] --> B[PoolManager解锁]
    B --> C[调用unlockCallback]
    C --> D[modifyLiquidity负值]
    D --> E{是否有beforeRemoveLiquidity Hook?}
    
    E -->|是| F[调用beforeRemoveLiquidity]
    E -->|否| G[直接执行移除逻辑]
    F --> G
    
    G --> H[计算移除的流动性]
    H --> I[更新Position状态]
    I --> J[计算累计手续费]
    J --> K[更新tick信息]
    K --> L[更新tick bitmap]
    
    L --> M{是否有afterRemoveLiquidity Hook?}
    M -->|是| N[调用afterRemoveLiquidity]
    M -->|否| O[跳过Hook]
    N --> O
    
    O --> P[计算Delta]
    P --> Q[更新用户Delta余额]
    Q --> R[检查Delta平衡]
    R --> S[解锁结束]
```

## 交换执行流程

```mermaid
sequenceDiagram
    participant User as 用户合约
    participant PM as PoolManager
    participant Hook as Hook合约
    participant Pool as Pool库
    participant SwapMath as SwapMath库
    
    User->>PM: unlock(swap_data)
    PM->>PM: 设置解锁状态
    PM->>User: unlockCallback(swap_data)
    
    User->>PM: swap(key, params, hookData)
    
    Note over PM: 参数验证
    PM->>PM: 检查amountSpecified != 0
    PM->>PM: 检查价格限制
    
    Note over PM: Hook前置处理
    alt Hook存在且有beforeSwap
        PM->>Hook: beforeSwap(sender, key, params, hookData)
        Hook-->>PM: 返回(selector, beforeSwapDelta, newFee)
        PM->>PM: 处理Hook返回的Delta
        PM->>PM: 应用动态手续费
    end
    
    Note over PM: 交换计算循环
    loop 直到交换完成或到达价格限制
        PM->>Pool: 获取下一个tick
        PM->>SwapMath: 计算当前step的交换量
        SwapMath-->>PM: 返回(amountIn, amountOut, fee)
        
        alt 跨越tick
            PM->>Pool: 跨越tick，更新流动性
        end
        
        PM->>Pool: 更新价格和tick
    end
    
    Note over PM: Hook后置处理
    alt Hook存在且有afterSwap
        PM->>Hook: afterSwap(sender, key, params, delta, hookData)
        Hook-->>PM: 返回(selector, hookDelta)
        PM->>PM: 应用Hook Delta
    end
    
    PM->>PM: 计算最终Delta
    PM->>PM: 更新协议手续费
    PM->>PM: 触发Swap事件
    PM-->>User: 返回BalanceDelta
    
    User->>PM: settle/take操作结算代币
    PM->>PM: 检查Delta平衡
    PM-->>User: unlockCallback返回值
    PM->>PM: 解除锁定状态
```

### 交换详细计算流程

```mermaid
flowchart TD
    A[开始交换] --> B[获取当前池状态]
    B --> C[确定交换方向]
    C --> D{剩余金额 > 0?}
    
    D -->|否| Z[交换完成]
    D -->|是| E[获取下一个有流动性的tick]
    
    E --> F[计算到下一个tick的交换量]
    F --> G[使用SwapMath计算实际交换]
    G --> H{是否跨越tick?}
    
    H -->|是| I[更新跨越tick的流动性]
    H -->|否| J[更新当前价格]
    
    I --> K[更新tick的feeGrowth]
    K --> J
    
    J --> L[累计手续费]
    L --> M[更新剩余交换金额]
    M --> N{到达价格限制?}
    
    N -->|是| Z
    N -->|否| D
    
    Z --> O[返回交换结果]
```

## Hook系统执行流程

### Hook权限验证流程

```mermaid
flowchart TD
    A[部署Hook合约] --> B[获取合约地址]
    B --> C[提取地址最低14位]
    C --> D[解析Hook权限位]
    
    D --> E{beforeInitialize位?}
    E -->|1| F[启用beforeInitialize]
    E -->|0| G[跳过beforeInitialize]
    
    F --> H{afterInitialize位?}
    G --> H
    H -->|1| I[启用afterInitialize]
    H -->|0| J[跳过afterInitialize]
    
    I --> K[继续检查其他权限位...]
    J --> K
    K --> L[权限验证完成]
```

### Hook调用流程

```mermaid
sequenceDiagram
    participant PM as PoolManager
    participant Hook as Hook合约
    participant Validation as 权限验证
    
    Note over PM: 操作开始前
    PM->>Validation: 检查Hook地址权限位
    Validation-->>PM: 权限验证结果
    
    alt 有相应权限
        PM->>Hook: 调用before Hook方法
        Hook->>Hook: 执行自定义逻辑
        Hook-->>PM: 返回selector和可选数据
        PM->>PM: 验证返回的selector
        
        alt selector无效
            PM->>PM: 抛出InvalidHookResponse错误
        end
    end
    
    Note over PM: 执行核心操作
    PM->>PM: 执行池操作逻辑
    
    Note over PM: 操作完成后
    alt 有after Hook权限
        PM->>Hook: 调用after Hook方法
        Hook->>Hook: 执行自定义逻辑
        Hook-->>PM: 返回selector和可选Delta
        PM->>PM: 验证返回值
        PM->>PM: 应用Hook Delta
    end
```

## Delta管理流程

```mermaid
sequenceDiagram
    participant User as 用户合约
    participant PM as PoolManager
    participant Delta as Delta管理
    participant Token as ERC20代币
    
    Note over PM: unlock开始
    PM->>Delta: 初始化用户Delta为0
    
    Note over PM: 操作执行期间
    loop 多次操作
        User->>PM: swap/modifyLiquidity/donate
        PM->>PM: 执行操作逻辑
        PM->>Delta: 更新用户Delta
        Delta->>Delta: delta += operationDelta
    end
    
    Note over PM: 结算阶段
    User->>PM: settle(currency, amount)
    PM->>Token: transferFrom(user, poolManager, amount)
    PM->>Delta: delta[currency] -= amount
    
    User->>PM: take(currency, to, amount)
    PM->>Token: transfer(to, amount)
    PM->>Delta: delta[currency] += amount
    
    Note over PM: unlock结束检查
    PM->>Delta: 检查所有currency的delta
    alt 任何delta != 0
        PM->>PM: 抛出CurrencyNotSettled错误
    else 所有delta == 0
        PM->>PM: 成功解锁
    end
```

### Delta计算详细流程

```mermaid
flowchart TD
    A[操作开始] --> B[初始化Delta = 0]
    
    B --> C{操作类型}
    C -->|交换| D[Swap Delta计算]
    C -->|流动性| E[Liquidity Delta计算]
    C -->|捐赠| F[Donate Delta计算]
    
    D --> G[amountSpecified为正：exactInput]
    D --> H[amountSpecified为负：exactOutput]
    G --> I[delta0 = -amountIn, delta1 = amountOut]
    H --> J[delta0 = amountIn, delta1 = -amountOut]
    
    E --> K[liquidityDelta为正：添加流动性]
    E --> L[liquidityDelta为负：移除流动性]
    K --> M[delta0 = -amount0, delta1 = -amount1]
    L --> N[delta0 = amount0, delta1 = amount1]
    
    F --> O[delta0 = -amount0, delta1 = -amount1]
    
    I --> P[累计到总Delta]
    J --> P
    M --> P
    N --> P
    O --> P
    
    P --> Q[返回操作Delta]
```

## 安全检查流程

```mermaid
flowchart TD
    A[操作请求] --> B{合约是否解锁?}
    B -->|否| C[抛出ManagerLocked错误]
    B -->|是| D[继续执行]
    
    D --> E{调用者权限检查}
    E -->|无权限| F[抛出Unauthorized错误]
    E -->|有权限| G[参数验证]
    
    G --> H{参数是否有效?}
    H -->|否| I[抛出相应参数错误]
    H -->|是| J[执行操作]
    
    J --> K[Hook安全检查]
    K --> L{Hook返回值有效?}
    L -->|否| M[抛出InvalidHookResponse错误]
    L -->|是| N[检查重入攻击]
    
    N --> O{是否重入?}
    O -->|是| P[防重入保护触发]
    O -->|否| Q[操作执行完成]
    
    Q --> R[Delta平衡检查]
    R --> S{Delta是否平衡?}
    S -->|否| T[抛出CurrencyNotSettled错误]
    S -->|是| U[操作成功完成]
```

### 重入攻击防护机制

```mermaid
sequenceDiagram
    participant User as 恶意用户
    participant PM as PoolManager
    participant Lock as Lock库
    participant Hook as 恶意Hook
    
    User->>PM: unlock(malicious_data)
    PM->>Lock: 检查当前锁定状态
    Lock-->>PM: 未锁定
    PM->>Lock: 设置锁定状态
    
    PM->>User: unlockCallback(malicious_data)
    User->>PM: 正常操作...
    PM->>Hook: 调用Hook
    
    Note over Hook: 恶意Hook尝试重入
    Hook->>PM: unlock(nested_data)
    PM->>Lock: 检查当前锁定状态
    Lock-->>PM: 已锁定!
    PM->>PM: 抛出AlreadyUnlocked错误
    
    Note over PM: 重入攻击被阻止
```

## 总结

这些流程图展示了Uniswap V4的核心工作机制：

1. **Singleton架构**通过统一的PoolManager管理所有池状态
2. **Hook系统**在操作的关键节点提供扩展能力
3. **Delta管理**确保操作的原子性和一致性
4. **安全机制**防止重入攻击和确保参数有效性

理解这些流程有助于：
- 开发自定义Hook合约
- 集成V4到现有DeFi应用
- 优化Gas使用和操作效率
- 确保合约安全性

建议结合源代码和测试用例来深入理解每个流程的具体实现细节。