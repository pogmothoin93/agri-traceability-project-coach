# 农产品溯源合约讲义

> 目标：在已经跑通的 FISCO BCOS `asset-app`（Java 项目）里，新增一个**农产品溯源合约 `AgriTrace.sol`**，并写一个 **Java 客户端 `AgriTraceClient.java`** 负责部署和调用，最后用 `agri_run.sh` 一键跑通。
>
> 本讲义重点讲透两个文件：**合约 `AgriTrace.sol`** 和 **客户端 `AgriTraceClient.java`**。其余步骤（编译、构建、脚本）作为支撑，讲清楚它们之间怎么串起来。

---

## 0. 先看全局：一张图理解它怎么跑起来

```
    ┌─────────────────────┐        你写的两个核心文件
    │  AgriTrace.sol      │  ← 智能合约（区块链上的"数据库 + 业务逻辑"）
    │  (Solidity)         │     定义数据结构 + register/addTrace/查询
    └──────────┬──────────┘
               │ sol2java.sh 编译
               ▼
    ┌─────────────────────┐
    │  AgriTrace.java     │  ← 自动生成的 Java 代理类（不用手写）
    │  (生成的)            │     deploy / load / 每个函数对应一个方法
    └──────────┬──────────┘
               │ 被调用
               ▼
    ┌─────────────────────┐
    │  AgriTraceClient.java│ ← 你写的客户端：连链、部署、发交易、查结果
    │  (Java)             │
    └──────────┬──────────┘
               │ agri_run.sh 启动
               ▼
    ┌─────────────────────┐
    │  FISCO BCOS 节点     │  ← 真正存数据、执行合约的地方
    │  (group1, 20200...)  │
    └─────────────────────┘
```

一句话理解三层分工：

| 层 | 文件 | 职责 |
|----|------|------|
| 链上逻辑层 | `AgriTrace.sol` | 数据长什么样、谁能改、怎么查（不可篡改） |
| 中间代理层 | `AgriTrace.java`（生成） | 把 Solidity 函数翻译成 Java 方法，处理 ABI 编解码 |
| 应用调用层 | `AgriTraceClient.java` | 你的业务代码：连链、调方法、打印结果 |

**为什么要分三层？** 因为 Java 不能直接跟区块链说 Solidity，中间必须有个"翻译官"（就是生成的 `AgriTrace.java`）。你要写的其实只有第一层和第三层，中间层交给工具。

---

## 1. 业务设计：一个"批次"就是一件农产品

先想清楚数据，再写代码。这是所有合约的第一步。

- **一批农产品**（比如一批赣南脐橙）= 一个唯一批次号 `batchId`。
- 登记后产品基本信息**上链不可改**。
- 之后每经过一个环节（种植、加工、运输、销售…）就**追加一条溯源记录**。
- 一个批次可以有很多条溯源记录 → 这是"一对多"关系。

| 实体 | 字段 |
|------|------|
| 产品 Product | `batchId` 批次号、`name` 产品名、`origin` 产地、`producer` 生产者 |
| 溯源记录 Trace | `batchId` 批次号、`operation` 环节、`operatorName` 操作人、`info` 说明 |

对外接口共 5 个：

| 接口 | 类型 | 作用 |
|------|------|------|
| `register(batchId, name, origin, producer)` | 写 | 登记产品 |
| `addTrace(batchId, operation, operatorName, info)` | 写 | 追加一条溯源环节 |
| `getProduct(batchId)` | 读 | 查产品信息 |
| `getTrace(batchId, index)` | 读 | 查第 index 条溯源记录 |
| `getTraceCount(batchId)` | 读 | 查该批次共有几条溯源记录 |

---

## 2. 核心文件一：`AgriTrace.sol` 逐段精讲

### 2.1 完整源码

```solidity
pragma solidity ^0.4.25;
pragma experimental ABIEncoderV2;

contract AgriTrace {

    // 产品信息
    struct Product {
        string batchId;      // 批次号，唯一标识
        string name;         // 产品名称
        string origin;       // 产地
        string producer;     // 生产者/企业
        bool   exist;        // 是否已登记（用来判断存在性）
    }

    // 溯源环节
    struct Trace {
        string batchId;      // 所属批次
        string operation;    // 操作环节，如 种植/加工/运输/销售
        string operatorName; // 操作方名称
        string info;         // 附加说明
        bool   exist;
    }

    mapping(string => Product) private products;   // 批次号 => 产品
    mapping(string => Trace[]) private traces;     // 批次号 => 溯源记录列表

    // ……（函数部分下面逐个讲）
}
```

