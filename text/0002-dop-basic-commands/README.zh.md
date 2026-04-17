# RFC: dop 基础指令集规范

- **提案名称:** `dop_basic_command_specifications`
- **开始日期:** 2026-04-15
- **作者:** Leon
- **原文链接:** [English Version](./README.md)

---

## 1. 摘要

> *(简要描述本提案内容。)*

继 RFC-0001 确立 `$DOP_HOME` 沙盒逻辑之后，本提案详细规划了 Dora Operating Platform (`dop`) 16 个基础命令的输入/输出（I/O）契约与预期行为。这些命令涵盖了脚手架创建、身份鉴权、基础生命周期管理和系统级运维观测，旨在为异构硬件上的具身智能开发提供一致的体验。

## 2. 动机

> *(为什么我们要发起这项变更？)*

- **可预测性:** 开发者在与 CLI 交互时需要明确的心理模型。模糊的命令行为会导致开发效率低下。
- **自动化支持:** CI/CD 流水线和 DoraFun 平台强依赖于标准的退出码（Exit Codes）和结构化终端输出（如 JSON 格式）来实现工作流自动化。
- **对齐 Cargo:** 采用 Rust 生态中广为熟知的范式（如基于 Token 的登录模式），能够大幅降低核心贡献者的入门门槛。

## 3. 详细设计

### 3.1 创世阵列 (Scaffold)

#### 3.1.1 `dop init`
- **命令签名:** `dop init [--force]`
- **设计哲学 (Why):** 具身智能开发常因全局 C++/Python 依赖冲突而崩溃。`init` 是构建确定性 `$DOP_HOME` 沙盒的“创世大爆炸”，确保 `dop` 绝不依赖宿主机不可预测的全局状态。
- **实现逻辑 (How):**
  1. **路径解析:** 解析 `$DOP_HOME`。若未设置，降级为 `~/.dop`。
  2. **目录分配:** 使用严格的 `0755` 权限执行 `mkdir -p`，建立 `bin`, `cache/cargo`, `cache/uv`, `models`, `logs` 目录树。
  3. **二进制内嵌 (Vendoring):** 发起并发异步 HTTP GET 请求，从 DoraFun CDN 拉取锁死版本的预编译底层引擎（如 `uv`, `dora-cli`, `zenohd`）。
  4. **链接与赋权:** 将二进制文件落盘至 `$DOP_HOME/bin` 并执行 `chmod +x`。
- **边缘情况与异常:** - 下载断网：必须回滚清除未下载完整的文件（故障安全）。
  - 目录已存在：以退出码 `0` 优雅结束，除非显式传入 `--force`（幂等性）。

#### 3.1.2 `dop new`
- **命令签名:** `dop new <project_name> [--template <registry/name>]`
- **设计哲学 (Why):** 手写 `dataflow.yaml` 和 IPC 共享内存代码极易出错。我们需要提供零阻力的“第一天”体验。
- **实现逻辑 (How):**
  1. **模板拉取:** 若未指定 `--template`，默认使用内嵌的 `basic-rust-python` 模板。若指定，则从 `github.com/dorafun-io/templates` 拉取压缩包。
  2. **水化引擎 (Hydration):** 使用轻量级模板引擎（如 Rust 的 `tera`）全局替换脚手架文件中的 `{{project_name}}`, `{{uuid}}`, `{{dop_version}}` 等变量。
  3. **Git 初始化:** 自动执行 `git init` 并注入专为 DoraFun 调优的 `.gitignore`（默认忽略 `mcap` 数据包和本地 `target/` 构建目录）。

### 3.2 云端与配置阵列 (Cloud & Metadata)

#### 3.2.1 `dop login`
- **命令签名:** `dop login [--token <jwt>]`
- **设计哲学 (Why):** 边缘计算设备（如树莓派、Jetson）通常运行在无头（Headless）模式下，无法唤起浏览器。为了极致的 CLI 兼容性，我们强制对齐 Cargo 风格的个人访问令牌（PAT）工作流。
- **实现逻辑 (How):**
  1. **交互式提示:** 若缺失 `--token`，调用安全的标准输入读取器（隐藏击键回显）等待用户粘贴。
  2. **凭证校验:** 携带 Bearer Token 向 DoraFun API 发送 `GET /v1/auth/verify`。
  3. **安全落盘:** 若校验通过 (HTTP 200)，将 Token 写入 `$DOP_HOME/config/credentials.toml`。**极其关键：** 必须显式将系统文件权限设置为 `0600`（仅所有者读写），防止提权泄露。

