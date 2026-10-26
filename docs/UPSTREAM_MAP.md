# RepoPilot Phase 0：上游源码与环境地图

核查日期：2026-10-26（Asia/Shanghai）。本轮仅调查和编写本文；**源码存在不等于运行验收通过**。首次发布仓库时，源码链接已改为固定 Commit 的 GitHub 链接，便于在未下载上游源码时查阅；环境与检查结果仍为 Phase 0 快照。

## 1. 版本、工作区与环境实测

- 已完整阅读 RepoPilot 的 `AGENTS.md`、`docs/PROJECT_PLAN.md`；已阅读上游 `AGENTS.md`、`backend/AGENTS.md`、`docs/ARCHITECTURE.md`、`backend/docs/GITHUB_AGENTS.md`、`Install.md` 及 `examples/deerflow-extension-example/` 的说明和实现。
- 上游位于 `upstream/deer-flow/`，origin 为 `https://github.com/bytedance/deer-flow.git`。本地 `HEAD` 与 `v2.1.0^{commit}` 均为 **`345f08be00c8a9495079b732a39b46aa9af1584e`**，提交日期 2026-09-24；detached HEAD，工作区干净。后端及前端包版本均为 `2.1.0`。这是本地 Git 证据，本轮未 fetch 或重新验证远端标签。
- RepoPilot 首检时无 `.git`，`git status` 返回 `not a git repository`；收尾检查时已有 `.git` 与 `.gitignore`，当前为 `main`、尚无提交。本轮未执行 `git init` 或提交。现有忽略规则已排除 `upstream/`、`.env`、`config.yaml`、私钥等，`git check-ignore` 已确认上游及其配置被忽略。

| 检查项 | 本机结果 | 判断 |
| --- | --- | --- |
| WSL2 | Ubuntu 24.04.3 LTS，x86_64，`6.6.87.2-microsoft-standard-WSL2`；20 个逻辑 CPU，约 23.1 GiB 内存、946.8 GiB 可用磁盘 | 满足上游 Docker 开发建议的资源量；挂载仍须实测 |
| Git / Python / Make | `2.43.0` / `python3 3.12.3` / `GNU Make 4.3` | Python 满足上游 `>=3.12`；没有 `python` 命令别名 |
| Docker / Compose | CLI、Server 均 `29.9.2`；Compose `v5.6.1`；`docker info` 成功，服务端为 Docker Desktop；`/var/run/docker.sock` 存在且为 socket | 可优先走官方 Docker 路径；Compose 通过脚本的 `>=2.24` 版本条件 |
| WSL 原生开发工具 | PATH 中无 `uv`、`node`、`pnpm`、`nginx`、`gh`；`npm 11.6.2` 来自 `/mnt/c/Program Files/nodejs/npm` | Windows npm 不代表 WSL 的 Node 工具链齐备；这些缺项不直接阻塞 Docker 方案 |
| 已有镜像 / 容器 | 已有 `deer-flow-dev-gateway`、`deer-flow-dev-frontend`、nginx、redis、AIO `1.11.0` 镜像；`docker ps -a` 无容器 | 镜像来源/构建对应提交、容器内依赖和 `git/gh` 尚未运行验证 |
| 本地依赖 / 配置 | 上游 `config.yaml`、`extensions_config.json`、根 `.env` 已存在；`backend/.venv/` 为空；无 `frontend/node_modules/` | 不能认定宿主依赖已安装；未打开任何 `.env` 或私钥文件 |
| 非敏感配置摘要 | 模型 `repopilot-default` → `deepseek-flash`，使用 `PatchedChatDeepSeek`，引用 `$REPOPILOT_API_KEY`；Sandbox 为 `AioSandboxProvider`、镜像 `1.11.0`，无 `provisioner_url`；GitHub 通道关闭；5 个 MCP server 均禁用，未配置 Python plugin | 已有配置可沿用；密钥有效性、模型可用性和 Tool Calling 均未验证 |

