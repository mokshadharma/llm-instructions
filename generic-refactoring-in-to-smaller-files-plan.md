# Refactoring Into Smaller Files: A Principles Guide

This guide describes the goals, constraints, and approach for refactoring a codebase to enforce smaller file sizes and reduced nesting depth. The principles here apply to any project regardless of language or framework.

## Why Refactor Into Smaller Files?

Large files with deeply nested code are harder to understand, test, and maintain. By enforcing limits on file size and nesting depth, you:

- Improve readability by keeping each file focused on a single responsibility
- Make code easier to navigate and reason about
- Reduce merge conflicts by distributing code across more files
- Enable better code reuse through explicit, well-parameterized helper functions
- Make testing easier by exposing more surface area at module level

## Target Constraints

These are recommended defaults; adjust based on your project's needs.

### File Size Limit

Every source file should be **100 lines or fewer**. This is a hard limit that forces meaningful decomposition rather than arbitrary splitting.

To leave room for future growth without immediately hitting the limit, aim for **70-80 lines per file** after refactoring, not 99.

### Nesting Depth Limit

No code should exceed **3 levels of indentation** from module level:

- Level 0: Module level (top of file)
- Level 1: Class body or top-level function body
- Level 2: Method body (inside a class)
- Level 3: One nested block (loop, conditional, context manager, etc.)

Code requiring deeper nesting should be extracted into separate functions.

## Requirements

### Extract Helpers as Module-Level Functions

When extracting logic to reduce nesting or file size, create **module-level functions with explicit parameters**, not additional methods on existing classes.

This approach:
- Reduces file size (methods still count toward the containing file's line count)
- Makes dependencies explicit through function parameters
- Enables moving the function to a separate file without changing its signature
- Improves testability by allowing direct unit testing of the extracted function

### No Circular Imports

Circular imports are **strictly prohibited**. This constraint has no exceptions.

When splitting files, you must structure the code so that dependencies flow in one direction. If you find yourself creating a circular dependency, it indicates that:
- The split point is wrong, or
- Some shared code needs to be extracted to a third module that both can import

### Update All Import Sites Directly

When moving code to new files, update every import site to use the new location directly. Do not create re-exports from the original location for backward compatibility.

Re-exports create confusion about the canonical import path and accumulate technical debt. It's better to update all imports immediately, even if there are many of them.

### Keep Test Imports in Sync

Whenever you refactor a file, update the corresponding test imports as part of the same task. Do not leave tests importing from old locations; verify they pass before moving on.

## Guidelines

### Follow Existing Code Style

Maintain consistency with the project's established conventions for:
- Naming (functions, classes, modules, files)
- Formatting and whitespace
- Documentation and comments
- Error handling patterns

The refactoring should be invisible to someone reading the code—it should look like it was always written this way.

### Verify After Each Change

After refactoring each file:
1. Run the full test suite to detect regressions
2. Verify the application starts and basic functionality works
3. Fix any issues before proceeding to the next file

Do not batch multiple file refactorings together without intermediate verification. Small, verified steps catch problems early when they're easy to diagnose.

## Recommended Approach: Dependency-Ordered Phases

Refactor files in dependency order to minimize touching the same file multiple times. When you refactor a leaf module, you won't need to revisit it when its dependents are refactored later.

### Phase 1: Leaf Modules

Start with files that have **no internal dependencies** on other files you plan to refactor. These are the "leaves" of your dependency tree—they may depend on external libraries or standard library modules, but not on other project files in scope for refactoring.

Refactoring these first ensures a stable foundation.

### Phase 2: Mid-Level Modules

Next, refactor files that depend **only on leaf modules** (which are now stable) or on modules outside the refactoring scope.

### Phase 3: High-Level Modules

Finally, refactor files that depend on mid-level modules. These are typically the "entry points" or orchestration layers of your application.

### Phase 4: Documentation Updates

After all code refactoring is complete, update documentation to reflect the new file structure:
- Architecture documents
- Developer guides
- Any documentation that references file names, classes, methods, or functions that changed

## Determining Dependency Order

Before starting, analyze your codebase to determine which files depend on which:

1. For each file in scope, list its imports of other in-scope files
2. Build a dependency graph
3. Identify files with no in-scope dependencies (leaf modules)
4. Order remaining files by depth in the dependency graph

If you discover circular dependencies during this analysis, resolve them before beginning the refactoring.

## Common Extraction Patterns

When reducing nesting or file size, look for these extraction opportunities:

### Loop Body Extraction

If a loop body is complex, extract it to a function:

```
# Before (4 levels of nesting)
for item in items:
    if condition(item):
        for sub in item.children:
            process(sub)

# After (3 levels of nesting)
def process_item(item):
    if condition(item):
        for sub in item.children:
            process(sub)

for item in items:
    process_item(item)
```

### Conditional Branch Extraction

If conditional branches are substantial, extract each branch:

```
# Before
if mode == "A":
    # 20 lines of A logic
elif mode == "B":
    # 20 lines of B logic
else:
    # 20 lines of C logic

# After
if mode == "A":
    handle_mode_a(context)
elif mode == "B":
    handle_mode_b(context)
else:
    handle_mode_c(context)
```

### Splitting a Large File

When a file exceeds the line limit even after extracting helpers:

1. Identify natural groupings of functions/classes by responsibility
2. Create new files named to reflect their responsibility
3. Move related code together to the new files
4. Update imports everywhere

## Risks and Mitigations

### Risk: Subtle Bugs from Refactoring

Refactoring can introduce bugs not caught by tests.

**Mitigation:** Verify after each file. If test coverage is low, consider adding tests before refactoring.

### Risk: Difficult Split Points

Some files may be hard to split cleanly at natural boundaries.

**Mitigation:** If a clean split isn't obvious, look for:
- Functions that are only called from one place (keep them together)
- Functions that share data structures (consider a types/data module)
- Functions that represent distinct phases or responsibilities (split by phase)

### Risk: Proliferation of Tiny Files

Aggressive splitting can create too many small files, making navigation harder.

**Mitigation:** The 70-80 line target (not 99) provides buffer. Don't split a 50-line file just because you could.

## Summary

The goal is a codebase where:
- Every file is 100 lines or fewer
- No code exceeds 3 levels of nesting
- Dependencies flow in one direction with no cycles
- Each file has a clear, focused responsibility
- Tests pass and the application works after every incremental change

Work in dependency order (leaves first), extract helpers as module-level functions with explicit parameters, update all imports directly, and verify continuously.
