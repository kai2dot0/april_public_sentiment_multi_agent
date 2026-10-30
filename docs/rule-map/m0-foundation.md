# M0-01 规则—实现—验证映射

这里只列当前交付触及的条款，未实现的规则明确保留；不能据此宣告整个条款或 M0 验收完成。

| HLD / 大纲依据 | 当前实现 | 验证 / 状态 |
| --- | --- | --- |
| 大纲 §2.4、总册 §2.2：分层依赖 | 顶层包 + `pyproject.toml` import-linter | `make lint`；`test_import_linter_rejects_forbidden_dependencies` 含真实违规反例 |
| 分册二 §8：连接器不得直接网络 / feedparser | import-linter + `scripts/check_connector_imports.py` | `test_connector_submodule_bypasses_are_rejected`、`test_connector_url_parsing_is_allowed`；默认拒绝网络策略未实现 |
| 大纲 §2.2 底线 3：worker 不持库凭据与驱动 | 驱动导入约束 | 当前仅静态约束；worker 运行与凭据隔离尚未实现 |
| 大纲 §2.2 底线 8、分册二 §1：迁移链 | Alembic、`check_migrations.py`、pgvector 基线 | 单链、分叉反例、离线 SQL、拒绝 SQLite、真实 PostgreSQL 重复升级 |
| 分册二 §1：基础开发环境 | `compose.yaml`、`deploy/observability/` | `compose-check`、真实 core health、`smoke-infra`、`smoke-observability` |
| 大纲 §3.3、§4.12：遥测不含原文 | 未启用应用自动埋点；smoke 仅无内容的合成信号 | 不是业务遥测无泄漏的完整验收；未来埋点与日志必须补夹具 |
| 总册 §8.4：共享 CI | `.github/workflows/ci.yml` | PR / main 上执行本阶段检查；完整决策表、锁序、删除门禁待对应模块 |
| 总册 §9.2 H1–H17 | 尚无业务运行时、契约与删除能力 | 全部仍待对应模块实现和逐项提供证据 |
| 用户：每项工作 PR merge、dev_memory | `AGENTS.md`、PR 模板、阶段记录 | 以本 PR 及后续 GitHub merge 记录为准 |