### 2.2 为什么用 `struct + mapping`，而不是 Asset 里的 Table？

你之前跑通的 `Asset.sol` 用的是**预编译表**（`import "./Table.sol";` + `t_asset` 表）。那是另一种存储方式。**本合约故意不用它**，原因有三：

1. **更直观**：`struct` 就是"一条记录"，`mapping` 就是"字典"，和普通编程语言一致，好理解。
2. **支持一对多**：`mapping(string => Trace[])` 天然就是一个批次挂一个数组，追加记录只需 `push`。用 Table 实现一对多很别扭。
3. **减少依赖**：不依赖 `Table.sol`，合约更纯粹，换环境也好移植。

> 所以：**这份合约开头不要写 `import "./Table.sol";`**，写了也用不上。

### 2.3 `pragma` 两行分别是什么

```solidity
pragma solidity ^0.4.25;              // 编译器版本：0.4.25 及以上、0.5.0 以下
pragma experimental ABIEncoderV2;     // 启用高级 ABI 编码（struct/mapping 相关）
```

- 第一行必须和 console 里的 solc 版本匹配。用 `grep pragma ~/fisco/console/contracts/solidity/Asset.sol` 确认。**若项目是 0.6.x，`uintToStr` 里的 `byte(...)` 要改成 `bytes1(...)`**（见 2.8）。
- 第二行：如果 `sol2java.sh` 报 `ABIEncoderV2` 相关的错，直接删掉这行再编译。

### 2.4 `register`：登记产品

```solidity
function register(
    string _batchId,
    string _name,
    string _origin,
    string _producer
) public returns (int256) {
    Product storage p = products[_batchId];
    require(!p.exist, "batch already registered");   // ① 防重复登记
    p.batchId  = _batchId;                            // ② 逐字段赋值
    p.name     = _name;
    p.origin   = _origin;
    p.producer = _producer;
    p.exist    = true;                                // ③ 打上"已存在"标记
    return 0;
}
```

**逐行说：**

- `Product storage p = products[_batchId];` — `storage` 表示直接指向链上存储的引用，改 `p` 就是改链上数据。如果用 `memory` 就只是副本，改了不落链。
- `require(!p.exist, ...)` — 如果这个批次已经登记过，直接报错回滚。这就是产品唯一性的保证。
- `return 0` — 返回 0 表示成功（老式惯例，Java 端主要靠交易状态判断，这个返回值一般不重点用）。

> **坑**：`string` 作为 `mapping` 的 key 是允许的（Solidity 会把字符串哈希后当键）。`products[_batchId]` 不需要先"创建"，访问即初始化一个空 `Product`。

### 2.5 `addTrace`：追加溯源环节

```solidity
function addTrace(
    string _batchId,
    string _operation,
    string _operatorName,
    string _info
) public returns (int256) {
    Product storage p = products[_batchId];
    require(p.exist, "batch not registered");    // ① 必须先登记过产品

    Trace memory t;                               // ② 在内存里拼一条记录
    t.batchId      = _batchId;
    t.operation    = _operation;
    t.operatorName = _operatorName;
    t.info         = _info;
    t.exist        = true;
    traces[_batchId].push(t);                     // ③ 追加到该批次的数组末尾
    return 0;
}
```

**逐行说：**

- `require(p.exist, ...)` — **顺序很关键**：必须 `register` 在前、`addTrace` 在后，否则这里直接拦下。这就是你运行时如果先 addtrace 会失败的原因。
- `Trace memory t;` — 这里用 `memory` 是**故意的**：先在内存里把一条记录拼好，再整体 `push` 进去，比直接操作 `traces[_batchId].push()` 里逐字段写更清晰。
- `traces[_batchId].push(t)` — 把记录追加到"该批次号对应的数组"。数组自动扩容，`getTraceCount` 拿到的就是它的 `length`。

> **坑**：某些 solc 版本对 `push(struct)` 会报 "copying of type struct to storage not supported"。若遇到，改成先 `traces[_batchId].length++` 再逐字段赋值，或者直接 `traces[_batchId].push(Trace(_batchId,_operation,_operatorName,_info,true))`。

