# claude-plugins

English | [한국어](README.ko.md)

A Claude Code plugin marketplace. Each plugin lives in its own directory under `plugins/`; install only the ones you use.

| Plugin | What it does |
|---|---|
| [`workdoc`](plugins/workdoc) | Work-order flow: write a work order → review it → implement it → close it |
| [`session`](plugins/session) | Session wrap-up and handoff: check for leftovers → summarize → resume in the next session |

> **Language note.** The skills are written in Korean and produce Korean output. The documents they write use Korean field names and status values (for example `구현 모델 권장`, `우선순위`, `완료`). These appear below as they are written to disk, with an English gloss.

## Install

Use either method. The result is the same.

### From the shell

Register the marketplace (once per machine).

```bash
claude plugin marketplace add idas4you/claude-plugins
```

Install a plugin.

```bash
claude plugin install workdoc@idas4you --scope user
```

### Inside a Claude Code session

Start `claude`, then type these in the prompt, one after the other.

```
/plugin marketplace add idas4you/claude-plugins
```

```
/plugin install workdoc@idas4you
```

Typing `/plugin` alone opens a list where you can pick a marketplace and a plugin to install. Installed skills show up from the next session (in a running session, use `/reload-plugins`).

## Update

```bash
claude plugin marketplace update idas4you
```

```bash
claude plugin update workdoc@idas4you --scope user
```

Updates apply from the next session (in a running session, use `/reload-plugins`).

## workdoc

Splits work into four steps (write the work order → review it → implement → close) and runs each step on the model that suits it.

| Skill | Model (`model:`) | Effort (`effort:`) | What it does |
|---|---|---|---|
| `/idas4you-doc <topic>` | `fable` | `medium` | Writes or edits a work order (design.md, plan.md, tasks.md, and an ADR when needed). Reads code but does not change it. When done, asks whether to review it right away with `/idas4you-doc-check` |
| `/idas4you-doc-check <work-order dir>` | `fable` | `high` | Reviews the work order before implementation (fact check against the code, whether the design works, whether it can be executed as written). Fixes factual errors directly and only reports design decisions. Writes the model to implement with into the `구현 모델 권장` (recommended implementation model) row at the top of tasks.md. Appends one row with the time and verdict to the `검수 이력` (review history) table in tasks.md on every run, so two reviews leave two rows. Ends with a recommended next step (such as `/idas4you-doc-impl`) that matches the verdict |
| `/idas4you-doc-check-all [filter]` | `fable` | `high` | Finds every work order that has not been reviewed yet (no row in `검수 이력`) and reviews each by the same criteria as `/idas4you-doc-check`. Hands each work order to its own subagent (up to three at a time), then updates the index and commits once at the end. **Asks nothing along the way**: anything that needs a decision is written into that work order's "미결 질문" (open questions) with a recommendation, and `/idas4you-doc-impl` asks about it before implementing. Work orders changed since their last review are only listed |
| `/idas4you-doc-impl <work-order dir>` | `opus` | session setting | Implements and tests the work order on a feature branch and records the results in the work order. If `구현 모델 권장` is `sonnet` or `fable`, hands the coding for that step to a subagent on that model. If a step still fails after two fix attempts, moves it up one model (`sonnet` → `opus` → `fable`). Asks before merging, even when tests pass |
| `/idas4you-doc-impl-list [filter]` | `opus` | session setting | Shows unfinished work orders in priority order as a table (a one-line description of what each one does, progress n/m, last review verdict, recommended implementation model, what blocks them), then lets you pick one from a selection prompt and hands it to `/idas4you-doc-impl` to start implementing. Work orders whose prerequisites are not done are left out of the choices. Picking `목록만 보기` (list only) starts nothing. Does not modify the index or the work orders |
| `/idas4you-doc-done <work-order dir>` | `sonnet` | session setting | Closes a finished work order. Checks the checklist, verification table, merge state, and open questions; if anything is missing, it reports instead of closing. Sets the status of the work order, the index, and the ADR to `완료` (done), unblocks work orders that were waiting on this one, and recommends the next one. Append `중단` (stopped) or `대체됨 <new work order>` (superseded) to close with that status. Does not merge or deploy |

