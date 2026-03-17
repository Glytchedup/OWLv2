# Copilot Studio Compliance Reference

Use this file to audit OWLv2 agent configuration after every update. All limits and best practices are sourced from Microsoft documentation as of March 2026.

---

## Character Limits

| Field | Hard Limit | Safe Limit | Notes |
|---|---|---|---|
| **Agent display name** | 42 chars | 42 chars | Keep short and descriptive |
| **Agent description** (`mcs.metadata.description`) | 1,024 chars | 800 chars | Used by generative orchestration for routing |
| **Agent instructions** | **8,000 chars** | 6,000 chars | Official documented limit is 8,000 chars (~1,500 words). A legacy bug report mentioned a 2,000-char post-edit limit but this is NOT the current behavior. Use the full budget; keep under 6,000 for clarity |
| **Topic `componentName`** | Not documented | ~50 chars | PascalCase, descriptive |
| **Topic `description`** | ~1,000 chars | 500 chars | Used by generative orchestration to decide which topic to invoke — keep concise and specific |
| **SendActivity message** | 10,240 chars | 4,000 chars | Hard limit is 10,240 chars per message; AI-generated responses typically cap at ~4,000; ACS channels limited to 28 KB total payload |
| **Node names** | 500 chars | 200 chars | Internal labels |
| **Variable names** | Not documented | ~100 chars | Use camelCase with prefixes (`varTeamId`) |
| **Glossary term descriptions** | 1,000 chars | 500 chars | If using glossary |

## Topic Limits

| Limit | Value | Notes |
|---|---|---|
| **Max topics per agent** | 1,000 | Use knowledge sources instead of one-topic-per-FAQ |
| **Max nodes per topic** | Not hard-documented | Keep topics small; Microsoft recommends many small topics over few large ones |
| **Trigger phrases per topic** | No hard max | **Recommended: 5–10 phrases**; can bulk-upload via file (max 3 MB) |
| **Trigger phrase length** | Short phrases | Avoid full sentences; brevity improves NLU matching |

## Knowledge Source Limits

| Limit | Value |
|---|---|
| **Max knowledge objects per agent** | 500 (files, folders, articles, websites) |
| **SharePoint exemption** | 500-file limit does NOT apply to SharePoint |
| **SharePoint/OneDrive file uploads** | 1,000 files per agent |
| **Max source types simultaneously** | 5 |
| **Public website sources (generative orchestration)** | 25 |
| **Public website sources (classic orchestration)** | 4 |
| **Generative answers node (topic-level)** | 4 sources (uses classic orchestration regardless of agent setting) |
| **File size (SharePoint/connectors)** | 512 MB |
| **File size (SharePoint with M365 Copilot license)** | 200 MB |
| **File size (SharePoint without M365 Copilot license)** | 7 MB |
| **Connector payload (public cloud)** | 5 MB |
| **Connector payload (GCC)** | 450 KB |
| **Folder depth** | 10 levels of subfolders |
| **SharePoint lists per session** | 15 |
| **SharePoint lookup columns in default view** | 12 max |

## Generative Orchestration Limits

| Limit | Value | Notes |
|---|---|---|
| **Max tools per agent** | 128 | Includes actions, flows, and plugins |
| **Recommended tools per agent** | 25–30 | Beyond this, routing accuracy degrades |
| **Conversation history turns referenced** | 10 | Agent considers last 10 turns for context |
| **Language support** | English only | For generative orchestration mode |
| **Agent chaining depth** | 1 level | Orchestrator → sub-agent; no deeper nesting |

## Rate Limits

| Environment | RPM | RPH |
|---|---|---|
| **Trial / Developer** | 10 | 200 |
| **Paid plans** | Configurable | Contact admin for increases |

Throttling error codes: `GenAIToolPlannerRateLimitReached`, `GenAISearchandSummarizeRateLimitReached`, `OpenAIRateLimitReached`

## Runtime and Publishing Limits

| Limit | Value |
|---|---|
| **Prompt execution timeout** | 100 seconds |
| **Flow response data to agent** | 1 MB per action |
| **Power Automate flow inactivity timeout** | 2 minutes |
| **ACS channel message size** | 28 KB |
| **Agent icon** | PNG, < 72 KB, max 192x192 px |
| **Copilot credits (M365 license)** | 25,000/month shared across users |

## Content Moderation

| Level | Behavior |
|---|---|
| **High** (our setting) | Most precise, fewest responses, strictest filtering |
| **Medium** | Balanced precision and coverage |
| **Low** | Most creative, higher hallucination risk |

Azure AI Content Safety protections remain active at all levels. Content moderation slider only applies to managed models.

---

## Instructions Best Practices