### 2.6 `getProduct` / `getTrace`：查询为什么返回拼接字符串？

```solidity
function getProduct(string _batchId) public view returns (string) {
    Product storage p = products[_batchId];
    if (!p.exist) {
        return "product not found";
    }
    return string(abi.encodePacked(
        "batchId=", p.batchId,
        "; name=", p.name,
        "; origin=", p.origin,
        "; producer=", p.producer
    ));
}
```

```solidity
function getTrace(string _batchId, uint256 _index) public view returns (string) {
    Product storage p = products[_batchId];
    if (!p.exist) {
        return "product not found";
    }
    if (_index >= traces[_batchId].length) {
        return "trace not found";
    }
    Trace storage t = traces[_batchId][_index];
    return string(abi.encodePacked(
        "#", uintToStr(_index),
        " operation=", t.operation,
        "; operator=", t.operatorName,
        "; info=", t.info
    ));
}
```

**这是本合约最关键的设计取舍**，一定要理解为什么：

> **如果让 Solidity 返回一个 `struct` 或多个字段**，Java 端拿到的是 `TupleN<...>` 这种元组对象，还要一个个 `.getValue1()/.getValue2()` 去取，而且 `uint` 还得自己转字符串，非常啰嗦。

> **所以这里直接在合约里用 `abi.encodePacked` 把所有字段拼成一段可读文本**，Java 端拿到的就是一个 `String`，`System.out.println` 直接能看。**用一点链上的拼接开销，换客户端的大幅简化**——对教学和演示来说非常划算。

**逐行说：**

- `view` 关键字 = 只读，不写链、不发交易、不花 gas。所以 Java 端调用它**不需要交易回执**，直接拿到返回值（对应第 3 节 client 里的写法）。
- `abi.encodePacked(...)` = 把若干个参数紧凑地拼成一个 `bytes`，再用 `string(...)` 转成字符串。
- `_index >= traces[_batchId].length` — 越界检查，防数组越界。
- 注意 `uintToStr(_index)` —— 序号是 `uint`，必须自己转成字符串才能拼（见 2.8）。

### 2.7 `getTraceCount`：查记录条数

```solidity
function getTraceCount(string _batchId) public view returns (uint256) {
    return traces[_batchId].length;
}
```

简单直接：返回数组长度。Java 端拿到的是 `BigInteger`。

### 2.8 `uintToStr`：uint 转字符串的小工具

```solidity
function uintToStr(uint256 v) internal pure returns (string) {
    if (v == 0) return "0";
    uint256 temp = v;
    uint256 digits;
    while (temp != 0) { digits++; temp /= 10; }     // ① 先算有几位
    bytes memory buffer = new bytes(digits);        // ② 开对应长度的缓冲区
    while (v != 0) {
        digits -= 1;
        buffer[digits] = byte(48 + uint8(v % 10));  // ③ 从低位往高位填 ASCII
        v /= 10;
    }
    return string(buffer);
}
```

**为什么要它？** Solidity 0.4 没有内置的 `uint → string`，要自己写。`48` 是字符 `'0'` 的 ASCII 码，`48 + 数字` 就得到对应字符。

**逐行说：**

- `internal pure` — `internal` 只在合约内部调用；`pure` 表示不读也不写链上状态（纯函数）。
- `byte(48 + ...)` — **这就是版本坑**：在 0.6.x 里 `byte` 已废弃，要改成 `bytes1(48 + ...)`。所以版本一定要先确认。

### 2.9 合约部分小结

| 设计点 | 选择 | 原因 |
|--------|------|------|
| 存储 | `struct + mapping` | 直观、支持一对多、不依赖 Table |
| 唯一性 | `bool exist` 标记 | register 防重复、addTrace 防未登记 |
| 查询返回 | 拼接字符串 | 让 Java 端拿 `String`，避免元组处理 |
| uint 处理 | 自带 `uintToStr` | 老版本无内置转换 |
| 版本 | `^0.4.25` | 必须与 console solc 一致 |

---

## 3. 从 `.sol` 到 `AgriTrace.java`：中间发生了什么

这一步不用你写代码，但要知道它做了什么，否则后面报错看不懂。

