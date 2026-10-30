---
name: t3-tidy
description: Settle finished T3 Code threads in one project, fold duplicate threads into one, rename unclear titles, and report what each settled thread still leaves to you.
disable-model-invocation: true
argument-hint: "[project name, defaults to this thread's project] [dry run]"
allowed-tools: ToolSearch Read Bash(gh *) Bash(jq *) Bash(git *) Bash(ls *) mcp__t3-code__t3_environment_read mcp__t3-code__t3_project_list mcp__t3-code__t3_thread_list mcp__t3-code__t3_thread_read mcp__t3-code__t3_thread_organize mcp__t3-code__t3_thread_update
---

# Tidy T3 threads

Sorts one project's unsettled threads into **live** and **finished**, settles the finished ones, and hands back a list of what you still owe. Settle and rename are the only writes: both are reversible from the sidebar, and a settled thread keeps its history. The skill reads threads and never sends to them, because a send starts a run.

**Arguments:** `$ARGUMENTS`. A project name, matched loosely against project titles; empty means the project this thread belongs to. If the arguments contain `dry run`, go from step 5 straight to the step 7 report, written as the plan.

The steps below name each T3 tool by its role ("the thread-list tool"). Step 1 resolves every role to the tool, parameter and field names the running T3 Code version uses; use those names everywhere after it.

## 1. Map the tool surface

Read [references/tool-surface.md](references/tool-surface.md). It holds the names last checked to work and the T3 Code version they were checked against.

Load the last-known tools in one call: `ToolSearch` with `select:` and every last-known full name. For each role that did not load, search by keyword, one role per call, so a renamed tool still turns up: query `+t3` plus two or three words for the role (`+t3 settle thread`, `+t3 rename thread title`) with `max_results: 5`. One query covering every role fills its result slots with neighbouring tools and misses some roles. Match each role in the reference file's capability table to a returned tool by its description and schema, and check the parameters and enum values the table lists. Read the server version from the environment-read tool when it exists.

Then take exactly one outcome:

- **Unchanged.** Every role matches its last-known name. Continue.
- **Renamed.** Every role has a match, but some names, parameters or enum values differ. Continue with the names you found, and list each difference under **Tool surface changes** in the report. If the Settle or Rename role resolved to a different tool or action value than the reference file lists, run as a dry run: go from step 5 straight to the step 7 report, and make no write until the user confirms the new tool and action. A wrong match for a write role changes threads; a wrong match for a read role only misreads them.
- **Missing.** A role has no match, or the match lacks a capability the table marks required (a settled filter, a settle action, a rename action, paging). Stop before any write. Report which role is missing, the server version, and the closest tools you found.

The thread status enum decides which threads count as busy. A status that the reference file does not list counts as busy: keep that thread live and report the status as new.

Response fields and error strings do not appear in tool schemas, so check them on first contact. If a response lacks a field the reference file names, use the field that carries the same value and report the change; if no field carries it, stop. Match error responses by meaning, using the reference file's known strings as examples.

Done when every role has a resolved name, or the run has stopped with a report.

## 2. Inventory

Call the thread-list tool with the project's id, unsettled only, subagents included, the largest page size, and page until the response says there is no next page. A page this size usually overflows and lands in a file; query that file with `jq`. Hold per thread: id, title, link, status, item count, pull requests, parent thread id, snoozed state, updated time.

Done when every unsettled thread in the project, top-level and subagent, is on the inventory.

## 3. Read each thread's tail

Skip the current thread, snoozed threads, and busy threads (see step 1). For every other top-level thread, call the thread-read tool starting 15 items before the end (floor 0), 3 items per page, about 450 characters per item, one run, and follow the next-page position until there are no more. The last item is often a server-sent `/compact`; the thread's real ending is the assistant message before it. Note the branch and worktree path the read returns.

Done when you hold, for every top-level thread read, its last user message and its last assistant message.

## 4. Check the loose ends

A thread's last message is a claim about the world as it was. For every issue, PR, file, script, setting or draft that a tail names as pending, look at its current state: `git -C <worktree path> status --short` for uncommitted drafts, `ls` for files and scripts, and `gh issue view` / `gh pr view` (`-R owner/repo` outside the current repository) for GitHub items.

The PR state in the thread list is a cache. When `gh` is missing or not signed in (`gh auth status` fails), use the cached state, and mark every PR line in the report `from the T3 cache, not checked`.

Done when every item you will report as still owed was checked this run, or marked as unchecked.

## 5. Classify

| Verdict | When |
|---|---|
| **Keep live** | Busy; snoozed; the current thread; an open PR the thread will push to again; an uncommitted draft, or a question to you, that the thread acts on once you answer. |
| **Settle** | The answer was delivered, or every remaining step is yours alone (send a message, run a script, click in a browser). Record that step as *still on you*. |
| **Fold** | Two or more threads cover one piece of work. Keep the one holding the latest state, rename it to name the whole workstream, settle the others, and carry their open items into its *still on you* line. |
| **Rename** | The title is cut off with `...`, is a raw lowercase prompt, or names the request rather than the work. New title: Title Case, six words or fewer, naming the work. |

Subagent threads follow their parent: settle each one whose parent is settled or being settled, and that is not busy. Leave a live parent's subagents in place, since the parent may still message them.

Done when every thread on the inventory carries exactly one verdict, plus a rename where one applies.

## 6. Apply

Make one change by itself first. T3 refuses changes to a thread whose runtime mode is broader than this thread's; this thread's live mode can differ from what the thread list shows for it. On that refusal, make no further changes: report the plan, and ask the user to switch this thread to full access and say go.

Then make the rest in parallel. A settle refused because the thread is running means the thread started a run after you read it: move it to Kept, reason `started a run during cleanup`, and leave it running.

Done when every planned change returned success or carries a recorded reason it did not.

## 7. Verify and report

Re-run the inventory. Report any thread created since step 2 as `new since the start, untouched`.

Lead with the count, unsettled before and after. Then:

- **Kept**: each thread with one line saying why it is live.
- **Settled**: a table of thread and *still on you*, "nothing" where nothing is.
- **Renamed** and **Folded**: old title to new.
- **Skipped**: anything that failed, and why.
- **Tool surface changes**: only when step 1 found any. Each old name to new name, any new status, and the server version, so the reference file can be updated.

Name each thread as a Markdown link built from its link field, angle brackets stripped: `[Title](t3-thread://v2/...)`.
