# agent-skills

Agent skills for [Claude Code](https://code.claude.com/docs/en/skills). Each directory holds one skill with its own `SKILL.md`.

| Skill | What it does |
|---|---|
| [t3-tidy](t3-tidy/SKILL.md) | Settles finished [T3 Code](https://github.com/pingdotgg/t3code) threads in one project, folds duplicates, renames unclear titles, and reports what each settled thread still leaves to you. With `merge`, it gathers related threads into one hub thread. |

## Install

Clone the repository and link the skill into your personal skills directory:

```sh
git clone https://github.com/thebristolsound/agent-skills.git ~/agent-skills
ln -s ~/agent-skills/t3-tidy ~/.claude/skills/t3-tidy
```

A symlink picks up updates on `git pull`. To install without one, copy the `t3-tidy` directory into `~/.claude/skills/` instead.

## t3-tidy

### Requirements

- You run Claude Code inside T3 Code, so the T3 Code MCP tools (`mcp__t3-code__*`) are available in the session.
- The [GitHub CLI](https://cli.github.com/) is optional. With `gh` signed in, the skill checks each pull request's live state. Without it, the skill uses the state T3 Code cached and marks those lines as unchecked.

### Usage

Run it from a thread in the project you want to tidy:

```text
/t3-tidy
/t3-tidy dry run
/t3-tidy gridcast dry run
/t3-tidy merge
/t3-tidy merge dry run
```

The argument is a project name, matched loosely; leave it out to tidy the current thread's project. `dry run` stops after classifying and prints the plan without changing anything.

By default the skill makes two kinds of change: it settles threads and renames them. You can undo both from the T3 Code sidebar, and a settled thread keeps its history. Without `merge`, the skill never sends a message to another thread.

### Merge mode

`merge` finds threads that worked on the same thing and gathers them into one new top-level hub thread. Threads are grouped by shared evidence: a pull request, an issue number, a branch or a worktree. A similar title alone does not group them, and threads that are still live never join a group.

For each group, the skill reads the threads in full and writes a summary covering the goal, what shipped, decisions, open items, where the work lives, and links to every source thread. It shows you all the groups and asks once before changing anything. After you confirm, it launches a hub thread with the summary as its first message, and then settles the grouped threads. If a launch fails, the skill settles nothing in that group.

Each hub launch starts one short run, in which the hub replies with a one-line confirmation. `merge dry run` prints the groups and summaries without launching or settling anything.

T3 Code's built-in `merge_back` is not used: it only merges a fork back into the thread it came from, and threads started separately are never forks of each other.

T3 Code refuses changes to a thread whose runtime mode is broader than the thread running the skill. If that happens, the skill stops after its first change and asks you to switch the current thread to full access.

### When T3 Code changes

T3 Code can rename tools, parameters and statuses between versions. The skill does not depend on exact names: its first step loads the tool names listed in [`references/tool-surface.md`](t3-tidy/references/tool-surface.md), which records the names last checked and the server version they were checked against. For any tool that does not load, it searches by keyword, one capability at a time, and compares what it finds with the reference file.

- If only names changed, the skill carries on with the new names and lists each change under **Tool surface changes** in its report.
- If the tool or action that settles or renames a thread changed, the skill runs as a dry run and waits for you to confirm the new names before changing anything.
- If a capability it needs is gone, the skill stops before changing anything and reports what is missing.
- A thread status the reference file does not list counts as busy, so the skill leaves that thread alone.

The `allowed-tools` line in `SKILL.md` lists tools by exact name, so outside bypass and auto permission modes, a renamed tool asks for permission instead of running unprompted. That prompt is a sign the reference file is out of date. In bypass and auto modes there is no prompt, and the dry-run rule above is the safeguard. To fix it, update `references/tool-surface.md` and the `allowed-tools` line, or open an issue with the **Tool surface changes** section of the report.

## License

[MIT](LICENSE)
