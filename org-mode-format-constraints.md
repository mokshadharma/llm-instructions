# Org‑Mode Formatting Specification
Version: 3.0.0

1. Top-Level Heading
   - Exactly one top-level Org heading (`* <Title>`); it MUST be descriptive.
   - No other level-1 (`*`) headings are allowed.
   - No blank line precedes this heading.

2. Subheading Hierarchy
   - Use additional leading `*` characters to create deeper levels (no level skipping).
   - A heading MAY omit body text if immediately followed by child headings.
   - Do not add filler prose solely to satisfy a structural placeholder.

3. Blank Line Restrictions Around Headings (High-Density Structural Artifact)
   - No blank line immediately before any heading, regardless of heading level or preceding content. This restriction is **absolute** and overrides all conventional document spacing practices for visual separation.
   - No blank line immediately after any heading.
   - A heading MUST be followed *directly* by its first prose line, a block directive (`#+BEGIN_...`), or the next child/sibling heading.
   - The ONLY allowed blank lines in the document's prose are single blank lines used to separate adjacent paragraphs, as governed by Rule 29.

4. Enumerations as Headings (List Syntax Prohibited)
   - All Org list constructs are forbidden (unordered: `-`, `+`, `*`; ordered: `1.`, `2.`; checkboxes; description lists `::`).
   - Any enumeration MUST be expressed as sibling headings at the appropriate depth.
   - Ordered sequences: encode the number in the heading text itself (e.g., `** 1 Initialize`, `** 2 Configure`).
   - Unordered sets: use headings without numeric prefixes.
   - Nested enumerations: increase heading depth.
   - Do not simulate bullets with punctuation or symbols at line starts—only heading stars introduce items.
   - If an item has no body, the next heading may follow immediately.
   - Number sequences SHOULD be strictly increasing integers without gaps (future rule may refine exceptions) unless overridden by Rule 36.

5. Inline Emphasis and Inline Code
   - Forbid all Org inline emphasis/verbatim markers: `*bold*`, `/italic/`, `_underline_`, `+strike+`, `=verbatim=`, `~code~`.
   - No inline styling for literals; all commands/code/config must appear in blocks (see Rule 13).

6. Allowed Block Directives
   - Only these block delimiters (uppercase) are permitted:
     - `#+BEGIN_SRC` / `#+END_SRC`
     - `#+BEGIN_EXAMPLE` / `#+END_EXAMPLE`
   - All other `#+` directives are forbidden (e.g., `#+TITLE:`, `#+INCLUDE:`, `#+OPTIONS:`, `#+PROPERTY:`, `#+MACRO:`).

7. Properties Drawers
   - `:PROPERTIES:` drawers are forbidden at all levels.

8. TODO / DONE / Priority Markers
   - Task keywords (e.g., `TODO`, `DONE`) and priority markers (e.g., `[#A]`) are forbidden in headings.

9. Footnotes
   - Org footnotes (definitions or references) are forbidden.

10. Tables
    - Org tables are forbidden.

11. Macros and Includes
    - Macros (`{{{...}}}`) and include directives are forbidden.

12. Source Block Delimiter Case
    - Use only uppercase `#+BEGIN_SRC` / `#+END_SRC`.

13. Source Block Requirement
    - Every command, code snippet, configuration fragment, or other technical literal content MUST be inside a source block (unless it is authentic command output, which belongs in an example block).

14. Source Block Language Annotation (Optional)
    - Language tag is OPTIONAL.
    - Omit if it adds no clarity; never use placeholder tags.
    - When used, it must improve understanding (e.g., `bash`, `json`, `yaml`, `python`).

15. Prohibited Inline Representation of Literals
    - Do not approximate inline code with quotes, unusual punctuation, or capitalization. Use source blocks regardless of snippet length.

16. Source Block Internal Blank Lines
    - First and last lines inside a source block MUST NOT be blank.
    - Internal blank lines between non-blank lines are allowed.

17. Non-Empty Source Blocks
    - Source blocks must contain at least one non-whitespace line (no empty or whitespace-only blocks).

18. Language Tag Style (When Present)
    - If provided, the language tag MUST be lowercase (e.g., `bash`, `json`, `yaml`, `python`, `text`).

19. Trailing Whitespace
    - Trailing whitespace (spaces or tabs) is forbidden on every line, including inside blocks.

