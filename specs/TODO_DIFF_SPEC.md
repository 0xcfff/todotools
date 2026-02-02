# Todo Diff Format Specification

## Overview

Todo Diff Format is a semantic diff format for todo.txt files that treats each line as a structured task rather than plain text. Unlike traditional line-based diff tools, it can match and compare tasks based on their semantic identity, even when tasks are reordered or modified.

This format is based on the [GNU Unified Diff Format](https://www.gnu.org/software/diffutils/manual/diffutils.html) but adapted for task-oriented comparisons.

## Key Concepts

### Semantic Task Matching

Tasks are matched between files using a prioritized strategy:

1. **Custom ID matching**: Tasks with the same `taskid:value` attribute are considered identical
2. **Exact text matching**: Tasks with identical descriptions (ignoring metadata) are matched
3. **Fuzzy matching** (optional): Similar task descriptions can be matched based on similarity threshold

This allows the diff to recognize that tasks are the same even when:
- They appear in different positions in the file
- Their metadata (priority, status, dates) has changed
- Their descriptions have minor variations (fuzzy mode)

### No Context Lines

Unlike standard unified diff which includes surrounding context lines (prefix: ` `), Todo Diff Format omits context entirely. Since tasks can be matched semantically without positional context, including context lines only adds noise.

### Surgical Delta Application

By emitting both `-` and `+` lines for matched tasks, the format enables:
- Computing the exact delta between task variants
- Applying changes surgically to a third version of the same task
- Three-way merge capability where the diff can be applied to files with different local modifications

Example: If the diff shows only a status change, that status change can be applied to a variant of the task that has different tags/contexts than either compared version.

## Format Specification

The format follows GNU Unified Diff structure with adaptations:

### File Headers

```
--- from-file   from-file-modification-time
+++ to-file     to-file-modification-time
```

- `---` prefix marks the original/source file
- `+++` prefix marks the modified/target file
- Tab-separated filename and ISO timestamp with timezone

### Hunk Headers

```
@@ -l,s +l,s @@
```

- `@@` markers surround line range information
- `-l,s` indicates line number and span in the original file
- `+l,s` indicates line number and span in the modified file
- When span is 0 (no lines), format is `-l,0` or `+0,0`
- When span is 1 (single line), the `,s` can be omitted: `@@ -l +l @@`

**Special cases:**
- Task deleted: `@@ -l +0,0 @@`
- Task added: `@@ -0,0 +l @@`
- Task modified: `@@ -l +l @@`

### Line Prefixes

Within hunks, only two prefixes are used:

- `-` marks a line from the original file
- `+` marks a line from the modified file

**Interpretation:**
- Adjacent `-` and `+` lines = matched task with modifications (compute delta)
- Standalone `-` line = task deleted (no match found)
- Standalone `+` line = task added (no match found)

## Examples

### Example 1: Status and Priority Changes

```diff
--- todo-old.txt	2025-12-10 10:00:00.000000000 +0000
+++ todo-new.txt	2025-12-12 14:00:00.000000000 +0000
@@ -3 +3 @@
-(A) Call dentist +health
+x (A) Call dentist +health
@@ -7 +7 @@
-(B) Write quarterly report +work @office
+(A) Write quarterly report +work @office
```

**Interpretation:**
- Line 3: Task matched by description, status changed from incomplete to complete
- Line 7: Task matched by description, priority changed from (B) to (A)

### Example 2: Task ID Matching with Major Changes

```diff
--- todo-old.txt	2025-12-10 10:00:00.000000000 +0000
+++ todo-new.txt	2025-12-12 14:00:00.000000000 +0000
@@ -9 +9 @@
-(B) Review pull request taskid:a3f2b9c1
+(C) Refactor authentication module +coding @refactor taskid:a3f2b9c1
```

**Interpretation:**
- Tasks matched by `taskid:a3f2b9c1` despite completely different descriptions
- Priority changed: (B) → (C)
- Description evolved: "Review pull request" → "Refactor authentication module"
- Tags and contexts added

This demonstrates stable identity tracking across significant task evolution.

### Example 3: Additions and Deletions

```diff
--- todo-old.txt	2025-12-10 10:00:00.000000000 +0000
+++ todo-new.txt	2025-12-12 14:00:00.000000000 +0000
@@ -12 +0,0 @@
-Fix bug in login form +coding
@@ -0,0 +15 @@
+(C) Schedule car maintenance +errands
```

**Interpretation:**
- Line 12: Task deleted from original file (no match found in new file)
- Line 15: New task added (no match found in original file)

### Example 4: Complete Diff

```diff
--- todo-old.txt	2025-12-10 10:00:00.000000000 +0000
+++ todo-new.txt	2025-12-12 14:00:00.000000000 +0000
@@ -3 +3 @@
-(A) Call dentist +health
+x (A) Call dentist +health
@@ -7 +7 @@
-(B) Write quarterly report +work @office
+(A) Write quarterly report +work @office
@@ -9 +9 @@
-(B) Review pull request taskid:a3f2b9c1
+(C) Refactor authentication module +coding @refactor taskid:a3f2b9c1
@@ -12 +0,0 @@
-Fix bug in login form +coding
@@ -0,0 +15 @@
+(C) Schedule car maintenance +errands
```

## Implementation Considerations

### Matching Algorithm

Recommended matching order:

1. **Phase 1**: Match tasks with identical `taskid:*` attributes
2. **Phase 2**: Match tasks with exact description text (ignoring priority, status, dates)
3. **Phase 3** (optional): Fuzzy match remaining tasks using similarity threshold

### Delta Computation

For matched task pairs (adjacent `-`/`+`), compute deltas for:
- Completion status (`x` prefix)
- Priority (`(A)`, `(B)`, etc.)
- Creation date
- Completion date
- Projects (`+project`)
- Contexts (`@context`)
- Custom key:value pairs
- Description text

### Three-Way Merge Application

When applying a diff to a third file:

1. Parse diff to extract task pairs and computed deltas
2. For each delta, find matching task in target file (using same matching strategy)
3. Apply only the changed attributes from the delta
4. Preserve local modifications to attributes not in the delta

This enables collaborative editing where different users modify different aspects of the same task.

### Line Number Usage

Line numbers in `@@` headers serve to:
- Allow manual navigation to task locations in files
- Provide approximate positioning hints
- Maintain compatibility with standard diff tools

Note: Line numbers may be "fuzzy" when tasks are matched semantically rather than positionally.

## Compatibility

The format is designed to be:
- **Parseable** by standard unified diff tools (though they won't understand semantic matching)
- **Applicable** using standard patch tools (with limitations - they use exact line matching)
- **Human-readable** with clear indication of task changes
- **Machine-friendly** for custom tooling that understands task semantics

## Benefits

- **Reorder-resistant**: Tasks can be rearranged without generating spurious diffs
- **Semantic clarity**: Shows what actually changed about tasks, not just text edits
- **Merge-friendly**: Enables intelligent three-way merging of task modifications
- **Collaborative**: Multiple people can modify different aspects of tasks independently
- **Version control ready**: Clean diffs for todo.txt files in git/svn/etc.
