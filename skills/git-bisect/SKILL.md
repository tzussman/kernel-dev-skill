---
name: git-bisect
description: Use when the user wants to find which commit introduced a bug or
  regression. Covers non-interactive git bisect run with vng for boot-testing,
  test script construction, boundary selection, and result interpretation.
  Trigger on "bisect", "find the commit that broke", "regression hunt",
  "which commit introduced".
---

## Important constraints

- NEVER use `git bisect` interactively (manual good/bad) from a tool call —
  it requires multiple round-trips. Always use `git bisect run <script>`.
- Always confirm the good and bad commits with the user before starting.
- Always check `git bisect log` after completion to verify the result.
- Bisect modifies HEAD — warn the user if they have uncommitted changes.

## Core workflow

### 1. Identify boundaries

Before bisecting, establish:
- **Bad commit**: a commit where the bug is present (often HEAD)
- **Good commit**: a commit where the bug is NOT present

```bash
# Check current state
git log --oneline -20

# Verify the bad commit actually fails
# Verify the good commit actually passes
```

If the user doesn't know a good commit, use exponential search:
```bash
# Test HEAD~10, HEAD~20, HEAD~40, etc. until you find a passing one
git stash  # save any work first
git checkout HEAD~10 && <test>
git checkout HEAD~20 && <test>
# etc.
git checkout -  # return to original branch
git stash pop
```

### 2. Write the test script

The script must exit 0 for good (bug absent) and exit 1 for bad (bug present).
Exit 125 means "skip this commit" (can't test, e.g., doesn't build).

#### Template: build + boot + run test

```bash
cat > /tmp/bisect_test.sh << 'SCRIPT'
#!/bin/bash
set -e

# Build — exit 125 (skip) if build fails
make LLVM=1 -j$(nproc) 2>/dev/null || exit 125

# Boot in vng and run the test — exit 1 if test fails, 0 if passes
timeout 120 vng --user root -- sh -c '<TEST_COMMAND>' 2>&1
exit_code=$?

# Timeout (exit 124) means hang — treat as bad
if [ $exit_code -eq 124 ]; then
    echo "BISECT: VM timed out — treating as bad"
    exit 1
fi

exit $exit_code
SCRIPT
chmod +x /tmp/bisect_test.sh
```

#### Template: build-only (compile regression)

```bash
cat > /tmp/bisect_test.sh << 'SCRIPT'
#!/bin/bash
make LLVM=1 -j$(nproc) 2>&1 | tail -20
exit ${PIPESTATUS[0]}
SCRIPT
chmod +x /tmp/bisect_test.sh
```

#### Template: boot-only (kernel panic/hang regression)

```bash
cat > /tmp/bisect_test.sh << 'SCRIPT'
#!/bin/bash
# Build — skip if it doesn't compile
make LLVM=1 -j$(nproc) 2>/dev/null || exit 125

# Boot with serial console and strict timeout
output=$(timeout 90 vng --user root --disable-microvm \
    --append "console=ttyS0,115200 earlyprintk=serial,ttyS0,115200 nokaslr panic=-1 oops=panic" \
    -- uname -a 2>&1)
exit_code=$?

# Check for panics/oopses in output even if exit was 0
if echo "$output" | grep -qi "kernel panic\|BUG:\|Oops:"; then
    echo "BISECT: Kernel panic/oops detected"
    echo "$output" | grep -A5 "panic\|BUG:\|Oops:" | head -20
    exit 1
fi

# Timeout = hang = bad
if [ $exit_code -eq 124 ]; then
    echo "BISECT: Boot timed out"
    exit 1
fi

exit $exit_code
SCRIPT
chmod +x /tmp/bisect_test.sh
```

#### Template: check dmesg for specific error

```bash
cat > /tmp/bisect_test.sh << 'SCRIPT'
#!/bin/bash
make LLVM=1 -j$(nproc) 2>/dev/null || exit 125

output=$(timeout 120 vng --user root -- sh -c \
    '<COMMAND>; dmesg' 2>&1)

if [ $? -eq 124 ]; then
    exit 1  # timeout = bad
fi

# Search for the specific error pattern
if echo "$output" | grep -q 'PATTERN_THAT_INDICATES_BUG'; then
    echo "BISECT: Bug pattern found"
    exit 1
fi

exit 0  # pattern not found = good
SCRIPT
chmod +x /tmp/bisect_test.sh
```

### 3. Run the bisect

```bash
# Stash uncommitted work
git stash --include-untracked -m "stash before bisect"

# Start bisect
git bisect start
git bisect bad <BAD_COMMIT>
git bisect good <GOOD_COMMIT>

# Run automated bisect
git bisect run /tmp/bisect_test.sh
```

The output will end with something like:
```
<SHA> is the first bad commit
```

### 4. Examine the result

```bash
# Show the offending commit
git show $(git bisect view --oneline | awk '{print $1}')

# Or after bisect completes:
git bisect log          # full bisect history
git log --oneline -1    # HEAD is now at the first bad commit

# Clean up
git bisect reset        # returns to original branch
git stash pop           # restore stashed work
```

### 5. Verify the result

After bisect points to a commit, verify it's correct:

```bash
# The commit bisect found
FOUND_SHA=$(git rev-parse HEAD)

# Test the commit before it (should be good)
git checkout ${FOUND_SHA}~1
/tmp/bisect_test.sh && echo "GOOD (as expected)" || echo "BAD (bisect may be wrong)"

# Test the found commit (should be bad)
git checkout ${FOUND_SHA}
/tmp/bisect_test.sh && echo "GOOD (bisect may be wrong)" || echo "BAD (as expected)"

# Return
git bisect reset
```

## Handling common issues

### Build failures at intermediate commits

Use exit code 125 in the test script to skip untestable commits.
If many commits don't build, consider `git bisect skip` ranges:

```bash
git bisect skip <SHA1> <SHA2>...<SHA3>
```

### Merge commits

`git bisect` handles merge commits automatically. If a merge commit is
the first bad commit, the regression was introduced by the merge itself
(not by any individual commit in either branch).

### Flaky tests

If the test is flaky, make the test script retry:

```bash
# In the test script, run 3 times and fail only if all 3 fail
for i in 1 2 3; do
    timeout 120 vng --user root -- <test> 2>&1 && exit 0
done
exit 1  # all 3 failed
```

### Long bisect ranges

For very large ranges (hundreds of commits), consider restricting bisect
to a specific path:

```bash
git bisect start -- mm/          # only consider commits touching mm/
git bisect start -- kernel/bpf/  # only bpf changes
```

## Recovering from a failed bisect

```bash
# Check if bisect is in progress
test -f .git/BISECT_LOG && echo "bisect in progress"

# View current state
git bisect log

# Abort and return to original branch
git bisect reset
```

## Decision tree

- *Know good and bad commits* → write test script, `git bisect run`
- *Don't know good commit* → exponential search backward (HEAD~10, ~20, ~40)
- *Build regression* → build-only test script
- *Boot/panic regression* → boot-only test script with serial console
- *Runtime bug* → build + boot + test command script
- *dmesg error* → build + boot + grep dmesg script
- *Flaky test* → add retry loop in test script
- *Too many commits* → restrict with `git bisect start -- <path>`