部署建议：使用官方 Docker 开发栈，单 Gateway/单 Worker、单主 Agent。它自带 Redis 用于流式事件等上游能力，见 [docker-compose-dev.yaml](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/docker/docker-compose-dev.yaml)；这不是 RepoPilot 新建任务队列。当前 AIO 配置走本地 Docker 容器，`scripts/docker.sh` 会选用 Docker socket overlay，无需 Kubernetes。上游 README 对 Windows/WSL 本地脚本兼容性有保留，因此不因版本检查通过就宣称部署成功。

## 2. 已证实的调用链与扩展接口

下文 `A/` 指 `upstream/deer-flow/backend/app/`，`H/` 指 `upstream/deer-flow/backend/packages/harness/deerflow/`。行号以本次固定 Commit 为准。

```mermaid
flowchart LR
    W[POST /api/webhooks/github] --> D[fanout_event / 绑定与触发过滤]
    D --> B[MessageBus / InboundMessage]
    B --> C[ChannelManager / SDK runs.create]
    C --> R[Gateway start_run / RunManager]
    R --> L[run_agent / assemble_lead_agent / LangGraph]
    L --> T[Skills / 内置工具 / MCP]
    T --> S[bash_tool / AIO Sandbox]
    S --> G[git 与 gh]
    G -. RepoPilot 后续交付策略 .-> P[Draft PR / 人工审核]
```

MCP 工具在自身进程/远端执行；图中的 Sandbox 箭头对应 `bash` 等沙箱工具，不能理解为全部 MCP 调用自动进入沙箱。

| 能力 | 源码入口、关键函数与行为 | RepoPilot 复用方式 |
| --- | --- | --- |
| Webhook | [A/gateway/routers/github_webhooks.py](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/backend/app/gateway/routers/github_webhooks.py)：`is_route_enabled` L85、`_verify_signature` L101、`receive_github_webhook` L173 | 复用 `POST /api/webhooks/github`、HMAC-SHA256 和常量时间比较；未配置 secret 且未开启开发绕过时路由不挂载；错误签名返回 401 |
| 路由 / 消息总线 | [A/gateway/github/dispatcher.py](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/backend/app/gateway/github/dispatcher.py)：`fanout_event` L188 → `registry.build_github_agent_registry/lookup_agents` → `triggers.event_should_fire` → `build_prompt` → `bus.publish_inbound` L483 | 精确匹配绑定的 repo/event，过滤自身事件；`identity.resolve_thread_id` 用 repo、Issue/PR number、agent_name 生成 UUID5；消息带 owner、agent_name、preferred_thread_id |
| ChannelManager | [A/channels/manager.py](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/backend/app/channels/manager.py)：`_resolve_run_params` L1610、`_apply_channel_policy` L1718、`_create_thread` L2235、`_handle_chat_on_thread` L2419 | SDK 创建/复用 Thread，把 Custom Agent 名写入上下文；GitHub 用 `runs.create` L2503、`multitask_strategy="reject"`，提交后台运行即返回；不是等待一次长 HTTP 请求完成 |
| Gateway / Runtime | [A/gateway/routers/thread_runs.py](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/backend/app/gateway/routers/thread_runs.py)：`create_run` L925 → [A/gateway/services.py](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/backend/app/gateway/services.py)：`start_run` L1459 → `H/runtime/runs/manager.py::RunManager.create_or_reject` → [H/runtime/runs/worker.py](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/backend/packages/harness/deerflow/runtime/runs/worker.py)：`run_agent` L807 | 内部接口 `POST /api/threads/{thread_id}/runs`；原生管理 Run、checkpointer、后台 task、StreamBridge/SSE，无需新增 Runtime 或队列 |
| Custom Agent / Agent Loop | `services.resolve_agent_factory` L751 返回 [H/agents/lead_agent/agent.py](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/backend/packages/harness/deerflow/agents/lead_agent/agent.py) 的 `assemble_lead_agent` L801 → `_assemble_lead_agent` L916 → `create_agent` L1239；worker 组装后调用 `agent.astream` L1262/L1295 | Custom Agent 是 Lead Agent 的配置与 SOUL；`load_agent_config` 选择模型/工具组/Skill，`build_middlewares` 接入线程目录、Skill 策略及 Sandbox。无需另建 Agent 内核 |
| Skills | [H/skills/storage/skill_storage.py](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/backend/packages/harness/deerflow/skills/storage/skill_storage.py)：`load_skills` L277 → `parse_skill_file`；[H/agents/lead_agent/prompt.py](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/backend/packages/harness/deerflow/agents/lead_agent/prompt.py)：`get_enabled_skills_for_config` L173；`SkillActivationMiddleware`、`SkillToolPolicyMiddleware` 处理激活和工具范围 | 后续安装 RepoPilot 的 `SKILL.md`，在 Agent 的 `skills` 列表中指定；先展示发现元数据，再按需加载正文。当前源码已迁移到 `skills/storage/`，不能照旧文档寻找 `skills/loader.py` |
| MCP / 工具集合 | [H/tools/tools.py](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/backend/packages/harness/deerflow/tools/tools.py)：`get_available_tools` L73 → `H/mcp/cache.py::get_cached_mcp_tools/initialize_mcp_tools` → [H/mcp/tools.py](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/backend/packages/harness/deerflow/mcp/tools.py)：`get_mcp_tools` L772 → `MultiServerMCPClient` | `extensions_config.json → mcpServers`；支持 `stdio`、`sse`、`http`，配置 `enabled/type/command/args/env/url/headers`、超时等；默认工具名加 server 前缀。独立 Python MCP 包即可接入 |
| Sandbox / git / gh | [H/sandbox/tools.py](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/backend/packages/harness/deerflow/sandbox/tools.py)：`bash_tool` L2118 → `ensure_sandbox_initialized` L1504 → provider → `_execute_bash_command` L1690 → [H/community/aio_sandbox/aio_sandbox.py](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox.py)：`execute_command` L387 | `AioSandboxProvider._create_backend` L253 选择 `LocalContainerBackend`，容器内执行仓库命令；默认工作区 `/mnt/user-data/workspace`。`git/gh` 是沙箱程序，不是上游自动生成 PR 的独立 Python 服务 |
| Python Extension（可选） | [examples/deerflow-extension-example/deerflow_extension_example/__init__.py](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/examples/deerflow-extension-example/deerflow_extension_example/__init__.py)：`install` L22；[H/extensions/loader.py](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/backend/packages/harness/deerflow/extensions/loader.py)：`load_extensions` L150 | `deerflow-extension-api` 的 `@extension(api="0.2.0")`、PEP 621 `deerflow.extensions` entry point、`config.yaml → plugins`；支持 middleware、task lifecycle、system-model observer、service、router，重启生效。统计扩展留到 Phase 6 |

