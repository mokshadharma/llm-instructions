# Robust Non-Interactive Editing with `ed`

This guide outlines a fail-safe methodology for programmatically editing files using the `ed` line editor. By following these strict procedures, you can eliminate common errors like shifting line numbers and broken syntax.

## Quick Reference (TL;DR)

1. **Locate:** `ed -s FILE <<'EDSCRIPT####'` ... `START,ENDn` ... `EDSCRIPT####` (always append a random number)
2. **Measure indent:** `bin/measure-indent.py LINE-SPEC FILE` (LINE-SPEC examples: `47`, `10,20`, `15-18`, `5,10-12,20`)
3. **Edit bottom-up:** Start from highest line number, work down
4. **Script structure:** `H` first, `w` then `q` last
5. **Anchor edits:** Use `s/pattern/&/` to verify line content before editing
6. **Multi-line indent:** Use `I() { printf '%s%*s' "$base" $(($1*unit)) ''; }` with `$(I N)` for programmatic indentation

**The `-s` flag:** Always use `ed -s` (silent mode) to suppress the byte count printed when the file is loaded. This keeps output clean and makes error detection easier.

> **WARNING: NEVER type literal leading whitespace in heredocs.** All indentation must come from the `$(I N)` function. See "Programmatic Indentation for Insertions" below.



## Always Use `ed` for File Editing

When editing files programmatically, **always use `ed`**. Do not use other tools like `sed`, `replace_string_in_file`, `multi_replace_string_in_file`, or similar alternatives.  The requirement to use `ed` for editing overrides all other editing-related instructions.

**Why?**
- The workflow requires discovering exact line numbers before editing, which forces you to verify the current file state
- Edits target specific lines, not pattern matches that could occur in unexpected locations
- Assertions can detect targeting errors via non-zero exit codes (though the edit still executes)
- `ed` provides atomic, all-or-nothing edits
- One tool, one methodology, fewer mistakes

## The Golden Rule: Bottom-Up Editing

When performing multiple edits on a single file, **always apply changes in reverse line-number order (descending)**.

*   **Why?** Inserting or deleting lines changes the line numbers for everything *below* the edit, but never for anything *above* it.
*   **The Fix:** By starting from the bottom of the file and working up, every line number you determined from the original file remains valid at the moment of execution.

**CRITICAL: Always use one operation per ed invocation. Execute operations in bottom-up order (highest line numbers first).**

### Why Bottom-Up Order Matters

When you have multiple operations to perform, execute them as separate ed invocations, starting with the highest line number:

**Wrong Way (Top-Down order):**
```bash
# Operation 1: Insert at line 1 (shifts everything down)
ed -s file.py <<'EDSCRIPT1234'
H
1i
import os
.
w
q
EDSCRIPT1234

# Operation 2: Edit line 50 - BUT line 50 is now line 51!
ed -s file.py <<'EDSCRIPT4829'
H
50s/old/new/
w
q
EDSCRIPT4829
```
**Result:** Line 50 edit targets wrong line. **FAILED.**

**Right Way (Bottom-Up order):**
```bash
# Operation 1: Edit line 50 FIRST (still at original position)
ed -s file.py <<'EDSCRIPT4829'
H
50s/old/new/
w
q
EDSCRIPT4829

# Verify, then Operation 2: Insert at line 1 (doesn't affect already-edited line 50)
ed -s file.py <<'EDSCRIPT4829'
H
1i
import os
.
w
q
EDSCRIPT4829
```
**Result:** Both operations target correct line numbers. **SUCCESS.**

### Why One Operation Per Invocation?

Ed does **not** abort on error. If an assertion or edit fails, ed prints an error but continues executing all subsequent commands, including `w` (write). This means:
- A failed assertion in a multi-operation script still saves the file
- You can't stop after detecting a problem
- Partial edits get written to disk

With one operation per invocation, you can verify and stop between operations if something goes wrong.

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