#### 3.2.2 `dop logout`
- **命令签名:** `dop logout`
- **实现逻辑:** 在删除文件前，先对 `$DOP_HOME/config/credentials.toml` 执行安全覆写（写零），确保 Token 无法被底层磁盘扇区恢复软件找回。

#### 3.2.3 `dop config`
- **命令签名:** `dop config <get|set> <key> [value]`
- **设计哲学:** 用户需要一个中心化的入口来修改底层行为，而手动修改 TOML 文件极易引发语法错误。
- **实现逻辑 (How):**
  1. **AST 解析:** 使用保留注释的解析器（如 Rust 的 `toml_edit`）。绝不能将其反序列化为结构体再序列化，因为这会抹除用户自己写的注释。
  2. **白名单校验:** 实现合法键值的注册表（如 `registry.mirror`）。遇到未知 key 直接拒绝并返回退出码 `1`。
  3. **状态突变:** 修改 AST 节点并刷入 `$DOP_HOME/config/config.toml`。

#### 3.2.4 `dop info`
- **命令签名:** `dop info [--json]`
- **设计哲学:** 在运行时执行前，提供静态项目状态的即时可观测性。对调试 CI/CD 流水线至关重要。
- **实现逻辑:**
  1. **图谱解析:** 向上追溯并解析工作区中的 `dataflow.yaml`。
  2. **系统探针:** 探测宿主机架构 (`uname -m`)、操作系统以及可用的算力加速器（如通过探查 `/dev/davinci0` 来判断是否有华为昇腾 CANN 芯片）。
  3. **格式化输出:** 打印人类可读的汇总。若携带 `--json` 参数，则向 `stdout` 输出严格类型的 JSON Schema，专供自动化脚本消费。

### 3.3 生命周期阵列 (Lifecycle)

#### 3.3.1 `dop build`
- **命令签名:** `dop build [--release]`
- **设计哲学 (Why):** 具身智能项目天生是多语言的。一个 `dataflow.yaml` 可能同时定义了 C++ 的传感器驱动、Rust 的核心路由和 Python 的神经网络节点。要求开发者手动调用各种编译工具是极其反人类的。
- **实现逻辑 (How):**
  1. **清单解析:** 解析 `dataflow.yaml`，构建节点依赖的有向无环图 (DAG)。
  2. **沙盒隔离编译:**
     - 针对 Rust 节点：分发给 `cargo build`，并注入 `--target-dir` 将构建产物重定向至 `$DOP_HOME/cache/cargo`。
     - 针对 Python 节点：分发给 `uv build` 或 `uv pip sync`，将依赖锁定在 `$DOP_HOME/envs/<uuid>` 中。
  3. **产物链接:** 将编译好的二进制文件通过软链接 (Symlink) 映射回项目本地的 `.dop/target/` 目录，同时借助全局 `.gitignore` 保持项目 Git 状态的纯净。

#### 3.3.2 `dop run`
- **命令签名:** `dop run [node_id]`
- **设计哲学 (Why):** 前台调试的核心入口。它必须保证绝对的状态可复现性。如果机器人在 `dop run` 期间崩溃，即使代码没有 commit，我们也必须精确知道当时运行的究竟是哪行代码。
- **实现逻辑 (How):**
  1. **Gitoxide 影子快照 (核心亮点):** 在拉起任何进程前，`dop` 调用内嵌的 `gitoxide` 引擎扫描工作区。如果发现未提交的脏代码，它会直接在内存中构建 Tree 和 Commit 对象写入 `.git/objects`，并打上 `refs/tags/dop/run-<timestamp>` 的游离标签。这为运行时构建了一个零性能损耗的“黑匣子”。
  2. **上下文劫持:** 突变子进程的环境变量：
     - 将 `$DOP_HOME/bin` 插入 `$PATH` 头部。
     - 将 `$PYTHONPATH` 指向隔离的虚拟环境。
     - 注入 `$DORA_ROOT` 以支持零拷贝共享内存映射。
  3. **前台执行与 IPC:** 拉起本地 Zenoh 路由器及指定节点。将 `stdout/stderr` 转发至终端，并按 `node_id` 进行颜色区分。阻塞进程直至接收到 `SIGINT` (Ctrl+C)。

