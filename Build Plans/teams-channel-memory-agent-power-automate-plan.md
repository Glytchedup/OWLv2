# Teams Channel Memory Agent — Power Automate Implementation Plan

## Goal
Extract Teams channel discussions **once an hour**, normalize them, summarize changed threads, extract issue/solution memory, and store the results in Dataverse for Copilot Studio.

## Assumptions
- You want to support one or more Teams channels.
- The agent will answer using:
  - official KB records from Dataverse
  - prior channel discussion memory from Dataverse
- You do **not** want a design that depends on you creating your own Azure AD app.

---

# Flow set overview

## Flow 1 — `Teams Channel Memory - Hourly Ingest`
Main scheduled ingestion flow.

## Flow 2 — `Teams Channel Memory - Search Knowledge and Memory`
Action flow called by Copilot Studio to retrieve grounding context.

## Flow 3 — `Teams Channel Memory - Submit Correction`
Optional action flow to capture user feedback/corrections.

## Flow 4 — `Teams Channel Memory - Promote to KB`
Optional admin/reviewer flow to convert reviewed issue/solution records into official KB articles.

---

# Flow 1 — Hourly Ingest

## Trigger
**Recurrence**
- Frequency: Hour
- Interval: 1

## Purpose
On each run:
1. identify monitored channels
2. pull messages since the last successful checkpoint
3. upsert raw messages
4. rebuild summaries for changed threads
5. extract issue/solution records
6. update checkpoint

---

## Step-by-step design

### 1. Trigger: Recurrence
Configure:
- every 1 hour
- optionally offset minutes to avoid top-of-hour congestion

Suggested name:
- `Recurrence - Hourly`

---

### 2. List active monitored channels
Action:
- Dataverse `List rows` from `Channel_Ingestion_Control`

Filter:
- `IsActive eq true`

Select:
- TeamId
- ChannelId
- TeamName
- ChannelName
- LastSuccessfulIngestUtc
- HoursBackOnFirstRun

Suggested name:
- `Get active channel controls`

---

### 3. Apply to each channel
Action:
- `Apply to each` over channel control rows

Inside the loop, create variables:

#### Variables
- `varTeamId`
- `varChannelId`
- `varLastCheckpointUtc`
- `varRunStartUtc`
- `varLookbackStartUtc`
- `varIngestionRunId`

Suggested expressions:
- `varRunStartUtc` = `utcNow()`
- `varIngestionRunId` = `guid()`

If `LastSuccessfulIngestUtc` is blank:
- set `varLookbackStartUtc` to `addHours(utcNow(), -coalesce(HoursBackOnFirstRun, 24))`
Else:
- set `varLookbackStartUtc` to `LastSuccessfulIngestUtc`

---

### 4. Retrieve channel messages since checkpoint
There are a few possible implementations depending on what is available in your environment.

## Preferred v1 approach
Use whatever supported Teams/Power Platform retrieval approach is available in your tenant to get channel messages. If direct historical delta retrieval is limited in your connector surface, use an HTTP/Graph pattern only if your organization already has an approved path.

### Output needed
For each returned message:
- message ID
- created time
- last modified time
- sender
- body
- thread/root relationship
- attachment metadata
- mentions
- link if available

Suggested internal object shape:
```json
{
  "messageId": "",
  "teamId": "",
  "channelId": "",
  "threadId": "",
  "replyToMessageId": "",
  "isRootMessage": true,
  "fromDisplayName": "",
  "fromUserId": "",
  "createdDateTimeUtc": "",
  "lastModifiedDateTimeUtc": "",
  "messageTextHtml": "",
  "messageTextPlain": "",
  "hasAttachments": false,
  "attachmentMetadataJson": "",
  "mentionsJson": "",
  "messageLink": ""
}
```

### Important implementation note
If you cannot reliably retrieve only changed messages:
- retrieve a bounded recent window, such as last 2–4 hours
- upsert by `MessageId`
- let Dataverse dedupe on business key

That still works well on an hourly cadence.

---

### 5. Normalize message text
For each message:
- strip HTML tags
- decode common HTML entities
- flatten line breaks
- resolve mentions into readable names when possible
- trim extra whitespace
- optionally remove signatures/bot boilerplate

Suggested actions:
- `Compose - Clean HTML`
- `Compose - Plain Text`
- `Compose - Normalized Text`

You may use:
- built-in expressions for replace/trim
- AI Builder or model-based cleanup only if truly needed

### Example cleanup pattern
- replace `<br>` and paragraph tags with line breaks
- remove remaining HTML tags
- normalize whitespace

---

### 6. Upsert raw messages into Dataverse
Table:
- `Channel_Message_Raw`

Use:
- `Add a new row` for new records
- `Update a row` for existing records

Best pattern:
1. `List rows` filtering by `MessageId`
2. if found → update
3. else → create

