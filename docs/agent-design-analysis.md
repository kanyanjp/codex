# Codex agent 设计梳理

本文结合仓库内主要实现，对 Codex CLI 的 agent 体系做一次快速拆解，便于理解核心数据流与协作模式。

## 总览
- 入口：CLI 子命令由 `codex-rs/exec/src/cli.rs` 解析，`run_main`（`codex-rs/exec/src/lib.rs`）负责初始化配置、启动内置 app-server client，并根据指令选择启动对话、恢复会话或发起代码审查。
- 会话管线：与 app-server 的双工通讯通过 JSON-RPC 事件完成，`codex-rs/exec/src/event_processor_with_human_output.rs`/`event_processor_with_jsonl_output.rs` 将后端事件映射为人类可读输出或 JSONL 事件流。
- 指令基线：模型初始提示词存于 `codex-rs/core/gpt_5_codex_prompt.md`，额外的工作区指导来自 `AGENTS.md`/`docs/agents_md.md`（以及可选的层级子 agent 提示）。

## 运行流程（单 agent）
1. 配置加载：`run_main` 根据 CLI 覆写与 `config.toml` 生成 `Config`，同时校验登录、模型提供方（API/ChatGPT/OSS）与本地工作区 (`find_thread_path_by_*`)。
2. 沙箱/策略：执行命令前会检查 execpolicy 与 sandbox 配置（`check_execpolicy_for_warnings`、`codex-linux-sandbox` 构建），并据此决定是否需要用户批准。
3. 启动会话：通过 `InProcessAppServerClient` 建立内嵌 app-server，发起 `ThreadStart`/`TurnStart` 或 `ReviewStart` 请求；请求 ID 由本地 `RequestIdSequencer` 顺序生成。
4. 事件消费：服务端流式返回 `EventMsg`，`EventProcessor` 将之拆分为命令执行、补丁、计划、代理消息等条目，并交给具体输出渲染器。
5. 终端输出：默认人类模式仅在 stdout 写最终回复，其余进度/日志写 stderr；`--json` 模式输出严格的 JSONL 事件，便于脚本消费。

## 事件模型与输出
- 事件类型：`codex-rs/exec/src/exec_events.rs` 定义了 turn 生命周期、命令/补丁项、agent 消息、计划 Todo 列表、协作 agent 状态等结构。
- 人类输出：`event_processor_with_human_output.rs` 负责将事件转成带颜色/样式的终端行，并处理审批、用户输入提示、计划更新等交互。
- 机器输出：`event_processor_with_jsonl_output.rs` 维持线程状态、`agents_states` 等上下文，确保事件序列化为稳定的 JSON 结构供上层 SDK 使用。

## 多 agent 协作
- 事件支撑：协作相关事件（spawn/resume/wait/interaction/close）由协议层的 `Collab*` 事件承载，状态在 `exec_events.rs` 中归档为 `CollabAgentState`。
- TUI 展示：`codex-rs/tui/src/multi_agents.rs`、`tui/src/app.rs` 管理 agent 线程列表（主/子 agent），提供 agent picker、状态点、提示摘要等 UI，并在聊天窗里回放协作事件。
- 用户体验：多 agent 默认可禁用，TUI 会提示启用；每个子线程有独立的历史/最后消息，切换时通过快照重放保持上下文一致。

## 配置与指令来源
- 工作区约束：`AGENTS.md`（若存在）提供项目级约束与偏好；启用 `child_agents_md` 时还会向子 agent 下发层级提示，明确作用域。
- 模型/提供方：CLI 支持指定模型、提供方与 sandbox 模式，落在配置文件与 CLI 覆写，最终影响 app-server 请求的模型参数与安全策略。

## 安全与执行
- 沙箱：Linux 下通过 `codex-linux-sandbox`（bubblewrap/landlock）编译，配合 execpolicy 决定命令可见路径、网络等权限；失败时事件流会报告具体错误并中止。
- 批准流：`AskForApproval` 事件提示用户审批潜在风险操作，事件处理器会阻塞等待用户输入或取消，保持外部副作用可控。

## 观察与后续可深入点
- 事件-输出解耦良好：同一事件源可以切换人类/机器输出，便于集成 CLI 与 SDK。
- 协作流程可扩展：`Collab*` 事件覆盖 spawn/resume/wait/close，全链路已有 UI 与 JSON 表达，可在此基础上增加队列或策略层。
- 安全基线明确：配置→策略→沙箱→审批形成闭环，但依赖系统库（如 libcap）与配置文件，部署前需确认环境就绪。
