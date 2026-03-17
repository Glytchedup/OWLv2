# CLAUDE.md

## Project Overview

OWLv2 is a **Microsoft Teams Channel Memory Agent** built in Copilot Studio. It answers operational questions using two grounded sources:

1. **Official curated knowledge** from Dataverse (`KB_Article` table)
2. **Prior channel discussion memory** (thread summaries and issue/solution extraction)

The agent prioritizes official guidance, uses channel history as supporting context, clearly communicates confidence levels, and offers escalation paths when grounding is weak.

## Repository Structure

```
OWLv2/
├── agent.mcs.yml                 # Agent metadata and system instructions
├── settings.mcs.yml              # AI model and capability settings
├── icon.png                      # Agent icon
├── topics/                       # Conversation flow topics (13 YAML files)
│   ├── Search.mcs.yml            # Main knowledge retrieval topic
│   ├── Escalate.mcs.yml          # Human escalation handling
│   ├── Fallback.mcs.yml          # Low-confidence handling
│   ├── Greeting.mcs.yml          # Greeting responses
│   ├── ConversationStart.mcs.yml
│   ├── EndofConversation.mcs.yml
│   ├── OnError.mcs.yml           # Error logging
│   └── ...                       # Other system/custom topics
├── Build Plans/                  # Design documentation
│   ├── teams-channel-memory-agent-copilot-studio-design.md
│   ├── teams-channel-memory-agent-dataverse-schema.md
│   └── teams-channel-memory-agent-power-automate-plan.md
├── docs/
│   ├── tenant-implementation-runbook.md
│   ├── claude-code-plan.md       # Automated implementation plan (Claude Code)
│   └── manual-implementation-plan.md  # Manual tenant setup plan
├── .github/ISSUE_TEMPLATE/       # Bug report and feature request templates
├── README.md
├── CONTRIBUTING.md
├── SECURITY.md
└── CLAUDE.md                     # This file
```

## Tech Stack

- **Microsoft Copilot Studio** — agent framework and runtime
- **Microsoft Dataverse** — data storage (7 tables: KB_Article, Channel_Message_Raw, Channel_Thread_Summary, Channel_Issue_Solution, etc.)
- **Microsoft Power Automate** — workflow orchestration (4 flows: ingest, search, correction, promotion)
- **Microsoft Teams** — deployment surface
- **YAML (.mcs.yml)** — all agent configuration uses Copilot Studio Adaptive Dialog YAML

There is **no traditional build system, package manager, or test framework**. This is a configuration/design repository for a cloud-native Power Platform solution.

## Development Workflow

### Branching

- `main` is the protected release branch
- Feature branches: `feat/<short-description>`, `fix/<short-description>`, `chore/<short-description>`

### Commit Messages

Use conventional-style, action-oriented messages with scope:

```
feat(agent): enforce official-guidance response format
feat(topic): add low-confidence escalation path
docs: add implementation runbook
```

### Pull Requests

- Keep scope to one implementation slice per PR
- Include: what changed, why, validation performed, rollout risk, rollback notes
- Require at least one reviewer approval

### Validation Checklist

Before opening a PR:

1. Verify all YAML parses without errors
2. Confirm topic behavior impact is documented
3. Confirm no credentials, tenant IDs, or secrets are committed

## Key Conventions

### YAML Topics

- Files use `.mcs.yml` extension with **PascalCase** names (e.g., `ConversationStart.mcs.yml`)
- Topics use **AdaptiveDialog** structure with triggers: `OnRecognizedIntent`, `OnUnknownIntent`, `OnError`
- Actions include: `SendActivity`, `Question`, `ConditionGroup`, `BeginDialog`, `SearchAndSummarizeContent`

### Variable Naming

- Use **camelCase** with descriptive prefixes: `varTeamId`, `varLookbackStartUtc`

### Dataverse Tables

- Use **PascalCase with underscores**: `KB_Article`, `Channel_Message_Raw`, `Channel_Thread_Summary`

### Response Format

The agent follows a structured response contract:

```
Official guidance:
<KB content>

Relevant prior channel discussion:
<channel history context>

Confidence:
<High / Medium / Low>

Next step:
<action / escalation / question>
```

### Scope Boundaries

- Keep official KB policy and prior channel memory **clearly separated**
- Never represent informal channel precedent as approved process
- Mark low-confidence answers clearly and always provide an escalation path
- AI-extracted content must be labeled as "AI Extracted" and flagged for human review

## Security

- Teams channel data is classified as **internal/sensitive**
- Never commit credentials, tenant IDs, or secrets
- See `SECURITY.md` for full data handling policy and incident procedures

## Agent Configuration Summary

### AI Settings
- `useModelKnowledge: false` — agent answers ONLY from grounded sources, never from general model knowledge
- `isSemanticSearchEnabled: true` — enables Dataverse knowledge search
- `contentModeration: High`
- `webBrowsing` and `codeInterpreter` are disabled (not relevant for this operational Q&A agent)

### Source Priority Order
1. `KB_Article` (official curated knowledge) — highest priority
2. `Channel_Issue_Solution` (prior resolved issues from channel threads)
3. `Channel_Thread_Summary` (recent thread summaries)
4. `Channel_Message_Raw` (raw messages — detail fallback only)

### Confidence Levels
- **High**: Strong official KB match exists
- **Medium**: No KB match, but confirmed resolved issue/solution from channel history
- **Low**: Only unconfirmed thread summaries or nothing found

## Implementation Plans

- **Claude Code plan** (automated): `docs/claude-code-plan.md` — covers all YAML/config optimization
- **Manual plan** (requires tenant access): `docs/manual-implementation-plan.md` — Dataverse schema, Power Automate flows, Copilot Studio wiring, pilot validation

## Key Design Documents

- **Agent design**: `Build Plans/teams-channel-memory-agent-copilot-studio-design.md`
- **Dataverse schema**: `Build Plans/teams-channel-memory-agent-dataverse-schema.md`
- **Power Automate flows**: `Build Plans/teams-channel-memory-agent-power-automate-plan.md`
- **Deployment guide**: `docs/tenant-implementation-runbook.md`
