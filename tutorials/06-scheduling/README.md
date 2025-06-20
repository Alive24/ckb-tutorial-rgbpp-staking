# Phase 6: Unlocking the Future - Implementing Scheduled and Automated CKB Transactions

## Introduction

In the previous chapters, we successfully built a Staking dApp where users can actively initiate staking and withdrawal transactions. However, in many real-world scenarios, what we need is to **automatically execute transactions at a specific point in the future**, without requiring the user to be online. For example, in our Staking contract, the ideal scenario is for the system to automatically trigger an unlock transaction to return the assets to the user once the lock period has ended.

This is where **Scheduled Transactions** come into play.

This chapter will introduce you to `ccc-schedule-send`, a powerful backend service built on Nest.js, specifically designed for scheduling and automatically sending transactions on the CKB network. We will learn how to configure and use it to automate the unlocking process of our Staking dApp.

**Learning Objectives:**

1.  Understand the necessity of a backend service for CKB dApps.
2.  Grasp the basic concepts and working principles of `ccc-schedule-send`.
3.  Learn how to configure `ccc-schedule-send` to monitor and automatically process Staking Cells that are ready to be unlocked.
4.  Integrate `ccc-schedule-send` into our Staking dApp workflow.

## Step 1: Understanding the Role of a Backend Service

So far, our dApp has been running entirely on the frontend, with all operations initiated by the user. This model is simple and direct but has its limitations:

-   **Reliance on User Action**: If a user forgets or is unable to manually initiate the transaction at the unlock time, the assets will remain locked.
-   **Inability to Handle Complex Logic**: For complex applications that require continuous monitoring of on-chain state and triggering transactions based on specific conditions (like oracles, DAO proposal execution, etc.), a purely frontend architecture falls short.

A backend service like `ccc-schedule-send` (sometimes called a "keeper" or "bot") solves these problems. It acts like a tireless robot, continuously monitoring the blockchain and executing necessary on-chain operations on our behalf when preset conditions are met.

## Step 2: Getting Started with `ccc-schedule-send`

`ccc-schedule-send` is an open-source, independently deployable scheduling service. Let's get it up and running.

1.  **Clone the Project**
    First, clone the `ccc-schedule-send` repository from GitHub.

    ```bash
    git clone https://github.com/ckb-devrel/ccc-schedule-send.git
    cd ccc-schedule-send
    ```

2.  **Install Dependencies**
    The project uses pnpm as its package manager.

    ```bash
    pnpm install
    ```

3.  **Configure Environment Variables**
    Most of the service's configuration is managed through environment variables. Create a new `.env` file by copying the `.env.example` file.

    ```bash
    cp .env.example .env
    ```

    Then, edit the `.env` file to fill in the most critical information:

    ```dotenv
    # Private key of the bot account used to send transactions
    BOT_PRIVATE_KEY=0x...your_private_key...

    # CKB Testnet RPC and Indexer addresses
    CKB_RPC_URL=https://testnet.ckb.dev/rpc
    CKB_INDEXER_URL=https://testnet.ckb.dev/indexer
    ```
    > **Security Tip**: The `BOT_PRIVATE_KEY` is highly sensitive information. The account it controls needs to have enough CKB to cover transaction fees. In a production environment, please secure this private key properly and use an account with the minimum necessary permissions.

4.  **Start the Service**
    Once everything is set up, start the service in development mode:

    ```bash
    pnpm run start:dev
    ```

    If you see the Nest.js startup log, it means the service is successfully running on your local port 3000.

## Step 3: Defining Our Scheduled Task

The core of `ccc-schedule-send` is "Schedules". Each task defines what the service needs to do (What), when to do it (When), and how to do it (How).

For our Staking dApp, the task is: **Periodically scan all locked Staking Cells, and if a cell's lock period has expired, automatically build and send an unlock transaction.**

`ccc-schedule-send` allows us to implement this logic by writing a custom "Schedule Provider".