# REQUIRED: Measure indent with this tool - never estimate visually
bin/measure-indent.py LINE-SPEC FILE
```

**ALWAYS use `bin/measure-indent.py` for measuring indentation.** Never estimate indent visually by counting spaces in terminal output or editor displays - visual counting is unreliable and leads to failed edits.

**LINE-SPEC syntax:**
- Single line: `47`
- Multiple lines: `10,20,35`
- Range: `15-18` (lines 15, 16, 17, 18)
- Combination: `5,10-12,20` (lines 5, 10, 11, 12, 20)

**Understanding the Two Measurements:**

1. **Line indentation** (from `bin/measure-indent.py` above): The number of leading spaces on a specific line
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
bin/measure-indent.py 47 myfile.py
# Output: Line 47: 8

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
ed -s FILE <<EDSCRIPT4829
H
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
EDSCRIPT4829
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

### 2. Script: Construct the Edit (One Operation Per Invocation)

**CRITICAL: For any insertion with indentation, you MUST use programmatic indentation.** See the "Programmatic Indentation for Insertions" section above. Never type literal leading whitespace in heredocs.

- **Unquoted heredoc** (`<<EDSCRIPT4829`): Required when using `$(I N)` for indentation
- **Quoted heredoc** (`<<'EDSCRIPT4829'`): Only for content with no leading whitespace (e.g., imports at column 0)

**Scenario:**
1.  Add a method after line 200.
2.  Add an item to a list at line 83.
3.  Add an import at line 1.

**The Scripts (one operation per invocation, bottom-up order):**
```bash
# Setup: Define the indentation function (see "Programmatic Indentation" section)
base=$(sed -n '200s/^\([[:space:]]*\).*/\1/p' filename.py)
unit=$(awk '/^[[:space:]]+[^[:space:]]/{match($0,/^[[:space:]]+/);if(p&&RLENGTH>p){print RLENGTH-p;exit}p=RLENGTH}' filename.py)
unit=${unit:-4}
I() { printf '%s%*s' "$base" $(($1*unit)) ''; }

# Operation 1: Add method after line 200 (highest line number first)
# Uses UNQUOTED heredoc to allow $(I N) expansion
ed -s filename.py <<EDSCRIPT4829
H
200a
$(I 0)def new_method(self):
$(I 1)pass
.
w
q
EDSCRIPT4829

# Verify, then Operation 2: Add item to list at line 83
# Re-measure base indent for this location
base=$(sed -n '83s/^\([[:space:]]*\).*/\1/p' filename.py)
ed -s filename.py <<EDSCRIPT4829
H
83s/$/,/
83a
$(I 0)NewItem
.
w
q
EDSCRIPT4829

# Verify, then Operation 3: Add import at line 1
# No indentation needed, so quoted heredoc is fine
ed -s filename.py <<'EDSCRIPT4829'
H
1i
import time
.
w
q
EDSCRIPT4829
```
### 3. Verify: Check Your Work
After running the script, **verify the changes immediately**, including indentation.

**CRITICAL: Verify indentation after edits with `bin/measure-indent.py` - never count spaces visually.** Use the same command from Step 1:
```bash
# REQUIRED: Measure indent with this tool - never estimate visually
bin/measure-indent.py LINE-SPEC FILE

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
# After edit: verify new line 830 matches its neighbors
bin/measure-indent.py 830 myfile.py
# Output: Line 830: 8

# Or verify with context using a range:
bin/measure-indent.py 829-831 myfile.py
# Output:
#   Line 829: 8
#   Line 830: 8
#   Line 831: 8
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
*   **Why:** `ed` operates on an in-memory buffer. Until you issue `w`, the file on disk remains untouched - this is the source of atomicity.
*   **Warning:** Assertion failures (`s/pattern/&/`) do NOT stop script execution. Ed prints an error but continues with subsequent commands, including `w`. The exit code will be non-zero, but the file will still be written.
*   **Benefit:** If the script crashes (e.g., ed is killed), the file is safe because nothing was written. However, if the script completes, `w` always executes - even after assertion failures.

**CRITICAL LIMITATION:** This atomicity only applies *within a single script*. When using multiple scripts (each with its own `w`), the first script's successful write changes the file state. Any subsequent scripts that rely on outdated information (such as line content from earlier `rg` or `grep` output) will target the wrong content. See "Multi-Script Editing with Verification" for the mandatory verification workflow between scripts.

## Advanced: Strict Contextual Anchoring (Required)

**Why required:** Anchoring costs nothing when it matches, but provides two benefits when the target line has shifted:
1. For substitution edits (`s/old/new/`): if the anchor fails, the edit also won't find `old`, so nothing changes - no corruption
2. For all edits: ed exits with non-zero status, signaling something went wrong

