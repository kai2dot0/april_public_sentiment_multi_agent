# ADR 0010：第二批契约、共享投影与受限提交事务

- 日期：2026-11-03；状态：工程实现采用，未模拟 HLD 组织角色签字。
- 依据：总册 §3 / §7.4.1 / §8.2，分册一 §1 / §3 / §4，分册二 §2 / §5 / §6，大纲 §4.2 / §4.7 / §4.11 / §4.12。
- 级别：按总册 §11.1 三级影响记录；新增跨组提交、控制和事务接口，无线上消费者 / 迁移，无 A/B/C 裁剪。

## 决定与理由

第二批建立独立 41 根模型注册和快照，复用第一批 primitives / FetchPageWire；不重写第一批 29 模型或放宽 import-linter。contracts 与 spi 仍互为叶子，通过 SDK / runtime 分别绑定泛型。

语义投影由共享 contracts 层实现，精确路径白名单只排除外层 lease_token，其他语义全部参与；DB 代理键等排除类别目前不被包接收。原 JCS 实现移入 contracts 并保留 runtime 重导出，避免 SDK 复制算法或反向依赖。原生快照再校验先于 JSON 序列化，防嵌套可变值绕过。

Demo 选 HMAC-SHA256，运行时独占密钥并对完整任务 / 模式 / 配置 / 代次 / 根 / 控制签名。使用标准库 [hmac](https://docs.python.org/3/library/hmac.html) 的 compare_digest；部署随机密钥可用 [secrets.token_bytes](https://docs.python.org/3/library/secrets.html#secrets.token_bytes)。未引入新依赖。签名工具不冒充完整计算校验器，不提供公开端点。

事务内端口只借已有 psycopg 活动事务，不给连接工厂或提交方法。运行时上下文约束十三步、预选最强控制锁、根前置断言、调用范围和失效。PG facade 的真实原子性单独实验，不用 I5 / I7 / I9 / I10 / I21 Fake 通过业务集成。

## 取舍与未实现边界

- 没有创建任务 / 控制 / 证据业务表；DTO 和建议约束供后续真实 DDL 使用，不能把 declaration 当 PG 证明。
- 静态检查只辅助审查，当前业务适配器 0 文件；可信 Python / SQL 不等于恶意代码沙箱。
- LogicalWrite 用钉定 payload_schema 表达载荷，Claim / CloudEvents 等详细契约和实际 resolver 尚待首个闭环；prepare_material 只附加 V1 / V19 检查，不是完整 18 / 19 判定。
- source 永久墓碑歧义按更明确的 §4.12 文档 / 账号写屏障与来源临时读门控解释；SourceControlPolicy 固定比较基准。六维除 computation 外的名称未定义，保留版本化维度目录，不发明其余五维。
- I16 控制入口按总册在 M0 末完成，本轮没有控制状态转换服务、注册持久化、读可用性服务、预算 / CAS 或旧代次投影执行。

## 兼容、验证与回退

完整规范见 [第二批 v1](../contracts/second-batch-v1.md)，逐项 HLD 影响见 [C04 登记](../rule-map/m0-second-contracts.md)。第一批快照不变，旧 JCS 导入兼容；没有锁文件、CI 拓扑、数据库迁移变化。

契约 / 签名 / 守卫单元反例和真实 PG 同事务实验纳入现有门禁，实际结果见 [dev_memory](../../dev_memory/2026-11-03-m0-04-second-contracts.md)。回退撤销新增契约与引用，将 JCS 实现移回原路径并恢复 schema 检查组；没有业务数据变更。出现实际消费者后，任何字段 / 摘要政策 / 签名域改动需显式升级与迁移，不可原地改义。
