# kernel-dev

A Claude Code plugin bundling skills for Linux kernel development workflows.

This repo is **both** a plugin and a single-plugin marketplace, so users can
install it directly via `/plugin marketplace add <git-url>` followed by
`/plugin install kernel-dev@kernel-dev`.

## Skills

| Skill | Purpose |
|-------|---------|
| `b4` | Submit patches to mailing lists; `b4 prep/send/trailers/am/shazam` workflows. |
| `git-bisect` | Non-interactive `git bisect run` with vng boot-testing for kernel regressions. |
| `git-rebase` | Non-interactive rebase: reword, squash/fixup, reorder, `--autosquash`, `--onto`. |
| `kernel-build` | Build the Linux kernel with `make`, LLVM/GCC, config fragments, ccache. |
| `virtme-ng` | Boot and run commands in a kernel VM via `vng`; serial console capture. |

After install, skills are namespaced by the plugin name (e.g., `kernel-dev:b4`).

## Install

Throughout the install commands below, `kernel-dev@kernel-dev` is
`<plugin-name>@<marketplace-name>`. Both happen to be named `kernel-dev`
because this repo serves as both the plugin and its single-plugin marketplace.

### Option 1: Local install from a checked-out copy (persistent)

If you have the repo checked out at e.g. `/mydata/kernel-dev`, register it as
a local marketplace and install from there. From inside any Claude Code
session:

```
/plugin marketplace add /mydata/kernel-dev
/plugin install kernel-dev@kernel-dev
```

`/plugin marketplace add` accepts a local filesystem path to any directory
containing `.claude-plugin/marketplace.json`. The install survives across
sessions and is the right choice if you intend to use the plugin regularly
from a local clone (e.g. while iterating on it).

To pick this up after editing files in the repo, run `/reload-plugins`.

### Option 2: One-session local load (no install)

To load the plugin into a single Claude Code session without any persistent
install or marketplace registration, pass `--plugin-dir` at launch:

```bash
claude --plugin-dir /mydata/kernel-dev
```

The directory must contain `.claude-plugin/plugin.json` (it does). The flag
can be passed multiple times to load multiple plugins. Useful for quickly
trying changes without touching your installed-plugins state.

### Option 3: Install from a git remote

```
/plugin marketplace add https://github.com/tzussman/kernel-dev-skill
/plugin install kernel-dev@kernel-dev
```

### Verifying the install

Run `/plugin` to open the plugin manager. The **Installed** tab lists all
loaded plugins grouped by scope (user, project, local) and shows the source
each was loaded from — useful for confirming whether `kernel-dev` came from
your local clone, a git remote, or a `--plugin-dir` flag.

After installing, individual skills are namespaced by the plugin name, e.g.
`kernel-dev:b4`, `kernel-dev:virtme-ng`.

### Uninstall or disable

```
/plugin disable kernel-dev@kernel-dev      # temporarily disable
/plugin uninstall kernel-dev@kernel-dev    # remove completely
/reload-plugins                            # apply without restarting
```

To remove the marketplace registration itself:

```
/plugin marketplace remove kernel-dev
```

### Per-project enable

To enable this plugin only inside a specific kernel tree, commit a
`.claude/settings.json` to that tree:

```json
{
  "enabledPlugins": {
    "kernel-dev@kernel-dev": true
  }
}
```

## Layout

```
kernel-dev/
├── .claude-plugin/
│   ├── plugin.json          # plugin manifest
│   └── marketplace.json     # marketplace catalog (points at "./" — this repo)
├── skills/
│   ├── b4/SKILL.md
│   ├── git-bisect/SKILL.md
│   ├── git-rebase/SKILL.md
│   ├── kernel-build/SKILL.md
│   └── virtme-ng/SKILL.md
└── README.md
```

The repo is co-located: the marketplace's `source: "./"` points back at the
same directory as the plugin, so one git remote serves both roles.

## Dependencies

These skills assume standard kernel-dev tooling on `$PATH`:

- **b4**: `pip install --user b4`
- **virtme-ng**: `pip install --user virtme-ng` (provides `vng`)
- **kernel-build**: `make`, GCC or LLVM/clang, `ccache` recommended
- **git-bisect**, **git-rebase**: just `git`

The individual `SKILL.md` files have detailed install/verification steps.

## Updating

```
/plugin marketplace update kernel-dev    # refresh the catalog
/plugin install kernel-dev@kernel-dev    # reinstall — picks up the new version
```

Bump `version` in both `plugin.json` and `marketplace.json` when cutting a
release.
