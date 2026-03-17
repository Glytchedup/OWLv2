# Manual Implementation Plan

This plan covers everything that **must be done manually** in the Microsoft Power Platform tenant. These steps cannot be automated through the repository and require access to Copilot Studio, Dataverse, Power Automate, and Teams Admin.

---

## Phase 1: Dataverse Schema Deployment

### Step 1.1: Create the Dataverse tables

Create all 7 tables in your target Dataverse environment. Reference: `Build Plans/teams-channel-memory-agent-dataverse-schema.md`

**v1 required tables (create first):**

#### Table 1: `KB_Article`
1. Go to **Power Apps** > **Tables** > **New table**
2. Display name: `KB Article`, Schema name: `KB_Article`
3. Add columns exactly as specified in the schema doc:
   - `Title` (Text, required, max 200 chars)
   - `Category` (Choice: Policy, Process, Troubleshooting, Job Aid — required)
   - `Audience` (Text, optional)
   - `ProductArea` (Text, optional)
   - `SourceType` (Choice: SharePoint, Dataverse, Manual — optional)
   - `SourceUrl` (URL, optional)
   - `BodyPlainText` (Multiline text, required, max 100,000 chars)
   - `ShortSummary` (Multiline text, optional, max 2,000 chars)
   - `Keywords` (Multiline text, optional)
   - `Status` (Choice: Draft, Approved, Retired — required, default: Draft)
   - `EffectiveDate` (Date, optional)
   - `LastReviewedDate` (Date, optional)
   - `Owner` (Text, optional)
   - `ConfidenceWeight` (Decimal, optional)
   - `VersionNumber` (Text, optional)
4. Enable **Dataverse search** on: `Title`, `BodyPlainText`, `ShortSummary`, `Keywords`, `ProductArea`
5. Set alternate key on `Title` + `VersionNumber` if versioning is needed

#### Table 2: `Channel_Message_Raw`
1. Create table with display name `Channel Message Raw`
2. Add all columns from schema doc (MessageId, TeamId, ChannelId, ThreadId, etc.)
3. **Critical**: Set alternate key on `MessageId` — this enables upsert in Power Automate
4. Mark `MessageId`, `TeamId`, `ChannelId`, `ThreadId`, `IsRootMessage`, `CreatedDateTimeUtc` as required

#### Table 3: `Channel_Thread_Summary`
1. Create table with display name `Channel Thread Summary`
2. Add all columns from schema doc
3. **Critical**: Set alternate key on `ThreadId`
4. Enable Dataverse search on: `ShortSummary`, `DetailedSummary`, `PrimaryTopic`, `Tags`

#### Table 4: `Channel_Issue_Solution`
1. Create table with display name `Channel Issue Solution`
2. Add all columns from schema doc
3. Set alternate key on `ThreadId` (one record per thread for v1)
4. Set `ReviewStatus` choice values: `AI Extracted`, `Reviewed`, `Promoted to KB`, `Rejected`
5. Default `ReviewStatus` to `AI Extracted`
6. Enable Dataverse search on: `IssueStatement`, `FinalAnswer`, `Keywords`, `AppliesTo`
7. Add non-unique indexes on: `DuplicateFingerprint`, `ResolvedFlag`, `ReviewStatus`

#### Table 5: `Channel_Ingestion_Control`
1. Create table with display name `Channel Ingestion Control`
2. Add all columns from schema doc
3. Set `LastRunStatus` choice values: `Success`, `Partial`, `Failed`

#### Table 6: `Agent_Feedback`
1. Create table with display name `Agent Feedback`
2. Add all columns from schema doc
3. Set `FeedbackType` choice values: `Correct`, `Incorrect`, `Incomplete`

**v2 tables (create after pilot proves value):**

#### Table 7: `Channel_Topic_Index`
1. Create table with all columns from schema doc
2. Optional for v1 — useful for topic clustering and trend analysis later

### Step 1.2: Set table permissions
1. Restrict `KB_Article` edit access to owners/admins only
2. Allow ingestion service account to create/update on: `Channel_Message_Raw`, `Channel_Thread_Summary`, `Channel_Issue_Solution`, `Channel_Ingestion_Control`
3. Allow agent service account read access on all tables
4. Enable auditing on `KB_Article` and `Channel_Issue_Solution.ReviewStatus`

