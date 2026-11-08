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

## 3. Join the shared beads database — **the recipe is the overlord's, and this is a pointer to it**

The join is a **workspace ops** step, so its single source of truth is
`overlord-operations` → §"Joining a space to the shared database", and **that section wins over this
one** if they ever disagree. What follows is the shape of the invocation, so this skill can be read
end-to-end, plus the reasoning that is *not* in that section:

```sh
mkdir -p <space>/.beads
BEADS_DIR=<space>/.beads bd init --server --server-host 127.0.0.1 --server-port 3307 \
  --database swarm -p swarm --stealth --skip-agents --non-interactive --role maintainer
```

**The hazard is a *bare* `bd init`, not `init` itself — get this distinction right, because an
earlier version of this skill got it wrong in both directions.** Bare `bd init` runs wherever
discovery lands: from a cwd whose walk-up reaches the master (`~/.beads`) it aborts, and from `$HOME`
it *is* the master's directory. `BEADS_DIR=…` removes the ambiguity by naming the target explicitly.
Equally wrong was the shortcut that replaced it: **copying `~/.beads/metadata.json` into a new
`.beads/` is not a join.** A working space's `.beads/` is a whole generated directory, not one file —
measured 2026-11-08 across 18 joined spaces:

| Present in a joined `.beads/` | Uniform? |
|---|---|
| `metadata.json` (217 B) | **yes** — one sha256 across all 18; it carries the shared `dolt_database` + `project_id` |
| `.gitignore`, `README.md`, `.local_version`, `dolt-server.port` | yes, one hash each |
| `config.yaml` (2872 B) | **yes across all 17 clients** (`6fcb85f4…`). The **master's** differs (`54d411c2…`, whose whole content is `no-git-ops: true`) — so the split is **master vs client, not space vs space** |
| `dolt/`, `dolt.gate.lock`, `embeddeddolt.gate.lock`, `last-touched` | present; `dolt/` is empty in server mode |

**Correction to an earlier version of this table:** it claimed `config.yaml` "differs between
spaces". It does not — it is uniform across all 17 clients and only the master's differs. The
conclusion below survives unchanged, but the row as written failed the first `sha256sum` a careful
reader would run, which is its own lesson about stating a hash you have not actually compared.

The shared `project_id` (`07b8230e-d7cf-4f22-a338-ef727894a4c1`) is what makes one tracker out of many
spaces, and it is the fingerprint to compare. `bd` then attaches at runtime by **walking up** from the
cwd to the nearest `.beads/metadata.json`. Note that **`BEADS_DIR` is not set in a live lane** (checked
in `/proc`) — it is *how the join is performed*, not how the attach works afterwards. Those two facts
are easy to conflate and were.

**Discovery appears to stop at a repository root, and that changes what an unjoined space does.**
Measured 2026-11-08, three data points: `bd where` in `/home/bitnom/Code/lfm3` (a git repo with no
`.beads/`) returns **`Error: No active beads workspace found`**; in `/home/bitnom/Documents/Fleet` and
`/home/bitnom` (neither is a repo) it returns the ancestor master `~/.beads`; in
`/home/bitnom/Code/synomone` (a repo, joined) it returns its own. So a **repo-shaped space that is not
joined fails loudly instead of silently reading or writing the master** — and for such a space the
join is not optional, it is the only way it sees the tracker at all. (Consistent with the observation,
not a code read; the walk-up rule for non-repo directories is the part that is directly observed.)

Then, per space:

- one **epic** labelled `project:<w-id>`;
- **every** bead carries it — bare `bd create`/`bd ready` is the whole fleet, so always scope
  (`bd ready --label project:<w-id>`);
- an **unlabelled bead is an anomaly to fix** (`bd update <id> --add-label project:<w-id>`), not to
  ignore;
- a closed bead carries the **evidence in its close reason** — `Closed` alone leaves the pane naming
  a closed bead, which is bdi's `stale-pane` anomaly.

Verify per the overlord's section (`bd dolt show` for Database/Host/Port, plus `bd list --all --json`
showing the shared graph — **not** `bd where`, which prints the local path with no `Mode:` line).
**Do not cite `~/.overlord/tools/join-audit.py` here** — it audits a different join: it reads
`bd list --has-metadata-key agent_pane` and compares it against the live herdr agents, i.e.
bead↔**pane**, not space↔database. Citing it for the database join is a mistake this file made twice.

