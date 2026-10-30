# M0-02F：worker cgroup、网络与可验证缓存清理

日期：2026-10-30（Asia/Shanghai）。工具：Codex(GPT)。分支：`feat/m0-worker-isolation-validation`。起点 main=`2b05440d5b0248b36472e3b849a90442873865ae`（PR #6 已合并）。用户要求继续下一项，本轮按 roadmap M0-02F 做真实机制验证。

## 1. 范围、设计依据和交付流程

依据分册二 §0.5 第 1 项、§5，总册 §2.4 角色表，以及大纲底线 2 / 3 的进程组网络、无存储凭据、只读文件系统、有界 tmpfs、cgroup 内存 / 墙钟硬终止与缓存可验证销毁。HLD 的“验证可行，或者给替代”是本轮选型依据，不把文档里的完整后续监督器功能冒充本轮已完成。

本轮覆盖：spawn、实际 cgroup 限额、真实 OOM 与同级隔离、setsid 后代强制终止、角色网络内核执法、受控出口白名单夹具、清理前的停止证据、缓存残留 / 无法核对反例。没有注册 API、权威退役、fencing fail、任务重派幂等、撤回前沿 / 缓存临界区或真实连接器部署；这些在 M0-05 / M0-06 / M1 接线，不能据此宣布完整 M0 硬出口通过。

独立分支提交 → PR → 核对最终提交实际 CI → merge；不直接向 main 提交。无子 Agent，无 .codegraph/，未索引；遵循 RTK shell 前缀、新代码 AI header / 函数说明和 ponytail 原则。复用 Docker / pytest / 标准库，无新增 Python 依赖。

## 2. 实测环境与选型

只读检查确认本机 Docker Desktop 29.1.3、LinuxKit 6.12.54、aarch64、cgroup v2。普通非特权容器中 /sys/fs/cgroup 只读，进程视图 `0::/`，创建子组失败。没有为了跑原推荐方案而授予 privileged / SYS_ADMIN、挂载宿主 cgroup 或 Docker socket。

选择替代原型：**每 CPU worker 独立受限容器，可信插件外控制端通过 Docker daemon 设置 / 终止其 cgroup**。容器内只有合成引导 + spawn 任务，真正控制能力在宿主试验驱动，不下发给 worker。每个 worker 的 OOM 范围独立，也不需要让插件写 cgroup 控制文件。

按总册 §11.1 二级实现变更信息记录到 ADR 0007 / PR：推荐路径替代及 OS 工具选择，含动机、影响、验证与回退。无 contracts / spi、提交协议、事务 / 锁序或 HLD 原件修改，无 A/B/C 裁剪；不虚构其他人工会签。正式监督器部署位置、受限 IPC 和服务身份仍待后续落地，不能把宿主试验驱动当作已完成的监督器角色隔离。

镜像：官方 Python 3.12.12 slim-bookworm，多架构索引摘要 `sha256:593bd06efe90efa80dc4eee3948be7c0fde4134606dd40d8dd8dbcade98e669c`。Dockerfile 只复制 policy.json / probe.py / fixture.py，无项目代码或 .env；Bookworm apt 安装 iptables，本机 1.8.9-2。apt 层不承诺逐字节重建，实际平台镜像 ID 随构建记录；最终本地镜像为 `sha256:48a18795a4a9c793910fe1d9f5882a779fbbff7777ccb70f0c3343bbc5b03810`。可信控制端将 tag 解析为不可变镜像 ID，worker 构造拒绝 tag / 非 ID 参数。

## 3. 核心文件、控制面与准入

- `adapters/isolation_probe.py`：固定错误 Docker CLI、所有权核对、限制参数、DockerWorker 生命周期、网络启动闸门、墙钟终止、缓存和本地证据。仅供可信试验控制端，不是插件或公开 SDK API。
- `tests/support/isolation.py`：IsolationLab 创建本次独有 internal bridge 和合成端点，管理 worker / helper；与现有 compose 项目网络和数据卷完全分离。
- `deploy/isolation/Dockerfile`：固定基础镜像；UID 65532、固定线程环境；iptables 可执行文件存在但 worker 没有修改规则的能力。
- `deploy/isolation/policy.json`：角色允许端口唯一来源，宿主和镜像共同读取，JSON 无注释 header。
- `deploy/isolation/probe.py`：可信 gate、spawn、合成资源 / 网络 / OOM / setsid 后代 / 缓存探针；固定 JSON 输出，不加载真实插件。
- `deploy/isolation/fixture.py`：活端点、UDP、HTTP transport / proxy / source 夹具；关闭 URL 日志，无宿主端口发布。

