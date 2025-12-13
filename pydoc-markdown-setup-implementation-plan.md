---
goal: Set up pydoc-markdown to generate Markdown API documentation for a Python project using uv
version: 1.0.0
date_created: 2025-12-12
last_updated: 2025-12-12
owner: Development Team
status: 'Active'
tags: [documentation, pydoc-markdown, markdown, python, uv, api-docs]
---

# Pydoc-Markdown Documentation Setup Plan

![Status: Active](https://img.shields.io/badge/status-Active-green)

This plan provides step-by-step instructions for setting up pydoc-markdown to generate Markdown-format API documentation from Python docstrings for any project that uses the `uv` package manager. The output is pure Markdown files (not HTML), suitable for reading directly, including in repositories, or as input to other documentation systems.

## 1. Requirements & Constraints

### Requirements

- **REQ-100**: API documentation must be auto-generated from Python docstrings
- **REQ-200**: Documentation output must be in Markdown format (not HTML)
- **REQ-300**: Build commands must use `uv run` to ensure correct virtual environment
- **REQ-400**: The setup must work with any uv-managed Python project
- **REQ-500**: A `make docs` command must be available from the project root
- **REQ-600**: Generated documentation must be organized as one file per module

### Constraints

- **CON-100**: The target project must have a `pyproject.toml` file (required for uv)
- **CON-200**: This plan supports flat package layout (`<project-root>/<package-name>/`); src-layout projects must adjust paths as noted
- **CON-300**: This plan does not include CI/GitHub Actions configuration
- **CON-400**: This plan does not include HTML generation or static site integration
- **CON-500**: pydoc-markdown is semi-maintained; the author recommends mkdocstrings for MkDocs-based HTML documentation

### Guidelines

- **GUD-100**: All paths in this plan use placeholders: `<project-root>` for the project directory, `<package-name>` for the Python package name
- **GUD-200**: Replace placeholder values (marked with `<...>`) with actual project-specific values during implementation
- **GUD-300**: The configuration file is placed in `conf/` to group configuration files together; adjust if your project uses a different convention
- **GUD-400**: If using a `src/` layout, adjust the `search_path` option from `[.]` to `[src]` in the loader configuration

## 2. Implementation Steps

> **Before you begin:** Ensure your git working directory is clean so you can easily revert if needed:
> ```bash
> git status        # Should show no uncommitted changes
> git stash         # If needed, stash current work
> ```
> If something goes wrong, you can revert all changes with `git checkout -- .` or `git stash pop` to restore your stashed work.

### Prerequisites Checklist

Before starting implementation, verify these prerequisites:

- [ ] `uv` is installed: `uv --version`
- [ ] Project has `pyproject.toml`: `ls pyproject.toml`
- [ ] Project dependencies are synced: `uv sync`
- [ ] Python package exists: `ls <package-name>/`
- [ ] GNU Make is available: `make --version`
- [ ] Git repository is initialized: `git status`

### Discover Your Modules

Before configuring pydoc-markdown, enumerate the Python modules in your package:

```bash
ls <package-name>/*.py | sed 's|.*/||; s|\.py$||' | grep -v '^__'
```

This will output module names like:
```
clients
config
database
graph
...
```

Record this list for use in Phase 3 (Configuration).

---

### Phase 1: Add Documentation Dependencies

> **IMPORTANT:** Before implementing this phase, review the Constraints and Decisions sections for applicable items.

- **GOAL-100**: Add pydoc-markdown as a documentation dependency

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-0100 | Run `uv add --group docs pydoc-markdown` to add the documentation dependency | | |
| TASK-0200 | Verify the `docs` group was added to `pyproject.toml` in the `[dependency-groups]` section | | |
| TASK-0300 | (Optional) If dependencies weren't installed automatically, run `uv sync --group docs` | | |
| TASK-0400 | Verify installation by running `uv run pydoc-markdown --version` | | |

#### Dependency Group Configuration

Run this command to add the documentation dependency:

```bash
uv add --group docs pydoc-markdown
```

This will add the following to `pyproject.toml`:

```toml
[dependency-groups]
docs = [
    "pydoc-markdown>=4.8",
]
```

**Note:** Always use `uv add` to modify `pyproject.toml` dependencies rather than editing the file manually. This ensures the lockfile stays in sync and dependencies are properly resolved.

#### Phase 1 Results

| Metric | Expected | Actual | Status |
|--------|----------|--------|--------|
| pydoc-markdown version output | Version 4.8.x or higher | | |

---

### Phase 2: Create Documentation Directory Structure

> **IMPORTANT:** Before implementing this phase, review the Constraints and Decisions sections for applicable items. See DEC-0100.

- **GOAL-200**: Create the directory structure for generated documentation

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-0500 | Create directory `<project-root>/doc/api/` | | |
| TASK-0600 | Verify structure exists: `ls -la doc/api/` | | |

Create the output directory:

```bash
mkdir -p doc/api
```

> **Note:** The `doc/api/` directory will be added to `.gitignore` in Phase 5. Avoid running `git add .` or committing until Phase 5 is complete, or add `doc/api/` to `.gitignore` now if you prefer.

#### Phase 2 Results

| Metric | Expected | Actual | Status |
|--------|----------|--------|--------|
| Directory structure created | `doc/api/` exists | | |

---

### Phase 3: Create Pydoc-Markdown Configuration

> **IMPORTANT:** Before implementing this phase, review the Constraints and Decisions sections for applicable items. See DEC-0200, DEC-0300, DEC-0400, DEC-0500, DEC-0600.

- **GOAL-300**: Create the pydoc-markdown configuration file with all required settings

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-0700 | Create `<project-root>/conf/pydoc-markdown.yml` with loader configuration | | |
| TASK-0800 | Add processor configuration for filtering and docstring processing | | |
| TASK-0900 | Add renderer configuration with per-module output files | | |
| TASK-1000 | Verify YAML syntax is valid | | |

#### Configuration Template

Create `<project-root>/conf/pydoc-markdown.yml`:

```yaml
# Pydoc-Markdown Configuration for <package-name>
# Documentation: https://niklasrosenstein.github.io/pydoc-markdown/
#
# Usage: uv run pydoc-markdown -c conf/pydoc-markdown.yml

# === Loaders ===
# Loaders extract API information from Python source files.
# The Python loader uses static analysis (via docspec) - it does not import your code.
loaders:
  - type: python
    search_path:
      - .  # Project root; change to 'src' for src-layout projects
    packages:
      - <package-name>

# === Processors ===
# Processors transform the extracted API information before rendering.
processors:
  # Filter out private members and undocumented items
  - type: filter
    expression: not name.startswith('_') and default()
    documented_only: true
    skip_empty_modules: true

  # Convert Google-style docstrings to Markdown
  - type: google

  # Smart processor for additional docstring cleanup
  - type: smart

  # Cross-reference processor for linking between documented objects
  - type: crossref

# === Renderer ===
# The Markdown renderer outputs plain Markdown files.
renderer:
  type: markdown
  render_module_header: true
  render_toc: true
  toc_maxdepth: 3
  descriptive_class_title: true
  descriptive_module_title: true
  add_module_prefix: false
  add_member_class_prefix: false
  filename: doc/api/{module}.md

# === Hooks (Optional) ===
# Commands to run before or after rendering.
# hooks:
#   pre-render:
#     - echo "Starting documentation generation..."
#   post-render:
#     - echo "Documentation generated in doc/api/"
```

#### Understanding the Configuration

**Loaders Section:**
- `search_path: [.]` - Tells pydoc-markdown where to find Python packages (project root for flat layout, `src` for src-layout)
- `packages: [<package-name>]` - Specifies which packages to document

**Processors Section:**
- `filter` - Excludes private members (`_`-prefixed) and undocumented items
- `google` - Parses Google-style docstrings (Args, Returns, Raises, etc.)
- `smart` - Additional cleanup and formatting
- `crossref` - Enables cross-references between documented objects using `#ClassName` syntax

**Renderer Section:**
- `type: markdown` - Outputs plain Markdown (not HTML)
- `render_toc: true` - Includes a table of contents in each file
- `filename: doc/api/{module}.md` - Output pattern; `{module}` is replaced with the module name

#### Phase 3 Results

| Metric | Expected | Actual | Status |
|--------|----------|--------|--------|
| conf/pydoc-markdown.yml exists | File present with valid YAML | | |
| YAML syntax valid | `python -c "import yaml; yaml.safe_load(open('conf/pydoc-markdown.yml'))"` succeeds | | |

---

### Phase 4: Add Makefile Targets

> **IMPORTANT:** Before implementing this phase, review the Constraints and Decisions sections for applicable items. See DEC-0700.

- **GOAL-400**: Add documentation build targets to the project Makefile

**Choose ONE of the following paths based on whether your project already has a Makefile:**

#### Path A: Create New Makefile (No existing Makefile)

If your project does not have a Makefile, use this path.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-1100 | Verify no Makefile exists: `ls Makefile` should return "No such file" | | |
| TASK-1200 | Create `<project-root>/Makefile` using the complete template below | | |
| TASK-1300 | Verify the Makefile works: `make` should display available commands | | |

**Complete Makefile Template:**

Create `<project-root>/Makefile` with this content:

```makefile
.PHONY: default docs docs-clean uv-sync-group-docs uv-sync-all-groups

default:
	@echo "Available commands:"
	@echo "  make docs                 Generate Markdown API documentation to doc/api/."
	@echo "  make docs-clean           Remove generated documentation."
	@echo "  make uv-sync-group-docs   Install documentation dependencies only."
	@echo "  make uv-sync-all-groups   Install all dependency groups."

# --- Documentation ---
docs:
	@echo "Generating API documentation..."
	uv run pydoc-markdown -c conf/pydoc-markdown.yml
	@echo "Documentation generated in doc/api/"

docs-clean:
	@echo "Cleaning generated documentation..."
	rm -rf doc/api/*.md
	@echo "Cleaned doc/api/"

# --- Dependency Management ---
uv-sync-group-docs:
	@echo "Installing documentation dependencies..."
	uv sync --group docs

uv-sync-all-groups:
	@echo "Installing all dependency groups..."
	uv sync --all-groups
```

**After completing Path A, skip to Phase 4 Results below.**

---

#### Path B: Modify Existing Makefile

If your project already has a Makefile, use this path to add the documentation targets.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-1400 | Open `<project-root>/Makefile` and locate the `.PHONY` declaration | | |
| TASK-1500 | Add `docs`, `docs-clean`, `uv-sync-group-docs`, and `uv-sync-all-groups` to `.PHONY` | | |
| TASK-1600 | Add `docs` target that runs `uv run pydoc-markdown -c conf/pydoc-markdown.yml` | | |
| TASK-1700 | Add `docs-clean` target that removes the `doc/api/*.md` files | | |
| TASK-1800 | Add `uv-sync-group-docs` target that runs `uv sync --group docs` | | |
| TASK-1900 | Add `uv-sync-all-groups` target that runs `uv sync --all-groups` | | |
| TASK-2000 | Update the `default` target help text to include documentation commands | | |

**Makefile Additions:**

Add the following targets to your existing `<project-root>/Makefile`:

```makefile
# --- Documentation ---
docs:
	@echo "Generating API documentation..."
	uv run pydoc-markdown -c conf/pydoc-markdown.yml
	@echo "Documentation generated in doc/api/"

docs-clean:
	@echo "Cleaning generated documentation..."
	rm -rf doc/api/*.md
	@echo "Cleaned doc/api/"

# --- Dependency Management ---
uv-sync-group-docs:
	@echo "Installing documentation dependencies..."
	uv sync --group docs

uv-sync-all-groups:
	@echo "Installing all dependency groups..."
	uv sync --all-groups
```

Update the `.PHONY` declaration to include the new targets:

```makefile
.PHONY: default ruff mypy test ... docs docs-clean uv-sync-group-docs uv-sync-all-groups
```

Update the `default` target to show the new commands:

```makefile
	@echo "  make docs                 Generate Markdown API documentation to doc/api/."
	@echo "  make docs-clean           Remove generated documentation."
	@echo "  make uv-sync-group-docs   Install documentation dependencies only."
	@echo "  make uv-sync-all-groups   Install all dependency groups."
```

---

#### Phase 4 Results

| Metric | Expected | Actual | Status |
|--------|----------|--------|--------|
| make docs -n | Shows pydoc-markdown command | | |
| make docs-clean -n | Shows rm command | | |

---

### Phase 5: Configure Version Control Ignores

> **IMPORTANT:** Before implementing this phase, review the Constraints and Decisions sections for applicable items. See DEC-0800.

- **GOAL-500**: Ensure generated files are not committed to version control

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-2100 | Add `doc/api/` to `<project-root>/.gitignore` | | |
| TASK-2200 | Verify gitignore entry is correct by running `git status` after a build | | |

#### Gitignore Entries

Add the following to `<project-root>/.gitignore`:

```gitignore
# Pydoc-markdown generated documentation
doc/api/
```

#### Phase 5 Results

| Metric | Expected | Actual | Status |
|--------|----------|--------|--------|
| doc/api/ ignored | Not shown in git status after build | | |

---

### Phase 6: Verify Complete Setup

> **IMPORTANT:** Before implementing this phase, review the Constraints and Decisions sections for applicable items.

- **GOAL-600**: Confirm the documentation generates successfully

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-2300 | Run `make docs` from project root | | |
| TASK-2400 | Verify Markdown files exist in `doc/api/` | | |
| TASK-2500 | Open a generated file and verify docstrings are rendered correctly | | |
| TASK-2600 | Verify table of contents is present at the top of each file | | |
| TASK-2700 | Verify cross-references are formatted (e.g., `#ClassName` links) | | |
| TASK-2800 | Run `make docs-clean` and verify `doc/api/` is empty | | |
| TASK-2900 | Run `git status` and verify `doc/api/` is not shown as untracked | | |

#### Phase 6 Results

| Metric | Expected | Actual | Status |
|--------|----------|--------|--------|
| Markdown files generated | `.md` files in doc/api/ for each module | | |
| Content correct | Docstrings visible with proper formatting | | |
| TOC present | Table of contents at top of each file | | |
| Clean works | doc/api/ is empty after clean | | |
| Gitignore working | doc/api/ not shown in git status | | |

---

## 3. Alternatives

- **ALT-100**: Use mkdocstrings instead of pydoc-markdown - rejected because the user specifically requested Markdown-only output without HTML generation; mkdocstrings is designed for MkDocs HTML output
- **ALT-200**: Use Sphinx with sphinx-markdown-builder - rejected because Sphinx is more complex to configure and primarily designed for reStructuredText
- **ALT-300**: Use pdoc or pdoc3 - rejected because pydoc-markdown provides more flexible configuration and better Google-style docstring support
- **ALT-400**: Place configuration in `pyproject.toml` - rejected because YAML format is more readable for pydoc-markdown's nested configuration
- **ALT-500**: Place configuration in project root - rejected because grouping config files in `conf/` matches existing project conventions
- **ALT-600**: Generate single combined Markdown file - rejected because per-module files are more navigable for packages with multiple modules
- **ALT-700**: Include private members in documentation - rejected because public API documentation is cleaner and more useful for users

## 4. Dependencies

- **DEP-100**: Python 3.8 or higher (pydoc-markdown requirement; most uv projects use 3.12+)
- **DEP-200**: uv package manager installed and configured
- **DEP-300**: The target project must have at least one Python module with docstrings for API documentation to be meaningful
- **DEP-400**: GNU Make (for Makefile-based builds)

## 5. Files

### Files to Create

| File | Description |
|------|-------------|
| FILE-100 | `conf/pydoc-markdown.yml` - Pydoc-markdown configuration |
| FILE-200 | `doc/api/` - Output directory for generated Markdown (created by build) |

### Files to Modify

| File | Description |
|------|-------------|
| FILE-300 | `pyproject.toml` - Add documentation dependencies |
| FILE-400 | `.gitignore` - Add `doc/api/` entry |
| FILE-500 | `Makefile` - Add documentation targets |

## 6. Testing / Verification

- **TEST-100**: `uv run pydoc-markdown --version` returns version 4.8.x or higher
- **TEST-200**: `uv run pydoc-markdown -c conf/pydoc-markdown.yml` completes without errors
- **TEST-300**: Markdown files exist in `doc/api/` after running `make docs`
- **TEST-400**: Generated Markdown contains properly formatted docstrings
- **TEST-500**: Table of contents is present in generated files
- **TEST-600**: Private members (starting with `_`) are not included in output
- **TEST-700**: Undocumented functions/classes are not included in output
- **TEST-800**: `make docs` and `make docs-clean` work correctly
- **TEST-900**: `git status` does not show `doc/api/` as untracked after building

## 7. Risks & Assumptions

### Risks

- **RISK-100**: pydoc-markdown is semi-maintained; the author recommends mkdocstrings for actively maintained alternative (if HTML output is acceptable)
- **RISK-200**: If the Python package has no docstrings, pydoc-markdown will generate minimal/empty documentation
- **RISK-300**: Complex or non-standard docstring formats may not parse correctly; verify output for edge cases
- **RISK-400**: If the project uses a `src/` layout, the `search_path` option must be adjusted to `[src]`
- **RISK-500**: Very large packages may cause slow generation times; consider documenting specific modules if needed
- **RISK-600**: The `crossref` processor may not resolve all references correctly; verify links in output

### Assumptions

- **ASSUMPTION-100**: The target project uses uv and has a working `pyproject.toml`
- **ASSUMPTION-200**: The target project uses a flat package layout (`<project-root>/<package-name>/`)
- **ASSUMPTION-300**: GNU Make is available on the system
- **ASSUMPTION-400**: The implementer will replace all `<placeholder>` values with actual project-specific values
- **ASSUMPTION-500**: The target Python package exists and has at least basic docstrings
- **ASSUMPTION-600**: The project uses Google-style docstrings; adjust processor configuration if using NumPy or Sphinx style

## 8. Optional Enhancements

### 8.1 Watch Mode for Automatic Regeneration

For automatic regeneration when Python files change, use a watch tool like `watchexec` or `entr`:

```bash
# Using watchexec (install: cargo install watchexec-cli)
watchexec -e py -- make docs

# Using entr (install: apt install entr / brew install entr)
find <package-name> -name '*.py' | entr make docs
```

Add as a Makefile target if desired:

```makefile
docs-watch:
	@echo "Watching for changes..."
	find <package-name> -name '*.py' | entr make docs
```

### 8.2 NumPy-Style Docstrings

If your project uses NumPy-style docstrings instead of Google-style, replace the `google` processor with `numpy`:

```yaml
processors:
  - type: filter
    expression: not name.startswith('_') and default()
    documented_only: true
    skip_empty_modules: true
  - type: numpy  # Changed from 'google'
  - type: smart
  - type: crossref
```

### 8.3 Sphinx-Style Docstrings

If your project uses Sphinx/reST-style docstrings, replace the `google` processor with `sphinx`:

```yaml
processors:
  - type: filter
    expression: not name.startswith('_') and default()
    documented_only: true
    skip_empty_modules: true
  - type: sphinx  # Changed from 'google'
  - type: smart
  - type: crossref
```

### 8.4 Include Private Members

To include private members (single underscore `_`) but exclude dunder methods (`__`):

```yaml
processors:
  - type: filter
    expression: not name.startswith('__') and default()
    documented_only: true
    skip_empty_modules: true
```

### 8.5 Document Specific Modules Only

To document only specific modules instead of the entire package:

```yaml
loaders:
  - type: python
    search_path: [.]
    modules:
      - <package-name>.clients
      - <package-name>.database
      - <package-name>.graph
```

### 8.6 Custom Output Filename Pattern

To customize the output filename pattern:

```yaml
renderer:
  type: markdown
  filename: doc/api/<package-name>_{module}.md  # Prefix with package name
```

Or for a flat structure without module hierarchy:

```yaml
renderer:
  type: markdown
  filename: doc/api/{module_name}.md  # Just the module name, no package prefix
```

### 8.7 Source Code Links

To add links to source code on GitHub:

```yaml
renderer:
  type: markdown
  source_linker:
    type: github
    repo: <username>/<repo-name>
    root: .
```

### 8.8 Testing Configuration

To test your configuration and see what pydoc-markdown discovers:

```bash
# Dump discovered modules and objects
uv run pydoc-markdown -c conf/pydoc-markdown.yml --dump | head -50

# Full tree view (requires docspec)
uv run pydoc-markdown -c conf/pydoc-markdown.yml --dump | uv run docspec -m --dump-tree
```

## 9. Decisions

- **Decision (DEC-0100):** Generated documentation is output to `doc/api/`, keeping documentation organized under the `doc/` directory.

- **Decision (DEC-0200):** The configuration file is placed at `conf/pydoc-markdown.yml` to group configuration files together, matching project conventions.

- **Decision (DEC-0300):** The plan uses pydoc-markdown's YAML configuration format for better readability and preprocessing support.

- **Decision (DEC-0400):** The plan uses the `markdown` renderer for plain Markdown output, not the `mkdocs` or `hugo` renderers which produce HTML.

- **Decision (DEC-0500):** The plan configures one output file per module using the `filename: doc/api/{module}.md` pattern.

- **Decision (DEC-0600):** The plan uses Google-style docstring processor. Projects using NumPy or Sphinx style should change the processor accordingly.

- **Decision (DEC-0700):** The plan adds two Makefile targets: `docs` (generate) and `docs-clean` (remove output). No live preview is needed for Markdown output.

- **Decision (DEC-0800):** The plan includes adding `doc/api/` to `.gitignore` to prevent committing generated artifacts.

- **Decision (DEC-0900):** Private members (names starting with `_`) are excluded from documentation via the `filter` processor.

- **Decision (DEC-1000):** Undocumented items (functions/classes without docstrings) are excluded via `documented_only: true`.

- **Decision (DEC-1100):** The plan does not include CI/GitHub Actions configuration, focusing on local documentation generation.

## 10. Comparison with mkdocstrings

| Aspect | pydoc-markdown | mkdocstrings |
|--------|----------------|--------------|
| Primary output | Markdown files | HTML (via MkDocs) |
| Configuration file | `pydoc-markdown.yml` (YAML) | `mkdocs.yml` (YAML) |
| Parsing method | Static (docspec, no imports) | Runtime (imports modules) |
| Docstring styles | Google, NumPy, Sphinx, Pydoc-Markdown | Google, NumPy, Sphinx |
| Live preview | Not applicable (Markdown output) | Built-in (`mkdocs serve`) |
| Maintenance status | Semi-maintained | Actively maintained |
| Best for | Markdown-only documentation | HTML documentation sites |
| Output location | Configurable (e.g., `doc/api/`) | `site/` or configurable |
| Dependencies | Only pydoc-markdown | MkDocs + theme + mkdocstrings |

## 11. Related Specifications

- [Pydoc-Markdown Documentation](https://niklasrosenstein.github.io/pydoc-markdown/)
- [Pydoc-Markdown GitHub](https://github.com/NiklasRosenstein/pydoc-markdown)
- [Pydoc-Markdown YAML Configuration](https://niklasrosenstein.github.io/pydoc-markdown/usage/yaml/)
- [docspec Documentation](https://niklasrosenstein.github.io/python-docspec/) (used by pydoc-markdown for Python parsing)
- [Google Python Style Guide - Docstrings](https://google.github.io/styleguide/pyguide.html#38-comments-and-docstrings)

## 12. Troubleshooting

### Common Issues

#### "No module named X" or empty output

pydoc-markdown uses static analysis (docspec), so it doesn't actually import your code. However, the `search_path` must be correct:

1. Verify `search_path` points to the directory containing your package
2. For flat layout: `search_path: [.]`
3. For src layout: `search_path: [src]`
4. Check that `packages` or `modules` lists the correct package name

#### Configuration file not found

pydoc-markdown looks for config in the current directory by default. When using `conf/pydoc-markdown.yml`, always specify the path:

```bash
uv run pydoc-markdown -c conf/pydoc-markdown.yml
```

#### Docstrings not formatted correctly

Check which docstring style you're using and configure the appropriate processor:

- Google style: `- type: google`
- NumPy style: `- type: numpy`
- Sphinx style: `- type: sphinx`

#### Private members appearing in output

Verify the `filter` processor is configured correctly:

```yaml
- type: filter
  expression: not name.startswith('_') and default()
```

#### Empty modules in output

Enable `skip_empty_modules` in the filter processor:

```yaml
- type: filter
  skip_empty_modules: true
```

#### Cross-references not working

The `crossref` processor must be included after the docstring processor:

```yaml
processors:
  - type: filter
  - type: google
  - type: smart
  - type: crossref  # Must come after docstring processor
```

#### YAML syntax errors

Validate your YAML:

```bash
python -c "import yaml; yaml.safe_load(open('conf/pydoc-markdown.yml'))"
```

### Testing the Configuration

To see what pydoc-markdown discovers without generating output:

```bash
uv run pydoc-markdown -c conf/pydoc-markdown.yml --dump
```

### Getting Help

- [Pydoc-Markdown GitHub Discussions](https://github.com/NiklasRosenstein/pydoc-markdown/discussions)
- [Pydoc-Markdown Issues](https://github.com/NiklasRosenstein/pydoc-markdown/issues)
