# Fixes: Tag Verification

This prompt provides detailed instructions for verifying Fixes: tags when they appear in commit messages.

## Purpose of Fixes: Tags

A Fixes: tag indicates that a patch fixes a bug in a previous commit. The tag:
- Makes it easy to determine where an issue originated
- Helps reviewers understand the bug fix context
- Assists maintainers in deciding which Lustre maintenance branches (e.g.
  b2_15, b2_16) should receive a cherry-pick of the fix
- Should be included even for bugs that won't be backported

**TodoWrite format** (one entry per Fixes: tag):
```
Fixes tag: [full tag text]
SHA-1: [commit ID] - length [N chars], exists ✓/✗ (git cat-file -t), reachable ✓/✗ (git merge-base)
Format: quotes ✓/✗, single line ✓/✗, location [sign-off area/below ---/other]
Subject: matches original ✓/✗ - [show both if different]
Bug fixed: ✓/✗/unclear - [reasoning]
Issues: [none OR list]
```

## Format Requirements [FIXES-001]

**Risk**: Parsing failures, incorrect backport scope

**Mandatory format validation:**

Track each Fixes: tag in the TodoWrite and verify:

1. **SHA-1 Length Check**
   - Check that the SHA-1 has at least 10 hex characters (Lustre accepts 10+;
     don't flag a 10- or 11-char hash as too short)
   - Verify hexadecimal characters only, and that it resolves unambiguously
   - Example: `9ce6a6b9bc` (10 chars) ✓, `50aaabfc16b2` (12 chars) ✓
   - Counter-example: `c0cbe70` (7 chars) ✗
   - Record SHA-1 length in TodoWrite

2. **Summary Format Check**
   - Verify subject line is enclosed in double quotes
   - Subject line should match the original commit's first line
   - Format: `Fixes: <10+char-SHA1> ("Original subject line")`
   - Example: `Fixes: 50aaabfc16b2 ("LU-19963 nodemap: add projid_set rbac role")` ✓
   - Record quote presence in TodoWrite

3. **Single Line Requirement**
   - Verify tag is NOT split across multiple lines
   - Tags are exempt from the commit-message line-wrap rule to simplify
     parsing scripts
   - If the line is very long, it should still remain on one line
   - Counter-example:
     ```
     Fixes: 50aaabfc16b2 ("LU-19963 nodemap: add projid_set
       rbac role")
     ```
     This is INCORRECT - tag must be on a single line
   - Record line wrapping status in TodoWrite

4. **Subject Line Accuracy**
   - Use `git log -1 --format=%s <commit-id>` to get original subject
   - Compare with subject in Fixes: tag
   - Common errors:
     - Truncated subject line
     - Modified or paraphrased subject
     - Missing subsystem prefix
   - Record comparison result in TodoWrite

## Tag Placement [FIXES-002]

**Risk**: Tag not recognized by automated tools

**Mandatory placement validation:**

Track tag location in TodoWrite and verify:

1. **Location in Commit Message**
   - Verify the tag appears in the trailer area (after the main commit
     description), with the other trailers
   - Typical Lustre ordering:
     ```
     <commit description>

     Fixes: <sha1> ("subject")
     Signed-off-by: <author>
     Change-Id: I<...>
     ```
   - Record tag location in TodoWrite

## Commit Verification [FIXES-003]

**Risk**: Invalid commit reference, incorrect attribution

**Mandatory commit validation:**

Track commit verification in TodoWrite:

1. **Commit Existence**
   - Run: `git cat-file -t <commit-id>`
   - Verify it returns "commit"
   - Record existence check result in TodoWrite

2. **Commit Reachability**
   - Run: `git merge-base --is-ancestor <commit-id> HEAD`
   - Verify the referenced commit is in the master branch history
   - Note: a fix may target a very recent commit not yet merged; if so, note it
     rather than failing
   - Record reachability check result in TodoWrite

3. **Verify the Bug Actually Exists**
   - Read the referenced commit using git show or git log
   - Analyze whether current patch actually fixes a bug introduced by
     that commit
   - Common errors to check for:
     - Fixes: tag points to wrong commit
     - Fixes: tag points to a commit that didn't introduce the bug
     - Multiple commits contributed to the bug, but only one is
       referenced
   - Record bug relationship analysis in TodoWrite

## Maintenance-branch backport considerations [FIXES-004]

Lustre does not use `Cc: stable@vger.kernel.org`. Backports to maintenance
branches (b2_15, b2_16, ...) are done as separate Gerrit cherry-picks, not
triggered by the `Fixes:` tag. So:

- Do not require or look for a stable/Cc tag.
- The `Fixes:` tag's value here is identifying the origin so maintainers can
  decide which maintenance branches need the cherry-pick; that decision is out
  of scope for this review.
- A note in the commit body about prerequisite commits is helpful but optional.

## Common Patterns and Edge Cases

### When Fixes: Tag Should Be Present

1. **Bug Fixes**
   - Fixing crashes, hangs, data corruption, security issues
   - Fixing incorrect behavior introduced by a specific commit
   - Even for bugs that won't be backported

2. **Regressions**
   - Any user-visible regression should have a Fixes: tag
   - Performance regressions
   - Functionality regressions

### When Fixes: Tag May Be Absent

1. **Improvements Without Specific Bug**
   - General optimizations
   - Code refactoring (without fixing a bug)
   - New features

2. **Fixes for Very Old Code**
   - Bug existed since initial git history
   - Alternative: Note in commit message "bug existed since ..."

3. **Multiple Contributing Commits**
   - If multiple commits contributed to a bug, typically reference the most direct/recent cause
   - Can include multiple Fixes: tags if necessary (rare)

## Git Configuration for Reviewers

To make Fixes: tag generation easier, configure git:

```
[core]
    abbrev = 12
[pretty]
    fixes = Fixes: %h (\"%s\")
```

Usage: `git log -1 --pretty=fixes <commit-id>` (12 chars is a fine default;
Lustre accepts 10+).

## Mandatory Self-verification gate

**After analysis:** Issues found: [none OR list]

## Quick Reference

**Correct Format:**
```
Fixes: 50aaabfc16b2 ("LU-19963 nodemap: add projid_set rbac role")
```

**Common Errors:**
- Too short: `Fixes: 50aaab (...)` (under 10 chars) ✗
- Missing quotes: `Fixes: 50aaabfc16b2 (LU-19963 nodemap: ...)` ✗
- Line wrapped: `Fixes: 50aaabfc16b2 ("LU-19963 nodemap:\n    add ...")` ✗
- Subject does not match the referenced commit's real subject ✗
