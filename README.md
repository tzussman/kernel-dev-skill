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

### From a git remote (recommended)

In any Claude Code session:

```
/plugin marketplace add <git-url-of-this-repo>
/plugin install kernel-dev@kernel-dev
```

The first command registers the marketplace; the second installs the plugin.
Both the marketplace and the plugin are named `kernel-dev`, hence
`kernel-dev@kernel-dev` (`<plugin>@<marketplace>`).

### Local install (development)

If you have this repo checked out and want to run the in-tree version:

```
/plugin marketplace add /mydata/kernel-dev
/plugin install kernel-dev@kernel-dev
```

Or launch Claude Code with the plugin directory directly:

```bash
claude --plugin-dir /mydata/kernel-dev
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
│   └── marketplace.json     # marketplace catalog (points at "." — this repo)
├── skills/
│   ├── b4/SKILL.md
│   ├── git-bisect/SKILL.md
│   ├── git-rebase/SKILL.md
│   ├── kernel-build/SKILL.md
│   └── virtme-ng/SKILL.md
└── README.md
```

The repo is co-located: the marketplace's `source: "."` points back at the
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
/plugin update kernel-dev@kernel-dev
```

Bump `version` in both `plugin.json` and `marketplace.json` when cutting a
release.