资源名均带随机 UUID，所有者标签为 april.isolation-probe=<随机实验 UUID>。所有删除操作先核对标签，不做 prune / 按前缀批量删除。网络准入除所有者匹配外，强制 Driver=bridge、Internal=true，再使用不可变网络 ID；因此不能误将 NET_ADMIN helper 加到 host 或非内部网络。镜像也必须是 sha256 ID。13 项 unit 中有标签不同禁止 rm、即使同标签也拒绝 host / none / 外部 bridge、错误不回显输出、预算非法等反例。

## 4. 内核资源、spawn 和停止机制

worker / 缓存 helper 使用 `restricted_options()`：UID 65532、read-only、cap-drop ALL、no-new-privileges、private cgroupns、memory=memory-swap=128 MiB、pids=32、cpus=0.5、/dev/shm 1 MiB、/tmp 8 MiB（noexec / nosuid / nodev / owner65532 / mode700）。/cache 是专有 named volume；没有宿主 bind mount、数据库变量或 Docker socket。

实际内核值：memory.max=134217728、memory.swap.max=0、pids.max=32、cpu.max='50000 100000'、CapEff=0、NoNewPrivs=1、UID=65532；root 写本来允许该用户写的目录仍 EROFS(30)，/tmp 执行拷贝 ELF 得 EACCES(13)，写满得 ENOSPC(28)。BLAS 四种线程环境均1，但没有用真实 numpy / 模型做负载评测。

spawn 验证：父引导在运行时设置全局标记，打开标记文件 FD 并主动设 inheritable；spawn 子任务看不到父全局，也没有对应标记 FD。没有借用真实 PG 连接做此实验。所有容器没有 DATABASE_URL / PGPASSWORD / MODEL_API_KEY，插件不继承宿主数据库连接。

OOM：子任务分配最多 512 MiB，128 MiB cgroup 实际触发 memory.events oom_kill≥1，子任务 exitcode=-9，容器引导以失败结束；同级 worker 容器保持运行。不是用 Python MemoryError、手动 exit137 或 kill 注入冒充 OOM。

墙钟：wait 接受 `(0,60]` 秒，先校验 bool / NaN / Infinity / 超大整数等非法预算；docker wait 的客户端超时后，必须另外 kill 容器，再 wait，并检查 engine Running=false / Pid=0。测试任务忽略 TERM，再创建 start_new_session=True 的孙进程；实测 session=孙 PID 但 /proc/<pid>/cgroup 仍0::/。TERM 后容器仍活，0.3 秒预算触发真实 SIGKILL / 完整停止，docker top 不再可用。依赖 PID 命名空间 init 终止语义覆盖所有后代，不依赖 POSIX 进程组未变。

## 5. 网络规则与出口夹具

启动顺序：create / start 可信 gate → 等 gate_ready → 可信临时 helper 加入**该 worker** netns → IPv4 / IPv6 OUTPUT 默认 DROP，按角色精确目的 IP / TCP 端口 ACCEPT → 核对固定规则安装结果 → 释放 /tmp/release → spawn 合成任务。安装失败直接关闭 worker，任务不会在未设策略时启动。

helper 仅获得 NET_ADMIN，不加 SYS_ADMIN / privileged / host network；它不与 worker 共享 PID 或 cgroup。worker 始终 cap0 + 非 root + NNP，实际重设 iptables 失败。没有通用 loopback / DNS / UDP 放行。trusted Docker 控制端的身份边界是本地可信操作者，尚无开放控制 RPC。

夹具端口：18001 runtime，18002 撤回登记，18003 evidence，18004 knowledge，18005 gateway，18006 transport，18080 source，18081 proxy，15432 / 16379 为 DB / Redis **活的合成拒绝目标**，UDP18053作活 UDP 对照，不是项目真实数据库。

Agent 放行18001–18005；connector 放行18001/18002/18003/18006；normalizer 放行前三；supervisor 放行前二。先验证所有端点可用，再按角色核对每个端口与UDP。规则是内核执法，不靠 HTTP_PROXY 变量或 import-linter。运行期用例没有公网访问，禁止目标与允许目标在同一可达测试网段，不用端口未监听冒充拒绝；镜像构建需要下载公共基础镜像与 APT 包。

connector 不能直接连 source / proxy，只能经 transport → proxy → source。域名夹具仅允许精确 http://allowed.source.test:18080；固定映射到合成来源，不按输入解析 DNS，不跟随重定向。七个 URL 状态为 [200,403,403,403,403,403,403]：合法域、其他域、IP、相似后缀、user@、空 userinfo @、错误端口。审查补充了空 userinfo 反例，避免把空字符串误作没有 userinfo。

