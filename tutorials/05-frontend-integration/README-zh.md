# 阶段 5: Staking dApp 实战 - 从零构建一个功能完备的去中心化质押应用

## 引言

在前面的章节中，我们已经深入学习了 CKB 原生时间锁和 RGB++ BTC-Time-Lock 的底层原理。现在，是时候将这些理论知识转化为实际应用了！

本阶段的目标是**从零开始，构建一个功能完整的 Staking dApp**。我们将利用 Nervos 官方重磅推出的 `CCC` (CKBers' Codebase) 开发套件和颠覆性的 `JoyID` Passkey 钱包，带领你体验 CKB dApp 开发的现代化流程。

**最终成果：** 一个简洁、易用的 Web 应用，用户可以通过浏览器：

1. 无缝连接 JoyID 钱包。
2. 发起一笔资产质押交易，并设置锁定时长。
3. 实时查询自己所有已质押的资产及其状态。

## 步骤 〇：部署智能合约 (ccc-deploy 指南)

在构建前端 dApp 之前，我们必须先将 `ckb-time-lock-script` 部署到 CKB 测试网。部署成功后，我们将获得合约的 `codeHash`，这是前端与我们的智能合约交互的唯一标识。

我们将使用 `ccc-deploy`，一个强大的 CKB 脚本部署工具。

### 1. 安装 ccc-deploy

首先，你需要通过 npm 全局安装 `ccc-deploy` CLI 工具。打开你的终端并运行：

```bash
npm install -g ccc-deploy
```

安装完成后，你可以通过 `ccc-deploy --version` 来验证安装是否成功。

### 2. 准备部署环境

请确保你已经按照 **[章节 03](../03-basic-ckb-time-lock-script/README-zh.md)** 的指引，创建了 `ckb-time-lock-script` 项目。

进入该项目目录：

```bash
cd path/to/your/ckb-time-lock-script
```

### 3. 执行交互式部署

`ccc-deploy` 提供了交互式的部署流程，极大地简化了操作。运行以下命令启动部署：

```bash
ccc-deploy deploy ckb_time_lock_script --privateKey YOUR_PRIVATE_KEY
```

**参数说明:**

- `ckb_time_lock_script`: 这是你在项目中定义的脚本名称。`ccc-deploy` 会根据此名称查找对应的配置。
- `--privateKey YOUR_PRIVATE_KEY`: 指定用于签名部署交易的私钥。为了安全，`ccc-deploy` 默认会从 `.env` 文件中读取 `MAIN_WALLET_PRIVATE_KEY` 环境变量。你也可以直接在命令行中提供。

`ccc-deploy` 将会引导你完成以下步骤：

- 编译 Rust 脚本（如果需要）。
- 构造部署交易。
- 使用你的私钥签名交易。
- 将交易发送至 CKB 测试网。

### 4. 获取部署结果

交易成功上链后，`ccc-deploy` 会在你的项目根目录创建一个 `ccc.json` (或更新 `config.toml`，取决于工具版本) 文件，其中包含了部署信息。

打开 `ccc.json` 文件:

```json
{
  "script": {
    "codeHash": "0x...YOUR_TIME_LOCK_SCRIPT_CODE_HASH...",
    "hashType": "type",
    "txHash": "0x...deployment_transaction_hash...",
    "index": "0x0",
    "depType": "code"
  }
}
```

**请务必复制 `codeHash` 的值**。在后续步骤中，我们将使用它来构造和查询我们的质押 Cell。

## 步骤一：项目初始化 —— 拥抱 `CCC` 脚手架

`CCC` 是 CKB 生态系统最新的 dApp 开发基础设施，它极大地简化了与 CKB 节点的交互、交易构建和状态查询。忘记繁琐的 RPC 手动拼接吧，`CCC` 为我们提供了一站式解决方案。

1. **创建项目**
   打开终端，运行以下官方推荐命令，创建一个新的 dApp 项目：

   ```bash
   pnpm create ccc-app my-staking-dapp
   ```

   在交互式提示中，你可以选择自己喜欢的前端框架（如 React + Vite）。`create-ccc-app` 会自动为我们配置好所有必需的依赖和基本项目结构。

2. **项目结构概览**
   项目创建后，你会看到一个清晰的目录结构。请特别关注 `ccc` 相关的配置文件，这里定义了 dApp 如何与 CKB 网络（主网或测试网）连接。

## 步骤二：集成 `JoyID` 钱包 —— 告别助记词的未来

为了让 dApp 触及最广泛的用户，我们将集成 `JoyID` 钱包。它基于 WebAuthn 标准，允许用户通过面容 ID、指纹或 PIN 等设备原生方式管理资产，无需记忆复杂的助记词，极大提升了安全性和用户体验。

1. **实现连接逻辑**
   `CCC` 内置了对 `JoyID` 的完美支持。我们只需在前端代码中初始化一个 `JoyIDSigner` 即可。

   下面是一个简单的 React 组件示例，展示了如何实现 "连接钱包" 按钮：

   ```jsx
   // src/components/ConnectButton.jsx
   import { JoyIDSigner } from "@joyid/ckb";
   import { useState } from "react";

   export function ConnectButton() {
     const [address, setAddress] = useState(null);

     const onConnect = async () => {
       const signer = new JoyIDSigner();
       // The address can be used to interact with CKB network
       setAddress(signer.address);
       // You can save the signer instance to your app's state for later use
     };

     return (
       <div>
         {address ? (
           <p>Connected: {address.toCKBAddress()}</p>
         ) : (
           <button onClick={onConnect}>Connect with JoyID</button>
         )}
       </div>
     );
   }
   ```

2. **UI 展示**
   当用户成功连接后，我们可以在界面上显示他们的 CKB 地址，表明 dApp 已准备好进行链上交互。

## 步骤三：核心功能 —— 实现 Staking 交易

`CCC` 将交易的构建、签名、发送等复杂流程封装在 `Signer` 对象中。我们不再需要手动构建交易，只需描述我们的意图即可。

```javascript
// 假设 'signer' 是一个已连接的 JoyIDSigner 实例

const onStake = async (amount, lockUntilTimestamp) => {
  try {
    // 1. 准备 Staking cell 的 lock script
    //    这是我们唯一需要手动构造的部分，用来定义资产的新状态。
    const lockScript = {
      codeHash: "YOUR_TIME_LOCK_SCRIPT_CODE_HASH",
      hashType: "type",
      // Args: lock until timestamp + original lock hash of the user
      // 注意：这里需要将时间戳（数字）转换为十六进制字符串
      args:
        `0x${lockUntilTimestamp.toString(16)}` +
        signer.address.getLockHash().slice(2),
    };

    // 2. CCC 会在 signer.sendTransaction 内部自动处理所有繁琐工作：
    //    - 查找合适的 live cells 作为 inputs
    //    - 构建交易
    //    - 计算找零并创建 change output
    //    - 弹出 JoyID 进行签名
    //    - 发送交易到 CKB 节点
    const txHash = await signer.sendTransaction({
      outputs: [
        {
          lock: lockScript,
          // 注意：金额需要是 bigint 类型，并以 shannon 为单位 (1 CKB = 100,000,000 shannons)
          capacity: BigInt(amount) * 100000000n,
        },
      ],
    });

    console.log("质押交易已发送！交易哈希:", txHash);
    // 在这里更新 UI，显示交易成功信息和 explorer 链接
  } catch (error) {
    console.error("Staking failed:", error);
    // 在这里处理错误，向用户显示提示
  }
};
```

## 步骤四：查询用户的质押状态

查询链上状态（如获取 Live Cells）的功能，由 `CCC` 的 `Client` 对象提供。

下面是一个使用 React Hook 的更佳实践示例：

```javascript
import { Client } from "@ckb-ccc/connector";
import { useEffect, useState } from "react";

// CCC Client 用于和 CKB 节点/Indexer 通信，应在应用中全局初始化
const client = new Client({
  rpcUrl: "YOUR_CKB_RPC_URL",
  indexerUrl: "YOUR_CKB_INDEXER_URL",
});

export function useStakedCells(address) {
  const [stakedCells, setStakedCells] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (!address) return;

    const fetchStakedCells = async () => {
      setLoading(true);
      try {
        // 我们可以通过构造一个 lock script 模板来查询所有匹配的 live cells
        const timeLockScriptTemplate = {
          codeHash: "YOUR_TIME_LOCK_SCRIPT_CODE_HASH",
          hashType: "type",
          // 通过用户地址的 lock hash 作为查询参数的前缀，
          // 来找到所有属于该用户的质押 cell。
          args: address.getLockHash(),
        };

        // 使用 client.getLiveCells 来进行强大的查询
        // argsLen: 'any' 表示我们不关心 args 的确切长度，只要前缀匹配即可
        const cells = await client.getLiveCells({
          lock: timeLockScriptTemplate,
          argsLen: "any",
        });

        setStakedCells(cells);
      } catch (error) {
        console.error("Failed to fetch staked cells:", error);
      } finally {
        setLoading(false);
      }
    };

    fetchStakedCells();
  }, [address]);

  return { stakedCells, loading };
}
```

## 总结与注意事项

恭喜你！通过本章的学习，你已经掌握了使用 `CCC` 和 `JoyID` 构建一个现代化 CKB dApp 的全过程。

- **`CCC` 的威力**：它抽象了底层的复杂性，让你能用更少的代码、更清晰的逻辑来构建、查询和发送交易。
- **`JoyID` 的体验**：它为你的 dApp 带来了 Web2 一般的流畅体验，是吸引大规模用户的关键。
- **关于 Cycle 限制**：正如 `CKB Awesome Issues` 中所讨论的，CKB 交易有 `max_tx_verify_cycles` 限制。复杂的脚本可能会消耗大量 Cycle。在设计 Staking 合约和构建交易时，务必通过 `ckb-debugger` 等工具提前预估性能，确保交易不会因为超出 Cycle 限制而被节点拒绝。

这个 Staking dApp 是一个坚实的基础。在此之上，你可以继续扩展功能，例如实现解锁逻辑、引入可替代的质押凭证等，我们将在后续的章节中继续探索。