Known and unresolved: this lane's own space (`/home/bitnom/Documents/Fleet`) has **no `.beads/` of its
own** — `bd where` there resolves to the ancestor master `~/.beads` and 297 beads are visible, so it
functions. The consequence, now that the repo-boundary rule above is measured: because Fleet is **not
a git repo**, it inherits the *master's* `metadata.json` and therefore the master's `project_id` — so
this space's beads are not separable from the machine-wide set and `project:<w-id>` scoping has nothing
of its own to scope against. Treat it as an open ops item for the overlord, not a model to copy.

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
| Role contract reached the model | **`~/.overlord/tools/verify-mode-composition.sh <pane> "<base-marker>" "<overlay-marker>" [--forbid "<marker>"]`** — use the maintained instrument first; it counts both halves in `sections.preamble`. Or read the `sections` object directly, or check the recorded digest. **Never grep the session file** — see below |
| Mode composed cleanly | footer `mode:<key>` with **no** `!`; `pi-modes state <session> --json` → `verdict: match` |
| Beads join | `bd dolt show` reports Database `swarm` / Host `127.0.0.1` / Port `3307`, and `bd list --all --json` shows the shared graph. (Not `bd where` — no `Mode:` line — and not `join-audit.py`, which is bead↔pane.) |
| Space visible in the portfolio | it appears in `leader-board.py` without any registration step |

`working` is not progress and `idle` is not done. A rejected composition prints **nothing** in the
pane.

**How to check the composition, exactly.** **Use the instrument that exists:**
`~/.overlord/tools/verify-mode-composition.sh <pane> <base-marker> <overlay-marker> [--forbid
<marker> ...]` counts the base contract and the overlay **inside `sections.preamble`** and
refuses to be fooled by tool output — it was fixed on 2026-11-02 for exactly the false positive
described below, and its own header records the same failure (`base=4/overlay=7` with a phantom
forbidden hit). If you are about to hand-roll this check, stop and run that instead.

A whole-file grep does not work and nearly produced a
false pass on 2026-11-08.** The composed prompt is not greppable text in the session file:

* the session's **system record** carries `content: ""` plus a **`sections`** object —
  `preamble` (the fleet contract), `addendum` (the space delta), `project_context`, `skills`,
  `cwd`. Read `sections`; that is where the text actually is.
* `grep` on the session file returns **echoes** — your own brief quoting the sentence, the
  lane's own greps of itself, summaries — so it reports hits even when the composition
  dropped, and it grows the longer the conversation runs.
* The strongest check needs no reading at all: the **recorded digest**. `pi-modes state
  <session> --json` → `verdict: match`, or `~/.overlord/tools/reconcile-mode.py --dry-run
  --only <pane>` → `before: match` / "already recorded the expected digest". A composed
  space half also shows `project text dropped` **zero** times in `sections`.


## Pitfalls

- **Creating the lane before its mode files** → the lane runs on the native prompt with no role
  contract, and the footer looks fine.
- **A *bare* `bd init`** — one whose target discovery decides, rather than one named with `BEADS_DIR`. That is the hazard §3 describes; `bd init` with `BEADS_DIR=<space>/.beads` is the *join*, not the hazard. An earlier version of this file said "never `bd init`" flatly and was wrong.
- **Renaming the tab** after the lane starts — the running lane keeps the old name.
- **Running `pil` inside a lane** — deletes the real lane's registry entry.
- **Editing a mode file and leaving it unauthorised** — the space half drops until `/mode-reapprove`
  in that lane (`--confirm-project` for an unrecorded project mode) or a relaunch carrying
  `PI_MODE_CONFIRM_PROJECT=1`. Drive the lane with `~/.overlord/tools/reconcile-mode.py`, which is the
  one implementation that pins the digest and answers both consent dialogs. A digest-pinned
  `--expect` does **not** clear pi-modes' `Use project mode "…"?` gate, and a lane can sit on that
  modal while still reporting drift.
- **A reload terminates the lane's background children.** `/reload`, `/profile reload`, and
  `~/.overlord/tools/reconcile-mode.py` end the lane's `bash_bg` jobs and any direct child of the
  session. Measured 2026-11-08: a pinned re-approval killed a `llama-server` 8m47s into a 256K
  prefill, and the run produced nothing. Work whose loss costs minutes — anything on real
  hardware especially — must be launched **detached** (`setsid nohup <cmd> &`) or from a worker
  pane, and a `bash_bg` handle must never be treated as durable state.
- **`fleet`** is the retired name of the `leader` profile (FT-36). Anything saying `PI_PROFILE=fleet`
  or `PI_PROFILE=orchestrator` is stale.

## Keeping the record

`pi-fleet-model/` is what the next lane inherits: `README.md` (run a lane), `EXECUTION-RECORD.md`
(what was done, with evidence), `HANDOFF-fleet-migration.md`, `OVERLORD-BRIEFING.md`. A change is not
finished until it is reloaded — `~/.overlord/tools/reload-fleet --only <pane> [--include-self]` in the
same turn. A stale claim in those docs costs someone a session; they are self-improvement-leader's to
correct.
