# AGENTS.md — 本仓库的 AI 协作说明

本仓库是 `dsh-rules` 的 fork，用于适配 **DSH Session format V4** 并修正会话目录解析。
任何 AI 代理在此仓库工作时先读本文件；行为准则与用户偏好以用户级 `~/.dsh/AGENTS.md` 为准。

## 不可破坏的不变量

1. **注入消息的 source kind 必须是 `plugin:dsh-rules`**（由 `SOURCE_KIND` 构造）。
   不要改回 `kind: "plugin"`，也不要恢复 `plugin` 字段——DSH 的 V4 准入会直接拒绝，
   而且失败发生在 agent loop 里，会让整轮请求报错而不是静默降级。
2. **不要给 `dependencies` 增加任何 `@deepseek-ai/*` 包**。宿主通过 loader 提供这些模块；
   如果本地 `node_modules` 里存在同名副本，会遮蔽宿主模块并可能产生双实例。
   它们只能出现在 `peerDependencies`（保持 `>=0.1.5-rc.2 <0.3.0-0`）与 `devDependencies`（测试用）。
3. **`lib/rules.js`、`lib/fs.js` 是纯逻辑模块**，不得引入 Cordis 上下文或副作用；
   所有 harness 交互留在 `lib/index.js`。
4. **`ctx` 上的可选服务一律用 `ctx.get("name")` 取**，不要直接 `ctx.name`——
   服务未挂载时属性访问会抛异常，会把 `touch()` / `compose()` 一起打断（0.1.8 的教训）。
5. 规则正文是**用户内容**，渲染必须保持 `&`、`<` 转义，不能出现能闭合 `<rules>` 框架的路径。

## 结构

| 路径 | 作用 |
|---|---|
| `lib/index.js` | Cordis 插件入口：配置 schema、`fs/observed` 跟踪、`agent/pre-step` 注入、会话状态 |
| `lib/rules.js` | 纯逻辑：frontmatter / `# Path:` 解析、picomatch 编译、优先级合并、预算渲染 |
| `lib/fs.js` | 规则源发现与读取：优先 harness `fs` 服务，缺失时回退 Node fs |
| `test/apply.test.mjs` | 插件行为测试：touch → compose → 注入消息，含 source 形状与恢复会话 |
| `test/rules.test.mjs` | 纯逻辑测试：解析、glob、优先级、预算、截断、转义 |
| `cordis.patch.yml` | bundle 层：向 profile 插入 `dsh-rules` 行 |

## 测试

```powershell
npm install --ignore-scripts --no-audit --no-fund
node --test
Remove-Item node_modules -Recurse -Force
```

- 期望 **49 项全过**；改动了 source kind、预算或匹配逻辑时必须同步更新 `test/apply.test.mjs`。
- **测完必须删 `node_modules`**：本插件会被 `file:` 安装进 profile，
  若源码目录保留 `node_modules`，其中的 `@deepseek-ai/*` 副本可能遮蔽宿主模块。

## 发布 / 安装流程（桌面版）

```powershell
# 1) 版本号 + CHANGELOG
# 2) 打包
npm pack                      # 产出 dsh-rules-<version>.tgz

# 3) 装进 profile（必须用 tgz，不能用目录 / link）
& 'D:\DSH\resources\runtime\cli\bin\dsh.cmd' plugin --profile desktop add `
  "file:D:/project/dsh-plugins/dsh-rules/dsh-rules-<version>.tgz"

# 4) 重启 DSH 后验证
```

验证方式：在某个工程会话里读一个匹配规则 glob 的文件（例如 `.c`），确认
① 会话里出现可折叠的 `<rules>` 行；② 没有 `format v4 message requires a producer-owned source kind` 报错。

## 已知坑（每条都真实踩过）

1. **`link:` / 指向源码目录的 `file:` 安装不会被加载**：loader 只认 profile `node_modules`
   里的真实目录，junction 形式的包会被静默跳过（无日志）。必须 `npm pack` 后装 tgz。
2. **`dsh plugin remove dsh-rules` 会把它从 `dsh.profile.bundles` 里删掉**；如果随后重装失败，
   bundles 列表就少了这一项，插件不会加载。重装后必须确认列表里仍有 `dsh-rules`。
3. 替换正在被加载的包时 pnpm 可能报 `EPERM ... rename`（旧 junction 无法原子替换）。
   先 `cmd /c rmdir <profile>\node_modules\dsh-rules` 删链接，再重新安装。
4. **规则文件本身不需要重启**：插件每步按 `mtime` 版本缓存重新探测，改完下一步生效；
   只有插件 JS 改动才需要重启 DSH。
5. `dsh --profile desktop --dump-config` 会被拒绝（桌面 profile 由 Electron 独占），
   验证走"重启 + 读匹配文件"，不要依赖 CLI 转储。

## 提交约定

- 提交信息用中文，格式 `<类型>: <描述>`（`fix` / `chore` / `docs` / `test`）；
- 不写任何 AI 助手署名；
- 修改 `lib/` 后必须提交并重新 `npm pack` + 安装，否则 profile 里跑的还是旧版本。