```bash
cd ~/fisco/console
bash sol2java.sh org.fisco.bcos.asset.contract
```

它会读你的 `AgriTrace.sol`，生成三样东西：

```
contracts/sdk/abi/AgriTrace.abi     # 接口描述（函数有哪些、参数类型）
contracts/sdk/bin/AgriTrace.bin     # 字节码（部署上链用）
contracts/sdk/java/.../AgriTrace.java  # Java 代理类（重点）
```

生成的 `AgriTrace.java` 里有：

- `AgriTrace.deploy(client, keyPair)` → 部署合约，返回合约对象
- `AgriTrace.load(address, client, keyPair)` → 加载已部署的合约
- `register(...)` / `addTrace(...)` → 返回 `TransactionReceipt`（写操作）
- `getProduct(...)` / `getTrace(...)` / `getTraceCount(...)` → **直接返回解码后的值**（读操作）

把它拷进项目：

```bash
cp ~/fisco/console/contracts/sdk/java/org/fisco/bcos/asset/contract/AgriTrace.java \
   ~/fisco/asset-app/src/main/java/org/fisco/bcos/asset/contract/AgriTrace.java
```

> **唯一需要核对的地方**：生成后 `grep -n "public" ~/fisco/asset-app/src/main/java/org/fisco/bcos/asset/contract/AgriTrace.java`，确认三个查询方法的返回类型是 `String`/`BigInteger`。若被包成 `Tuple1<...>`，客户端取值要跟着改。以生成文件为准。

---

## 4. 核心文件二：`AgriTraceClient.java` 逐段精讲

这个文件是你写的"应用层"。理解它之前，先记住一个**贯穿全文的规律**（这是你项目 java-sdk 2.7 的真相，也是之前踩过的坑）：

> ## 写操作 vs 读操作，两种完全不同的写法
>
> | 类型 | Solidity 端 | Java 端返回 | 怎么判断成功 / 怎么取值 |
> |------|------------|-------------|------------------------|
> | **写**（发交易） | `register` / `addTrace` | `TransactionReceipt` | `receipt.isStatusOK()` |
> | **读**（只查询） | `getProduct` / `getTrace` / `getTraceCount` | **直接是 `String` / `BigInteger`** | 直接用，不判断状态 |
>
> ⚠️ **不要用 `TransactionResponse` / `getReturnObject()` / `getReturnCode()`**——那是别的 SDK 版本的 API，你这套 java-sdk 2.7 **根本没有**（grep `AssetClient.java` 一个都没命中）。用了必编译失败。

### 4.1 完整源码

```java
package org.fisco.bcos.asset.client;

import java.io.FileOutputStream;
import java.math.BigInteger;
import java.util.Properties;

import org.fisco.bcos.asset.contract.AgriTrace;
import org.fisco.bcos.sdk.BcosSDK;
import org.fisco.bcos.sdk.client.Client;
import org.fisco.bcos.sdk.crypto.keypair.CryptoKeyPair;
import org.fisco.bcos.sdk.model.TransactionReceipt;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import org.springframework.core.io.ClassPathResource;
import org.springframework.core.io.Resource;

public class AgriTraceClient {
  static Logger logger = LoggerFactory.getLogger(AgriTraceClient.class);

  // 与 Asset 的 address 键分开，避免互相覆盖
  private static final String ADDR_KEY = "agriTraceAddress";

  private BcosSDK bcosSDK;
  private Client client;
  private CryptoKeyPair cryptoKeyPair;

  public void initialize() throws Exception {
    @SuppressWarnings("resource")
    ApplicationContext context =
        new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
    bcosSDK = context.getBean(BcosSDK.class);
    client = bcosSDK.getClient(1);
    cryptoKeyPair = client.getCryptoSuite().createKeyPair();
    client.getCryptoSuite().setCryptoKeyPair(cryptoKeyPair);
    logger.debug("create client for group1, account address is " + cryptoKeyPair.getAddress());
  }
  // ……（其余方法下面逐个讲）
}
```

### 4.2 `initialize()`：连上区块链

```java
ApplicationContext context =
    new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
bcosSDK = context.getBean(BcosSDK.class);
client = bcosSDK.getClient(1);                       // 群组1
cryptoKeyPair = client.getCryptoSuite().createKeyPair();      // 新建一对密钥（=一个账户）
client.getCryptoSuite().setCryptoKeyPair(cryptoKeyPair);      // 设为默认签名账户
```

