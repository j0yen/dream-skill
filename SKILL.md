---
name: dream
description: Craft PRDs from vision. Listen to the user, research what's actually happening on this laptop, develop a coherent vision, then write a fleet of PRDs that build the vision out piece by piece. Partners with /build via a shared gossip channel at ~/wintermute/autobuilder/notes/gossip.md. Use when the user says /dream, /dream <topic>, asks Claude to "imagine the next batch of PRDs," "sketch out a vision for X," or wants to expand the autobuilder queue from a high-level idea.
model: opus
---

# /dream — vision into PRDs

`/dream` is the generative counterpart to `/build`. Where `/build` walks
existing PRDs from draft to shipped, `/dream` walks ideas to PRDs.

Cadence is **every 30 minutes overnight** (systemd-user timer
`claude-dream.timer` — 20 fires per night between 21:00 and 06:30 local,
night-only by design so dreaming happens while the user isn't actively
working). The schedule lives in `~/.config/systemd/user/claude-dream.timer`
as explicit `OnCalendar` entries; change cadence by editing those, then
`systemctl --user daemon-reload && systemctl --user restart claude-dream.timer`.
Manual invocations (`/dream …`) run independently of the timer.

The unit of work is a **vision** — a coherent direction the user (or
recent observations) points at. A vision decomposes into a **fleet** of
PRDs, each PRD-sized (one `/autobuilder` cycle's worth), each with clear
acceptance criteria, each citing what on this laptop motivated it.

Dream doesn't implement. Dream listens, researches, imagines, and writes.

The end-state is an unending web of code: tools that compose, skills
that gossip, repos that extend each other, each PRD a node in a graph
that grows toward something coherent.

## Partnership with /build

The two skills share `~/wintermute/autobuilder/`:

- **/dream writes** `PRD-*.md` files and `visions/<slug>.md` into it.
- **/build reads** PRDs and ticks them through to shipped.

They also share a gossip channel at
`~/wintermute/autobuilder/notes/gossip.md` — an append-only markdown log
both append to and both read.

Before /dream drafts, it reads recent gossip to learn what /build is
blocked on, what shipped recently, what follow-on PRDs /build drafted in
its own Phase 6. After drafting, /dream appends a note summarizing what
landed and why, plus any ordering hints or open questions for /build.

Before /build advances a PRD, it can check gossip for /dream's intent —
which PRDs belong to which vision, what depends on what, what's coming
next so /build doesn't ship something dream is about to deprecate.

The relationship is collaborative, not hierarchical. Both skills are
peers; the gossip file is where they become their best together.

## Phases

- **Phase 0 — Listen** — gather context before generating
- **Phase 0.5 — Check the field** — fallow gate before research (exit 0 → proceed; exit 1 → rest or escalate)
- **Phase 1 — Research** — probe the laptop for what's actually there

### Phase 0 — Listen

Enter the dreaming with full context, never from a cold start.

If the user gave a topic or articulation, that's the seed. Either way,
read:

- The user's most recent prompt for cues and constraints
- The tail of `~/wintermute/autobuilder/notes/gossip.md`
  (what /build has been saying)
- The last 3 days of `~/brain/journal/YYYY-MM-DD.md`
  (what's been happening)
- `~/.claude/CLAUDE_SELF.md` (how Claude operates here)
- `~/.claude/skills/build/state/manifest.json`
  (what's queued, in-flight, shipped)
- `~/wintermute/autobuilder/visions/` (what visions already exist)

Sit with this before generating. The point of listening is to notice
what the user actually wants — sometimes it's not the topic they named.

### Phase 0.5 — Check the field

Before running the full research walk, check whether the inward signal has moved:

1. Run `fallow check --json`.
   - **Exit 0 (fresh):** the evidence has moved. Proceed to Phase 1 as normal.
   - **Exit 1 (fallow), `escalate` is false:** the field is saturated. Do NOT walk Phases 1–3. Append a **one-line** gossip note: `/dream fallow — field unchanged (streak=K); rested` where K is the streak count from the JSON output. End this pass.
   - **Exit 1 (fallow), `escalate` is true:** the streak has crossed the threshold.
     - **Interactive session:** surface the outward-steer question to the user via `AskUserQuestion` with these options: (1) homeward — work on the lost-pets vision; (2) constellation — work on the fleet buildout; (3) companion-kin — imagine what a peer intelligence would want from this box; (4) name a topic. This is a built-in step, not a model-discretion choice.
     - **Non-interactive (timer-fired):** append a one-line gossip note: `/dream fallow — streak=K threshold crossed; needs user steer` and end this pass cheaply.
   - **Exit 2 (error) or binary missing:** treat as `fresh` and proceed to Phase 1. This wire must never *block* dreaming because the tool isn't installed yet.
2. **At the end of every pass** — whether drafting or resting — call:
   `fallow record --drafted <N> --seed <seed> --note <vision-slug-or-"none">`
   so the ledger and streak stay current. N=0 on a rested/fallow pass.

**Exit-code contract:** `fallow check` exits 0 (fresh), 1 (fallow), 2 (error/unknown). The `--json` output includes at least `{"fallow": bool, "streak": int, "escalate": bool}`.

**Resilience:** if `fallow` is not on `$PATH`, skip this phase entirely and proceed as fresh. Document this fallback explicitly in the phase text.

### Phase 1 — Research

Probe the laptop for what's actually there. Don't just read passively;
play with it.

**Step 1 — Seed the dream from recall (mandatory).** Before doing
anything else in this phase, query recall for ideation seeds. This is
not optional; the dreaming Claude shows itself unrelated to the user's
recent thinking when this step is skipped.

The recall store uses `<kind>/<context>` subjects. `--kind` filters
the kind axis; `--subject <substr>` filters by substring across both.

- `recall list --kind reflective --limit 20` — recent self-observations
  and half-formed thoughts (the "things I noticed but didn't act on"
  bucket — often the next PRD)
- `recall list --kind procedural --subject project --limit 20` —
  current project-state procedural notes
- `recall list --kind semantic --limit 20` — domain knowledge accrued
  across sessions (when a kernel primitive lands, an experiment closes)
- If a topic was given: `recall query "<topic>" --limit 20 --hybrid`
  — semantic match against the topic phrase
- Always: `recall query "ideas observations todo half-built notes" --limit 15 --hybrid`
  — generic ideation-flavored memories (use --hybrid so the BGE
  embedder catches paraphrases the FTS5 layer misses)

Then continue with the rest of Phase 1:

- `ls ~/wintermute/` — what repos exist; pick the relevant ones and
  skim their `README.md` and `src/lib.rs` to know the real shape
- `ls ~/.claude/skills/` and `ls ~/.local/bin/` — existing tooling
- `ctrace status` for active tracer sessions, then `ctrace query --since 24h` for what actually ran (note: `ctrace` subcommands are `start|stop|status|query|tail`; there is no `ls` or `summary`)
- `wchg list` then `wchg since <watched-dir>` for each — where the
  laptop has actually been changing this week. (Note `wchg since` is
  consuming: if `self-review` is the only thing meant to advance the
  cursor, prefer `ctrace query` instead so /dream doesn't steal the
  delta. If you do call `wchg since`, capture it once and move on.)
- `pevent list` — orphaned or long-running supervised processes are
  often the next PRD ("X observer should have been cleaned up by Y").
- `procstat self` and `procstat snap <pid>` against any recent heavy
  Claude PIDs — resource pressure points to where a tool is missing.
- Try the relevant CLIs in a sandbox (`sbx`) — does the tool already
  do what the vision wants? Where does it stop?
- Read any open PRDs in the topic area to avoid duplicating intent

Cite specific findings in the PRDs you draft. A PRD that says
"motivated by `recall` taking 500ms cold-load per query (see PRD
§10 and observed 2026-05-24 in journal)" is honest; a PRD that
asserts a problem without evidence is fiction.

### Phase 1.5 — Survey what the kernel can now do

As of 2026-05-24 the wintermute kernel exposes new primitives. A vision
that wants "where did this file come from" or "what was Claude thinking
just before it compacted" no longer has to invent userspace plumbing —
the kernel surface is there. Check what you can lean on:

- **`/dev/memlog`** (`~/wintermute/memlog/`): per-uid circular log of
  pre-compaction context snapshots. Writers in the `memlog` group;
  records survive process death. `cli/memlog show --since 1h --format
  json` is the reader. Use when proposing context-audit, episodic
  promotion, or post-mortem forensics features.

- **`provfs` LSM** (`~/wintermute/provfs/lsm/`): stamps
  `user.prov.session` and `user.prov.ts` xattrs on every closed-after-
  write file (skip-prefixed for `/proc`, `/tmp`, `.git`, `target`,
  `node_modules`). `getfattr -d <file>` reveals provenance. When
  `CONFIG_AGENT_NS=y`, `user.prov.session` carries the agentns 128-bit
  session id; otherwise `comm:<comm>:pid:<n>:uid:<n>`. Use for any
  "who wrote this file," cross-session attribution, or stale-config
  detection feature.

- **agent namespaces** (`~/wintermute/agentns/`): `unshare(CLONE_NEWAGENT)`
  gives a 128-bit session id, per-NS counters
  (syscalls/openat/write_bytes/connect/unlink/fork), an `intent_tag`
  (set via `prctl(PR_SET_AGENT_INTENT_TAG)`), and budget enforcement
  (`PR_SET_AGENT_BUDGET_LIMITS` → SIGTERM/SIGKILL on overage). The
  `/proc/$PID/agent_session`, `/proc/$PID/agent_counters`, and
  `/proc/$PID/ns/agent` surfaces are stable. Use whenever a vision
  wants per-session identity, resource accounting, or "kill the leaked
  thing" semantics without ad-hoc PID-tree walks.

- **`linux-wintermute` kernel package** (`~/wintermute/wintermute-kernel/pkg/`):
  Arch PKGBUILD that bakes all three in. `~/wintermute/wintermute-kernel/pkg/apply-agentns.py`
  is the inline-edits pattern — idempotent, anchor-based, more durable
  than unified diffs across kernel version bumps. Reuse this pattern
  for any future kernel extension; don't ship raw `.patch` files.

If a proposed PRD looks like "we need the kernel to do X" and X is
already in this list, the PRD becomes "consume X from userspace" — much
smaller scope. If X is not in this list, the PRD is honest: it
proposes a new kernel patch (drop a new file + Kconfig + Makefile +
extend `apply-agentns.py` with a new anchor block).

### Phase 2 — Vision

Synthesize. What is the user pointing at? What is the end-state in
this direction? Write or update the vision doc at
`~/wintermute/autobuilder/visions/<slug>.md`:

- **TL;DR** — one paragraph
- **End-state** — what is true when this is done?
- **Components** — PRD-sized pieces; one bullet per future PRD
- **Order** — what depends on what?
- **Open questions** — things to discuss with the user

If a vision already exists for this topic, update it; visions
accumulate, they aren't replaced.

### Phase 3 — Decompose & draft

Write the PRDs. One per component, in dependency order. Each PRD lives
at `~/wintermute/autobuilder/PRD-<slug>.md` and carries:

- `Status: Draft v0.1`
- `build_auto: false` — user opts in before /build auto-implements
- `build_target: rust-cli|rust-lib|rust-extend|shell|hooks|config|mixed`
- `build_into: <abs-path>` (required for rust-extend)
- `Vision: visions/<slug>.md` — provenance
- A TL;DR that opens with the problem this PRD solves
- A "Why this exists" section citing the research from Phase 1
- A "What this builds" section with concrete shape (modules, deps, UX)
- Numbered, testable acceptance criteria

A dream invocation may draft multiple PRDs. **Three to seven per vision
is the typical range** — fewer than three is usually a single PRD; more
than seven is usually two visions wearing one hat.

Don't draft past what you've actually thought through. If component
seven is hand-wavy, write components one through six and leave seven as
a bullet in the vision doc for the next /dream pass.

### Phase 4 — Gossip

Append to `~/wintermute/autobuilder/notes/gossip.md`:

```
## 2026-05-25T14:30  /dream  vision-<slug>
Drafted: PRD-foo.md, PRD-bar.md, PRD-baz.md
Vision: visions/<slug>.md
Order: foo → bar → baz (baz depends on bar's API)
Notes for /build: PRD-foo can ship independently; PRD-bar needs the
  recall daemon socket from PRD-recall-daemon.md so wait if that's
  still in-flight.
Open questions: should baz be a separate crate or extend foo?
```

Format is loose — what matters is that the next /build or /dream tick
can read it and act.

### Phase 5 — Persist

- Update `~/.claude/skills/dream/state/manifest.json` with the vision
  and the PRDs drafted.
- **Commit and push.** Stage every PRD-*.md, vision doc, and gossip
  update this invocation produced (specific file names; never
  `git add -A`). Commit with the Joe Yen identity:
  `git -c user.email=jyen.tech@gmail.com -c user.name="Joe Yen" commit
  -m "<message>"`. Message format:
  `dream: <N> PRDs + <M> visions from <seed>` with a brief body listing
  the slugs. Then `git push origin main`. Auto-commit is the default per
  user instruction (2026-05-27) — drafts under
  `~/wintermute/autobuilder/` are public-by-design; lag adds no review
  value. If a specific draft genuinely shouldn't ship yet, give it a
  `.draft.md` suffix and exclude it from the staged set.

## State files

```
~/wintermute/autobuilder/
├── PRD-*.md          # the fleet — read by /build, written by /dream
├── visions/          # vision docs — written by /dream
│   └── <slug>.md
└── notes/
    └── gossip.md     # shared channel — both skills read + append

~/.claude/skills/dream/state/
└── manifest.json     # which PRDs dream drafted, from which vision
```

## Manifest entry shape

```json
{
  "visions": {
    "<slug>": {
      "path": "/home/jsy/wintermute/autobuilder/visions/<slug>.md",
      "created": "2026-05-25T14:30:00Z",
      "updated": "2026-05-25T14:30:00Z",
      "prds_drafted": ["PRD-foo.md", "PRD-bar.md", "PRD-baz.md"],
      "status": "active|paused|fulfilled|abandoned",
      "seed": "user-prompt|gossip|journal|reflection"
    }
  }
}
```

A vision is **fulfilled** when /build has shipped all its drafted PRDs.
Dream marks fulfilled visions on next invocation by cross-referencing
/build's manifest.

## Invocations

- `/dream` → interactive. Listen to the user, ask what to dream about
  if it isn't obvious, then walk all phases.
- `/dream <topic>` → start with the topic as the seed.
- `/dream gossip` → print the tail of `gossip.md` and exit.
- `/dream visions` → list known visions and their status.
- `/dream from <date>` → seed research from a specific journal date.
- `/dream extend <vision-slug>` → add new PRDs to an existing vision.

## Hard rules

1. **Every PRD is buildable. No opt-outs.** Per user instruction
   2026-05-27, `build_auto` is no longer a meaningful gate — every
   PRD /dream drafts will be picked up by /build. Notebooks too. Omit
   `build_auto` from new PRDs entirely; if it appears in an older PRD
   it's ignored.
2. **Never delete or modify existing PRDs.** Only draft new ones.
   If a PRD needs replacing, draft a successor and let the user
   archive the old one.
3. **Cite the research.** Every PRD's "Why" section must reference
   specific evidence from Phase 1 (a file, a recall hit, a ctrace
   summary, a journal entry, a gossip note).
4. **Visions are durable.** Update them; don't silently replace them.
5. **Gossip is append-only.** Never rewrite history in `gossip.md`.
6. **Don't dream past the research.** If the laptop doesn't motivate
   a component, the component isn't real yet — leave it in the vision
   doc as an open question.
7. **Use Joe Yen identity** for /wintermute commits.
   `git -c user.email=jyen.tech@gmail.com -c user.name="Joe Yen"`.
   (Updated 2026-05-27: commits + pushes from /dream are now the
   default per Phase 5, not user-gated. The identity rule still
   applies to every such commit.)
8. **Check the field first.** A saturated field (`fallow check` exits 1) means rest, not a thinner fleet of PRDs. Never draft past `fallow check`. A missing `fallow` binary is treated as fresh — the gate must never prevent dreaming.

## Disable

```
systemctl --user disable --now claude-dream.timer
```

Re-enable with `enable --now`. The skill itself stays usable manually
either way.
