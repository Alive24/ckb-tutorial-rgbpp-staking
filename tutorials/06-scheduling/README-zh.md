# 阶段 6: 解锁未来 - 实现 CKB 交易的计划任务与自动化

## 引言

在之前的章节中，我们成功构建了一个 Staking dApp，用户可以主动发起质押和赎回交易。然而，在许多真实世界的场景中，我们需要的是**在未来的特定时间点自动执行**交易，而无需用户届时在线操作。例如，在我们的 Staking 合约中，当锁定期结束后，理想的情况是系统能自动触发解锁交易，将资产归还给用户。

这就是**计划任务 (Scheduled Transactions)** 的用武之地。

本章节将向你介绍 `ccc-schedule-send`，一个基于 Nest.js 构建的强大后端服务，专门用于在 CKB 网络上调度和自动发送交易。我们将学习如何配置和使用它来自动完成 Staking dApp 的解锁流程。

**学习目标:**

1. 理解 CKB dApp 后端服务的必要性。
2. 掌握 `ccc-schedule-send` 的基本概念和工作原理。
3. 学会如何配置 `ccc-schedule-send` 以监控并自动处理待解锁的 Staking Cell。
4. 将 `ccc-schedule-send` 集成到我们的 Staking dApp 工作流中。

## 步骤一：理解后端服务的角色

到目前为止，我们的 dApp 完全运行在前端，所有操作都由用户发起。这种模式简单直接，但有其局限性：

- **依赖用户操作**：如果用户忘记或无法在解锁时间点手动发起交易，资产将一直被锁定。
- **无法处理复杂逻辑**：对于需要持续监控链上状态并根据特定条件触发交易的复杂应用（如预言机、DAO 投票执行等），纯前端架构力不从心。

`ccc-schedule-send` 这样的后端服务（有时也称为 "keeper" 或 "bot"）解决了这些问题。它像一个不知疲倦的机器人，持续监控区块链，并在预设条件满足时，代表我们执行必要的链上操作。

## 步骤二：`ccc-schedule-send` 快速上手

`ccc-schedule-send` 是一个开源的、可独立部署的调度服务。让我们先将它运行起来。

1. **克隆项目**
    首先，从 GitHub 克隆 `ccc-schedule-send` 的代码仓库。

    ```bash
    git clone https://github.com/ckb-devrel/ccc-schedule-send.git
    cd ccc-schedule-send
    ```

2. **安装依赖**
    项目使用 pnpm 作为包管理器。

    ```bash
    pnpm install
    ```

3. **配置环境变量**
    服务的大部分配置通过环境变量管理。复制 `.env.example` 文件创建一个新的 `.env` 文件。

    ```bash
    cp .env.example .env
    ```

    然后，编辑 `.env` 文件，填入最关键的信息：

    ```dotenv
    # 用于发送交易的机器人账户的私钥
    BOT_PRIVATE_KEY=0x...your_private_key...

    # CKB 测试网的 RPC 和 Indexer 地址
    CKB_RPC_URL=https://testnet.ckb.dev/rpc
    CKB_INDEXER_URL=https://testnet.ckb.dev/indexer
    ```

    > **安全提示**: `BOT_PRIVATE_KEY` 是一个非常敏感的信息，它控制的账户需要有足够的 CKB 来支付交易手续费。在生产环境中，请务 C 善保管此私钥，并使用权限最小化的账户。

4. **启动服务**
    一切就绪后，以开发模式启动服务：

    ```bash
    pnpm run start:dev
    ```

    如果看到 Nest.js 的启动日志，说明服务已经成功运行在你本地的 3000 端口。

## 步骤三：定义我们的计划任务

`ccc-schedule-send` 的核心是 "计划任务" (Schedules)。每个任务定义了服务需要做什么 (What)、何时做 (When) 以及如何做 (How)。

对于我们的 Staking dApp，任务是：**定期扫描所有处于锁定状态的 Staking Cell，如果某个 Cell 的锁定期已过，就自动构建一笔解锁交易并发送**。

`ccc-schedule-send` 允许我们通过编写自定义的 "Schedule Provider" 来实现这个逻辑。

