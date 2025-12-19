# Robust Non-Interactive Editing with `ed`

This guide outlines a fail-safe methodology for programmatically editing files using the `ed` line editor. By following these strict procedures, you can eliminate common errors like shifting line numbers and broken syntax.

## Quick Reference (TL;DR)

1. **Locate:** `ed -s FILE <<'EDSCRIPT####'` ... `START,ENDn` ... `EDSCRIPT####` (always append a random number)
2. **Measure indent:** `awk -v n=LINE 'NR==n {match($0, /^[ \t]*/); print length(substr($0, RSTART, RLENGTH))}' FILE`
3. **Edit bottom-up:** Start from highest line number, work down
4. **Script structure:** `H` first, `w` then `q` last
5. **Anchor edits:** Use `s/pattern/&/` to verify line content before editing
6. **Multi-line indent:** Use `I() { printf '%s%*s' "$base" $(($1*unit)) ''; }` with `$(I N)` for programmatic indentation

**The `-s` flag:** Always use `ed -s` (silent mode) to suppress the byte count printed when the file is loaded. This keeps output clean and makes error detection easier.



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

**CRITICAL: This applies whether using one script or multiple scripts.**

### Single Script (Preferred)

When all edits can be determined from the original file state, use one atomic script with commands in reverse order.

**Wrong Way (Top-Down in single script):**
```bash
ed -s file.py <<'EDSCRIPT1234'
H
# Insert at line 1 (shifts everything down)
1i
import os
.
# Edit line 50 (now at line 51 - WRONG!)
50s/old/new/
w
q
EDSCRIPT1234
```
**Result:** Line 50 edit targets wrong line. **FAILED.**

**Right Way (Bottom-Up in single script):**
```bash
ed -s file.py <<'EDSCRIPT4829'
H
# Edit line 50 FIRST (still at original position)
50s/old/new/
# Insert at line 1 SECOND (doesn't affect line 50)
1i
import os
.
w
q
EDSCRIPT4829
```
**Result:** Both edits target correct original line numbers. **SUCCESS.**

### Multiple Scripts

When using multiple scripts (each with `w`), each script sees a NEW file state. Line numbers shift after each write. See "Multi-Script Editing with Verification" section for details.

**Key principle:** Bottom-up ordering within each script, and verification between scripts to account for line number shifts.

## The Robust Workflow

### 1. Locate AND Measure (REQUIRED BEFORE AND AFTER EDITING)

IMPORTANT: DO NOT write any edit script until you complete BOTH parts of this step.

When preparing to edit, run these commands together to get line numbers, and indentation for source code or configuration files:

```bash
# View lines with numbers to find your target
ed -s FILE <<'EDSCRIPT4829'
H
START,ENDn
EDSCRIPT4829

# Get exact indentation of the line you'll edit (replace LINE with the line number)
awk -v n=LINE 'NR==n {match($0, /^[ \t]*/); print length(substr($0, RSTART, RLENGTH))}' FILE
```

**Understanding the Two Measurements:**

1. **Line indentation** (from the awk command above): The number of leading spaces on a specific line
2. **Indent width** (from the Example workflow below): The file's indent unit (typically 4 spaces)

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

### Programmatic Indentation for Insertions

When inserting multiple lines with varying indentation levels, **do not type leading whitespace literally**. Instead, generate it programmatically to ensure correctness.

**Setup: Extract base indent and compute indent function**
```bash
# Extract base indent from a sibling line (e.g., line 829)
base=$(sed -n '829s/^\([[:space:]]*\).*/\1/p' FILE)

# Detect indent unit from file (typically 4 spaces for Python)
unit=$(awk '/^[[:space:]]+[^[:space:]]/{match($0,/^[[:space:]]+/);if(p&&RLENGTH>p){print RLENGTH-p;exit}p=RLENGTH}' FILE)
unit=${unit:-4}

# Function: I N = base indent + N additional indent units
I() { printf '%s%*s' "$base" $(($1*unit)) ''; }
```

**Usage in ed heredoc:**
```bash
ed -s FILE <<EOF
830a
$(I 0)# Comment at base indent level
$(I 0)try:
$(I 1)import litellm
$(I 1)litellm.callbacks = []
$(I 0)except ImportError:
$(I 1)pass
.
w
q
EOF
```

**Key points:**
- `$(I 0)` = base indent (same level as sibling line)
- `$(I 1)` = base + 1 indent unit (one level deeper)
- `$(I 2)` = base + 2 indent units (two levels deeper)
- The heredoc uses unquoted `<<EOF` to allow `$(I N)` expansion
- No literal leading whitespace is typed - all indentation comes from the `I` function

