---
name: git-rebase
description: Use when the user wants to rebase commits, reword commit messages,
  squash/fixup commits, reorder patches, or resolve rebase conflicts. Covers
  non-interactive rebase via GIT_SEQUENCE_EDITOR, --autosquash, --onto, and
  sequencer state recovery. Trigger on "rebase", "squash", "fixup", "reword",
  "reorder commits", "edit commit message".
---

## Important constraints

- NEVER use `git rebase -i` bare -- it opens an interactive editor. Always
  pair it with `GIT_SEQUENCE_EDITOR` to script the todo list.
- NEVER use `--no-edit` with `git rebase` -- it is not a valid rebase flag.
- NEVER amend a commit you didn't just create -- use reword instead.
- Always confirm the branch and range with `git log --oneline` before
  rebasing.
- Always run `git status` and `git log --oneline -5` after a rebase to
  verify the result.

## Core patterns

### 1. Reword a commit message

To change the message of a specific commit without touching its content.

**CRITICAL: `reword` requires BOTH `GIT_SEQUENCE_EDITOR` AND `GIT_EDITOR`.**
`GIT_SEQUENCE_EDITOR` marks the commit for reword in the todo list.
`GIT_EDITOR` supplies the new commit message when git opens the editor.
Without `GIT_EDITOR`, git uses the default editor and keeps the old message.

**Single commit reword (tested, works):**

```bash
GIT_SEQUENCE_EDITOR="sed -i 's/^pick <SHORT_SHA>/reword <SHORT_SHA>/'" \
GIT_EDITOR="bash -c 'printf \"%s\n\" \"new subject line\" \"\" \"Optional body.\" > \"\$1\"' --" \
    git rebase -i <TARGET_SHA>~1
```

**For the most recent commit only**, `git commit --amend -m "new message"`
is simpler and correct.

**For rewording multiple commits**, you need a different `GIT_EDITOR` script
for each stop. The most reliable approach is a stateful editor script:

```bash
cat > /tmp/reword_editor.sh << 'SCRIPT'
#!/bin/bash
# Reword editor that cycles through messages
MSG_DIR=/tmp/reword_msgs
COUNTER_FILE="$MSG_DIR/counter"
count=$(cat "$COUNTER_FILE" 2>/dev/null || echo 0)
count=$((count + 1))
echo "$count" > "$COUNTER_FILE"
cat "$MSG_DIR/msg_${count}" > "$1"
SCRIPT
chmod +x /tmp/reword_editor.sh

# Set up messages (one file per reword stop, numbered in order)
mkdir -p /tmp/reword_msgs
rm -f /tmp/reword_msgs/counter
printf '%s\n' "first new subject" "" "First body." > /tmp/reword_msgs/msg_1
printf '%s\n' "second new subject" "" "Second body." > /tmp/reword_msgs/msg_2

GIT_SEQUENCE_EDITOR="sed -i -e 's/^pick <SHA1>/reword <SHA1>/' -e 's/^pick <SHA2>/reword <SHA2>/'" \
GIT_EDITOR="/tmp/reword_editor.sh" \
    git rebase -i <OLDEST_SHA>~1
```

### 2. Squash / fixup commits

#### Using --autosquash (preferred for targeted squash)

```bash
# Create a fixup commit that targets a specific prior commit
git commit --fixup=<TARGET_SHA>

# Rebase with autosquash -- fixup commits auto-reorder and squash
GIT_SEQUENCE_EDITOR="true" git rebase -i --autosquash <TARGET_SHA>~1
```

`--fixup` silently folds into the target. Use `--squash` instead if you
want to edit the combined message.

#### Manual squash via GIT_SEQUENCE_EDITOR

```bash
# Squash the last 3 commits into one
GIT_SEQUENCE_EDITOR="sed -i '2,3s/^pick/squash/'" \
    git rebase -i HEAD~3
```

This marks commits 2 and 3 as `squash` (folded into commit 1). Git will
open an editor for the combined message -- to also script that:

```bash
# Use fixup instead of squash to keep only the first commit's message
GIT_SEQUENCE_EDITOR="sed -i '2,3s/^pick/fixup/'" \
    git rebase -i HEAD~3
```

### 3. Reorder commits

Inline heredocs in `GIT_SEQUENCE_EDITOR` are fragile (quoting issues, shell
escaping). **Use a script file instead** (tested, works reliably):

```bash
cat > /tmp/reorder_editor.sh << SCRIPT
#!/bin/bash
cat > "\$1" << TODO
pick <SHA_third> third commit subject
pick <SHA_first> first commit subject
pick <SHA_second> second commit subject
TODO
SCRIPT
chmod +x /tmp/reorder_editor.sh

GIT_SEQUENCE_EDITOR="/tmp/reorder_editor.sh" git rebase -i HEAD~3
```

