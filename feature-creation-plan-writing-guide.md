---
goal: Guide for Writing Feature Implementation Plans
version: 1.0.0
date_created: 2025-12-09
last_updated: 2025-12-09
owner: Development Team
status: 'Active'
tags: [architecture, planning, process, documentation, features]
---

# Introduction

![Status: Active](https://img.shields.io/badge/status-Active-green)

This document provides guidance for writing implementation plans for new features, enhancements, or significant changes to the codebase. It is a meta-plan describing what a good feature implementation plan should contain, not a plan for implementing a specific feature.

A well-written plan enables an AI agent or developer to execute the implementation without ambiguity, reduces risk through explicit decision documentation, and provides traceability from requirements through verification.

## 1. Requirements & Constraints

### Requirements

- **REQ-100**: The plan must be comprehensive enough for an AI agent or developer to execute without ambiguity
- **REQ-200**: The plan must document all significant decisions made during planning with rationale
- **REQ-300**: The plan must include verification criteria that confirm the feature works correctly
- **REQ-400**: The plan must be structured according to the standard implementation plan template
- **REQ-500**: The plan must identify all files to be created, modified, or deleted

### Constraints

- **CON-100**: Plans should not include inline code snippets; they describe what to do, not how to implement it
- **CON-200**: Plans should reference decisions by identifier (DEC-NNNN) rather than restating rationale
- **CON-300**: Plans should be executable in phases that can be validated independently
- **CON-400**: Generated code must not contain references to plan identifiers (DEC-NNNN, TASK-NNNN, REQ-NNNN, etc.); code comments must be self-explanatory without requiring access to the plan document

### Guidelines

Guidelines are divided into two categories: planning-time guidelines (applied when writing the plan) and implementation-time guidelines (applied when executing the plan).

#### Planning-Time Guidelines

- **GUD-0100**: Use precise, unambiguous language that requires no interpretation
- **GUD-0200**: Structure tasks so they can be executed in parallel where dependencies allow
- **GUD-0300**: Include measurable completion criteria for each phase
- **GUD-0400**: Phase review reminders will be verified in Phase 7; use blockquote format: `> **IMPORTANT:** Before implementing this phase, review the Constraints, Decisions, and Risks sections for applicable items.`
- **GUD-0500**: Task descriptions must describe exactly one action to perform; never include negations like "DO NOT DO THIS YET" or conditional clauses that contradict the main action
- **GUD-0600**: Plans must not reference line numbers for code that will be modified, moved, or deleted; identify code by unique characteristics (class name, function name, decorator string)
- **GUD-0700**: Tasks within each phase must be ordered by dependency; if task B requires task A's output, task A must precede task B
- **GUD-0800**: Always derive metrics from automated tooling, not manual inspection; include the verification command and its output in the plan
- **GUD-0900**: Document registration/discovery mechanisms explicitly; when new modules need to be discovered by a framework (pytest, plugins, entry points), specify the registration steps required
- **GUD-1000**: Scan for naming collisions across the entire system, not just the scope being modified; globally-registered names must be unique system-wide
- **GUD-1100**: Analyze state dependencies before deciding on migration strategy; some changes must be atomic (batch), others can be incremental
- **GUD-1200**: Identify duplicates and conflicts in existing code before adding new code that might compound the problem

#### Implementation-Time Guidelines

> **Note:** These guidelines apply when *executing* an implementation plan, not when *writing* one. They are included here for reference and should be copied into implementation plans where relevant.

- **GUD-1300**: Before adding an import, verify the module is actually required by the code being written; do not add imports based on anticipated future use
- **GUD-1400**: When defining module-level collections that will be iterated by multiple functions, include explicit type annotations
- **GUD-1500**: When writing new code that uses an existing untyped or loosely-typed module-level variable, first add or tighten the type annotation on that variable
- **GUD-1600**: When moving code to new modules, re-export from the original location for backward compatibility

## 2. Implementation Steps

### Phase 1: Gather Requirements and Context

> **Note:** Phase reminders for Constraints/Decisions/Risks are not applicable here since those sections do not yet exist at this stage.

- GOAL-100: Collect all information needed to write a complete implementation plan

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-0100 | Create YAML front matter with goal, version, dates, status, and tags (see Section 8.1 later in this guide) | | |
| TASK-0200 | Document the feature goal in a single sentence that could serve as the plan's title | | |
| TASK-0300 | Identify the user-facing behavior or API the feature must provide | | |
| TASK-0400 | Identify any existing code, patterns, or modules the feature must integrate with | | |
| TASK-0500 | Identify external dependencies (libraries, services, APIs) the feature requires | | |
| TASK-0600 | Document any constraints from the user (performance, compatibility, style) | | |
| TASK-0700 | Identify similar features in the codebase that can serve as implementation patterns | | |
| TASK-0800 | List questions that require user input before planning can proceed | | |

### Phase 2: Conduct Planning Discussion

> **Note:** See Section 9 (Planning Discussion Protocol) for guidance on the one-question-at-a-time approach. Phase reminders for Constraints/Decisions/Risks are not applicable here since those sections are being created during this phase.

- GOAL-200: Resolve open questions and document decisions with rationale

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-0900 | Present each question from TASK-0800 to the user, one at a time | | |
| TASK-1000 | For each answer, document a decision as an inline note: `[DEC-NNNN] Brief statement of decision and rationale` | | |
| TASK-1100 | Identify follow-up questions that arise from answers and repeat until all decisions are made | | |
| TASK-1200 | Review decisions for internal consistency; flag any contradictions for resolution | | |
| TASK-1300 | Summarize the feature scope based on all decisions made | | |
| TASK-1400 | Create the formal Decisions section and consolidate all inline DEC-NNNN entries into it with full rationale and question references | | |

### Phase 3: Define the Target Architecture

> **IMPORTANT:** Before implementing this phase, review the Constraints, Decisions, and Risks sections for applicable items.

- GOAL-300: Specify what the implementation will look like when complete

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-1500 | List each new file to be created with a one-sentence responsibility description | | |
| TASK-1600 | List each existing file to be modified with a description of the changes | | |
| TASK-1700 | List any files to be deleted or deprecated | | |
| TASK-1800 | Define new classes, functions, or interfaces with their responsibilities | | |
| TASK-1900 | Specify how the new code integrates with existing modules (imports, registrations, configurations) | | |
| TASK-2000 | Document any new configuration options, CLI arguments, or environment variables | | |
| TASK-2100 | If the feature has complex behavior, create a dedicated specification section (see Section 8.3) | | |

### Phase 4: Document Risks and Assumptions

> **Note:** The Risks section does not yet exist at this stage; it is created during this phase. Review the Constraints and Decisions sections for applicable items.
- GOAL-400: Capture potential issues and prerequisites

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-2200 | Document assumptions about the codebase, environment, or user needs | | |
| TASK-2300 | Identify risks related to backward compatibility | | |
| TASK-2400 | Identify risks related to external dependencies (API changes, version conflicts) | | |
| TASK-2500 | Identify risks related to performance or scalability | | |
| TASK-2600 | Identify risks related to error handling and edge cases | | |
| TASK-2700 | For each risk, document mitigation strategies or acceptance criteria | | |

### Phase 5: Plan the Implementation Sequence

> **IMPORTANT:** Before implementing this phase, review the Constraints, Decisions, and Risks sections for applicable items.

- GOAL-500: Define the order of operations for executing the implementation

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-2800 | Group related tasks into phases with clear goals | | |
| TASK-2900 | Order phases so that foundational work (infrastructure, schemas, utilities) comes before dependent work (business logic, UI, CLI) | | |
| TASK-3000 | Within each phase, order tasks by dependency (files must exist before they can be imported; schemas before validators) | | |
| TASK-3100 | Identify tasks that can be parallelized within a phase | | |
| TASK-3200 | Review risks from Phase 4 and assign mitigations to appropriate implementation phases | | |
| TASK-3300 | Add a placeholder verification task at the end of each phase (to be finalized in Phase 6) | | |

### Phase 6: Define Testing and Verification

> **IMPORTANT:** Before implementing this phase, review the Constraints, Decisions, and Risks sections for applicable items.

- GOAL-600: Specify how to confirm the implementation is correct

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-3400 | List unit tests to be created or modified | | |
| TASK-3500 | List integration tests if the feature spans multiple components | | |
| TASK-3600 | Specify manual verification steps for behavior that is difficult to test automatically | | |
| TASK-3700 | Include concrete, copy-pasteable verification commands (not just criteria) | | |
| TASK-3800 | Specify that the full test suite must pass, not just tests for the new feature | | |
| TASK-3900 | If the feature modifies existing behavior, specify regression testing requirements | | |

### Phase 7: Review and Finalize the Plan

> **IMPORTANT:** Before implementing this phase, review the Constraints, Decisions, and Risks sections for applicable items.

- GOAL-700: Ensure the plan is internally consistent, complete, and unambiguous

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-4000 | Verify phase review reminders (per GUD-0400 blockquote format) are present in all implementation phases in the plan being written, referencing relevant Constraints, Decisions, and Risks | | |
| TASK-4100 | Review all cross-references (DEC-NNNN, TASK-NNNN) for correctness | | |
| TASK-4200 | Verify all phase reminders reference the correct section names and item identifiers | | |
| TASK-4300 | Check that task dependencies within phases are correctly ordered | | |
| TASK-4400 | Verify that every decision referenced in tasks actually exists in the Decisions section | | |
| TASK-4500 | Verify that every risk has corresponding mitigation in the implementation phases | | |
| TASK-4600 | Ensure all identifiers follow the numbering convention with appropriate spacing | | |
| TASK-4700 | Confirm the plan includes all standard sections (see Section 8.1) | | |
| TASK-4800 | If issues are found, return to Phases 3-6 to revise; repeat Phase 7 (Review) until stable | | |

## 3. Alternatives

- **ALT-100**: Write plans with inline code examples - rejected because it couples the plan to implementation details that may change and encourages copy-paste without understanding
- **ALT-200**: Write plans at a very high level without specific file/function names - rejected because it leaves too much to interpretation and risks inconsistent implementation
- **ALT-300**: Skip the planning phase and implement directly - rejected because it increases risk of rework, makes review difficult, and loses decision rationale
- **ALT-400**: Document decisions without question references - rejected because it loses traceability to the planning discussion

## 4. Dependencies

- **DEP-100**: Access to the codebase for analysis of existing patterns
- **DEP-200**: User availability for answering planning questions
- **DEP-300**: Understanding of the project's coding standards and conventions
- **DEP-400**: Access to any external API documentation the feature will use

## 5. Files

- **FILE-100**: The resulting implementation plan document (output)
- **FILE-200**: This guide document (reference)
- **FILE-300**: Any existing implementation plans as examples (reference)

## 6. Plan Quality Criteria

- **TEST-100**: The plan document contains all required sections (see Section 8.1)
- **TEST-200**: All tasks in the plan are unambiguous and describe exactly one action
- **TEST-300**: All decisions include rationale and question reference
- **TEST-400**: All cross-references are valid (no dangling DEC-NNNN or TASK-NNNN)
- **TEST-500**: The plan can be understood without additional context or conversation history

## 7. Risks & Assumptions

### Risks

- **RISK-0100**: Insufficient requirements gathering may result in a plan that misses important constraints
- **RISK-0200**: The plan may become stale if the codebase changes significantly before execution
- **RISK-0300**: Overly detailed plans may be harder to maintain than the flexibility they provide
- **RISK-0400**: Plans written without user input on key decisions may require significant revision
- **RISK-0500**: Line number references become stale after file modifications; identify code by content instead (see also GUD-0600)
- **RISK-0600**: Cross-references in phase reminders may use incorrect section names or item identifiers if sections are renamed or identifiers renumbered during drafting
- **RISK-0700**: Naming collisions may exist across modules when new code is registered globally; scan the entire system for conflicts, not just the scope being modified
- **RISK-0800**: State dependencies between components may force batch migration instead of incremental changes; analyze state sharing before choosing migration strategy
- **RISK-0900**: Duplicate implementations of the same interface may exist; identify and consolidate before adding new code that compounds the problem
- **RISK-1000**: Metrics and counts derived from manual inspection are error-prone; always use automated tooling (e.g., `rg -c 'pattern'`) to derive counts

### Assumptions

- **ASSUMPTION-100**: The person writing the plan has access to the codebase and can perform necessary analysis
- **ASSUMPTION-200**: The plan will be executed relatively soon after writing, before significant codebase drift
- **ASSUMPTION-300**: The user is available to answer planning questions in a reasonable timeframe
- **ASSUMPTION-400**: Existing patterns in the codebase are appropriate to follow for new features

## 8. Standard Plan Sections

### 8.1 Required Sections

Every implementation plan should include these sections:

1. **Front Matter** - YAML header with goal, version, dates, status, tags
2. **Introduction** - Brief description of what the plan accomplishes
3. **Requirements & Constraints** - REQ-NNN, CON-NNN, GUD-NNNN items
4. **Implementation Steps** - Phases with GOAL-NNN and TASK-NNNN tables
5. **Alternatives** - ALT-NNN rejected approaches with rationale
6. **Dependencies** - DEP-NNN items for external requirements
7. **Files** - FILE-NNNN items categorized as create/modify/delete/reference
8. **Plan Quality Criteria / Testing** - TEST-NNN verification criteria with concrete commands
9. **Risks & Assumptions** - RISK-NNNN and ASSUMPTION-NNN items
10. **Decisions** - DEC-NNNN items with question references and rationale (created during Phase 2)
11. **Related Specifications** - Links to external documentation

Note: Sections 8 (Standard Plan Sections) and 9 (Planning Discussion Protocol) in this guide are meta-documentation for plan authors; they are not sections to include in individual implementation plans.

### 8.2 Phase Structure

Each implementation phase should include:

```markdown
### Phase N: Phase Title

> **IMPORTANT:** Before implementing this phase, review the Constraints, Decisions, and Risks sections for applicable items.

- **GOAL-NNN**: One-sentence description of what this phase accomplishes

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-NNNN | Task description | | |
| TASK-NNNN | Verify all Phase N goals are met by running verification commands | | |

#### Phase N Results

| Metric | Expected | Actual | Status |
|--------|----------|--------|--------|
| (metric name) | (expected value) | (to be filled) | (PASS/FAIL) |
```

### 8.3 Specification Sections

For features with complex behavior, add dedicated specification sections that serve as authoritative references. Examples:

- **API Specification**: Endpoints, request/response formats, error codes
- **Configuration Specification**: Option names, types, defaults, validation rules
- **Data Format Specification**: Schema definitions, field requirements
- **Behavior Specification**: State machines, decision trees, algorithm descriptions

Format specification sections with tables and clear definitions:

```markdown
## N. Specification: Feature Name

Per DEC-NNNN, DEC-MMMM, the feature behaves as follows:

| Property | Value | Description |
|----------|-------|-------------|
| ... | ... | ... |
```

### 8.4 Decision Documentation

Document decisions using this pattern:

```markdown
**Decision (DEC-NNNN):** [QNNN] Brief statement of the decision. Rationale explaining why this choice was made over alternatives.
```

Key elements:
- Sequential numbering with gaps for future insertions (0100, 0200, 0300...)
- Question reference [QNNN] for traceability to planning discussion
- Both the decision AND the reasoning in the same entry

### 8.5 Identifier Numbering Conventions

| Type | Format | Example | Spacing |
|------|--------|---------|---------|
| Requirements | REQ-NNN | REQ-100 | 100s |
| Constraints | CON-NNN | CON-100 | 100s |
| Guidelines | GUD-NNNN | GUD-0100 | 100s |
| Decisions | DEC-NNNN | DEC-100 | 100s |
| Dependencies | DEP-NNN | DEP-100 | 100s |
| Tasks | TASK-NNNN | TASK-0200 | 100s within phase |
| Risks | RISK-NNNN | RISK-0100 | 100s |
| Alternatives | ALT-NNN | ALT-100 | 100s |
| Files | FILE-NNNN | FILE-400 | 100s |
| Tests | TEST-NNN | TEST-100 | 100s |
| Goals | GOAL-NNN | GOAL-100 | 100s per phase |
| Assumptions | ASSUMPTION-NNN | ASSUMPTION-100 | 100s |

Spacing by 100s allows inserting items between existing ones without renumbering.

Use 4-digit identifiers (NNNN) for items that may exceed 9 per plan (Guidelines, Decisions, Tasks, Risks, Files). Use 3-digit identifiers (NNN) for items typically fewer than 10 per plan (Requirements, Constraints, Dependencies, Alternatives, Tests, Goals, Assumptions).

If identifiers need to be renumbered after significant plan revisions, use the `bin/renumber-prefixes` tool:

```bash
# Renumber all identifiers with default gap of 100
bin/renumber-prefixes plan.md plan-renumbered.md

# Preview mapping without writing
bin/renumber-prefixes plan.md plan-renumbered.md --verbose
```

### 8.6 Common Patterns

This section documents patterns that recur across feature implementations.

#### Backward Compatibility via Re-exports

When moving code to new modules, re-export from the original location:

```python
# In original_module.py after moving SomeClass to new_module.py
from new_module import SomeClass  # Re-export for backward compatibility
```

This allows existing code to continue importing from the original location.

#### Bootstrap Problem Detection

Identify circular dependencies in user workflows. Example: if a tool requires a config file to run, but the tool generates config files, new users cannot bootstrap. Solution: make config-generating commands work without requiring a config.

#### TTY/Interactive Requirements

Features with interactive prompts must:
1. Detect if running with a TTY (`sys.stdin.isatty()`)
2. Refuse to run non-interactively with a clear error message
3. Suggest non-interactive alternatives (e.g., "use --generate instead and migrate values manually")

#### Placeholder vs Example Values

For generated configuration files:
- Use obvious placeholder values in actual keys: `api-key = "your-api-key-here"`
- Include realistic examples in comments: `# Example: sk-abc123...`

This makes it clear which values need replacement while showing the expected format.

#### Prose Content Separation

Store prose-heavy documentation (prefaces, headers, explanatory text) in separate resource files rather than embedding in code. Benefits:
- Easier to edit without touching code
- Format matches output exactly (e.g., TOML comments for TOML output)
- Can be reviewed/edited by non-developers

#### Debugging and Fix Planning

When planning features that fix bugs or modify existing behavior:

1. **Trace the lifecycle, not the error location** - Find where problematic values are created, assigned, and used. The root cause is usually upstream of where symptoms appear. Search for all assignment sites, not just the line the error mentions.

2. **Map all possibilities before fixing** - Before adding any fix, enumerate every value or state the problematic element can hold across all code paths. Write them down explicitly. The correct fix often emerges naturally once you see the complete picture.

3. **Fix at the source, not the symptom** - When multiple errors share a common cause, resist fixing each error individually. Look for the single upstream change that resolves all of them. One declaration or structural fix beats N scattered guards.

4. **Prefer declarations over guards** - When a type checker or validator complains, ask "Is this properly declared?" not "How do I guard this usage?" Guards and assertions are downstream fixes; declarations and contracts are upstream fixes. Prefer upstream.

5. **Validate inputs at component boundaries** - Errors frequently occur at hand-off points between components. Confirm that the output of component A honors the exact expectations of component B. Specify data structure shape, content, and key names in the plan.

6. **Structure for isolated testing** - Plan the implementation so components can be tested in isolation before integration. When a large system fails, the bug is often masked by separate features that change system state.


## 9. Planning Discussion Protocol

When conducting the planning discussion (Phase 2), follow this protocol:

1. **Number questions sequentially** - Use Q1, Q2, Q3... in a monotonically increasing sequence so each question can be unambiguously referred to from anywhere in the conversation
2. **Present one question at a time** - Only ask one choice at a time and wait for the user's reply before moving on to the next question
3. **Explain the issue first** - Before presenting options, explain the issue and why it is a problem
4. **Compare options thoroughly** - For each option, describe pros, cons, ripple effects, and potential unintended consequences compared to the other options
5. **Make a recommendation** - State which option you recommend and explain why
6. **Ask the user to choose** - After presenting the analysis and recommendation, ask the user to decide
7. **Document immediately** - Create the DEC-NNNN entry as soon as the user decides
8. **Identify follow-up questions** - Answers often raise new questions; add them to the queue

Example question format:

```
**Q5:** Should the CLI command support reading from stdin?

**The Issue:** Currently the command only accepts a filename argument. This means users cannot pipe output from other commands directly into the tool, which limits scripting flexibility and deviates from Unix conventions.

**Option A: Yes, support stdin**
- Pro: Enables piping and scripting workflows
- Pro: Follows Unix conventions that users expect
- Con: Complicates implementation (must detect stdin vs file)
- Con: May conflict with interactive prompts if stdin is used for both
- Ripple effect: Error messages must distinguish "no input" from "empty file"
- Unintended consequence: Users may accidentally hang the command waiting for stdin

**Option B: No, require explicit filename**
- Pro: Simpler implementation
- Pro: Clearer error messages ("file not found" vs ambiguous stdin states)
- Con: Less flexible for advanced users who expect Unix-style piping
- Ripple effect: None significant
- Unintended consequence: May frustrate users familiar with Unix pipelines

**Recommendation:** Option A, because Unix conventions are important for CLI tools and the additional complexity is manageable. The stdin detection pattern is well-established and the interactive prompt conflict can be resolved by requiring `--interactive` flag.

Which option do you prefer?
```

## 10. Decisions

This section is populated during Phase 2 (Planning Discussion). Each decision references the question that prompted it.

(Decisions will be added here as DEC-NNNN entries during the planning discussion phase.)
