**Target Function:** `[FUNCTION_NAME]`

Refactor this function following these principles:

## Core Objective
Transform the function into a clean orchestrator that contains **zero conditional logic**. It should be a linear sequence of function calls where each called function makes its own decisions internally.

## Refactoring Steps

1. **Identify all conditional branches** (if/else statements, ternary operators, early returns)

2. **Extract each conditional branch into a dedicated, simple, single-purpose function** that:
   - Has a clear, descriptive name indicating its purpose
   - Contains the decision-making logic internally
   - Handles all side effects related to that branch
   - Returns what the caller needs (or nothing if it handles everything)

3. **For mutually exclusive branches**, create a dispatcher function that:
   - Takes the condition parameters
   - Decides which path to execute internally
   - Calls the appropriate helper function
   - Never exposes the branching logic to its caller

4. **Consolidate related operations** into single functions:
   - Merge functions that are always called together
   - Combine conditional message/logging with the operations they describe
   - Group setup and teardown operations that belong to the same workflow

5. **Handle early exits** by:
   - Making functions that need to exit call `sys.exit()` internally
   - Or creating wrapper functions that handle the branching and call the appropriate exit-handling function

6. **Final result** should be a function that reads like a recipe:
   - One statement per line
   - Each line is a meaningful step in the workflow
   - No if/else/return/break/continue keywords
   - No ternary operators or inline conditionals
   - The flow is obvious from top to bottom

## Anti-Patterns to Eliminate
- `if condition: func_a() else: func_b()`
- `if condition: return something`
- `value = x if condition else y`
- boolean short-circuit evaluation
- Checking flags/modes before calling functions

## Desired Pattern Example
```python
def target_function():
    """Clear description of overall purpose."""
    result1 = step_one()
    result2 = step_two(result1)
    step_three(result2)
    step_four(result2)

## Note: Keep in mind for testing purposes that program expects to be run with python3 from a venv, in which all the dependencies reside.  So it should be tested with .venv/bin/python3, not python.
