# oh-my-opencode-slim Customization Plan (Revised)

This document outlines the corrected, step-by-step procedure to customize the `oh-my-opencode-slim` fork to perfectly align with your personalized workflow.

## Phase 1: Model Configuration (Other Provider Integration)
By default, the plugin uses OpenAI and OpenCode Go models. To switch to other models, we will set up a custom preset. Crucially, because we are adding `codexer` as a custom agent (until we hardcode it as built-in), it must be defined under `agents`.

**Action Item:**
Modify `oh-my-opencode-slim.json` (located at `~/.config/opencode/oh-my-opencode-slim.json`):

```jsonc
{
  "preset": "custom",
  "presets": {
    "custom": {
      "orchestrator": { "model": "google/gemini-3.1-pro-preview", "variant": "high", "skills": ["*, !agent-browser, !simplify"], "mcps": ["*", "!context7"] },
      "oracle": { "model": "google/gemini-3.1-pro-preview", "variant": "high", "skills": ["simplify"], "mcps": [] },
      "librarian": { "model": "google/gemini-3-flash-preview", "variant": "low", "skills": [], "mcps": ["websearch", "context7", "grep_app"] },
      "explorer": { "model": "google/gemini-3-flash-preview", "variant": "medium", "skills": [], "mcps": [] },
      "designer": { "model": "google/gemini-3.1-pro-preview", "variant": "medium", "skills": ["agent-browser, impeccable"], "mcps": [] },
      "fixer": { "model": "github-copilot/gpt-5.2-codex", "variant": "high", "skills": [], "mcps": [] },
      "council": { "model": "google/gemini-3.1-pro-preview" }
    }
  },
  "agents": {
    "codexer": {
      "model": "github-copilot/gpt-5.4-mini",
      "skills": ["writing-auditor"]
    }
  },
  "council": {
    "default_preset": "default",
    "presets": {
      "default": {
        "alpha": { "model": "opencode-go/kimi-k2.6" },
        "beta": { "model": "google/gemini-3.1-pro-preview", "variant": "medium"},
        "gamma": { "model": "github-copilot/gpt-5.2", "variant": "high" }
      }
    }
  },
  "multiplexer": {
    "type": "auto",
    "layout": "main-vertical",
    "main_pane_size": 60
  },
}
```

## Phase 2: Agent Role Tuning (Fixer & Oracle)

To safely append instructions to built-in agents *without* breaking source code or risking merge conflicts on future upstream pulls, OMO-slim supports dynamic prompt appending.

**Action Item:**
Create two files in `~/.config/opencode/oh-my-opencode-slim/`:

1.  **`fixer_append.md`:**
    ```text
    Prioritize writing tests and fixing broken builds above all else.
    When tests fail, parse the noisy stack traces in isolation and only communicate the necessary fixes back to the Orchestrator.
    ```

2.  **`oracle_append.md`:**
    ```text
    Act as a strict, senior code reviewer.
    Focus heavily on enforcing best practices, maintainability, and clean architecture (e.g. YAGNI) after execution phases are complete.
    ```

*OMO-slim automatically detects these `_append.md` files and injects them into the base prompts.*

## Phase 3: Integrating the `impeccable` Skill into Designer

`impeccable` is NOT a bundled skill in OMO-slim and cannot be executed via hardcoded bash commands in agent prompts. OpenCode handles skill tool permissions independently via its SDK.

**Action Items:**
1.  **Install the Skill Externally:** 
    You must install the skill using OpenCode's CLI so it downloads to your global `~/.config/opencode/skills` directory:
    ```bash
    opencode skills add https://github.com/pbakaus/impeccable --skill impeccable -a opencode -y --global
    ```
2.  **Assign to Designer:** Ensure `"skills": ["impeccable"]` is present in the `designer` block of your JSON config (as shown in Phase 1).
3.  **Permission Hand-off:** Do NOT hardcode bash execution rules into `designer.ts`. OpenCode's SDK reads `impeccable/SKILL.md` and will natively prompt you (the user) to allow bash permissions on the first execution.

## Phase 4: Adding the Missing `Codexer` (Documenter) Agent

To fully elevate `codexer` from a custom user agent to a native, built-in OMO-slim agent, several files must be modified to satisfy TypeScript and schema requirements.

**Action Items:**
1.  **Create the Agent File:** Create `src/agents/codexer.ts` mirroring the structure of `fixer.ts`, focusing its prompt on documentation generation.
2.  **Register the Agent in Source:**
    *   **`src/agents/index.ts`:** Import and add `codexer: createCodexerAgent` to `SUBAGENT_FACTORIES`.
    *   **`src/config/constants.ts`:** Add `"codexer"` to `SUBAGENT_NAMES` and add a default model to `DEFAULT_MODELS`.
    *   **`src/agents/orchestrator.ts`:** Add `@codexer` to `AGENT_DESCRIPTIONS` explaining when to use it, and add to `VALIDATION_ROUTING`: `"- Route documentation tasks (READMEs, API specs, changelogs) to @codexer"`.
3.  **Schema Updates:** Run `bun run generate-schema` to update `oh-my-opencode-slim.schema.json`.

## Phase 5: Build and Link
After completing the source edits in Phase 4:
1.  Run `bun run build` inside `/home/arjun/oh-my-opencode-slim`.
2.  Update `/home/arjun/.config/opencode/opencode.json` to load your local fork:
    ```json
    "plugin": [
      "file:///home/arjun/oh-my-opencode-slim"
    ]
    ```