1. **创建 Schedule Provider**
    在 `src/schedules` 目录下，我们可以创建一个新的文件 `staking-unlock.schedule.ts`。这个文件将导出一个类，实现 `ScheduleProvider` 接口。

    ```typescript
    // src/schedules/staking-unlock.schedule.ts
    import { Injectable } from "@nestjs/common";
    import { Cron, CronExpression } from "@nestjs/schedule";
    import { CKBService } from "../ckb/ckb.service";
    import { ScheduleProvider } from "./schedule.provider";
    import { packed, OutPoint, Script } from "@ckb-lumos/base";

    @Injectable()
    export class StakingUnlockSchedule implements ScheduleProvider {
      // 任务的唯一名称
      name = "staking-unlock";

      constructor(private readonly ckbService: CKBService) {}

      // 使用 @Cron 装饰器定义执行频率，这里是每分钟执行一次
      @Cron(CronExpression.EVERY_MINUTE)
      async run() {
        console.log("Running staking-unlock schedule...");

        // 1. 定义我们的 Staking Lock Script 模板
        const timeLockScriptTemplate: Script = {
          codeHash: "YOUR_TIME_LOCK_SCRIPT_CODE_HASH", // 替换成你在步骤 〇 中部署的 codeHash
          hashType: "type",
          args: this.ckbService.botAddress.getLockHash(), // 查询所有属于机器人账户的质押 cell
        };

        // 2. 查找所有可解锁的 Cells
        const cells = await this.ckbService.collectCells(
          timeLockScriptTemplate
        );
        const now = Math.floor(Date.now() / 1000);

        const unlockableCells = cells.filter((cell) => {
          // lock.args 的前 10 个字节 (0x + 8 字节时间戳) 是解锁时间
          const lockUntilTimestamp = parseInt(
            cell.cellOutput.lock.args.slice(0, 18),
            16
          );
          return now >= lockUntilTimestamp;
        });

        if (unlockableCells.length === 0) {
          console.log("No unlockable cells found.");
          return;
        }

        console.log(`Found ${unlockableCells.length} unlockable cells.`);

        // 3. 构建并发送解锁交易
        try {
          const txHash = await this.ckbService.sendUnlockTransaction(
            unlockableCells
          );
          console.log(`Unlock transaction sent: ${txHash}`);
        } catch (error) {
          console.error("Failed to send unlock transaction:", error);
        }
      }
    }
    ```

2. **实现交易构建逻辑**
    在 `ckb.service.ts` 中，我们需要添加发送解锁交易的具体实现。

    ```typescript
    // src/ckb/ckb.service.ts
    // ... (省略已有代码)

    async sendUnlockTransaction(cellsToUnlock: LiveCell[]): Promise<string> {
      // 获取用户原始的 lock script hash，它存储在 time lock args 的后半部分
      const originalLockHash = cellsToUnlock[0].cellOutput.lock.args.slice(18);
      const userLockScript = {
        codeHash: 'YOUR_DEFAULT_LOCK_SCRIPT_CODE_HASH', // 例如 SECP256K1_BLAKE160
        hashType: 'type',
        args: `0x${originalLockHash}`,
      };

      const outputs = cellsToUnlock.map(cell => ({
        ...cell.cellOutput,
        lock: userLockScript, // 将 lock script 恢复为用户原始的 lock
      }));

      const tx = {
        inputs: cellsToUnlock.map(cell => ({
          previousOutput: cell.outPoint,
          since: '0x0', // since 必须为 0 才能解锁
        })),
        outputs,
        // ccc-sdk-js 或 lumos 会帮助我们处理找零
      };

      // 使用 ckbService 中已有的签名和发送逻辑
      return this.sendTransaction(tx);
    }
    ```

3. **注册 Provider**
    最后，不要忘记在 `src/schedules/schedule.module.ts` 中注册我们新的 `StakingUnlockSchedule`。

    ```typescript
    // src/schedules/schedule.module.ts
    import { Module } from "@nestjs/common";
    import { StakingUnlockSchedule } from "./staking-unlock.schedule";
    // ...

    @Module({
      providers: [StakingUnlockSchedule /* ... 其他 providers */],
      // ...
    })
    export class ScheduleModule {}
    ```

## 步骤四：总结与展望

恭喜你！你已经成功地为我们的 Staking dApp 添加了一个强大的自动化后台服务。

通过本章的学习，我们解锁了 CKB dApp 开发的一个全新维度——后端自动化。`ccc-schedule-send` 不仅可以用于解锁 Staking 资产，它的应用场景非常广泛，例如：

- **DAO 治理**：在投票期结束后自动执行提案。
- **链上游戏**：根据时间流逝更新游戏状态或分发奖励。
- **资产管理**：执行定投策略或自动化的投资组合再平衡。

这个自动化服务是你从一个简单的 dApp 开发者迈向构建复杂、健壮的去中心化系统的关键一步。在后续章节中，我们将继续探索更多高级主题，敬请期待！
