# Working Rules (Pragmatic Minimum-Credit Architecture)

Place this file at `~/.codex/AGENTS.md` (applies to every project)
or at a project root to scope it to that project.

## Golden Rule: Do Not Spam Sub-Agents (Warm Cache First!)
Every spawned sub-agent reloads the system prompt, rules, and MCP/plugin tool schemas from scratch (`~25k–40k` tokens). Meanwhile, your main session (`gpt-6.1-sol`, `medium` effort) already has a **97–99% discounted Warm Prompt Cache**.

Therefore:
1. **1–2 File Tasks & Routine Reads/Greps → DO IT YOURSELF:**
   - Never spawn a sub-agent just to run `rg` (grep) or read a file. Run `rg` yourself and read only the relevant `file:line` range (30–80 lines) instead of the whole file.
   - Never spawn a planner sub-agent: write your own 5–10 line plan in the main session on turn 1.
   - If a task touches 1–2 files, implement and test it directly in the main session.
2. **3+ Independent Modules / Heavy Fan-Out → USE SUB-AGENTS:**
   - **`sol_explorer` (`gpt-6-luna`, `medium` effort):** Spawn **only** when you need to scan/map 10+ files across the repo and want to keep raw search noise out of your main context. It returns `file:line` + max 5-line excerpts.
   - **`sol_worker` (`gpt-6.1-sol`, `medium` effort):** Spawn 2–3 in parallel **only** when editing 3+ independent modules/files. Give each worker only: `(1) one task sentence, (2) owned file:line paths, (3) test command`. No two workers touch the same file.
   - **`terra_asset_worker` (`gpt-6.1-terra`, `medium` effort):** Use only for multi-asset texture/sprite/visual production passes.
3. **Verification Without Waste:**
   - If a compiler check (`cargo check`, `tsc`) or unit test (`cargo test`, `npm test`) proves the change works on a small/routine task, **do not spawn a verifier sub-agent**.
   - **`sol_verifier` (`gpt-6.1-sol`, `medium` effort, fresh context):** Spawn once at the end **only** for complex multi-file, protocol, or logic changes where tests alone could miss silent regressions. Never use `xhigh`.
   - **`terra_visual_verifier` (`gpt-6.1-terra`, `medium` effort):** Never call mid-build. Call **only once at the very end** after all code tests pass, and only if the task involves UI/render/visual output.

## Call `sol_advisor` (`gpt-6.1-sol`, `high`) ONLY When (5 Crisis Triggers)
1. The same error survives 2 honest fix attempts.
2. A major architecture, data model, or library choice is required.
3. The change touches auth, payments, migrations, or user data.
4. Tests pass, but the behavior is still wrong and you do not know why.
5. A fix would change more than 5 files or a public interface.

### Advisor Brief Format (Max 300 Words — Never Paste the Transcript)
- **Goal:** What should work, in one sentence.
- **Map:** The 10-line project map.
- **Tried:** The 2 attempts and why they failed.
- **Evidence:** Exact error or wrong vs. expected behavior.
- **Files:** Relevant `file:line` ranges (max 5).
- **Options:** The 2 approaches under consideration.
- **Question:** The single decision needed.

## Context & Token Protection
- Batch independent tool calls (`rg`, line-range reads) into a single turn.
- Never read a large file end-to-end; inspect line numbers with `rg` first and read only the needed section.
- Skip `node_modules`, build output, and lockfiles.
- Never dump long command/test outputs into the chat: redirect to a file (e.g., `reports/`) and print only the summary + failing lines.
- Never use `xhigh` reasoning effort unless the user explicitly asks for it.