**逐行说：**

- `ClassPathXmlApplicationContext("classpath:applicationContext.xml")` — 读取 `conf/` 里的配置文件。**这个文件定义了节点 IP、端口、证书路径、群组**，你项目里已经有了，别改它（尤其别删 `dist/conf/conf/` 里的证书）。
- `bcosSDK.getClient(1)` — 取**群组 1** 的客户端。FISCO 是多群组结构，这里的 `1` 就是群组号。
- `createKeyPair()` — 每次运行**随机生成一个新账户**（有私钥地址）。发交易用它签名。教学场景够用；生产环境应加载固定的私钥。
- `setCryptoKeyPair(...)` — 把它设为该 suite 的默认签名账户，否则发交易会没有签名者。

### 4.3 `deploy()`：部署合约上链

```java
public void deploy() {
  try {
    AgriTrace c = AgriTrace.deploy(client, cryptoKeyPair);
    System.out.println(" deploy AgriTrace success, contract address is " + c.getContractAddress());
    recordAddr(c.getContractAddress());     // 把地址存进 contract.properties
  } catch (Exception e) {
    System.out.println(" deploy AgriTrace failed, error message is " + e.getMessage());
  }
}
```

**逐行说：**

- `AgriTrace.deploy(client, cryptoKeyPair)` — 用前面生成的 `AgriTrace.java` 里的静态方法，把 `.bin` 字节码真正发上链，得到一个合约实例。
- `c.getContractAddress()` — 合约部署后有一个**唯一地址**（形如 `0xdfd0...`）。以后所有调用都靠这个地址找到合约。
- `recordAddr(...)` — **必须存下来**！否则下一次运行（比如 register）就不知道合约在哪了。这就是为什么需要 `contract.properties`。

> **坑**：`gradle build` 会用模板里的**空 `contract.properties` 覆盖 `dist/conf/`**，所以每次重新 build 之后，之前 deploy 的地址都会没，必须重新 deploy。这是模板固有行为，记住即可。

### 4.4 `register` / `addTrace`：写操作（返回 `TransactionReceipt`）

```java
public void register(String batchId, String name, String origin, String producer) {
  try {
    AgriTrace c = AgriTrace.load(loadAddr(), client, cryptoKeyPair);  // ① 按地址加载合约
    TransactionReceipt receipt = c.register(batchId, name, origin, producer);  // ② 发交易
    check(receipt, "register");                                          // ③ 判断结果
    System.out.printf(" register success => batchId: %s, name: %s \n", batchId, name);
  } catch (Exception e) {
    logger.error("register exception, error message is {}", e.getMessage());
    System.out.println(" register failed, error message is " + e.getMessage());
  }
}
```

```java
public void addTrace(String batchId, String operation, String operator, String info) {
  try {
    AgriTrace c = AgriTrace.load(loadAddr(), client, cryptoKeyPair);
    TransactionReceipt receipt = c.addTrace(batchId, operation, operator, info);
    check(receipt, "addTrace");
    System.out.printf(" addTrace success => batchId: %s, operation: %s \n", batchId, operation);
  } catch (Exception e) {
    logger.error("addTrace exception, error message is {}", e.getMessage());
    System.out.println(" addTrace failed, error message is " + e.getMessage());
  }
}
```

**逐段说：**

- **① `AgriTrace.load(loadAddr(), ...)`** — 从 `contract.properties` 读出合约地址，加载已经部署的合约。**所有非 deploy 操作都得先 load**，因为合约早就部署好了，只是要"连到"它。
- **② `TransactionReceipt receipt = c.register(...)`** — 写操作会**真的发一笔交易上链**，返回的是交易回执 `TransactionReceipt`。
- **③ `check(receipt, ...)`** — 用回执判断这笔交易到底成没成（见 4.7）。

> **为什么写操作不直接返回结果？** 因为发交易是异步的，交易要先打包、执行，结果体现在"回执"里。读操作（view）不发交易，所以能直接拿返回值。

### 4.5 `product` / `trace` / `count`：读操作（直接返回）