1.  **Create a Schedule Provider**
    In the `src/schedules` directory, we can create a new file `staking-unlock.schedule.ts`. This file will export a class that implements the `ScheduleProvider` interface.

    ```typescript
    // src/schedules/staking-unlock.schedule.ts
    import { Injectable } from '@nestjs/common';
    import { Cron, CronExpression } from '@nestjs/schedule';
    import { CKBService } from '../ckb/ckb.service';
    import { ScheduleProvider } from './schedule.provider';
    import { packed, OutPoint, Script } from '@ckb-lumos/base';

    @Injectable()
    export class StakingUnlockSchedule implements ScheduleProvider {
      // The unique name of the task
      name = 'staking-unlock';

      constructor(private readonly ckbService: CKBService) {}

      // Use the @Cron decorator to define the execution frequency, here it's every minute
      @Cron(CronExpression.EVERY_MINUTE)
      async run() {
        console.log('Running staking-unlock schedule...');

        // 1. Define our Staking Lock Script template
        const timeLockScriptTemplate: Script = {
          codeHash: 'YOUR_TIME_LOCK_SCRIPT_CODE_HASH', // Replace with the codeHash you deployed in Step 0
          hashType: 'type',
          args: this.ckbService.botAddress.getLockHash(), // Query for all staking cells belonging to the bot account
        };

        // 2. Find all unlockable cells
        const cells = await this.ckbService.collectCells(timeLockScriptTemplate);
        const now = Math.floor(Date.now() / 1000);

        const unlockableCells = cells.filter(cell => {
          // The first 10 bytes of lock.args (0x + 8-byte timestamp) are the unlock time
          const lockUntilTimestamp = parseInt(cell.cellOutput.lock.args.slice(0, 18), 16);
          return now >= lockUntilTimestamp;
        });

        if (unlockableCells.length === 0) {
          console.log('No unlockable cells found.');
          return;
        }

        console.log(`Found ${unlockableCells.length} unlockable cells.`);

        // 3. Build and send the unlock transaction
        try {
          const txHash = await this.ckbService.sendUnlockTransaction(unlockableCells);
          console.log(`Unlock transaction sent: ${txHash}`);
        } catch (error) {
          console.error('Failed to send unlock transaction:', error);
        }
      }
    }
    ```

2.  **Implement Transaction Building Logic**
    In `ckb.service.ts`, we need to add the specific implementation for sending the unlock transaction.

    ```typescript
    // src/ckb/ckb.service.ts
    // ... (omitting existing code)

    async sendUnlockTransaction(cellsToUnlock: LiveCell[]): Promise<string> {
      // Get the user's original lock script hash, stored in the latter half of the time lock args
      const originalLockHash = cellsToUnlock[0].cellOutput.lock.args.slice(18);
      const userLockScript = {
        codeHash: 'YOUR_DEFAULT_LOCK_SCRIPT_CODE_HASH', // e.g., SECP256K1_BLAKE160
        hashType: 'type',
        args: `0x${originalLockHash}`,
      };

      const outputs = cellsToUnlock.map(cell => ({
        ...cell.cellOutput,
        lock: userLockScript, // Restore the lock script to the user's original lock
      }));

      const tx = {
        inputs: cellsToUnlock.map(cell => ({
          previousOutput: cell.outPoint,
          since: '0x0', // since must be 0 to unlock
        })),
        outputs,
        // ccc-sdk-js or lumos will help us handle the change output
      };

      // Use the existing signing and sending logic in ckbService
      return this.sendTransaction(tx);
    }
    ```

3.  **Register the Provider**
    Finally, don't forget to register our new `StakingUnlockSchedule` in `src/schedules/schedule.module.ts`.

    ```typescript
    // src/schedules/schedule.module.ts
    import { Module } from '@nestjs/common';
    import { StakingUnlockSchedule } from './staking-unlock.schedule';
    // ...

    @Module({
      providers: [StakingUnlockSchedule, /* ... other providers */],
      // ...
    })
    export class ScheduleModule {}
    ```

## Step 4: Summary and Future Outlook

Congratulations! You have successfully added a powerful automated backend service to our Staking dApp.

Through this chapter, we have unlocked a new dimension of CKB dApp development—backend automation. `ccc-schedule-send` can be used for much more than just unlocking staked assets. Its applications are extensive, including:

-   **DAO Governance**: Automatically execute proposals after the voting period ends.
-   **On-Chain Gaming**: Update game states or distribute rewards as time progresses.
-   **Asset Management**: Execute dollar-cost averaging strategies or automated portfolio rebalancing.

This automation service is a crucial step for you to transition from a simple dApp developer to building complex, robust decentralized systems. In the upcoming chapters, we will continue to explore more advanced topics. Stay tuned! 