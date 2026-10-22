# dsh-rules（Session format V4 兼容 fork）

基于 [rj-jiangyichen/dsh-rules](https://github.com/rj-jiangyichen/dsh-rules) 的个人 fork，
修复 **DSH 0.2.1（Session format V4）下规则注入失败**的问题，并修正会话工作目录解析。

> Glob-activated rule prompts for [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) —
> Claude Code `rules.md` / `# Path:` 风格：规则声明 glob，模型**读或写**匹配文件时，规则正文以
> `<rules>` 快照注入对话（快照取代旧快照）。

当前版本：**0.1.11**　基线：上游 `c509688`（0.1.6）　依赖：DSH `0.2.1-rc.2` 实测通过

## 为什么需要这个 fork

上游 0.1.6 注入消息时用的是 **V3 时代的 source 包装**：

```js
source: { kind: "plugin", plugin: "dsh-rules", form: "notice", summary }
```

而 DSH 0.2.1 持久化的是 **Session format V4**，其消息准入门禁
（`@deepseek-ai/dsh-session-format-v3-to-v4` 的 `source()`）明确拒绝 `kind === "plugin"`：

```
format v4 message requires a producer-owned source kind
```

后果不是"规则不生效"，而是**整轮请求失败**——消息由 agent loop 在插件 try/catch 之外追加。
V4 的规范形态是 producer 自有 kind，第三方插件按迁移逻辑为 `plugin:<包名>`。

## 与上游的差异

| 方面 | 上游 0.1.6 | 本 fork 0.1.11 |
|---|---|---|
| 注入消息 source | `kind: "plugin"` + `plugin` 字段（V3，V4 拒绝） | `kind: "plugin:dsh-rules"`（V4 producer 自有 kind） |
| 恢复会话时的判据 | 按 `kind === "plugin"` 匹配 | 按 `SOURCE_KIND` 匹配，兼容迁移后的日志 |
| 会话工作目录 | `session.header.cwd ?? process.cwd()`，header 缺 cwd 时退化到宿主目录 | `header.cwd` → `ctx.get("workingDirectory")`（安全取值，不抛错）→ `process.cwd()` |
| 项目根解析 | 只按会话 cwd 向上找标记 | 会话根不含该文件时，**从被读文件目录向上找项目根**兜底 |
| 项目级规则加载 | 依赖会话 cwd | 优先使用"最近一次成功 touch 的文件所在根" |
| 测试 | 49 项 | 49 项，夹具与断言跟进 V4 |

## 安装（桌面版）

桌面版 profile 用 `link:` / `file:` 指向源码目录时，**loader 不会加载该 bundle**（只认
`node_modules` 里的真实目录）。因此必须打包后安装：

```powershell
cd D:\project\dsh-plugins\dsh-rules
npm pack

$dsh = 'D:\DSH\resources\runtime\cli\bin\dsh.cmd'
& $dsh plugin --profile desktop add "file:D:/project/dsh-plugins/dsh-rules/dsh-rules-0.1.11.tgz"
```

然后**重启 DSH**。`dsh plugin add` 会把 `dsh-rules` 写进 `dsh.profile.bundles`；
若某次安装失败，请检查该列表是否仍含 `dsh-rules`（`plugin remove` 会删掉它）。

回滚到上游版：

```powershell
& $dsh plugin --profile desktop remove dsh-rules
& $dsh plugin --profile desktop add 'dsh-rules@0.1.6'
```

## 规则格式

规则文件放两处：

- 用户级：`~/.dsh/rules/*.md`
- 项目级：`<项目根>/.dsh/rules/*.md`

```markdown
---
path:
  - "**/*.c"
  - "!**/third_party/**"
---

# 规则正文（任意 markdown / 提示词）
```

| 字段 | 说明 |
|---|---|
| `path`（**单数**） | 字符串或字符串数组，相对项目根的 POSIX glob；`!` 前缀为排除；**缺省或为空 = 常驻规则** |
| `name` | 可选，规则标识；默认取文件名（去 `.md`） |

- 优先级：项目级 `100` > 用户级 `200` > `AGENTS.md`/`CLAUDE.md` 的 `# Path:` 段落 `300`；同名取高优先级
- 注入形态：`<rules>` 快照作为 user 消息进入会话，UI 中是可折叠行；每条快照取代旧快照
- 只有项目根内的文件能触发；读、写、编辑都会记录

## 配置

在目标 profile 的 `cordis.patch.yml` 里按 id 覆盖：

```yaml
- id: dsh-rules
  config:
    maxBytes: 12288              # 单次注入字节上限（默认 32768）
    projectRootMarkers: [".git"]  # 项目根标记，可加 ".dsh"
    includeUserRules: true        # 读取 ~/.dsh/rules
    includeClaudeSections: false  # 解析 AGENTS.md 的 # Path: 段落
```

## 已知限制

- **只有项目根内的文件能触发**；跨盘读取时依赖"按文件位置找根"的兜底
- `touched` 集合按会话保留 512 条（FIFO），超长会话可能淘汰较早的匹配
- 注入是 user 角色消息：**克隆的第三方仓库若带 `.dsh/rules/*.md`，会被执行**——陌生仓库先看一眼
- 上游使用的 `session.snapshotEvents()` 在 DSH 中已标记弃用，0.3 升级时需跟进
- 同一会话混读多个工程时，项目级规则按"最近一次 touch 的根"加载

## 开发与测试

```powershell
cd D:\project\dsh-plugins\dsh-rules
npm install --ignore-scripts --no-audit --no-fund   # 仅测试需要
node --test                                          # 49 项
Remove-Item node_modules -Recurse -Force             # 测完必须删，见 AGENTS.md
```

## 上游同步

```powershell
git fetch upstream                     # upstream = 上游仓库
git log --oneline upstream/main -5
git rebase upstream/main                # 冲突点预期在 source kind 与工作目录解析
```

上游若自行适配 V4，本 fork 的 `SOURCE_KIND` 改动可直接丢弃，回到 npm 版本。

## License

MIT，上游版权归 [rj-jiangyichen](https://github.com/rj-jiangyichen/dsh-rules)；
本 fork 的修改同样以 MIT 发布。
> 说明：本 fork 移除了上游的 .github/workflows/ci.yml（推送用的 PAT 无 workflow 权限）。
ode --test 可本地跑全部 49 项。
