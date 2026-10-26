# RepoPilot

基于 **Python + DeerFlow 2.x** 的 GitHub Issue-to-Draft-PR Agent 二次开发项目。

项目面向自有、已授权的测试仓库，计划支持文档修复、单元测试补充和可复现的小 Bug 修复：

```text
GitHub Issue → Agent 分析 → Sandbox 修改文件 → 运行测试
             → 测试与 Diff 报告 → Draft PR → 人工审核
```

## 当前状态

**Phase 0 已完成：上游源码定位与 WSL2 环境调查。**

已核对 Webhook、Custom Agent、Skills、MCP、Sandbox 和 Runtime 的源码调用链，并记录扩展方式与待验证项。尚未完成运行验收，也没有实现 Issue-to-Draft-PR 闭环。

## 计划中的项目增量

- RepoPilot Custom Agent 与任务 Skill：限定任务范围，定义验证和交付流程。
- Python MCP 工具：标准化测试报告，生成 Diff 摘要并检查敏感或越界改动。
- 受控测试任务、失败路径与可复现的演示记录。

复用 DeerFlow 的 Agent Runtime、LangGraph 编排、GitHub Webhook、Skills、MCP 接入、Sandbox 和 Web UI。仅交付 Draft PR，由人工审核与合并。

## 文档

- [项目计划与阶段验收](docs/PROJECT_PLAN.md)
- [上游源码与环境地图](docs/UPSTREAM_MAP.md)
- [本地开发约束](AGENTS.md)

## 上游归属

本项目基于 [bytedance/deer-flow](https://github.com/bytedance/deer-flow) 二次开发，以上复用能力归属上游，不作为 RepoPilot 原创成果。

- 固定基线：`v2.1.0`
- Commit：`345f08be00c8a9495079b732a39b46aa9af1584e`
- 上游许可证：[MIT](https://github.com/bytedance/deer-flow/blob/345f08be00c8a9495079b732a39b46aa9af1584e/LICENSE)

上游源码独立放在本地 `upstream/deer-flow/`，不随本仓库提交。真实配置、模型密钥和 GitHub 凭据同样不提交。

`RepoPilot` 为暂用项目代号，不宣称名称独创。
