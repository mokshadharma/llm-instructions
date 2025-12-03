# Robust Non-Interactive Editing with `ed`

This guide outlines a fail-safe methodology for programmatically editing files using the `ed` line editor. By following these strict procedures, you can eliminate common errors like shifting line numbers and broken syntax.


## Always Use `ed` for File Editing

When editing files programmatically, **always use `ed`**. Do not use other tools like `sed`, `replace_string_in_file`, or similar alternatives.

**Why?**
- The workflow requires discovering exact line numbers before editing, which forces you to verify the current file state
- Edits target specific lines, not pattern matches that could occur in unexpected locations
- Strict contextual anchoring catches errors before they corrupt files
- `ed` provides atomic, all-or-nothing edits
- One tool, one methodology, fewer mistakes

## The Golden Rule: Bottom-Up Editing

When performing multiple edits on a single file, **always apply changes in reverse line-number order (descending)**.

*   **Why?** Inserting or deleting lines changes the line numbers for everything *below* the edit, but never for anything *above* it.
*   **The Fix:** By starting from the bottom of the file and working up, every line number you determined from the original file remains valid at the moment of execution.

### Example

**Wrong Way (Top-Down):**
1.  Insert import at line 1 (shifts all subsequent lines down by 1).
2.  Edit line 50 (which is now line 51). **Result:** You edit the wrong line.

**Right Way (Bottom-Up):**
1.  Edit line 50 (it is still exactly at line 50).
2.  Insert import at line 1. **Result:** Success.

## The Robust Workflow

### 1. Locate: Definitively Determine Line Numbers
*Immediately* before generating your edit script, use non-interactive tools to find the exact line numbers. Do not guess.

**Using `rg` (ripgrep) with line numbers:**
```bash
rg -n "class MyClass" filename.py
```

**Using `ed` to print specific ranges:**
```bash
# Print lines 10-20 with line numbers
echo '10,20n' | ed -s filename.py
```


### 2. Measure: Calculate Exact Indentation
Guessing indentation leads to `IndentationError` in Python and broken formatting in other languages. **Always** calculate the exact indentation of the target line before generating the script.

**The Standardized Indentation Check:**
Use this `awk` command to get the integer count of leading spaces/tabs for a specific line number (`n`).

```bash
awk -v n=140 'NR==n {match($0, /^[ \t]*/); print length(substr($0, RSTART, RLENGTH))}' filename.py
```

**How to use the result:**
1. **Sibling Line:** If inserting a line at the same level, use the returned number (e.g., `12` spaces).
2. **Child Block:** If inserting into a new block (e.g., after a line ending in `:`), calculate the file's indent width (see below) and add it to the result (e.g., `12 + 4 = 16` spaces).

**Detecting Indent Width:**
To robustly determine the file's indentation width (spaces or tabs) without guessing, run this command:

```bash
awk '!NF { next } match($0, /^[ \t]*/){ curr = RLENGTH; if (prev_defined && curr > prev) { print curr - prev; exit } prev = curr; prev_defined = 1 }' filename.py
```
### 3. Script: Construct the Atomic Edit
Create a single script using a **Quoted Heredoc** (`<<'EDSCRIPT'`).

**Why Quoted Heredoc?**
Using `'EDSCRIPT'` (with quotes) prevents the shell from expanding variables (`$var`) or interpreting backslashes. This allows you to paste code snippets (including quotes and special characters) directly into the script without the "quoting nightmare" of `printf`.

**Scenario:**
1.  Add a method after line 200.
2.  Add an item to a list at line 83.
3.  Add an import at line 1.

**The Script:**
```bash
ed -s filename.py <<'EDSCRIPT'
H
200a
    def new_method(self):
        pass
.
83s/$/,/
83a
    NewItem
.
1i
import time
.
w
q
EDSCRIPT
```
### 4. Verify: Check Your Work
After running the script, verify the changes immediately using the same tools used in step 1.,

```bash
# Check the import
head -n 5 filename.py

# Check the list item
echo '80,90n' | ed -s filename.py
```

## Critical Safeguards (Required)

### 1. Verbose Error Messages (`H`)
By default, `ed` only prints `?` on error. You **must** enable verbose error messages to debug failures effectively.
*   **The Fix:** Always start your script with the `H` command.
*   **Why:** If a strict assertion fails, `ed` will print "No match" or "Invalid address" instead of just `?`.

### 2. The "Write-Once" Atomicity Rule
The `w` (write) command **must appear exactly once, at the very end of the script**.
*   **Why:** `ed` operates on an in-memory buffer. If your script fails halfway through (e.g., a strict assertion fails), and you haven't issued a `w` command, the file on disk remains **completely untouched**.
*   **Benefit:** This provides "All-or-Nothing" transactional safety. If the script crashes, the file is safe.

