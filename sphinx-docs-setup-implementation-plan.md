---
goal: Set up Sphinx documentation with MyST Parser for a Python project using uv
version: 1.0.0
date_created: 2025-12-11
last_updated: 2025-12-11
owner: Development Team
status: 'Active'
tags: [documentation, sphinx, myst, python, uv]
---

# Sphinx Documentation Setup Plan

![Status: Active](https://img.shields.io/badge/status-Active-green)

This plan provides step-by-step instructions for setting up Sphinx-based documentation with MyST Parser (Markdown support) for any Python project that uses the `uv` package manager. The setup includes auto-generated API documentation from docstrings, multiple output formats, and optional executable code examples.

## 1. Requirements & Constraints

### Requirements

- **REQ-100**: Documentation must be written in Markdown (not reStructuredText)
- **REQ-200**: API documentation must be auto-generated from Python docstrings
- **REQ-300**: Documentation must build to HTML, man pages, single-page HTML, and Markdown formats
- **REQ-400**: Build commands must use `uv run` to ensure correct virtual environment
- **REQ-500**: The setup must work with any uv-managed Python project

### Constraints

- **CON-100**: The target project must have a `pyproject.toml` file (required for uv)
- **CON-200**: The target project must use a flat package layout (`<project-root>/<package-name>/`); src-layout projects must adjust paths as noted
- **CON-300**: This plan does not include CI/GitHub Actions configuration
- **CON-400**: This plan does not include the project's Python package itself; only documentation infrastructure

### Guidelines

- **GUD-0100**: All paths in this plan use placeholders: `<project-root>` for the project directory, `<package-name>` for the Python package name
- **GUD-0200**: Replace placeholder values (marked with `<...>`) with actual project-specific values during implementation
- **GUD-0300**: The HTML theme (`pydata-sphinx-theme`) can be swapped for alternatives (Furo, Alabaster) by changing `html_theme` in `conf.py` and updating dependencies
- **GUD-0400**: If using a `src/` layout, adjust autodoc2 path from `"../../<package-name>"` to `"../../src/<package-name>"`

## 2. Implementation Steps

### Phase 1: Add Documentation Dependencies

> **IMPORTANT:** Before implementing this phase, review the Constraints and Decisions sections for applicable items.

- **GOAL-100**: Add all required documentation dependencies to the project

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-0100 | Open `<project-root>/pyproject.toml` and locate or create the `[dependency-groups]` section | | |
| TASK-0200 | Add a `docs` dependency group (or add to existing `dev` group) containing: `sphinx>=8.0`, `myst-parser>=4.0`, `sphinx-autodoc2>=0.5.0`, `pydata-sphinx-theme>=0.16`, `sphinxcontrib-mermaid>=1.0`, `linkify-it-py>=2.0` | | |
| TASK-0300 | Run `uv sync` to install the new dependencies | | |
| TASK-0400 | Verify installation by running `uv run sphinx-build --version` | | |

#### Phase 1 Results

| Metric | Expected | Actual | Status |
|--------|----------|--------|--------|
| sphinx-build version output | Version 8.x or higher | | |

---

### Phase 2: Create Documentation Directory Structure

> **IMPORTANT:** Before implementing this phase, review the Constraints and Decisions sections for applicable items.

- **GOAL-200**: Create the directory structure for documentation source files

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-0500 | Create directory `<project-root>/doc/` | | |
| TASK-0600 | Create directory `<project-root>/doc/source/` | | |
| TASK-0700 | Create directory `<project-root>/doc/source/_static/` (can be empty, but must exist) | | |
| TASK-0800 | Create directory `<project-root>/doc/build/` (will hold generated output) | | |
| TASK-0900 | Verify structure exists: `doc/`, `doc/source/`, `doc/source/_static/`, `doc/build/` | | |

#### Phase 2 Results

| Metric | Expected | Actual | Status |
|--------|----------|--------|--------|
| Directory structure created | All 4 directories exist | | |

---

### Phase 3: Create Sphinx Configuration

> **IMPORTANT:** Before implementing this phase, review the Constraints and Decisions sections for applicable items. See DEC-0200, DEC-0500, DEC-0600, DEC-0700, DEC-1400, DEC-1500, DEC-1600.

- **GOAL-300**: Create the Sphinx configuration file with all required settings

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-1000 | Create `<project-root>/doc/source/conf.py` with project information section: `project`, `copyright`, `author`, `release` variables set to appropriate placeholder or actual values | | |
| TASK-1100 | Add extensions list to `conf.py`: `autodoc2`, `myst_parser`, `sphinxcontrib.mermaid`, `sphinx.ext.napoleon` | | |
| TASK-1200 | Add autodoc2 configuration to `conf.py`: `autodoc2_packages` with path `"../../<package-name>"` (relative to `doc/source/`), and `autodoc2_render_plugin = "myst"` | | |
| TASK-1300 | Add source suffix configuration to `conf.py`: `source_suffix = {'.rst': 'restructuredtext', '.md': 'markdown'}` | | |
| TASK-1400 | Add HTML output configuration to `conf.py`: `html_theme = "pydata_sphinx_theme"`, `html_static_path = ['_static']`, `html_title = "<package-name>"` | | |
| TASK-1500 | Add MyST extensions configuration to `conf.py`: `myst_enable_extensions` list containing `amsmath`, `colon_fence`, `deflist`, `html_admonition`, `html_image`, `strikethrough`, `dollarmath`, `attrs_block`, `attrs_inline`, `substitution`, `fieldlist`, `replacements`, `linkify` | | |
| TASK-1600 | Add MyST heading anchors configuration: `myst_heading_anchors = 3` | | |
| TASK-1700 | Verify `conf.py` has no syntax errors by running `uv run python -m py_compile doc/source/conf.py` | | |

#### Phase 3 Results

| Metric | Expected | Actual | Status |
|--------|----------|--------|--------|
| conf.py syntax check | Exit code 0, no output | | |

---

### Phase 4: Create Starter Documentation Pages

> **IMPORTANT:** Before implementing this phase, review the Constraints and Decisions sections for applicable items. See DEC-1100.

- **GOAL-400**: Create minimal documentation pages to verify the setup works

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-1800 | Create `<project-root>/doc/source/index.md` with a title, brief project description, and a toctree directive including `api` | | |
| TASK-1900 | Create `<project-root>/doc/source/api.md` with a title, brief description, and a toctree directive pointing to `apidocs/index` | | |
| TASK-2000 | Verify both files exist and have valid Markdown syntax | | |

#### Suggested Additional Pages

The following pages are commonly useful but not required for the initial setup. Add them to the toctree in `index.md` as needed:

- `usage.md` — How to install and use the package
- `changelog.md` — Version history and release notes
- `examples.md` — Code examples and tutorials
- `faq.md` — Frequently asked questions
- `contributing.md` — Guidelines for contributors
- `api.md` — API reference (already included)

#### Phase 4 Results

| Metric | Expected | Actual | Status |
|--------|----------|--------|--------|
| index.md exists | File present with toctree | | |
| api.md exists | File present with apidocs reference | | |

---

### Phase 5: Create Build Infrastructure

> **IMPORTANT:** Before implementing this phase, review the Constraints and Decisions sections for applicable items. See DEC-0400, DEC-0800.

- **GOAL-500**: Create Makefiles for building documentation

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-2100 | Create `<project-root>/doc/Makefile` with variables: `SPHINXBUILD = sphinx-build`, `SOURCEDIR = source`, `BUILDDIR = build` | | |
| TASK-2200 | Add `html` target to `doc/Makefile`: runs `uv run $(SPHINXBUILD) -b html $(SOURCEDIR) $(BUILDDIR)/html` | | |
| TASK-2300 | Add `man` target to `doc/Makefile`: runs `uv run $(SPHINXBUILD) -b man $(SOURCEDIR) $(BUILDDIR)/man` | | |
| TASK-2400 | Add `singlehtml` target to `doc/Makefile`: runs `uv run $(SPHINXBUILD) -b singlehtml $(SOURCEDIR) $(BUILDDIR)/singlehtml` | | |
| TASK-2500 | Add `markdown` target to `doc/Makefile`: runs `uv run $(SPHINXBUILD) -b markdown $(SOURCEDIR) $(BUILDDIR)/markdown` | | |
| TASK-2600 | Add `all` target to `doc/Makefile` that depends on `html`, `man`, `singlehtml`, `markdown` | | |
| TASK-2700 | Add `clean` target to `doc/Makefile`: runs `rm -rf $(BUILDDIR)/*` | | |
| TASK-2800 | Add `help` target to `doc/Makefile` that prints available targets | | |
| TASK-2900 | Add `.PHONY` declaration for all targets | | |
| TASK-3000 | Create `<project-root>/Makefile` (or modify existing) with delegation targets: `html`, `man`, `singlehtml`, `markdown`, `cleandoc` that run `$(MAKE) -C doc <target>` | | |
| TASK-3100 | Verify `doc/Makefile` syntax by running `make -C doc -n html` (dry run) | | |

#### Phase 5 Results

| Metric | Expected | Actual | Status |
|--------|----------|--------|--------|
| doc/Makefile dry run | No errors, shows sphinx-build command | | |

---

### Phase 6: Configure Version Control Ignores

> **IMPORTANT:** Before implementing this phase, review the Constraints and Decisions sections for applicable items. See DEC-1200.

- **GOAL-600**: Ensure generated files are not committed to version control

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-3200 | Add `doc/build/` to `<project-root>/.gitignore` (create file if it doesn't exist) | | |

#### Phase 6 Results

| Metric | Expected | Actual | Status |
|--------|----------|--------|--------|
| doc/build/ ignored | Not shown in git status after build | | |

---

### Phase 7: Verify Complete Setup

> **IMPORTANT:** Before implementing this phase, review the Constraints and Decisions sections for applicable items. See DEC-1300, DEC-1700, DEC-1800.

- **GOAL-700**: Confirm the documentation builds successfully

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-3500 | Run `make html` from project root (or `make -C doc html` from any directory) | | |
| TASK-3600 | Verify HTML output exists at `doc/build/html/index.html` | | |
| TASK-3700 | Open `doc/build/html/index.html` in a browser and verify it displays correctly | | |
| TASK-3800 | Verify API documentation was generated at `doc/build/html/apidocs/index.html` | | |
| TASK-3850 | Create `<project-root>/doc/source/apidocs/.gitignore` containing a single line: `*` | | |
| TASK-3860 | Verify gitignore entries are correct by running `git status` (build artifacts should not appear as untracked) | | |
| TASK-3900 | Run `make -C doc man` and verify output at `doc/build/man/` | | |
| TASK-4000 | Run `make -C doc clean` and verify `doc/build/` contents are removed | | |

#### Phase 7 Results

| Metric | Expected | Actual | Status |
|--------|----------|--------|--------|
| HTML build succeeds | Exit code 0, html/index.html exists | | |
| API docs generated | apidocs/index.html exists | | |
| Man build succeeds | Exit code 0, man page files exist | | |
| Clean works | doc/build/ is empty | | |
| Gitignore working | doc/build/ and apidocs/ not shown in git status | | |

---

## 3. Alternatives

- **ALT-100**: Use reStructuredText instead of Markdown — rejected because Markdown is more widely known and easier to write; MyST Parser provides full Sphinx functionality with Markdown syntax
- **ALT-200**: Use `sphinx.ext.autodoc` instead of `autodoc2` — rejected because autodoc2 works better with MyST/Markdown output and doesn't require importing modules at build time
- **ALT-300**: Use a single Makefile at project root — rejected because separating `doc/Makefile` keeps documentation build logic contained and matches common project patterns
- **ALT-400**: Use Sphinx's built-in quickstart — rejected because it generates reStructuredText and requires manual conversion to match this plan's MyST-based approach

## 4. Dependencies

- **DEP-100**: Python 3.12 or higher (per typical uv project requirements)
- **DEP-200**: uv package manager installed and configured
- **DEP-300**: The target project must have at least one Python module with docstrings for API documentation to be meaningful
- **DEP-400**: GNU Make (for Makefile-based builds)

## 5. Files

### Files to Create

| File | Description |
|------|-------------|
| FILE-0100 | `doc/source/conf.py` — Sphinx configuration |
| FILE-0200 | `doc/source/index.md` — Documentation landing page |
| FILE-0300 | `doc/source/api.md` — API documentation entry point |
| FILE-0400 | `doc/source/_static/` — Static assets directory (empty initially) |
| FILE-0500 | `doc/source/apidocs/.gitignore` — Ignore auto-generated API files |
| FILE-0600 | `doc/Makefile` — Documentation build targets |
| FILE-0700 | `doc/build/` — Build output directory (created by build process) |

### Files to Modify

| File | Description |
|------|-------------|
| FILE-0800 | `pyproject.toml` — Add documentation dependencies |
| FILE-0900 | `.gitignore` — Add `doc/build/` entry |
| FILE-1000 | `Makefile` (root) — Add documentation delegation targets (create if doesn't exist) |

## 6. Testing / Verification

- **TEST-100**: `uv run sphinx-build --version` returns version 8.x or higher
- **TEST-200**: `uv run python -m py_compile doc/source/conf.py` exits with code 0
- **TEST-300**: `make -C doc html` completes without errors
- **TEST-400**: `doc/build/html/index.html` exists and renders in browser
- **TEST-500**: `doc/build/html/apidocs/index.html` exists (API docs generated)
- **TEST-600**: `make -C doc man` completes without errors
- **TEST-700**: `make -C doc clean` removes build artifacts
- **TEST-800**: `git status` does not show `doc/build/` or `doc/source/apidocs/` as untracked after building

## 7. Risks & Assumptions

### Risks

- **RISK-0100**: If the Python package has no docstrings, autodoc2 will generate minimal/empty API documentation
- **RISK-0200**: If the package has import-time side effects, autodoc2 may exhibit unexpected behavior (though it's designed to avoid importing)
- **RISK-0300**: MyST extension `linkify` requires `linkify-it-py`; if not installed, Sphinx will error on build
- **RISK-0400**: The `pydata-sphinx-theme` may have version-specific configuration; check theme documentation if styling issues occur
- **RISK-0500**: If the project uses a `src/` layout, the autodoc2 path must be adjusted (see GUD-0400)

### Assumptions

- **ASSUMPTION-100**: The target project uses uv and has a working `pyproject.toml`
- **ASSUMPTION-200**: The target project uses a flat package layout (`<project-root>/<package-name>/`)
- **ASSUMPTION-300**: GNU Make is available on the system
- **ASSUMPTION-400**: The implementer will replace all `<placeholder>` values with actual project-specific values
- **ASSUMPTION-500**: The target Python package exists and has at least basic docstrings

## 8. Optional Enhancement: Executable Code Examples (markdown-code-runner)

> **Note:** This section describes an optional enhancement. The core documentation setup works without it.

Per DEC-0300, executable code examples via `markdown-code-runner` are documented here as an optional addition.

### Overview

`markdown-code-runner` allows Python code blocks in Markdown files to be executed during the build process, with their output inserted into the documentation. This ensures code examples are always accurate and tested.

### Additional Dependencies

Add to the docs dependency group in `pyproject.toml`:
- `markdown-code-runner>=2.0`

### Preprocessing Pipeline Changes

Enabling markdown-code-runner requires a preprocessing step:

1. Copy source files to a temporary directory (`doc/build/processed_source/`)
2. Run `markdown-code-runner` on each Markdown file
3. Run a cleanup script to format the output
4. Run Sphinx with the processed directory as source

### Path Adjustment Required

When preprocessing is enabled, Sphinx runs from `doc/build/processed_source/` instead of `doc/source/`. The autodoc2 package path must change:

| Setup | autodoc2_packages path |
|-------|------------------------|
| Without preprocessing | `"../../<package-name>"` |
| With preprocessing | `"../../../<package-name>"` |

### Makefile Modifications

The `doc/Makefile` must be modified to:
1. Define `TEMPDIR = $(BUILDDIR)/processed_source`
2. Add `copy_source` target to copy source files to temp directory
3. Add `run_mcr` target to execute markdown-code-runner on Markdown files
4. Add `cleanup_mcr_output` target to format the output markers
5. Modify build targets (`html`, `man`, etc.) to depend on preprocessing and use `$(TEMPDIR)` as source

### Cleanup Script

A Python script is needed to clean up markdown-code-runner's output markers. Create `<project-root>/scripts/cleanup_mcr_output.py` that:
1. Reads Markdown files
2. Finds `<!-- OUTPUT:START -->` and `<!-- OUTPUT:END -->` markers
3. Removes the warning comment and wraps output in code fences
4. Writes the modified content back

### Code Block Syntax

To mark a code block for execution, add `markdown-code-runner` to the language specifier:

~~~markdown
```python markdown-code-runner
print("Hello, world!")
```

Expected output:
<!-- OUTPUT:START -->
<!-- This will be replaced -->
<!-- OUTPUT:END -->
~~~

## 9. Decisions

- **Decision (DEC-0100):** [Q1] The plan is generalizable to any Python project using the uv package manager, rather than targeting a specific project. File paths use placeholder names that the implementer must replace.

- **Decision (DEC-0200):** [Q2] The plan includes autodoc2 for auto-generated API documentation from Python docstrings. The configuration uses `autodoc2_render_plugin = "myst"` for Markdown output.

- **Decision (DEC-0300):** [Q3] The markdown-code-runner feature is documented as an optional enhancement rather than a required part of the setup. The core pipeline works without it.

- **Decision (DEC-0400):** [Q4] The plan includes all output formats from the reference project: HTML, man pages, single-page HTML, and Markdown. The Makefile provides targets for each.

- **Decision (DEC-0500):** [Q5] The plan specifies `pydata-sphinx-theme` as the default HTML theme, with a note that it can be swapped for alternatives by changing `html_theme` and dependencies.

- **Decision (DEC-0600):** [Q6] The plan enables the full set of MyST Parser extensions: `amsmath`, `colon_fence`, `deflist`, `html_admonition`, `html_image`, `strikethrough`, `dollarmath`, `attrs_block`, `attrs_inline`, `substitution`, `fieldlist`, `replacements`, and `linkify`.

- **Decision (DEC-0700):** [Q7] The plan includes Mermaid diagram support via `sphinxcontrib-mermaid` as part of the core setup.

- **Decision (DEC-0800):** [Q8] The plan uses a two-Makefile structure: a root-level Makefile with delegation targets and a `doc/Makefile` that performs the actual Sphinx build.

- **Decision (DEC-0900):** [Q9] The plan assumes `pyproject.toml` exists and specifies dependencies to add. A note about minimum requirements is included for new projects.

- **Decision (DEC-1000):** [Q10] The plan does not include CI/GitHub Actions configuration. It focuses on local documentation build setup.

- **Decision (DEC-1100):** [Q11] The plan creates minimal starter pages (index + api) with a list of suggested additional pages for the implementer to add as needed.

- **Decision (DEC-1200):** [Q12] The plan includes adding `doc/build/` to the root `.gitignore` file.

- **Decision (DEC-1300):** [Q13] The plan gitignores `doc/source/apidocs/` by creating a `.gitignore` file inside it containing `*`.

- **Decision (DEC-1400):** [Q14] The plan includes `sphinx.ext.napoleon` for Google/NumPy-style docstring support.

- **Decision (DEC-1500):** [Q15] The plan documents both autodoc2 package paths: `"../../<package-name>"` for standard setup, and `"../../../<package-name>"` for setups with markdown-code-runner preprocessing.

- **Decision (DEC-1600):** [Q16] The plan assumes a flat package layout (`<project-root>/<package-name>/`). Implementers using src-layout must adjust paths accordingly.

- **Decision (DEC-1700):** [Q17] Move TASK-3300 (create apidocs/.gitignore) to Phase 7, after the first build, so that the apidocs/ directory exists before attempting to create the .gitignore inside it.

- **Decision (DEC-1800):** [Q18] Move TASK-3400 (verify gitignore) to Phase 7, after the build tasks, so that gitignore verification happens after the build it depends on.

## 10. Related Specifications

- [Sphinx Documentation](https://www.sphinx-doc.org/en/master/)
- [MyST Parser Documentation](https://myst-parser.readthedocs.io/en/latest/)
- [sphinx-autodoc2 Documentation](https://sphinx-autodoc2.readthedocs.io/en/latest/)
- [pydata-sphinx-theme Documentation](https://pydata-sphinx-theme.readthedocs.io/en/stable/)
- [sphinxcontrib-mermaid Documentation](https://sphinxcontrib-mermaid-demo.readthedocs.io/en/latest/)
- [markdown-code-runner Documentation](https://github.com/basnijholt/markdown-code-runner)
