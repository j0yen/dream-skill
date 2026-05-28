# dream-skill

Claude Code skill that crafts PRDs from vision. Listens to the user,
researches what's actually happening on the host, develops a coherent
vision, then writes a fleet of PRDs that build the vision out piece by
piece. Partners with `/build` via a shared gossip channel.

Designed to run overnight on a 30-min cadence (20 fires per night between
21:00 and 06:30 local). `/dream` walks ideas to PRDs;
[`build-skill`](https://github.com/j0yen/build-skill)'s `/build` walks
those PRDs to shipped repos. The two share a working directory and a
gossip log; the dance is the autonomous self-extension loop.

## Install

```sh
curl -fsSL https://raw.githubusercontent.com/j0yen/dream-skill/main/install.sh | bash
```

The installer self-clones the repo into `~/.local/share/dream-skill/` and
symlinks `~/.claude/skills/dream/` to it. Re-running picks up new commits.
Existing runtime state (`state/manifest.json`) is rescued across re-installs.

## Repo layout

```
.
├── SKILL.md         # the spec Claude loads
├── install.sh       # mode-1 (local) and mode-2 (curl|bash) installer
├── state/           # runtime — gitignored
│   └── manifest.json
├── LICENSE-MIT
└── LICENSE-APACHE
```

## Working directory

`/dream` and `/build` both expect to operate against a shared working
directory — typically `~/wintermute/autobuilder/` — where PRDs, visions,
and the `notes/gossip.md` channel live. The skill reads/writes there; it
doesn't create the directory itself.

## Timer setup

The timer unit is not installed by this script (it lives in
`~/.config/systemd/user/claude-dream.timer` on the author's machine and
isn't appropriate for general distribution). For the overnight cadence,
create a user unit with explicit `OnCalendar` entries at your preferred
night-window times.

## See also

- [j0yen/build-skill](https://github.com/j0yen/build-skill) — the
  implementation counterpart that picks up the PRDs `/dream` drafts

## License

Dual-licensed: MIT or Apache-2.0 at your option.
