# window-ctl-skill

```bash
git clone https://github.com/muthuishere-agent-skills/window-control
cd window-control
sh install.sh
```

**Claude can edit your repo and run your tests. It can't move the windows you're staring at while it works. This skill fixes that.**

## What it is

A Claude Code / Codex agent skill that turns natural-language window
instructions into one CLI call — across macOS, Windows, and Linux.

Ask the same thing two ways:

> **To Claude directly:** "Move chrome to the left half" → it doesn't have a tool for that.
>
> **With this skill loaded:** Claude reads your monitor layout (1-indexed, leftmost is always `1`), finds the focused Chrome window, calls `windowctl move --app "Google Chrome" --zone 1A`, and confirms what landed where. If macOS clamped the window because Chrome won't go below ~500px, you hear about it — no silent drift.

It's the routing + recipe layer over [`windowctl`](https://github.com/muthuishere/windowctl), the cross-platform window-management CLI that does the actual work.

## What it can do

| | Family | What it handles |
|---|---|---|
| 🪟 | Windows | List open app windows (filtered to real apps, not menubar widgets), find the focused one, filter by title / app |
| 🖥️ | Monitors | List displays with stable 1-indexed IDs; resolve "the laptop" / "the external" / "where my cursor is" / "where I'm typing" |
| 📐 | Move | Predefined zones (`1A` left half, `2B` top-right, `3:2` middle of three columns), absolute or monitor-relative coords, auto-resolve current monitor |
| 📏 | Resize | In-place width / height change, current X/Y preserved |
| 🎯 | Focus | Raise + activate; "focus jira", "bring chrome to front", or "focus this" via the focused-window field |
| 📦 | Batch | One JSON layout, one call. Save your arrangement as `work-mode.json`, restore it later. Multi-window placements don't run as N sequential moves |
| 🔐 | Permissions | macOS Accessibility prompt + status check, with `WCTL_AX_DEBUG=1` triage for the `window <id> is gone from the AX tree` errors that show up with multi-process apps like Chrome / Slack / Electron |

## What's in the room

11 reference files. Each is a recipe catalogue the agent loads on demand:

```
references/
  windows.md      filter open windows, find the focused one
  monitors.md     1-indexed by (X,Y), Active vs Focused
  move.md         zones, splits, coords, multi-monitor, OS-clamp handling
  resize.md       in-place sizing, focused-window resize
  batch.md        bulk layouts, save / restore, JSON case translation
  focus.md        raise + activate, "this window" via Focused
  permissions.md  AX prompt, --status --json, WCTL_AX_DEBUG triage
  zones.md        cheatsheet: 1A..2D + N:M
  recipes.md      composite layouts (split two apps, three-way IDE,
                  rescue stranded windows, save/restore named layout)
```

69 recipe-format checks + install idempotency in `tests/all.sh`. The catalogue stays internally consistent or `task validate` fails loud.

## When to use

- **Multi-window placement in one turn** — *"split chrome and slack 50/50, put terminal on monitor 2 bottom-half, send my browser to the external"* compiles to one `windowctl batch` call, not five `move` calls.
- **Save / restore named layouts** — *"save my current arrangement as work-mode"*, *"restore work-mode"*. The skill captures `windows list --json`, translates the case, writes JSON, replays via `batch`.
- **Cross-monitor moves with stable IDs** — Unplug your external; macOS renumbers system displays. The skill re-resolves by `(X, Y)` ordering — your scripts don't rot.
- **"Snap this window"** — When the user says *"this"* without naming an app, the skill reads the per-window `Focused: true` field instead of guessing.
- **Honest clamps** — Chrome refuses widths below ~500 px. The CLI exits non-zero with the actual landed bounds; the skill surfaces it instead of pretending the move worked.
- **AX-bridge triage** — `window <id> is gone from the AX tree` on Chrome / Slack / VS Code is usually a multi-process AX walk landing on the wrong PID. The skill knows to retry with `WCTL_AX_DEBUG=1` and surface the per-PID dump.

Use Claude directly when the user gives you the exact CLI flags. Use this skill when they speak in human ("the right half", "my external monitor", "this window", "save this layout").

## Requirements

| Tool | For | Install |
|---|---|---|
| **Claude Code** or **Codex** | The agent runtime loading this skill | https://claude.com/claude-code |
| **`windowctl`** | The CLI this skill drives | `npm install -g @muthuishere/windowctl` |

