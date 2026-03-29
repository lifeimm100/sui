<!--
Copyright (c) Mysten Labs, Inc.
SPDX-License-Identifier: Apache-2.0
-->

# AGENT.md

本文件为 Codex 在此目录中处理代码时提供指导。

## crate 级别的 Codex 指引文件
始终查阅子 crate 中的 Codex 指引文件（例如 `AGENT.md` 或 `AGENTS.md`）。如果与本文件冲突，以局部指引文件中的说明为准。

# 个人偏好
个人偏好会覆盖并扩展项目偏好：
- `@CODEX.local.md`

## 核心开发命令

### 许可证注释

所有新文件都必须在文件顶部使用注释写入以下许可证内容：

    Copyright (c) Mysten Labs, Inc.
    SPDX-License-Identifier: Apache-2.0

### 构建与安装

```bash
# 构建指定 crate。通常不需要构建 release 版本。
cargo build -p sui-core

# 仅做检查而不构建（推荐）
cargo check
```

### 测试

```bash
# 运行 e2e 测试。simtest 必须使用 `cargo simtest` 运行，以避免误报失败
cargo simtest -p sui-e2e-tests

# 运行 Rust 单元测试。跳过 simulation tests，因为它们在 `cargo nextest` 下可能产生误报失败
SUI_SKIP_SIMTESTS=1 cargo nextest run
```

**测试的重要说明：**
- 在本仓库中编译或运行测试时，由于代码库较大，超时时间至少设置为 10 分钟
- 为了加快迭代速度，使用 `-p` 选择最相关的 package。必要时可使用多个 `-p`，例如 `cargo nextest run -p sui-types -p sui-core`
- 使用 `cargo nextest --lib` 只运行库测试并跳过集成测试，以获得更快反馈
- 修改某些 crate 中的文件时，测试应如何运行，请查阅对应 crate 下的 Codex 指引文件

### Lint 与格式化

```bash
# 格式化并 lint 所有 Rust 与 Move 代码，提交前运行：
./scripts/lint.sh

# 或者分别运行各项 lint：
cargo fmt --all -- --check
cargo xclippy
```

`cargo xclippy does not recognize -p option`：这是某些 clippy 命令变体中的已知问题。

## 高层架构

### 核心组件结构

```text
sui/
├── crates/                             # 主要 Rust crates
│   ├── sui-core/                       # 区块链核心逻辑
│   ├── sui-node/                       # 验证者节点实现
│   ├── sui-framework/                  # Move 系统包与标准库
│   ├── sui-types/                      # 核心类型定义
│   ├── sui-json-rpc/                   # JSON-RPC API 服务
│   ├── sui-indexer-alt-graphql/        # GraphQL API 服务
│   └── sui-indexer-alt/                # 区块链数据索引器
├── consensus/                          # 共识机制（Mysticeti）
├── sui-execution/                      # Move 执行层，包含多个版本（v0、v1、v2 和 latest）
├── apps/                               # 前端应用
└── external-crates/                    # Move 编译器与虚拟机
```

### 关键架构模式

1. **Authority System**：Sui 使用一组验证者（authorities）并行处理交易。每个 authority 都维护自己的状态，并参与拜占庭共识。

2. **Object Model**：不同于基于账户的区块链，Sui 采用以对象为中心的模型，其中：
   - 每个对象都有唯一的 ID 和版本
   - 对象可以是 owned、shared 或 immutable

3. **Transaction Flow**：
   - Client → Transaction Driver → Authority Client → Validator
   - 仅影响 owned objects 的交易可以在共识之前开始执行
   - 涉及 shared objects 的交易需要先经过共识排序，再执行

4. **Storage Layer**：
   - 使用 RocksDB 进行持久化存储
   - 为对象、交易和 effects 分别维护独立的存储
   - 提供用于状态同步的 checkpointing 系统

5. **Execution Pipeline**：
   - 交易校验 → 证书创建 → 执行 → effects 提交
   - Move VM 负责执行智能合约并进行 gas 计量
   - 对无冲突交易进行并行执行

### 关键开发注意事项
1. **测试要求：**
   - 提交改动前始终运行测试
   - Framework 相关改动需要更新 snapshots
2. **Protocol Config 改动：**
   - 修改 `crates/sui-protocol-config/src/lib.rs` 时，始终调用 `/protocol-config` 以验证改动是否安全。错误的修改可能破坏网络共识。
3. **关键：开发收尾步骤：**
   - **完成开发后务必运行 `cargo xclippy`**，确保代码通过所有 lint 检查
   - **绝不要禁用或忽略测试**，所有测试都必须启用且通过
   - **绝不要使用 `#[allow(dead_code)]`、`#[allow(unused)]` 或其他 lint 抑制手段**，应修复根本问题
   - **所有单元测试都必须正常工作**，异步测试应使用 `#[tokio::test]`，而不是 `#[test]`

### **注释编写规范**

**不要为显而易见的内容写注释**。注释不应只是重复代码本身已经表达的意思。

**适合写注释的情况：**
- 不明显的算法或业务逻辑
- 临时排除、超时、阈值及其原因
- 复杂计算中不容易直接看出的“为什么”
- 微妙的竞态条件或线程相关考虑
- 对外部状态或前置条件的假设

**不适合写注释的情况：**
- 简单变量赋值
- 标准库用法
- 含义明确的函数调用
- 基本控制流（if/for/while）