## Advanced: Strict Contextual Anchoring (Required)

Even with bottom-up editing, you might target the wrong line if the file changed unexpectedly. To prevent silent failures or corruption, you **must** use Strict Contextual Anchoring for every edit.

**The Technique:**
Use the substitute command `s/pattern/&/` to assert that the target line matches a specific pattern before executing the edit.
*   **Why:** If the pattern is not found, `ed` returns an error (`?`) and aborts the script immediately. This prevents the script from editing the wrong line or continuing in an invalid state.
* **Note:** You must escape special regex characters (like `*`, `[`, `.`) in the pattern. Failure to escape `[` or `.` will cause `ed` to interpret them as regex classes or wildcards, leading to "No match" errors or incorrect edits.

**Example:**
Instead of blindly appending to line 83:
```bash
83s/$/,/
83a
    NewItem
.
```
* **Note:** You must escape special regex characters (like `*`, `[`, `.`) in the pattern. Failure to escape `[` or `.` will cause `ed` to interpret them as regex classes or wildcards, leading to "No match" errors or incorrect edits.
**Strict Assertion:**
Assert that line 83 actually contains "Keys" before editing:
```bash
# 1. Assert: Try to replace 'Keys' with itself (&). Fails if missing.
83s/Keys/&/
# 2. Edit: Now safe to proceed
83s/$/,/
83a
    NewItem
.
```

This ensures you don't accidentally append to the wrong list if line numbers have shifted. If line 83 does not contain "Keys", the script fails safely.

## Robust Editing Patterns

### Comma Safety
When appending items to lists or dictionaries in code, the previous line might be missing a trailing comma.

**The Pattern:**
Combine a substitution (to ensure the comma exists) with the append command.

```bash
# Ensure line 83 ends with a comma, then append
83s/$/,/
83a
    NewItem
.
```
This works perfectly in reverse order because the `a` command inserts *after* the target line, so line 83 remains stable for the substitution.

### Atomic Scripts
Always use a single `ed` invocation with a **Quoted Heredoc** (`<<'EDSCRIPT'`). This ensures the file is opened and written only once, prevents race conditions, and handles special characters safely.

```bash
# Good
ed -s file <<'EDSCRIPT'
H
...commands...
w
q
EDSCRIPT
```

### Heredoc Delimiter Conflicts

If the content you're inserting contains your heredoc delimiter, the script will terminate prematurely and fail.

**Why `EDSCRIPT` instead of `EOF`?**
`EOF` is extremely common in code examples and documentation. Using `EDSCRIPT` as the standard delimiter avoids conflicts when inserting content that contains heredoc examples.

## Essential Commands Reference

| Command       | Description                                             |
|---------------|---------------------------------------------------------|
| `n`           | Print lines with line numbers (essential for locating). |
| `a`           | Append text *after* the current line.                   |
| `i`           | Insert text *before* the current line.                  |
| `c`           | Change (replace) the current line(s).                   |
| `d`           | Delete the current line(s).                             |
| `s/old/new/`  | Substitute text on the current line.                    |
| `m`           | Move line(s) to after another line.                     |
| `w`           | Write changes to disk.                                  |
| `q`           | Quit.                                                   |

### Command Limitations

**`s/old/new/`** modifies text *within* a line. It cannot:
- Move lines to different positions
- Swap line order
- Delete entire lines
- Insert new lines

For structural changes (reordering, moving, deleting), use `d`, `m`, `a`, or `i`.

## Swapping or Reordering Lines

Swapping lines requires **delete + insert** or **move**, not substitution. The `s/` command changes text within a line; it cannot move lines.

**Common Mistake:** Using substitution to "swap" identifiers
```bash
# WRONG: This does NOT swap lines - it just renames content in place
10s/TASK-029/TASK-030/
11s/TASK-030/TASK-029/
# Result: Line 10 still comes before line 11, just with swapped labels
```

**Solution 1: Use the `m` (move) command**
```bash
ed -s file.md <<'SCRIPT'
H
# Move line 11 to after line 9 (effectively swaps lines 10 and 11)
11m9
w
q
SCRIPT
```

**Solution 2: Delete and reinsert in new order**
```bash
ed -s file.md <<'SCRIPT'
H
# First, capture the content you need (or know it beforehand)
# Delete both lines (bottom-up to preserve line numbers)
11d
10d
# Insert in new order after line 9
9a
| TASK-029 | (content from original line 11) |
| TASK-030 | (content from original line 10) |
.
w
q
SCRIPT
```

**Key insight:** To reorder lines, you must physically move them using `m` (move), or `d` (delete) combined with `a`/`i` (append/insert). Substitution only changes text within existing lines.
