# Claude Code Implementation Plan

This plan covers everything that can be built and optimized directly in the repository YAML configuration files. All items below are executed by Claude Code.

---

## Phase 1: Agent Core Configuration (`agent.mcs.yml`)

### 1.1 Rewrite system instructions
The current instructions are functional but miss several design-doc requirements:
- Add explicit guardrails from the design doc (no unsupported certainty, handle conflicting sources, label AI-extracted content)
- Add the three response templates inline (standard, no-KB, conflict)
- Add correction offer behavior (prompt user to submit feedback when confidence is low)
- Add scope boundary enforcement (never present channel precedent as official policy)
- Add explicit retrieval priority order (KB > Issue/Solution > Thread Summary > Raw)

### 1.2 Improve agent description
- Make the `description` field more specific about capabilities for better discoverability in Teams

### 1.3 Optimize AI settings
- Disable `webBrowsing` (agent should only use Dataverse grounding, not the open web)
- Disable `codeInterpreter` (not relevant for an operational Q&A agent)
- These reduce hallucination surface and keep answers grounded

---

## Phase 2: Agent Settings (`settings.mcs.yml`)

### 2.1 Tighten AI settings
- Set `useModelKnowledge: false` — the agent should ONLY answer from grounded sources, never from general model knowledge
- Keep `isSemanticSearchEnabled: true` for Dataverse knowledge search
- Keep `contentModeration: High`
- Keep `isFileAnalysisEnabled: true` for attachment context

---

## Phase 3: Topic Optimization

### 3.1 ConversationStart — Improve welcome message
- Reference agent name and capabilities clearly
- Set expectations about grounding sources (official KB + channel memory)
- Add suggested starter questions from the design doc

### 3.2 Search (Conversational Boosting) — Enhance the core retrieval topic
- Improve the `description` to match the design document terminology
- Improve the low-confidence message to match the design doc response format
- Keep `SearchAndSummarizeContent` as the primary action (Power Automate actions will be wired manually later)

### 3.3 Fallback — Improve progressive disclosure
- First attempt: ask for more detail (product area, error text, what they're trying to do)
- Second attempt: offer to search with different phrasing
- Third attempt: route to Escalate topic
- Improve the messages to be more helpful and specific

### 3.4 Escalate — Enhance with design-doc patterns
- Already well-built; minor message improvements
- Add structured ticket template that matches the design doc's correction/feedback pattern
- Clarify next steps more explicitly

### 3.5 EndofConversation — Tighten feedback loop
- Improve feedback prompts to align with the SubmitCorrection action design
- Make the feedback offer more specific about what happens with their input

### 3.6 Greeting — Add capability overview
- After greeting, briefly state what the agent can help with
- Reference topic starters from the design doc

### 3.7 OnError — Improve user-facing message
- Make the production error message less technical
- Keep telemetry logging as-is (well designed)

### 3.8 ThankYou — Add continuation prompt
- After acknowledging thanks, ask if they need anything else

### 3.9 Goodbye — Keep as-is
- Already well structured with confirmation flow

### 3.10 Other system topics (Signin, StartOver, ResetConversation, MultipleTopicsMatched)
- Keep as-is — these are standard system topics with correct behavior

---

## Phase 4: Update CLAUDE.md
- Update to reflect all changes made

---

## Execution order
1. `agent.mcs.yml` — instructions and metadata
2. `settings.mcs.yml` — AI behavior constraints
3. Topics in order: Search, Fallback, Escalate, ConversationStart, Greeting, EndofConversation, OnError, ThankYou
4. CLAUDE.md update
5. Commit and push
