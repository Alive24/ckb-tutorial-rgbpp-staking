# Prerequisite: CKB Development Environment

Before starting with dApp (frontend) or script (on-chain) development, set up the basic CKB development environment (Node.js, Rust, Clang, ckb-debugger, ckb-js-vm, etc.). The exact steps and tools may vary by OS and needs.

**For a complete, up-to-date setup process, always refer to the official Installation Guide:**

- [CKB Installation Guide](https://docs.nervos.org/docs/getting-started/installation-guide)

> This guide covers all required tools and platform-specific instructions. Please follow it before proceeding.

---

## Scaffolding

CKB and RGB++ development typically involves two environments:

### 1. Frontend/dApp Development (Off-chain)

- **Goal:** Build user interfaces, connect wallets, and interact with CKB nodes and smart contracts.
- **Typical Path:**
  1. Scaffold a new dApp project using the official CCC toolkit:

     ```sh
     pnpm create ccc-app my-ccc-app
     # or
     npx create-ccc-app@latest my-ccc-app
     # or
     yarn create ccc-app my-ccc-app
     ```

  2. Follow the prompts to select your preferred framework and start building.
  3. Use the [CCC Playground](https://docs.ckbccc.com/docs/CCC#try-in-the-playground) for instant browser-based experimentation and code sharing.
  4. For advanced usage, manual installation, and code examples, refer to the official CCC docs.
- **Authoritative Guide:**
  - [CCC Documentation (Official)](https://docs.ckbccc.com/docs/CCC)
  - [CCC (CKBers' Codebase) GitHub](https://github.com/ckb-devrel/ccc)

### 2. Script Development (On-chain, Rust or JS)

- **Goal:** Write and test CKB on-chain scripts (smart contracts) in Rust or JavaScript (ckb-js-vm).

#### Rust Typical Path

1. Use [`ckb-script-templates`](https://docs.nervos.org/docs/script/rust/rust-quick-start) to scaffold your project:

   ```sh
   cargo generate gh:cryptape/ckb-script-templates workspace --name rust-script-examples
   ```

2. Enter your project and generate a contract:

   ```sh
   cd rust-script-examples
   make generate CRATE=hello-world
   ```

3. Implement your contract logic in `contracts/hello-world/src/main.rs` with the `program_entry` function.
4. Build all contracts:

   ```sh
   make build
   ```

5. Test and debug locally with `ckb-debugger` or `ckb-testtool`:

   ```sh
   ckb-debugger --bin build/release/hello-world
   # or
   cargo test -- tests::test_hello_world --nocapture
   ```

6. For full details and up-to-date workflow, always refer to the [official Rust Quick Start](https://docs.nervos.org/docs/script/rust/rust-quick-start).

#### JS Typical Path

1. Use [`pnpm create ckb-js-vm-app`](https://docs.nervos.org/docs/script/js/js-quick-start) to scaffold a TypeScript contract project:

   ```sh
   pnpm create ckb-js-vm-app
   ```

2. Edit your contract in `packages/on-chain-script/src/index.ts` (TypeScript recommended).
3. Build the contract:

   ```sh
   pnpm build
   ```

4. Test and debug with `ckb-debugger` and `ckb-testtool`:

   ```sh
   ckb-debugger --read-file dist/hello-world.bc --bin deps/ckb-js-vm -- -r
   # or use the provided Jest test framework
   ```

5. For full details, advanced usage, and best practices, always refer to the [official JS Quick Start](https://docs.nervos.org/docs/script/js/js-quick-start).

- **Authoritative Docs:**
  - [CKB Rust Script Quick Start](https://docs.nervos.org/docs/script/rust/rust-quick-start)
  - [CKB JS Script Quick Start](https://docs.nervos.org/docs/script/js/js-quick-start)

> For the latest setup, best practices, and troubleshooting, always refer to the official documentation linked above. The CKB ecosystem evolves quickly, and these resources are kept up to date.
