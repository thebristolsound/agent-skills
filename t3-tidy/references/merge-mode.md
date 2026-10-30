# Merge mode

Runs only when the arguments contain `merge`. It gathers threads that worked on one piece of work into a new top-level **hub** thread that holds a summary of all of them, then settles the threads it gathered. These steps run between step 5 (Classify) and step 6 (Apply) of `SKILL.md`, and replace the **Fold** verdict for the threads they group.

T3 Code's own `merge_back` cannot do this job: it only merges a fork back into the thread it was forked from, and a thread can hold pending merges from one fork at a time. Threads started separately are never forks of each other. The hub therefore gets its context as a first message, and that message starts one short run.

## M1. Group

Candidates are the top-level threads step 5 marked **Settle** or **Fold**. Subagent threads are never candidates: they follow their parent under the step 5 subagent rule, and step 3 never reads their tails. Put two or more candidates in one group when they share at least one piece of evidence:

- a pull request number, from the thread list or a tail;
- an issue number named in their tails;
- a branch;
- a worktree path.

A shared topic in the titles supports a grouping but never makes one by itself. A thread belongs to at most one group; when it qualifies for two, put it with the group it shares the most evidence with.

A **Keep live** thread never joins a group. When it shares evidence with a group, list it in that group's summary as related and still live, and leave it alone.

Done when every candidate is in exactly one group or in none, and every group has at least two threads.

## M2. Read each group in full

The step 3 tail is too short to replace a thread. For every thread in a group, read the messages view from 60 items before the end (the start position is exclusive: pass item count minus 61, and omit it when that is below 0), 20 items per page, about 1,500 characters per item, following the next-page position until there are no more. Read the thread's final assistant message in full: when an item comes back cut off, continue it with its item id and the next text offset until it is complete. Run step 4 on everything these reads name as pending.

Done when you hold, for every grouped thread, its goal, its final assistant message in full, and the current state of everything it left pending.

## M3. Write each group's summary

One summary per group, in this order:

1. **Goal**: the piece of work, in one or two sentences.
2. **Shipped**: what landed, with each pull request and issue at the state step 4 found.
3. **Decisions**: choices made in the threads, with the reason given there.
4. **Open items**: each still-owed step, and whether it is the user's or the hub's to do.
5. **Where the work lives**: branches and worktree paths, with their `git status` when uncommitted work remains.
6. **Source threads**: a Markdown link to every grouped thread, and to any related live thread from M1.

Keep a summary under 8,000 characters, and always under the launch tool's message limit from step 1. End it with this line, verbatim:

`Context handoff only: reply with one line confirming you have it, and take no other action.`

The hub title names the whole workstream: Title Case, six words or fewer.

Done when every group has a title and a summary within the limit.

## M4. Confirm

In a dry run, skip this step and print each group as the plan: hub title, member links, the evidence that grouped them, the workspace the hub will use, and the summary.

Otherwise show the same plan and ask the user once, with `AskUserQuestion`, whether to launch the hubs and settle the members. Make no launch and no settle before the answer. Drop any group the user names; on any answer other than yes, run the rest of the skill without merge mode.

Done when the user has confirmed, or merge mode has been dropped.

## M5. Launch, then settle

For each group, in step 6:

1. Call the launch tool with the project id, the hub title, and the summary as the message. When every member shares one worktree path, pass a workspace strategy of type existing worktree with that path; otherwise omit it, and the hub opens where the launch tool puts new threads for that project: the project root, or a new scratch folder for a project with no repository. Omit model and modes so the hub inherits this thread's. Pass a client request id of `t3-tidy-merge-` plus the thread id of the member created earliest, and on an error or lost reply retry once with the same id.
2. Only after the launch returns the new thread's id, settle every member, and then their subagents under the step 5 subagent rule.

A failed launch settles nothing in its group: report the group under **Skipped** with the error. Do not wait for the hub's reply.

Done when every confirmed group has a hub and settled members, or a recorded reason it does not.
