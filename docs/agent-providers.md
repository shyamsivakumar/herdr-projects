# Agent providers

Shepherdr starts two agent CLIs today: Claude Code (`claude`) and Codex (`codex`). This note maps every place that
knows about them, compares that with how [t3code](https://github.com/pingdotgg/t3code) handles providers, and proposes
the smallest interface that lets omp and Cursor (`cursor-agent`) run as task agents, reviewers and subagents. It is a
design note only. No runtime code changes with it.

Sources read on 2026-11-02: Shepherdr `main` at `d1cda04`, t3code at `47f2c8d`, omp 18.14.0 and cursor-agent
2026.10.22-27dcf5f on this machine.

## 1. Shepherdr's per-CLI touch points

Every line below was read in the file. "Kind" means `AgentKind`.

| Area | Where | What is CLI-specific |
| --- | --- | --- |
| Kind type | `server/core/model.ts:3` | `AgentKind = 'claude' \| 'codex'`. `CoreAgent.kind` at `:21`, `transcriptFormat?: 'codex-exec'` at `:33`. |
| Names and parsing | `server/core/settings.ts:13-15` | `CLI_NAME` maps kind to "Claude Code" / "Codex"; `kindOf` parses a word into a kind. Unknown words are refused. |
| Role defaults | `server/core/settings.ts:16-19`, `:134` | Agent row options are the two names. Reviewer and backup default to Codex, others to Claude Code. Effort options come from `TABLE.claude` or `TABLE.codex`. |
| Model catalog | `shared/models.ts:37` (`TABLE`), `:71` `parseClaude`, `:100` `parseCodex`, `:122` `parsePi` | Built-in catalogs and parsers. `server/core/settings.ts:178-183` serves them; `:142` maps the words opus/sonnet/haiku/fable for Claude only. |
| Model probe | `server/core/settings.ts:217-227` | Only Codex is probed, with `codex debug models`. `server/common/models.ts:63-110` probes claude, codex and pi but is used only by `server/demo/world.ts`. |
| `shepherdr models` | `server/core/commands.ts:55-60` | Accepts `claude\|codex`, prints `settings.clis()`, which is hard-coded to the two kinds (`settings.ts:183`). |
| `--agent` on task add | `server/core/source.ts:536` (help text), `server/core/tasks.ts:26`, `:37`, `:43` | Task kind, reviewer kind and `review.kind` are `AgentKind`. |
| Restart refusals | `server/core/source.ts:428`, `:439`, `server/core/tasks.ts:595`, `server/core/pilot-lifecycle.ts:59`, `server/core/pilot-policy.ts:40` | Each rejects any kind other than claude or codex ("Choose Claude Code or Codex"). |
| Launch argv | `server/core/agents.ts:135-161` | The whole per-CLI branch. Claude: `--settings <file>`, `--append-system-prompt-file`, `--print` and `--permission-mode plan` when read-only, `--resume`, `--model`, `--effort`. Codex: `exec --json --sandbox read-only` when read-only, else `--no-daemon --no-alt-screen --sandbox workspace-write --ask-for-approval on-request`; hooks, `developer_instructions` and `shell_environment_policy` as `-c` overrides; `resume <id>` at the end. |
| Session files | `server/core/agents.ts:90-120` | Per launch in `~/.shepherdr/sessions`: `<grant>.credential.json`, `<grant>.hooks.jsonl`, `<grant>.settings.json` (Claude hooks), `<grant>.instructions.md`. Codex gets the same hooks inline (`:153-155`). |
| Hook events | `server/core/agents.ts:19` | The Claude Code hook event names, reused for Codex minus `Notification` and `PostToolUseFailure` (`:153`). |
| Launch prompt | `server/core/agents.ts:112-120`, `:164-166` | Role words from `skill/*/SKILL.md`, then the first turn passed after `--`. |
| Hook helper | `helper/core-hook.ts:15-34` | Reads Claude/Codex hook JSON (`hook_event_name`, `session_id`, `transcript_path`, `tool_name`) and answers `PermissionRequest` in Claude's `hookSpecificOutput` shape. |
| Working, idle, blocked | `server/core/agents.ts:243-247` | `SessionStart`/`Stop`/`SessionEnd` set idle, `UserPromptSubmit`/`PreToolUse` set working, `PermissionRequest` sets blocked. |
| Status and Needs you | `server/core/source.ts:228-241`, `server/core/tasks.ts:118-121` | herdr's terminal status (`terminal.status.state`) wins over the hook state. Needs you comes from open questions, secrets or permissions, `blocked`, a changed pid, a provider error over its retries, or `logs.waiting` (idle with no `ready`). |
| Ready | `server/core/tasks.ts:413` | Set only by `shepherdr ready`, so it is the same for every CLI. |
| Permission card | `server/core/source.ts:602` | Title names "Claude Code" or "Codex". |
| Transcript readers | `server/core/logs.ts:127`, `server/core/activity.ts:59-61`, `:65-73`, `:81`, `server/common/readers/find.ts:170-176` | Picks `ClaudeReader`, `CodexReader` or `CoreReviewReader` by kind. `activity.ts:65-73` walks `~/.codex/sessions` for Codex child logs; `find.ts` tells a log's CLI from its path. |
| Codex setup line | `server/core/source.ts:276` | Codex-only hint that it may ask to trust the folder or hooks. |
| Review | `server/core/tasks.ts:745-747`, `server/core/agents.ts:136`, `:172-180`, `helper/core-review.ts:36-55`, `server/core/review-reader.ts:5-22` | Reviewer launches with `task.review.kind`. Only Codex has a headless read-only path (`codex exec --json`). The capture helper treats a run as good only after Codex's `turn.completed` event, and `CoreReviewReader` parses Codex messages. A Claude reviewer runs `--print --permission-mode plan`. |
| Takeover card | `server/core/tasks.ts:369` | Names only the two CLIs. |
| Usage | `server/core/pilot-usage.ts:76` | Codex-only session matching. |
| Isolated workspace | `helper/core-pilot-launch.ts:21`, `:41`, `helper/core-pilot-native.ts:7-8`, `helper/core-pilot-home.ts:28-30`, `helper/core-pilot-sandbox.ts:22-24`, `:73`, `helper/core-pilot-capture.ts:19`, `helper/core-pilot-records.ts:9`, `server/core/pilot-hook.ts:16` | Kind is `'claude' \| 'codex'` and other kinds are rejected. Credentials, config sanitising, native home layout, captured directories and transcript prefixes (`.claude/projects/`, `.codex/sessions/`) are per CLI. New providers stay refused in isolated workspaces until each of these is extended. |
| Herdr kinds | `server/common/settings.ts:21-26` | herdr already knows many kinds, including `cursor` and `omp` ("Oh My Pi"); this list is for the demo catalog only. |

## 2. How t3code does the same

t3code is MIT licensed (`LICENSE:1-3`, "Copyright (c) 2026 T3 Tools Inc."). We may copy code if we keep the notice.
Its adapters depend on separately licensed packages (`@anthropic-ai/claude-agent-sdk`, `@cursor/sdk`,
`@opencode-ai/sdk`); check each before adopting it. Recommendation: use t3code as reference for protocol details and
copy only small, clearly attributed pieces. Its Effect-based structure does not fit Shepherdr's code style.

The shared interface is `packages/provider-core/src/server/ProviderAdapter.ts`. `ProviderAdapterV2` (`:664`) has
`getCapabilities()` and `openSession(input)`. The session runtime (`:574`) exposes an `events` stream, `startTurn`,
`steerTurn`, `interruptTurn`, `respondToRuntimeRequest` (approvals), `resumeThread`, `readThreadSnapshot` and others.
Canonical events (`:83-181`) end each turn with `turn.terminal` (completed, interrupted, cancelled, failed). Runtime
modes are `approval-required | auto-accept-edits | auto | full-access` (`packages/contracts/src/providerPolicy.ts:25-31`).

| Provider | Start | Turn events | Approvals | Resume | Stop |
| --- | --- | --- | --- | --- | --- |
| Codex | `codex app-server`, JSON-RPC over stdio (`apps/server/src/provider/codexLaunchArgs.ts:12`) | `turn/started`, `turn/completed` | `item/*/requestApproval` server requests | `thread/resume` | `turn/interrupt` |
| Claude | `@anthropic-ai/claude-agent-sdk` `query()` (`apps/server/src/orchestration-v2/Adapters/ClaudeAdapterV2.ts`) | SDK messages; `result` ends a turn | SDK `canUseTool` callback; modes mapped to `default`/`acceptEdits`/`bypassPermissions`/`plan` | `resume: sessionId` | `interrupt()` |
| Cursor | `@cursor/sdk` `Agent.create` / `Agent.resume` (`packages/provider-cursor/src/server/CursorAgentSdk.ts`), no subprocess | `agent.send`, `run.wait()` | None: `respondToRuntimeRequest` always fails (`packages/provider-cursor/src/server/adapter.ts:2710`); only auto-review and sandbox toggles | `agentId` | `cancelRun` |
| Pi | `pi --mode rpc [--session <path>]`, JSON lines over stdio (`packages/provider-pi/src/server/mcpInjection.ts:263-270`) | `agent_start`, `message_end`, `agent_settled` | `extension_ui_request` | `switch_session` with the session file | `{type:"abort"}` |
| OpenCode | `opencode serve`, HTTP + SSE | `session.idle` settles a turn | `permission.asked` / `permission.reply` | native session id | `session.abort` |
| ACP (generic, fx) | `<bin> acp` (`packages/provider-acp`, `packages/provider-fx/src/server/acpSupport.ts`) | ACP session updates | ACP permission requests | `session/load` | ACP cancel |
| Grok | `grok agent stdio`, with `--permission-mode` or `--always-approve` per runtime mode (`packages/provider-grok/src/server/acpSupport.ts:47-56`) | ACP session updates | ACP permission requests | `session/load` | ACP cancel |
| Muse | `muse serve` through `@muse-code/sdk` (`packages/provider-muse/src/server/sdk.ts`) | SDK events | SDK requests | `session/resume` | SDK cancel |

Antigravity and the ACP registry driver are also registered (`apps/server/src/provider/builtInDrivers.ts:31-41`); they are not covered here.

t3code owns the conversation through a headless protocol and draws its own UI. Shepherdr keeps each CLI's own
terminal UI running under herdr, reads its transcript, and gets lifecycle signals from hooks. So t3code's protocol
layer is reference for headless reviewers, and its approval and resume mapping is reference for every role.

## 3. The smallest provider interface that fits both

Keep the terminal model. Replace the `kind === 'claude'` branches with one record per CLI, looked up by kind:

```ts
interface AgentProvider {
  kind: string                      // 'claude' | 'codex' | 'omp' | 'cursor'
  name: string                      // card and settings label
  binary: string                    // checked with Bun.which on the target machine
  argv(launch: LaunchInput): string[]  // interactive or read-only, model, effort, resume, instructions file, hooks file
  hooks?(launch: LaunchInput): { files: Record<string, string>; env: Record<string, string> }
  hook(input: unknown): AgentHook | undefined   // native hook JSON to Shepherdr's event names
  permissionReply?(allow: boolean): string      // stdout the CLI expects from its approval hook
  reader(): TranscriptReader | undefined        // undefined: chat shows hook events only
  headless?: {                                  // read-only reviewer; absent means the reviewer runs in a terminal
    argv(launch: LaunchInput): string[]
    settle(event: unknown): 'completed' | 'failed' | undefined  // ends the capture in helper/core-review.ts
    report(events: unknown[]): string           // the reviewer's final text for the task chat
  }
  models(machine: string): Promise<CliModels | undefined>
  efforts: string[]
}
```

`LaunchInput` is what `CoreAgents.launch` already computes: role, read-only, model, effort, previous session id, the
instructions and settings paths, the hook command and the shell environment. What the interface leaves out on purpose:
turns, steering and interrupts stay in the terminal (`terminal.send`), and `ready` stays a `shepherdr` command. A
provider without a working hook or approval path must say so, and Shepherdr then refuses it for roles that need it
(for example, no approval route means task agents run only in a mode the human chose).

## 4. omp and cursor-agent

Both are installed here. Nothing was installed or signed in for this note.

| | omp | cursor-agent |
| --- | --- | --- |
| On this machine | `/Users/shyamsivakumar/.bun/bin/omp`, `omp/18.14.0` | `/Users/shyamsivakumar/.local/bin/cursor-agent` (also `agent`), `2026.10.22-27dcf5f`, up to date per `cursor-agent about` |
| Install | `bun install -g @oh-my-pi/pi-coding-agent` (the package in `~/.bun/install/global/package.json`) | `curl https://cursor.com/install -fsS \| bash` (Cursor's documented installer); `cursor-agent update` |
| Sign in | `omp login [provider]` (OAuth per model provider), or provider API keys in the environment. `omp models` already lists models here. | `cursor-agent login`, or `CURSOR_API_KEY`. `cursor-agent status` says logged in as Shyam. |
| Terminal mode | `omp [prompt]` | `cursor-agent [prompt]` |
| Headless mode | `-p/--print`, `--mode json\|rpc\|rpc-ui`, or `omp acp` (ACP server over stdio) | `-p --output-format json\|stream-json`, or `cursor-agent acp` (ACP server) |
| Instructions | `--append-system-prompt <text or file>` | No flag. Use the first prompt or a rule file; needs a decision in its follow-up. |
| Model and effort | `--model <fuzzy or provider/id>`, `--thinking off\|minimal\|low\|medium\|high\|xhigh\|max\|auto`; catalog from `omp models` | `--model <id>` (effort as `model[effort=high]`); catalog from `cursor-agent models` or `--list-models` |
| Permissions | `--approval-mode always-ask\|always-ask-cached\|write\|yolo`, `--auto-approve`; read-only planning with `--plan-yolo` is not read-only after the plan, so use `--tools` to restrict a reviewer | Default asks; `--force`/`--yolo` runs everything; `--auto-review` lets a server classifier approve safe calls; `--mode plan\|ask` is read-only; `--sandbox enabled\|disabled`; `--trust` skips the workspace trust prompt |
| Per-launch hooks | `--hook <file>` / `-e <file>` loads an extension for that run. herdr's own extension (`~/.omp/agent/extensions/herdr-omp-agent-state.ts`) listens to `session_start`, `agent_start`, `tool_approval_requested`, `tool_approval_resolved`, `tool_execution_start`, `tool_execution_end`, `agent_end` and `session_shutdown`. | No per-launch flag in `--help`. Hooks live in `~/.cursor/hooks.json` or the project's `.cursor/hooks.json`. herdr installs a global `sessionStart` hook that exits unless `HERDR_ENV=1`; Shepherdr can do the same, gated on `SHEPHERDR_CORE_HOOK_FILE`. |
| Working, idle, done | `agent_start` is working, `tool_approval_requested` is blocked, `agent_end` is idle only when `willContinue` is not true and the error is not retryable (herdr holds working during retries, `herdr-omp-agent-state.ts:439-463`). `session_shutdown` only clears timers; process exit is the end signal. herdr already reports these to its terminal status, which Shepherdr reads first (`source.ts:236`). | herdr's hook reports only the session today. Cursor's documented events (`beforeSubmitPrompt`, `beforeShellExecution`, `afterFileEdit`, `stop` and others) would give working and idle; not yet checked on this version. |
| Approvals to a card | Unproven: the extension sees `tool_approval_requested`, but whether an extension can answer it needs a check. Fallback is `--approval-mode` chosen per role. | Unproven: `beforeShellExecution` and `beforeMCPExecution` hooks can return allow, deny or ask per Cursor's docs. File edits have no approval hook. |
| Resume | `-r/--resume <id prefix or path>`, `--continue`; the extension can read the session id and file (`ctx.sessionManager.getSessionId()`, `getSessionFile()`). Sessions are in `~/.omp/agent/sessions/<cwd>/`. | `--resume <chatId>`, `--continue`, `create-chat` returns a new id. Chats are in `~/.cursor/chats/<hash>/<chatId>/`. |
| Transcript | JSONL session file; t3code's pi RPC events are reference. A reader is new work. | SQLite chat store; no reader. Headless `stream-json` is the readable path for reviewers. |

## Installs and sign-ins Shyam must approve

- None to install: omp and cursor-agent are both present.
- Writing Shepherdr's hook into `~/.cursor/hooks.json` (global file, next to herdr's entry) needs a yes, because it
  changes every Cursor run on the machine even though it exits outside Shepherdr launches.
- Any extra omp provider login (`omp login anthropic`, `omp login openai-codex`) for the models a role should use.
- Cursor usage bills to the current Cursor plan (Ultra); running it as a reviewer or task agent spends that plan.

## Follow-up tasks

1. **Provider interface.** Add `AgentProvider` as above, move the Claude and Codex branches (including `helper/core-review.ts` and `CoreReviewReader` as Codex's `headless`) into two records, widen
   `AgentKind`, and turn every refusal in section 1 into a lookup. No behaviour change; prove with a Claude task and a
   Codex reviewer on a real task.
2. **omp provider.** Launch argv, a per-launch `--hook` extension that writes Shepherdr hook events, model catalog from
   `omp models`, resume from the session id, and an approval answer path or a documented `--approval-mode` per role.
   Read-only reviewer through `--mode json` with restricted `--tools`, with its own `headless.settle` and `report` so the capture helper ends on omp's events, not Codex's.
3. **Cursor provider.** Launch argv, a gated global hook in `~/.cursor/hooks.json` (after Shyam's yes), working and idle
   from its hook events, model catalog from `cursor-agent models`, resume by chat id, and a headless reviewer through
   `-p --mode plan --output-format stream-json` with its own `headless.settle` and `report`. Decide how role instructions reach it.
