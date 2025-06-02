# 编写 《从0到1构建RGB++资产锁仓应用》图文教程

Writing the "From 0 to 1: Building an RGB++ Asset Lock-up Application" Illustrated Tutorial

## 项目说明

这是 《从0到1构建》系列教程之一，通过详细的图文教程，希望让开发者能跟着教程一步一步地实现该应用。我们将在之后的线上活动、线下活动展示这些应用案例，作为吸引开发者加入 CKB 社区的抓手。

## 项目需求

资产的质押锁仓功能用处很多，比如质押生息(锁仓挖矿)，代币经济稳定、参与项目DAO治理投票权、流动性挖矿激励、激励长期持有等，而目前没有样板给开发者参考，使得RGB++ 资产目前的使用场景大大受限。使用create-ccc-app脚手架实现一个简易的dapp应用，用户通过ccc连接上钱包，可以：

- 输入资产的typeHash，加载钱包里该资产的余额（为了方便起见，不展示所有xudt资产）
- 输入一定数额的资产进行锁仓
- 选择或者输入锁仓的周期
- 看到锁仓到期时间（锁仓到期后是不是无需操作资产自动变为可用状态？）

需注意的点：

- 锁仓的时候需区分CKB链的余额和Bitcoin链的余额

---
English Translation

## Project Description

This is one of the "From 0 to 1" series tutorials. Through detailed illustrated instructions, we hope developers can follow along step by step to implement this application. We will showcase these application cases in future online and offline events as a way to attract developers to join the CKB community.

## Project Requirements

The asset staking and lock-up function has many uses, such as earning interest through staking (lock-up mining), stabilizing token economics, participating in project DAO governance voting rights, liquidity mining incentives, and encouraging long-term holding. Currently, there is no template for developers to refer to, which greatly limits the use cases of RGB++ assets. Use the create-ccc-app scaffolding to implement a simple dapp application, where users can connect their wallet via ccc and:

- Enter the asset's typeHash to load the balance of that asset in the wallet (for convenience, do not display all xudt assets)
- Enter a certain amount of assets to lock up
- Select or enter the lock-up period
- See the lock-up expiration time (After the lock-up expires, does the asset automatically become available without further action?)

Points to note:

- When locking up, it is necessary to distinguish between the balance on the CKB chain and the balance on the Bitcoin chain