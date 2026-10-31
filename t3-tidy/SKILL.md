---
name: t3-tidy
description: Settle finished T3 Code threads in one project, fold duplicate threads into one, rename unclear titles, and report what each settled thread still leaves to you. With merge, gather related threads into one new hub thread that holds their summary.
disable-model-invocation: true
argument-hint: "[project name, defaults to this thread's project] [merge] [dry run]"
allowed-tools: ToolSearch Read Bash(gh *) Bash(jq *) Bash(git *) Bash(ls *) mcp__t3-code__t3_environment_read mcp__t3-code__t3_project_list mcp__t3-code__t3_thread_list mcp__t3-code__t3_thread_read mcp__t3-code__t3_thread_organize mcp__t3-code__t3_thread_update mcp__t3-code__t3_thread_launch AskUserQuestion
---

# Tidy T3 threads

Sorts one project's unsettled threads into **live** and **finished**, settles the finished ones, and hands back a list of what you still owe. Settle and rename are the only changes it makes to existing threads: both are reversible from the sidebar, and a settled thread keeps its history. Without `merge`, the skill reads threads and never sends to them, because a send starts a run. With `merge`, it also launches one hub thread per group of related threads, and each launch starts one short run; [references/merge-mode.md](references/merge-mode.md) holds those steps.

**Arguments:** `$ARGUMENTS`. A project name, matched loosely against project titles; empty means the project this thread belongs to. If the arguments contain `merge`, run merge mode (steps M1 to M5 in [references/merge-mode.md](references/merge-mode.md)) between steps 5 and 6. If the arguments contain `dry run`, go from step 5 straight to the step 7 report, written as the plan; with `merge`, run M1 to M4 first.

The steps below name each T3 tool by its role ("the thread-list tool"). Step 1 resolves every role to the tool, parameter and field names the running T3 Code version uses; use those names everywhere after it.

## 1. Map the tool surface

Read [references/tool-surface.md](references/tool-surface.md). It holds the names last checked to work and the T3 Code version they were checked against.

Load the last-known tools in one call: `ToolSearch` with `select:` and every last-known full name. For each role that did not load, search by keyword, one role per call, so a renamed tool still turns up: query `+t3` plus two or three words for the role (`+t3 settle thread`, `+t3 rename thread title`) with `max_results: 5`. One query covering every role fills its result slots with neighbouring tools and misses some roles. Match each role in the reference file's capability table to a returned tool by its description and schema, and check the parameters and enum values the table lists. Read the server version from the environment-read tool when it exists.

Then take exactly one outcome:

- **Unchanged.** Every role matches its last-known name. Continue.
- **Renamed.** Every role has a match, but some names, parameters or enum values differ. Continue with the names you found, and list each difference under **Tool surface changes** in the report. If the Settle, Rename or (with `merge`) Launch role resolved to a different tool or action value than the reference file lists, run as a dry run: go from step 5 straight to the step 7 report, and make no write until the user confirms the new tool and action. A wrong match for a write role changes threads; a wrong match for a read role only misreads them.
- **Missing.** A role has no match, or the match lacks a capability the table marks required (a settled filter, a settle action, a rename action, paging; with `merge`, a launch tool that takes a title and a first message). Stop before any write. Report which role is missing, the server version, and the closest tools you found.

The thread status enum decides which threads count as busy. A status that the reference file does not list counts as busy: keep that thread live and report the status as new.

Response fields and error strings do not appear in tool schemas, so check them on first contact. If a response lacks a field the reference file names, use the field that carries the same value and report the change; if no field carries it, stop. Match error responses by meaning, using the reference file's known strings as examples.

Done when every role has a resolved name, or the run has stopped with a report.

## 2. Inventory

Call the thread-list tool with the project's id, unsettled only, subagents included, the largest page size, and page until the response says there is no next page. A page this size usually overflows and lands in a file; query that file with `jq`. Hold per thread: id, title, link, status, item count, pull requests, parent thread id, snoozed state, updated time. From the project list, hold the project's repository as `owner/name`, or none when the project has no repository identity.

Done when every unsettled thread in the project, top-level and subagent, is on the inventory.

## 3. Read each thread's tail

Skip the current thread, snoozed threads, and busy threads (see step 1). For every other top-level thread, call the thread-read tool in its messages view, starting 15 items before the end, 5 items per page, about 450 characters per item, one run, and follow the next-page position until there are no more. The start position is exclusive: pass item count minus 16, and omit it when that is below 0, or the first item is skipped. Note the branch and worktree path the read returns.