If volume grows later, use alternate keys and an upsert-capable pattern if available in your environment.

Columns to populate:
- MessageId
- TeamId
- ChannelId
- ThreadId
- ReplyToMessageId
- IsRootMessage
- FromDisplayName
- FromUserId
- CreatedDateTimeUtc
- LastModifiedDateTimeUtc
- MessageTextPlain
- MessageTextHtml
- HasAttachments
- AttachmentMetadataJson
- MentionsJson
- MessageLink
- IngestionRunId

---

### 7. Build distinct changed thread list
As you process each message, append the `ThreadId` into an array variable.

After all messages are processed:
- use `union(array, array)` trick to dedupe thread IDs

Suggested actions:
- `Append to array variable - ChangedThreadIds`
- `Compose - DistinctThreadIds`

---

### 8. Rebuild each changed thread
Action:
- `Apply to each` over distinct thread IDs

For each thread:
1. Dataverse `List rows` from `Channel_Message_Raw`
2. Filter on `ThreadId eq <current thread>`
3. Sort ascending by `CreatedDateTimeUtc`

Suggested name:
- `Get raw messages for thread`

---

### 9. Assemble thread transcript
Create a formatted plain-text transcript such as:

```text
[2026-03-16 14:05 UTC] Alex:
Question text here

[2026-03-16 14:12 UTC] Jordan:
Reply text here
```

Include:
- timestamps
- sender names
- message text

Store in a `Compose - ThreadTranscript`.

---

### 10. Extract thread summary and issue/solution JSON
Use your AI extraction step here. The output should be strict JSON.

## Recommended extraction prompt
Use a structured prompt that tells the model:

- Read the full Teams channel thread transcript.
- Identify the main issue or question.
- Separate symptoms from actual root cause.
- Identify proposed solutions.
- Determine whether the thread reached a confirmed resolution.
- If resolved, capture the best final answer and who resolved it.
- Ignore greetings, side chatter, and duplicate phrasing.
- Return JSON only.

## Recommended JSON schema
```json
{
  "thread_title_guess": "",
  "short_summary": "",
  "detailed_summary": "",
  "primary_topic": "",
  "tags": [],
  "resolved": false,
  "resolution_summary": "",
  "issue_statement": "",
  "symptoms": [],
  "important_context": [],
  "root_cause": "",
  "proposed_solutions": [],
  "resolution_steps": [],
  "final_answer": "",
  "resolver_name": "",
  "resolved_datetime_utc": "",
  "confidence_score": 0.0,
  "keywords": [],
  "duplicate_fingerprint": ""
}
```

### Suggested action name
- `Run AI extraction for thread`

---

### 11. Parse extracted JSON
Use `Parse JSON` with a stable schema.

Suggested name:
- `Parse thread extraction output`

This makes downstream mapping easier.

---

### 12. Upsert `Channel_Thread_Summary`
Business key:
- `ThreadId`

Map:
- TeamId
- ChannelId
- ThreadStartDateTimeUtc
- LastActivityDateTimeUtc
- ThreadTitleGuess
- ShortSummary
- DetailedSummary
- ResolvedFlag
- ResolutionSummary
- PrimaryTopic
- Tags
- TopContributors
- SourceMessageCount
- LastSummarizedAtUtc

---

### 13. Upsert `Channel_Issue_Solution`
Only do this when at least one of these is true:
- `IssueStatement` is not blank
- `ResolvedFlag` is true
- `FinalAnswer` is not blank

Business key options:
- one record per thread using `ThreadId`
- or one record per deduped issue using `DuplicateFingerprint`

For v1, use:
- **one record per thread**, keyed by `ThreadId`

Map:
- ThreadId
- TeamId
- ChannelId
- IssueStatement
- Symptoms
- ImportantContext
- RootCause
- ProposedSolutions
- ResolutionSteps
- FinalAnswer
- ResolverName
- ResolvedDateTimeUtc
- ResolvedFlag
- ConfidenceScore
- AppliesTo
- Keywords
- DuplicateFingerprint
- ReviewStatus = `AI Extracted`
- SourceMessageCount
- LastExtractedAtUtc = `utcNow()`

---

### 14. Optional: Upsert `Channel_Topic_Index`
For v1, this can be skipped.

If included:
- increment occurrence count by topic
- update `LastSeenDateTimeUtc`
- link related records

---

### 15. Update ingestion control checkpoint
Only after the channel loop completes successfully:
- update `LastSuccessfulIngestUtc` to `varRunStartUtc`
- update `LastAttemptedIngestUtc` to `utcNow()`
- update `LastRunStatus` to `Success`
- clear `LastRunMessage`

If partial failure:
- set `LastRunStatus` = `Partial`
- log error summary

If full failure:
- set `LastRunStatus` = `Failed`
- leave `LastSuccessfulIngestUtc` unchanged

---

### 16. Error handling
Use `Scope` actions:

- `Scope - Main`
- `Scope - Success`
- `Scope - Failure`

Use `Configure run after` so failures update the control table.

Recommended logging:
- channel
- thread count
- message count
- run ID
- last successful checkpoint
- error summary

---

# Flow 2 — Search Knowledge and Memory

## Purpose
This is the flow Copilot Studio calls when a user asks a question.

## Trigger
- `Manually trigger a flow` or Copilot Studio action trigger

## Inputs
- `userQuestion` (text)
- `teamId` (text, optional)
- `channelId` (text, optional)
- `hoursBack` (number, optional; default 168)
- `maxResults` (number, optional; default 5)

## Retrieval strategy

### Step 1 — Search official KB
Dataverse `List rows` on `KB_Article`

Suggested filter:
- `Status eq 'Approved'`

Then apply relevance logic using:
- Dataverse search if available
- keyword contains
- optional AI ranking step

### Step 2 — Search issue/solution memory
Dataverse `List rows` on `Channel_Issue_Solution`

Recommended filters:
- same `TeamId` and `ChannelId` when provided
- prioritize:
  - `ResolvedFlag = true`
  - `ReviewStatus != Rejected`

### Step 3 — Search recent thread summaries
Dataverse `List rows` on `Channel_Thread_Summary`

Recommended filter:
- `LastActivityDateTimeUtc ge addHours(utcNow(), -hoursBack)`

### Step 4 — Rank and package
Return a payload like:

```json
{
  "officialKbMatches": [
    {
      "title": "",
      "summary": "",
      "sourceUrl": "",
      "confidence": 0.0
    }
  ],
  "recentResolvedMatches": [
    {
      "issueStatement": "",
      "finalAnswer": "",
      "resolverName": "",
      "resolvedDateTimeUtc": "",
      "confidence": 0.0
    }
  ],
  "recentThreadSummaries": [
    {
      "threadTitleGuess": "",
      "shortSummary": "",
      "lastActivityDateTimeUtc": ""
    }
  ],
  "answerRecommendation": "",
  "confidence": "High"
}
```

## Agent usage intent
The flow is not expected to write the final user-facing answer. It should return strong grounding context for the agent.

---

# Flow 3 — Submit Correction

## Purpose
Allows users to say the answer was wrong or incomplete.

## Trigger
Copilot Studio action

## Inputs
- `questionText`
- `answerText`
- `feedbackType`
- `userComment`
- `linkedKBArticleId`
- `linkedIssueSolutionId`
- `teamId`
- `channelId`
- `submittedBy`

## Action
Create a row in `Agent_Feedback`

## Follow-up options
- route to reviewer
- send Teams message to owner
- create queue item

---

# Flow 4 — Promote to KB

## Purpose
Converts a reviewed issue/solution into a curated KB article.

## Trigger options
- manual button flow
- Dataverse trigger when `ReviewStatus` changes to `Promoted to KB`

## Inputs
- `IssueSolutionId`

## Actions
1. read `Channel_Issue_Solution`
2. create new `KB_Article`
3. set:
   - Title from issue or generated title
   - Category = Troubleshooting / Process
   - BodyPlainText from `FinalAnswer` + `ResolutionSteps` + `ImportantContext`
   - ShortSummary from `FinalAnswer`
   - Status = Draft or Approved based on governance
4. optionally update source `ReviewStatus`

---

# Example implementation notes

## Example recurrence settings
- every 1 hour
- start at 10 minutes past the hour if desired

## Example channel bootstrap behavior
When first turning on a new channel:
- ingest last 24 hours or 72 hours
- avoid massive historical backfill until v2

## Example v1 limits
- max 1–3 monitored channels
- max last 24 hours on initial load
- only one issue/solution record per thread

---

# Suggested flow naming convention
- `Teams Channel Memory - Hourly Ingest`
- `Teams Channel Memory - Search Knowledge and Memory`
- `Teams Channel Memory - Submit Correction`
- `Teams Channel Memory - Promote to KB`

---

# Key implementation recommendations

## Do in v1
- use Dataverse as your source of truth
- upsert raw messages by `MessageId`
- summarize only changed threads
- extract structured issue/solution JSON
- make Copilot Studio call a retrieval flow for fresh grounding

## Avoid in v1
- custom webhook subscriptions
- vector databases
- complex graph of cross-thread memory
- raw chat grounding without summarization
- treating all extracted answers as official

---

# Testing checklist

## Hourly ingest flow
- new message captured
- edited message updated
- replies linked to correct thread
- duplicate message does not create duplicate row
- changed thread summary refreshes
- unresolved thread stays unresolved
- resolved thread creates useful final answer record

## Search flow
- official KB outranks chat memory
- same question returns prior similar issue/solution
- low-confidence cases return limited results gracefully

## Governance
- incorrect extracted solution can be rejected
- reviewed good solution can be promoted to KB
