# dream-skill

A Claude Code skill that turns a direction into a fleet of buildable PRDs. `/dream` is the generative half of an autonomous loop: where `/build` walks existing PRDs to shipped repos, `/dream` walks ideas to PRDs.

The two skills are peers, not a pipeline with a boss. `/dream` listens to the user and to what is actually happening on the host, synthesizes a coherent vision, and decomposes it into PRD-sized pieces — each citing the evidence that motivated it. `/build` picks those up and ships them. They share a working directory and an append-only gossip log, and that shared channel is the whole trick: `/dream` reads what `/build` is blocked on before it drafts, and `/build` reads `/dream`'s intent before it ships, so neither one ships something the other is about to deprecate.

It is designed to run overnight on a 30-minute cadence — night-only by design, so the dreaming happens while the user isn't working — but every invocation also runs fine by hand.

## What a pass does

`/dream` never generates from a cold start. A pass walks a fixed sequence:

1. **Listen** — the user's prompt, the tail of the gossip log, the last few days of journal, `CLAUDE_SELF.md`, `/build`'s manifest, existing visions.
2. **Check the field** — a `fallow` gate. If the inward signal hasn't moved since the last pass, rest instead of drafting a thinner fleet. (A missing `fallow` binary is treated as fresh — the gate never blocks dreaming.)
3. **Research** — probe the host for what's really there: recall seeds, repos, installed tooling, ctrace activity, recent changes, kernel primitives. Findings get cited in the PRDs; a PRD that asserts a problem without evidence is fiction.
4. **Vision** — synthesize a vision doc: end-state, PRD-sized components, dependency order, open questions. Visions accumulate; they aren't replaced.
5. **Decompose & draft** — write the PRDs, one per component in dependency order, three to seven per vision, each with testable acceptance criteria. Don't draft past what you've actually thought through.
6. **Gossip & persist** — append a note for `/build`, update the manifest, commit and push.

## Install

```sh
curl -fsSL https://raw.githubusercontent.com/j0yen/dream-skill/main/install.sh | bash
```

The installer self-clones into `~/.local/share/dream-skill/` and symlinks `~/.claude/skills/dream/` to it. Run from a checkout (`./install.sh`) instead and it symlinks that directory directly. Either way it backs up an existing non-symlink target, and rescues runtime `state/` across re-installs.

## Repo layout

```
.
├── SKILL.md                    # the spec Claude loads — phases, invocations, hard rules
├── install.sh                  # local and curl|bash installer
├── scripts/
│   └── answerable-emit.sh      # record drafts/pushes to the answerable ledger (no-op if absent)
├── state/                      # runtime manifest — gitignored
├── LICENSE-MIT
└── LICENSE-APACHE
```

## Working directory

`/dream` and `/build` both operate against a shared working directory — typically `~/wintermute/autobuilder/` — where PRDs (`PRD-*.md`), vision docs (`visions/<slug>.md`), and the gossip channel (`notes/gossip.md`) live. The skill reads and writes there; it does not create the directory.

## Invocations

| Invocation | Does |
|---|---|
| `/dream` | Interactive — listen, ask what to dream about if it isn't obvious, walk all phases |
| `/dream <topic>` | Seed the pass with a topic |
| `/dream gossip` | Print the tail of `gossip.md` and exit |
| `/dream visions` | List known visions and their status |
| `/dream from <date>` | Seed research from a specific journal date |
| `/dream extend <slug>` | Add new PRDs to an existing vision |

## Timer setup

This installer does not install the systemd-user timer. The overnight cadence lives in `~/.config/systemd/user/claude-dream.timer` on the author's machine, as explicit `OnCalendar` entries — not appropriate to ship generally. For the overnight cadence, create a user unit with your own night-window times. Disable an installed timer with `systemctl --user disable --now claude-dream.timer`; the skill stays usable by hand either way.

## See also

- [j0yen/build-skill](https://github.com/j0yen/build-skill) — the implementation counterpart that picks up the PRDs `/dream` drafts and ships them.

## License

Dual-licensed: MIT or Apache-2.0 at your option.
