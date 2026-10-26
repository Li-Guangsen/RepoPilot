# RepoPilot

> 基于 Python + DeerFlow 的 GitHub 智能研发协作 Agent

RepoPilot 是一个基于字节跳动开源项目 [DeerFlow](https://github.com/bytedance/deer-flow) 二次开发的 GitHub 智能研发协作 Agent。项目以 Python 为主要开发语言，结合 LangGraph、MCP、Agent Skills 和 Docker Sandbox，探索从 GitHub Issue 分析到代码修改、测试验证和 Draft PR 创建的研发流程。

项目面向自有、已授权的测试仓库，首期限定文档修复、单元测试补充和可复现的小 Bug 修复：

```text
GitHub Issue → Agent 分析 → Sandbox 修改文件 → 运行测试
             → 测试与 Diff 报告 → Draft PR → 人工审核
```

**当前状态：开发中（Work in Progress）。** 已完成 Phase 0 的上游源码定位与 WSL2 环境调查，尚未完成运行验收或 Issue-to-Draft-PR 闭环。以下功能为开发规划。

## 项目功能

- **Issue 分析与任务分流**：提取需求和验收条件，制定执行计划；对模糊或超出支持范围的任务请求澄清或终止。
- **仓库分析与代码修改**：通过 Agent 工具调用检索仓库，在受控工作区内完成小范围修改。
- **沙箱测试与结果汇总**：在 Docker Sandbox 中运行测试，记录真实退出码和测试结果，生成结构化报告。
- **Diff 检查与 Draft PR 交付**：汇总变更，检查敏感文件与越界改动；满足验证和授权条件后创建草稿 PR，附上变更说明、测试证据与限制。
- **Agent 能力扩展**：通过 Custom Agent、Skills 和独立 Python MCP 工具定义 RepoPilot 的任务流程与报告能力。

所有交付均保留人工审核，不自动合并 PR。测试失败或权限、任务边界不明确时，输出失败原因，不将任务标记为成功。

## 技术栈

| 类别 | 技术与用途 |
| --- | --- |
| 开发语言 | Python 3.12+ |
| Agent 框架 | DeerFlow 2.x、LangGraph，复用 Agent Runtime 与编排能力 |
| 后端与界面 | 复用 DeerFlow 的 FastAPI Gateway 和 Web UI |
| 工具与扩展 | MCP、Agent Skills、Custom Agent、Tool Calling |
| 代码执行 | Docker Sandbox，隔离执行仓库命令与测试 |
| 平台集成 | GitHub App/API、官方 GitHub Webhook、沙箱内 git/gh |

## RepoPilot 的开发增量

DeerFlow 提供运行时、模型调用、工具分发、Webhook、Skills/MCP 接入、Sandbox 和 UI。RepoPilot 计划在这些能力上实现：

- **专用 Agent 与任务 Skill**：定义任务范围、工具使用顺序、验证门槛、失败处理和 PR 模板。
- **Python 测试报告工具**：优先解析 pytest/JUnit XML，输出测试统计、退出码、错误摘要和证据引用。
- **Python Diff 检查工具**：统计变更文件和增删行，识别敏感路径、二进制文件及越界改动，生成可审核摘要。
- **测试与演示材料**：建立成功、失败、模糊需求和越权任务等受控用例，记录真实执行结果。

上述 Python 工具和业务流程尚待实现，不将上游现成能力计作 RepoPilot 原创成果。

## 开发进度

- [x] 固定 DeerFlow `v2.1.0` 源码基线，完成 WSL2 与 Docker 环境检查
- [x] 完成 Webhook、Custom Agent、Skills、MCP、Sandbox 和 Runtime 的源码梳理
- [x] 已有模型 API 与 Docker Sandbox 本地配置，运行有效性待验收
- [x] 建立项目仓库、阶段计划与上游源码地图
- [ ] 完成 DeerFlow 启动、模型 Tool Calling 和 Docker Sandbox 运行验收
- [ ] 接入受控 GitHub 事件与 RepoPilot Custom Agent，先验证只读流程
- [ ] 开发专用 Skill、Python MCP 工具及单元测试
- [ ] 完成 Issue → Sandbox 修改与验证 → Draft PR 闭环
- [ ] 完善受控任务评测、部署说明与演示记录

阶段划分与验收标准见 [项目计划](docs/PROJECT_PLAN.md)；当前环境检查结果及待验证项见 [上游源码地图](docs/UPSTREAM_MAP.md)。

## 文档

- [项目计划与阶段验收](docs/PROJECT_PLAN.md)
- [上游源码与环境地图](docs/UPSTREAM_MAP.md)
- [本地开发约束](AGENTS.md)

## 上游归属与致谢

本项目基于 [bytedance/deer-flow](https://github.com/bytedance/deer-flow) 二次开发。感谢 DeerFlow 项目及其开源社区提供的技术基础。

- 固定基线：`v2.1.0`
- Commit：`345f08be00c8a9495079b732a39b46aa9af1584e`
- 上游许可证：[MIT](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/LICENSE)

上游源码独立放在本地 `upstream/deer-flow/`，不随本仓库提交。真实配置、模型密钥和 GitHub 凭据同样不提交。

`RepoPilot` 为暂用项目代号，不宣称名称独创。
