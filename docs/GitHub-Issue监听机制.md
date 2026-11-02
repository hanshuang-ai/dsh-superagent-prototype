# GitHub Issue 监听机制

> 本文讲解 dsh-superagent 是**如何监听 GitHub Issue（以及 PR、评论、CI）**的：它用什么方式接收事件、一轮轮询做了什么、一条评论如何被规范化成领域里的 `Trigger`、以及它如何做到不丢不重、不超速率。
>
> 阅读本文不需要先读其它文档；如需了解事件进入领域之后的去向（评估、授权、工单），可参阅 [001 领域模型](001-domain.md) 与 [007 架构](007-architecture.md)。

## 一句话结论

dsh-superagent **不使用 GitHub Webhook，也不监听任何端口**。它把 **agent 账号的 GitHub「通知收件箱」当作入站队列**，通过 **GitHub CLI（`gh api`）每隔约一分钟轮询一次增量**，把新到的 @提及、Issue、评论、CI 变化**规范化成不可变的 `Trigger`**，再喂进领域引擎。

这正是它要求 agent 必须是一个**真实 GitHub 用户账号**（而不是 GitHub App）的根本原因：GitHub App 没有这种「通知收件箱」可读，而 @提及之所以能唤醒 agent，正是因为 GitHub 会把 `@某账号` 转成该账号收到的一条通知。

## 为什么不是 Webhook

| 对比 | Webhook | 本项目的轮询 |
| --- | --- | --- |
| 是否需要公网端点 | 需要 | **不需要**，纯出站请求 |
| 是否要求 agent 是用户账号 | 否 | **是**（读通知收件箱） |
| 实时性 | 近实时 | 有延迟（默认约 60 秒） |
| 部署复杂度 | 要暴露服务、验签 | 只需一个登录好的 `gh` |
| 事件可靠性 | 依赖投递与重试 | 依赖游标 + 幂等，重启不重放 |

选轮询的取舍很清楚：**牺牲一点实时性，换取「无需公网入口、无需密钥验签、本地即可运行」**，符合项目「少打扰、易部署」的设计。

## 总体结构：一个常驻轮询循环

核心代码在 `src/sources/github.ts`。GitHub 作为 `SourceKind`（来源类型）被注册，随后启动一个常驻的 `watch` 循环：

```ts
// src/sources/github.ts —— watch()
async function watch(ctx: SourceContext, signal: AbortSignal): Promise<void> {
  let interval = 60
  let failures = 0
  await ctx.ready
  if (!ctx.configured()) { ctx.report('github', health('paused', 'setupPending')); return }
  while (!signal.aborted) {
    const config = ctx.config<GithubConfig>(SOURCE)
    try {
      if (config) interval = await pollRound(ctx, signal)   // 一轮轮询
      else ctx.report('github', health('paused', 'sourceOff'))
      failures = 0
    } catch (error) {
      failures += 1
      const failed = pollFailure(failures, config?.failuresBeforeError ?? 1, error)
      if (!signal.aborted && failed) ctx.report('github', failed)
    }
    await pause(Math.max(interval, ctx.config<GithubConfig>(SOURCE)?.pollSeconds ?? 60) * 1000, signal)
  }
}
```

要点：

- **前置门槛**：只有当 `setup` 解析出账号、来源处于启用状态时才真正轮询；否则上报 `setupPending` / `sourceOff` 并空转。
- **节奏**：每轮结束后 sleep。间隔取 **GitHub 要求的 `interval` 与配置 `pollSeconds` 的较大值**，下限 60 秒。
- **失败隔离**：单轮失败不会让循环崩溃，只累加 `failures`，连续失败到阈值才在健康面板报错。

## 一轮轮询（`pollRound`）做四类事

`pollRound` 是一次完整的取数，按顺序执行四步；**每一步、每个 Issue/PR/线程都独立容错**，一步失败不影响其它步：

```ts
export async function pollRound(ctx, signal): Promise<number> {
  const round = newRound(signal)
  const notified = await pollNotifications(ctx, round)   // ① 通知收件箱（核心）
  await pollRepository(ctx, round)                       // ② 仓库级事件流 + 默认分支 CI
  await pollTicketPulls(ctx, signal, round)              // ③ 已开工单的 PR 及其 CI
  await pollThreads(ctx, signal, round)                  // ④ 已订阅的线程
  ...
}
```