**Why this works:**
- `$base` is extracted directly from the file - guaranteed correct
- `$unit` is detected from the file or defaults to 4
- `printf '%*s'` generates exactly the specified number of spaces
- Shell expansion happens before `ed` sees the content

### 2. Script: Construct the Atomic Edit
Create a single script using a **Quoted Heredoc** (`<<'EDSCRIPT4829'`).

**Why Quoted Heredoc?**
Using a quoted delimiter like `'EDSCRIPT4829'` (with quotes) prevents the shell from expanding variables (`$var`) or interpreting backslashes. This allows you to paste code snippets (including quotes and special characters) directly into the script without the "quoting nightmare" of `printf`.

**Scenario:**
1.  Add a method after line 200.
2.  Add an item to a list at line 83.
3.  Add an import at line 1.

**The Script:**
```bash
ed -s filename.py <<'EDSCRIPT4829'
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
EDSCRIPT4829
```
### 3. Verify: Check Your Work
After running the script, **verify the changes immediately**, including indentation.

**CRITICAL: Always verify indentation after edits.** Use the same measurement command from Step 1:
```bash
# Verify indentation of the newly inserted line (replace LINE with actual line number)
awk -v n=LINE 'NR==n {match($0, /^[ \t]*/); print length(substr($0, RSTART, RLENGTH))}' FILE

# View the edited region with line numbers
ed -s FILE <<'EDSCRIPT####'
H
START,ENDn
EDSCRIPT####
```

**Why measure after?**
- Confirms the edit applied the correct indentation
- Catches whitespace issues before they cause runtime errors
- The before/after measurements should match your expectations

**Example verification workflow:**
```bash
# Before edit: measured line 829 has 8 spaces
# After edit: verify new line 830 also has 8 spaces
awk -v n=830 'NR==n {match($0, /^[ \t]*/); print length(substr($0, RSTART, RLENGTH))}' myfile.py
# Output should be: 8
```


**Verify syntax immediately after every edit.** Before making additional changes, run the language's syntax checker:
```bash
# Python
python3 -m py_compile FILE && echo "Syntax OK"

# JavaScript/TypeScript
node --check FILE

# JSON
python3 -m json.tool FILE > /dev/null && echo "Valid JSON"
```
This catches errors early when they're easiest to fix, before additional edits create cascading problems.

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

**Warning: `q` vs `Q` when testing assertions**

When using `s/pattern/&/` to *test* whether a line matches (without intending to write), the substitution modifies the buffer even though the content is unchanged. If you then use `q` (quit), `ed` will refuse to exit and report "Warning: buffer modified" with exit code 1.

**Solutions:**
- **For verification only:** Use `Q` (uppercase) to quit unconditionally without saving:
  ```bash
  ed -s file.py <<'EDSCRIPT4829'
  H
  83s/Keys/&/
  Q
  EDSCRIPT4829
  ```
- **For actual edits:** Continue to use `q` after `w` (write) as normal - the warning only occurs when quitting a modified-but-not-written buffer.

**Key insight:** The `s/pattern/&/` command *always* marks the buffer as modified, even when replacing text with itself. This is expected behavior, not an error in your pattern.

### 3. After every successful edit, immediately verify before proceeding to additional edits.

## Multi-Script Editing with Verification

When making multiple edits across several `ed` invocations (each with its own `w`), verification between scripts is critical to maintain accuracy.

**Mandatory verification after each script:**
1. Verify the exact change with `ed -s FILE <<< 'H\nSTART,ENDn'`
2. Measure indentation of changed lines with `awk` (for code files)
3. Run syntax checker (`python3 -m py_compile`, etc.)
4. Only proceed to the next script if all checks pass

**Example of verified multi-script editing:**
```bash
# Script 1: Add import at top
ed -s file.py <<'EDSCRIPT4829'
H
1i
import os
.
w
q
EDSCRIPT4829

# VERIFY before continuing
ed -s file.py <<'EDSCRIPT1920'
H
1,3n
EDSCRIPT1920
# Confirm import present at line 1

python3 -m py_compile file.py || { echo "Syntax error after script 1"; exit 1; }

# Script 2: Add function at bottom
# Line numbers from original file are now shifted by +1
# Must account for the import insertion
ed -s file.py <<'EDSCRIPT3847'
H
$a

def new_function():
    pass
.
w
q
EDSCRIPT3847

# VERIFY again
python3 -m py_compile file.py || { echo "Syntax error after script 2"; exit 1; }
```

