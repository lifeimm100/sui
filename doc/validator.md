# Validator 如何处理 `TransactionSubmitter` 提交的交易

## 范围

这份说明回答的是：

> 当 `TransactionSubmitter` 把一笔 `SubmitTxRequest` 发给某个 validator 后，  
> 这个 validator 在服务端到底会怎么处理？

这里关注的主要是 validator 侧这条链路：

- `crates/sui-core/src/authority_server.rs`
- `crates/sui-core/src/authority.rs`
- `crates/sui-core/src/consensus_adapter.rs`
- `crates/sui-core/src/consensus_handler.rs`

## 一句话总结

validator 收到 `SubmitTxRequest` 后，做的不是“在这个 RPC 里立刻把交易完整执行完”，而是：

1. 先做本地校验与快速判定。
2. 已执行过的交易直接返回执行结果。
3. 其余可接受交易包装成 consensus message，提交到本地 consensus 管线。
4. 等交易被 consensus 排序后，再进入执行调度器，最终产出 effects。

所以 `submit_transaction` 这一跳的真实语义更接近：

> validator 是否接受这笔交易，并愿意把它推进到自己的执行管线里。

而不是：

> validator 是否已经在这次 RPC 返回前把交易完整 finalize 了。

## 高层调用链

把调用链压缩后，可以画成：

```text
TransactionSubmitter
  -> AuthorityClient::submit_transaction
    -> ValidatorService::handle_submit_transaction
      -> handle_submit_transaction_inner
        -> 基础校验 / 已执行检查 / handle_vote_transaction
        -> handle_submit_to_consensus_for_position
          -> ConsensusAdapter::submit_batch
            -> consensus sequenced
              -> ConsensusHandler
                -> 转成 VerifiedExecutableTransaction
                -> enqueue 到 execution scheduler
                  -> 执行并写入 effects
```

然后 `EffectsCertifier` 再通过 `wait_for_effects` 去等这些 effects 可见，并把它们提升成 quorum-certified finalized response。

## 服务端收到请求后的处理阶段

### 1. 先做反序列化、基础合法性和过载检查

`ValidatorService::handle_submit_transaction_inner` 收到请求后，会先对每笔交易做这些检查：

- 反序列化 `Transaction`
- `validity_check`
- 请求类型和 batch 大小限制检查
- 当前系统过载检查
- 签名和 alias 校验

如果这些步骤里出现明显错误：

- 有些错误会直接让整个 RPC 失败
- 有些错误会在对应位置返回 `SubmitTxResult::Rejected`

这一阶段的目标是尽量早拦住坏请求和过载请求。

### 2. 先检查“这笔交易是不是已经执行过了”

validator 在做输入对象检查前，会先看本地缓存里有没有这笔交易的 effects。

这里有两个重要分支：

- 如果当前 epoch 本地已经有 executed effects，则直接返回 `SubmitTxResult::Executed`
- 如果它是在上一 epoch 执行过的，则返回 `SubmitTxResult::Rejected`

所以 submit RPC 并不总是“排队提交”。

对已经执行过的交易，它可以直接变成一次结果读取。

### 3. 等 fastpath 依赖对象，再做 validator 侧准入检查

如果交易还没执行过，validator 会继续做两件事：

- 等 fastpath dependency objects 就绪
- 调 `AuthorityState::handle_vote_transaction`

`handle_vote_transaction` 的职责不是执行交易，而是确认这笔交易当前是否应该被 validator 接受。

它会做：

- epoch reconfig 窗口检查
- 最近是否已经 finalized / executed 的检查
- deny checks
- owned object live version 校验

如果这里失败，服务端会返回 `SubmitTxResult::Rejected`。

所以这一层回答的是：

> 这笔交易在 validator 看来，现在是否值得继续往执行路径推进？

### 4. 校验通过后，交易会被包装成 consensus message

一旦 `handle_vote_transaction` 通过，validator 并不会在这个 RPC 里直接执行交易。

它会先收集并附加一些 claims，例如：

- immutable object claims
- address alias claims

然后把交易包装成 `ConsensusTransaction::UserTransactionV2`，再交给：

- `handle_submit_to_consensus_for_position`
- `ConsensusAdapter::submit_batch`