### Step 1.3: Set data retention policies
- `Channel_Message_Raw`: 90–180 day retention
- `Channel_Thread_Summary`: 1 year minimum
- `Channel_Issue_Solution`: long-term (unless sensitive)
- `KB_Article`: long-term with Retired status for lifecycle
- `Agent_Feedback`: 1 year minimum

---

## Phase 2: Seed Ingestion Control

### Step 2.1: Create pilot channel record
1. Open `Channel_Ingestion_Control` table in Power Apps
2. Create one row:
   - `TeamId`: your pilot team's ID (get from Teams Admin or Graph Explorer)
   - `ChannelId`: your pilot channel's ID
   - `TeamName`: friendly name (e.g., "Revenue Operations")
   - `ChannelName`: friendly name (e.g., "General")
   - `IsActive`: Yes
   - `HoursBackOnFirstRun`: 24

### Step 2.2: Get Team and Channel IDs
1. In Teams, right-click the channel > **Get link to channel**
2. The URL contains `groupId` (this is TeamId) and `channelId` (URL-encoded)
3. Or use Graph Explorer: `GET https://graph.microsoft.com/v1.0/me/joinedTeams` then `GET .../teams/{id}/channels`

---

## Phase 3: Power Automate Flows

Build these 4 flows in the same environment as your Dataverse tables.

### Flow 1: `Teams Channel Memory - Hourly Ingest`

**Trigger**: Recurrence — every 1 hour (offset 10 minutes past the hour recommended)

**Step-by-step build instructions:**

1. **Add trigger**: Recurrence, Frequency = Hour, Interval = 1
2. **List active channels**: Add Dataverse "List rows" action
   - Table: `Channel_Ingestion_Control`
   - Filter: `IsActive eq true`
   - Select: TeamId, ChannelId, TeamName, ChannelName, LastSuccessfulIngestUtc, HoursBackOnFirstRun
3. **Apply to each** channel control row:

   a. **Initialize variables**:
   - `varTeamId` = current item TeamId
   - `varChannelId` = current item ChannelId
   - `varRunStartUtc` = `utcNow()`
   - `varIngestionRunId` = `guid()`
   - `varLookbackStartUtc`:
     - If `LastSuccessfulIngestUtc` is blank: `addHours(utcNow(), -coalesce(HoursBackOnFirstRun, 24))`
     - Else: `LastSuccessfulIngestUtc`

   b. **Scope - Main** (wrap remaining steps for error handling):

   c. **Retrieve channel messages** since `varLookbackStartUtc`:
   - Use the Teams connector or Graph API HTTP action (whichever is approved in your tenant)
   - Get messages and replies for the channel
   - For each message, capture: messageId, createdDateTime, lastModifiedDateTime, from.user.displayName, body.content, replyToId, mentions, attachments

   d. **Apply to each message**:
   - **Normalize text**: Strip HTML tags, decode entities, flatten line breaks, resolve @mentions to display names, trim whitespace
   - **Upsert to Channel_Message_Raw**:
     - List rows filtered by `MessageId eq '{messageId}'`
     - If found: Update row
     - If not found: Add new row
     - Map all fields per schema doc
   - **Track thread IDs**: Append `ThreadId` to an array variable `ChangedThreadIds`

   e. **Deduplicate thread list**: `union(ChangedThreadIds, ChangedThreadIds)`

   f. **Apply to each distinct ThreadId**:
   - **Get all messages for thread**: List rows from `Channel_Message_Raw` where `ThreadId eq '{threadId}'`, sort by `CreatedDateTimeUtc` ascending
   - **Build transcript**: Compose formatted text:
     ```
     [2026-03-16 14:05 UTC] Alex:
     Question text here

     [2026-03-16 14:12 UTC] Jordan:
     Reply text here
     ```
   - **AI extraction**: Use AI Builder or an AI prompt action with this instruction:
     > Read the full Teams channel thread transcript. Identify the main issue or question. Separate symptoms from root cause. Identify proposed solutions. Determine if the thread reached a confirmed resolution. If resolved, capture the final answer and who resolved it. Ignore greetings, side chatter, and duplicate phrasing. Return strict JSON only.

     Expected JSON output schema:
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
   - **Parse JSON**: Use Parse JSON action with the schema above
   - **Upsert Channel_Thread_Summary**: Business key = ThreadId. Map all fields from parsed output
   - **Upsert Channel_Issue_Solution**: Only when `issue_statement` is not blank OR `resolved` is true OR `final_answer` is not blank. Business key = ThreadId. Set `ReviewStatus = AI Extracted`, `LastExtractedAtUtc = utcNow()`

   g. **Scope - Success** (configure run after: Main succeeded):
   - Update `Channel_Ingestion_Control`:
     - `LastSuccessfulIngestUtc` = `varRunStartUtc`
     - `LastAttemptedIngestUtc` = `utcNow()`
     - `LastRunStatus` = Success
     - `LastRunMessage` = (clear)

   h. **Scope - Failure** (configure run after: Main failed/timed out):
   - Update `Channel_Ingestion_Control`:
     - `LastAttemptedIngestUtc` = `utcNow()`
     - `LastRunStatus` = Failed
     - `LastRunMessage` = error summary
     - Do NOT update `LastSuccessfulIngestUtc`