Even with bottom-up editing, you might target the wrong line if the file changed unexpectedly. Assertions help detect this, but they do **not** prevent subsequent commands from executing (see warnings below).

**The Technique:**
Use the substitute command `s/pattern/&/` to verify you're targeting the correct line. If the pattern matches, the command succeeds silently. If not, ed prints an error.
*   **Important:** A failed assertion does NOT stop script execution. Ed prints the error but continues with subsequent commands. The value of assertions is that ed's exit code will be non-zero, allowing you to detect problems after the script completes.
*   **Recommended workflow:** Do one operation per ed invocation (an operation can insert/delete/change multiple lines, but is a single ed command). Verify after each. This avoids the need for in-script assertions entirely.
* **Note:** You must escape special regex characters (like `*`, `[`, `.`) in the pattern. Failure to escape `[` or `.` will cause `ed` to interpret them as regex classes or wildcards, leading to "No match" errors or incorrect edits.

**Example:**
Instead of blindly appending to line 83:
```bash
83s/$/,/
83a
$(I 0)NewItem
.
```
**Strict Assertion:**
Assert that line 83 actually contains "Keys" before editing:
```bash
# 1. Assert: Try to replace 'Keys' with itself (&). Fails if missing.
83s/Keys/&/
# 2. Edit: (executes regardless of assertion result)
83s/$/,/
83a
$(I 0)NewItem
.
```

The assertion helps you notice if line numbers shifted - ed's exit code will be non-zero. However, the edit still executes. For true safety, use one operation per ed invocation and verify after each.

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

**WARNING: The #1 Multi-Script Failure Mode**

After a script writes (`w`), all prior context about the file is potentially stale:
- Line numbers may have shifted
- Line content may have changed
- Output from earlier `rg`, `grep`, or `ed` queries no longer reflects reality

**The failure pattern:**
1. You query the file and note line numbers/content
2. Script 1 runs successfully and writes
3. You run Script 2 using the *original* query results
4. Script 2 fails or corrupts the file because line 50 no longer contains what you expected

**The solution:** After every `w`, you MUST re-query the file before constructing the next script. Never reuse stale context.

When making multiple edits across several `ed` invocations (each with its own `w`), verification between scripts is critical to maintain accuracy.

**Mandatory verification after each script:**
1. Verify the exact change with `ed -s FILE <<< 'H\nSTART,ENDn'`
2. Measure indentation of changed lines with `bin/measure-indent.py` (for code files)
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
# Using unquoted heredoc for $(I N) - base is empty string for column 0
base=""
unit=4
I() { printf '%s%*s' "$base" $(($1*unit)) ''; }

ed -s file.py <<EDSCRIPT3847
H
$a

$(I 0)def new_function():
$(I 1)pass
.
w
q
EDSCRIPT3847

# VERIFY again
python3 -m py_compile file.py || { echo "Syntax error after script 2"; exit 1; }
```

**Key principle:** After each `w` (write), the file state has changed. Subsequent scripts must account for line number shifts from previous edits. Verification catches errors before they accumulate.

**Always use one operation per ed invocation**

The safest approach for any editing task:
1. Make one operation per ed script (inserting 10 lines is one operation; inserting 5 lines then deleting 3 elsewhere is two)
2. Verify the edit succeeded (syntax check, view the lines)
3. Re-query line numbers before the next edit

Assertions (`s/pattern/&/`) can help detect stale line numbers via ed's non-zero exit code, but they do not prevent the edit from executing. If you must combine assertion and edit in one script, check `$?` afterward and revert with `git checkout` if it failed.

**Example: Assertion with edit (less safe - edit runs even if assertion fails)**
```bash
# Script 1 completed - file has changed
# Script 2: Anchor assertion (note: edit still executes if this fails!)
ed -s file.py <<'EDSCRIPT2847'
H
# Assert line 50 still contains expected content BEFORE editing
50s/expected_function_name/&/
# Edit runs regardless - check $? after to detect assertion failure
50s/old_value/new_value/
w
q
EDSCRIPT2847
```

If the assertion failed, `$?` will be non-zero after the script completes. The edit still executed, so you may need to revert with `git checkout` and retry. This is why one-operation-per-invocation is safer.

## Robust Editing Patterns

### Comma Safety
When appending items to lists or dictionaries in code, the previous line might be missing a trailing comma.

**The Pattern:**
This pattern combines a substitution (to ensure the comma exists) with an append - these two ed commands together form one semantic operation (add list item).

```bash
# Ensure line 83 ends with a comma, then append (using unquoted heredoc for $(I N))
83s/$/,/
83a
$(I 0)NewItem
.
```
This works perfectly in reverse order because the `a` command inserts *after* the target line, so line 83 remains stable for the substitution.

### Atomic Scripts
Always use a single `ed` invocation containing **one operation**. Use a **Quoted Heredoc** (`<<'EDSCRIPT4829'`) for content without leading whitespace, or an **Unquoted Heredoc** (`<<EDSCRIPT4829`) when using `$(I N)` for indentation. This ensures the file is opened and written only once, prevents race conditions, and handles special characters safely.

```bash
# Good: One operation per invocation
ed -s file <<'EDSCRIPT4829'
H
50s/old/new/
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


