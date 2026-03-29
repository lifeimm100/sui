# Transaction Driver 写路径总览

## 范围

这份说明把下面三个文件放在一起看：

- `crates/sui-core/src/transaction_driver/mod.rs`
- `crates/sui-core/src/transaction_driver/transaction_submitter.rs`
- `crates/sui-core/src/transaction_driver/effects_certifier.rs`

目标不是逐函数拆细节，而是把它们看成同一个分层子系统。

## 一句话总结

这三个文件一起实现了 fullnode 写路径中“面向 validator 的执行推进层”：

- `mod.rs` 是最外层驱动器，负责重试策略、超时策略、指标和 reconfig 生命周期。
- `transaction_submitter.rs` 是提交层，负责尽快把交易送进某个可用 validator 的处理路径。
- `effects_certifier.rs` 是认证层，负责把 validator 观察到的执行结果提升成 quorum-certified finalized effects。

换句话说：

`TransactionDriver` 决定**要不要继续尝试**，  
`TransactionSubmitter` 决定**往哪里提交、怎么提交**，  
`EffectsCertifier` 决定**当前执行结果是否已经足够可信、可以作为最终结果返回**。

## 分层职责

### 1. `transaction_driver/mod.rs`：外层驱动层

`TransactionDriver` 是这套子系统的总控层。

它自己并不负责和 validator 打交道的细节，而是负责一次完整尝试的生命周期管理：

- 根据 gas price 决定提交强度
- 对 submission-retriable 错误进行整轮重试
- 对永久错误立即停止
- 施加整体超时
- 在 epoch 切换时替换 `AuthorityAggregator`
- 记录端到端指标和 validator 反馈

这一层的核心抽象是：

> 持续把一笔交易往 finalized effects 推，直到成功、永久失败、或者超时。

所以 `mod.rs` 管的是策略，不是底层传输细节。

### 2. `transaction_submitter.rs`：快速提交层

`TransactionSubmitter` 的目标是尽快让交易进入 validator 的处理路径。

它的成功条件刻意设得比较弱：

- 只要有一个 validator 接受提交，并返回非 `Rejected` 的 `SubmitTxResult`，这一层就算成功

它不负责 finality，也不负责认证 effects。

它主要解决的是：

- 怎么选 validator
- 起手并发打给多少个 validator
- 目标慢的时候何时补发 backup request
- 什么时候可以认为“这轮提交已经成功”
- 如果目标都试完了，错误该怎么聚合

这一层的核心优化目标是延迟和可达性。

### 3. `effects_certifier.rs`：结果认证层

`EffectsCertifier` 接过提交层的阶段性成功，把它升级成一个安全的最终结果。

它负责：

- 收集 validator 对 effects digest 的确认
- 判断是否有某个 digest 获得 quorum 支持
- 获取一份完整的执行结果明细
- 验证这份完整 effects 的 digest 和 quorum-certified digest 一致
- 只有在两者同时成立时，才返回 finalized response

这一层的核心优化目标是安全性和语义正确性。

它真正解决的问题是：

> 不是“某个 validator 说执行了”，而是“网络已经对某个执行结果形成了可认证的一致意见”。

## 高层调用链

在这三个文件内部，端到端调用链是：

```text
TransactionDriver::drive_transaction
  -> TransactionDriver::drive_transaction_once
    -> TransactionSubmitter::submit_transaction
      -> TransactionSubmitter::submit_transaction_once
    -> EffectsCertifier::get_certified_finalized_effects
      -> EffectsCertifier::wait_for_acknowledgments
      -> EffectsCertifier::get_full_effects_with_fallback
        -> EffectsCertifier::get_full_effects
      -> EffectsCertifier::get_quorum_transaction_response
  -> QuorumTransactionResponse
```

如果换成职责链的视角，可以概括成：

```text
外层重试与生命周期管理
  -> 快速把交易送进某个 validator
  -> 对 effects digest 做 quorum 认证
  -> 拉取与认证 digest 匹配的完整 effects
  -> 组装最终 finalized response
```

## 每一层分别增加了什么能力

### `mod.rs` 增加的是整轮尝试管理

`TransactionDriver::drive_transaction` 用外层循环包住整条链路。

