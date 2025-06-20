# 阶段 7: 高级话题 - Spore 协议与链上数字对象 (DOB)

## 引言

在之前的章节中，我们已经构建了一个功能相对完整的 Staking dApp。然而，一个真正成熟的去中心化应用，其价值不仅仅在于核心功能，更在于其**资产的可组合性、可扩展性和在整个生态中的互操作性**。

这正是 CKB 生态中 **Spore 协议** 和 **数字对象 (Digital Object, DOB)** 发挥巨大威力的地方。

本章将带你进入一个更广阔的世界，探索如何将我们的 Staking 凭证升级为一种标准化的、可被整个生态（钱包、市场、浏览器）识别和渲染的链上数字资产。

**学习目标:**

1. 理解 Spore 协议的核心思想及其在 CKB 生态中的地位。
2. 学习什么是数字对象 (DOB) 以及它如何为链上资产提供丰富的表现力。
3. 掌握如何使用 `dob-cookbook` 作为实践指南，为我们的 Staking dApp 创建一个 DOB 作为质押凭证。
4. 探索 DOB 的不同渲染模式和最佳实践。

## 步骤一：Spore 协议 —— CKB 上的数字资产发行标准

Spore 是 CKB 上的一个开源、无需许可的数字资产发行协议。你可以把它理解为 CKB 版本的 ERC-721 (NFT) 或 ERC-1155，但它远不止于此。

**核心特性:**

- **链上存储**: Spore 协议强调将资产的**核心数据尽可能存储在 CKB 链上**。这与许多其他链将元数据存储在 IPFS 或 Arweave 等链下方案形成对比，提供了更强的去中心化和持久性保证。
- **可扩展性**: Spore 协议是高度可扩展的。它允许开发者通过“共建”模式，为协议添加新的功能和特性，而无需分叉或获得中心化机构的许可。
- **互操作性**: 基于 Spore 协议发行的资产，可以无缝地在所有支持 Spore 的 CKB 应用和平台之间流转和使用。

## 步骤二：DOB - 会“呼吸”的数字对象

如果说 Spore 是协议层，那么 DOB 就是其上具体的实现和表现层。DOB 的核心思想是，让链上资产不仅仅是一串静态的数据，而是能够**自我描述、自我渲染**的“活”对象。

`dob-cookbook` 是 Spore 协议官方提供的一个全面的实战代码库，它展示了发行和渲染 DOB 的各种范例和最佳实践。

**一个 DOB 主要包含两部分：**

1. **`content` 字段**: 这是资产的核心数据，通常是一个 URI，指向一个链上或链下的资源。例如，它可以是一段 SVG 代码、一张存储在 BTCFS (一种 CKB 链上文件系统) 中的图片，或是一个 IPFS 哈希。
2. **`contentType` 字段**: 这个字段是 DOB 的“灵魂”。它定义了 `content` 字段应如何被解析和渲染。例如：
    - `"image/svg+xml"`: 告诉钱包或市场，`content` 是一段 SVG 代码，请直接渲染它。
    - `"application/json;dob_extension=background_image"`: 表示 `content` 是一个 JSON 对象，它遵循 `background_image` 的扩展规范，通常用于为 DOB 添加可组合的背景层。
    - `"text/plain"`: 表示 `content` 是纯文本。

这种设计使得 DOB 具有极强的表现力和灵活性。

## 步骤三：实战 - 为我们的 Staking dApp 创建 DOB 凭证

现在，让我们把理论付诸实践。我们将修改 Staking dApp，当用户质押资产时，不再只是创建一个简单的 `time-lock` Cell，而是创建一个符合 Spore/DOB 标准的**质押凭证 DOB**。

这个 DOB 将具备以下特性：

- 它本身是一个 NFT，可以被转让或在市场上交易。
- 它能直观地向用户展示其质押状态，例如质押的金额和解锁时间。

