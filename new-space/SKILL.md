---
name: new-space
description: Stand up or add a fleet space (a herdr workspace with a leader lane, its own AGENTS.md, role mode files, beads join and derived watcher/board registration). Use when creating a new workspace, activating an existing project as a space, onboarding a repo into the fleet, or when asked what the current end-to-end procedure is for a new space. Leader/overlord only — not a worker task.
---

# Standing up a new space

A **space** is one herdr workspace whose leader lane owns one directory. "Onboarding a repo",
"creating a workspace", "adding a space" and "standing one up" are the same job.

**Audience: the overlord, or a leader lane standing up its own space.** A worker must not run this —
it writes mode files and creates lanes. If you are a worker reading this, report it to your leader.

**Where this lives, and who can see it.** Canonical source: `~/Code/new-space-skill` (own git repo).
It is symlinked into exactly two cwd discovery roots — `/home/bitnom/Documents/Fleet/.agents/skills/new-space`
and `/home/bitnom/.overlord/.agents/skills/new-space` — because project-scope resources, including an
**ancestor `.agents/skills`**, are never encoded into the profile filter: Pi discovers them natively
whenever the project is trusted, and the profile neither adds nor excludes them
(`~/Code/pi-profile-switch/src/settings-generator.ts:27-30`). That is what keeps this skill off every
other leader lane: a HOME-level `~/.agents/skills/` copy would need listing in the `leader` profile,
which every leader lane shares. **Two consequences worth remembering:** reachability depends on
project trust being stored for those paths, not on the symlink alone; and a worker in either space
shares that cwd, so it sees the skill advertised and relies on the sentence above to know it is not
for them. Edit the repo, never a symlink target's parent.

## Who owns what — read this before §2

Running a space and *designing* one are different jobs, and the split is deliberate:

| Job | Owner |
|---|---|
| **Workspace ops** — creating the workspace/tab/pane, joining beads, launching and keeping the leader lane healthy, the watcher/board/bdi side, restarting lanes | **overlord** holds the scope for managing these across the fleet; the space's own leader runs them for its space |
| **Scaffolding and templates** — the space's `AGENTS.md` skeleton, its `.pi/modes/{leader,worker}.json` role halves, profile/skill wiring, any harness config | **self-improvement-leader**, and nobody else |

