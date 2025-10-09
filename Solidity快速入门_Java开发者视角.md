# Solidity快速入门 - Java开发者视角

## 🔄 Java vs Solidity 核心对比

### 1. 基本语法对比

#### 变量声明
```java
// Java
String name = "Alice";
int age = 25;
HashMap<String, Integer> balances = new HashMap<>();
```

```solidity
// Solidity
string name = "Alice";
uint age = 25;
mapping(address => uint) balances;
```

#### 函数定义
```java
// Java
public int add(int a, int b) {
    return a + b;
}

public synchronized void withdraw(int amount) {
    // 线程安全操作
}
```

```solidity
// Solidity
function add(uint a, uint b) public pure returns (uint) {
    return a + b;
}

function withdraw(uint amount) public {
    // 使用ReentrancyGuard防重入
}
```

#### 继承
```java
// Java
public class ERC20Token extends BaseToken implements IERC20 {
    @Override
    public boolean transfer(address to, uint amount) {
        // 实现
    }
}
```

```solidity
// Solidity
contract ERC20Token is BaseToken, IERC20 {
    function transfer(address to, uint amount) public override returns (bool) {
        // 实现
    }
}
```

---

## 🆕 Solidity独特概念（Java没有的）

### 1. 函数可见性修饰符
```solidity
contract Example {
    // public: 内外部都可调用（类似Java public）
    function publicFunc() public {}
    
    // external: 仅外部调用（Java无对应，节省Gas）
    function externalFunc() external {}
    
    // internal: 仅合约内部和继承合约（类似Java protected）
    function internalFunc() internal {}
    
    // private: 仅当前合约（类似Java private）
    function privateFunc() private {}
}
```

### 2. 状态可变性修饰符
```solidity
contract Example {
    uint public count = 0;
    
    // view: 只读，不修改状态（类似Java getter，但编译器强制）
    function getCount() public view returns (uint) {
        return count;
    }
    
    // pure: 纯函数，不读不写状态（类似Java静态工具方法）
    function add(uint a, uint b) public pure returns (uint) {
        return a + b;
    }
    
    // payable: 可接收ETH（Java无对应）
    function deposit() public payable {
        // msg.value 是发送的ETH数量
    }
}
```

### 3. Mapping（特殊的HashMap）
```solidity
contract MappingExample {
    // 不可遍历！没有keys()方法
    mapping(address => uint) public balances;
    
    // 嵌套mapping
    mapping(address => mapping(address => uint)) public allowances;
    
    function setBalance(address user, uint amount) public {
        balances[user] = amount;
    }
    
    // ⚠️ 无法获取所有keys，需要额外数组存储
    address[] public userList;
    
    function addUser(address user) public {
        if (balances[user] == 0) {
            userList.push(user);
        }
        balances[user] = 100;
    }
}
```

### 4. 事件（Events）- 链下监听必备
```solidity
contract EventExample {
    // 类似Java的观察者模式，但存储在链上日志
    event Transfer(address indexed from, address indexed to, uint amount);
    
    function transfer(address to, uint amount) public {
        // 业务逻辑...
        
        // 发出事件（Gas费用很低）
        emit Transfer(msg.sender, to, amount);
    }
}
```

**Java对比**：
```java
// Java需要自己实现观察者
public interface TransferListener {
    void onTransfer(Address from, Address to, int amount);
}

private List<TransferListener> listeners = new ArrayList<>();

public void transfer(Address to, int amount) {
    // 业务逻辑...
    
    // 手动通知监听者
    for (TransferListener listener : listeners) {
        listener.onTransfer(this.address, to, amount);
    }
}
```

---

## 🔒 安全模式（Critical！）

### 1. 检查-生效-交互（Checks-Effects-Interactions）
```solidity
contract BankV1 {
    mapping(address => uint) public balances;
    
    // ❌ 错误：重入攻击漏洞
    function withdrawBad() public {
        uint amount = balances[msg.sender];
        
        // 先转账（交互）
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success);
        
        // 后更新状态（生效）- 攻击者可重入！
        balances[msg.sender] = 0;
    }
    
    // ✅ 正确：先更新状态
    function withdrawGood() public {
        uint amount = balances[msg.sender];
        
        // 1. 检查
        require(amount > 0, "No balance");
        
        // 2. 生效（先改状态！）
        balances[msg.sender] = 0;
        
        // 3. 交互（最后转账）
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success);
    }
}
```