- **Priority**: each work order gets `상`/`중`/`하` (high/medium/low) in the `우선순위` (priority) column of the index (`docs/work/README.md`). `/idas4you-doc` decides it after writing the work order (production incident, deadline, whether it blocks other work orders), and `/idas4you-doc-check` re-evaluates it during review. A value you change is marked `(사용자 지정)` (set by user) and is never overwritten automatically. Calling `/idas4you-doc-impl` without a path proposes the highest-priority work order whose prerequisites are done.
- **Models required**: the account needs access to Fable and Opus for this to run as intended (Sonnet is also used when implementation is delegated and when closing a work order). A skill's `model:` and `effort:` apply for the turn that invokes the skill, in place of the session setting. If your organization restricts available models, `model:` may not take effect.
- If another skill has the same name, call it with the plugin name, as in `/workdoc:idas4you-doc`.
- Repository-specific values (work-order location, ADR location, branch naming, commit author, test and deploy commands) are not hard-coded in the skills. They are read from the repository's `CLAUDE.md` and `AGENTS.md` and from your personal instructions; when absent, the defaults written in the skills apply (such as `docs/work/YYYY-MM-DD-<slug>/`).
- Where it runs: the Claude Code CLI and the desktop app's Code tab (local sessions). claude.ai/code web sessions do not load plugins.

## session

Three skills for ending a session and picking it up in the next one. They work for any kind of task.

```bash
claude plugin install session@idas4you --scope user
```

| Skill | Model (`model:`) | What it does |
|---|---|---|
| `/idas4you-session-check` | `sonnet` | Before you end a session, finds what is left over and shows it in a table: uncommitted changes, unpushed commits, merged local and remote branches and worktrees that are still around, a default branch that is behind, stashes, processes left running, and installed plugins that are out of date. It only finds things; it asks before cleaning anything up, and it never discards uncommitted changes |
| `/idas4you-session-summary` | `opus` | Summarizes the session as the last message in the chat: what was done, decisions, changed files and commits, verification, and remaining work. Typing "끝 정리" (wrap up) also triggers it. Renames the session to `MM-DD <main task> — <status>` (only in the desktop app, where a tool for that exists; otherwise it writes the suggested title). Saves the same summary as a file under `~/.claude/session-notes/<repo name>/` |
| `/idas4you-session-resume [topic]` | `sonnet` | Picks up a previous session. Finds this repository's summary files that still have remaining work, compares them with the current state (commits since then, branches, the work order's tasks.md), marks each item as remaining, done, or could not verify, and proposes what to start with. It does not start the work itself. Closes a summary once all of its remaining work is done |

- The order is: clean up with `/idas4you-session-check`, summarize with `/idas4you-session-summary`, then pick up in the next session with `/idas4you-session-resume`. The skills do not call each other automatically.
- Summary files accumulate in your home directory (`~/.claude/session-notes`), not in the repository. They stay out of your commits, but they are not shared across machines. Worktree sessions are grouped under the main repository's name. Files are never deleted automatically.
- In 1.1.0, `/idas4you-end-check` and `/idas4you-end-summary` from 1.0.0 were renamed to `/idas4you-session-check` and `/idas4you-session-summary`.
- If another skill also responds to "끝 정리" (for example a personal skill at `~/.claude/skills/end-summary`), which one runs is not defined. Keep only one, or call `/idas4you-session-summary` by name.

## Adding a plugin

1. Create `plugins/<name>/.claude-plugin/plugin.json` and `plugins/<name>/skills/<skill>/SKILL.md`.
2. Add an entry to `plugins` in `.claude-plugin/marketplace.json`.
3. Add a row to the table above in both READMEs (`README.md` and `README.ko.md`). When you change a plugin, bump `version` in its `plugin.json`.

## License

[MIT](LICENSE)