#### 3.3.3 `dop start`
- **命令签名:** `dop start [node_id]`
- **设计哲学 (Why):** 专为无头边缘计算主板（如 Jetson, 昇腾 CANN）和生产环境部署设计，要求进程的生命周期必须超越 SSH 会话。
- **实现逻辑 (How):**
  1. **快照与构建:** 继承与 `dop run` 完全相同的 `gitoxide` 影子快照与编译逻辑。
  2. **守护进程化:** Fork 出子进程并脱离当前终端的控制 (例如在 Rust 中使用 `nix::unistd::daemon`)。
  3. **I/O 重定向:** 强制将标准输出/错误重定向至 `$DOP_HOME/logs/<project_uuid>/<node_id>.log`，并启用按大小（如 50MB）滚动的日志轮转机制。
  4. **状态追踪:** 将主进程及各节点的 PID 写入 `$DOP_HOME/run/<project_uuid>.pid` 文件。主程序立即以退出码 `0` 结束，将终端控制权交还给用户。

#### 3.3.4 `dop stop`
- **命令签名:** `dop stop [node_id] [--force]`
- **设计哲学 (Why):** 具身智能节点通常独占物理硬件的锁（如 `/dev/video0` 摄像头或 CAN 总线）。暴力强杀会导致硬件陷入僵死状态。我们必须强制执行优雅卸载契约。
- **实现逻辑 (How):**
  1. **PID 解析:** 从 `$DOP_HOME/run/<project_uuid>.pid` 读取目标 PID。
  2. **优雅卸载 (SIGTERM):** 向进程发送 `SIGTERM` 信号。给予各节点严格的 5.0 秒时间窗口，用于刷新内存映射并释放硬件物理锁。
  3. **强制升级 (SIGKILL):** 如果 5.0 秒超时后进程仍未退出，引擎将升级发送 `SIGKILL` 强制终结进程。
  4. **零拷贝垃圾回收:** 唤醒清理例程，擦除因进程崩溃而遗留在系统中的孤儿 POSIX 共享内存段 (`/dev/shm/dora_*`)，防止内存泄漏。

### 3.4 运维与观测阵列 (Ops)

#### 3.4.1 `dop status`
- **命令签名:** `dop status`
- **设计哲学 (Why):** 在具身智能中，CPU/GPU 突刺或零拷贝总线的拥塞会直接转化为物理延迟（例如机械臂反应慢了 200 毫秒）。开发者需要一个实时、原生于终端的监控大盘，而无需启动沉重的 Web 服务。
- **实现逻辑 (How):**
  1. **TUI 渲染:** 使用终端 UI 库（如 Rust 的 `ratatui`）绘制非阻塞的交互界面。
  2. **指标采集:**
     - 探查宿主机 `procfs` 获取各 `node_id` 进程的 CPU/内存利用率。
     - 质询本地 Zenoh 路由器，获取发布/订阅的吞吐量及 IPC 共享内存的时延。
  3. **视觉告警:** 如果某个节点的消息队列背压（Backpressure）超过阈值（如导致 60FPS 摄像头节点开始丢帧），系统将该节点标红高亮。

#### 3.4.2 `dop logs`
- **命令签名:** `dop logs [node_id] [-f/--follow]`
- **设计哲学 (Why):** 多语言节点混合输出的原始 `stdout` 是灾难性的文本瀑布。我们需要结构化、可查询的日志流。
- **实现逻辑 (How):**
  1. **日志聚合:** 从 `$DOP_HOME/logs/<project_uuid>/` 读取由 `dop start` 生成的 JSON-line 结构化日志。
  2. **终端格式化:** 反序列化 JSON，按日志级别（ERROR=红色, WARN=黄色）和 `node_id` 前缀着色，格式化为人类可读的输出。
  3. **实时追踪:** 如果附带 `-f`，则调用文件系统监听器（如 `notify` crate）实时流式输出新日志。