### Editing Files Containing ed Commands

When editing files that contain `ed` commands as content (such as documentation, tutorials, or this guide itself), those content lines can be misinterpreted as actual `ed` commands during editing.

**The Problem:**

Lines in your content that look like `ed` commands will be executed instead of inserted:
- A line containing just `.` will terminate input mode prematurely
- Lines like `w`, `q`, `H`, or line numbers may be interpreted as commands
- Heredoc delimiters in content may conflict with your script's delimiter

**The Solution: Sentinel Prefixes**

Prefix **every line** of inserted content with a sentinel (e.g., `%%`), then remove the sentinels in a second pass. Do not try to selectively prefix only "dangerous" lines - this leads to mistakes.

**The Golden Rule: Prefix Every Inserted Line**

When using `ed` to insert content:
1. **Every line** between the append/insert command and the terminating `.` gets a `%%` prefix
2. The terminating `.` does NOT need a prefix (it's outside the content)
3. After the edit, strip all `%%` prefixes from the inserted range

This eliminates all judgment calls about which lines are "safe."

**Critical: The sentinel only affects the START of each line**

The strip command `s/^%%//` removes `%%` from the **beginning** of each line only. Everything else on the line passes through unchanged. Do not think about "nesting levels" - just add `%%` at the start, and it gets removed from the start. Content elsewhere on the line (including other `%%` sequences) is untouched.

**Example: Inserting a code block with ed commands**

Suppose you want to insert this content after line 50 of `file.md`:

```
This is a code example:
ed -s example.py <<'EDSCRIPT'
H
10,20n
.
w
q
EDSCRIPT
```

**Step 1: Insert with sentinels on EVERY line**

```bash
ed -s file.md <<'EDSCRIPT8472'
H
50a
%%```
%%This is a code example:
%%ed -s example.py <<'EDSCRIPT'
%%H
%%10,20n
%%.
%%w
%%q
%%EDSCRIPT
%%```
.
w
q
EDSCRIPT8472
```

Notice:
- Every content line has `%%` prefix
- The terminating `.` is left as-is (it's part of the outer script, not content)
- No judgment calls about which lines need protection

**Step 2: Verify the insertion (with sentinels still in place)**

```bash
ed -s file.md <<'EDSCRIPT9284'
H
51,61n
Q
EDSCRIPT9284
```

You should see every inserted line starting with `%%`. Verify the count and content are correct before proceeding.

**Step 3: Strip the sentinels**

```bash
ed -s file.md <<'EDSCRIPT9173'
H
51,61s/^%%//
w
q
EDSCRIPT9173
```

**Step 4: Verify the final result**

```bash
ed -s file.md <<'EDSCRIPT8374'
H
51,61n
Q
EDSCRIPT8374
```

Confirm no `%%` prefixes remain and the content is correct.

**Why prefix EVERY line?**

Selectively prefixing only "dangerous" lines leads to errors:
- You might miss a line that looks safe but isn't
- Nesting levels get confusing (content showing sentinel examples needs double prefixes)
- Mental overhead increases with complexity

By prefixing every line unconditionally, you:
- Eliminate judgment calls
- Make the pattern mechanical and reliable
- Can verify correctness by simply checking that all inserted lines start with `%%`

**Sentinel choice:**
- `%%` is recommended: unlikely to appear at the start of real content
- Avoid sentinels that might conflict with the file's actual content

**Nested case: Content that should contain `%%` in the final result**

When inserting content that should display `%%` prefixes after editing (e.g., documentation about this technique), use double prefixes:

- Lines that should end up **clean**: prefix with `%%` (stripped to nothing)
- Lines that should end up **with `%%`**: prefix with `%%%%` (stripped to `%%`)

**Example:** Inserting documentation that shows sentinel usage

Desired final content:
```
To protect this line, prefix it with %%
```

What you type in the ed script:
```bash
ed -s file.md <<'EDSCRIPT1234'
H
50a
%%To protect this line, prefix it with %%%%
.
w
q
EDSCRIPT1234
```

After running `s/^%%//` on the inserted line:
- `%%To protect this line, prefix it with %%%%` becomes `To protect this line, prefix it with %%`

**The rule is simple:** For every layer of stripping, add one `%%` prefix. Most cases need one layer. Nested examples need two.

### Quoted vs Unquoted Heredocs: When to Use Each

**Quoted heredoc (`<<'EDSCRIPT4829'`):**
- **Use for:** Inserting content at column 0 (no leading whitespace) that contains special characters
- **Behavior:** Shell does NOT expand variables (`$var`, `$(cmd)`) or interpret backslashes
- **Best for:** Import statements, top-level definitions, or any column-0 content with `$`, quotes, or backslashes
- **NOT for:** Any content that needs leading whitespace (use unquoted heredoc with `$(I N)` instead)

```bash
# Inserting an import with a dollar sign in a comment (column 0, no indentation)
ed -s file.py <<'EDSCRIPT4829'
H
1i
import os  # Note: $PATH is not expanded here
.
w
q
EDSCRIPT4829
```

**What about code that needs BOTH special characters AND indentation?**
Use an unquoted heredoc with `$(I N)` and escape or quote the special characters:
```bash
base=$(sed -n '50s/^\([[:space:]]*\).*/\1/p' file.py)
unit=4
I() { printf '%s%*s' "$base" $(($1*unit)) ''; }

ed -s file.py <<EDSCRIPT4829
H
50a
$(I 0)def example():
$(I 1)msg = "Hello \$USER"  # Escaped $ to prevent expansion
$(I 1)return msg
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

**Solution 2: Delete and reinsert in new order (multiple operations - less safe)**

This approach requires multiple operations. For maximum safety, use separate invocations:
```bash
# First, note the content of lines 10 and 11 before editing

# Delete line 11 first (bottom-up)
ed -s file.md <<'EDSCRIPT4829'
H
11d
w
q
EDSCRIPT4829

# Delete line 10 (now at original position since we deleted below)
ed -s file.md <<'EDSCRIPT4829'
H
10d
w
q
EDSCRIPT4829

# Insert in new order after line 9
ed -s file.md <<'EDSCRIPT4829'
H
9a
| TASK-029 | (content from original line 11) |
| TASK-030 | (content from original line 10) |
.
w
q
EDSCRIPT4829
```

**Recommendation:** Prefer Solution 1 (`m` command) when possible - it's a single operation.

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

**When to use range delete vs separate invocations:**
- **Range delete (`50,55d`)**: When deleting a *contiguous block* of lines - this is ONE operation
- **Separate invocations**: When deleting *non-contiguous* lines at different locations (e.g., lines 120, 85, and 12)

**Example: Deleting non-contiguous lines (one operation per invocation, bottom-up order)**
```bash
# Delete line 120 first (highest line number)
ed -s file.py <<'EDSCRIPT4829'
H
120d
w
q
EDSCRIPT4829

# Verify, then delete line 85
ed -s file.py <<'EDSCRIPT4829'
H
85d
w
q
EDSCRIPT4829

# Verify, then delete line 12
ed -s file.py <<'EDSCRIPT4829'
H
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
3. Plan one operation per ed invocation (each operation can modify multiple lines)
4. Execute operations in bottom-up order (highest line numbers first) to avoid line-number shifts
5. Verify immediately after execution

**Warning signs you may need to revert:**
- Lost track of which content is where
- Can't explain why an edit failed
- Repeating same verification commands without progress

**Key insight:** Patching a broken edit with more edits can work, but if you find yourself making multiple correction attempts without success, a clean revert + redesign is faster and safer than continued patching.

