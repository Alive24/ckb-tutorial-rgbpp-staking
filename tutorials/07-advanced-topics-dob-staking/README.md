# Phase 7: Advanced Topics - Spore Protocol and On-Chain Digital Objects (DOBs)

## Introduction

In the previous chapters, we built a relatively complete Staking dApp. However, the value of a truly mature decentralized application lies not just in its core functionality, but in its **asset composability, extensibility, and interoperability within the entire ecosystem**.

This is where the **Spore Protocol** and **Digital Objects (DOBs)** in the CKB ecosystem demonstrate their immense power.

This chapter will take you into a broader world, exploring how to upgrade our staking vouchers into a standardized on-chain digital asset that can be recognized and rendered by the entire ecosystem (wallets, marketplaces, explorers).

**Learning Objectives:**

1.  Understand the core ideas of the Spore Protocol and its position in the CKB ecosystem.
2.  Learn what a Digital Object (DOB) is and how it provides rich expressiveness for on-chain assets.
3.  Master how to use the `dob-cookbook` as a practical guide to create a DOB as a staking voucher for our dApp.
4.  Explore different rendering modes and best practices for DOBs.

## Step 1: Spore Protocol - The Digital Asset Issuance Standard on CKB

Spore is an open-source, permissionless protocol for issuing digital assets on CKB. You can think of it as the CKB version of ERC-721 (NFT) or ERC-1155, but it is much more than that.

**Core Features:**

-   **On-Chain Storage**: The Spore protocol emphasizes storing the **core data of assets on the CKB chain as much as possible**. This contrasts with many other chains that store metadata off-chain in solutions like IPFS or Arweave, providing stronger decentralization and durability guarantees.
-   **Extensibility**: The Spore protocol is highly extensible. It allows developers to add new features and functionalities to the protocol through a "co-build" model, without needing to fork or get permission from a centralized entity.
-   **Interoperability**: Assets issued based on the Spore protocol can seamlessly flow and be used across all CKB applications and platforms that support Spore.

## Step 2: DOBs - "Living" Digital Objects

If Spore is the protocol layer, then DOB is its concrete implementation and presentation layer. The core idea of DOB is to make on-chain assets not just a static string of data, but "living" objects that can **self-describe and self-render**.

The `dob-cookbook` is a comprehensive, practical repository provided by the Spore Protocol team, showcasing various examples and best practices for issuing and rendering DOBs.

**A DOB primarily consists of two parts:**

1.  **`content` field**: This is the core data of the asset, usually a URI pointing to an on-chain or off-chain resource. For example, it could be a piece of SVG code, an image stored on BTCFS (a CKB on-chain file system), or an IPFS hash.
2.  **`contentType` field**: This field is the "soul" of the DOB. It defines how the `content` field should be parsed and rendered. For example:
    -   `"image/svg+xml"`: Tells a wallet or marketplace that the `content` is a piece of SVG code and should be rendered directly.
    -   `"application/json;dob_extension=background_image"`: Indicates that the `content` is a JSON object that follows the `background_image` extension specification, often used for adding a composable background layer to a DOB.
    -   `"text/plain"`: Indicates that the `content` is plain text.

This design gives DOBs tremendous expressiveness and flexibility.

## Step 3: Hands-On - Creating DOB Vouchers for Our Staking dApp

Now, let's put theory into practice. We will modify our Staking dApp so that when a user stakes assets, instead of just creating a simple `time-lock` Cell, we create a **staking voucher DOB** that conforms to the Spore/DOB standard.

This DOB will have the following features:

-   It is an NFT itself and can be transferred or traded on marketplaces.
-   It can visually display its staking status to the user, such as the staked amount and unlock time.