So: **do not hand-author a space's contract from scratch.** §2 describes what the scaffolding must
contain so you can review it and know what you are asking for — but the authoring is a request to
self-improvement-leader (or self-improvement-leader's own work, if that is who is reading this). A
space-by-space hand-written contract is exactly how the template drifts; the whole point of the
split is that one lane holds the shape.

**Do the phases in this order: §1 tab → §2 scaffolding → §3 beads → §4 launch.** They are numbered in
the only order that is safe, and two of the inversions fail *silently*: launching before §2 (or before
the project mode is confirmed) gives you a lane whose role contract falls back to `native` while the
footer still reads `profile: leader`, and joining beads last leaves a lane whose `bd ready` reads the
whole fleet instead of its space.

## 1. Workspace, tab and pane

The herdr **tab label is the lane's identity**, so it is set at creation and never changes: `pil`
slugs it into `PI_ROOMS_NAME`, renames the herdr agent in a detached poller and records the role. A
later `herdr tab rename` does **not** rename a running lane.

```bash
herdr tab create --workspace <w-id> --cwd <space-dir> --label <space>-leader --no-focus
#   -> {"tab":"w29:tN","label":"lfm3-leader","pane":"w29:pN"}
```

Keep `--no-focus` while configuring. Confirm the shell reached its prompt before typing into its pty:

```bash
until herdr pane read <pane> --source visible --lines 5 | grep -q '❯'; do sleep 1; done
```

**Never run `pil` from inside a lane.** It claims the name of the pane it runs in and deregisters it
on exit, deleting the real lane's registry entry. Test from a pane with no lane, or set
`PI_LANE_NAME`. A space may hold several tabs; only the `*-leader` one carries the leader role.

## 2. The space's `AGENTS.md` and its role halves — **owned by self-improvement-leader**

What follows is the *shape* of the scaffolding, written down so you can review it, request it
precisely, and tell a correct one from a broken one. **Authoring it is self-improvement-leader's**
(see "Who owns what" above).

`<cwd>/AGENTS.md` gets **only what is true for every lane in that directory**: shared facts, paths,
holds, invariants, and a `## Degraded start` section. Role-only prose does not belong there — it goes
in `<cwd>/.pi/modes/leader.json` and `worker.json`:

- `schemaVersion: 2`
- `key` equal to the fleet mode key it composes onto (`leader` / `worker`)
- `promptStrategy: "append"` — **never** `replace_base`/`replace_final` for a role delta; that
  shadows the fleet contract instead of extending it
- `systemPrompt` opening with the exact sentence the shared `## Degraded start` section names

Four absolutes, each of which has already cost someone a session:

1. **Never duplicate a rule across halves.** Both roles need it → `AGENTS.md`. One role needs it →
   that role's file only, or the other role inherits a phantom duty.
2. **Keep the core role-neutral.** No "if you are a leader" clauses in the shared file; the role
   comes from `PI_MODE`.
3. **The project half is used only once confirmed**, and an unconfirmed project mode blocks the
   **whole** composition — the lane falls back to *native*, losing its entire role contract, not
   just the space half. Startup acceptance records nothing, so the flag is part of the launch line.
4. **Editing a mode file is drift, not a re-read.** `pi-modes` re-reads definitions every turn and
   compares digests against the one recorded at activation. For a project delta composed onto a
   global contract, a changed digest drops the **space half** and keeps the global half (with a
   `project text dropped …` diagnostic). `/reload` does **not** clear mode drift.

Long form, on disk — read it, don't re-derive it: `pi-fleet-model/PER-SPACE-DECOMPOSITION.md`
§"The recipe, when a space is activated", and `pi-fleet-model/SPACE-ROLE-INSTRUCTIONS.md` for the
leader-only overlay proven through `pil`. `~/Documents/Fleet/AGENTS.md` is the worked example.

## 3. Join the shared beads database — a file, not a command

**Never `bd init`.** It targets beads' global state dir, and in a lane whose cwd is under
`/home/bitnom` the blast radius is the whole fleet.

The join is one byte-identical file: copy `~/.beads/metadata.json` to `<space>/.beads/metadata.json`

```json
{
  "database": "dolt",
  "backend": "dolt",
  "dolt_mode": "server",
  "dolt_server_host": "127.0.0.1",
  "dolt_server_port": 3307,
  "dolt_database": "swarm",
  "project_id": "07b8230e-d7cf-4f22-a338-ef727894a4c1"
}
```

`bd` then attaches by **walking up** from the cwd to the nearest `.beads/metadata.json` carrying that
shared `dolt_database` + `project_id`. **`BEADS_DIR` is not set in a live lane** (checked in `/proc`);
the env-var form is migration-plan intent, not runtime behaviour.

Then, per space:

- one **epic** labelled `project:<w-id>`;
- **every** bead carries it — bare `bd create`/`bd ready` is the whole fleet, so always scope
  (`bd ready --label project:<w-id>`);
- an **unlabelled bead is an anomaly to fix** (`bd update <id> --add-label project:<w-id>`), not to
  ignore;
- a closed bead carries the **evidence in its close reason** — `Closed` alone leaves the pane naming
  a closed bead, which is bdi's `stale-pane` anomaly.

Audit with `~/.overlord/tools/join-audit.py`.

## 4. Spawn the leader lane

```bash
herdr pane run <pane> "PI_MODE=leader PI_PROFILE=leader PI_MODE_CONFIRM_PROJECT=1 pil"
```

- **`pil`, not `pi`.** A `herdr pane run` is non-interactive and does not source rc files, so the
  `pi()`→`pil` shim in `.bashrc` does not exist there. `pi` starts an unnamed lane and makes the tab
  label decorative.
- **`PI_MODE_CONFIRM_PROJECT=1` is not optional** once §2 exists — see §2.3.
- A **worker** in that space is `PI_PROFILE=worker` and shares the leader's cwd on purpose, so it
  inherits the space's `AGENTS.md`. Without the prefix a worker silently gets the leader's profile.

## 5. Registration is derived — there is no step

Both the watcher and the board key off `*-leader` tab names, so the new space appears **automatically**
once its lane is a named pi agent:

- `~/.overlord/tools/watch-leaders.py` — persistent watcher: BLOCKED / STALE / GONE / GOAL-PAUSED /
  CTX-HIGH.
- `~/.overlord/tools/leader-board.py` — one row per leader lane. It reads lifecycle from the herdr
  API and goal state from the pane footer **on purpose**: `herdr agent get`/`agent list` carry no goal
  fields.
- **bdi:** re-run `~/.overlord/tools/beady-eye-config` when a space is added, never hand-edit. It
  emits **one `[[projects]]` block per tracker**, not per checkout.

## Verification — from artifacts, never from a status field

| Claim | How to check |
|---|---|
| Lane exists and is named | `herdr agent get <pane>` → `name` is the tab label |
| Rooms registration | `rooms_who` — **a footer name is not registration** |
| Role contract reached the model | grep the lane's **own transcript** for the role file's first sentence. The footer still reads `profile: leader` when the composition was dropped |
| Mode composed cleanly | footer `mode:<key>` with **no** `!`; `pi-modes state <session> --json` → `verdict: match` |
| Beads join | `bd where` resolves to the space, and `~/.overlord/tools/join-audit.py` is clean |
| Space visible in the portfolio | it appears in `leader-board.py` without any registration step |

`working` is not progress and `idle` is not done. A rejected composition prints **nothing** in the
pane — the transcript is the only evidence.

## Pitfalls

- **Creating the lane before its mode files** → the lane runs on the native prompt with no role
  contract, and the footer looks fine.
- **`bd init`** anywhere near this, for any reason.
- **Renaming the tab** after the lane starts — the running lane keeps the old name.
- **Running `pil` inside a lane** — deletes the real lane's registry entry.
- **Editing a mode file and leaving it unauthorised** — the space half drops until `/mode-reapprove`
  in that lane (`--confirm-project` for an unrecorded project mode) or a relaunch carrying
  `PI_MODE_CONFIRM_PROJECT=1`. Drive the lane with `~/.overlord/tools/reconcile-mode.py`, which is the
  one implementation that pins the digest and answers both consent dialogs. A digest-pinned
  `--expect` does **not** clear pi-modes' `Use project mode "…"?` gate, and a lane can sit on that
  modal while still reporting drift.
- **`fleet`** is the retired name of the `leader` profile (FT-36). Anything saying `PI_PROFILE=fleet`
  or `PI_PROFILE=orchestrator` is stale.

## Keeping the record

`pi-fleet-model/` is what the next lane inherits: `README.md` (run a lane), `EXECUTION-RECORD.md`
(what was done, with evidence), `HANDOFF-fleet-migration.md`, `OVERLORD-BRIEFING.md`. A change is not
finished until it is reloaded — `~/.overlord/tools/reload-fleet --only <pane> [--include-self]` in the
same turn. A stale claim in those docs costs someone a session; they are self-improvement-leader's to
correct.