**Java对比**：
```java
// Java的synchronized可防止多线程重入
public synchronized void withdraw() {
    int amount = balances.get(user);
    
    // 即使顺序错误，synchronized也能保护
    transfer(user, amount);
    balances.put(user, 0);
}

// 但Solidity中，外部调用可回调当前合约（类似递归），synchronized无效！
```

### 2. 使用OpenZeppelin库
```solidity
// ✅ 推荐：使用经过审计的库
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract SafeBank is ReentrancyGuard {
    mapping(address => uint) public balances;
    
    // nonReentrant 自动防止重入
    function withdraw() public nonReentrant {
        uint amount = balances[msg.sender];
        require(amount > 0);
        
        balances[msg.sender] = 0;
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success);
    }
}
```

---

## 💡 实战代码示例

### 示例1：ERC20代币（完整版）
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MyToken is ERC20, Ownable {
    // Java开发者注意：没有构造函数重载！
    constructor() ERC20("MyToken", "MTK") Ownable(msg.sender) {
        // 铸造100万代币给部署者
        _mint(msg.sender, 1000000 * 10 ** decimals());
    }
    
    // 只有owner可以铸造（类似Java的@RolesAllowed）
    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }
    
    // 销毁代币
    function burn(uint256 amount) public {
        _burn(msg.sender, amount);
    }
}
```

**对应的Java伪代码**：
```java
public class MyToken extends ERC20 {
    private final Owner owner;
    
    public MyToken(Owner owner) {
        super("MyToken", "MTK");
        this.owner = owner;
        this.mint(owner.getAddress(), 1000000);
    }
    
    @RequiresOwner
    public void mint(Address to, int amount) {
        // 实现
    }
}
```

### 示例2：简单NFT市场
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract NFTMarketplace is ReentrancyGuard {
    struct Listing {
        address seller;
        uint price;
        bool active;
    }
    
    // NFT合约地址 => tokenId => 挂单信息
    mapping(address => mapping(uint => Listing)) public listings;
    
    event Listed(address indexed nftContract, uint indexed tokenId, uint price);
    event Sold(address indexed nftContract, uint indexed tokenId, address buyer);
    
    // 挂单NFT
    function list(address nftContract, uint tokenId, uint price) external {
        IERC721 nft = IERC721(nftContract);
        
        // 检查：调用者是否拥有NFT
        require(nft.ownerOf(tokenId) == msg.sender, "Not owner");
        // 检查：是否已授权本合约
        require(
            nft.getApproved(tokenId) == address(this) ||
            nft.isApprovedForAll(msg.sender, address(this)),
            "Not approved"
        );
        
        // 生效：创建挂单
        listings[nftContract][tokenId] = Listing({
            seller: msg.sender,
            price: price,
            active: true
        });
        
        emit Listed(nftContract, tokenId, price);
    }
    
    // 购买NFT
    function buy(address nftContract, uint tokenId) 
        external 
        payable 
        nonReentrant // 防重入
    {
        Listing memory listing = listings[nftContract][tokenId];
        
        // 检查
        require(listing.active, "Not listed");
        require(msg.value >= listing.price, "Insufficient payment");
        
        // 生效：先改状态
        listings[nftContract][tokenId].active = false;
        
        // 交互：转账和转移NFT
        IERC721(nftContract).safeTransferFrom(
            listing.seller,
            msg.sender,
            tokenId
        );
        
        // 支付卖家（使用call而非transfer）
        (bool success, ) = listing.seller.call{value: listing.price}("");
        require(success, "Transfer failed");
        
        // 退回多余的钱
        if (msg.value > listing.price) {
            (bool refundSuccess, ) = msg.sender.call{
                value: msg.value - listing.price
            }("");
            require(refundSuccess, "Refund failed");
        }
        
        emit Sold(nftContract, tokenId, msg.sender);
    }
}
```

---

## ⚠️ 常见陷阱（Java开发者易犯）

### 1. ❌ 整数除法截断
```solidity
// Java
int result = 5 / 2;  // 2（整数除法）
double result = 5.0 / 2.0;  // 2.5

// Solidity（没有浮点数！）
uint result = 5 / 2;  // 2（永远是整数）

// ✅ 解决：使用大单位
uint result = (5 * 1e18) / 2;  // 2.5 * 1e18（表示2.5，精度18位）
```

### 2. ❌ Mapping默认值
```solidity
mapping(address => uint) public balances;

// Java HashMap
if (balances.get(user) == null) {  // 未设置返回null
    balances.put(user, 0);
}

// Solidity mapping
if (balances[user] == 0) {  // 未设置返回0（不是null！）
    // 无法区分"未设置"和"设置为0"
}

// ✅ 解决：使用额外的bool标记
mapping(address => uint) public balances;
mapping(address => bool) public hasBalance;

function setBalance(address user, uint amount) public {
    balances[user] = amount;
    hasBalance[user] = true;
}
```

