# RFC: Dora Operating Platform (dop) 基础架构与指令矩阵

- **提案名称:** `dop_foundation_and_command_matrix`
- **开始日期:** 2026-04-15
- **作者:** Leon
- **原文链接:** [English Version](./README.md)

---

## 1. 摘要

> *(简要描述本提案内容。)*

本提案旨在确立 **Dora Operating Platform (`dop`)** 命令行工具的底层架构基础与全局指令矩阵。作为连接 LoopFun 开源社区与 DoraOne 商业交付的核心引擎，`dop` 将通过严格的物理隔离环境（`DOP_HOME`）和统一的 26 项核心指令，消除由于环境碎片化、跨语言调用和边缘部署带来的工程摩擦。

## 2. 动机

> *(为什么我们要发起这项变更？)*

- **痛点:** 现有的机器人工具链过于沉重且缺乏现代包管理；`cargo` 或 `uv` 等工具无法原生解决跨语言零拷贝共享内存总线的调度。
- **预期收益:** 实现开发环境与宿主机的零污染隔离；通过单一入口接管从脚手架创建到真机部署的全生命周期；确立 DoraFun 平台的标准接入契约。
- **不作为的代价:** 开发者将继续受困于碎片化的脚本，增加商业交付的非标运维成本。

## 3. 详细设计

### 3.1 核心哲学

1. **约定优于配置**: 直接复用 `dataflow.yaml`、`Cargo.toml` 等既有契约，拒绝引入冗余配置。
2. **快速失败与优雅降级**: 系统需具备针对断电、磁盘饱和等极端物理环境的自愈能力。

### 3.2 沙盒引擎

所有底层 I/O 遵循以下逻辑实现隔离：

- 优先读取 `$DOP_HOME`，默认为 `~/.dop/`。
- 维护四个核心区域：
  - `bin/`: 存放底层引擎的二进制执行文件。
  - `cache/`: 全局依赖缓存与环境隔离锁。
  - `models/`: 端侧大模型权重的统一收容所，支持跨项目软链接。
  - `logs/`: 系统级守护进程运行日志。

### 3.3 指令矩阵 (26 个核心命令)

本 RFC 确立 `dop` V1.0 的 6 大阵列、26 个核心指令签名的合法性：

- **创世阵列 (Scaffold)**: `init`, `new`
- **云端阵列 (Cloud)**: `login`, `logout`, `config`, `search`, `info`
- **生命周期 (Lifecycle)**: `start`, `run`, `build`, `test`, `stop`
- **生态阵列 (Ecosystem)**: `add`, `remove`, `update`, `publish`
- **具身阵列 (Embodied)**: `record`, `replay`, `model`, `deploy`
- **运维阵列 (Ops)**: `status`, `logs`, `doctor`, `shell`, `clean`, `self-update`

## 4. 权衡与技术债 (缺点)

- **认知负荷**: 26 个命令对初学者有压力。缓解方案：收敛高频命令，隐藏高级操作。
- **磁盘空间**: 绝对隔离会导致空间占用。缓解方案：通过 `dop clean` 清理及软链接减少冗余。

## 5. 备选方案

- **基于 Docker**: 被否决。在 macOS/Windows 下挂载物理硬件（CAN/USB）损耗高，破坏开发者体验。
- **原生脚本封装**: 被否决。无法实现真正的跨语言统一调度，且 Windows 平台兼容性差。

## 6. 未决问题

- [ ] `dop record` 的 MCAP 格式与 TF Tree 时空对齐的具体实现。
- [ ] `dop deploy` 针对昇腾 CANN 等异构硬件的自动烧录规范。
- [ ] `dop publish` 时的 AST 解析防抖策略。