**Agent 与触发配置。** [H/config/agents_config.py](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/backend/packages/harness/deerflow/config/agents_config.py) L61/L97/L206 是实际 Schema。文件存储下 Agent 位于 `{DEER_FLOW_HOME}/users/{user_id}/agents/{name}/{config.yaml,SOUL.md}`；本机 `agent_storage.backend=file`。关键字段如下：

| 层级 | 实际字段 / 语义 |
| --- | --- |
| 全局 `channels.github` | `enabled`、`default_mention_login`；绑定不放这里 |
| Agent | `name`、`model`、`tool_groups`、`skills`、`allowed_subagents`、`model_settings`、`memory_enabled`、`github` 等；`skills: null` 为全部可用，`[]` 为无 Skill |
| Agent 的 `github` | `installation_id`、`bot_login`、`recursion_limit`、`bindings: [{repo, triggers}]`；GitHub 默认 recursion limit 250，可按 Agent 收紧 |
| `triggers.<event>` | 仅 `actions`、`require_mention`、`allow_authors`、`mention_login`；只注册显式声明的 event。`issue_comment` 默认需 mention，`pull_request` 默认 action 为 `opened` |

接口 `POST/PUT /api/agents/...` 的创建/更新请求模型未暴露 `github` 字段；UI 更新会保留已有非托管字段，不能假设通过普通 Agent 表单就能配置 GitHub。Phase 2 可沿用文件存储，在 Agent 自己的 YAML 中绑定测试仓库，先用 `issue_comment / created / require_mention=true` 做受控只读验证。

**必须保留的边界：**

