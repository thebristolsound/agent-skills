# T3 Code tool surface

The names below were checked against T3 Code server `0.1.4-nightly.20261028.3841` on 2026-10-29. Step 1 of the skill compares the running server against this file. When a run reports **Tool surface changes**, update this file and the `allowed-tools` line in `SKILL.md` together.

The tools come from the T3 Code MCP server, which Claude Code names `t3-code`, so each full name is `mcp__t3-code__<tool>`.

## Capabilities

| Role | Last-known tool | What the skill needs from it | Required |
|---|---|---|---|
| Environment read | `t3_environment_read` | `serverVersion` in the response | No |
| Project list | `t3_project_list` | `id`, `title`, and `repositoryIdentity` (with `owner` and `name`, or null) per project | Yes |
| Thread list | `t3_thread_list` | Parameters `projectId`, `settled`, `includeSubagents`, `limit` (max 100), `cursor`; a `statuses` enum | Yes |
| Thread read | `t3_thread_read` | Parameters `threadId`, `view` with the value `messages`, `afterPosition`, `limit`, `maxCharsPerItem`, `runLimit`, `itemId`, `textOffset` | Yes |
| Settle | `t3_thread_organize` | Parameters `threadId`, `action` with the value `settle` | Yes |
| Rename | `t3_thread_update` | Parameters `threadId`, `action` with the value `rename`, `title` | Yes |
| Launch | `t3_thread_launch` | Parameters `projectId`, `title`, `message` (max length 120000), `clientRequestId`, `workspaceStrategy` with `type` value `existing_worktree` and `worktreePath` | With `merge` only |

## Response fields

| Tool | Fields |
|---|---|
| Thread list | `threads`, `nextCursor` (a numeric offset, null on the last page), `total`, `currentThreadId`; per thread `threadId`, `title`, `link`, `status`, `itemCount`, `pullRequests`, `parentThreadId`, `snoozed`, `updatedAt` |
| Thread read | `items` (each with `type`, `creationSource`, `text`, `position`, `textTruncated`, `nextTextOffset`, `itemId`; a user message from the server has `creationSource` `server`), `nextPosition`, `hasMore`; under `thread`, `branch`, `worktreePath`, `settled` and `archived`. `afterPosition` is exclusive |
| Settle and rename | A `sequence` number |
| Launch | `threadId`, `link`, `runId`, `status` |

## Thread statuses

| Status | Treat as |
|---|---|
| `preparing`, `queued`, `starting`, `running`, `waiting` | Busy |
| `idle`, `completed`, `interrupted`, `failed`, `cancelled`, `rolled_back` | Ended |

Any status not in this table counts as busy until this file is updated.

## Known error strings

| Meaning | Last-known text |
|---|---|
| This thread's runtime mode is narrower than the target thread's | `runtime_mode_escalation_denied` |
| The target thread started a run after it was read | a message containing `is still running` |
