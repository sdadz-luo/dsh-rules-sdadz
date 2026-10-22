# Changelog

All notable changes to this project are documented in this file.

## [0.1.10] — 2026-10-22

### Fixed

- Resolve the session directory safely: the plugin now reads `session.header.cwd` first and the `workingDirectory` service through `ctx.get()` (the direct `ctx.workingDirectory` property throws while the service is absent, which broke both `touch()` and `compose()`).
- `touch()` falls back to the touched file's own project root when the session root is unknown or does not contain the file, so glob activation no longer depends on the session cwd being committed.
- `compose()` prefers the project root recorded by the most recent touched file, so project-level rules load even when the session header carries no cwd.
- Failures now leave a diagnostic line (`touch-root-failed`, `pre-step-error`) instead of only an invisible logger warning.
## [0.1.7] — 2026-10-21

### Fixed

- Rules snapshots now carry the Session format V4 producer-owned source kind (`plugin:dsh-rules`) instead of the retired V3 wrapper (`kind: "plugin"` plus a `plugin` field). DSH 0.2.1-rc.2 persists format V4, whose message admission rejects `kind: "plugin"` with `format v4 message requires a producer-owned source kind`, so every injection failed the enclosing turn. The seed-from-log matcher and the duplicate check now key on the same kind.

## [0.1.6] — 2026-10-19

### Fixed

- The declared DSH range no longer rejects DSH 0.2.x. Since 0.1.4 the three host packages were declared as `>=0.1.5-rc.2 <0.2.0`, and DSH's startup compatibility check compares the running version — not each package's own version — against every `@deepseek-ai/dsh-*` peer. On DSH 0.2.1-rc.2 all three requirements were unmet, so the profile loader put the bundle in `skippedBundles`: the plugin stayed installed, never mounted, and injected no rule. The range is now `>=0.1.5-rc.2 <0.3.0-0`; the `-0` ceiling admits 0.2.x prereleases (the check runs semver with `includePrerelease`) while still rejecting 0.3.0 and its prereleases, which remain untested.

### Changed

- Development pins track the harness this fix was verified against: `@deepseek-ai/dsh-llm`, `dsh-home-paths`, and `dsh-session` move to `0.2.1-rc.2`, with `@deepseek-ai/cordis` `^4.0.5` and `@deepseek-ai/schemastery` `^3.18.5`. The suite passes unchanged (49 tests): `boundContextSummary` / `createUserMessage` from `dsh-llm`, `dshHomeDisplay` / `resolveDshHome` from `dsh-home-paths`, and `Session.id` / `header` / `snapshotEvents()` are all still exported in 0.2.1-rc.2, so no runtime code needs to change.

## [0.1.5] — 2026-09-22

### Fixed

- `exports` no longer advertises declaration files that do not exist. Both subpaths resolved `types` to `./lib/types/*.d.ts`, a directory that never existed in the repository or in the published tarball — so the conditions now read as plain module targets. The package ships JavaScript only; real declarations would need `tsc --emitDeclarationOnly` over the JSDoc plus a build step, which this package deliberately has none of.

### Changed

- Re-injection is now keyed on the active rule set and rule content, not on the rendered snapshot text. A snapshot's `(matched files: …)` intro records which files had matched when it was rendered; because that list was part of the compared text, every newly matched file re-sent the full body of every active rule. In a real workspace (four rules, 6.5–7.4 KB each) 13 of 15 observed injections were caused by nothing but that list growing, carrying 71.2 KB of 113.1 KB injected bytes. `snapshotKey` in `lib/rules.js` reduces a snapshot to its list-free form for the comparison, so the injected text (and therefore resume restoration from the log) is unchanged while the redundant snapshots disappear. Adds `snapshotKey` unit tests plus an `apply` test that a second matching file injects nothing and a second rule still does.

## [0.1.4] — 2026-09-21

### Fixed

- Rule injection works again on the current harness. `@deepseek-ai/dsh-session` no longer exposes the `session.events` array (the log is read through `eventAt()` / `snapshotEvents()` / `ownEvents()`), so seeding a resumed session threw `TypeError: Cannot read properties of undefined (reading 'length')` on every `agent/pre-step`. The throw happened before the session was marked as seeded, so it recurred on every step, the handler swallowed it as a warning, and no rule was ever injected — the plugin looked loaded but did nothing. The seed now reads `session.snapshotEvents()`, and a regression test covers a resumed session whose log carries an older snapshot.
- Dependency declarations now name the harness this plugin is built against: `@deepseek-ai/dsh-llm`, `@deepseek-ai/dsh-home-paths`, and `@deepseek-ai/dsh-session` are declared as `>=0.1.5-rc.2 <0.2.0` (development pins `0.1.5-rc.2`). The previous `^0.1.0-rc.6` ranges excluded every later prerelease, so a break like this one installed with no warning.
- `scripts/install-desktop.mjs` finds the packaged CLI again: it hardcoded `resources/app.asar.unpacked`, which the current Desktop build no longer ships (the app payload lives under `resources/app`). Both layouts are now probed.

## [0.1.3] — 2026-09-13

### Added

- UI-visible rule activation notice: each injected `<rules>` snapshot now uses the host's `notice` context form, so the DSH web UI shows a collapsed one-line row (same chrome as tool-call rows) — e.g. `dsh-rules · Active rules: typescript, docs` — with the full snapshot text behind the disclosure. Clearing the active set shows `No active rules`. No frontend changes required; the resume path (`seedFromSession`) accepts both the new `notice` form and the legacy `rules` form.

## [0.1.1] — 2026-08-29

### Fixed

- Project rules load again: the path sanitization that normalizes `ruleDirNames` (and instruction-file candidate lists) rejected every entry containing a path separator, so the built-in default `.dsh/rules` was silently filtered out and no project rule directories were ever scanned. Entries are now validated per segment instead — non-empty, without `.` / `..`, and not absolute — so multi-segment project-relative directories such as `.dsh/rules` work as documented. Dropped unsafe entries are reported through a warning rather than ignored silently.

## [0.1.0] — 2026-08-16

Initial release.

### Added

- Glob-activated rule prompts for DeepSeek Harness (DSH): rules declare glob patterns in frontmatter; when the agent reads or edits a matching file, the rule activates and its content is injected into the conversation as a superseding `<rules>` snapshot.
- Claude Code compatibility: plain rule files (`.dsh/rules/*.md`, `~/.dsh/rules/*.md`) and `# Path:` sections inside `AGENTS.md` / `CLAUDE.md` (including `.local.md` variants).
- Byte-budget rendering (32 KB default) with priority-based dropping and truncation; content escaping prevents breaking out of the framing tags.
- Per-session touched-path tracking (subagents included), resume-friendly snapshot restoration from the session log.
- Versioned rule discovery and caching: edits to rule files take effect on the next agent step.
- Standard DSH plugin bundle (`dsh.bundle` manifest with `cordis.patch.yml`) — installable via `dsh plugin --profile <name> add dsh-rules`.

[0.1.6]: https://github.com/rj-jiangyichen/dsh-rules/releases/tag/v0.1.6

[0.1.5]: https://github.com/rj-jiangyichen/dsh-rules/releases/tag/v0.1.5

[0.1.4]: https://github.com/rj-jiangyichen/dsh-rules/releases/tag/v0.1.4

[0.1.3]: https://github.com/rj-jiangyichen/dsh-rules/releases/tag/v0.1.3

[0.1.1]: https://github.com/rj-jiangyichen/dsh-rules/releases/tag/v0.1.1

[0.1.0]: https://github.com/rj-jiangyichen/dsh-rules/releases/tag/v0.1.0
