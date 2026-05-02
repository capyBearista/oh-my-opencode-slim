# oh-my-opencode-slim Customization Plan

This document outlines the steps to customize the `oh-my-opencode-slim` fork to perfectly align with a personalized workflow, particularly focusing on Model usage (Anthropic), Agent Role Tweaks, Skill Integration (`impeccable`), and adding a new Documentation agent (`Codexer`).

## Phase 1: Model Configuration (Anthropic Integration)
By default, the plugin uses OpenAI and OpenCode Go models. To switch to Anthropic models (Claude 3.5/3.7 Sonnet for high reasoning, Haiku for execution), we will set up a custom preset.

**Action Item:**
Modify the `src/cli/install.ts` or explicitly configure `oh-my-opencode-slim.json` to use:
*   **Orchestrator & Oracle & Council:** `anthropic/claude-3.5-sonnet` (or `claude-3.7-sonnet`)
*   **Fixer, Explorer, Librarian, Designer:** `anthropic/claude-3.5-haiku`

Example `oh-my-opencode-slim.json` override snippet:
```jsonc
{
  "preset": "custom-anthropic",
  "presets": {
    "custom-anthropic": {
      "orchestrator": { "model": "anthropic/claude-3.5-sonnet", "skills": ["*"], "mcps": ["*"] },
      "oracle": { "model": "anthropic/claude-3.5-sonnet", "variant": "high", "skills": ["simplify"], "mcps": [] },
      "librarian": { "model": "anthropic/claude-3.5-haiku", "variant": "low", "skills": [], "mcps": ["websearch", "grep_app"] },
      "explorer": { "model": "anthropic/claude-3.5-haiku", "variant": "low", "skills": [], "mcps": [] },
      "designer": { "model": "anthropic/claude-3.5-sonnet", "variant": "medium", "skills": ["impeccable"], "mcps": [] },
      "fixer": { "model": "anthropic/claude-3.5-haiku", "variant": "low", "skills": [], "mcps": [] },
      "codexer": { "model": "anthropic/claude-3.5-sonnet", "variant": "high", "skills": ["writing-auditor"], "mcps": [] }
    }
  }
}
```

## Phase 2: Agent Role Tuning (Fixer & Oracle)

To align `Fixer` more closely with `QA-Engineer` and `Oracle` with `Code-Reviewer`, while retaining their broader orchestration utility, we will *append* instructions to their prompts rather than replacing them entirely.

**Action Item for Fixer (`src/agents/fixer.ts`):**
Append rules to prioritize its role in testing and stack trace debugging:
*   "Prioritize writing tests and fixing broken builds above all else."
*   "When tests fail, parse the noisy stack traces in isolation and only communicate the necessary fixes back to the Orchestrator."

**Action Item for Oracle (`src/agents/oracle.ts`):**
Append rules to prioritize its role in post-code review:
*   "Act as a strict, senior code reviewer."
*   "Focus heavily on enforcing best practices, maintainability, and clean architecture (e.g. YAGNI) after execution phases are complete."

## Phase 3: Integrating the `impeccable` Skill into Designer

The `impeccable` skill is highly complex and relies on executing Node scripts to read design/product guidelines (`node .rovodev/skills/impeccable/scripts/load-context.mjs`).

**Action Items:**
1.  **Dynamic Config (Recommended):** Add `"skills": ["impeccable"]` to the `designer` configuration in `oh-my-opencode-slim.json`.
2.  **Hardcode Execution Rules:** In `src/agents/designer.ts`, explicitly instruct the Designer agent:
    *   "Before making ANY frontend or UI changes, you MUST execute `node .rovodev/skills/impeccable/scripts/load-context.mjs` to fetch product and design guidelines."
    *   "Adhere strictly to the design laws provided by the impeccable skill."

## Phase 4: Adding the Missing `Codexer` (Documenter) Agent

The OMO-slim architecture currently lacks a dedicated agent for writing documentation, forcing the Orchestrator to do it and bloating its context window.

**Action Items:**
1.  **Create the Agent:** Add `src/agents/codexer.ts` with instructions focused entirely on writing clean, high-quality documentation (READMEs, API references, changelogs).
2.  **Update Orchestrator Routing:** In `src/agents/orchestrator.ts`, add `@codexer` to the `AGENT_DESCRIPTIONS`.
    *   **Delegate when:** "Writing READMEs, generating API specs, documenting project architecture, or writing release notes."
    *   **Don't delegate when:** "Writing inline code comments during active execution."
    *   Add validation routing: "- Route all README, API spec, and changelog generation to @codexer."
3.  **Register the Agent:** Export and register `createCodexerAgent` inside `src/agents/index.ts`.

## Phase 5: Build and Use
After making these changes:
1.  Run `bun run build` inside `/home/arjun/oh-my-opencode-slim`.
2.  Update `/home/arjun/.config/opencode/opencode.json` to load the local fork:
    ```json
    "plugin": [
      "file:///home/arjun/oh-my-opencode-slim"
    ]
    ```
