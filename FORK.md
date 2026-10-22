# Fork 说明（dsh-rules）

本目录是 [`rj-jiangyichen/dsh-rules`](https://github.com/rj-jiangyichen/dsh-rules) 的本地 fork，
用于适配 **Session format V4** 的消息来源（source kind）要求。

- upstream：`origin` 上游仓库，基线提交 `c509688`（release 0.1.6）
- 版本号：`0.1.7`（只改本 fork，不发布）
- 安装方式：以 `link:` 依赖装进 desktop profile

## 为什么要 fork

上游 0.1.6 注入规则快照时用的是 **V3 时代的 source 包装**：

```js
source: { kind: "plugin", plugin: "dsh-rules", form: "notice", summary }
```

而 DSH 0.2.1-rc.2 持久化的是 **Session format V4**，其消息准入
（`@deepseek-ai/dsh-session-format-v3-to-v4` 的 `source()`）明确拒绝 `kind === "plugin"`：

```
format v4 message requires a producer-owned source kind
```

后果不是「规则不生效」而是**整轮请求失败**：消息由 agent loop 在插件 try/catch 之外追加。

V4 的等价形态是 producer 自有 kind，第三方插件按迁移规则为 `plugin:<包名>`。

## 改动清单

| 文件 | 改动 |
|---|---|
| `lib/index.js` | 新增 `SOURCE_KIND = \`plugin:${name}\``；`rulesMessage()` 的 source 改为 `{ kind: SOURCE_KIND, form, summary }`（删除 V3 的 `plugin` 字段）；`seedFromSession()` 与 `sameRulesMessage()` 的判据同步改为 `SOURCE_KIND` |
| `test/apply.test.mjs` | 断言与恢复会话夹具跟进到 `plugin:dsh-rules` |
| `package.json` | `0.1.6` → `0.1.7` |
| `CHANGELOG.md` | 新增 0.1.7 条目 |

## 安装 / 回滚

```powershell
$dsh = 'D:\DSH\resources\runtime\cli\bin\dsh.cmd'

# 安装（已在 desktop profile 生效，profile/package.json 记作 link:）
& $dsh plugin --profile desktop add 'D:\project\dsh-plugins\dsh-rules'

# 回滚到 npm 版
& $dsh plugin --profile desktop remove dsh-rules
& $dsh plugin --profile desktop add 'dsh-rules@0.1.6'
```

安装后必须重启 DSH；`dsh-rules` 已在 `dsh.profile.bundles` 列表中，无需再改。

## 修改后如何自测

仓库测试依赖 devDependencies（`@deepseek-ai/dsh-llm` 等），装完测完即可删，
**不要长期保留 `node_modules`**：以 `link:` 方式安装时，本地副本会遮蔽宿主提供的
`@deepseek-ai/*` 模块，可能造成双实例。

```powershell
cd D:\project\dsh-plugins\dsh-rules
npm install --ignore-scripts --no-audit --no-fund
node --test          # 49 项（含 apply.test.mjs）
Remove-Item node_modules -Recurse -Force
```

## 跟进上游

```powershell
cd D:\project\dsh-plugins\dsh-rules
git fetch origin
git log --oneline origin/main -5
git rebase origin/main      # 冲突点预期在 source kind 三处与测试断言
node --test                 # 需要先临时装依赖
```

上游若已自行适配 V4，本 fork 的 `SOURCE_KIND` 改动应能在 rebase 时丢弃，
届时直接回到 npm 版本即可。
