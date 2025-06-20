# 任务5：理解与使用 RGB++ 协议中的 BTC Time Lock

## 引言

在 RGB++ 协议中，BTC Time Lock（比特币时间锁）是实现比特币与 CKB 跨链安全转移的核心机制之一。开发者**无需自行实现时间锁脚本**，而是直接利用协议标准脚本，将比特币 UTXO 的状态与 CKB 资产锁定条件绑定，实现安全、可编程的跨链资产管理。

本节将结合 [RGB++ Guide](https://www.rgbppfans.com/) 讲解 BTC Time Lock 的原理、流程及实际用法，帮助开发者正确理解和使用这一机制。

---

## 什么是 BTC Time Lock？

BTC Time Lock 是 RGB++ 协议中的一种跨链安全时间锁。它允许 CKB 上的资产锁定条件与比特币链上的某个事件（如 UTXO 被花费、区块高度/时间戳达到等）绑定，只有满足比特币侧条件后，CKB 上的资产才能被解锁。

**核心要点：**

- CKB 资产的锁定状态与特定比特币 UTXO 的状态加密绑定。
- 解锁 CKB 资产时，必须提供比特币链上事件发生的证明（如 UTXO 已被花费、区块已产生等）。
- 该机制结合了比特币的安全性与 CKB 的可编程性，实现了无需信任的跨链资产流转。

---

## BTC Time Lock 的典型工作流程

（参考 [RGB++ Guide: Typical Transaction Flow](https://www.rgbppfans.com/)）

1. **链下预计算：**  
   - 用户准备 CKB 交易（CKB_TX_B），引用特定的比特币 UTXO（btc_utxo#1）作为单次密封（single-use seal）。
   - CKB 的 lock script 参数中编码了比特币 UTXO 及预期新 UTXO（btc_utxo#2），建立绑定关系。

2. **比特币交易提交：**  
   - 用户提交比特币交易，花费 btc_utxo#1 并创建 btc_utxo#2。
   - 该交易包含一个 OP_RETURN 输出，存储与 CKB 交易及新 UTXO 相关的承诺（commitment）。

3. **CKB 交易提交：**  
   - 用户提交准备好的 CKB 交易，并将比特币交易作为见证（witness）包含其中。
   - CKB 脚本校验：
     - 比特币交易有效且已被比特币链确认（通过 SPV 证明）。
     - 正确的 UTXO 被花费，commitment 匹配。
     - 时间锁或其他条件已满足。

4. **链上验证：**  
   - CKB 的脚本系统结合 SPV 证明，确保比特币侧事件（如时间锁到期、UTXO 被花费）真实发生，才允许 CKB 资产解锁或转移。

---

## 为什么要用 BTC Time Lock？

- **安全性：**  
  资产解锁条件锚定比特币链，继承比特币的安全性。
- **可编程性：**  
  CKB 脚本系统支持灵活的条件组合（如多签、时间锁、跨链触发等）。
- **用户体验：**  
  用户无需运行全节点或复杂的客户端验证，所有校验均由 CKB 脚本和 SPV 证明自动完成。

---

## 开发者如何实际使用 BTC Time Lock？

你**无需自己实现 BTC Time Lock 脚本**，只需：

1. **理解协议标准：**  
   - BTC Time Lock 已作为 RGB++ 协议标准脚本实现（见 [RGB++ Script Standard](https://www.rgbppfans.com/)）。
   - 只需调用标准 lock script，按协议流程构造交易即可。

2. **准备交易：**  
   - 在比特币到 CKB 的资产转移或"leap"场景下，构造 CKB 交易时正确引用比特币 UTXO，并填写必要参数。
   - 推荐使用 RGB++ 生态提供的 SDK 和工具（如 [ccc SDK](https://www.rgbppfans.com/)），自动完成交易构造和 SPV 证明生成。

3. **解锁操作：**  
   - 比特币侧事件（如确认数达到、UTXO 被花费）发生后，提交包含 SPV 证明的 CKB 交易。
   - CKB 脚本会自动校验比特币事件，满足条件后解锁资产。

**示例：**  

- 参考 [RGB++ Guide: Unlocking the BTC_TIME_lock](https://www.rgbppfans.com/) 获取 ccc SDK 代码样例和详细流程。

---

## BTC Time Lock 跨链质押资产实操分步教程

本节以"BTC → CKB 跨链质押"为例，详细讲解如何利用 RGB++ 协议的 BTC Time Lock 实现安全、可编程的资产锁定与解锁。

### 步骤一：环境与工具准备

1. **安装 CKB 节点与 BTC 节点（或使用公共节点）**
   - CKB 测试网/主网节点（用于链上交互）
   - BTC 测试网/主网节点（用于 UTXO 查询与交易广播）

2. **安装 RGB++ 相关 SDK/工具**
   - 推荐 [ccc SDK](https://www.rgbppfans.com/)（TypeScript/JavaScript）
   - 安装命令（以 npm 为例）：
     ```bash
     npm install @rgbpp/ccc
     ```

3. **准备钱包**
   - BTC 钱包（用于管理 UTXO 和签名）
   - CKB 钱包（用于管理 Cell 和签名）

---

### 步骤二：链下预计算与参数准备

1. **选择/生成 BTC UTXO 作为 single-use seal**
   - 查询你的 BTC 钱包，选定一个未花费的 UTXO（btc_utxo#1）

2. **确定质押参数**
   - 质押金额、锁定时长（如需时间锁）、接收方 CKB 地址等

3. **构造 CKB 交易（CKB_TX_B）**
   - 使用 ccc SDK 构造一笔 CKB 交易，输入为绑定 btc_utxo#1 的 RGB++ Cell
   - 输出为带有 BTC Time Lock 的新 Cell，lock script 参数需包含 btc_utxo#1、btc_utxo#2（新 UTXO 占位符）、commitment 等

   示例伪代码（TypeScript）：
   ```typescript
   const btcUtxo1 = { txid: '...', vout: 0 };
   const btcUtxo2 = { placeholder: true }; // 先用占位符
   const ckbTx = await ccc.rgbpp.buildBtcTimeLockTx({
     from: ckbAddress,
     btcUtxo1,
     btcUtxo2,
     amount: '100000000', // 1 BTC
     lockTime: 1680000000, // 可选，时间戳
   });
   ```

---

### 步骤三：提交比特币交易

1. **构造并广播 BTC 交易**
   - 花费 btc_utxo#1，创建 btc_utxo#2
   - 在 OP_RETURN 输出中写入 commitment（可由 SDK 自动生成）

   示例伪代码：
   ```typescript
   const btcTx = await ccc.rgbpp.buildBtcTxWithCommitment({
     from: btcAddress,
     to: btcAddress2,
     utxo: btcUtxo1,
     commitment: ccc.rgbpp.computeCommitment(ckbTx, btcUtxo1, btcUtxo2),
   });
   await btcClient.sendRawTransaction(btcTx);
   ```

2. **等待 BTC 交易确认**
   - 通常需 6 个区块确认，确保安全

---

### 步骤四：提交 CKB 交易并解锁

1. **准备 SPV 证明**
   - 使用 ccc SDK 查询 BTC 交易的 Merkle 证明（SPV proof）
   - SDK 会自动生成所需的 witness 数据

2. **补全 CKB 交易参数**
   - 用真实的 btc_utxo#2 替换占位符
   - 填写 SPV 证明

   示例伪代码：
   ```typescript
   const spvProof = await ccc.rgbpp.getSpvProof(btcTx.txid);
   const finalCkbTx = await ccc.rgbpp.completeBtcTimeLockTx({
     ckbTx,
     btcTx,
     spvProof,
   });
   ```

3. **广播 CKB 交易**
   - 使用 CKB 钱包签名并发送交易
   - 等待链上确认

---

### 步骤五：资产解锁与后续操作

- 一旦 CKB 交易上链且验证通过，BTC Time Lock 质押资产即可解锁/转移
- 后续可将资产转为普通 CKB Cell，或继续参与 DeFi、流动性等操作

---

### 常见问题与注意事项

- **BTC 交易必须真实上链且被足够确认，否则 CKB 侧无法解锁**
- **SPV 证明必须由可信服务或本地工具生成，防止伪造**
- **务必使用 RGB++ 官方 SDK 和标准脚本，避免自定义实现带来安全隐患**
- **所有参数（UTXO、commitment、lock args）需严格按协议格式填写**

---

## 参考资料

- [RGB++ Guide](https://www.rgbppfans.com/)
- [RGB++ Script Standard](https://www.rgbppfans.com/)
- [RGB++ SDK](https://www.rgbppfans.com/)
- [BTC Time Lock: 典型交易流程](https://www.rgbppfans.com/)

---

**总结：**  
RGB++ 协议中的 BTC Time Lock 是实现比特币与 CKB 跨链安全、可编程资产管理的协议级特性。开发者应直接使用标准脚本和 SDK，关注交易构造和证明提交，无需重复造轮子实现底层锁定逻辑。 