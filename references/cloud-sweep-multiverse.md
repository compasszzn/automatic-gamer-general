# 云端批量执行 — UOS Multiverse 模块（可选）

> SKILL.md 步骤 5 的 **cloud 模式**落地参考。Multiverse 不是必做步骤——local 模式（本地并发跑批）已能满足绝大多数迭代需求。当用户明确要求云端跑批、或需要超过本机并发能力的大规模 sweep 时，才接入本模块。

## 1. 何时使用

- 用户明确要求用 Multiverse / 云端跑批
- 需要的并发规模超过本机能力（本机 concurrency 受 CPU/内存/存档隔离限制）
- 需要真实的服务器环境（Linux Dedicated Server）验证 bot 行为

## 2. 接入前提与各引擎现状

游戏侧需要两件事：

1. **游戏支持以 Dedicated Server 形态运行**：打包为 Linux Dedicated Server 构建目标（Unity：先装对应构建支持模块 + Server subtarget，见 `engine-unity.md` §7；其他引擎的等价服务器构建方式见对应手册 §7）。游戏从环境变量读取全部运行参数（speed、level、auto、seed、run-mode、sequence 等），game over 后主动退出进程——这些是 cloud 模式的硬约定
2. **接入 Multiverse 平台**：当前 Multiverse 官方 SDK 是 Unity 版（Unity 游戏按 multiverse skill 的接入指引做）。其他引擎接入思路：SDK 的职责本质是"向平台上报进程生命周期与心跳"——可以按等价职责自行实现最小化上报（创建 allocation 后平台会以环境变量注入参数并启动你的服务器进程），或先与平台确认非 Unity 的支持方式

## 3. allocation 生命周期（sweep 脚本视角）

cloud 模式下每一局游戏，run_sweep 脚本按以下步骤执行：

1. 创建一个 Multiverse allocation，以环境变量形式传入游戏可执行文件的必要参数（speed、level、auto、seed 等）
2. 等待 allocation 启动并运行
3. 等待 allocation 运行结束
4. 获取 allocation 的 log，保存到本地
5. 抓取 allocation log 中的 battle log：以 `BEGIN_COMBAT_LOG_JSON` 开始、`END_COMBAT_LOG_JSON` 结束，中间是一个 JSON string——用该 JSON 替代本地文件形式的 battle log

同时，sweep 脚本把所有调用 Multiverse API 的 request/response 记录到 log，方便日后 debug。

## 4. 踩坑清单（实战总结，逐项实现）

- **allocation `create` 失败时进程退出码可能仍是 0**，错误藏在响应 body 的 `code` 字段——必须解析 body 检查成功与否，否则会对不存在的 allocation 空等到超时
- **`--allocation-ttl` 不能超过 game 的 allocationTTL**（默认 10m）——超限直接失败
- **对局结束后日志走 COS 文件**（logFileStatus: generating → finished），实时 `log` 字段会变空——要轮询等 finished 再下载 `.gz` 并解压，不要在 generating 阶段抓实时 log
- **COS 日志是 k8s 收集器包裹格式**，且可能把长 JSON 摊平进记录顶层——抓 `BEGIN_COMBAT_LOG_JSON...END_COMBAT_LOG_JSON` 时要兼容"已转义 / 未转义 / 被摊平"三种形态。建议按标志性字段定位 + 括号配平提取，而非假设它是干净的一行
- **battle log 必须设计成单行紧凑 JSON**（避免多行与收集器字段冲突），游戏内输出时用 `BEGIN_COMBAT_LOG_JSON` / `END_COMBAT_LOG_JSON` 包裹
- **replay 局跑在云端时**：sequence 文件需要随游戏一起进入 allocation（挂载或打包进镜像），等价性验证通常更适合 local 单实例跑

## 5. 相关资源

- Multiverse 的 allocation API 细节与 CLI 用法详见 **multiverse skill** 的 `references/allocation.md` 与 `references/cli.md`（不在本 skill 内重复维护）
- 本 skill 的 run_sweep 工程健壮性要求（进程锁、断点续跑、超时 kill、原子写入）对 cloud 模式同样适用，见 SKILL.md 步骤 5