20. Tabs
    - Tabs are forbidden in all prose lines.
    - Tabs MAY appear in source blocks only when semantically required by the represented language/format (e.g., Makefile recipes).
    - Example blocks (Rule 24) MAY contain tabs only if they appear in the authentic captured output.
    - Do not use tabs for optional alignment; use spaces where tabs are not mandatory.

21. Line Length
    - Prose lines (outside blocks) MUST NOT exceed 80 characters.
    - Lines inside source or example blocks have no line length limit; do not wrap code, commands, or output artificially.

22. Shell Prompts
    - Forbid leading prompt characters (`$`, `#`, `%`, `>`, etc.) in source blocks.
    - Root or privilege requirements must be conveyed in prose, not via prompt symbols.

23. Separation of Inputs (Commands / Artifacts) and Output
    - Source blocks may contain user-entered commands and/or user-authored input artifacts (e.g., configuration files, manifests, scripts, sample request/response bodies intended as inputs).
    - Command output MUST NOT appear in source blocks.
    - Example blocks (Rule 24) are exclusively for authentic command output.
    - Do not split a single coherent artifact unnecessarily across multiple source blocks.

24. Example Blocks (Command Output)
    - Use `#+BEGIN_EXAMPLE` / `#+END_EXAMPLE` for command output only.
    - Example blocks MUST NOT contain commands or user-authored input artifacts.
    - Non-empty: must include at least one non-whitespace line.
    - First and last lines inside an example block MUST NOT be blank.
    - Internal blank lines allowed.

25. Command + Output Pairing Order
    - When output is shown, the pattern is:
      - Source block (commands and/or associated input artifact if produced by the user)
      - Immediately adjacent example block (output)
    - No intervening blank line or prose between a command source block and its associated example block.
    - Multiple pairs may appear sequentially.

26. Output Inclusion Criterion
    - Include output only if authentic captured output exists in the source material.
    - Never fabricate or synthesize output. If authentic output is not available, omit the example block.

27. Output Redaction
    - No redaction, sanitization, or normalization. Reproduce output verbatim exactly as captured (including any sensitive/environment-specific values).

28. Output Truncation
    - Truncation, elision, or omission markers are forbidden.
    - If any output is shown, include the entire captured output in full.

29. Consecutive Blank Prose Lines
    - At most one blank prose line may appear consecutively.
    - Two or more sequential blank lines in prose are forbidden.

30. External Links / URLs
    - External URLs may appear either as bare URLs or using Org link syntax (`[[url]]` or `[[url][label]]`).
    - When a label is supplied it MUST be meaningful (not "here" or "link").
    - No requirement to label every link.

31. Character Set / Encoding
    - The file MUST be encoded in UTF-8 without a byte-order mark (BOM).
    - All characters must be valid UTF-8 code points.
    - Invalid byte sequences are forbidden.

32. Heading Title Uniqueness
    - Among sibling headings (same depth), each heading title MUST be unique (case-sensitive).
    - Identical titles at the same level are forbidden.
    - The same title may appear at different depths.

33. Validation Severity
    - Every rule in this specification is enforced as an Error.
    - Any violation constitutes non-compliance. No warning or informational levels exist.

34. Non-Command Data in Source Blocks
    - Configuration/data artifacts placed in source blocks must be faithfully represented.
    - Such artifacts MUST NOT be labeled or altered to resemble output (they are user-provided inputs).
    - Do not include explanatory prose inside the block; keep prose outside.

35. Differentiation Markers
    - No special markers, mandatory language tags, or header comments are required to distinguish commands from non-command artifacts inside source blocks.
    - Optional language tags remain governed solely by Rule 14.

36. Ordered Enumeration Numbering (Mirror Source with Prefix Uniqueness)
    - Numeric or alphanumeric prefixes at the start of heading titles (e.g., `1`, `01`, `2.1`, `A`, `A.1`, `3b`, `0`, `-1`) are preserved exactly as they appear in the source material.
    - No normalization, renumbering, padding adjustment, sequence validation, or contiguity requirement is imposed.
    - Leading zeros, composite/dotted identifiers, alphanumeric schemes, zero, and negative values are all allowed if present in the source.
    - Among sibling headings that begin with a recognizable first-token prefix (text before the first space), that exact prefix MUST be unique (case-sensitive). Example: `** 5 Install` and `** 5 Configure` are forbidden together.
    - Heading title uniqueness (Rule 32) still applies to the entire title string.
    - Mixed numbered and unnumbered siblings are allowed; unnumbered headings have no prefix for uniqueness comparison.
    - Tooling must not flag gaps, descending order, or irregular patterns.
    - Authors must not fabricate or alter prefixes solely for numeric consistency; mirror the source except when changing a prefix is necessary to resolve a sibling prefix collision.