从策略上看，它做的是：

1. 计算提交阶段的放大系数。
2. 执行一次完整尝试。
3. 成功则立即返回。
4. 永久错误则立即停止。
5. 可重试错误则退避后再来一轮。
6. 整体超时则带着最近一次 retriable 上下文返回。

所以这一层重试的单位不是单个 RPC，而是整轮组合尝试：

`提交交易 -> 认证 effects -> 返回 finalized response`

这使得 `mod.rs` 成为整个子系统的 liveness policy 所在层。

### `transaction_submitter.rs` 增加的是目标选择和渐进式扇出

提交层并不是一开始就把请求广播给所有 validator。

它的策略是：

- 先根据 client monitor 选择较优节点
- 起手只发到有限数量的 validator
- 如果这些请求迟迟不回，再逐步补发 backup request
- 扩散是渐进的，而不是一次性全量广播

这一层最关键的设计点是：

- 成功是局部成功
- 只要有一个 validator 接住交易，就足够把控制权交给认证层

这样可以把提交阶段做得更快、更省。

### `effects_certifier.rs` 增加的是 quorum 语义

一旦提交阶段在某处成功，认证层就开始工作。

它会并行推进两件事：

- 收集足够多的 validator 确认，以认证某个 effects digest
- 从某个 validator 拉取完整 effects，必要时做 fallback

这里最关键的规则是：

- 完整 effects 只有在其 digest 和 quorum-certified digest 一致时才算可信

这使整个设计形成了清晰分工：

- 提交层负责尽快找到一个可能正确的来源
- 认证层负责证明“正确的 digest 到底是什么”
- full effects 获取只负责拿到那份与认证结果一致的具体载荷

这正是这套设计最核心的安全性质。

## 为什么要拆成三层

这三个文件之所以分开，是因为它们解决的是三类不同问题：

- `mod.rs` 解决重试和生命周期管理
- `transaction_submitter.rs` 解决 validator 可达性和低延迟提交
- `effects_certifier.rs` 解决 finality 和结果正确性

如果把这些职责全揉在一个大函数里，会立刻混乱：

- 重试策略会和 validator 选择缠在一起
- 延迟优化会和安全检查缠在一起
- reconfig 与 metrics 会和 digest 认证逻辑缠在一起

现在这种拆法的好处是很明确的：

- 最外层只关心活性和策略
- 中间层只关心把请求快速送进去
- 最底层只关心 quorum 和 correctness

## 失败语义的高层理解

这套子系统从高层看，区分三类结果：

- 成功：拿到了 quorum-certified finalized effects
- 可重试失败：这一轮没有完成 finality，但重新来一轮还有机会成功
- 永久失败：网络返回的信息已经表明继续沿同一路径重试没有意义

如果用一句问题来概括三层分别在回答什么：

- `transaction_submitter.rs` 回答的是：交易有没有成功进入 validator 处理路径？
- `effects_certifier.rs` 回答的是：网络有没有对某个执行结果形成可认证的一致意见？
- `mod.rs` 回答的是：如果还没有，我们是否应该整轮再试一次？

## 它在更大写路径里的位置

这三个文件并不是 fullnode 写路径的全部。

它们位于 orchestrator 之下：

- orchestrator 处理的是 fullnode 本地语义，比如去重、pending WAL 恢复、等待本地执行完成
- transaction driver 处理的是面向 validator 的提交与认证

所以更完整的心智模型是：

```text
TransactionOrchestrator
  -> TransactionDriver
    -> TransactionSubmitter
    -> EffectsCertifier
```

可以把它理解成：

- orchestrator 是 fullnode 侧的总协调器
- 这里这三个文件组成的是 validator 侧的 finality 推进引擎

## 实用的阅读方式

如果你在查 bug 或性能问题，可以先用这一层分法定位：

- 如果问题是“为什么停止重试了”或“为什么超时了”，先看 `mod.rs`
- 如果问题是“为什么选了这个 validator”或“为什么发了 backup request”，先看 `transaction_submitter.rs`
- 如果问题是“为什么认证失败了”或“为什么出现 forked execution”，先看 `effects_certifier.rs`

这样通常能先找对层，再决定要不要继续往函数级细节里钻。
