# 开发约定

## 文档与范围

- 开发前阅读 `hld/` 对应分册、总册相关接口和大纲条款。大纲是规则规范；最新职责调整按总册 §7.3 与分册说明执行。
- `README.md` 是运行入口，`docs/roadmap.md` 记录阶段进度，`dev_memory/` 记录每项功能的开发与接手信息。
- 新建的可运行代码必须有最顶部 AI header，包含 AI 辅助生成、工具/模型 `Codex(GPT)` 和 Asia/Shanghai 时区的 `yyyy-MM-dd HH:mm` 日期；必须先行的 shebang/encoding/license 除外。修改旧文件不主动补 header。纯 JSON 等不支持注释的文件除外。
- 新增函数使用注释或 docstring 说明作用。复用已有封装和组件，保持项目风格。

## 交付流程

- 每项功能、模块或重要设计使用独立功能分支，验证后创建 PR 并通过 PR merge；禁止直接向 main 提交或推送。功能分支上的 commit 用于创建 PR。
- 每项工作在根目录 `dev_memory/` 创建详细 Markdown；同一功能后续迭代更新原文件。至少包含背景、HLD 依据、方案、核心文件、接口/迁移影响、验证命令与实际结果、限制、接手顺序和 PR 信息。
- 在合并前查看 PR 的实际 CI 结果；失败必须修复。权限或外部服务阻塞时如实报告，不把未执行的检查标为通过。
- PR 描述使用 `.github/pull_request_template.md` 的信息要求，不包含凭据、来源正文和未脱敏标识。

## 检查与架构

- `make setup` 安装锁定依赖，`make check` 执行现阶段门禁。新增依赖须说明用途、选型依据与 HLD 变更级别。
- `make test-integration` 必须显式指定专用的 `TEST_DATABASE_URL`；不得指向生产或有价值的数据库。
- 保持 `pyproject.toml` 中的 import-linter 边界。静态导入检查不等于完成网络隔离。
- 应用提交路径用 psycopg 显式 SQL；SQLAlchemy 仅供 Alembic 迁移使用。
- 未实现能力不得返回成功空结果；桩化与 unsupported 的完整契约在 M0 后续阶段落地。
- 日志、追踪、span 属性不包含来源正文、SQL 参数、模型提示词或密钥。
- shell 工具调用遵循用户指定的 `/Users/kchankc/.codex/RTK.md`；CI/Makefile 不依赖个人机器上的 rtk。

## CodeGraph

仅当项目根目录存在 `.codegraph/` 时，理解或定位代码先使用 CodeGraph；不存在时不建立索引，使用 `rg`。
