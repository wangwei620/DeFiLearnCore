# Uniswap V4 核心流程图

## 🔄 系统整体流程图

```mermaid
graph TB
    A[用户] --> B[Router/RouterProxy]
    B --> C[PoolManager]
    C --> D[Pool Singleton]
    D --> E[Hook System]
    E --> F[Token Transfers]
    
    G[LP Provider] --> H[Position Manager]
    H --> C
    
    I[Protocol Admin] --> J[PoolManager Admin]
    J --> C
    
    style A fill:#e1f5fe
    style C fill:#f3e5f5
    style D fill:#e8f5e8
    style E fill:#fff3e0
```

## 🏊 池子创建流程

```mermaid
sequenceDiagram
    participant U as User
    participant PM as PoolManager
    participant P as Pool
    participant H as Hook
    participant T as Tokens
    
    U->>PM: initialize(tokenA, tokenB, fee, tickSpacing, hook)
    PM->>PM: validate parameters
    PM->>P: create pool instance
    PM->>H: beforeInitialize()
    H-->>PM: hook response
    PM->>P: initialize pool state
    PM->>T: verify token contracts
    PM->>H: afterInitialize()
    H-->>PM: hook response
    PM-->>U: pool created event
```

## 💱 交易执行流程

```mermaid
sequenceDiagram
    participant U as User
    participant R as Router
    participant PM as PoolManager
    participant P as Pool
    participant H as Hook
    participant T as Tokens
    
    U->>R: swap(tokenIn, tokenOut, amount, minAmountOut)
    R->>PM: exactInputSingle(params)
    PM->>PM: validate swap params
    PM->>H: beforeSwap()
    H-->>PM: hook response
    PM->>P: calculate swap amounts
    P->>P: update pool state
    PM->>T: transfer tokens
    T-->>PM: transfer success
    PM->>H: afterSwap()
    H-->>PM: hook response
    PM-->>R: swap result
    R-->>U: tokens received
```

## 💧 流动性管理流程

```mermaid
sequenceDiagram
    participant LP as Liquidity Provider
    participant PM as PoolManager
    participant P as Pool
    participant H as Hook
    participant NP as NFT Position Manager
    
    LP->>PM: modifyPosition(params)
    PM->>PM: validate position params
    PM->>H: beforeModifyPosition()
    H-->>PM: hook response
    PM->>P: calculate liquidity changes
    P->>P: update pool state
    PM->>NP: mint/burn position NFT
    NP-->>PM: position updated
    PM->>H: afterModifyPosition()
    H-->>PM: hook response
    PM-->>LP: position updated event
```

## 🎣 Hook 系统执行流程

```mermaid
graph TD
    A[Pool Operation] --> B{Has Hook?}
    B -->|Yes| C[Before Hook]
    B -->|No| D[Execute Core Logic]
    C --> E[Hook Validation]
    E --> F{Valid?}
    F -->|Yes| G[Execute Core Logic]
    F -->|No| H[Revert]
    G --> I{Has After Hook?}
    I -->|Yes| J[After Hook]
    I -->|No| K[Complete]
    J --> L[Hook Post-Processing]
    L --> K
    
    style A fill:#e3f2fd
    style C fill:#f3e5f5
    style G fill:#e8f5e8
    style J fill:#fff3e0
```

## 🔄 多跳交易流程

```mermaid
graph LR
    A[Token A] --> B[Pool 1]
    B --> C[Token B]
    C --> D[Pool 2]
    D --> E[Token C]
    E --> F[Pool 3]
    F --> G[Token D]
    
    H[Router] --> I[Calculate Path]
    I --> J[Execute Swaps]
    J --> K[Optimize Gas]
    
    style A fill:#e1f5fe
    style G fill:#e8f5e8
    style H fill:#fff3e0
```

## 🎯 价格计算流程

```mermaid
graph TD
    A[Swap Request] --> B[Get Current Tick]
    B --> C[Calculate Price]
    C --> D[Check Slippage]
    D --> E{Within Limits?}
    E -->|Yes| F[Execute Swap]
    E -->|No| G[Revert]
    F --> H[Update Tick]
    H --> I[Calculate New Price]
    I --> J[Emit Event]
    
    style A fill:#e3f2fd
    style F fill:#e8f5e8
    style G fill:#ffebee
```

## 🔐 权限控制流程

```mermaid
graph TD
    A[Function Call] --> B{Check Access Control}
    B -->|Owner| C[Execute Function]
    B -->|Authorized| D[Check Permissions]
    B -->|Unauthorized| E[Revert]
    D --> F{Has Permission?}
    F -->|Yes| C
    F -->|No| E
    C --> G[Function Complete]
    
    style A fill:#e3f2fd
    style C fill:#e8f5e8
    style E fill:#ffebee
```

## 💰 费用计算流程

```mermaid
graph TD
    A[Swap Amount] --> B[Get Fee Tier]
    B --> C[Calculate Base Fee]
    C --> D{Has Dynamic Fee Hook?}
    D -->|Yes| E[Call Fee Hook]
    D -->|No| F[Use Base Fee]
    E --> G[Get Dynamic Fee]
    G --> H[Calculate Total Fee]
    F --> H
    H --> I[Subtract Fee from Amount]
    I --> J[Execute Swap]
    
    style A fill:#e3f2fd
    style E fill:#f3e5f5
    style J fill:#e8f5e8
```

## 🔄 状态更新流程

```mermaid
sequenceDiagram
    participant P as Pool
    participant S as State
    participant E as Events
    participant H as Hook
    
    P->>S: update liquidity
    S->>S: recalculate ticks
    S->>S: update price
    P->>E: emit Swap event
    P->>H: notify state change
    H->>H: process state update
    H-->>P: hook response
```

## 📊 关键数据流

### 池子状态数据结构
```
Pool State:
├── token0, token1
├── fee, tickSpacing
├── liquidity
├── sqrtPriceX96
├── tick
├── protocolFees
└── hook fees
```

### 交易参数结构
```
Swap Params:
├── tokenIn, tokenOut
├── fee
├── recipient
├── deadline
├── amountIn
├── amountOutMinimum
└── sqrtPriceLimitX96
```

### Hook 回调参数
```
Hook Callback:
├── sender
├── recipient
├── token0, token1
├── fee
├── amount0, amount1
├── sqrtPriceX96
└── liquidity
```