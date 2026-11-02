# Agent launch

How `CoreAgents.launch` (`server/core/agents.ts`) picks a native CLI's sandbox
and approvals, and how a launch settles. Settings are read at launch time, so
a change applies to new starts and restarts only.

```mermaid
stateDiagram-v2
    [*] --> Refused: already starting, already live or previous start unconfirmed
    [*] --> Prepared: grant, credential, hook file and instructions written
    Prepared --> Pilot: Autopilot run
    Prepared --> Reviewer: role reviewer, read only
    Prepared --> Coordinator: role coordinator
    Prepared --> Task: role task (and backup)

    Pilot --> Submitted: contained launcher, native approvals and sandbox bypassed
    Reviewer --> Submitted: Codex read-only, Claude plan mode, no extra roots
    Coordinator --> Submitted: Codex workspace-write, network on, approvals on-request

    state Task {
        [*] --> Roots: Claude Code keeps its own settings
        [*] --> Roots: Codex workspace-write, network on
        Roots --> Escalations: writable_roots from task.writableRoots (none: nothing)
        Escalations --> [*]: refuse gives never, ask me gives on-request
    }
    Task --> Submitted

    Submitted --> Working: core terminal.run returns
    Submitted --> RolledBack: core refuses before submit
    RolledBack --> [*]: previous record restored, grant revoked
    Working --> NeedsYou: permission hook (on-request agents only)
    NeedsYou --> Working: human answers
    Working --> [*]: Stop or SessionEnd
```

With `refuse`, a blocked command returns an error to the agent instead of a
permission hook, so a Codex task agent does not reach Needs you through an
approval prompt; it still can through an ask card or a turn that ends
without `ready`.