这一步拿到的是一个 `consensus_position`。

因此这时服务端通常返回的是：

- `SubmitTxResult::Submitted { consensus_position }`

它的语义是：

> 这笔交易已经被 validator 接受，并成功送进了本地 consensus 提交流水线。

不是：

> 这笔交易已经执行完成。

## 提交到 consensus 之后发生什么

### 1. consensus adapter 负责把消息真正送进 consensus

`ConsensusAdapter` 会做提交时机控制、重试和状态等待。

它要解决的问题包括：

- 当前是否轮到本 authority 提交
- 如果 block 被 garbage collected，是否需要重提
- 这笔消息是否已经通过 consensus 或 checkpoint 被观察到

所以从 validator 自己的视角看，`Submitted` 也只是说明：

- 交易已经进入了这台 validator 的 consensus 推进路径

还不等于：

- 交易已经被 consensus 排序
- 或者已经被本地执行

### 2. consensus sequenced 后，`ConsensusHandler` 才把它变成可执行事务

当 consensus 真正产出排序结果后，`ConsensusHandler` 会处理这些 sequenced consensus transactions。

对用户交易来说，它会把：

- `UserTransaction`
- `UserTransactionV2`

转换成：

- `VerifiedExecutableTransaction`

然后再进入后续的调度逻辑。

这一层还会做一些 post-consensus 处理，例如：

- 合并和重排事务
- 拥塞控制下的 deferral / cancellation
- 对 randomness 相关事务做单独处理

所以 transaction 真正变成“可执行项”，是在 consensus 之后，而不是 submit RPC 返回时。

### 3. 最终由 execution scheduler 排队执行

经过 consensus handler 的整理后，交易会被 enqueue 到 execution scheduler。

之后才会发生真正的：

- 输入对象读取
- Move 执行
- effects 生成
- effects / events / object changes 落本地状态

这时这台 validator 才算真的拥有了这笔交易的本地执行结果。

## `SubmitTxResult` 三种结果分别表示什么

validator 在 submit RPC 上可能返回三类结果：

### 1. `Rejected`

表示 validator 明确拒绝继续处理这笔交易。

常见原因包括：

- 输入或签名非法
- 已在上一 epoch 执行
- 当前 epoch / reconfig 状态不允许接受
- 对象版本或 deny checks 失败
- 系统过载拒绝

对 `TransactionSubmitter` 来说，这属于失败。

### 2. `Executed`

表示 validator 本地已经有这笔交易的执行结果，可以直接把 effects digest 和 details 返回出来。

这是一种“命中现成结果”的成功，不需要再走“先提交到 consensus 再等待”的路径。

### 3. `Submitted`

表示 validator 接受了这笔交易，并且已经把它送进本地 consensus 提交流水线，同时返回一个 `consensus_position`。

这只是“成功接单并排入推进路径”，不是“执行已经结束”。

对 `TransactionSubmitter` 来说，`Submitted` 和 `Executed` 都算提交成功。

## 这和 `EffectsCertifier` 的关系

`TransactionSubmitter` 在看到某个 validator 返回非 `Rejected` 结果后，就认为提交阶段已经成功。

但这时交易通常还没有形成最终结果。

后续还需要 `EffectsCertifier` 去做两件事：

- 根据 `consensus_position` 或已知 digest 去 `wait_for_effects`
- 收集足够多 validator 的确认，把结果提升成 quorum-certified finalized effects

所以职责边界是：

- validator submit RPC 负责“接受并推进”
- `wait_for_effects` 负责“等这台 validator 的 effects 变得可见”
- `EffectsCertifier` 负责“把局部观察提升成网络级 finality”

## 最实用的理解方式

如果只记一个结论，建议记这个：

> `TransactionSubmitter` 把交易打到某个 validator 后，  
> validator 首先做的是本地准入和投递到 consensus，  
> 真正的排序、执行和 effects 产出是在这之后异步发生的。

换句话说：

- `Submitted` = “validator 接住了，并且愿意继续推进”
- `Executed` = “validator 手里已经有结果了”
- 这两者都不等于“网络 finality 已经完成”

网络 finality 是 `EffectsCertifier` 后面那一层要解决的问题。