**Key principle:** After each `w` (write), the file state has changed. Subsequent scripts must account for line number shifts from previous edits. Verification catches errors before they accumulate.

**When single script is better:**
If all line numbers can be determined from the original file state, combine all edits into one script using bottom-up ordering. This is simpler and safer than multiple scripts.

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
Always use a single `ed` invocation with a **Quoted Heredoc** (`<<'EDSCRIPT4829'`). This ensures the file is opened and written only once, prevents race conditions, and handles special characters safely.

```bash
# Good
ed -s file <<'EDSCRIPT4829'
H
...commands...
w
q
EDSCRIPT4829
```

### Heredoc Delimiter Conflicts

If the content you're inserting contains your heredoc delimiter, the script will terminate prematurely and fail.

**Required: Always append a random number to your delimiter**

To eliminate any risk of delimiter conflicts, you **must** append a random 4-digit number to `EDSCRIPT` (e.g., `EDSCRIPT4829`):

```bash
ed -s file.py <<'EDSCRIPT4829'
H
50,60n
EDSCRIPT4829
```

This guarantees uniqueness even when editing files that contain `EDSCRIPT` (like this guide itself, or code that uses `ed`). Pick any random number for each editing session.

**Why `EDSCRIPT` as the base instead of `EOF`?**
`EOF` is extremely common in code examples and documentation. Using `EDSCRIPT` as the base makes the delimiter's purpose clear and further reduces collision risk.

**Example: Editing a file that contains heredoc examples**
```bash
ed -s file.md <<'EDSCRIPT7391'
H
50a
ed -s example.py <<'EDSCRIPT'
H
10,20n
EDSCRIPT

.
w
q
EDSCRIPT7391
```

### Quoted vs Unquoted Heredocs: When to Use Each

**Quoted heredoc (`<<'EDSCRIPT4829'`):**
- **Use for:** Inserting literal code with special characters
- **Behavior:** Shell does NOT expand variables (`$var`, `$(cmd)`) or interpret backslashes
- **Best for:** Python/JS/JSON with quotes, dollar signs, backslashes

```bash
# Inserting Python code with string literals and variables
ed -s file.py <<'EDSCRIPT4829'
H
10a
def example():
    msg = "Hello $USER"  # Literal $USER, not expanded
    return msg
.
w
q
EDSCRIPT4829
```

**Unquoted heredoc (`<<EDSCRIPT4829`):**
- **Use for:** Programmatic indentation with the `I()` function
- **Behavior:** Shell DOES expand variables (`$base`, `$(I 2)`) before `ed` sees them
- **Required for:** Multi-level indentation patterns

```bash
# Programmatic indentation (requires unquoted heredoc)
base=$(sed -n '50s/^\([[:space:]]*\).*/\1/p' file.py)
unit=4
I() { printf '%s%*s' "$base" $(($1*unit)) ''; }

ed -s file.py <<EDSCRIPT4829
H
50a
$(I 0)def method(self):
$(I 1)return True
.
w
q
EDSCRIPT4829
```

**Common mistake:** Using `$(I 0)` inside a quoted heredoc
```bash
# WRONG: $(I 0) appears literally in file
ed -s file.py <<'EDSCRIPT4829'
H
10a
$(I 0)def example():  # This is LITERAL text: "$(I 0)def example():"
.
w
q
EDSCRIPT4829
```

**Decision flowchart:**
1. Does your insertion need shell variable expansion (like `$(I 0)`)? → Use UNQUOTED `<<EDSCRIPT`
2. Otherwise → Use QUOTED `<<'EDSCRIPT'` (safer default)

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
| `Q`           | Quit unconditionally (no warning if buffer modified). |
| `p`           | Print the current line (without line numbers).        |
| `$=`          | Print the total number of lines in the buffer.        |

### Command Limitations

**`s/old/new/`** modifies text *within* a line. It cannot:
- Move lines to different positions
- Swap line order
- Delete entire lines
- Insert new lines

For structural changes (reordering, moving, deleting), use `d`, `m`, `a`, or `i`.

**`c` (change) completely replaces lines.** When using `START,ENDc`, the entire range is deleted and replaced with your new content. Before using `c` on a range:
1. Print the exact lines first (`START,ENDn`)
2. Identify any syntax-significant characters (`)`, `]`, `}`, `:`) that must be preserved
3. Include those characters in your replacement - nothing is kept unless you explicitly include it

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
ed -s file.md <<'EDSCRIPT4829'
H
# Move line 11 to after line 9 (effectively swaps lines 10 and 11)
11m9
w
q
EDSCRIPT4829
```

**Solution 2: Delete and reinsert in new order**
```bash
ed -s file.md <<'EDSCRIPT4829'
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
EDSCRIPT4829
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
ed -s file.py <<'EDSCRIPT4829'
H
# Delete lines 120, 85, and 12 (bottom-up to preserve line numbers)
120d
85d
12d
w
q
EDSCRIPT4829
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
ed -s file.py <<'EDSCRIPT4829'
H
225,236n
EDSCRIPT4829

