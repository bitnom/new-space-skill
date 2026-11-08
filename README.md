# new-space — the fleet's space-onboarding procedure

The `new-space` agent skill: the current, end-to-end procedure for standing up or adding a fleet
space (a herdr workspace with a leader lane, its own `AGENTS.md`, role mode files, a beads join, and
derived watcher/board registration).

## Why this is its own repo

The skill is deliberately scoped to **two lanes** — `self-improvement-leader` (which designs space
scaffolding) and `overlord` (which holds the scope for workspace ops). Keeping the canonical copy out
of either lane's cwd is what makes that scoping honest: the two cwds carry symlinks, and nothing else
does.

## Layout

```
new-space-skill/
└── new-space/
    └── SKILL.md          # the skill package (a directory containing SKILL.md)
```

## Installing it (how the symlinks were made)

```bash
ln -sfn ~/Code/new-space-skill/new-space ~/Documents/Fleet/.agents/skills/new-space
ln -sfn ~/Code/new-space-skill/new-space ~/.overlord/.agents/skills/new-space
```

Why this works, and why it is not the obvious thing to do: project-scope resources — including an
**ancestor `.agents/skills`** — are never encoded into the profile filter. Pi discovers them natively
whenever the project is trusted, and the profile neither adds nor excludes them
(`~/Code/pi-profile-switch/src/settings-generator.ts:27-30`). A skill placed in the HOME-level
`~/.agents/skills/` instead would load only when the profile's `skills` array lists it — and the
`leader` profile is shared by every leader lane in the fleet, so it would have gone to all of them.

Two consequences to keep in mind:

- reachability depends on **project trust** being stored for those two paths, not on the symlink alone;
- a **worker** in either space shares that cwd, so it sees the skill advertised too — hence the
  audience sentence at the top of the skill. Project-scope skills cannot be narrowed per role.

## Editing

Edit `new-space/SKILL.md` in this repo and commit. Never edit through a symlinked path in a lane's
cwd — the lanes only carry links.

## Verifying it reaches both lanes

```bash
find /home/bitnom -name new-space -not -path "*/.git/*" -not -path "*/node_modules/*"
# expect exactly: this repo's new-space/, plus the two .agents/skills/new-space symlinks

readlink -f ~/.overlord/.agents/skills/new-space/SKILL.md   # -> this repo
sha256sum new-space/SKILL.md
```

A lane picks up a new skill on `/reload` (or a new session). For another lane, use
`~/.overlord/tools/reload-fleet --dry-run` first — a `/reload` sent into a working lane's pane can
abort its turn.