This also works for any complex todo rewrite (mixing pick/reword/squash/drop).

### 4. Drop a commit

```bash
GIT_SEQUENCE_EDITOR="sed -i '/<SHORT_SHA>/d'" \
    git rebase -i <SHA>~1
```

Or use `drop` instead of deleting the line:

```bash
GIT_SEQUENCE_EDITOR="sed -i 's/^pick <SHORT_SHA>/drop <SHORT_SHA>/'" \
    git rebase -i <SHA>~1
```

### 5. Move commits to a different base (--onto)

```bash
# Move commits from old_base to new_base
# This replays commits (old_base..branch] onto new_base
git rebase --onto <new_base> <old_base> <branch>
```

Common use cases:
- Rebase a feature branch onto updated main:
  `git rebase --onto main old_main_sha feature-branch`
- Extract a sub-range of commits onto a new base:
  `git rebase --onto target start_exclusive end_inclusive`

### 6. No-op rebase (accept todo as-is)

When you need a rebase -i but don't want to change anything (e.g., just
to trigger --autosquash):

```bash
GIT_SEQUENCE_EDITOR="true" git rebase -i --autosquash <base>
```

`true` is a shell builtin that exits 0, leaving the todo list unchanged.

## Conflict handling

### Detect rebase in progress

```bash
# Check for active rebase
test -d .git/rebase-merge && echo "rebase in progress" || echo "clean"
```

### Inspect sequencer state

```bash
# What commit is being applied
cat .git/rebase-merge/stopped-sha 2>/dev/null

# How far along
cat .git/rebase-merge/msgnum 2>/dev/null    # current step
cat .git/rebase-merge/end 2>/dev/null       # total steps

# Remaining todo
cat .git/rebase-merge/git-rebase-todo 2>/dev/null

# Original branch being rebased
cat .git/rebase-merge/head-name 2>/dev/null
```

### Resolve and continue

```bash
# After fixing conflicts in the working tree:
git add <resolved_files>
git rebase --continue

# If the commit should be skipped entirely:
git rebase --skip

# If the rebase should be abandoned:
git rebase --abort
```

### Scripting the commit message during conflict resolution

When `git rebase --continue` would open an editor (e.g., after a squash),
set `GIT_EDITOR`:

```bash
GIT_EDITOR="true" git rebase --continue          # keep existing message
GIT_EDITOR="cat <<'MSG' > \"\$1\"
new message
MSG" git rebase --continue                         # replace message
```

## Kernel-specific notes

- Kernel commit messages must have a blank line after the subject
- `Fixes:` tags use the 12-char SHA format: `Fixes: xxxxxxxxxxxx ("subject")`
- When rewording to add `Fixes:` or `Cc: stable`, put them in the trailer
  block (after the body, before `Signed-off-by`)
- Do NOT add or modify `Signed-off-by` -- only the author may certify DCO
- Series rebase onto updated base: prefer `git rebase --onto` over
  cherry-pick to preserve the linear history that `git format-patch` expects

## Decision tree

- *Change last commit message* -> `git commit --amend -m "..."`
- *Change older commit message* -> GIT_SEQUENCE_EDITOR reword pattern
- *Fold a fix into a prior commit* -> `git commit --fixup` + `--autosquash`
- *Squash N recent commits* -> GIT_SEQUENCE_EDITOR squash/fixup pattern
- *Reorder commits* -> GIT_SEQUENCE_EDITOR with explicit todo list
- *Move branch to new base* -> `git rebase --onto`
- *Rebase is stuck / in progress* -> read `.git/rebase-merge/` state, then
  `--continue`, `--skip`, or `--abort`

## Common mistakes to avoid

- **Reword without `GIT_EDITOR`**: `reword` in the todo list only tells git
  to stop and open an editor. If you only set `GIT_SEQUENCE_EDITOR` but not
  `GIT_EDITOR`, git uses the default editor which may silently keep the old
  message. Always set both for non-interactive reword.
- **Inline heredocs in `GIT_SEQUENCE_EDITOR`**: shell quoting and escaping
  makes these fragile. Write a script file to `/tmp/` instead -- it's more
  reliable and easier to debug.
- **Using `--no-edit` with rebase**: not a valid rebase flag, will error
- **Bare `git rebase -i`**: opens interactive editor, hangs in non-interactive
  contexts. Always use `GIT_SEQUENCE_EDITOR`.
- **Wrong base SHA**: always use `<target>~1` (parent) as the base, not the
  target commit itself -- otherwise the target won't appear in the todo list
- **Forgetting `GIT_SEQUENCE_EDITOR="true"` with --autosquash**: without it,
  git opens an editor to confirm the todo list
- **Sed on wrong line numbers**: the todo list order can shift if git adds
  comments. Prefer matching on SHA over line numbers when possible.