### 3. ❌ 数组越界
```solidity
uint[] public items;

// Java
items.get(10);  // 抛出IndexOutOfBoundsException

// Solidity
items[10];  // 直接revert整个交易，Gas不退！

// ✅ 检查长度
require(index < items.length, "Index out of bounds");
uint item = items[index];
```

### 4. ❌ 字符串比较
```solidity
// Java
if (str1.equals(str2)) { }

// Solidity（没有直接比较！）
if (str1 == str2) { }  // ❌ 编译错误

// ✅ 正确方法：比较哈希
if (keccak256(abi.encodePacked(str1)) == keccak256(abi.encodePacked(str2))) {
    // 相等
}
```

---

## 🚀 开发环境设置（5分钟快速上手）

### 1. 安装Node.js和Hardhat
```bash
# 安装Node.js 18+
node -v

# 创建项目
mkdir my-defi-project
cd my-defi-project
npm init -y

# 安装Hardhat
npm install --save-dev hardhat
npx hardhat init  # 选择 "Create a JavaScript project"

# 安装OpenZeppelin
npm install @openzeppelin/contracts
```

### 2. 编写第一个合约
```solidity
// contracts/MyFirstToken.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MyFirstToken is ERC20 {
    constructor() ERC20("MyFirstToken", "MFT") {
        _mint(msg.sender, 1000000 * 10 ** decimals());
    }
}
```

### 3. 编写测试（类似JUnit）
```javascript
// test/MyFirstToken.test.js
const { expect } = require("chai");

describe("MyFirstToken", function () {
  it("Should mint initial supply to deployer", async function () {
    const [owner] = await ethers.getSigners();
    
    const Token = await ethers.getContractFactory("MyFirstToken");
    const token = await Token.deploy();
    
    const balance = await token.balanceOf(owner.address);
    expect(balance).to.equal(ethers.parseUnits("1000000", 18));
  });
});
```

### 4. 运行测试
```bash
npx hardhat test
```

---

## 📊 Gas优化技巧（类似Java性能优化）

### 1. 使用`calldata`代替`memory`
```solidity
// ❌ 贵：memory需要复制
function process(uint[] memory data) external {
    // Gas: ~50,000
}

// ✅ 便宜：calldata直接读取
function process(uint[] calldata data) external {
    // Gas: ~30,000
}
```

### 2. 打包变量
```solidity
// ❌ 贵：每个变量占一个存储槽（32字节）
contract Unpacked {
    uint8 a;   // 槽0
    uint256 b; // 槽1
    uint8 c;   // 槽2
}

// ✅ 便宜：紧密打包，节省2个槽
contract Packed {
    uint8 a;    // 槽0
    uint8 c;    // 槽0（和a共享）
    uint256 b;  // 槽1
}
```

### 3. 使用不可变变量
```solidity
contract Example {
    // ❌ 贵：每次读取消耗2100 Gas
    address public owner;
    
    // ✅ 便宜：编译时内联，免费读取
    address public immutable owner;
    
    constructor() {
        owner = msg.sender;
    }
}
```

---

## 🎯 学习路径总结

### 第1周：基础语法
- [ ] 完成 CryptoZombies 课程1-6
- [ ] 手写一个简单的ERC20代币
- [ ] 理解 view/pure/payable 区别

### 第2周：进阶特性
- [ ] 学习继承和接口
- [ ] 使用OpenZeppelin库
- [ ] 编写NFT合约（ERC721）

### 第3周：安全实践
- [ ] 完成Ethernaut前5关
- [ ] 理解重入攻击原理
- [ ] 使用Slither静态分析工具

### 第4周：实战项目
- [ ] 完成一个简单的DeFi项目（质押、借贷等）
- [ ] 编写完整测试用例
- [ ] 部署到测试网（Sepolia）

---

## 📚 推荐资源

1. **官方文档**：https://docs.soliditylang.org/
2. **Solidity by Example**：https://solidity-by-example.org/
3. **OpenZeppelin文档**：https://docs.openzeppelin.com/
4. **Ethernaut安全挑战**：https://ethernaut.openzeppelin.com/

---

**下一步**：阅读工作区中的 `Uniswap_V4_代码实例与最佳实践.md` 和 `Aave_V3_技术深度分析.md`，学习真实世界的DeFi协议！
