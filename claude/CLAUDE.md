# Team Rules (Pragmatic Minimum-Credit Crew)

## Golden Rule: Do Not Spam Sub-Agents (Warm Cache First!)
Every spawned sub-agent reloads the system prompt, `CLAUDE.md`, and tool schemas from scratch. Meanwhile, your main session (`sonnet`, `medium` effort) already has a warm prompt cache.

1. **1–2 File Tasks & Routine Reads/Greps → DO IT YOURSELF:**
   - Do not spawn a sub-agent just to grep or read a single file. Use Grep and line-range Reads directly in the main session.
   - Write your own concise 5–10 line plan on turn 1.
   - If a task touches 1–2 files, implement and test it yourself in the main session.
2. **Large Multi-File Tasks → USE SUB-AGENTS:**
   - **Scout (`Explore` — `haiku`, `low` effort):** Spawn **only** for broad repo-wide searches (10+ files) so raw search noise stays out of the main context. Returns `file:line` and max 5-line excerpts.
   - **Builders (`worker` — `sonnet`, `medium` effort):** Spawn for 3+ independent module tasks (one per module, non-overlapping files). **Never let `haiku` write code.**
   - **Judge (`checker` — `opus`):** Run before calling a multi-file/complex task done (reads only spec + diff + test result with fresh context). Skip for trivial 1-file changes already verified by tests.
   - **Crisis Advisor (`advisor` — `opus`, `high` effort, read-only):** Built-in advisor is disabled (`CLAUDE_CODE_DISABLE_ADVISOR_TOOL=1`). Call `advisor` **ONLY** on the 5 triggers below with a max 300-word brief + 10-line map.

## Call the `advisor` ONLY When (5 Triggers)
1. The same error survives 2 honest fix attempts.
2. You must choose an architecture, a data model, or a library.
3. The change touches auth, payments, migrations, or user data.
4. Tests pass, but the behavior is still wrong and you do not know why.
5. A fix would change more than 5 files or a public interface.

## Advisor Brief Format (Max 300 Words — Never Paste the Transcript)
- **Goal:** What should work, in one sentence.
- **Map:** The 10-line project map.
- **Tried:** Each attempt and why it failed.
- **Evidence:** The exact error, failing test output, or wrong vs. expected behavior.
- **Files:** Paths and line ranges that matter (max 5).
- **Options:** The 2 approaches you are choosing between, if any.
- **Question:** The one decision you need help with.

## Context & Token Discipline
- Keep `CLAUDE.md`, settings, and the plan unchanged during a session so the prompt cache keeps hitting.
- Never read a large file end-to-end; grep line numbers first and read only the relevant slice.
- Skip `node_modules`, build output, and lockfiles.
- Never pour long test/command outputs into the chat — keep only the summary and failing lines.
