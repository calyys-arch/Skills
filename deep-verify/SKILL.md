---
name: deep-verify
description: >-
  Prevents three common Cursor mistakes: (1) shallow code review that only reads README/PRD instead
  of actual source files, (2) skimming documents/books instead of reading them fully, and (3)
  writing code that is never actually executed but is documented as if it were running.
  Use whenever the user asks to optimize a system, read a document or book, implement a feature,
  or verify that the system matches its documentation.
---

# Deep Verify

Three rules, always enforced. No exceptions.

---

## Rule 1 — Optimization Reviews Must Explore Actual Code

When asked whether a system can be optimized or improved:

**WRONG approach:**
- Read README.md or PRD.md and suggest improvements based on what's documented

**CORRECT approach:**
1. Use Glob to list all source files in the relevant directories
2. Read every file that could contain the logic being reviewed — not just entry points
3. Trace actual function call chains; follow imports to their implementations
4. Only after reading the real code, identify concrete optimization opportunities with file + line references
5. Never recommend changes based solely on what README/PRD says the system does

> If a file has not been read, you do not know what it does. Documentation lies; code does not.

---

## Rule 2 — Documents and Books Must Be Read in Full

When asked to read, learn from, or summarize a document, book, PDF, or any long-form text:

**WRONG approach:**
- Read only the introduction, abstract, or summary section
- Skim headings and infer content

**CORRECT approach:**
1. Read the document section by section, from beginning to end
2. For long files, use `offset` and `limit` parameters to paginate through the entire content
3. Keep track of which sections have been read; do not skip any
4. Only synthesize conclusions after all sections have been processed
5. If the document references other documents as prerequisites, read those too

> A summary of a summary is useless. Read the source.

---

## Rule 3 — Code Must Be Verified as Actually Running Before Being Documented

When implementing a feature, module, or integration:

**WRONG approach:**
- Write the code
- Update README.md or PRD.md to say the feature is complete
- Move on without verifying execution

**CORRECT approach:**
1. Write the code
2. Verify it is actually invoked by the running system:
   - Check that it is imported or called by an active entry point
   - Run the relevant test, script, or server command and confirm the code path executes
   - If the code is conditional, verify the condition is actually met
3. Only after confirmed execution, update README/PRD to reflect the feature as live
4. If execution cannot be verified, mark the feature as `[NOT YET ACTIVE]` in documentation

**Before closing any implementation task, explicitly state:**
- Which file and function was added/changed
- How you confirmed it runs (test output, log line, import chain, etc.)
- Whether README/PRD has been updated — and only if execution was confirmed

> If you wrote it but did not run it, it does not exist. Document only what runs.

---

---

## Rule 4 — Run Smoke Tests After Every Significant Update

When a significant change is completed (new feature, refactor, dependency update, config change, multi-file edit):

**WRONG approach:**
- Tell the user "the update is complete" without running anything
- Assume the system still works because the individual pieces look correct
- Leave the user to discover broken functionality themselves

**CORRECT approach:**
1. After finishing the changes, identify the system's critical paths (entry points, key APIs, main workflows)
2. Run smoke tests before declaring the task done:
   - Start the relevant server/process and confirm it starts without errors
   - Execute the core functions that were changed or could have been affected
   - Run existing test suites if available (`pytest`, `npm test`, `cargo test`, etc.)
   - If no tests exist, manually invoke the changed code and verify output
3. Report the smoke test results explicitly:
   - Which commands were run
   - What the output was (success, warnings, errors)
   - Whether any previously working functionality is now broken
4. If a smoke test fails, fix the issue before closing the task — do not hand back a broken system

**Smoke test scope by change size:**

| Change Size | Minimum Smoke Test |
|-------------|-------------------|
| Single function fix | Run the affected function / unit test |
| New module or feature | Start the app + exercise the new feature end-to-end |
| Refactor across multiple files | Full test suite + manual check of changed flows |
| Dependency or config update | Full startup + regression check on all major features |

> The user should never be the first one to discover that something is broken.

---

## Checklist — Apply Before Completing Any Task

- [ ] Have I read the actual source files, not just README/PRD?
- [ ] Have I read the full document, not just the summary?
- [ ] Have I verified the code runs, not just that it compiles or exists?
- [ ] Does every claim in README/PRD correspond to something that actually executes?
- [ ] Have I run smoke tests and reported the results before saying the task is done?
- [ ] Is the system in a working state right now, confirmed by me — not left to the user to find out?
