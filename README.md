# agent-skills

Agent skills for [Claude Code](https://code.claude.com/docs/en/skills). Each directory holds one skill with its own `SKILL.md`.

| Skill | What it does |
|---|---|
| [t3-tidy](t3-tidy/SKILL.md) | Settles finished [T3 Code](https://github.com/pingdotgg/t3code) threads in one project, folds duplicates, renames unclear titles, and reports what each settled thread still leaves to you. |

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
```

The argument is a project name, matched loosely; leave it out to tidy the current thread's project. `dry run` stops after classifying and prints the plan without changing anything.

The skill makes two kinds of change: it settles threads and renames them. You can undo both from the T3 Code sidebar, and a settled thread keeps its history. The skill never sends a message to another thread.

T3 Code refuses changes to a thread whose runtime mode is broader than the thread running the skill. If that happens, the skill stops after its first change and asks you to switch the current thread to full access.

### When T3 Code changes

T3 Code can rename tools, parameters and statuses between versions. The skill does not depend on exact names: its first step finds the T3 tools by keyword and compares them with [`references/tool-surface.md`](t3-tidy/references/tool-surface.md), which records the names last checked and the server version they were checked against.

- If only names changed, the skill carries on with the new names and lists each change under **Tool surface changes** in its report.
- If a capability it needs is gone, the skill stops before changing anything and reports what is missing.
- A thread status the reference file does not list counts as busy, so the skill leaves that thread alone.

The `allowed-tools` line in `SKILL.md` lists tools by exact name, so a renamed tool asks for permission instead of running unprompted. That prompt is a sign the reference file is out of date. To fix it, update `references/tool-surface.md` and the `allowed-tools` line, or open an issue with the **Tool surface changes** section of the report.

## License

[MIT](LICENSE)