37. Extremely Large Command Output
    - Full authentic output MUST always be included regardless of size.
    - No exceptions, thresholds, externalization, summarization, hashing, windowing, or elision beyond Rules 26–28 are permitted.
    - Performance or size concerns are accepted trade-offs.

38. Specification Versioning
    - The specification uses Semantic Versioning (MAJOR.MINOR.PATCH).
    - MAJOR: Backward-incompatible rule changes (previously valid documents may become invalid).
    - MINOR: Backward-compatible additions or relaxations (previously valid documents remain valid).
    - PATCH: Editorial corrections that do not change validation outcomes.
    - Each published change increments exactly one component.
    - Published versions are immutable.

39. Changelog
    - No changelog section is maintained.
    - Version history is discoverable only through repository history (e.g., Git tags or commit log).
    - Tooling must not expect or parse an in-document changelog.

40. Version Declaration Line
    - The first non-blank line immediately following the top-level heading MUST be `Version: MAJOR.MINOR.PATCH`.
    - This line is the canonical in-file source of the current specification version.
    - No other lines may begin with `Version:` elsewhere in the document.

41. Content Consolidation in Headings
    - When converting list items to headings, all content MUST be incorporated into the heading title itself whenever possible.
    - Body text under a heading should only exist when the content is too complex, lengthy, or multi-sentence to fit naturally in the heading title.
    - A heading with a single short sentence or phrase as body text is prohibited; that content MUST be moved into the heading title.
    - Repetition between heading title and body text is forbidden.
    - When a source list item contains a brief label followed by expansion/explanation, prefer making the label the heading title and placing only the expansion in the body, OR combine both into a comprehensive heading title if the result remains clear and under 80 characters.

42. Category Heading Without Repeated Prefix in Children
    - When converting a list with a category or section name (e.g., "Style Requirements:") followed by multiple related items, create one parent heading with the category name and child headings with the specific item content.
    - The category name MUST NOT be repeated as a prefix in each child heading title.
    - Example: A source section "Style Requirements" with items "Short functions", "Pure functions", "Defensive programming" converts to:
      ```
      ** Style
      *** Short single-purpose functions that do one thing well
      *** Pure functions where possible with no side effects
      *** Defensive programming with input validation at function boundaries
      ```
    - WRONG: Creating sibling headings like `** Style - Short functions`, `** Style - Pure functions`, `** Style - Defensive programming` at the same level.
    - The parent heading establishes context; children inherit that context without repetition.

43. Avoiding Redundant Hierarchy
    - Do not create a parent heading with a single child heading that essentially repeats the parent's meaning.
    - Do not create intermediate organizational headings solely to avoid having body text under a parent heading.
    - If all "children" are really peers (e.g., a flat list of related items under a common category), keep them as children of a single parent with the category name, not as siblings with repeated prefixes.
    - Hierarchy depth should reflect genuine conceptual nesting in the source material, not artifact from list-to-heading mechanical conversion.

44. Converting Multi-Level Lists
    - When source material contains multi-level lists (list items with sub-items), increase heading depth to match nesting.
    - Each nesting level in the source list becomes one additional `*` in the heading.
    - Do not flatten nested source structure into single-level headings.
    - Do not introduce extra hierarchy levels beyond what the source structure contains.

45. Recognizing Category Sections in Source Material
    - When source material has a section title or label followed by a list of related items (whether bulleted, numbered, or paragraph-separated), that structure indicates a parent-child relationship.
    - The section title becomes the parent heading.
    - Each list item becomes a child heading (one level deeper).
    - Do NOT create sibling headings with the section title as a repeated prefix.
    - Look for structural indicators: colons after category names, indentation, explicit grouping, or contextual relationship among items.

46. Forbidden Citation, Attribution, and Agent Tracking Markers (Absolute Agent Override)
    - All forms of explicit source citation, attribution, tracking markers, and agent-added metadata are strictly forbidden in the final transformed document.
    - The transformed output MUST be presented as a standalone artifact, without external references.
    - Prohibition includes Inline Source References (e.g., `[1]`, `(Smith, 2024)`), Footnote / Endnote Markers, and Agent/Tool Metadata (e.g., ``).
    - No text whose sole purpose is source referencing is permitted in the document body or headings.