We will refer to the `programmatic-img` example from the `dob-cookbook` ([9.programmatic-img.md](https://github.com/sporeprotocol/dob-cookbook/blob/main/examples/dob0/9.programmatic-img.md)). This example shows how to create a dynamically generated SVG image whose content is determined by on-chain state.

1.  **Define the DOB's `type script`**
    First, we need to define a unique `type script` for our Staking DOB. This script will serve as the unique identifier for this class of assets. Typically, we can choose an `input` from the transaction that creates the first DOB to be the `args` for the `type script`, ensuring its global uniqueness.

2.  **Refactor the Staking Transaction**
    In the `onStake` function, we need to modify the transaction's `output` to conform to the Spore protocol's structure.

    ```javascript
    // ... onStake function
    const onStake = async (amount, lockUntilTimestamp) => {
      // ...
      // 1. Generate the content for the staking voucher DOB (dynamic SVG)
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

      // 2. Construct the output for the Spore Cell
      const sporeOutput = {
        capacity: '...', // Capacity needs to be sufficient to store Spore data
        lock: signer.address.getLock(), // Owned by the user
        type: {
          codeHash: 'SPORE_PROTOCOL_TYPE_SCRIPT_CODE_HASH', // The officially deployed codeHash for Spore
          hashType: 'type',
          args: 'UNIQUE_STAKING_DOB_TYPE_ID', // The previously defined unique type script args
        },
      };

      // 3. Construct the data for the Spore Cell
      const sporeData = {
        contentType: 'image/svg+xml',
        content: `0x${Buffer.from(svgContent).toString('hex')}`,
        // The clusterId field can be used to group related DOBs
      };

      // 4. We also need to place the actual staked assets into a Lock Proxy Cell controlled by the Spore Cell.
      //    This way, only the holder of the DOB has the right to unlock the staked CKB in the future.
      const lockProxyLock = {
        codeHash: 'SPORE_LOCK_PROXY_CODE_HASH',
        hashType: 'type',
        args: sporeOutput.type.args, // Points to the Spore DOB
      };

      const timeLockCell = {
        capacity: BigInt(amount) * 100000000n,
        lock: {
          // Still use the time-lock, but its unlocking power is now tied to the Spore DOB
          codeHash: 'YOUR_TIME_LOCK_SCRIPT_CODE_HASH',
          hashType: 'type',
          args: `0x${lockUntilTimestamp.toString(16)}` + lockProxyLock.args.slice(2),
        },
      };

      // 5. Send the transaction
      const txHash = await signer.sendTransaction({
        outputs: [sporeOutput, timeLockCell],
        outputsData: [`0x${Buffer.from(JSON.stringify(sporeData)).toString('hex')}`, '0x'],
      });

      console.log('Staking DOB created! Tx Hash:', txHash);
    };
    ```

## Step 4: Ecosystem Compatibility and Best Practices

By following the Spore and DOB protocols, the staking vouchers we create will be automatically compatible with the CKB ecosystem.

-   **In-Wallet Rendering**: Modern CKB wallets like JoyID will automatically recognize this as a DOB and render the SVG we generated directly based on its `contentType`, allowing users to intuitively see their staking information.
-   **NFT Marketplaces**: Marketplaces like Omiga or Element will automatically index and display our staking vouchers as NFTs, allowing users to trade them.
-   **The Importance of the Cookbook**: The `dob-cookbook` is not just a collection of code examples; it is the **gold standard for compatibility testing**. The tables in the cookbook detail the rendering compatibility of various DOBs on different platforms (JoyID, Omiga, CKB Explorer, etc.). When designing your DOB, it is highly recommended to refer to this table to ensure your digital assets achieve the best possible display effect on your target platforms.

## Conclusion

This chapter has opened the door to advanced CKB dApp development for you. By integrating the Spore/DOB protocol into our dApp, we are not just creating a feature, but a **standardized, interoperable, and richly expressive digital asset**.

This is the core value of CKB as the "Store of Assets" Layer 1. A well-designed CKB dApp should produce assets that can be freely combined and used throughout the ecosystem, like Lego bricks.

**Where to go next?**

-   Dive deeper into other examples in the `dob-cookbook`, especially those involving multimedia and composability.
-   Read the official Spore protocol documentation to understand its deeper design philosophy and technical details.
-   Try to design and implement a unique DOB for your own project. 