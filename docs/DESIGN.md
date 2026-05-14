# Design Decisions

This document captures the architectural decisions behind `cardano-dev-skills`. It records what was decided, why, and what alternatives were considered.

## Decision 1: Two repos, not one

**Decision:** Keep the knowledge base (`cardano-dev-skills`) and the MCP server (`cardano-unified-mcp-server`) as separate repositories.

**Why:**
- **Security.** The knowledge base is pure content (YAML, Markdown). Anyone can fork, audit, and contribute without reviewing server code. Users downloading the skills repo don't execute arbitrary TypeScript.
- **Different audiences.** Content contributors (Cardano developers) don't need to understand MCP infrastructure. Infrastructure maintainers don't need to curate documentation.
- **Different cadences.** Skills and sources change with the ecosystem. Server code changes with protocol and tooling updates.

**Alternative considered:** Monorepo with both. Rejected because it couples content contributions to infrastructure reviews and creates a security concern for users who just want the knowledge base.

## Decision 2: Skills organized by developer workflow (flat directory layout)

**Decision:** Skills live flat under `skills/<name>/SKILL.md`. Logical categorization is conveyed by skill names and descriptions, not by directory structure.

**Why:** Developers think in terms of "what am I trying to do?" not "what category is this tool in?" A developer building an NFT marketplace needs `write-validator` + `build-transaction` + `connect-wallet` — these map to workflows, not to the registry's source categories. Flat layout also matches the Claude Code plugin discovery contract, which expects `skills/<name>/SKILL.md` and doesn't traverse arbitrary nesting.

**Alternative considered:** Mirror the registry's source categories (infrastructure, smart-contracts, sdk, standards, governance, scaling, testing, oracles) as subdirectories. Rejected because skills like `build-transaction` span multiple categories, and the plugin discovery contract doesn't require it.

## Decision 3: YAML registry, not TypeScript

**Decision:** The canonical source list is `registry/sources.yaml`, not a TypeScript file.

**Why:** Lower contribution barrier. A Cardano developer who wants to add a new project doesn't need to know TypeScript or the MCP server's type system. YAML is universally readable and editable. A sync script generates the TypeScript for the MCP server.

**Sync flow:**
1. Community PRs update `sources.yaml`
2. CI validates the YAML schema
3. MCP server maintainer runs `scripts/sync-sources.sh` to regenerate `sources.ts`

## Decision 4: Skills are standalone

**Decision:** Skills must work without the MCP server. No skill references `search_docs` or any MCP-specific tool.

**Why:**
- Users who install just the skills plugin (without the MCP server) should still get useful guidance.
- Skills that depend on MCP break for Codex users, Cursor users, or anyone without the server running.
- Skills guide the workflow; the MCP server provides supplementary data. If both are available, the experience is richer but not required.

**How it works:** Skills use `allowed-tools: Read Grep Glob` — Claude searches local documentation, the user's codebase, or its own knowledge. If the MCP server is also connected, Claude can additionally call `search_docs` for deeper lookups, but the skill doesn't require it.

## Decision 5: Progressive disclosure

**Decision:** SKILL.md files are capped at 500 lines. Deep reference content goes in `references/` subdirectories, one level deep only.

**Why:**
- **Context budget.** At session start, Claude loads only the name + description of each skill (~100 tokens per skill). When a skill activates, the full SKILL.md loads (~2,000 tokens). References load only on demand. This keeps context usage manageable.
- **Maintainability.** A 500-line file is reviewable in a single PR. A 2,000-line file is not.
- **Trail of Bits pattern.** Their production skills follow this exact structure and it works at scale (35+ plugins, 100+ skills).

## Decision 6: Agent Skills standard compliance

**Decision:** Follow the Agent Skills open standard for SKILL.md format — YAML frontmatter with `name`, `description`, `allowed-tools`, and structured markdown body.

**Why:**
- Compatible with Claude Code plugins (`.claude-plugin/` + `skills/`)
- Compatible with Codex (`.agents/skills/` symlink)
- Future-proof for other tools that adopt the standard
- Established quality standards (naming conventions, description requirements, section structure)

**Cross-tool compatibility:** Symlinks handle multi-tool support without file duplication:
- `.claude/skills` → `../skills` (Claude Code project-level discovery)
- `.agents/skills` → `../skills` (Codex discovery)
- Plugin installation (`/plugin add`) uses `skills/` directly

## Decision 7: Single source of truth flow

**Decision:** Content flows one direction: `cardano-dev-skills` → `cardano-unified-mcp-server`.

