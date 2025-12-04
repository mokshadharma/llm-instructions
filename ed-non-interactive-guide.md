# Robust Non-Interactive Editing with `ed`

This guide outlines a fail-safe methodology for programmatically editing files using the `ed` line editor. By following these strict procedures, you can eliminate common errors like shifting line numbers and broken syntax.


## Always Use `ed` for File Editing

When editing files programmatically, **always use `ed`**. Do not use other tools like `sed`, `replace_string_in_file`, `multi_replace_string_in_file`, or similar alternatives.  The requirement to use `ed` for editing overrides all other editing-related instructions.

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

### 1. Locate AND Measure (REQUIRED BEFORE EDITING)

IMPORTANT: DO NOT write any edit script until you complete BOTH parts of this step.

When preparing to edit, run these commands together to get line numbers, and indentation for source code or configuration files:

```bash
# View lines with numbers to find your target
ed -s FILE <<'EDSCRIPT'
H
START,ENDn
EDSCRIPT

# Get exact indentation of the line you'll edit (replace LINE with the line number)
awk -v n=LINE 'NR==n {match($0, /^[ \t]*/); print length(substr($0, RSTART, RLENGTH))}' FILE
```

**Understanding the Two Measurements:**

1. **Line indentation** (from the first awk command): The number of leading spaces on a specific line
2. **Indent width** (from the second awk command): The file's indent unit (typically 4 spaces)

**How to use them together:**
- **Sibling lines** (same nesting level): Use the exact indentation of an existing sibling
- **Child lines** (one level deeper): Add the indent width to the parent's indentation
- **Parent lines** (one level shallower): Subtract the indent width

**Example workflow:**
```bash
# 1. Find the indent width for the file (looks for first def/class and measures body indent)
awk '/^(def |class )/ { in_func=1; next } in_func && /^[[:space:]]+[^[:space:]]/ { match($0, /^[[:space:]]+/); print RLENGTH; exit }' myfile.py
# Output: 4

# 2. Find the indentation of line 47 (an existing attribute you want to add a sibling to)
awk -v n=47 'NR==n {match($0, /^[ \t]*/); print length(substr($0, RSTART, RLENGTH))}' myfile.py
# Output: 8

# Conclusion: New sibling attributes need 8 spaces
# A child block inside that line would need 8 + 4 = 12 spaces
```

**Record the indentation number before proceeding.** Use exactly that many spaces for sibling lines, or add the file's indent width for child blocks.

**Common Failure Mode:**
```bash
# You think: "it's inside a try block in a function, so probably 12 spaces"
# Reality: it's 8 spaces
# Result: IndentationError, multiple retry attempts, wasted time
```

Guessing indentation WILL fail. Measuring takes 5 seconds. Fixing failed edits takes minutes.

### 2. Script: Construct the Atomic Edit
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
### 3. Verify: Check Your Work
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

**When your content contains `EDSCRIPT`:**
If the text you're inserting contains the literal string `EDSCRIPT` (e.g., editing this guide, or inserting code that uses `ed`), use a different delimiter like `OUTER_ED` or `ED_SCRIPT_END`:

```bash
ed -s file.md <<'OUTER_ED'
H
50a
ed -s example.py <<'EDSCRIPT'
H
10,20n
EDSCRIPT

.
w
q
OUTER_ED
```

## Essential Commands Reference

| Command       | Description                                             |
|---------------|---------------------------------------------------------|
| `n`           | Print lines with line numbers (essential for locating). |
| `a`           | Append text *after* the current line.                   |
| `i`           | Insert text *before* the current line.                  |
| `c`           | Change (replace) the current line(s).                   |
| `d`           | Delete the current line.                                |
| `START,ENDd`  | Delete lines START through END (inclusive).             |
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


## Deleting Contiguous Lines

To delete a block of consecutive lines, use a **range delete** - not individual line deletions.

**Wrong Way (inefficient):**
```bash
# Deleting lines 50-55 one at a time (wasteful, error-prone)
55d
54d
53d
52d
51d
50d
```

**Right Way:**
```bash
# Delete lines 50 through 55 in one command
50,55d
```

**When to use bottom-up individual deletes vs range delete:**
- **Range delete (`50,55d`)**: When deleting a *contiguous block* of lines
- **Bottom-up individual deletes**: When deleting *non-contiguous* lines at different locations (e.g., lines 120, 85, and 12)

**Example: Deleting non-contiguous lines**
```bash
ed -s file.py <<'EDSCRIPT'
H
# Delete lines 120, 85, and 12 (bottom-up to preserve line numbers)
120d
85d
12d
w
q
EDSCRIPT
```

## Safe Range Patterns for Querying

When exploring a file's content, use patterns that can't fail due to out-of-bounds addresses.

### Getting File Length First (Two-Step)
If you need precise control, get the line count first in a separate command:
```bash
# Step 1: Get line count
wc -l < filename.py   # Returns just the number

# Step 2: Use that info to construct valid ranges
```

### Using `$` for "End of File"
The `$` address always refers to the last line, regardless of file length. Use it instead of hardcoded line numbers when you don't know the file size:

```bash
# WRONG: Assumes file has at least 236 lines
ed -s file.py <<'EDSCRIPT'
H
225,236n
EDSCRIPT

# RIGHT: Print from line 225 to end of file (whatever that is)
ed -s file.py <<'EDSCRIPT'
H
225,$n
EDSCRIPT

# RIGHT: Print last 15 lines with line numbers
ed -s file.py <<'EDSCRIPT'
H
$-14,$n
EDSCRIPT
```

### Don't Combine Discovery with Assumptions
Never issue multiple commands in one script where a later command depends on assumptions about output you haven't seen yet.

```bash
# WRONG: Assumes 236 exists based on nothing
ed -s file.py <<'EDSCRIPT'
H
$=
225,236n
EDSCRIPT

# RIGHT: Get length first, then query in a second invocation
ed -s file.py <<'EDSCRIPT'
H
$=
EDSCRIPT
# Now you know it's 235 lines, so query accordingly:
ed -s file.py <<'EDSCRIPT'
H
225,$n
EDSCRIPT
```

### Verify Edits with `ed` (Not `read_file`)

After editing a file with `ed`, always verify using `ed` itself or terminal commands. **Do not rely on IDE file-reading tools** (like VS Code's `read_file`), as they may return cached/stale content that doesn't reflect recent external changes.

**Use `ed` with the `n` command for targeted verification:**
```bash
# Print lines 50-60 with line numbers
ed -s file.py <<'EDSCRIPT'
H
50,60n
EDSCRIPT

# Print from line 225 to end of file
ed -s file.py <<'EDSCRIPT'
H
225,$n
EDSCRIPT

# Print last 10 lines with line numbers
ed -s file.py <<'EDSCRIPT'
H
$-9,$n
EDSCRIPT
```

**Why `ed` over shell tools?**
- `cat` - dumps entire file, no line numbers, can be huge
- `head`/`tail` - no line numbers, requires mental math to determine positions
- `sed -n 'Np'` - no line numbers
- `ed` with `n` - gives exact lines with their line numbers, confirming both content and position

**Get line count:**
```bash
wc -l < filename.py
```
