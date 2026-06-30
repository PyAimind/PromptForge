# Muscular Prompt Engineering — Methodology

A structured approach to building software with AI collaboration, developed and tested across real projects (including [RACE v2.0](https://github.com/PyAimind/RACE-AI-Rescue-Tournament-v2)).

## Roles

| Role | Who | Responsibility |
|---|---|---|
| **Supervisor** | Human (me) | Explains the full project, sends prompts, reviews errors, makes final decisions. Usually drafted in a Notepad/Word file first. |
| **Engineer** | An AI model | Breaks the project into phases, writes precise prompts for the Coder, fixes errors by issuing new prompts. |
| **Coder** | An AI model | Writes code strictly based on the Engineer's prompt. |

> **Model choice is up to you.** This methodology was originally developed using ChatGPT as Engineer and DeepSeek as Coder, and it worked well. It isn't tied to those two models — any capable model can fill either role, and a project could even use a single strong model for both responsibilities at once (e.g. Claude). What matters is keeping the Engineer/Coder separation of concerns and the rules below, not which specific model sits in which seat. Choose the best model(s) available to you at the time.

## Workflow

1. The Supervisor describes the entire project in a text file to the Engineer.
2. The Engineer designs the overall structure (file names, folders, functions).
3. The Engineer writes one prompt at a time for the Coder — never the whole project at once.
4. The Coder writes code based on that single prompt, following the rules below.
5. The Supervisor tests the code and reports the result.
6. If there's an error, the Supervisor sends it back to the Engineer along with the original prompt that caused it.
7. The Engineer analyzes the error and writes a corrected prompt.
8. Repeat steps 3–7 until the project is complete.

Every prompt the Engineer writes must fully specify how the code should be written, according to the rules below — leaving no ambiguity for the Coder to fill in.

## Core Rules

**1. Absolute clarity in prompts**
A prompt must read like an instruction manual — precise and complete. Every part of the task must be covered. The smallest ambiguity leads to incorrect or inconsistent code.

**2. Strict size and complexity limits**
- No function should exceed **30 lines** if it can reasonably be written shorter.
- No file should exceed **200 lines** (up to 250 in justified cases).
- Use the simplest, most modern approach available — never outdated patterns.
- Example: if a database layer can be written in 30 lines, it should not be written in 100.

**3. Consistent naming**
File, folder, function, and variable names must stay consistent across the entire project so modules connect cleanly. A function used on page one and page three must keep the same name everywhere. This prevents errors like `KeyError` and `NameError`.

**4. Testability after every part**
After each module, the Engineer provides a short test script (5–10 lines) so that part can be verified in isolation, with a clearly defined expected output. The test script itself follows its own rules (see Rule 6). Tests are written by the Engineer, not the Coder.

**5. Disciplined error recovery**
When an error is reported, the Engineer must:
- Diagnose where the error originates.
- Write a new prompt that fixes it.
- Re-apply all core rules in the new prompt.
- Leave the overall architecture unchanged (see Rule 10).

**6. Incremental, cumulative integration**
Never write all modules separately and wire them together at the end in one large file. Instead:
- **Phase 1:** Build and test Module 1 alone. Confirm it works.
- **Phase 2:** Build Module 2, then test Modules 1 + 2 together.
- **Phase 3:** Build Module 3, then test Modules 1 + 2 + 3 together.

Each phase's test prompt must specify exactly which functions connect to which, and how — including cases where a function name is shared but its output must change behavior in the new context.

For UI/UX quality (which can't be unit-tested because it's subjective), use manual review phases instead: after a phase like a login screen is done, the Supervisor uses it directly, gives concrete feedback ("buttons are too small," "should submit on Enter"), and the Engineer issues a follow-up prompt. The accompanying test script should still surface technical errors in the terminal (e.g., a UI failing to load) even though aesthetic judgment stays manual.

**7. Phase exit criteria**
No phase is complete, and no next phase begins, until all tests for the current phase pass 100%. No uncategorized errors or lint issues may remain.

**8. No unsolicited commentary in code**
The Coder delivers only raw code and test scripts — no extra explanation, not even as comments — unless the Supervisor explicitly requested it in the prompt.

**9. Hallucination control**
If the Coder references a library or function that doesn't actually exist, that's a hallucination. To prevent this, the Engineer's prompts must explicitly state which libraries are allowed and discourage unfamiliar or obscure ones.

**10. No structural changes during bug fixes**
When fixing an error, the Engineer may not rename core files, folders, or functions unless the Supervisor explicitly approves it. Only the actual bug and data-type mismatches may be corrected.

## Additional Notes

- Write prompts in English — this reduces Coder errors significantly.
- Every prompt should state which previous part(s) it connects to.
- Define the full file/folder structure before any code is written.
- For each part, specify exactly which functions are needed.

## Prompt History Log (for larger projects)

Not recommended for small projects, but useful once a project grows. Keep a simple text file (`prompt_history.txt`) with entries like:

```
=== Session 2025-03-15 10:30 ===
[My prompt to Engineer]   Project: login system, explain structure...
[Engineer's prompt to Coder]  Module: auth/login.py, ...
[Coder's code]   <code received>
[Test result]   Error: KeyError 'password' in line 12
[Fix round]   Sent error directly to Coder, got corrected code.
```

This is useful because:
- You can later spot recurring error patterns (e.g., "Coder always forgets imports").
- If you resume the project days later, you can see which prompts already worked.

## Example Initial Structure

This is only an example — not a required structure.

```
project_name/
├── main.py
├── auth/
│   ├── __init__.py
│   ├── register.py
│   └── login.py
├── database/
│   ├── __init__.py
│   └── db_handler.py
└── templates/
    └── index.html
```
