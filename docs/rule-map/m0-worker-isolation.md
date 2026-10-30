# M0-02F 规则—实现—验证映射

| 条款 | 实现 | 验证 / 边界 |
| --- | --- | --- |
| 分册二 §0.5.1：先验证容器内委派或替代 | 只读私有 cgroup 探测；Docker 每 worker 独立 cgroup 替代 | 写子组失败，memory.max / swap / CPU / pids 实值；无 privileged / 可写宿主 cgroup |
| spawn、不继承父连接 / 状态 | multiprocessing spawn 合成 worker | 父全局和 inheritable 标记 FD 不继承；未用真实 PG 连接 |
| cgroup 内存限制 | 128 MiB worker 容器 | 真实 oom_kill ≥1、任务 -9；另一个 worker 仍运行 |
| 墙钟必须终止实际整组 | Docker kill / wait / Running=false / Pid=0 | 忽略 TERM、setsid 孙进程同 cgroup、超时实际容器停止；不只杀 CLI |
| 底线 2 / 总册 §2.4：进程组默认拒绝、按角色放行 | 私有 netns 内核 v4/v6 OUTPUT DROP + 精确 IP/端口 | 四角色对活端点正反例；UDP 默认拒绝；host / 非内部网络准入反例 |
| connector 外部访问经 transport / proxy 白名单 | 固定合成 HTTP 转发链 | 合法域 200，其他域、IP、后缀、userinfo / 空 userinfo、错误端口均 403；无生产 TLS 代理声明 |
| 底线 3：只读文件系统 / 有界 tmpfs / 无驱动凭据 | UID65532、cap0、NNP、只读根、8 MiB tmpfs、专有缓存卷 | EROFS / EACCES / ENOSPC；无数据库环境或 Docker socket；worker 不能改 fence |
| 进程终止 + 缓存已清空才可给完成证据 | require_stopped → 清理 helper → 新 helper 核对 | 活进程 / 残留 / 不可核对均失败；符号链接不跟随；只给 local_probe_only |
| 正式 report_retired / fencing fail / 重启核对 | 保留到实际监督器和登记服务接入 | 当前没有权威退役或重派成功，不把机制验证当端到端 H 判据 |
| 用户：PR、header、dev_memory | 独立分支、6 个新 Python + Dockerfile header、详细接手记录 | CI 新增 isolation job，全部真实结果通过后合并 |
