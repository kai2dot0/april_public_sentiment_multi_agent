# M0-02A 规则—实现—验证映射

本次只验证并发基础与内部守卫，不将实验表当作业务任务、控制对象或证据实现。

| 条款 | 实现 / 实验 | 验证与边界 |
| --- | --- | --- |
| 分册一 §0.5.1：四种行锁冲突 | `tests/integration/test_postgres_concurrency.py` | `test_row_lock_conflict_matrix` 16 对真实 PG 组合，独立期望表，55P03 |
| 大纲 §4.7：永久撤回取显式 FOR UPDATE | 同上 | `test_non_key_update_does_not_replace_explicit_update_barrier`；尚未实现删除状态转换 |
| 分册一 §2：SKIP LOCKED 候选领取 | 同上 | `test_skip_locked_only_skips_currently_locked_candidates`；同键顺序协议仍待开发 |
| 大纲 §4.7、总册 §7.4.1：锁序、不升级、层内顺序 | `runtime/core/lock_order.py` | `tests/unit/test_lock_order.py` 全链、相邻逆序、同层排序、6 种升级、强锁重复、缺失持锁与隐私反例 |
| 总册 §7.4.1：每事务独立上下文 | 守卫原型 | `test_required_locks_and_transaction_isolation`；正式跨组事务上下文未冻结 |
| 分册一 §4：命名 failpoint | `tests/support/failpoints.py` | `tests/unit/test_failpoints.py`；真实 Read Committed/死锁/顺序锁实验均使用 |
| 分册一 §4：死锁整事务回滚、重试 | 真实等待环与事务上下文外捕获 | `test_deadlock_rolls_back_and_retries_whole_transaction`，40P01、1/2 次尝试、最终两行各增 2、效果唯一 |
| 大纲 §4.3：取锁后用独立语句重新汇总 | Read Committed 等锁重读 | `test_read_committed_refreshes_after_waiting_for_lock`，旧值 0、新值 10；完整 blob 回收未实现 |
| 大纲 §9：SET LOCAL | 显式事务，提交/回滚两条路径 | `test_set_local_does_not_escape_transaction` |
| 全局锁序消除等待环 | 两事务按相同顺序、守卫 + 显式 SQL | `test_sorted_guarded_locks_complete_without_deadlock`；不等同于全部实际提交路径验证 |
| 用户：PR 与 memory | 原功能分支、阶段记录 | 结果与 PR 地址见 `dev_memory/2026-10-30-m0-02a-postgres-concurrency.md` |