```
cardano-dev-skills/                    WRITES
  registry/sources.yaml          ──→   MCP sources.ts (via sync script)
  skills/*/SKILL.md              ──→   MCP prompts (via loader at startup)

cardano-unified-mcp-server/            READS
  src/config/sources.ts          ←──   generated from sources.yaml
  src/tools/prompts.ts           ←──   reads SKILL.md content
```

**Why:** Eliminates drift. There is exactly one place to update a source entry or a skill's workflow. The MCP server is a consumer, not a peer.

## Decision 8: Content authored from scratch

**Decision:** Skill content is written by humans (or AI-assisted), not extracted from the MCP server's chunked/embedded data.

**Why:**
- MCP chunks are optimized for retrieval, not for teaching. They're fragments, not workflows.
- Skills need behavioral guidance ("when to use X over Y", "check for Z before doing W") that doesn't exist in raw documentation.
- Freshness: skills are written against current best practices, not against whatever was last indexed.

## Decision 9: Lifecycle automation shipped incrementally

**Decision:** Automate the refresh lifecycle in small, reviewable steps rather than building a single big system.

**Shipped:**
- **Weekly upstream refresh** (`.github/workflows/refresh-docs.yml`) — every Monday 06:00 UTC, fetches all sources, opens a PR with the diff. Human review + merge before content lands on `main`.
- **Schema validation** (`.github/workflows/validate.yml`) — runs on every PR touching `skills/**` or `registry/**`.
- **Manifest self-healing** — `scripts/_fetch_docs.py` derives `.manifest.yaml` from disk state after every fetch (partial or full), so the manifest can't drift.
- **Doc-count auto-derivation** — `scripts/update-doc-counts.sh` rewrites sentinels in CLAUDE.md and README.md from disk state. CI runs `--check` to fail PRs on drift.

**Planned (tracked, not built):**
- `UserPromptSubmit` hook that auto-injects "consult bundled docs first" guidance on Cardano-keyword-matched prompts, with local usage telemetry under `~/.cardano-dev-skills/usage.log`.
- PR-time source-build check: when `registry/sources.yaml` changes, CI fetches the touched source(s) and verifies the clone + glob patterns produce files.
- AI-powered governance review on PRs touching `registry/`, `skills/`, `hooks/`, or `scripts/` — Claude reads the diff and the rules from CONTRIBUTING.md, posts a verdict comment (advisory, not blocking).

These additions follow the principle: ship small, observe, iterate.

## Decision 10: Documentation governance

**Decision:** Docs in this repo (CLAUDE.md, README.md, DESIGN.md, CONTRIBUTING.md) must reflect current state. Externally-observable changes (counts, capabilities, structure, interfaces) require a doc update in the same PR. Internal tweaks (refactors, typo fixes) do not.

**Why:** Stale READMEs are the most common rot in tooling repos. They mislead new contributors, make the project look abandoned, and damage credibility — particularly when the goal is broader adoption.

**Mechanism (two layers):**

1. **Auto-derived counts** for the most rot-prone numbers (skill count, source count). Inline HTML-comment sentinels (`<!-- COUNT:skills -->14<!-- /COUNT:skills -->`) are rewritten by `scripts/update-doc-counts.sh`. CI runs `--check` on every PR.
2. **Per-change-type checklist** in `CLAUDE.md` mapping change types (new skill, new source, new schema field, new script, new hook) to the docs that must be updated. Enforced by reviewer judgement and (planned) AI governance review.

**Alternative considered:** Generate the entire README from a template + computed values. Rejected because narrative sections ("Why this exists", "How to set the Cardano context") need human prose, and a template-only approach makes those harder to evolve.

## Decision 11: Hook strategy

**Decision:** Use Claude Code's hook surface to make the plugin behave well by default, without requiring users to explicitly invoke skills.

**Shipped:**
- `SessionStart` (`hooks/check-docs.sh`) — reports doc freshness on every session start in any directory.

**In active development (separate session):**
- `UserPromptSubmit` (`hooks/cardano-router.sh`) — keyword-matches the user's prompt against a Cardano-specific term list and injects an `additionalContext` reminder to consult skills/docs first.
- `PostToolUse` (`hooks/log-tool.sh`) — logs which bundled docs/skills get consulted per session, to a local `~/.cardano-dev-skills/usage.log`. Backs `scripts/usage-report.sh`.

**Why hooks (not just skills):** Skill matching depends on description-based heuristics — vague prompts often miss. Hooks fire unconditionally on every prompt and provide a reliable nudge. Skills remain the right surface for *how* to do something; hooks ensure they get consulted in the first place.