我们将参考 `dob-cookbook` 中的 `programmatic-img` 范例 ([9.programmatic-img.md](https://github.com/sporeprotocol/dob-cookbook/blob/main/examples/dob0/9.programmatic-img.md))。这个范例展示了如何创建一个动态生成的、内容由链上状态决定的 SVG 图像。

1. **定义 DOB 的 `type script`**
    首先，我们需要为我们的 Staking DOB 定义一个独一无二的 `type script`。这个脚本将作为我们这类资产的唯一标识。通常，我们可以选择在创建第一个 DOB 时，将一笔交易的 `input` 作为 `type script` 的 `args`，这能保证其全局唯一性。

2. **改造 Staking 交易**
    在 `onStake` 函数中，我们需要修改交易的 `output`，使其符合 Spore 协议的结构。

    ```javascript
    // ... onStake function
    const onStake = async (amount, lockUntilTimestamp) => {
      // ...
      // 1. 生成质押凭证 DOB 的内容 (动态 SVG)
      const svgContent = `
        <svg width="200" height="200" xmlns="http://www.w3.org/2000/svg">
          <rect width="100%" height="100%" fill="#f0f0f0" />
          <text x="50%" y="30%" font-size="16" text-anchor="middle">CKB Staking Voucher</text>
          <text x="50%" y="55%" font-size="20" text-anchor="middle" fill="#007bff">
            ${amount} CKB
          </text>
          <text x="50%" y="75%" font-size="12" text-anchor="middle">
            Unlocks at: ${new Date(lockUntilTimestamp * 1000).toLocaleString()}
          </text>
        </svg>
      `;

      // 2. 构造 Spore Cell 的 output
      const sporeOutput = {
        capacity: "...", // 需要足够存储 Spore 数据的 capacity
        lock: signer.address.getLock(), // 归用户所有
        type: {
          codeHash: "SPORE_PROTOCOL_TYPE_SCRIPT_CODE_HASH", // Spore 官方部署的 codeHash
          hashType: "type",
          args: "UNIQUE_STAKING_DOB_TYPE_ID", // 之前定义的唯一 type script args
        },
      };

      // 3. 构造 Spore Cell 的 data
      const sporeData = {
        contentType: "image/svg+xml",
        content: `0x${Buffer.from(svgContent).toString("hex")}`,
        // clusterId 字段可以用于将一组相关的 DOB 归类
      };

      // 4. 同时，我们还需要将真正的质押资产放入一个由 Spore Cell 控制的 Lock Proxy Cell 中
      //    这样，DOB 的持有者才有权在未来解锁质押的 CKB
      const lockProxyLock = {
        codeHash: "SPORE_LOCK_PROXY_CODE_HASH",
        hashType: "type",
        args: sporeOutput.type.args, // 指向 Spore DOB
      };

      const timeLockCell = {
        capacity: BigInt(amount) * 100000000n,
        lock: {
          // 依然使用 time-lock，但它的解锁权力现在与 Spore DOB 绑定
          codeHash: "YOUR_TIME_LOCK_SCRIPT_CODE_HASH",
          hashType: "type",
          args:
            `0x${lockUntilTimestamp.toString(16)}` +
            lockProxyLock.args.slice(2),
        },
      };

      // 5. 发送交易
      const txHash = await signer.sendTransaction({
        outputs: [sporeOutput, timeLockCell],
        outputsData: [
          `0x${Buffer.from(JSON.stringify(sporeData)).toString("hex")}`,
          "0x",
        ],
      });

      console.log("Staking DOB created! Tx Hash:", txHash);
    };
    ```

## 步骤四：生态兼容性与最佳实践

通过遵循 Spore 和 DOB 协议，我们创建的 Staking 凭证将自动被 CKB 生态系统所兼容。

- **钱包内渲染**: 像 JoyID 这样的现代 CKB 钱包，会自动识别出这是一个 DOB，并根据其 `contentType` 直接将我们生成的 SVG 渲染出来，用户可以直观地看到自己的质押信息。
- **NFT 市场**: 像 Omiga 或 Element 这样的市场，会自动将我们的 Staking 凭证作为 NFT 进行索引和展示，允许用户进行交易。
- **Cookbook 的重要性**: `dob-cookbook` 不仅仅是代码范例，它更是**兼容性测试的黄金标准**。Cookbook 中的表格详细列出了各种 DOB 在不同平台（JoyID, Omiga, CKB Explorer 等）上的渲染兼容性情况。在设计你的 DOB 时，强烈建议参考此表格，以确保你的数字资产能在目标平台获得最佳的展示效果。

## 总结

本章为你打开了通往 CKB 高级应用开发的大门。通过将 Spore/DOB 协议集成到我们的 dApp 中，我们不仅仅是创建了一个功能，而是创造了一种**标准化的、可互操作的、具有丰富表现力的数字资产**。

这正是 CKB 作为 "Store of Assets" (资产存储) Layer 1 的核心价值所在。一个设计良好的 CKB dApp，其产出的资产应该能像乐高积木一样，在整个生态中被自由地组合和应用。

**接下来去哪儿？**

- 深入研究 `dob-cookbook` 的其他范例，特别是涉及多媒体和可组合性的例子。
- 阅读 Spore 协议的官方文档，理解其更深层次的设计哲学和技术细节。
- 尝试为你自己的项目设计和实现一种独特的 DOB。