That's the minimum. On macOS the first `move` / `resize` / `focus` prompts for **Accessibility** permission for the parent process (the terminal / IDE that launches your agent). The skill walks you through it — the recipe lives in `references/permissions.md`.

### Linux notes

- X11: needs `wmctrl` and `xrandr` on PATH (`sudo apt install wmctrl x11-xserver-utils` or equivalent).
- Wayland: best-effort only — pixel-exact moves not guaranteed.

## Install

```bash
git clone https://github.com/muthuishere-agent-skills/window-control
cd window-control
sh install.sh
```

`install.sh` symlinks this directory into `~/.claude/skills/window-ctl-skill` (and `~/.agents/skills/window-ctl-skill` if `codex` is on PATH). Edits to `SKILL.md` and `references/*.md` are picked up live — no reinstall.

Override the Claude skills dir:

```bash
CLAUDE_SKILLS_DIR=/some/other/path sh install.sh
```

Restart your agent session afterwards so the skill is picked up.

## Uninstall

```bash
sh uninstall.sh
```

Removes the symlinks. Leaves `windowctl` and any granted macOS Accessibility trust alone.

## Typical session

```
You    : list my open windows
Skill  : windowctl windows list --json → 5 apps, 7 windows. Chrome, Slack,
         Ghostty (×2), Code, Finder. Focused: Code.

You    : split chrome left, slack right, put terminal on monitor 2
Skill  : builds batch JSON (lowercase keys), pipes to windowctl batch,
         renders one summary. 3/3 placed.

You    : save this as work-mode
Skill  : reads windows list, projects to {app, x, y, w, h}, writes
         ~/.windowctl/layouts/work-mode.json.

You    : restore work-mode
Skill  : windowctl batch --file ~/.windowctl/layouts/work-mode.json,
         5/5 placed.
```

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
- `windowctl batch [--file <path>] [--json]` — JSON array of move specs from stdin or `--file`
- `windowctl focus (--title <s> | --app <s>)`
- `windowctl permissions [--status] [--json]`

That's the entire surface. The skill never reads window lists from any other source and never calls into platform window APIs directly.

## Testing

```bash
sh tests/all.sh
```

Validates recipe format + cross-references + install idempotency. Run before committing changes to `references/*.md`.

## Troubleshooting

**Skill doesn't load after install** — Restart your Claude Code / Codex session. The agent reads skills at startup.

**`windowctl` not found** — `npm install -g @muthuishere/windowctl`. Check `$(npm prefix -g)/bin` is on your PATH.

**macOS: every move returns "Accessibility permission denied"** — Run `windowctl permissions` from the same terminal / IDE you launch the agent in (the AX grant is keyed per parent process, not per binary). Approve in System Settings, then retry. `windowctl permissions --status` reports the current trust state without prompting.

**macOS: `window <id> is gone from the AX tree`** — Re-run with `WCTL_AX_DEBUG=1` to dump the macOS AX walk to stderr; see `references/permissions.md` PERM-3 for the full diagnostic.

**Window moves to weird coordinates** — `--monitor` makes `--x/--y` relative to that monitor; without `--monitor`, they're absolute in the virtual desktop. Mixing them up is the #1 cause of off-screen windows.

**Linux: `move` returns "wmctrl: command not found"** — `sudo apt install wmctrl x11-xserver-utils` (or distro equivalent). Wayland sessions can't be controlled this way; switch to an X11 session.

## Other skills by the same author

- [**huddle**](https://github.com/muthuishere-agent-skills/huddle) — A persistent expert room for engineering decisions. 21 personas with named scars and influences. Where this skill *does* something, huddle helps you *decide* what to do.
- [**all-purpose-data-skill**](https://github.com/muthuishere-agent-skills/all-purpose-data-skill) — Mail / calendar / Teams / meetings / Drive / GitHub across Google Workspace + Microsoft 365 in one skill, via the [`apl`](https://github.com/muthuishere/all-purpose-login) OAuth broker.

Same install pattern (`sh install.sh`), same convention (skill is the routing layer; underlying tool does the work).

## Upstream

For deeper docs (per-OS adapter notes, AX trust internals, zone math derivation, the AX-bridge debugging story), see the `windowctl` repo:

**https://github.com/muthuishere/windowctl**

## License

MIT — see [`LICENSE`](./LICENSE).