### Flow 2: `Teams Channel Memory - Search Knowledge and Memory`

**Trigger**: Manually trigger a flow (Copilot Studio action trigger)

**Inputs**: `userQuestion` (text, required), `teamId` (text), `channelId` (text), `hoursBack` (number, default 168), `maxResults` (number, default 5)

**Step-by-step build instructions:**

1. **Search official KB**:
   - Dataverse "List rows" on `KB_Article`
   - Filter: `Status eq 'Approved'`
   - Use Dataverse search or keyword contains against `Title`, `BodyPlainText`, `ShortSummary`, `Keywords`
   - Limit to top `maxResults` rows
   - Optional: Add AI ranking step for relevance scoring

2. **Search issue/solution memory**:
   - Dataverse "List rows" on `Channel_Issue_Solution`
   - Filter: `ReviewStatus ne 'Rejected'` and optionally `TeamId eq '{teamId}'` and `ChannelId eq '{channelId}'`
   - Prioritize rows where `ResolvedFlag eq true`
   - Match against `IssueStatement`, `FinalAnswer`, `Keywords`
   - Limit to top `maxResults`

3. **Search recent thread summaries**:
   - Dataverse "List rows" on `Channel_Thread_Summary`
   - Filter: `LastActivityDateTimeUtc ge addHours(utcNow(), -hoursBack)`
   - Optionally scope to `TeamId`/`ChannelId`
   - Match against `ShortSummary`, `PrimaryTopic`, `Tags`
   - Limit to top `maxResults`

4. **Package response**: Compose the output JSON:
   ```json
   {
     "officialKbMatches": [...],
     "recentResolvedMatches": [...],
     "recentThreadSummaries": [...],
     "answerRecommendation": "",
     "confidence": "High|Medium|Low"
   }
   ```
   Confidence logic:
   - High: strong KB match exists
   - Medium: no KB but strong issue/solution match
   - Low: only thread summaries or nothing found

5. **Return response** to Copilot Studio

### Flow 3: `Teams Channel Memory - Submit Correction`

**Trigger**: Copilot Studio action trigger

**Inputs**: `questionText`, `answerText`, `feedbackType` (Correct/Incorrect/Incomplete), `userComment`, `linkedKBArticleId`, `linkedIssueSolutionId`, `teamId`, `channelId`, `submittedBy`

**Steps:**
1. Create new row in `Agent_Feedback` table mapping all inputs
2. Set `SubmittedAtUtc` = `utcNow()`
3. Return `{ "status": "Success", "message": "Feedback recorded. Thank you." }`
4. Optional: Send a Teams notification to the review queue owner

### Flow 4: `Teams Channel Memory - Promote to KB`