### Do
- Use precise, specific verbs: "ask", "search", "send", "check", "use"
- Break workflows into modular steps with Goal, Action, and Transition
- Use sections and bullets to organize parallel tasks
- Define organization-specific terms the agent needs to understand
- Reference only actually configured tools and knowledge sources
- Test incrementally: remove all instructions, add back one at a time, test between each
- Use up to **8,000 characters** (the official limit); aim for under 6,000 for clarity and processing efficiency

### Avoid
- Writing what the agent should NOT do (focus on what it SHOULD do)
- Numbered lists that imply sequence for parallel tasks
- Repeating similar instructions (increases token costs and confuses the model)
- Long verbose instructions — brief and direct is more reliable
- Referencing tools or knowledge sources that are not actually configured
- Using markdown headers or formatting that the instruction parser may not render

---

## Topic Design Best Practices

### Structure
- Keep topics small and focused on one task
- Use redirects (`BeginDialog`) to chain topics rather than building deep nesting
- Pass variables between topics using `init:` prefix for topic-scoped variables
- Use `System.FallbackCount` for progressive fallback behavior

### Trigger Phrases
- Provide **5–10 trigger phrases** per topic
- Use short, natural phrases (not full sentences)
- Avoid overlap between topics — test with MultipleTopicsMatched
- Include common misspellings and abbreviations

### Messages
- Keep SendActivity messages under 2,000 characters
- Use markdown formatting sparingly (bold for key terms only)
- Avoid emojis unless your audience expects them
- Always provide a clear next step or continuation prompt

---

## YAML / Adaptive Dialog Constraints

| Constraint | Detail |
|---|---|
| **Variable init prefix** | Question action variables MUST use `init:` prefix (e.g., `init:Topic.Confirm`) |
| **Text concatenation** | Use `Concatenate()` function, NOT the `&` operator |
| **Variable scope** | Use `Global` or `Topic` scope; `Dialog` scope can cause recognition errors |
| **Topic names** | Periods (`.`) in topic names block solution export |
| **Complex GotoAction** | Can cause silent failures; prefer `BeginDialog` redirects |

---

## Common Publishing Failures

1. **DLP policy violations** — Data Loss Prevention policies blocking required channels (Direct Line, etc.)
2. **Broken Power Automate flows** — expired connections, invalid tokens
3. **Generative AI blocked** — admin must enable "Publish Copilots with AI features" in Power Platform admin center
4. **Knowledge source limits exceeded** — too many Bing/website sources
5. **YAML/topic validation errors** — broken variable references, invalid Power Fx expressions
6. **Teams channel sync errors** — mismatched bot/app IDs
7. **Authentication not configured** — environment requires sign-in but agent doesn't have it set up
8. **No active channel** — at least one active channel is required to publish
9. **Bing source limit** — max 4 Bing sources; exceeding blocks publish
10. **Sub-agent not published** — must publish sub-agents before the orchestrator agent

---

## OWLv2 Audit Checklist

Run this checklist after every agent update before publishing:

### Agent Core
- [ ] `displayName` is ≤ 42 characters
- [ ] `mcs.metadata.description` is ≤ 1,024 characters
- [ ] `instructions` is ≤ **8,000 characters** (official limit; aim for ≤ 6,000 for clarity)
- [ ] Instructions use positive phrasing (what TO do, not what NOT to do)
- [ ] Instructions do not reference tools/actions that are not configured
- [ ] Instructions do not repeat similar guidance in multiple places
- [ ] `useModelKnowledge` is `false` (grounded-only answers)
- [ ] `contentModeration` is `High`
- [ ] `webBrowsing` is disabled (or omitted)
- [ ] `codeInterpreter` is disabled (or omitted)
- [ ] Agent icon is PNG, < 72 KB, ≤ 192x192 px

### Topics
- [ ] Every topic has a `componentName` (≤ 50 chars)
- [ ] Every custom topic has a `description` (≤ 500 chars, specific enough for orchestration routing)
- [ ] Trigger phrases: 5–10 per intent topic, short and natural
- [ ] No trigger phrase overlap between topics
- [ ] All `SendActivity` messages are ≤ 2,000 characters
- [ ] All `BeginDialog` references point to valid topic IDs
- [ ] Low-confidence paths always offer escalation
- [ ] Error topic logs to telemetry and shows user-friendly message

### Knowledge Sources
- [ ] Dataverse knowledge source configured for `KB_Article`
- [ ] Searchable columns specified: `Title`, `BodyPlainText`, `ShortSummary`, `Keywords`
- [ ] Total knowledge objects ≤ 500

