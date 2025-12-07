---
goal: Guide for Writing Test Decomposition Plans
version: 1.3.1
date_created: 2025-12-07
last_updated: 2025-12-07
owner: Development Team
status: 'Planned'
tags: [architecture, testing, refactoring, process, documentation]
---

# Introduction

![Status: Planned](https://img.shields.io/badge/status-Planned-blue)

This document provides guidance for writing implementation plans that decompose monolithic test files into modular test packages. It is a meta-plan describing what a good decomposition plan should contain, not a plan for performing the decomposition itself.

## 1. Requirements & Constraints

- **REQ-100**: The plan must be comprehensive enough for an AI agent or developer to execute without ambiguity
- **REQ-200**: The plan must reference the established architectural patterns already in use in the codebase
- **REQ-300**: The plan must include verification criteria that confirm behavior is preserved
- **REQ-400**: The plan must be structured according to the standard implementation plan template

- **CON-100**: Plans should not include inline code snippets; they describe what to do, not how to implement it
- **CON-200**: Plans should reference patterns by name rather than duplicating pattern definitions
- **CON-300**: Plans should be executable in phases that can be validated independently

- **GUD-100**: Use precise, unambiguous language that requires no interpretation
- **GUD-200**: Structure tasks so they can be executed in parallel where dependencies allow
- **GUD-300**: Include measurable completion criteria for each phase
- **GUD-400**: (In output plans) Include a reminder after each phase heading to review Constraints, Risks, and Specification sections (such as Context Class Specification) before implementation
- **GUD-500**: (In output plans) Use blockquote format for phase review reminders: `> **IMPORTANT:** Before implementing this phase, review Section N (Name), Section M (Name)... for applicable items.`

## 2. Implementation Steps

### Phase 1: Gather Context for the Plan

- GOAL-100: Collect all information needed to write a complete decomposition plan

| Task      | Description                                                                                                    | Completed | Date |
| --------- | -------------------------------------------------------------------------------------------------------------- | --------- | ---- |
| TASK-0100 | Identify the monolithic test file to be decomposed                                                             |           |      |
| TASK-0200 | Measure the file's size, step definition count, and fixture count                                              |           |      |
| TASK-0300 | Identify logical sections within the file by examining comments, groupings, and related functionality          |           |      |
| TASK-0400 | Review existing decomposed test packages in the codebase to understand the established pattern                 |           |      |
| TASK-0500 | Identify the corresponding feature file and verify scenario-to-step mappings                                   |           |      |
| TASK-0600 | Document any shared fixtures or utilities the monolith depends on                                              |           |      |
| TASK-0700 | Analyze how pytest-bdd step definitions are registered/discovered in this codebase (e.g., via conftest.py pytest_plugins, direct imports, or other mechanisms) and document any registration steps required for new modules |           |      |
| TASK-0800 | Analyze how steps share state (function attributes, globals, fixtures, context objects) and document whether incremental or batch migration is feasible |           |      |
| TASK-0900 | Scan for duplicate step definitions (same decorator string, multiple implementations) and count unique vs total decorators |           |      |
| TASK-1000 | Identify type or interface inconsistencies across steps that will need standardization (e.g., mixed object attribute vs dictionary access patterns) |           |      |

### Phase 2: Define the Target Structure in the Plan

- GOAL-200: Specify what the decomposed structure will look like

| Task      | Description                                                                                                    | Completed | Date |
| --------- | -------------------------------------------------------------------------------------------------------------- | --------- | ---- |
| TASK-1100 | Define the package directory name following established naming conventions                                     |           |      |
| TASK-1200 | List each step definition module to be created with a one-sentence responsibility description                  |           |      |
| TASK-1300 | Describe what belongs in the context-and-fixtures module (context class, fixtures, helpers)                    |           |      |
| TASK-1400 | Describe what remains in the thin entrypoint after decomposition                                               |           |      |
| TASK-1500 | Map each logical section from the monolith to its target module                                                |           |      |
| TASK-1600 | Include expected step counts per target module (based on mapping from TASK-1500, accounting for duplicates identified in TASK-0900) (for validation that nothing was lost during migration) |           |      |
| TASK-1700 | Specify the context class attributes based on state-sharing analysis from TASK-0800 (what data steps need to share) |           |      |
| TASK-1800 | Specify the context class methods (typically at minimum a `reset()` method) |           |      |

### Phase 3: Specify the Migration Sequence in the Plan

- GOAL-300: Define the order of operations for executing the decomposition

| Task      | Description                                                                                                    | Completed | Date |
| --------- | -------------------------------------------------------------------------------------------------------------- | --------- | ---- |
| TASK-1900 | Specify that infrastructure (package, harness module) must be created before step migrations                   |           |      |
| TASK-2000 | Specify how fixtures and helpers should be extracted before the steps that depend on them                      |           |      |
| TASK-2100 | Specify that plans must include a baseline extraction step before migration to enable exact pattern comparison afterward |           |      |
| TASK-2200 | Specify the order in which step definition groups should be migrated (typically starting from sections at the bottom of the file and working upward) |           |      |
| TASK-2300 | Specify that step decorator strings must be copied verbatim from the original file - no paraphrasing, regeneration, or modification allowed |           |      |
| TASK-2400 | If duplicate step definitions were identified (TASK-0900), specify that they must be consolidated into single implementations during migration |           |      |
| TASK-2500 | If type/interface inconsistencies were identified (TASK-1000), specify the standardization approach (e.g., all results use dictionary format) |           |      |
| TASK-2600 | If state-sharing analysis (TASK-0800) determined batch migration is required, document this decision with rationale |           |      |
| TASK-2700 | Specify that each migration should be verified before proceeding to the next                                   |           |      |
| TASK-2800 | Specify intermediate verification checkpoints after each phase (distinct from per-migration verification in TASK-2700; phase checkpoints confirm the entire system still works) (e.g., "run test suite to confirm tests still pass") |           |      |

### Phase 4: Document Risks and Assumptions in the Plan

- GOAL-400: Capture potential issues and prerequisites

| Task      | Description                                                                                       | Completed | Date |
| --------- | ------------------------------------------------------------------------------------------------- | --------- | ---- |
| TASK-2900 | Document the assumption that the established pattern is the correct model to follow              |           |      |
| TASK-3000 | Document the risk of circular imports and how to avoid them                                      |           |      |
| TASK-3100 | Document the risk of fixture scope changes affecting behavior                                    |           |      |
| TASK-3200 | Document any ordering decorator considerations                                                   |           |      |
| TASK-3300 | Document the risk of step regeneration vs verbatim copying, and require plans to mandate exact copying of decorator strings (reinforcing TASK-2300) |           |      |
| TASK-3400 | Document the correct `__init__.py` import pattern: modules with step definitions must be imported AS MODULES, not just for symbols |           |      |

### Phase 5: Review, Resolve, and Iterate

- GOAL-500: Ensure the plan is internally consistent, unambiguous, and addresses all identified risks

| Task      | Description                                                                                       | Completed | Date |
| --------- | ------------------------------------------------------------------------------------------------- | --------- | ---- |
| TASK-3500 | Review Phases 2-4 for internal consistency (do the target modules match the section mapping? do step counts add up?) |           |      |
| TASK-3600 | Identify any ambiguities in the plan that could lead to multiple interpretations                 |           |      |
| TASK-3700 | Identify any contradictions between plan elements (e.g., task dependencies that conflict)        |           |      |
| TASK-3800 | For each documented risk, verify the migration sequence includes mitigation steps                |           |      |
| TASK-3900 | Identify decisions that require user input and document questions to ask                         |           |      |
| TASK-4000 | If issues are found, return to Phases 2-4 to revise (including documenting any newly discovered risks); repeat Phase 5 until the plan is stable     |           |      |

### Phase 6: Define Verification Criteria in the Plan

- GOAL-600: Specify how to confirm the decomposition was successful

| Task      | Description                                                                                       | Completed | Date |
| --------- | ------------------------------------------------------------------------------------------------- | --------- | ---- |
| TASK-4100 | Specify that unique step definition count before and after must match (accounting for any duplicates consolidated per TASK-0900) |           |      |
| TASK-4200 | Specify that no duplicate step definition warnings should appear                                 |           |      |
| TASK-4300 | Specify that all scenarios must bind to exactly one step definition each                         |           |      |
| TASK-4400 | Specify that test pass/fail status must match the pre-decomposition baseline                     |           |      |
| TASK-4500 | Specify linting and import verification requirements                                             |           |      |
| TASK-4600 | Specify that plans must include concrete verification commands (not just criteria) that can be copy-pasted and executed |           |      |

### Phase 7: Format and Finalize the Plan Document

- GOAL-700: Produce a complete, template-compliant plan document

| Task      | Description                                                                                       | Completed | Date |
| --------- | ------------------------------------------------------------------------------------------------- | --------- | ---- |
| TASK-4700 | Use the standard implementation plan template with all required sections, plus the additional sections described in Section 8 (Standard Plan Sections) |           |      |
| TASK-4800 | Ensure all identifiers follow the numbering convention (spaced appropriately for the expected task count) |           |      |
| TASK-4900 | Include front matter with goal, version, date, owner, status, and tags                           |           |      |
| TASK-5000 | List all files that will be created, modified, or serve as references                            |           |      |
| TASK-5100 | Include related specifications and references to architectural documentation                     |           |      |

## 3. Alternatives

- **ALT-100**: Write plans with inline code examples - rejected because it couples the plan to implementation details that may change
- **ALT-200**: Write plans at a very high level without specific module names - rejected because it leaves too much to interpretation
- **ALT-300**: Skip the plan and decompose directly - rejected because it increases risk of errors and makes review difficult

## 4. Dependencies

- **DEP-100**: The standard implementation plan template used for all plans in the repository
- **DEP-200**: Existing architectural documentation describing the test decomposition pattern
- **DEP-300**: Access to the monolithic test file and its corresponding feature file
- **DEP-400**: Access to existing decomposed test packages as reference examples

## 5. Files

- **FILE-100**: The monolithic test file (input, for analysis)
- **FILE-200**: Existing decomposed test packages (reference, for pattern extraction)
- **FILE-300**: Architectural plan documentation (reference)
- **FILE-400**: The resulting decomposition plan document (output)

## 6. Testing

- **TEST-100**: The plan document passes template validation (all required sections present)
- **TEST-200**: All tasks in the plan are unambiguous and actionable
- **TEST-300**: The plan can be reviewed and understood without additional context
- **TEST-400**: The plan references verification criteria that can be objectively measured

## 7. Risks & Assumptions

- **RISK-100**: Insufficient analysis may result in a plan that misses important fixtures or helpers
- **RISK-200**: Overly granular module splitting may make the plan harder to execute than the monolith was to maintain
- **RISK-300**: The plan may become stale if the monolith changes before execution
- **RISK-400**: Step definitions may be regenerated/paraphrased instead of copied verbatim, causing pattern mismatches with the feature file
- **RISK-500**: Modules containing step definitions may be imported only for symbols (not as modules), preventing pytest-bdd from registering the steps
- **RISK-600**: The codebase may use a step registration mechanism (such as `pytest_plugins` in conftest.py) that requires explicit registration of new modules beyond just importing them in the entrypoint file
- **RISK-700**: Duplicate step definitions may exist in the monolith (same decorator string, multiple implementations) - only the last-registered implementation runs, and during migration these must be consolidated into single implementations
- **RISK-800**: Steps may share state via function attributes or other non-obvious mechanisms, forcing batch migration instead of incremental migration
- **RISK-900**: Type or interface inconsistencies across steps (e.g., some steps using object attributes, others using dictionary keys for the same data) may require standardization during migration

- **ASSUMPTION-100**: The person writing the plan has access to the codebase and can perform the necessary analysis
- **ASSUMPTION-200**: The established decomposition pattern is documented or can be inferred from existing examples
- **ASSUMPTION-300**: The plan will be executed relatively soon after writing, before significant codebase drift
- **ASSUMPTION-400**: When duplicate step definitions exist, the correct implementation to keep can be determined by analysis (typically the most complete or most recently written version)

## 8. Standard Plan Sections

Plans written using this guide should include the following sections beyond the standard template sections (Introduction, Requirements & Constraints, Implementation Steps, Alternatives, Dependencies, Files, Testing, Risks & Assumptions, Related Specifications):

### 8.1 Phase Results Tables

Each phase should include a results table template for recording outcomes during execution:

| Metric | Expected | Actual | Status |
| ------ | -------- | ------ | ------ |
| (metric name) | (expected value) | (to be filled) | (PASS/FAIL) |

### 8.2 Context Class Specification

Include a specification section defining the shared context class with:
- Table of attributes with types and purposes
- Table of methods with purposes
- Note indicating which existing pattern it follows (if any)

### 8.3 Section-to-Module Mapping Table

Include a dedicated section with a two-column table mapping each logical section in the monolith to its target module. This serves as the authoritative reference for migration.

### 8.4 Decision Documentation

When significant choices affect the approach (e.g., batch vs incremental migration), document them using the pattern:

**Decision (DEC-NNNN):** [Brief statement of the decision and rationale]

## 9. Related Specifications / Further Reading

- Standard implementation plan template
- Architectural plan for new test suites
- Existing decomposition plans for reference (if available)