**Trigger**: Manual button flow OR Dataverse trigger when `Channel_Issue_Solution.ReviewStatus` changes to `Promoted to KB`

**Inputs**: `IssueSolutionId`

**Steps:**
1. Read the `Channel_Issue_Solution` row
2. Create new `KB_Article` row:
   - `Title`: Use thread title guess or generate from issue statement
   - `Category`: Troubleshooting (default, or map from context)
   - `BodyPlainText`: Combine `FinalAnswer` + `ResolutionSteps` + `ImportantContext`
   - `ShortSummary`: Use `FinalAnswer`
   - `Status`: Draft (require human approval before setting to Approved)
   - `SourceType`: Dataverse
   - `Keywords`: Copy from issue/solution record
3. Update source `Channel_Issue_Solution.ReviewStatus` to `Promoted to KB` if not already set
4. Return success confirmation

---

## Phase 4: Copilot Studio Wiring

### Step 4.1: Register Power Automate actions
1. Open OWLv2 agent in Copilot Studio
2. Go to **Actions** > **Add an action**
3. Register each flow as an action:
   - `SearchKnowledgeAndMemory` → Flow 2
   - `SubmitCorrection` → Flow 3
   - `GetLatestChannelContext` → (optional, build if needed using a simplified version of Flow 2's thread summary search)
4. Map inputs and outputs per the contracts in `Build Plans/teams-channel-memory-agent-copilot-studio-design.md`

### Step 4.2: Add Dataverse knowledge source
1. In Copilot Studio, go to **Knowledge** > **Add knowledge**
2. Add `KB_Article` table as a Dataverse knowledge source
3. Map searchable fields: `Title`, `BodyPlainText`, `ShortSummary`, `Keywords`
4. This enables the `SearchAndSummarizeContent` action in the Search topic to use KB data directly

### Step 4.3: Configure action invocation guidance
1. In the action settings for `SearchKnowledgeAndMemory`, add invocation guidance:
   > Call this action when the user asks an operational question, asks if an issue has come up before, or asks how the team previously handled something.
2. For `SubmitCorrection`:
   > Call this action when the user says the answer is wrong, outdated, or incomplete, or when the user provides a correction.

### Step 4.4: Publish agent
1. Publish the agent in Copilot Studio
2. Add to the pilot Teams channel
3. Verify sign-in and Dataverse access work end-to-end

---

## Phase 5: Pilot Validation

Run through this checklist before going live:

- [ ] **Ingest flow**: New message appears in `Channel_Message_Raw` within 1 hour
- [ ] **Ingest flow**: Edited message updates (not duplicates) the existing row
- [ ] **Ingest flow**: Replies link to correct ThreadId
- [ ] **Thread summary**: Changed thread gets refreshed summary
- [ ] **Issue/Solution**: Resolved thread creates a `Channel_Issue_Solution` record
- [ ] **Search**: Official KB result outranks channel memory when both exist
- [ ] **Search**: Prior similar issue/solution is returned for matching questions
- [ ] **Low confidence**: Agent clearly states low confidence and offers escalation
- [ ] **Escalation**: User can reach escalation path and get a ticket template
- [ ] **Feedback**: Correction flow writes to `Agent_Feedback` table
- [ ] **Conflict**: Agent states conflict when KB and channel memory disagree
- [ ] **No answer**: Agent does not invent answers; says grounding is weak

---

## Phase 6: Operations and Hardening

### Monitoring
- Set up an alert if `Channel_Ingestion_Control.LastSuccessfulIngestUtc` is more than 3 hours old
- Track weekly: answer count, correction rate, escalation rate, low-confidence rate
- Review `Agent_Feedback` table weekly for quality trends

### Governance
- Only promote `Channel_Issue_Solution` records that have been reviewed by a human
- Run quarterly access review on KB edit permissions
- Run quarterly retention review on `Channel_Message_Raw`

### v2 Roadmap
- Add `Channel_Topic_Index` for topic clustering and trend reporting
- Add `Agent_Feedback` table if not created in v1
- Add recurring issue detection (same `DuplicateFingerprint` appearing multiple times)
- Expand to additional Teams channels via new `Channel_Ingestion_Control` rows