| 顺序 | 取什么 | 何时取 | 对应 GitHub API | 代码 |
| --- | --- | --- | --- | --- |
| ① | **通知**（@提及走这里） | 总是 | `GET /notifications` | `pollNotifications` |
| ② | **仓库级流**：新 Issue、新 PR、评论、代码评审评论；**默认分支 CI** | 仅当打开对应 `observe*` 开关 | `/issues`、`/issues/comments`、`/pulls/comments`、`/commits/<sha>/checks` | `pollRepository` |
| ③ | **已开工单的 PR** 与它的 CI 状态 | 有进行中的工单时 | 读 PR / checks | `pollTicketPulls` |
| ④ | **已订阅线程**（如某 PR 的后续讨论） | 有订阅时 | 读该线程 | `pollThreads` |

> `pollThreads` 等公共轮询逻辑在 `src/sources/poll.ts`：它只轮询「仍在 `POLL_HORIZON_MS`（14 天）内有活动或有活跃工单」的线程，失败会记在线程的 `pollError` 上，成功则清除。

## 核心：通知收件箱（`pollNotifications`）

这是「@一下就唤醒」的关键。GitHub 会把任何 `@agent` 的提及变成该账号收到的一条 **notification**，因此只要读通知就能拿到所有点名 agent 的事件。

```ts
const path = `notifications?all=true&per_page=50&since=${encodeURIComponent(since)}`
const raw = await commands.api(path, '--include')
```

流程：

1. **增量读取**：用 `since` 游标只取上次之后的新通知（`all=true` 连已读一起取）。
2. **分页**：响应头含 `rel="next"` 时继续翻页（`commands.pages`）。
3. **限定仓库**：`sameName(notification.repository.full_name, repository)`，只处理配置里那个仓库的通知。
4. **定位对象**：从通知的 `subject.url` 解析出它指向的是 Issue 还是 PR、编号是多少（`githubAddress`）。
5. **读正文**：再去读那条 Issue / 评论的完整内容，交给 `commentTrigger` / `issueTrigger` 规范化。
6. **失败重试表**：读取失败的条目记入 `retries`，下一轮**优先重试**，且游标与重试表**原子推进**，保证不漏。

GitHub 还会通过响应头 `x-poll-interval` 告知「请至少多久再拉一次」，代码据此尊重其节流要求：

```ts
const asked = Number(/x-poll-interval:\s*(\d+)/i.exec(headers)?.[1] ?? 60)
const interval = Math.max(60, Number.isFinite(asked) ? asked : 60)
```

## 从「一条评论」到「一个 Trigger」

拿到原始 GitHub 数据后，`src/sources/github-trigger.ts` 把它**规范化成领域 `Trigger`**（不可变事件）。以一条评论为例：

```ts
export function commentTrigger(comment, ref, kind, agent): Trigger {
  const at = comment.updated_at ?? comment.submitted_at ?? comment.created_at ?? new Date().toISOString()
  const version = comment.updated_at ?? comment.submitted_at ?? ''
  const id = key('github-comment', comment.html_url ?? `github:${ref.number}`, String(comment.id), version)
  return {
    id,                                        // 确定性 id：同样输入 → 同样 id（幂等基础）
    source: SOURCE,
    threads: [{ source: SOURCE, id: String(ref.number) }],   // 属于哪个线程（Issue/PR 编号）
    ...author(comment.user, agent),            // actor + human（是否真人）
    at,
    text: commentText(comment, kind),
    ...(mentions(comment.body ?? '', agent) ? { addressed: 'mention' as const } : {}),  // 是否点名了 agent
    channel: githubChannel(ref.number, comment.html_url ?? undefined),
    meta: { number: ref.number, isPr: ref.isPr, commentId: comment.id, kind, ... },
  }
}
```

几个关键字段的含义：