1. [triggers.py](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/backend/app/gateway/github/triggers.py) L150 没有 label-name 过滤；`actions: [labeled]` 会匹配所有加标签事件，不能等同于仅 `ai-task`。`allow_authors` 是绕过 mention 的便利名单，不会拒绝其他作者的有效 mention；对 `issues` 它取 Issue 作者而非加标签的操作者。HMAC 和 repo 绑定也不等于维护者授权。MVP 先限定自有受控仓库，Phase 4 开写前补齐授权验收。
2. 凭据路径为 [run_policy.py](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/backend/app/gateway/github/run_policy.py) 的 `inject_github_credentials` → [app_auth.py](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/backend/app/gateway/github/app_auth.py) 的 `mint_installation_token` → 内部 `run_context.github_token` → `sandbox/tools.py::_github_env_from_runtime` → 单次命令的 `GH_TOKEN/GITHUB_TOKEN`。需要 `GITHUB_APP_ID`、`GITHUB_APP_PRIVATE_KEY` 或 `GITHUB_APP_PRIVATE_KEY_PATH`，以及 `GITHUB_WEBHOOK_SECRET`。非内部调用不能注入该上下文键。安装令牌请求未按本次 repo 进一步缩权，因此 App 安装范围应限定测试仓库；长任务令牌刷新未实现。
3. [A/channels/github.py](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/backend/app/channels/github.py) 的 `GitHubChannel.send` L85 仅记录日志；回复、push、PR 都要显式执行 `gh/git`。上游没有保证 `--draft`、禁止默认分支 push 或“测试通过才发布”的交付事务。Skill 提示词与两个报告工具本身不能强制这些门槛；Phase 4 必须验收写入控制和人工放行后再赋予写能力。
4. Skills 的当前用户目录为 `{DEER_FLOW_HOME}/users/{user_id}/skills/custom/`；`skills/public/` 与 legacy custom 仍参与发现，见 `skills/storage/user_scoped_skill_storage.py`。RepoPilot 仓库中的 Skill 不会自动被另一套 DeerFlow 部署发现，需要官方安装/部署步骤。`tool_groups` 只过滤配置工具，MCP 集合另行加入；Skill `allowed-tools` 也不是任意 shell 命令的参数级安全边界。
5. MCP stdio 在 Gateway 环境启动。[mcp/tools.py](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/backend/packages/harness/deerflow/mcp/tools.py) L476 的 `_make_session_pool_tool` 已按 user/thread 复用会话，并把 cwd 固定在线程 workspace。AIO 的 `_get_thread_mounts` 把同一目录映射到 `/mnt/user-data/workspace`。因此优先让 RepoPilot MCP **只解析工作区内的相对报告路径**，拒绝越界、符号链接逃逸和超限文件；测试、安装依赖、git diff 采集仍在 Sandbox 内执行。Docker 内还需能导入独立工具包，不能只把包放在 WSL 目录就认为接入完成。
6. AIO 带凭据的命令经 `_execute_with_env` → `_run_bash_exec` L512 → `client.bash.exec(env=..., hard_timeout=600)`，使用临时 session；连续调用不保留 cwd/激活环境。`execute_command(timeout=...)` 当前忽略这个参数，不能把宿主 LocalSandbox 的超时选项当成 AIO 已生效；短测试预算须另行实测，不修 Runtime。
7. 原生 delivery 去重有期限；默认内存去重不跨重启，忙时 follow-up 缓冲有单进程限制。继续采用单 Gateway，不处理 #4121；Phase 4 创建 PR 前仍需检查相同 Issue/任务分支是否已有 Draft PR。

