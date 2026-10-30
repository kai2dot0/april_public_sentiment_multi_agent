# ADR 0007：M0-02F worker 隔离与可验证清理原型

日期：2026-10-30。依据：分册二 §0.5 第 1 项、§5；总册 §2.4；大纲 §2.2 底线 2 / 3、worker cgroup 内存与墙钟约束、缓存可验证销毁。

## 实测约束与替代方案

本机 Docker Desktop 29.1.3、LinuxKit 6.12.54、aarch64 使用 cgroup v2。普通非特权容器的私有 cgroup 根为 `0::/`，挂载只读，不能创建子组。HLD 推荐的“监督器移入叶子组、启用 memory 控制器、为 worker 建子组”不能在这个默认部署方式下直接执行。本轮没有授予 privileged、SYS_ADMIN 或挂载宿主可写 cgroup。

采用 HLD §0.5 允许验证的替代：**每个 CPU worker 使用独立受限容器，由可信控制端经 Docker daemon 管理整个容器 cgroup**。容器中的微型引导仅等待网络闸门并 spawn 合成任务；真正的 Docker 控制权留在插件外。OOM 隔离到单 worker，墙钟终止容器 PID1 / PID 命名空间，包含后代新 session。无需把 cgroup 写权或 Docker socket 下发给 worker。

这是二级实现变更信息（推荐路径替代、引入 OS 规则安装工具），无 contracts / spi、任务提交、事务或锁序变化，无 A/B/C 裁剪，不修改 HLD 原件。当前验证替代方案的物理机制，不把它当作正式监督器 / 进程注册接口已交付。M0-05 / M0-06 必须明确可信控制端的部署、最小 IPC 权限、身份和 fencing fail 接线；不能向插件暴露本适配器或任意 Docker API。

## 固定探针镜像和资源边界

`deploy/isolation/Dockerfile` 使用官方 Python 3.12.12 slim-bookworm，固定多架构摘要 `sha256:593bd06efe90efa80dc4eee3948be7c0fde4134606dd40d8dd8dbcade98e669c`。只复制三份明确的探针文件，没有项目源码、.env 或凭据；无新增 Python 依赖。iptables 来自 Bookworm 仓库，本机版本 1.8.9-2；apt 层不宣称逐字节可重建，实际镜像 ID 和 CI 是验证依据。

可信控制端 `adapters/isolation_probe.py` 只接受不可变镜像 ID、专有标签匹配的内部 bridge 网络；网络改用 inspect 得到的不可变 ID，拒绝 host / none / 非内部 bridge。随机所有者标签在删除容器 / 卷 / 网络前逐项核对，不做全局 prune 或按前缀批量删除。

worker 参数与实际内核验证：

| 项 | 当前探针值 / 结果 |
| --- | --- |
| user / capability / 提权 | UID 65532；CapEff=0；NoNewPrivs=1；不能改 iptables |
| 内存 / swap | memory.max=134217728；memory.swap.max=0 |
| CPU / PID | cpu.max=`50000 100000`；pids.max=32 |
| 根文件系统 | read-only；对本来可写的用户目录写入失败 EROFS |
| tmpfs | /tmp 8 MiB、noexec/nosuid/nodev；实际执行 EACCES、写满 ENOSPC；/dev/shm 1 MiB |
| 缓存 | 每 worker 一个专有 named volume，只挂载 /cache |
| BLAS | OPENBLAS / OMP / MKL / NUMEXPR 线程变量均 1；未做真实 BLAS 负载测试 |
| 凭据 / 管理接口 | 不继承 DATABASE_URL / PGPASSWORD / MODEL_API_KEY；无 Docker socket |
| 墙钟 | 探针调用预算 `(0,60]` 秒；CLI 超时后另行 kill / wait 实际容器 |

这些值是合成实验预算，不是模型、来源或生产 LoadProfile 的资源承诺。

## 网络执法与出口链

任务尚未启动时，可信一次性 helper 只加入该 worker 的私有 netns，获得 NET_ADMIN 安装 IPv4 / IPv6 OUTPUT 默认 DROP。只对指定夹具 IP 的固定 TCP 端口放行；没有通用 loopback、UDP / DNS 或外部路由放行。安装成功前不释放任务闸门，失败即清理 worker。worker 自身非 root、无 capability，不能重置规则。

角色表唯一来源 `deploy/isolation/policy.json`，宿主和镜像共同读取：