- **`id`**：由 `key()` 从「类型 + 线程/URL + 评论 id + 版本」确定性生成。**相同输入永远得到相同 id**，这是后续「接受两次只产生一个工单」的基石。
- **`threads`**：这条事件属于哪个 `Thread`（GitHub 用 Issue/PR 编号，如 `github:12`）。同一 Issue 的所有事件都归到同一个线程。
- **`human`**：由 `isHumanLogin` 判断——登录名以 `[bot]` 结尾或 `type` 不是 `User` 的不算真人（CI、机器人由此被识别）。
- **`addressed: 'mention'`**：当正文里 `@agent` 了才设置。判断用的是 `mentions()` 的正则，要求 `@` 前不是字母/数字/下划线/`@`，后面也不是——避免把 `@agent-other` 误认成 `@agent`。
- **`meta`**：GitHub 原生细节（编号、是否 PR、文件/行号、`in_reply_to_id` 等），**领域核心不解释它**，只由来源适配器使用。

新 Issue / 新 PR 则走 `issueTrigger` / `pullTrigger`，把「标题 + 正文」作为 `text`，同样在正文 @了 agent 时标记 `addressed`。

## 过滤与范围判定（`scope`）

事件不会无条件进入领域。`githubKind.scope` 决定「这条事件是否被监听」与「是否被观察」：

```ts
scope: ({ meta }, config) => {
  const row = scopeOf(meta)                       // issues | pulls | comments | ci
  return {
    listened: true,
    observed: row !== undefined && config[OBSERVED[row]],   // 对应 observe* 开关
    ...row ? { row } : {},
  }
},
```

- **两种触发模式**：
  - **默认「仅 @」**：只靠通知收件箱，只有 `@agent` 的事件才会叫醒它——**噪音最低**，适合起步。
  - **`observe*` 全量**：打开 `observeIssues` / `observePulls` / `observeComments` / `observeCi` 后，仓库里**没 @ 它**的新 Issue、新 PR、评论、默认分支 CI 也会被接收，再交给 **Policy 的「准入表」**判断要不要成为候选。
- **丢掉自己的声音**：`sameName(item.user.login, login)` 会跳过 agent 自己发的评论/Issue，避免自我触发。
- **过滤原因留痕**：被来源过滤掉的事件，会以来源自己的原因码记录到 `disposition`（`filtered`），可在面板查看。

## 幂等、游标与「不丢不重」

这是整套监听最工程化的部分，保证**重启不重放、网络抖动不漏、重复不产生重复工单**：

| 机制 | 做法 | 代码 |
| --- | --- | --- |
| **确定性 id** | `Trigger.id` 由内容确定性生成；重复收到同一事件是 no-op | `key()` in `src/core/ids.ts` |
| **增量游标** | `since` 游标记入 KV，每轮只读新增 | `KV.github.since` |
| **游标回退** | 游标向前回退 **5 秒**，避免漏掉「迟到但属于上一轮」的事件 | `cursorAt` in `src/sources/poll.ts` |
| **原子推进** | 游标与失败重试表一起提交，读失败的条目下轮优先重试 | `pollNotifications` |
| **ETag 省额度** | 游标没动时带 `If-None-Match`，列表没变返回 **304**，不耗速率 | `readStream` |
| **重启不重放** | 事件以「是否已存」判定，历史不重读 | 来源持有自己的读位置 |

> 「接受两次只产生一个工单」由领域层保证：`Engine.receive` 对同一 `Trigger` 返回 `duplicate`，绝不重复入账。

## 节流与速率

- **轮询间隔**：默认 60 秒，可在来源配置里调（`pollSeconds`，范围 60–3600 秒）；GitHub 若要求更长间隔，**以 GitHub 为准**。
- **条件请求**：游标未推进时用 ETag，未变更即 304，几乎零成本。
- **每页限量**：通知 `per_page=50`，仓库流 `per_page=100`，配合游标分批消化。

## 失败处理与健康上报

- **单点容错**：每类流、每个 Issue/PR/线程单独 `try`，一个失败不影响其它；一轮结束若有失败，抛出汇总错误。
- **失败阈值**：连续失败达 `failuresBeforeError`（默认 3）才上报 `error` 健康，单次网络抖动保持安静。
- **删除与放弃**：GitHub 返回 **410（Gone）** 的条目记为「已删除」；读取持续失败超过 24 小时的线程记为「放弃」，二者都不算轮询失败。
- **健康入口**：GitHub 来源在健康面板的条目名是 `github`，报告当前账号、`N notifications; next poll ≥ Xs` 等信息。