**GitHub App 最小权限依据（公开文档核对，尚未创建 App）。** Issue 事件需要 `Issues: read`，只读仓库访问按需给 `Contents: read`；进入 push/Draft PR 阶段才考虑 `Contents: write`、`Pull requests: write`。评论接口接受 `Issues: write` 或 `Pull requests: write`，应按实际调用取所需权限；不申请组织管理权限或 Workflows 权限。依据：[Webhook 事件](https://docs.github.com/en/webhooks/webhook-events-and-payloads#issues)、[Git 访问权限](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app#choosing-permissions-for-git-access)、[创建 PR](https://docs.github.com/en/rest/pulls/pulls#create-a-pull-request)、[创建评论](https://docs.github.com/en/rest/issues/comments#create-an-issue-comment)。REST 权限核对不能替代沙箱内实际 `gh` 的鉴权验收。

## 3. 自己实现什么、阻塞项和下一步

**最小增量可走上游扩展点，当前未发现必须修改核心的理由。** 以下均为后续拟建，本轮没有新增 Python 业务代码、Skill 或配置：

| 拟建文件（`packages/repopilot-tools/src/repopilot_tools/` 下） | 自己实现的职责 |
| --- | --- |
| `server.py` | 注册 `summarize_test_results`、`inspect_diff`；MCP 参数/异常映射、报告路径校验；不在 Gateway 执行目标仓库脚本 |
| `test_report.py` | 先解析 pytest 原生 JUnit XML及执行记录，提取通过/失败/跳过、退出码、有限错误摘要和证据引用；空/缺失/截断报告不能报 PASS |
| `diff_report.py` | 解析 Sandbox 采集的 Diff/文件清单，计算文件数和增删行，识别 `.env`、密钥、锁文件、二进制与越界改动；明确只是确定性检查 |
| `schemas.py` | Pydantic 输入输出、状态和错误码、长度/大小约束；为正常、失败、越界、敏感变更建立独立 fixture 与单元测试 |

另需 Phase 2 的 `skills/repopilot-issue-to-pr/SKILL.md` 与 Custom Agent 配置，规定任务分流、工具顺序、验证门槛、失败报告、PR 模板；它们不计作 Python 原创模块。Phase 4 的授权/发布门槛应先选择最小可验证的人工放行或受限发布工具方案；如确需额外 Python 模块，届时单独说明，不在本轮扩建架构。

| 阻塞或待验证项 | 当前结论 |
| --- | --- |
| Phase 1 启动与工具调用 | Docker 可用且镜像已存在，未发现硬件/daemon 硬阻塞；现存镜像能否启动、模型 API/Tool Calling、AIO 挂载与容器内 `git/gh` 均未验收 |
| WSL 原生开发 | 缺 uv/Node/pnpm/nginx，虚拟环境为空；若坚持原生路径需要另行授权安装。优先 Docker，当前无需补齐宿主整套工具链 |
| Phase 2 GitHub | 通道未开启；App/安装绑定、Webhook secret、公网 HTTPS 回调、仓库权限未核验。未检查外部账号，不把这些写成“确定不存在” |
| Phase 3 工具接入 | 确定 stdio 工作区与包部署方式，并验证隔离、报告真实退出码及超时；上游有接口不代表本机组合配置已成功 |
| Phase 4 写入 | 严格触发授权、Draft-only、禁止默认分支/merge、失败不发布、PR 幂等均需验收；不能只靠 Prompt 或 `allow_authors` 宣称已安全 |
| 工程记录 | RepoPilot 为 `main`、尚无提交，现有文档和 `.gitignore` 未跟踪；本轮仅新增本文，上游源文件未改动 |

**实际检查与测试状态：** 执行了本地 Git 版本/状态检查、工具版本检查、`docker info`、`docker ps -a`、`docker image ls`，以及源码/Schema 静态阅读与本文链接校验。未运行上游单元测试、模型调用、容器冒烟、Webhook 或 PR 端到端测试；没有这些测试通过的结论。可供后续选用的现成测试有 `backend/tests/test_github_{webhooks,triggers,dispatcher,token_plumbing}.py`、`test_lead_agent_skills.py`、`test_aio_sandbox.py`。未安装软件、未启动服务、未修改 GitHub 资源；联网仅只读查阅 GitHub 官方文档。

**建议下一步只做一个验收：** 获准进入 Phase 1 后，在 `upstream/deer-flow/` 用官方 `make docker-start` 启动单实例，完成“一次模型工具调用，在 AIO Sandbox 创建并读回无害临时文件”的原生冒烟验证，记录 Run 和真实工具结果。启动可能构建镜像/同步容器依赖，并开放本机入口；执行前确认 loopback 绑定和单 Worker。已有固定 `1.11.0` 镜像，勿机械重跑 `make docker-init`：本 Tag 的 `scripts/docker.sh::init` 写死拉取 `latest`。本轮按用户授权范围停在 Phase 0。
