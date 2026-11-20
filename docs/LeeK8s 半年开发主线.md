# LeeK8s 半年开发主线

## 项目目标

实现一个面向多租户的 Kubernetes 批处理调度系统。两个团队共享一个集群，并发提交数据处理或报表作业。LeeK8s 按租户的 CPU、内存配额和排队策略决定何时放行 Job，再通过 Kubernetes Scheduling Framework 扩展 Pod 的节点打分。用同一条链路学习 Kubernetes 的声明式 API、控制循环、调度框架和节点侧执行机制。

正式交付范围为单集群、原生 `batch/v1` Job、按 requests 记账的租户配额、FIFO 与小作业绕行、一个 `Score` 插件，以及可复现的策略对照和故障演示。第一版每个 Job 运行一个 Pod。配额借用、拓扑感知和组调度作为主线验收后的扩展。

## 作业链路与模块边界

1. **配置与提交**。`BatchQueue` CRD 管理租户的 CPU、内存配额。用户提交关联队列的暂停 Job。
2. **排队与准入**。策略计算根据当前快照选择候选 Job，控制器执行并确认对象写入。只有配额允许时才解除暂停，同时保留可查询的等待原因。
3. **Pod 调度**。原生 Job 控制器创建 Pod，`kube-scheduler` 执行过滤、打分和绑定。LeeK8s 的 `Score` 插件实验节点装箱策略。租户逻辑配额和节点实际可调度容量分别判断。
4. **执行与恢复**。kubelet 启动容器。控制器根据已放行且未结束的 Job 恢复配额占用，遇到 API 写入结果未知时先确认对象状态，再继续放行。

## 按月推进

| 时间 | 开发任务 | 学习与验收 |
| --- | --- | --- |
| 第 1 月 | 建立本地 kind 集群，用 `client-go` 跟踪原生 Job 和 Pod 的状态变化，记录从 API 提交到容器启动的链路 | 理解 API Server、List/Watch、`resourceVersion` 和 Informer 缓存。分别复现 Pod 无法调度与镜像拉取失败，能从对象状态和事件定位负责的组件 |
| 第 2 月 | 用 `controller-runtime` 实现 `BatchQueue`、FIFO 与配额准入 | 理解 CRD、`spec/status`、Reconcile、工作队列与幂等。配额满时新的 Job 保持暂停，旧 Job 完成后排队作业自动放行，等待原因可查 |
| 第 3 月 | 增加小作业绕行和队首大作业等待保护。实现占用恢复，覆盖重启和写入结果未知 | 在解除暂停成功但内存记账尚未更新时终止控制器，验证重启后的配额一致性。比较 FIFO 与绕行时小作业排队收益和大作业等待代价 |
| 第 4 月 | 实现节点装箱 `Score` 插件，通过 `schedulerName` 选择调度配置 | 跟踪 Filter、Score、Bind。用同一组小 Pod 填充多节点集群，再比较两种打分下是否还有节点能容纳大 Pod |
| 第 5 月 | 打通准入、调度和容器启动的状态观测。复现节点资源不足、镜像拉取失败和控制器崩溃 | 理解原生 Job 控制器、kubelet 与 CRI 的职责。能区分暂停排队、已准入但 Pod 未绑定、已绑定但容器未启动 |
| 第 6 月 | 固化实验负载与故障注入步骤，整理 README、架构笔记、模块设计和 Bug 复盘 | 分别对比准入策略与节点打分策略，用观察数据说明各自的收益、代价和适用条件 |

## 实验与验收

- **准入策略**。固定节点打分方案、负载到达顺序、资源请求和运行时长，对比 FIFO 与小作业绕行。记录 P95 准入等待、大作业等待和逻辑配额空闲时间。
- **节点打分**。固定准入策略和节点规格，对比默认打分与装箱插件。记录 Pod 调度等待、大 Pod 是否成功落位、各节点剩余的可调度 requests 容量。
- **故障恢复**。验证重复调谐、控制器重启、API 写入结果未知后的恢复。观察是否超额放行、是否多计占用，以及释放后等待作业能否继续推进。

## 进度提前时的扩展顺序

| 分支 | 下一步 | 验证重点 |
| --- | --- | --- |
| 优先扩展准入策略 | 配额借用 → 自然归还 → 公平共享或作业级抢占 | 用同一批负载比较配额闲置、原租户的等待时间和被抢占作业损失的计算时间 |
| 深入调度框架 | 拓扑感知 → 并行作业的组调度实验 | 观察 `Reserve` 和 `Permit` 超时、回滚，以及部分 Pod 放行对作业完成时间的影响 |
| 深入控制器可靠性 | 双副本单活 → 故障切换 → 大队列压测 | 在切换期间持续提交作业，测量接管时间并检查配额约束 |
| 扩展工作负载 | 并行 Job → JobSet 等多角色作业 | 明确多 Pod 资源请求、终态确认和配额释放时点 |

## 开发记录

每个里程碑记录需求场景、组件职责、关键决策、源码阅读入口和验收结果。每次 Bug 复盘记下复现条件、实际与预期状态、根因、修复和验证方法。第一篇可从《一个离线 Job 从提交到运行，经过了 Kubernetes 的哪些组件？》开始。

## 参考

- [Kubernetes API 概念](https://kubernetes.io/docs/reference/using-api/api-concepts/)
- [Kubernetes 控制器](https://kubernetes.io/docs/concepts/architecture/controller/)
- [Kubernetes Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [Kubernetes Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
- [Kueue 作业准入](https://kueue.sigs.k8s.io/docs/concepts/admission/)