| 端口 | 合成角色端点 |
| --- | --- |
| 18001 / 18002 / 18003 | runtime / 撤回登记 / evidence |
| 18004 / 18005 / 18006 | knowledge / gateway / transport |
| 18080 / 18081 | 来源夹具 / 出口代理夹具 |
| 15432 / 16379 / UDP 18053 | 活的 DB / Redis / UDP 拒绝对照；不是本项目真实数据库 |

Agent 只放行 18001–18005；connector 放行 runtime / 登记 / evidence / transport；normalizer 放行前三者；supervisor 只放行 runtime / 登记。测试先证明所有合成目标实际监听，再验证允许集合与拒绝集合。所有端点位于专有 internal 网络，没有宿主端口映射、现有项目网络或公网流量。

connector 不能直连来源或代理，只经 transport → proxy → 固定合成来源。代理只接受精确 `http://allowed.source.test:18080`，不按用户输入解析 DNS，不跟随重定向；拒绝其他域、IP、相似后缀、非空或空 userinfo 及错误端口。**这是 HTTP 域名白名单夹具，不是生产 HTTPS / CONNECT / WSS 代理、DNS 重绑定完整防护或实际合规清单加载器**。

“supervisor”角色容器只验证该网络规则；可信 Docker 控制端当前是宿主试验驱动，不冒充其正式受限部署。后续需保持控制面不对插件开放，并按总册角色表布置正式监督器网络身份。

## 进程和缓存完成证据

容器引导使用 multiprocessing spawn；父全局标记和刻意设为 inheritable 的标记 FD 均不继承。任务再以 setsid 创建孙进程，实测其仍在同 cgroup。忽略 TERM 时，控制端墙钟超限后 SIGKILL 容器 PID1，并 wait，再核对 Running=false / Pid=0；Linux PID 命名空间 init 终止会结束其中全部后代，不依赖 POSIX 进程组仍相同。

真实 OOM 实验分配超过 128 MiB：memory.events 的 oom_kill 增加，任务 exitcode=-9；另一个 worker 容器保持运行。不是手动返回 137 或用 MemoryError 冒充 OOM。

清理流程：先验证进程不存在，再用独立 network=none / 无 capability helper 清理该专有卷，随后用另一个 helper 重新检查空目录。符号链接只 unlink，不跟随；cleaner 内额外的 /tmp 标记保留。活进程、退出后残留缓存、缓存 chmod 000 导致无法核对，三者都不能返回完成。

`cleanup_evidence` 只给 `scope=local_probe_only` 的进程已终止 / 缓存空证据，**没有 report_retired 或权威清理完成状态**。注册、前沿、恢复对账、租约 / fencing、缓存临界区和实际撤回日志仍在 M0-05 / M0-06 / M1 接入。本轮只验证本地逻辑清空，不声称磁盘字节安全擦除。

## 验证、CI 与回退

10 项真实 Docker 用例覆盖四角色、活目标对照、受控出口、spawn / 实际 cgroup / 文件系统 / 权限、setsid 后代墙钟终止、OOM 不影响同级 worker、缓存三种负例与完成证据。13 项单元用例覆盖所有权保护、host / 非内部网络拒绝、错误脱敏和非法预算。最终 make check 为 353 项；新增 isolation CI job 另跑真实 Linux 内核实验，原 PG / Kafka / 可观测 CI 保持。

只在专用可丢弃开发 Docker Engine 运行 `make isolation-build`、`TEST_ISOLATION_ENABLED=1 make test-isolation`。缺少开关 / Docker / cgroup v2 / NET_ADMIN 所需能力时明确失败，不自动跳过，不用 mock 替代。实验只清理自己创建的资源；本机实验后带探针标签的容器、卷、网络均为 0。

回退不涉及数据库 / 数据迁移：移除适配器、探针镜像构建与测试入口即可。不能回退为未设内存或仅杀 docker wait / 单 PID 的实现后继续宣称隔离通过。真实业务接入前仍需控制面最小权限、持久身份、fencing、重启 / 丢失副本恢复、资源并发准入及端到端撤回验收。

资料：[Docker 资源约束](https://docs.docker.com/engine/containers/resource_constraints/)、[Docker run](https://docs.docker.com/engine/containers/run/)、[内核 cgroup v2](https://docs.kernel.org/admin-guide/cgroup-v2.html)、[Linux PID 命名空间](https://man7.org/linux/man-pages/man7/pid_namespaces.7.html)。
