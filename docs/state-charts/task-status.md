# Task status

How `CoreTasks.view` in `server/core/tasks.ts` picks a working task's dot.
Only Needs you alerts the human and counts for Notify's Needs me; Waiting
does neither.

```mermaid
stateDiagram-v2
    [*] --> Working: agent starts
    Working --> NeedsYou: open card, question, secret request or permission
    Working --> NeedsYou: agent blocked, its process gone, or its terminal finished
    Working --> Waiting: turn ended, status names gate, check, review, rebase, task
    Working --> NeedsYou: turn ended, status names none of those, or asks for you
    Waiting --> Working: agent takes a new turn
    Waiting --> NeedsYou: agent opens a card or reports a status that asks for you
    NeedsYou --> Working: the human answers
    Working --> Ready: shepherdr ready
    Ready --> Merging: shepherdr merge
    Merging --> Done
```

A start error is always Needs you. Queued, To do, Ready, Merging and Done
come from the task's workflow before any of the agent's own signals.
