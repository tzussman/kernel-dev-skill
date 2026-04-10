---
name: b4
description: Use when the user wants to submit patches to mailing lists, apply
  patches from lore, prepare a patch series, collect trailers, or compare
  series revisions. Covers b4 prep/send/trailers/am/shazam workflows. Trigger
  on "b4", "send patches", "submit to lkml", "apply from lore", "collect
  trailers", "prep series", "patch series", "send-email".
---

## Installation

```bash
which b4 && b4 --version
```

If not installed:

```bash
pip install --user b4
# or, if PEP 668 blocks it:
pip install --user --break-system-packages b4
```

Ensure `~/.local/bin` is on `$PATH`. Verify: `b4 --version`

### Dependencies for sending

b4 can send via:
1. **Web endpoint** (kernel.org's endpoint, easiest) — no local MTA needed
2. **Local SMTP** — requires git-send-email config in `~/.gitconfig`

For web endpoint (recommended for first-time setup):
```bash
b4 send --web-auth-new
# Follow the email verification flow
```

For local SMTP, configure in `~/.gitconfig`:
```ini
[sendemail]
    smtpServer = smtp.example.com
    smtpServerPort = 587
    smtpEncryption = tls
    smtpUser = you@example.com
```

## Workflow overview

b4 has two major workflows:

1. **Submitting patches** (author): `prep` → `send`
2. **Applying patches** (maintainer/reviewer): `am` or `shazam`

Plus trailer collection (`trailers`) and series comparison (`diff`).

## Submitting patches

### 1. Start a new series

```bash
# Create a new prep-tracked branch forked from a base
b4 prep -n my-feature -f origin/main

# Or enroll an existing branch
b4 prep -e origin/main
```

This creates tracking metadata in git for cover letter, version, change-id.

### 2. Edit the cover letter

```bash
b4 prep --edit-cover
```

This opens `$EDITOR`. The cover letter is stored in git's notes — not a
file in the tree. Structure:

```
Subject line for the cover letter

Body of the cover letter explaining the series.

---
Changes in v2:
- Fixed thing X
- Addressed review feedback on Y
```

**Non-interactive cover letter editing** (for tool calls):

The cover letter is stored in git notes under `refs/notes/b4-cover`. To
update it non-interactively:

```bash
# Read current cover
git notes --ref=refs/notes/b4-cover show HEAD 2>/dev/null

# Replace it
GIT_EDITOR="bash -c 'cat <<\"EOF\" > \"\$1\"
New cover letter subject

Body of cover letter.
EOF' --" b4 prep --edit-cover
```

### 3. Work on your commits

Make commits normally. b4 tracks everything between the fork point and HEAD.

### 4. Run checks before sending

```bash
b4 prep --check
```

Runs checkpatch and other local checks on the series.

### 5. Auto-populate To/Cc from MAINTAINERS

```bash
b4 prep --auto-to-cc
```

Uses `get_maintainer.pl` to populate To: and Cc: trailers in the cover
letter and individual patches based on the files touched.

### 6. Send

**Prerequisite:** b4 send requires either a signing key (`patatt genkey`)
or SMTP config. Without either, `--dry-run` and `--reflect` will fail with
"patatt.signingkey is not set". Options:
- Run `patatt genkey` to generate an ed25519 signing key (recommended)
- Configure SMTP in `~/.gitconfig` and use `--no-sign`
- Use `b4 prep -p /tmp/outdir` to just generate patches without sending

```bash
# Generate patches to a directory (always works, no SMTP/signing needed)
b4 prep -p /tmp/patches

# Preview first (requires signing key or SMTP)
b4 send --dry-run

# Send to yourself for final review
b4 send --reflect

# Actually send
b4 send

# Force through web endpoint
b4 send --use-web-endpoint
```

### 7. Version bumps (re-rolling)

After sending v1, to start v2:

```bash
# This increments the version counter and lets you update the cover
b4 prep --compare-to v1          # see range-diff from v1
b4 prep --edit-cover              # update Changes in v2 section
b4 send                           # sends as [PATCH v2 ...]
```

To force a specific version number:
```bash
b4 prep --force-revision 3
```

### 8. Add prefixes (RFC, resend, etc.)

```bash
b4 prep --set-prefixes RFC        # [RFC PATCH v1 ...]
b4 prep --add-prefixes net-next   # [PATCH net-next v1 ...]
```

## Applying patches from lore

### Quick apply (shazam)

```bash
# Apply a series directly to your tree
b4 shazam <message-id>
b4 shazam https://lore.kernel.org/.../<msgid>/

# Apply a specific version
b4 shazam -v 3 <message-id>

# Cherry-pick specific patches from a series
b4 shazam -P 1-3,5 <message-id>
```

### Download as mbox (for manual review before applying)

```bash
# Download as an am-ready mbox
b4 am <message-id>
b4 am -o /tmp/patches/ <message-id>    # save to specific dir

# Then apply manually
git am /tmp/patches/*.mbx
```

### Apply with 3-way merge support

```bash
b4 am -3 <message-id>                  # prepare 3-way merge info
```

## Collecting trailers

After posting a series and receiving Reviewed-by/Acked-by/Tested-by tags:

```bash
# Update your branch commits with trailers received on the list
b4 trailers -u

# From a specific thread
b4 trailers -u -F <message-id>

# Sloppy matching (when email addresses don't match exactly)
b4 trailers -u -S
```

## Comparing series revisions

```bash
# Range-diff between your current prep and a previously sent version
b4 prep --compare-to v1

# Diff two versions of someone else's series on lore
b4 diff <message-id>
```

## Show series info

```bash
b4 prep --show-info             # show all prep metadata
b4 prep --show-revision         # just the version number
```

## Non-interactive considerations

**b4 prep --edit-cover** opens `$EDITOR` interactively. To script it, set
`GIT_EDITOR` or `EDITOR`:

```bash
EDITOR="bash -c 'cat <<\"EOF\" > \"\$1\"
Cover letter subject

Cover body here.
EOF' --" b4 prep --edit-cover
```

**b4 send** without `--dry-run` or `--reflect` actually sends email. Always
do a dry run first from a tool call:

```bash
b4 send --dry-run 2>&1           # check what would be sent
b4 send --reflect 2>&1           # send to yourself for review
```

Only run `b4 send` (the real send) after user confirmation.

## Cleanup

```bash
b4 prep --cleanup               # archive finished prep branches
b4 prep --cleanup branch-name   # clean specific branch
```

## Common mistakes to avoid

- **Sending without `--auto-to-cc`**: patches go nowhere if To/Cc aren't
  populated. Always run `b4 prep --auto-to-cc` before sending.
- **Forgetting `--dry-run`**: never run `b4 send` from a tool call without
  the user explicitly confirming. Use `--dry-run` or `--reflect` first.
- **Editing cover interactively**: `b4 prep --edit-cover` opens an editor
  and hangs. Use the `EDITOR` override pattern above.
- **Wrong fork point**: if `b4 prep --show-info` shows unexpected commits
  in the series, the fork point is wrong. Re-enroll with the correct base.
- **Empty series after prep -n**: `b4 prep -n name -f fork` creates the
  branch with a cover letter commit but no patches. `--show-info` will
  error until you make at least one commit on the branch.
- **Missing signing key**: `b4 send --dry-run` requires either
  `patatt genkey` or SMTP config. Use `b4 prep -p /tmp/out` to generate
  patches without sending infrastructure.
- **Not running `--check` before send**: catches checkpatch issues,
  missing trailers, and other common problems.

## Decision tree

- *Start a new patch series* → `b4 prep -n name -f base`
- *Enroll existing branch* → `b4 prep -e base`
- *Edit cover letter* → `b4 prep --edit-cover` (with EDITOR override)
- *Check series before sending* → `b4 prep --check`
- *Populate To/Cc* → `b4 prep --auto-to-cc`
- *Preview what will be sent* → `b4 send --dry-run`
- *Send to yourself first* → `b4 send --reflect`
- *Actually submit* → `b4 send` (only after user confirms)
- *Apply someone's patches* → `b4 shazam <msgid>`
- *Collect review trailers* → `b4 trailers -u`
- *Compare versions* → `b4 prep --compare-to v1`
