# 前置条件：CKB 开发环境

在开始 dApp（前端）或脚本（链上）开发前，应先搭建好 CKB 基础开发环境，包括 Node.js、Rust、Clang、ckb-debugger、ckb-js-vm 等工具。不同操作系统和开发需求可能有所不同。

**完整、最新的搭建流程请查阅官方安装指南：**

- [CKB 安装指南](https://docs.nervos.org/docs/getting-started/installation-guide)

> 该指南涵盖所有必需工具和各平台详细说明。请务必先完成环境搭建，再继续阅读下方内容。

---

## 脚手架准备

CKB 和 RGB++ 开发通常分为两类环境：

### 1. 前端/dApp 开发（链下）

- **目标：** 构建用户界面、连接钱包、与 CKB 节点和合约交互。
- **典型流程：**
  1. 推荐使用官方 CCC 工具脚手架快速初始化 dApp 项目：

     ```sh
     pnpm create ccc-app my-ccc-app
     # 或
     npx create-ccc-app@latest my-ccc-app
     # 或
     yarn create ccc-app my-ccc-app
     ```

  2. 按照提示选择你喜欢的前端框架，开始开发。
  3. 可使用 [CCC Playground](https://docs.ckbccc.com/docs/CCC#try-in-the-playground) 在浏览器中即时实验和分享代码。
  4. 进阶用法、手动安装、代码示例等请查阅官方 CCC 文档。
- **权威文档：**
  - [CCC 官方文档](https://docs.ckbccc.com/docs/CCC)
  - [CCC（CKBers' Codebase）GitHub](https://github.com/ckb-devrel/ccc)

### 2. 脚本开发（链上，Rust 或 JS）

- **目标：** 用 Rust 或 JavaScript（ckb-js-vm）编写和测试 CKB 链上脚本（智能合约）。

#### Rust 典型流程

1. 按照 [Rust Quick Start](https://docs.nervos.org/docs/script/rust/rust-quick-start) 使用 `ckb-script-templates` 脚手架初始化项目：

   ```sh
   cargo generate gh:cryptape/ckb-script-templates workspace --name rust-script-examples
   ```

2. 进入项目目录并生成合约：

   ```sh
   cd rust-script-examples
   make generate CRATE=hello-world
   ```

3. 在 `contracts/hello-world/src/main.rs` 实现 `program_entry` 入口函数。
4. 编译所有合约：

   ```sh
   make build
   ```

5. 用 `ckb-debugger` 或 `ckb-testtool` 本地测试和调试：

   ```sh
   ckb-debugger --bin build/release/hello-world
   # 或
   cargo test -- tests::test_hello_world --nocapture
   ```

6. 详细流程和最新用法请查阅 [官方 Rust 快速入门](https://docs.nervos.org/docs/script/rust/rust-quick-start)。

#### JS 典型流程

1. 按照 [JS Quick Start](https://docs.nervos.org/docs/script/js/js-quick-start) 使用 `pnpm create ckb-js-vm-app` 脚手架初始化 TypeScript 合约项目：

   ```sh
   pnpm create ckb-js-vm-app
   ```

2. 在 `packages/on-chain-script/src/index.ts` 中开发合约（推荐 TypeScript）。
3. 构建合约：

   ```sh
   pnpm build
   ```

4. 用 `ckb-debugger` 和 `ckb-testtool` 测试和调试：

   ```sh
   ckb-debugger --read-file dist/hello-world.bc --bin deps/ckb-js-vm -- -r
   # 或使用内置的 Jest 测试框架
   ```

5. 详细流程、进阶用法和最佳实践请查阅 [官方 JS Quick Start](https://docs.nervos.org/docs/script/js/js-quick-start)。

- **权威文档：**
  - [CKB Rust Script Quick Start](https://docs.nervos.org/docs/script/rust/rust-quick-start)
  - [CKB JS Script Quick Start](https://docs.nervos.org/docs/script/js/js-quick-start)

> 最新的环境搭建、最佳实践和常见问题，请始终查阅上述官方文档。CKB 生态更新较快，官方资源会持续维护。
