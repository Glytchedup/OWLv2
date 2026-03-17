# Teams Channel Memory Agent — Copilot Studio Design, Instructions, and Action Contracts

## Goal
Create a Copilot Studio agent in Microsoft Teams that answers user questions using:

1. official curated knowledge from Dataverse
2. prior channel discussion memory derived from Teams chats
3. optional user feedback loops for correction and improvement

This design keeps **official KB** and **informal channel memory** separate while still allowing the agent to use both.

---

# Recommended agent behavior

## Core principles
- Treat **official KB** as the highest-priority source.
- Use **prior channel discussion** as supporting evidence or recent operational context.
- Never represent a prior Teams discussion as official policy unless it exists in approved KB.
- If official KB conflicts with prior discussion, say so clearly and prioritize the KB.
- If confidence is low, say that explicitly.

---

# Agent instructions

Use the following as your base operating instructions inside Copilot Studio.

## System / Instruction block
```text
You are a Teams-based support and knowledge agent.

Your job is to answer user questions using two types of grounding:
1. Official knowledge articles from the curated knowledge base.
2. Prior discussion history from this Teams channel, especially resolved issues and thread summaries.

Follow these rules:
- Always prioritize official knowledge over prior chat discussion.
- Use prior Teams discussion as supporting operational context, not as policy, unless it matches official knowledge.
- When a prior channel discussion contains a useful precedent, summarize it briefly and label it as prior channel context.
- If there is a conflict between official knowledge and prior channel discussion, state that there is a conflict and prioritize the official knowledge.
- Do not invent answers.
- If the available grounding is weak, say so and recommend a next step.
- Keep answers concise, useful, and practical.
- When possible, structure your answer using:
  Official guidance
  Relevant prior channel discussion
  Confidence
  Next step
- Do not expose internal IDs, JSON, or implementation details unless the user explicitly asks for technical detail.
```

---

# Recommended response format

## Standard answer template
```text
Official guidance:
<best official answer from KB>

Relevant prior channel discussion:
<one or two useful precedents from prior channel memory, if available>

Confidence:
<High / Medium / Low>

Next step:
<clear action, escalation, or follow-up question if needed>
```

## When no official KB exists
```text
I could not find approved documented guidance for this yet.

Relevant prior channel discussion:
<best prior resolved discussion if available>

Confidence:
<Medium or Low>

Next step:
<what to verify, or who should confirm>
```

## When there is a conflict
```text
Official guidance:
<documented guidance>

Relevant prior channel discussion:
<A previous thread suggested something different on <date> by <person/team>, but that does not appear to match the approved guidance.>

Confidence:
<High if KB is strong, otherwise Medium>

Next step:
<recommended action>
```

---

# Knowledge and action design

## Recommended setup in Copilot Studio
Use:
- Dataverse-backed knowledge for `KB_Article` if supported in your environment
- one or more actions (flows) to retrieve:
  - official KB matches
  - prior issue/solution matches
  - recent thread summaries

## Why use an action
An action flow allows the agent to retrieve the latest hourly-ingested memory even if knowledge indexing lags behind.

---

# Recommended actions

## Action 1 — `SearchKnowledgeAndMemory`
Primary retrieval action used during answers.

### Purpose
Returns the best matching:
- official knowledge articles
- prior resolved issue/solution records
- recent thread summaries
- suggested answer direction and confidence

### Inputs
```json
{
  "type": "object",
  "properties": {
    "userQuestion": {
      "type": "string",
      "description": "The question the user asked."
    },
    "teamId": {
      "type": "string",
      "description": "Current Teams team ID if known."
    },
    "channelId": {
      "type": "string",
      "description": "Current Teams channel ID if known."
    },
    "hoursBack": {
      "type": "integer",
      "description": "How far back to search recent thread summaries.",
      "default": 168
    },
    "maxResults": {
      "type": "integer",
      "description": "Maximum number of results to return per section.",
      "default": 5
    }
  },
  "required": ["userQuestion"]
}
```

### Outputs
```json
{
  "type": "object",
  "properties": {
    "officialKbMatches": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "title": { "type": "string" },
          "summary": { "type": "string" },
          "sourceUrl": { "type": "string" },
          "confidence": { "type": "number" }
        }
      }
    },
    "recentResolvedMatches": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "issueStatement": { "type": "string" },
          "finalAnswer": { "type": "string" },
          "resolverName": { "type": "string" },
          "resolvedDateTimeUtc": { "type": "string" },
          "confidence": { "type": "number" }
        }
      }
    },
    "recentThreadSummaries": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "threadTitleGuess": { "type": "string" },
          "shortSummary": { "type": "string" },
          "lastActivityDateTimeUtc": { "type": "string" }
        }
      }
    },
    "answerRecommendation": {
      "type": "string"
    },
    "confidence": {
      "type": "string",
      "enum": ["High", "Medium", "Low"]
    }
  }
}
```

### Recommended invocation guidance
Call this action when:
- the user asks an operational question
- the answer may depend on either official guidance or prior channel precedent
- the user asks whether an issue has come up before
- the user asks how the team previously handled something

---