```java
public void product(String batchId) {
  try {
    AgriTrace c = AgriTrace.load(loadAddr(), client, cryptoKeyPair);
    String r = c.getProduct(batchId);            // 直接是 String！
    System.out.println(" product => " + r);
  } catch (Exception e) {
    logger.error("getProduct exception, error message is {}", e.getMessage());
    System.out.println(" query product failed, error message is " + e.getMessage());
  }
}

public void trace(String batchId, int index) {
  try {
    AgriTrace c = AgriTrace.load(loadAddr(), client, cryptoKeyPair);
    String r = c.getTrace(batchId, BigInteger.valueOf(index));   // 直接是 String！
    System.out.println(" trace => " + r);
  } catch (Exception e) {
    logger.error("getTrace exception, error message is {}", e.getMessage());
    System.out.println(" query trace failed, error message is " + e.getMessage());
  }
}

public void count(String batchId) {
  try {
    AgriTrace c = AgriTrace.load(loadAddr(), client, cryptoKeyPair);
    BigInteger n = c.getTraceCount(batchId);     // 直接是 BigInteger！
    System.out.println(" trace count => " + n);
  } catch (Exception e) {
    logger.error("getTraceCount exception, error message is {}", e.getMessage());
    System.out.println(" query count failed, error message is " + e.getMessage());
  }
}
```

**这是整个客户端里最容易写错的地方，划重点：**

| Solidity 函数 | 生成 Java 方法返回 | 你该怎么拿 |
|--------------|-------------------|-----------|
| `getProduct(string)` → `returns(string)` | `String` | `String r = c.getProduct(batchId);` |
| `getTrace(string,uint256)` → `returns(string)` | `String` | `String r = c.getTrace(batchId, BigInteger.valueOf(index));` |
| `getTraceCount(string)` → `returns(uint256)` | `BigInteger` | `BigInteger n = c.getTraceCount(batchId);` |

- **没有 `TransactionReceipt`，没有 `getReturnObject()`**。因为这是 `view` 函数，只读、不发交易，生成的方法直接把解码后的值返回给你。
- `uint256` 参数 → Java 用 `BigInteger`，所以 `index` 用 `BigInteger.valueOf(index)` 转一下。
- 返回值的具体类型**最终以生成的 `AgriTrace.java` 为准**（第 3 节已提醒怎么 grep 确认）。

### 4.6 `recordAddr` / `loadAddr`：地址的存取

```java
public void recordAddr(String address) throws Exception {
  Properties prop = new Properties();
  final Resource resource = new ClassPathResource("contract.properties");
  java.io.File f = resource.getFile();
  if (f.exists()) {                                  // 保留已有键（比如 Asset 的 address）
    try (java.io.InputStream in = new java.io.FileInputStream(f)) {
      prop.load(in);
    }
  }
  prop.setProperty(ADDR_KEY, address);               // 写入 agriTraceAddress
  try (FileOutputStream out = new FileOutputStream(f)) {
    prop.store(out, "contract address");
  }
}

public String loadAddr() throws Exception {
  Properties prop = new Properties();
  final Resource resource = new ClassPathResource("contract.properties");
  prop.load(resource.getInputStream());
  String addr = prop.getProperty(ADDR_KEY);
  if (addr == null || addr.trim().equals("")) {
    throw new Exception(" load AgriTrace address failed, please deploy it first. ");
  }
  return addr;
}
```

**说清楚两件事：**

- `contract.properties` 是个 Java 的 `Properties`（键值对文件）。**Asset 用键 `address`，我们用键 `agriTraceAddress`** —— 两个合约各存各的，互不覆盖。
- `recordAddr` 里先 `if (f.exists()) prop.load(in)` 把旧内容读进来，是为了**不把 Asset 存的 `address` 覆盖掉**。这是比直接在 AssetClient 里 `new Properties()` 更稳妥的写法。

### 4.7 `check()`：判断写操作是否成功

```java
private void check(TransactionReceipt receipt, String action) {
  if (!receipt.isStatusOK()) {
    throw new RuntimeException(
        action + " failed, status = " + receipt.getStatus() + ", msg = " + receipt.getStatusMsg());
  }
}
```

- `receipt.isStatusOK()` — java-sdk 2.7 里判断交易执行成功的标准方法（返回 `boolean`）。
- 失败时抛异常，会被外层 `catch` 接住并打印原因（比如 `batch not registered` 这种 require 报错）。
- `getStatus()/getStatusMsg()` — 拿状态码和错误信息，排查问题用。

