---
description: "Stage specific parts of a file's uncommitted changes, leaving the rest unstaged. Useful when git add -p can't split hunks finely enough."
---

# Partial Stage

Stage only specific changes from a file into the git index, leaving other changes in the working tree for later commits.

## When to use

- `git add -p` can't split a hunk finely enough (adjacent additions that belong to different commits)
- A file has interleaved changes for multiple logical commits
- You need surgical control over what's staged vs what stays in the working tree

## Technique

### Step 1: Stage the full file

```bash
git add path/to/file.ext
```

### Step 2: Extract the staged content to a temp file

```bash
git show :path/to/file.ext > /tmp/staged.ext
```

### Step 3: Remove the parts that belong to later commits

Use `perl -0777 -i -pe` for multiline regex removal:

```bash
perl -0777 -i -pe 's/\n\nfunction recalculateLegacyIndex\(\) \{.*?\n\}//s' /tmp/staged.ext
```

Key flags:
- `-0777` slurp entire file
- `-i` edit in place
- `-pe` execute pattern
- `s` flag on the regex for `.` to match newlines

For simpler single-line removals:
```bash
perl -i -pe 's/  debugMode: true,\n//' /tmp/staged.ext
```

### Step 4: Verify the edit

```bash
grep -n 'recalculateLegacyIndex' /tmp/staged.ext  # should return nothing
grep -n 'processIncomingData' /tmp/staged.ext      # should return matches
```

### Step 5: Write the filtered content back to the git index

```bash
git hash-object -w /tmp/staged.ext | xargs -I{} git update-index --cacheinfo 100644,{},path/to/file.ext
```

This writes the modified content as a git blob (`hash-object -w`), then updates the index entry for that file path to point to the new blob (`update-index --cacheinfo`).

### Step 6: Verify the staged diff

```bash
git diff --cached -- path/to/file.ext
```

## Arguments

$ARGUMENTS should describe which changes to stage and which to defer. For example:

"Stage the `createUser` and `updateUser` functions from `user-service` but leave the `deleteUser` function for a later commit"

## Notes

- The working tree file is untouched — only the index (staging area) is modified
- `100644` is the standard file mode for non-executable files; use `100755` for executable files
- After committing, the remaining changes are still in the working tree ready for the next commit
- If you make a mistake, `git reset HEAD -- path/to/file.ext` unstages and you can start over
- For new (untracked) files, `git add` the whole file first, then apply this technique to trim the staged version