这不是生产 HTTPS / CONNECT / WebSocket 代理、合规清单加载、服务鉴权或完整 DNS / TLS 安全验收。四角色网络规则已物理验证，但宿主 Docker 控制端不冒充“受限 supervisor 容器”；后续需明确最小控制 IPC、role 身份与正常端点策略更新。

## 6. 缓存清理和完成证据

每 worker 单独 named volume，镜像目录预置 owner65532 后由 Docker 初始化；任务写合成内容和指向 /tmp/keep 的 symlink。不能因为 worker 退出就宣布清理完成。

流程：`require_stopped()` → 无网络 / 无 capability helper 在专有卷清理 → 再用**另一个 helper** inspect 空目录。symlink 只 unlink，不跟随；cleaner 的 /tmp/keep 标记必须保留。独立验证进程既防旧任务继续写，也区分目录仍有内容与无法核对。

三种负例均通过：活进程 → process_still_present；退出但内容残留 → cache_not_empty；chmod000 无法访问 → docker_failed，不能生成完成证据。测试 finally 恢复权限再清理，最后 cleanup_evidence 返回 process_gone / cache_empty 和 `scope=local_probe_only`。

本地证据不是 report_retired，不更新进程前沿表、租约或任务状态，也不是字节安全擦除承诺。正式登记、重启查 list_unretired、失联副本、fencing fail / 重派幂等、撤回广播及连续前沿须在后续实现。试验 teardown 会销毁自己记录的卷，不能将 teardown 成功倒推为业务退役成功。

## 7. 验证、执行与 CI

实际完成：

- `make isolation-build`：固定摘要镜像成功构建；只涉及本次测试目录。
- `TEST_ISOLATION_ENABLED=1 make test-isolation`：最终镜像 10 项真实 Docker 用例通过，约27秒；包含四角色 / 受控出口 / spawn和实际限额 / 整树墙钟 / OOM同级隔离 / 缓存反例。
- 最后补60秒预算上限后执行 `make check`：353 passed、39 个集成用例按 marker 排除；总收集392项（353 unit +24 PG +5 Kafka +10 isolation）。8 条导入契约、迁移链1revision、锁文件和3份 schema均通过；最终提交在 CI 重新跑真实隔离。
- 实验后只读核对，带 april.isolation-probe 标签的容器 / 卷 / 网络均0，没有全局清理或改动原项目基础设施拓扑。
- 6 个新 Python 与 Dockerfile 的 header / 日期格式、函数说明、Python 语法和 `git diff --check` 均通过。

命令：

```bash
rtk proxy make setup
rtk proxy make isolation-build
rtk proxy env TEST_ISOLATION_ENABLED=1 make test-isolation
rtk proxy make check
```

缺少显式开关 / Docker / cgroup v2 / 所需内核能力会失败，不跳过。Make 的 test-integration 明确排除 kafka / isolation，保持 PG 入口含义。独立 CI isolation job 按 build → test 顺序运行；checks / infrastructure 两个原 job 继续执行所有门禁。合并必须三项均通过，不能只看原两项。

设计见 ADR 0007，规则映射 `docs/rule-map/m0-worker-isolation.md`，入口 README / roadmap / memory索引已更新；上一模块 memory 补 PR #6 最终合并证据。交付 PR 创建后补链接，最终 CI / merge SHA以PR记录为准，不把未执行项记为通过。

## 8. 接手、限制与回退

先读本 memory / ADR，然后 adapter 的准入与所有权、deploy 探针 / 角色表、IsolationLab 和10项真实测试，再读13项unit与CI。六个新 Python 和 Dockerfile 顶部有 AI header，JSON不加注释，函数均有说明。

下一项为 M0-02G：Jetstream / AppView 协议、游标、缓存与许可核实。正式阶段接本隔离原型时，须保持不可变实例身份与标签、策略先于任务、插件无控制端权限、资源限制与停止 / 缓存核对分离，并实现任务 / lease 绑定、fencing fail、控制面持久恢复和真实 report_retired。普通 cgroup 内委派未验证成功，不应回改为可写宿主 cgroup；若更换部署方案须重新做真实验收。

没有实际插件 / CPU模型负载、生产TLS出口、完整监督器服务、进程重派或业务状态变化；探针128MiB /0.5CPU预算不是LoadProfile。控制端永久丢失 / Docker不可达时没有凭空清理成功，正式系统应按待核验副本处理；本原型会报错。多worker每容器的隔离路径、委派no-internal-process协议也未实现，本轮选择另一条可行路径。

回退无需数据库或数据迁移：撤销adapter、测试镜像 / 入口与CI job；只清理本次有据可核对的资源，不动现有卷。真实业务集成后不得无审查关闭内存 / 网络限制、放宽host网络或改为单PID终止来规避失败。
