# Harness 版本兼容

> [English](harness-compatibility.md) · **中文** · [文档索引](README.zh.md)

## 本插件支持到哪

| Harness 序列 | 已验证 | 证据 |
| --- | --- | --- |
| `0.1.1-rc.2` | 是 | 就是 pin 在 `devDependencies` 的版本；`npm run typecheck`、`npm test` |
| `0.1.2-rc.1` | 是 | 对该序列实际发布的包跑 typecheck + 测试 |
| `0.1.3-alpha.2` | 是 | 同上 |
| `0.1.5-rc.2` | 是 | 同上，即 `next` 当前解析到的版本 |
| `0.1.7-rc.1` | 是 | typecheck + 71 个测试；**本 fork 的适配基线** |

每条序列的验证状态与依据记在 [acceptance.zh.md](acceptance.zh.md)。

peer 范围是 `>=0.1.7-rc.1 <0.2.0-0`。**本 fork 重新基线到 0.1.7**：0.1.7 删掉了消息来源里通用的 `plugin` 种类、并把 `agent/created` 改成 serial，本插件因此只声明 `0.1.7-rc.1` 起的区间（早期序列请用上游 `Crosery/dsh-viewer@0.1.1`）。**按证据放宽，不靠乐观**：`0.1.4` 元组不在里面，是因为上游根本没在它上面发过东西。

一条序列只有在本插件需要的每个包都发布到它上面时才装得上，而有好几条不是。每次都是同一个形状：某个 tag 发布的 `@deepseek-ai/dsh-tools` 要求该 tag 从未发布过的 `@deepseek-ai/dsh-user-approval`，而确实存在的最近那个版本又要求第三个该 tag 同样没发的包。`0.1.2-alpha.5`、`0.1.5-alpha.1`、`0.1.5-alpha.2`、`0.1.5-rc.1` 都是这样，而且每一条都是在依赖图里完全没有本插件的情况下确认的。这是上游的状态，不是本插件的声明；定时 job 会继续如实报告。

## 0.1.2 的 API 改名

0.1.2 把 settings 挂载从包导出搬成了服务方法：

```ts
// 0.1.1 及更早
import { installSettingsSection, settingsNamespace } from '@deepseek-ai/dsh-settings'
installSettingsSection(ctx, settingsNamespace('crosery-viewer'), schema, entry, hooks)

// 0.1.2 起
ctx.settings.installSection(ctx, 'crosery-viewer', schema, entry, hooks)
```

hooks 与它们接线的注册完全一致，搬走的只是写法。`settingsNamespace()` 没有替代品，因为校验改由服务自己做。

真正把用户打崩的是**静态导入**：ESM 在执行任何代码前先解析具名导出，所以 0.1.2 及以后整个 Host 入口加载失败——`does not provide an export named 'installSettingsSection'`，插件不是降级到组合配置，而是根本没起来。`src/index.ts` 里的 `mountSettingsSection` 按运行时实际提供的接口驱动，`tests/settings-mount.test.ts` 把两种实现加上「两套都没有」的回退都钉住了。

同一条序列里还搬了两样东西，本插件刻意都不绑定：

- 客户端 `sessions` 服务：0.1.1 及以前在 `@deepseek-ai/dsh-client-runtime`，0.1.2 起在 `@deepseek-ai/dsh-api-session-controller`。插件以结构化类型声明自己读的那一小片（`ViewerSessionFace`、`ViewerSessions`、`ToolCallBlockLike`），不导入任何一个包的类型，因此一份构建在两边的 typecheck 都过。
- `dsh-client-runtime` 在 `0.1.1-rc.2` 之后停止发布，这也是它不再是 devDependency 的原因。

## 0.1.7 的两处 API 变化

**消息来源不再有通用的 `plugin` 种类。** `MessageSourceMap` 的 `plugin: { kind: 'plugin', plugin: string }` 成员被删除，改为「每个生产者在自己的模块里声明自己的 kind，消费者对不认识的 kind 直接穿透」——上游明确写了「there is no shared catch-all `plugin` kind」。本插件改为声明并发送自己的 `dsh-viewer` kind（`src/display-file.ts` 顶部的 `declare module '@deepseek-ai/dsh-llm'`），这是合并扩展的求和类型，所以不需要上游再提供 catch-all。

**`agent/created` 从 emit 变成 serial。** 0.1.7 起该边按注册顺序 `await` 每个监听器，回调签名要求返回 `Promise<undefined> | undefined`，返回 `void` 的函数过不了类型检查。`src/supersede-read-image.ts` 里显式 `return undefined`；工作本身仍然同步、不会 reject（监听器抛错会否决整个 agent 的发布，这一点 0.1.5 起就是如此）。

**settings 侧无需改动。** `ctx.settings.installSection` 与 `settings.register` 在 0.1.7 双双退役，`mountSettingsSection` 的两条探测分支都不命中，于是 profile 条目成为配置的唯一来源；而 0.1.7 的设置界面本来就是从每个条目的 Config schema 自动生成的（本插件 `Config = ViewerSettingsSchema`），四个开关照样可编辑，改动由 loader 重新 apply。

## 预发布陷阱

node-semver 只有在范围里**某个比较符与该版本的 `major.minor.patch` 元组完全一致、且自身也带预发布标签**时，才放行预发布版本。看着很宽的范围没有用：

```jsonc
// 看起来很宽，实际一个 0.1.x 预发布都匹配不到
">=0.0.1-rc.1 <0.2.0"

// 每个元组一条显式预发布分支 —— 我们用的是这个
">=0.1.1-rc.0 <0.1.2-0 || >=0.1.2-rc.0 <0.1.3-0"
```

harness 正处在预发布序列上，这里写错就意味着每个用户都撞 `ERESOLVE` 然后自己手工绕。`npm run check` 会断言 peer 范围能接纳 `devDependencies` 里 pin 的那个版本，这是让两者不脱节的最省事办法。

## 漂移 job 开了 issue 之后

`.github/workflows/harness-compat.yml` 每周对 `next` 和 `alpha` 两条 tag 跑一遍：把所有 harness devDependency 重新指向该 tag 当前解析到的版本，安装、typecheck、测试。**失败就是信号本身**，不是意外，所以它会开一个带 `upstream-drift` 标签的 issue。

job 用两次安装回答两个问题，并且分开报告：

- **插件还能不能对着该序列发布的类型编译？** 每次都问。这里失败才是漂移信号。
- **它自己的测试过不过？** 这些测试在运行时 import harness 包，需要一套真正解析成功的 peer 图。好几个 `alpha` tag 发布的 `dsh-tools` 要求该 tag 从未发过的包，npm 自己的解析器直接拒绝；job 把这种情况认作**序列不完整**，跳过运行时测试并在 run summary 里写明，而不是开漂移 issue。只有 `ERESOLVE` 算这一类，其它安装失败仍然如实报告。

按这个顺序处理：

1. **先读 typecheck 输出。** 被改名或删除的导出会自己报出来。
2. **判断那条序列是否完整。** 确认本插件依赖的每个 harness 包都真的发了那个版本。不完整的序列还不到适配的时候。
3. **先适配，再放宽。** 先改代码、对着那个 tag 验过，**然后**才扩 peer 范围——并且要在新元组上带预发布标签的比较符。
4. **以 patch 版本发出去。** peer 范围是包契约的一部分，改它需要一个版本号。

**不要为了让 job 变绿而放宽范围。** 范围是一句「什么能跑」的承诺，而这个 job 存在的意义就是让这句承诺保持属实。