## 相关配置项一览

来源配置（`GithubConfig`，见 `src/sources/github.ts`）：

| 配置 | 默认 | 含义 |
| --- | --- | --- |
| `repository` | 必填 | 监听的仓库，`owner/name` |
| `cli` | 必填 | agent 的 GitHub CLI（`gh` wrapper，登录 agent 账号） |
| `pollSeconds` | 60 | 轮询间隔（秒），60–3600 |
| `failuresBeforeError` | 3 | 连续失败几次后报异常 |
| `observeIssues` | false | 接收仓库里的新 Issue（不 @ 也收） |
| `observePulls` | false | 接收仓库里的新 PR |
| `observeComments` | false | 接收仓库里的评论 / 代码评审评论 |
| `observeCi` | false | 接收默认分支的 CI 失败与恢复 |

> `observe*` 全部关闭时即「仅 @」模式；打开后由 Policy 的**准入表**（admission table）决定这些「没被点名」的事件要不要成为候选。

## 完整链路：从「@一下」到「一个工单」

以「在 Issue 里评论 `@agent 修个 bug`」为例：

1. **GitHub 侧**：你 @ 了 agent 账号 → GitHub 给该账号的通知收件箱塞一条 notification。
2. **下一轮轮询**（≤60 秒）`pollNotifications` 读到它 → 定位到是哪个 Issue、读出那条评论正文。
3. **规范化**：`commentTrigger` 生成 `Trigger`，标记 `addressed: 'mention'`、`human: true`、归属线程 `github:<issue 编号>`。
4. **过滤**：非本人消息、在监听范围内 → 通过 `ctx.receive(trigger)` 进入领域。
5. **入账与扇出**：`Engine.receive` 存 Trigger → 扇出到订阅该线程的工单 → 无工单则进 **Eval（评估）**。
6. **授权与建单**：Eval 依据 **Policy（准入 gate + 研判）** 判断；接受则创建 **Ticket（工单）+ Mandate（授权）**，派给执行器在独立 Session 与目录里干活。
7. **交付**：结果通过 **Channel（GitHub 评论 / PR）** 回到原对话。

## 已知行为与「坑」

- **编辑旧 Issue 正文加 @ 不会被收到**：通知只在**新发评论 / 新建 Issue** 时产生，编辑不触发通知。要 @ 就发**新评论**。
- **必须是用户账号**：GitHub App 没有可读的「通知收件箱」，不能作为 agent。
- **有延迟**：反应速度取决于轮询间隔（默认约 1 分钟），不是实时。
- **只支持 github.com**：不支持 GitHub Enterprise Server。
- **仅 @ 模式下，没 @ 它的事件一律不收**：想让它「主动发现」问题，需打开 `observe*` 并配置准入表。

## 源码索引

| 文件 | 职责 |
| --- | --- |
| `src/sources/github.ts` | GitHub 来源类型定义、`watch` 轮询循环、`pollRound` 及四类取数、仓库流与默认分支 CI |
| `src/sources/github-trigger.ts` | 把 Issue / PR / 评论 / 评审 / CI 规范化为 `Trigger`；`mentions`、`githubAddress`、`isHumanLogin` |
| `src/sources/poll.ts` | 轮询公共件：`POLL_HORIZON_MS`、`cursorAt`（游标回退）、`pollThreads`（订阅线程轮询）、`pause` |
| `src/sources/github-pull.ts` | PR 规则：哪些工单跟随哪个 PR、其关闭/合并如何上报 |
| `src/sources/github-read.ts` | CI 快照、仓库读取、`githubThreadContext`（Issue/PR 会话上下文） |
| `src/sources/github-channel.ts` | GitHub 写渠道：评论、评审线程回复 |
| `src/core/ids.ts` | `key()`：确定性 id，幂等的基础 |
| `src/sources/kind.ts` | `SourceKind` 协议：`scope`、`channel`、`threadContext` 等扩展点 |