# RIGHT: Print from line 225 to end of file (whatever that is)
ed -s file.py <<'EDSCRIPT4829'
H
225,$n
EDSCRIPT4829

# RIGHT: Print last 15 lines with line numbers
ed -s file.py <<'EDSCRIPT4829'
H
$-14,$n
EDSCRIPT4829
```

### Don't Combine Discovery with Assumptions
Never issue multiple commands in one script where a later command depends on assumptions about output you haven't seen yet.

```bash
# WRONG: Assumes 236 exists based on nothing
ed -s file.py <<'EDSCRIPT4829'
H
$=
225,236n
EDSCRIPT4829

# RIGHT: Get length first, then query in a second invocation
ed -s file.py <<'EDSCRIPT4829'
H
$=
EDSCRIPT4829
# Now you know it's 235 lines, so query accordingly:
ed -s file.py <<'EDSCRIPT4829'
H
225,$n
EDSCRIPT4829
```

### Verify Edits with `ed` (Not `read_file`)

After editing a file with `ed`, always verify using `ed` itself or terminal commands. **Do not rely on IDE file-reading tools** (like VS Code's `read_file`), as they may return cached/stale content that doesn't reflect recent external changes.

**Use `ed` with the `n` command for targeted verification:**
```bash
# Print lines 50-60 with line numbers
ed -s file.py <<'EDSCRIPT4829'
H
50,60n
EDSCRIPT4829

# Print from line 225 to end of file
ed -s file.py <<'EDSCRIPT4829'
H
225,$n
EDSCRIPT4829

# Print last 10 lines with line numbers
ed -s file.py <<'EDSCRIPT4829'
H
$-9,$n
EDSCRIPT4829
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

## Troubleshooting

Common errors and their causes:

| Error | Cause | Solution |
|-------|-------|----------|
| `?` (no details) | Forgot the `H` command | Always start scripts with `H` to enable verbose errors |
| `No match` | Pattern not found on target line | Check line number is correct; check regex escaping for `*`, `[`, `.` |
| `Invalid address` | Line number out of range | Use `$=` to check file length first, or use `$` for end-of-file |
| `Warning: buffer modified` | Used `q` after modifying without `w` | Use `Q` to quit unconditionally, or add `w` before `q` |
| `Invalid command suffix` | Wrong syntax in command | Check command format (e.g., `s/old/new/` needs both delimiters) |
| File unchanged after script | Script failed before `w` command | Check exit code; review verbose error output |

### Special Characters That Need Escaping in Patterns

When using `s/pattern/&/` for assertions or `s/old/new/` for substitutions, these characters have special meaning in `ed` regex and must be escaped with `\`:

| Character | Meaning | Example |
|-----------|---------|---------|
| `.` | Matches any single character | `\.` to match literal period |
| `*` | Zero or more of preceding | `\*` to match literal asterisk |
| `[` | Start character class | `\[` to match literal bracket |
| `]` | End character class | `\]` to match literal bracket |
| `^` | Start of line (or negation in `[]`) | `\^` to match literal caret |
| `$` | End of line | `\$` to match literal dollar sign |
| `\` | Escape character | `\\` to match literal backslash |

### Recovering from Failed Edits

When an edit produces invalid syntax or incorrect results:

1. **First failed edit:** Try ONE targeted correction using `ed` to fix the specific problem.

2. **Second failed edit:** Consider whether continuing is productive. If you've lost track of file state or can't explain why edits are failing, reverting may be faster:
   ```bash
   git checkout FILE
   ```
   Then analyze what went wrong and identify the line numbers to edit before trying again.

**After revert, before retry:**
1. Re-read the file to understand current state
2. Identify exact line numbers for all needed edits
3. Determine whether edits can be combined in a single atomic script
4. Use single script with bottom-up ordering when possible
5. Verify immediately after execution

**Warning signs you may need to revert:**
- Lost track of which content is where
- Can't explain why an edit failed
- Repeating same verification commands without progress

**Key insight:** Patching a broken edit with more edits can work, but if you find yourself making multiple correction attempts without success, a clean revert + redesign is faster and safer than continued patching.

