# RepoPilot — 如何交给本地 Coding Agent

1. 将整个 `RepoPilot/` 文件夹放在你准备开发这个项目的工作目录。**不要把其中的 `AGENTS.md` 直接覆盖到官方 DeerFlow 仓库中。**
2. 在 VS Code / Codex / Claude Code 等本地 Coding Agent 中打开这个独立项目目录。
3. 让本地 Agent 先阅读 `AGENTS.md` 和 `docs/PROJECT_PLAN.md`。
4. 第一轮只做 **Phase 0：上游源码定位与环境核查**，先不要创建 GitHub App、修改环境密钥、安装全局工具或发 PR。
5. 上游 DeerFlow 源码建议单独克隆在RepoPilot/upstream相邻目录，固定官方 `v2.1.0` 版本，并以实际源码验证文档说法。

建议直接发给本地 Agent 的第一条消息：

> 我们现在开始 RepoPilot。你先完整阅读当前仓库的 AGENTS.md 和 docs/PROJECT_PLAN.md。这个项目是以 Python + DeerFlow 2.x 构建 GitHub Issue-to-Draft-PR Agent 的二开项目，目标是 Agent 开发实习，不是解决 DeerFlow 底层缺陷。严格执行 Phase 0：检查环境，定位 DeerFlow v2.1.0 的 GitHub Webhook、Custom Agent、Skills、MCP、Sandbox 和 Runtime 源码，生成 docs/UPSTREAM_MAP.md。第一轮不要写业务代码、不要擅自安装软件或操作 GitHub 外部资源。完成后汇报源码调用链、需要自己实现的 Python 模块、阻塞项和建议下一步。