### Security
- [ ] No credentials, tenant IDs, or secrets in YAML files
- [ ] Authentication mode is `Integrated` with `GroupMembership` access control
- [ ] `authenticationTrigger` is `Always`

### Flows (verify in Power Platform)
- [ ] All registered actions have valid, active Power Automate flows
- [ ] Flow connections are not expired
- [ ] Flow response payload ≤ 1 MB

---

## Automated Audit Script

Count characters for all critical fields:

```bash
python3 -c "
import yaml, os, glob

# Agent core
with open('OWLv2/agent.mcs.yml') as f:
    agent = yaml.safe_load(f)

name = agent.get('displayName', '')
desc = agent.get('mcs.metadata', {}).get('description', '')
inst = agent.get('instructions', '')

print('=== AGENT CORE ===')
print(f'displayName: {len(name)}/42 chars', '  PASS' if len(name) <= 42 else '  FAIL')
print(f'description: {len(desc)}/1024 chars', '  PASS' if len(desc) <= 1024 else '  FAIL')
print(f'instructions: {len(inst)}/8000 chars', '  PASS' if len(inst) <= 8000 else '  FAIL (exceeds limit)')
print()

# Topics
print('=== TOPICS ===')
for f in sorted(glob.glob('OWLv2/topics/*.mcs.yml')):
    with open(f) as fh:
        data = yaml.safe_load(fh)
    fname = os.path.basename(f)
    meta = data.get('mcs.metadata', {})
    tdesc = meta.get('description', '')
    triggers = data.get('beginDialog', {}).get('intent', {}).get('triggerQueries', [])

    def find_max_activity(obj):
        mx = 0
        if isinstance(obj, dict):
            if obj.get('kind') == 'SendActivity':
                act = obj.get('activity', '')
                if isinstance(act, str): mx = max(mx, len(act))
                elif isinstance(act, dict):
                    for t in (act.get('text', []) if isinstance(act.get('text'), list) else [act.get('text', '')]):
                        mx = max(mx, len(str(t)))
            for v in obj.values(): mx = max(mx, find_max_activity(v))
        elif isinstance(obj, list):
            for i in obj: mx = max(mx, find_max_activity(i))
        return mx

    max_msg = find_max_activity(data)
    status = 'PASS'
    issues = []
    if len(tdesc) > 500: issues.append(f'desc {len(tdesc)}/500')
    if max_msg > 2000: issues.append(f'msg {max_msg}/2000')
    if triggers and (len(triggers) < 5 or len(triggers) > 10):
        issues.append(f'triggers {len(triggers)} (rec: 5-10)')
    if issues: status = 'WARN: ' + ', '.join(issues)
    print(f'{fname}: {status}')
"
```

---

## Sources

- [Quotas and limits — Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/requirements-quotas)
- [Write agent instructions](https://learn.microsoft.com/en-us/microsoft-copilot-studio/authoring-instructions)
- [Generative orchestration guidance](https://learn.microsoft.com/en-us/microsoft-copilot-studio/guidance/generative-mode-guidance)
- [Topic authoring best practices](https://learn.microsoft.com/en-us/microsoft-copilot-studio/guidance/topic-authoring-best-practices)
- [Trigger phrases best practices](https://learn.microsoft.com/en-us/microsoft-copilot-studio/guidance/trigger-phrases-best-practices)
- [Knowledge sources summary](https://learn.microsoft.com/en-us/microsoft-copilot-studio/knowledge-copilot-studio)
- [Content moderation settings](https://learn.microsoft.com/en-us/microsoft-copilot-studio/prompt-model-settings)
- [Instructions character limit discussion — Microsoft Community](https://techcommunity.microsoft.com/discussions/microsoft365copilot/ai-agent-instructions-character-limit-in-studio-license/4468072)
- [Copilot Studio instructions issues — Microsoft Q&A](https://learn.microsoft.com/en-us/answers/questions/4419038/copilot-studio-instructions-issues)
- [Generative orchestration capabilities](https://learn.microsoft.com/en-us/microsoft-copilot-studio/advanced-generative-actions)
- [Resolve throttling errors](https://learn.microsoft.com/en-us/troubleshoot/power-platform/copilot-studio/licensing/throttling-errors-agents)
- [Optimize prompts with custom instructions](https://learn.microsoft.com/en-us/microsoft-copilot-studio/guidance/optimize-prompts-custom-instructions)
- [Prompts performance and execution](https://learn.microsoft.com/en-us/microsoft-copilot-studio/prompts-performance-execution)
- [Code editor in topics](https://learn.microsoft.com/en-us/microsoft-copilot-studio/guidance/topics-code-editor)
- [Create and delete agents](https://learn.microsoft.com/en-us/microsoft-copilot-studio/authoring-first-bot)
