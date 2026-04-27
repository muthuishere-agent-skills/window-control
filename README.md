# window-ctl-skill

A Claude Code / Codex agent skill that turns natural-language window-management requests — *"move chrome to the left half"*, *"put slack on monitor 2"*, *"split terminal and editor 50/50"*, *"focus jira"*, *"list my open windows"* — into [`windowctl`](https://github.com/muthuishere/windowctl) commands across **macOS**, **Windows**, and **Linux**.

The skill is the *routing + recipe layer*. It knows which intent maps to which CLI invocation, how to resolve monitor IDs, the zone grammar (`1A`, `1B`, `2A..2D`, `N:M`), and the macOS Accessibility flow. The CLI itself ships separately as [`@muthuishere/windowctl`](https://www.npmjs.com/package/@muthuishere/windowctl).

---

## Requirements

| Tool | For | Install |
|---|---|---|
| **Claude Code** or **Codex** | The agent runtime loading this skill | https://claude.com/claude-code |
| **`windowctl`** | The CLI this skill drives | `npm install -g @muthuishere/windowctl` |

That's the minimum. On macOS the first `move` or `focus` prompts for **Accessibility** permission for the parent process (your terminal / IDE). The skill walks you through it via `references/permissions.md`.

### Linux notes

- X11: needs `wmctrl` and `xrandr` on PATH (`sudo apt install wmctrl x11-xserver-utils` or equivalent).
- Wayland: best-effort only — pixel-exact moves are not guaranteed.

---

## Install

```bash
git clone https://github.com/muthuishere/window-ctl-skill
cd window-ctl-skill
sh install.sh
```

`install.sh` symlinks this directory into `~/.claude/skills/window-ctl-skill` (and `~/.agents/skills/window-ctl-skill` if `codex` is on PATH). Edits to `SKILL.md` and `references/*.md` are picked up live — no reinstall needed.

Override the Claude skills dir:

```bash
CLAUDE_SKILLS_DIR=/some/other/path sh install.sh
```

Restart your agent session afterwards so the skill is picked up.

---

## Uninstall

```bash
sh uninstall.sh
```

Removes the symlinks. Leaves `windowctl` and any granted macOS Accessibility trust alone.

---

## Testing

```bash
sh tests/all.sh
```

Validates recipe format + install idempotency. Useful before committing changes to recipe files.

---

## Relationship to `windowctl`

| | `windowctl` | this skill |
|---|---|---|
| Role | Cross-platform window manager CLI / Go library | Intent router + recipe catalogue |
| Knows about OS APIs | yes (Quartz, Win32, X11) | no |
| Talks to the agent | no | yes (via SKILL.md frontmatter) |
| Distribution | npm: `@muthuishere/windowctl` | git clone + `sh install.sh` |

The skill calls `windowctl` via these subcommands only:

- `windowctl windows list [--title <s>] [--app <s>] [--json]`
- `windowctl monitors list [--json]`
- `windowctl move (--title <s> | --app <s>) [--monitor <n>] (--zone <z> | --x <n> --y <n> --w <n> --h <n>)`
- `windowctl resize (--title <s> | --app <s>) --w <n> --h <n>`
- `windowctl batch [--file <path>] [--json]` — reads a JSON array of move specs from stdin or `--file`
- `windowctl focus (--title <s> | --app <s>)`
- `windowctl permissions [--status] [--json]`

That's the entire surface. The skill never reads window lists from any other source and never calls into platform window APIs directly.

---

## Typical session

1. You: *"list my open windows"*
2. Skill runs `windowctl windows list --json`, summarizes by app.
3. You: *"put chrome on the left half"*
4. Skill runs `windowctl move --app "Google Chrome" --zone 1A`, confirms.
5. You: *"and slack on the right"*
6. Skill runs `windowctl move --app "Slack" --zone 1B`, confirms.
7. You: *"focus jira"*
8. Skill runs `windowctl focus --title "jira"`.
9. You: *"resize this to 900x700"*
10. Skill runs `windowctl resize --app "Google Chrome" --w 900 --h 700`.
11. You: *"split chrome left, slack right, and put terminal on monitor 2 bottom-half"*
12. Skill builds a JSON layout (lowercase keys: `app`, `monitor`, `zone`) and pipes it to `windowctl batch`, then renders one per-entry summary.

Multi-monitor: *"send chrome to my external display"* →
`windowctl monitors list --json` → pick the non-primary monitor by id →
`windowctl move --app "Google Chrome" --monitor <id> --zone 1A`.

---

## What's inside

- [`SKILL.md`](./SKILL.md) — agent manifest + frontmatter triggers + core rules
- [`references/windows.md`](./references/windows.md) — `windows list` recipes (filter by title / app, JSON output)
- [`references/monitors.md`](./references/monitors.md) — `monitors list` + how to resolve IDs from heuristics
- [`references/move.md`](./references/move.md) — `move` recipes (zone, split, absolute / relative coords, monitor auto-resolve)
- [`references/resize.md`](./references/resize.md) — `resize` recipes (change W/H in place, keep current X/Y)
- [`references/batch.md`](./references/batch.md) — `batch` recipes (bulk-apply / save / restore JSON layouts)
- [`references/focus.md`](./references/focus.md) — `focus` recipes (raise + activate)
- [`references/permissions.md`](./references/permissions.md) — macOS Accessibility (`windowctl permissions [--status]`)
- [`references/zones.md`](./references/zones.md) — cheatsheet for `1A`, `1B`, `2A..2D`, and `N:M` splits
- [`references/recipes.md`](./references/recipes.md) — composite layouts (split two apps, three-way IDE layout, send-to-external)

---

## Upstream

For deeper docs (per-OS adapter notes, AX trust internals, zone math derivation, layout YAML format), see the `windowctl` repo:

**https://github.com/muthuishere/windowctl**

---

## Troubleshooting

**Skill doesn't load after install** — Restart your Claude Code / Codex session. The agent reads skills at startup.

**"`windowctl` not found" in skill output** — Run `npm install -g @muthuishere/windowctl`. Check `$(npm prefix -g)/bin` is on your PATH.

**macOS: every move returns "Accessibility permission denied"** — Run `windowctl permissions` from the same terminal / IDE you launch the agent in (the AX grant is keyed per parent process, not per binary). Approve in System Settings, then retry. `windowctl permissions --status` reports the current trust state without prompting.

**macOS: `window <id> is gone from the AX tree`** — Re-run the failing command with `WCTL_AX_DEBUG=1` to dump the macOS AX walk to stderr; see `references/permissions.md` for the full diagnostic recipe.

**Window moves to weird coordinates** — `--monitor` makes `--x/--y` relative to that monitor; without `--monitor`, they are absolute in the virtual desktop. Mixing them up is the most common cause of off-screen windows.

**Linux: `move` returns "wmctrl: command not found"** — `sudo apt install wmctrl x11-xserver-utils` (or your distro equivalent). Wayland sessions cannot be controlled this way; switch to an X11 session.

---

## License

MIT — see [`LICENSE`](./LICENSE).