#### 3.4.3 `dop doctor`
- **命令签名:** `dop doctor`
- **设计哲学 (Why):** 80% 的“第一天”运行失败源于硬件和操作系统的错误配置。`dop doctor` 是环境完整性的最终“体检报告”。
- **实现逻辑 (How):**
  1. **设备权限验证:** 检查当前用户是否属于 `video` 和 `dialout` 用户组（Linux），以确保拥有 USB 摄像头和串口 (`/dev/ttyUSB0`) 的读写权限。
  2. **加速器探针:** 执行 NVML 调用验证 NVIDIA GPU 驱动，或调用 `npu-smi` 验证华为昇腾 NPU 的完备性。
  3. **共享内存限制:** 检查 `/dev/shm` 的大小和系统 `ulimit -l` (memlock) 限制。若上限不足以支撑大张量（Tensor）的零拷贝传递，向用户发出警告。
  4. **输出指引:** 打印带有 `[PASS]`, `[WARN]`, `[FAIL]` 的检查清单，并提供可直接复制粘贴的修复命令（例如 `sudo usermod -aG video $USER`）。

#### 3.4.4 `dop shell`
- **命令签名:** `dop shell`
- **设计哲学:** 有时开发者需要进入与 `dop run` 完全相同的微环境，手动执行底层的 `python` 或 `cargo` 命令以排查诡异的依赖问题。
- **实现逻辑:**
  1. 从 `$SHELL` 环境变量获取用户的默认 Shell。
  2. 派生子进程启动该 Shell，并突变其环境变量：将 `$DOP_HOME/bin` 注入 `$PATH`，将 `$PYTHONPATH` 指向项目的虚拟环境。
  3. 修改命令提示符（`$PS1`），在前面增加 `(dop) ` 前缀，提示开发者目前已处于沙盒接管状态。

#### 3.4.5 `dop clean`
- **命令签名:** `dop clean [--all]`
- **设计哲学 (Why):** 具身智能项目会堆积海量的产物：C++ 编译缓存、大模型权重，以及我们系统底层的 `gitoxide` 影子提交。必须提供强有力的垃圾回收机制。
- **实现逻辑 (How):**
  1. **常规 GC:** 删除 `$DOP_HOME/cache/cargo/target` 和 `$DOP_HOME/logs/` 中超过 7 天的文件。
  2. **深度 GC (`--all`):**
     - **模型修剪:** 扫描 `$DOP_HOME/models`。如果某个权重未被已知项目中的 `dop.lock` 引用，则予以删除。
     - **影子提交清理 (核心环节):** 调用 `gitoxide` 寻找所有超过 30 天的 `refs/tags/dop/*` 标签。删除标签后，执行底层的 `git gc` 以清洗 `.git/objects` 中游离的 Blob/Tree 对象，彻底释放磁盘空间。

#### 3.4.6 `dop self-update`
- **命令签名:** `dop self-update`
- **实现逻辑:**
  1. 从 DoraFun CDN 拉取最新的发布清单。
  2. 将新版 `dop` 二进制文件下载为临时文件 (`dop.tmp`)。
  3. 执行**原子级重命名**（如 Rust 的 `std::fs::rename`）。直接覆写正在运行的二进制文件会引发操作系统的 ETXTBSY（文本文件忙）错误。原子重命名巧妙绕过了此限制，确保即便在断电瞬间，CLI 也绝不会变砖。

## 4. 权衡与技术债

- **命令表面积:** 16 个命令构成了庞大的基础面。但我们通过底层子命令解析器（如 Rust 的 `clap`）提供详尽的 `--help` 文档，以此来缓解学习曲线。
- **后台守护进程 (`start`/`stop`):** 实现稳健的跨平台守护进程（尤其是在 Windows 上）技术极具挑战。但为了满足边缘设备 Headless（无头）部署的刚需，我们选择背负此技术债。

## 5. 未决问题

- [ ] 当 `dop status` 的输出被管道（Pipe）传递给其他程序时，是否应自动降级输出为严格的 JSON 格式，还是必须要求用户显式传入 `--json` 标志？
- [ ] `dop stop` 在升级发送 SIGKILL 之前的确切超时阈值应设定为多少？