The item count covers every timeline item, but the messages view returns only messages, so 15 items can hold no user message at all. When the window has none, read the window before it, twice as wide: if the last window started after position `p` and spanned `w` items, pass `p` minus `2w` (omitted below 0), and stop paging once positions pass `p`. Repeat until a user message turns up, the read starts at position 0, or 500 items have been read back; in the last two cases, record that the thread has no user message in that range. A user message whose `creationSource` is `server`, such as the `/compact` that often ends a thread, was not sent by the user and does not count; the thread's real ending is the assistant message before it.

The last assistant message usually holds the deciding sentence at its end. When it comes back with `textTruncated`, continue it with its item id and `textOffset` set to `nextTextOffset` until it is complete.

Done when you hold, for every top-level thread read, its last user message or a record that none was found, and its last assistant message in full.

## 4. Check the loose ends

A thread's last message is a claim about the world as it was. For every issue, PR, file, script, setting or draft that a tail names as pending, look at its current state:

- **Worktree.** Check the path with `ls -d` first. A missing path is `worktree gone`. A path where `git -C <path> rev-parse` fails is not a git repository and holds no uncommitted draft. Otherwise `git -C <path> status --short` shows uncommitted drafts.
- **Files and scripts.** `ls`.
- **Issues and PRs.** `gh issue view` or `gh pr view` with `-R owner/repo`. Take the repository from, in order: an `owner/repo` or URL the tail names, the PR's repository in the thread list, the project's repository from step 2. With none of these, mark the item `not checked: no repository`.
- **Open PRs.** Also read `gh pr view <n> -R owner/repo --json state,isDraft,reviewDecision,mergeable,statusCheckRollup`; step 5 decides on these fields.

The PR state in the thread list is a cache. When `gh` is missing or not signed in (`gh auth status` fails), use the cached state, and mark every PR line in the report `from the T3 cache, not checked`.

Done when every item you will report as still owed was checked this run, or marked as unchecked.

## 5. Classify

| Verdict | When |
|---|---|
| **Keep live** | Busy; snoozed; the current thread; an open PR that still needs work, meaning it is a draft, has changes requested, has a failing check, or is not mergeable; an open PR the tail names a next step for that the thread itself takes; an uncommitted draft the thread will finish; a question to you whose answer starts a step the thread takes ("Should I push the branch?"). A question about a step you take yourself, such as deleting a label, does not keep a thread live: record it as *still on you*. |
| **Settle** | The answer was delivered, or every remaining step is yours alone (send a message, run a script, click in a browser, review and merge a ready PR). Record that step as *still on you*. |
| **Fold** | Two or more threads cover one piece of work. Keep the one holding the latest state, rename it to name the whole workstream, settle the others, and carry their open items into its *still on you* line. |
| **Rename** | The title is cut off with `...`, is a raw lowercase prompt, or names the request rather than the work. New title: Title Case, six words or fewer, naming the work. |

Subagent threads follow their parent: settle each one whose parent is settled, archived, gone, or being settled, and that is not busy. Leave a live parent's subagents in place, since the parent may still message them. A parent missing from the inventory is settled, archived or gone, and the inventory cannot say which: read each such parent once with the thread-read tool, 1 item of 1 character, and check `settled` and `archived` under `thread`. A read that fails means the parent is gone.

Done when every thread on the inventory carries exactly one verdict, plus a rename where one applies. With `merge`, continue to M1 before step 6.

## 6. Apply

Make one change by itself first; with `merge`, that is the first hub launch. T3 refuses changes to a thread whose runtime mode is broader than this thread's; this thread's live mode can differ from what the thread list shows for it. On that refusal, make no further changes: report the plan, and ask the user to switch this thread to full access and say go.

Then make the rest in parallel. With `merge`, launch each group's hub before settling its members, as M5 describes. A settle refused because the thread is running means the thread started a run after you read it: move it to Kept, reason `started a run during cleanup`, and leave it running.

Done when every planned change returned success or carries a recorded reason it did not.

## 7. Verify and report

Re-run the inventory. Report any thread created since step 2 as `new since the start, untouched`, except the hubs this run launched.

Lead with the count, unsettled before and after. Then:

- **Kept**: each thread with one line saying why it is live.
- **Settled**: a table of thread and *still on you*, "nothing" where nothing is.
- **Renamed** and **Folded**: old title to new.
- **Merged**: only with `merge`. Each hub as a link, the members it settled, and any related thread left live.
- **Skipped**: anything that failed, and why.
- **Tool surface changes**: only when step 1 found any. Each old name to new name, any new status, and the server version, so the reference file can be updated.

Name each thread as a Markdown link built from its link field, angle brackets stripped: `[Title](t3-thread://v2/...)`.
