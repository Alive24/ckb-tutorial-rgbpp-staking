# CKB 时间锁脚本核心源码详解（官方实现，按思考顺序重组）

> 本节介绍 CKB 时间锁脚本的官方实现原理与源码，适合有一定 Rust/CKB 基础的开发者。该脚本实现了基于区块头时间戳的锁定机制，常用于定期解锁、质押等场景。

## 核心概念

### 质押（Staking）

质押（Staking）是区块链生态中的一种资产管理机制，用户将数字资产锁定在智能合约中，以获得网络奖励、参与治理或保障协议安全。主要常见形式包括：

- **安全质押**：如以太坊 2.0，验证者需质押 32 ETH 参与共识，若作恶将被罚没，激励诚实行为，保障网络安全。
- **治理质押**：如 Compound，持有者质押 COMP 代币获得投票权，参与协议参数调整、资产上线等社区治理。
- **收益质押**：例如 ETH 2.0 验证者通过质押获得区块奖励和手续费分成，体现质押的经济激励作用。
- **流动性管理质押**：许多 DeFi 协议（如 Curve、Convex）要求用户锁定代币一段时间以获得更高收益，通过时间锁提升协议资金池的稳定性，抑制短期套利。

在 Nervos 生态中，Stable++ 协议作为首个基于 RGB++ 的类Liquidity**超额抵押稳定币系统**，将 BTC 和 CKB 作为抵押资产，允许用户通过质押这些资产铸造锚定美元的稳定币 RUSD。Stable++ 充分利用 RGB++ 的跨链能力和 CKB-VM 的图灵完备性，构建了安全、去中心化且高效的质押与清算机制：

- 用户可通过开设质押仓位，将 BTC 或 CKB 锁定为抵押品，系统根据抵押资产的实时价值允许借出一定数量的 RUSD，整个过程由智能合约自动执行，确保协议不可篡改和资产安全。
- Stable++ 的清算机制同样由合约自动管理，当抵押率低于安全阈值时任何人都可以自动触发清算，保障整体稳定性。
- 通过这种质押模式，Stable++ 不仅为 RGB++ 生态和比特币资产带来新的流动性和应用场景，也为用户提供了安全、透明、去中心化的稳定币借贷与治理平台，推动了跨链 DeFi 生态的繁荣发展。

### CKB质押原理：Script

在 CKB 中，质押机制主要通过为资产Cell编写**Script**实现。Lock Script 通过自定义锁定脚本实现资产锁定，只有满足特定条件（如时间、签名、多签等）才能解锁，适用于需要精细控制权限的场景。Type Script 则通过定义资产类型和转移规则，实现更细粒度的资产管理和质押机制，适用于需要资产类型化管理的场景。

本节聚焦的时间锁机制是质押最为常见的方式之一：通过 Lock Script 的参数（lock args）携带时间戳，并要求在交易的 `header_deps` 字段中提供区块头，脚本会遍历这些区块头，校验其 timestamp 是否达到解锁条件。只有当至少一个区块头的时间戳大于等于设定的时间限制时，资产才可解锁。

除此之外，由于RGB++实现了CKB和BTC的同构绑定，在CKB上可以实现BTC Time Lock，在CKB上解锁时验证BTC网络上的时间戳是否满足解锁条件，从而实现跨链时间锁定机制（我们将在下一章讲解）。

---

## 逐步实现ckb-script-time-lock

