# 环境与能力探测

## 原则

- 只记录当前 Kimi Code 会话实际探测到的事实，不沿用上一次会话或其他宿主的假设。
- 先做只读探测，再做需要副作用、后台任务或联网的动作。
- 不预置平台避坑清单；环境差异通过探测、证据和记忆处理。
- 工具缺失、权限拒绝或输出不可解析时，标记未覆盖，不模拟结果。

## 必记字段

| 字段 | 说明 |
|---|---|
| Kimi Code 版本 | 从会话、CLI、安装元数据或用户陈述中确认；不确定就写未知 |
| 模型标识与版本 | 只记录可观察到的模型名、路由和推理强度 |
| 工作目录 | `Bash` 实际 cwd；支持 `cwd` 参数时也要记录每次调用值 |
| 目标路径 | 用户给出的真实绝对路径；记录是否可读 |
| 权限模式 | Always Ask、Ask When Needed、Never Ask 或未知 |
| Plan mode | 是否开启；开启时只限制 `Write` / `Edit` 和 `TaskStop` |
| swarm mode | 是否开启；`AgentSwarm` 只在开启时可用 |
| 操作系统与架构 | 当前进程实际报告的系统与架构 |
| shell 与版本 | Kimi Code 当前 `Bash` 实际使用的 shell |
| 解释器 | Python、Node、Java 等的实际路径和版本 |
| Git | 版本、仓库根、当前分支、工作区状态 |
| 构建工具 | 实际可用的构建、测试、包管理工具及版本 |
| 容器/虚拟化 | 运行时是否可用、实际架构、权限 |
| 网络 | 是否可联网、是否被代理或策略限制 |
| 写权限 | 工作区、临时目录、报告目录、外部记忆目录 |
| 技能发现 | 技能面板、斜杠补全或实际 `/skill:project-health-audit` 调用的结果 |
| 工具面 | `Read`、`Grep`、`Glob`、`ReadMediaFile`、`Bash`、`Skill`、`Agent`、`WebSearch`、`FetchURL` 等是否注册 |
| 后台任务 | `Bash` 的 `run_in_background`、`TaskOutput`、`WaitFor` 是否可用；输出路径如何读取 |
| 子代理 | `Agent` / `AgentSwarm` 是否可用；实际子代理类型和只读状态 |

## 能力状态

每项能力只使用以下状态：

- `present`：已实际调用或确认可用；
- `partial`：存在但版本、权限、参数或工具面受限；
- `absent`：确认不可用；
- `unknown`：无法确认，必须标未覆盖。

状态不是布尔值，不能把 `unknown` 写成 `present`。

## Kimi Code 专项探测

1. 确认目标路径真实存在且可读。
2. 用当前会话的技能面板、斜杠补全或实际调用确认技能是否真的被发现；不要仅凭目录存在判断。
3. 记录权限模式：Always Ask、Ask When Needed（`/yolo`）或 Never Ask（`/auto`）；审计不得默认使用 Never Ask。
4. 记录 Plan mode 是否开启；Plan mode 不是只读沙箱。
5. 记录 swarm mode 是否开启；`AgentSwarm` 只在 swarm mode 下运行，审计默认不用。
6. 确认 `Read` 的字符和 token 预算；中文长文件必须分段读取。
7. 需要命令时，记录 `Bash` 实际 cwd、shell、超时设置和输出/错误格式。
8. 需要长任务时，确认 `run_in_background` 返回的 task ID、`TaskOutput` 的 `output_path`、自动通知和 `WaitFor` 行为。
9. 需要视觉结论时，确认当前模型有 `image_in` / `video_in`，并实际调用 `ReadMediaFile`；不要从文件名或 URL 推断。
10. 确认技能根和外部记忆目录是否可写；不要在技能根内创建本地记忆。
11. 检查目标仓库的 `.kimi-code/agents/` 和 `.agents/agents/`；`override: true` 的 Agent 文件是系统提示，不是普通文档。
12. 检查用户级 `~/.kimi-code/config.toml` 的 Hook 配置；Hook 是 fail-open，不是完整安全边界。
13. 不读取或输出 Kimi Code 凭据、令牌、session 数据库和密钥文件。

## 失败与降级

- 目标不在可见工作区：停止并说明需要用户提供可读路径或只读副本。
- `Read` 输出被截断：用 `line_offset` / `n_lines` / `max_chars` 继续分段读取，或标记未覆盖。
- `Grep` / `Glob` 不可用：改用只读 `Bash` 搜索；若也不能保证只读，标未覆盖。
- `Bash` 超时或转后台：用 `TaskOutput` 和 `output_path` 读取到终态；无法读取则标未覆盖。
- `WaitFor` 超时：不是错误；继续等待、改用 `TaskOutput`，或标记未完成。
- 权限拒绝：记录工具、模式和拒绝信息，不把最终回答当成成功。
- 写权限不确定：不要写报告或记忆，先确认位置。
- 子代理不可用：顺序完成双轴审查，不模拟并行结果。
