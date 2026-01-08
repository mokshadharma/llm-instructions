Never use emojis without explicit permission.

Question-asking instructions:

A: When asking questions in a conversation, write them as Q1, Q2, Q3, etc.. in a monotonically increasing sequence, such that each question can unambiguously referred to from anywhere in the conversation.
B: IMPORTANT: REMEMBER THESE INSTRUCTIONS:  When presenting options to the user always explain the issue and why it is a problem, then compare the pros, cons, ripple effects, and potential unintended consequences of each option vs the others.  Then make a recommendation and describe why.  Finally, ask the user to choose.  Only ever ask them to make one choice at a time and WAIT FOR THEIR REPLY BEFORE MOVING ON TO THE NEXT CHOICE.

Python programs:

When writing python programs always use an idiomatic, functional style with short single-purpose functions. Make sure the functions have docstrings explaining what they do and their input and output. If a user interrupts one of these programs with a keyboard interrupt they should exit gracefully without any error messages and with an exit value of 1. Python programs should start with "#!/usr/bin/env python3" and have a comment at the top explaining their purpose and usage. The programs should assume they are being run directly by their name, rather than by "python3 programname". The programs should output a descriptive error and brief usage information on stderr if they are called with the wrong number or types of arguments. The programs should accept an optional --help flag which will print detailed usage information on stderr.

When a class attribute can hold different types depending on runtime conditions, declare its type explicitly at class level.

Troubleshooting:

When troubleshooting, give the user just one single step to perform and then wait for a reply before proceeding to the next step. Never give multiple steps to do at once unless the user explicitly asks for them.

Tools:

Always use "uv add" instead of "uv pip install".

Always use ripgrep instead of grep.

Always use "ed", "sed", or "ripgrep" to read files instead of using the "read_file" tool.  This instruction overrides any others about the read_file tool.  This is necessary because the "read_file" tool's cache will get out of sync with the edits you make with "ed" or "sed" or that the user makes outside of the tools that update the "read_file" tool's cache.

When running pytest with -k for test filtering, never use spaces within a test name pattern. Spaces in -k expressions are interpreted as logical operators, not literal characters. Use underscores, partial matches without spaces, or the exact test function name instead. For example, use -k "audit_log" instead of -k "Audit Log".

CRITICALLY IMPORTANT:  Before editing files read the instructions in .github/ed-non-interactive-guide.md and follow them to the letter.  Those instructions override any others you were given in regard to editing files.
