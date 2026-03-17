# OWLv2 Teams Channel Memory Agent

OWLv2 is a Copilot Studio agent for Microsoft Teams that answers operational questions using:

1. Official curated knowledge from Dataverse
2. Prior channel discussion memory (thread summaries and issue/solution extraction)

## Objectives

- Prioritize official guidance from `KB_Article`
- Use prior channel discussion as supporting context
- Clearly communicate confidence and next steps
- Capture corrections and promote validated learnings into official KB

## Current Architecture

- Agent project: `OWLv2/`
- Design plans: `Build Plans/`
- Core retrieval behavior (target state):
  1. `KB_Article`
  2. `Channel_Issue_Solution`
  3. `Channel_Thread_Summary`
  4. `Channel_Message_Raw` (detail fallback only)

## Implementation Tracks

1. Dataverse schema deployment (7 tables)
2. Power Automate flows (ingest, search, correction, promotion)
3. Copilot Studio topic wiring and response contract alignment
4. Pilot rollout, governance, and hardening

## Local Development

This repository mainly contains Copilot Studio YAML and design artifacts.

Recommended workflow:

1. Create a feature branch
2. Make scoped topic/config changes
3. Validate YAML integrity
4. Open PR with test notes and rollout impact

## Security and Data Handling

See `SECURITY.md` for data classification and handling expectations.