### 4.8 `main` / `Usage`：命令行入口

```java
public static void main(String[] args) throws Exception {
  if (args.length < 1) {
    Usage();
  }
  AgriTraceClient c = new AgriTraceClient();
  c.initialize();                          // 先连链
  switch (args[0]) {
    case "deploy":   c.deploy(); break;
    case "register":
      if (args.length < 5) { Usage(); }
      c.register(args[1], args[2], args[3], args[4]); break;
    case "addtrace":
      if (args.length < 5) { Usage(); }
      c.addTrace(args[1], args[2], args[3], args[4]); break;
    case "product":
      if (args.length < 2) { Usage(); }
      c.product(args[1]); break;
    case "trace":
      if (args.length < 3) { Usage(); }
      c.trace(args[1], Integer.parseInt(args[2])); break;
    case "count":
      if (args.length < 2) { Usage(); }
      c.count(args[1]); break;
    default: Usage();
  }
  System.exit(0);
}
```

- 把命令行第一个参数当"子命令"，后面的参数当参数传给对应方法。这就是 `bash agri_run.sh register batch001 ...` 能工作的原因。
- `System.exit(0)` 在最后，确保程序跑完能正常退出（否则 SDK 的线程池可能让 JVM 挂着不退出）。
- 每个分支前的 `args.length` 检查是**防空指针**：参数不够就打印用法。

---

## 5. 启动脚本 `agri_run.sh`

放在 `asset-app/tool/` 下，`gradle build` 时会被自动拷到 `dist/`。

```bash
#!/bin/bash

function usage() {
    echo "Usage:"
    echo "  bash agri_run.sh deploy"
    echo "  bash agri_run.sh register <batchId> <name> <origin> <producer>"
    echo "  bash agri_run.sh addtrace <batchId> <operation> <operator> <info>"
    echo "  bash agri_run.sh product  <batchId>"
    echo "  bash agri_run.sh trace    <batchId> <index>"
    echo "  bash agri_run.sh count    <batchId>"
    exit 0
}

case $1 in
    deploy) ;;
    register) [ $# -lt 5 ] && { usage; } ;;
    addtrace) [ $# -lt 5 ] && { usage; } ;;
    product)  [ $# -lt 2 ] && { usage; } ;;
    trace)    [ $# -lt 3 ] && { usage; } ;;
    count)    [ $# -lt 2 ] && { usage; } ;;
    *) usage ;;
esac

java -Djdk.tls.namedGroups="secp256k1" \
     -cp 'apps/*:conf/:lib/*' \
     org.fisco.bcos.asset.client.AgriTraceClient "$@"
```

**两个关键点：**

- `-Djdk.tls.namedGroups="secp256k1"` — **绝对不能省**。JDK 14 默认不带 `secp256k1` 曲线，不加这个参数会 SSL 握手失败、连不上链。
- `-cp 'apps/*:conf/:lib/*'` — classpath：`apps/` 是你的 jar，`conf/` 是配置和证书，`lib/` 是依赖库。三者缺一不可。

---

## 6. 完整跑一遍（从构建到查询）

> **构建必须用本地 Gradle：`~/opt/gradle-6.3/bin/gradle build`。千万不要用 `./gradlew`**——wrapper 每次都去 `services.gradle.org` 下载，网络不好就卡死（你已经踩过这个坑）。

```bash
# 1. 构建
cd ~/fisco/asset-app
chmod +x tool/agri_run.sh
~/opt/gradle-6.3/bin/gradle clean build

# 2. 到 dist 目录运行
cd dist

# 3. 部署合约
bash agri_run.sh deploy
# 输出：deploy AgriTrace success, contract address is 0x....

# 4. 登记产品
bash agri_run.sh register batch001 "赣南脐橙" "江西赣州" "某果业公司"

# 5. 追加溯源环节（必须先 register）
bash agri_run.sh addtrace batch001 "种植" "张农户" "2026-03 春季种植，施用有机肥"
bash agri_run.sh addtrace batch001 "加工" "李工"   "2026-08 分拣清洗包装"
bash agri_run.sh addtrace batch001 "运输" "某物流" "2026-09 冷链运输至上海"

# 6. 查询
bash agri_run.sh product batch001
bash agri_run.sh count  batch001
bash agri_run.sh trace  batch001 0
bash agri_run.sh trace  batch001 1
```