本节内容基于 [ckb-script-time-lock](https://github.com/Hanssen0/ckb-script-time-lock) 仓库，所有代码片段均为仓库原文，并附有永久链接。每一行/每段代码下方都配有详细中文解释，帮助你深入理解每个实现细节。

> 注意：本仓库使用的capsule工具已经弃用，请使用ckb-script-template工具生成合约。合约代码本身不受影响。

### 1. 合约入口与主流程

> [contracts/time_lock/src/main.rs#L6-L29](https://github.com/Hanssen0/ckb-script-time-lock/blob/master/contracts/time_lock/src/main.rs#L6-L29)

```rust
#![no_std]
#![no_main]
#![feature(lang_items)]
#![feature(alloc_error_handler)]
#![feature(panic_info_message)]

// define modules
mod entry;
mod error;

use ckb_std::default_alloc;

ckb_std::entry!(program_entry);
default_alloc!();

/// program entry
fn program_entry() -> i8 {
    // Call main function and return error code
    match entry::main() {
        Ok(_) => 0,
        Err(err) => err as i8,
    }
}
```

- 该文件由 ckb-script-template 工具生成，主要用于定义 Rust 语言项和模块，主逻辑见 entry.rs，错误类型见 error.rs。
- `#![no_std]`：不链接标准库，适用于区块链裸机环境。
- `ckb_std::entry!` 宏定义合约入口。
- `program_entry` 作为合约主入口，调用 `entry::main()`，并将错误类型转换为 i8 返回，CKB VM 能正确识别合约执行结果。
  - 这样设计的好处是，主入口 program_entry 只负责输出最后的i8结果（0为通过，其他均为错误代码），具体的错误处理和返回值都交给 entry::main()，因此 entry::main() 可以直接使用 Result 类型，便于在合约主逻辑中灵活处理各种错误情况，代码结构更清晰、可维护性更高。

---

### 2. 依赖导入与参数结构

#### 2.1 依赖导入

> [contracts/time_lock/src/entry.rs#L1-L15](https://github.com/Hanssen0/ckb-script-time-lock/blob/master/contracts/time_lock/src/entry.rs#L1-L15)

```rust
use core::result::Result;
use ckb_std::{
    ckb_constants::Source,
    ckb_types::{bytes::Bytes, prelude::*},
    debug,
    dynamic_loading_c_impl::CKBDLContext,
    high_level::{load_header, load_script, QueryIter},
};
use crate::error::Error;
use ckb_lib_secp256k1::LibSecp256k1;
```

- 导入 Rust 标准库的 Result 类型，用于错误处理。
- 导入 CKB-std 的常用类型和高阶 API：
  - `Source`：指定数据来源（如 HeaderDep、GroupInput 等）。
  - `Bytes`、`prelude::*`：字节处理和 CKB 类型转换。
  - `debug`：链上调试输出。
  - `CKBDLContext`：动态加载 secp256k1 C 实现。
  - `load_header`、`load_script`、`QueryIter`：高阶 API，用于加载区块头、脚本和遍历数据。
- 导入自定义错误类型 Error，便于后续统一错误处理。
- 导入本地 secp256k1 库的主类型 LibSecp256k1，用于签名校验和公钥恢复。
  - 虽然 Secp256k1 及其相关签名校验逻辑是 CKB 上最常用的默认 lock 之一，但本合约展示了如何将其作为一个模块嵌入到自定义 lock 脚本中。开发者可以在自定义的 lock 逻辑（如时间锁、多签、权限控制等）中复用这套签名校验流程，实现更复杂的资产管理和权限控制。这种组合方式极大提升了 CKB 脚本的灵活性和可扩展性。

### 2.2 在实际主函数中解析Lock Args和调用校验流程

> [contracts/time_lock/src/entry.rs#L16-L32](https://github.com/Hanssen0/ckb-script-time-lock/blob/master/contracts/time_lock/src/entry.rs#L16-L32)

```rust
pub fn main() -> Result<(), Error> {
    let script = load_script()?;
    let args: Bytes = script.args().unpack();

    if args.len() != 28 {
        return Err(Error::Encoding);
    }

    let pubkey = args.slice(0..20);
    let time_limit = args.slice(20..28);

    if !has_passed_time_limit(time_limit) {
        return Err(Error::TimeLimitNotReached);
    }

    // create a DL context with 128K buffer size
    let mut context = unsafe { CKBDLContext::<[u8; 128 * 1024]>::new() };
    let lib = LibSecp256k1::load(&mut context);

    test_validate_blake2b_sighash_all(&lib, &pubkey)
}
```

- 合约实际上的主入口，返回 Result<(), Error>，所有错误都用自定义 Error 类型。
- 调用 load_script 加载脚本，并解析 lock script 参数。
- 解析 lock script 参数，必须为 28 字节（20 字节 pubkey + 8 字节时间戳）。
- 前 20 字节为 pubkey_hash，后 8 字节为 time_limit。
- 调用 has_passed_time_limit 检查 header_deps 是否有区块头时间戳大于等于 time_limit。
- 创建 secp256k1 动态加载上下文，加载本地 secp256k1 库。
- 调用 test_validate_blake2b_sighash_all 进行签名校验。

---

## 3. 核心流程与校验逻辑

### 3.1 时间锁校验

> [contracts/time_lock/src/entry.rs#L37-L45](https://github.com/Hanssen0/ckb-script-time-lock/blob/master/contracts/time_lock/src/entry.rs#L37-L45)

```rust
fn has_passed_time_limit(time_limit: Bytes) -> bool {
    for header in QueryIter::new(load_header, Source::HeaderDep) {
        let timestamp = header.raw().timestamp().unpack().to_be_bytes();
        if time_limit.le(&timestamp) {
            return true;
        }
    }
    false
}
```

- 遍历所有 header_deps，取出区块头时间戳。
- 如果有任意一个区块头的 timestamp 大于等于 time_limit，则返回 true。
- 否则返回 false，表示未到解锁时间。

### 3.2 签名校验

> [contracts/time_lock/src/entry.rs#L44-L59](https://github.com/Hanssen0/ckb-script-time-lock/blob/master/contracts/time_lock/src/entry.rs#L44-L59)

```rust
fn test_validate_blake2b_sighash_all(
    lib: &LibSecp256k1,
    expected_pubkey_hash: &[u8],
) -> Result<(), Error> {
    let mut pubkey_hash = [0u8; 20];
    lib.validate_blake2b_sighash_all(&mut pubkey_hash)
        .map_err(|_err_code| {
            debug!("secp256k1 error {}", _err_code);
            Error::Secp256k1
        })?;

    // compare with expected pubkey_hash
    if &pubkey_hash[..] != expected_pubkey_hash {
        return Err(Error::WrongPubkey);
    }
    Ok(())
}
```

- 用 secp256k1 库校验签名（validate_blake2b_sighash_all），并输出调试信息。
- 校验通过后，将恢复出的 pubkey_hash 与 lock args 里的 pubkey_hash 比较。
- 不一致则返回 WrongPubkey 错误，否则校验通过。

---

## 4. 错误处理与类型定义

### 4.1 错误类型定义

> [contracts/time_lock/src/error.rs#L1-L11](https://github.com/Hanssen0/ckb-script-time-lock/blob/master/contracts/time_lock/src/error.rs#L1-L11)

```rust
use ckb_std::error::SysError;

/// Error
#[repr(i8)]
pub enum Error {
    IndexOutOfBound = 1,
    ItemMissing,
    LengthNotEnough,
    Encoding,
    Secp256k1,
    WrongPubkey,
    TimeLimitNotReached,
}
```

- 定义合约可能返回的错误类型，便于调试和定位问题。
- 每个错误类型都分配唯一的 i8 编号，便于 CKB VM 识别和调试。
- 错误类型涵盖索引越界、项缺失、长度不足、编码错误、secp256k1 错误、公钥不符、未到解锁时间等。

### 4.2 错误类型转换

> [contracts/time_lock/src/error.rs#L13-L24](https://github.com/Hanssen0/ckb-script-time-lock/blob/master/contracts/time_lock/src/error.rs#L13-L24)

```rust
impl From<SysError> for Error {
    fn from(err: SysError) -> Self {
        use SysError::*;
        match err {
            IndexOutOfBound => Self::IndexOutOfBound,
            ItemMissing => Self::ItemMissing,
            LengthNotEnough(_) => Self::LengthNotEnough,
            Encoding => Self::Encoding,
            Unknown(err_code) => panic!("unexpected sys error {}", err_code),
        }
    }
}
```

- 实现 `From<SysError>`，将底层系统错误统一转为自定义错误类型。
- 针对不同系统错误映射到不同的自定义错误，遇到未知错误直接 panic。
- 这样主流程只需 `return Err(Error::XXX)`，入口函数统一转换为 i8 返回。

## 总结

本节详细解析了 CKB 官方时间锁脚本（ckb-script-time-lock）的实现原理与源码结构。该合约通过 lock args 携带时间戳参数，并要求在 header_deps 中提供区块头，校验区块头的 timestamp 是否达到解锁条件，从而实现了无需依赖 Since 字段的灵活时间锁定机制。签名校验部分则复用了 CKB 默认的 secp256k1_blake160_sighash_all 逻辑，并以模块化方式嵌入到自定义 lock 中，极大提升了脚本的可组合性和扩展性。
这种设计不仅适用于常见的定期解锁、质押等场景，也为开发者实现多签、权限控制、跨链时间锁等复杂业务逻辑提供了范例。通过灵活组合 CKB-VM 的能力和标准加密库，开发者可以构建出安全、去中心化且高度可定制的资产管理合约，推动 CKB 生态的创新与发展。

CKB 所采用的 Cell/Script 机制，突破了传统 UTXO 区块链只能实现简单支付和多签的局限。通过在每个 Cell 上绑定自定义的 Lock Script 和 Type Script，开发者可以实现任意复杂的状态验证和资产管理逻辑。配合 CKB-VM 的图灵完备性，这种模型不仅支持时间锁、权限控制、跨链验证等高级功能，还能实现类似账户模型链上的复杂合约和去中心化应用。CKB 由此成为 UTXO 类区块链中少有的具备完整链上编程能力的平台，为创新型 DeFi、NFT、跨链协议等应用提供了坚实基础。