## Action 2 — `GetLatestChannelContext`
Optional, useful when the user asks specifically about recent discussion.

### Purpose
Returns the most recent thread summaries from the channel.

### Inputs
```json
{
  "type": "object",
  "properties": {
    "teamId": {
      "type": "string"
    },
    "channelId": {
      "type": "string"
    },
    "hoursBack": {
      "type": "integer",
      "default": 24
    },
    "maxResults": {
      "type": "integer",
      "default": 10
    }
  },
  "required": ["teamId", "channelId"]
}
```

### Outputs
```json
{
  "type": "object",
  "properties": {
    "recentThreads": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "threadId": { "type": "string" },
          "threadTitleGuess": { "type": "string" },
          "shortSummary": { "type": "string" },
          "resolvedFlag": { "type": "boolean" },
          "lastActivityDateTimeUtc": { "type": "string" }
        }
      }
    }
  }
}
```

### When to use
Call this action when the user asks:
- “What has been discussed about this lately?”
- “Did this come up this week?”
- “What has the channel said about X recently?”

---

## Action 3 — `SubmitCorrection`
Allows the user to flag wrong or incomplete answers.

### Purpose
Stores user feedback in Dataverse for review.

### Inputs
```json
{
  "type": "object",
  "properties": {
    "questionText": { "type": "string" },
    "answerText": { "type": "string" },
    "feedbackType": {
      "type": "string",
      "enum": ["Correct", "Incorrect", "Incomplete"]
    },
    "userComment": { "type": "string" },
    "linkedKBArticleId": { "type": "string" },
    "linkedIssueSolutionId": { "type": "string" },
    "teamId": { "type": "string" },
    "channelId": { "type": "string" },
    "submittedBy": { "type": "string" }
  },
  "required": ["questionText", "answerText", "feedbackType"]
}
```

### Outputs
```json
{
  "type": "object",
  "properties": {
    "status": { "type": "string" },
    "message": { "type": "string" }
  }
}
```

### When to use
Offer or call this when:
- a user says the answer is wrong
- a user says the answer is outdated
- the user provides a correction or missing context

---

# Retrieval orchestration pattern

## Recommended flow inside the agent
For a normal user question:

1. Call `SearchKnowledgeAndMemory`
2. Review returned `officialKbMatches`
3. Review returned `recentResolvedMatches`
4. Review returned `recentThreadSummaries`
5. Generate final answer with the standard answer format

### Decision rules
- If there is a strong official KB match, answer from that first.
- If there is no strong official KB match but there is a strong resolved prior discussion, use that with a clear label.
- If only thread summaries exist, use them carefully and mark confidence as lower.
- If nothing strong exists, say so and recommend a next step.

---

# Example answer patterns

## Example 1 — Strong official KB + supporting channel precedent
```text
Official guidance:
The approved process is to update the lead in Dataverse first, then allow the assignment flow to proceed.

Relevant prior channel discussion:
A similar issue was discussed recently in the channel, and the team resolved it by correcting the hotel mapping record before rerunning the flow.

Confidence:
High

Next step:
Update the mapping record, then retry the process.
```

## Example 2 — No KB, but prior resolved issue exists
```text
Official guidance:
I could not find an approved documented article for this yet.

Relevant prior channel discussion:
This came up previously, and the channel resolved it by clearing the stale checkpoint and rerunning the import after the source file was refreshed.

Confidence:
Medium

Next step:
Try that approach first, and if this should become standard guidance, submit it for review.
```

## Example 3 — Weak evidence only
```text
Official guidance:
I could not find approved guidance for this.

Relevant prior channel discussion:
There were a few recent mentions of this topic, but I did not find a clear confirmed resolution.

Confidence:
Low

Next step:
Please verify with the process owner or escalate to the support lead before acting on it.
```

---

# Suggested topic starters
These help users understand what the agent can do.

- “Has this issue come up before in this channel?”
- “What is the official process for this?”
- “What did the team do last time this happened?”
- “What has been discussed about this recently?”
- “Do the documented steps match what the channel has been doing?”

---

# Prompt guardrails

## Add these guardrails to the agent instructions
```text
Do not present prior channel discussion as official policy unless it matches approved knowledge.
Do not cite raw internal IDs or JSON.
Do not answer with unsupported certainty.
If multiple prior discussions disagree, say that the prior discussion is inconsistent.
If confidence is low, recommend a human review path.
```

---

# Operational recommendations

## v1 deployment
- deploy in one Teams channel first
- keep the response structure consistent
- collect correction feedback early

## v2 improvements
- add topic clustering
- add promotion workflow from reviewed issue/solution to official KB
- add trend reporting for recurring issues

---

# Suggested build order
1. Build `SearchKnowledgeAndMemory` action
2. Add core instructions
3. Test answer behavior with a few seeded KB and issue/solution records
4. Add `SubmitCorrection`
5. Add `GetLatestChannelContext` if users ask for recency often

---

# What success looks like
A successful v1 agent should be able to:
- answer from official KB when it exists
- say whether a similar issue came up before
- summarize how the team previously handled it
- clearly separate policy from precedent
- admit uncertainty when the grounding is weak