**预期输出：**

```
deploy AgriTrace success, contract address is 0x....
register success => batchId: batch001, name: 赣南脐橙
addTrace success => batchId: batch001, operation: 种植
addTrace success => batchId: batch001, operation: 加工
addTrace success => batchId: batch001, operation: 运输
product => batchId=batch001; name=赣南脐橙; origin=江西赣州; producer=某果业公司
trace count => 3
trace => #0 operation=种植; operator=张农户; info=2026-03 春季种植，施用有机肥
trace => #1 operation=加工; operator=李工; info=2026-08 分拣清洗包装
```

> 运行时看到 `WARNING: An illegal reflective access ... BouncyCastle` 是**正常现象**，JDK 14 对 BouncyCastle 的反射提示，不影响功能，忽略即可。

---

## 7. 常见坑速查表

| 现象 | 原因 | 解决 |
|------|------|------|
| `gradlew` 卡在 Downloading gradle-6.3-bin.zip | wrapper 总要联网下载 | 改用 `~/opt/gradle-6.3/bin/gradle build` |
| 编译报 `getReturnObject` / `TransactionResponse` 找不到 | 用了别的 SDK 版本 API | 写操作用 `TransactionReceipt`，读操作直接拿返回值（第 4 节） |
| `addTrace failed, batch not registered` | 没先 register | 先 `register` 再 `addtrace` |
| 中文乱码/参数被拆开 | shell 按空格拆参数 | 中文参数加引号 `"赣南脐橙"` |
| 重建后查询说地址不存在 | `gradle build` 覆盖了空 `contract.properties` | 重新 `deploy` |
| `sol2java.sh` 报 ABIEncoderV2 错 | 编译器对实验特性支持问题 | 删掉 `pragma experimental ABIEncoderV2;` |
| `push(t)` 报 struct 不能拷贝到 storage | 某些 solc 版本限制 | 改逐字段赋值或 `push(Trace(...))` |
| `byte(...)` 报废弃 | solc 是 0.6.x | 改成 `bytes1(...)`，或确认版本用 0.4.25 |
| SSL handshake / 连不上 | 缺 secp256k1 曲线 | 脚本里保留 `-Djdk.tls.namedGroups="secp256k1"` |

---

## 8. 知识地图（一页总结）

```
业务：一批农产品 batchId  →  登记基本信息 + 追加多个溯源环节
        │
        ▼
合约 AgriTrace.sol
  ├─ struct Product / Trace      数据结构（一条记录）
  ├─ mapping(string=>Product)    批次→产品
  ├─ mapping(string=>Trace[])    批次→溯源记录数组（一对多）
  ├─ register    (写)  防重复
  ├─ addTrace    (写)  防未登记，push 追加
  ├─ getProduct  (读)  拼字符串返回
  ├─ getTrace    (读)  拼字符串返回
  ├─ getTraceCount(读) 返回 length
  └─ uintToStr   工具：uint→string
        │
        ▼ sol2java.sh
生成 AgriTrace.java（不要手写）
  ├─ deploy / load
  ├─ 写函数 → TransactionReceipt
  └─ 读函数 → 直接返回 String / BigInteger
        │
        ▼
客户端 AgriTraceClient.java
  ├─ initialize()   连群组1、建账户
  ├─ deploy()       部署 + 存地址(agriTraceAddress)
  ├─ register/addTrace  → check(receipt.isStatusOK())
  ├─ product/trace/count→ 直接打印返回值
  ├─ recordAddr/loadAddr→ 读写 contract.properties
  └─ main()         子命令分发
        │
        ▼ agri_run.sh（带 secp256k1 参数）
    上链执行，读/写结果打印
```

**记住三句话就够了：**

1. **合约管数据结构和规则**：`struct + mapping` 存数据，`register/addTrace` 改数据，`getXxx` 查数据（拼成字符串最省事）。
2. **写用回执、读直接拿**：`TransactionReceipt.isStatusOK()` 判写成功；读操作生成方法直接返回 `String`/`BigInteger`。
3. **地址要存、参数加引号、构建用本地 gradle、脚本带 secp256k1**——这四个细节决定你能否顺利跑通。
