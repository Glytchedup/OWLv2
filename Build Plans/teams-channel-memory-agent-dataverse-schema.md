# Teams Channel Memory Agent — Dataverse Schema

## Purpose
This schema supports a Copilot Studio agent in Microsoft Teams that answers questions using:

- curated knowledge articles
- hourly-ingested Teams channel history
- extracted issue/solution memory
- optional user feedback and review workflows

## Design principles
- Use **Dataverse** as the primary grounding layer for channel deployment.
- Keep **official knowledge** separate from **informal channel memory**.
- Store **raw data**, **thread summaries**, and **normalized issue/solution records** separately.
- Support promotion of repeated good answers from channel memory into official KB.

---

## Table 1: `KB_Article`
Stores curated, reviewed, official knowledge.

### Columns
| Column | Type | Required | Notes |
|---|---|---:|---|
| `KBArticleId` | GUID / primary key | Yes | Unique record ID |
| `Title` | Text | Yes | Human-readable title |
| `Category` | Choice / Text | Yes | Example: Policy, Process, Troubleshooting, Job Aid |
| `Audience` | Text | No | Example: Revenue Manager, Leader, Analyst |
| `ProductArea` | Text | No | Example: SMART, DASH, Copilot Agent |
| `SourceType` | Choice / Text | No | Example: SharePoint, Dataverse, Manual |
| `SourceUrl` | URL | No | Link to original authoritative source |
| `BodyPlainText` | Multiline text | Yes | Full article text in plain language |
| `ShortSummary` | Multiline text | No | Concise summary for retrieval/use in agent answers |
| `Keywords` | Multiline text / Text | No | Comma-separated or normalized values |
| `Status` | Choice | Yes | Draft, Approved, Retired |
| `EffectiveDate` | Date | No | Date the guidance became valid |
| `LastReviewedDate` | Date | No | Human review date |
| `Owner` | Text / Lookup | No | Team or individual owner |
| `ConfidenceWeight` | Decimal | No | Optional manual boost for retrieval |
| `VersionNumber` | Text | No | Optional version tracking |
| `CreatedOn` | System | Yes | Standard Dataverse |
| `ModifiedOn` | System | Yes | Standard Dataverse |

### Recommended indexes / alternate keys
- Alternate key on `Title` + `VersionNumber` if versioning matters
- Searchable columns:
  - `Title`
  - `BodyPlainText`
  - `ShortSummary`
  - `Keywords`
  - `ProductArea`

---

## Table 2: `Channel_Message_Raw`
Stores every ingested Teams channel message and reply.

### Columns
| Column | Type | Required | Notes |
|---|---|---:|---|
| `MessageId` | Text / alternate key | Yes | Teams message ID |
| `TeamId` | Text | Yes | Team identifier |
| `ChannelId` | Text | Yes | Channel identifier |
| `ThreadId` | Text | Yes | Root message ID or normalized thread ID |
| `ReplyToMessageId` | Text | No | Parent/root message reference |
| `IsRootMessage` | Yes/No | Yes | True for root thread post |
| `FromDisplayName` | Text | No | Sender display name |
| `FromUserId` | Text | No | AAD/Object ID if available |
| `CreatedDateTimeUtc` | DateTime | Yes | Original create time |
| `LastModifiedDateTimeUtc` | DateTime | No | Edit time if available |
| `MessageTextPlain` | Multiline text | No | Cleaned plain text |
| `MessageTextHtml` | Multiline text | No | Raw/HTML message body |
| `HasAttachments` | Yes/No | No | Attachment indicator |
| `AttachmentMetadataJson` | Multiline text | No | Raw attachment metadata |
| `MentionsJson` | Multiline text | No | Mention payload |
| `MessageLink` | URL | No | Deep link to message if available |
| `LanguageCode` | Text | No | Optional, if detecting language |
| `IngestionRunId` | Text | No | Correlates rows to an ingest run |
| `NormalizedHash` | Text | No | Optional dedupe hash of normalized text |
| `CreatedOn` | System | Yes | Standard Dataverse |
| `ModifiedOn` | System | Yes | Standard Dataverse |

### Notes
- Upsert using `MessageId` as the business key.
- Keep this table raw and audit-friendly.
- Do not use this table as the primary grounding source for agent answers unless you need fallback detail.

---

## Table 3: `Channel_Thread_Summary`
Stores one refreshed summary per thread.

### Columns
| Column | Type | Required | Notes |
|---|---|---:|---|
| `ThreadSummaryId` | GUID / primary key | Yes | Unique record ID |
| `ThreadId` | Text / alternate key | Yes | Matches root thread |
| `TeamId` | Text | Yes | Team identifier |
| `ChannelId` | Text | Yes | Channel identifier |
| `ThreadStartDateTimeUtc` | DateTime | Yes | First post time |
| `LastActivityDateTimeUtc` | DateTime | Yes | Most recent message/reply time |
| `ThreadTitleGuess` | Text | No | Generated title based on thread |
| `ShortSummary` | Multiline text | Yes | 1–3 sentence concise summary |
| `DetailedSummary` | Multiline text | No | More complete thread summary |
| `ResolvedFlag` | Yes/No | Yes | Whether the thread reached a real resolution |
| `ResolutionSummary` | Multiline text | No | Final confirmed answer if any |
| `PrimaryTopic` | Text | No | Main subject |
| `Tags` | Multiline text / Text | No | Search-oriented keywords |
| `TopContributors` | Multiline text / Text | No | Names or IDs |
| `SourceMessageCount` | Whole number | No | Count of raw messages in thread |
| `LastSummarizedAtUtc` | DateTime | Yes | Refresh time |
| `SummarizationModel` | Text | No | Optional observability field |
| `SummarizationConfidence` | Decimal | No | Optional score |
| `CreatedOn` | System | Yes | Standard Dataverse |
| `ModifiedOn` | System | Yes | Standard Dataverse |

### Notes
- Upsert using `ThreadId`.
- This is your first useful retrieval layer for “what was discussed recently?”

---

## Table 4: `Channel_Issue_Solution`
Stores normalized issue/resolution records extracted from threads.

## Why this table matters
This becomes the highest-value memory table for the agent. It is much more useful than raw chat because it converts noisy conversation into reusable resolution records.

### Columns
| Column | Type | Required | Notes |
|---|---|---:|---|
| `IssueSolutionId` | GUID / primary key | Yes | Unique record ID |
| `ThreadId` | Text | Yes | Source thread |
| `TeamId` | Text | Yes | Team identifier |
| `ChannelId` | Text | Yes | Channel identifier |
| `IssueStatement` | Multiline text | Yes | Main question/problem in plain language |
| `Symptoms` | Multiline text | No | Bullet/JSON/text list |
| `ImportantContext` | Multiline text | No | Conditions, hotel/system/process context |
| `RootCause` | Multiline text | No | If known |
| `ProposedSolutions` | Multiline text | No | Candidate resolutions from thread |
| `ResolutionSteps` | Multiline text | No | Ordered steps taken |
| `FinalAnswer` | Multiline text | No | Best clean answer |
| `ResolverName` | Text | No | Person who solved it |
| `ResolvedDateTimeUtc` | DateTime | No | When resolution became clear |
| `ResolvedFlag` | Yes/No | Yes | Whether the issue was actually solved |
| `ConfidenceScore` | Decimal | No | Model confidence |
| `AppliesTo` | Text | No | Product/system/region/hotel/process label |
| `Keywords` | Multiline text / Text | No | Retrieval terms |
| `DuplicateFingerprint` | Text | No | Stable dedupe/merge key |
| `ReviewStatus` | Choice | Yes | AI Extracted, Reviewed, Promoted to KB, Rejected |
| `Reviewer` | Text / Lookup | No | Human reviewer |
| `ReviewNotes` | Multiline text | No | Audit/comments |
| `SourceMessageCount` | Whole number | No | Size of originating thread |
| `LastExtractedAtUtc` | DateTime | Yes | Extraction time |
| `CreatedOn` | System | Yes | Standard Dataverse |
| `ModifiedOn` | System | Yes | Standard Dataverse |

### Recommended indexes / alternate keys
- Alternate key on `ThreadId` if one thread should produce one issue/solution record
- Additional non-unique index on:
  - `DuplicateFingerprint`
  - `ResolvedFlag`
  - `ReviewStatus`
  - `Keywords`

---

## Table 5: `Channel_Topic_Index`
Supports fast routing and grouping of similar questions.

### Columns
| Column | Type | Required | Notes |
|---|---|---:|---|
| `TopicIndexId` | GUID / primary key | Yes | Unique record ID |
| `TopicName` | Text | Yes | Canonical topic |
| `CanonicalQuestion` | Multiline text | No | Best representative phrasing |
| `AlternatePhrasings` | Multiline text | No | Similar ways users ask |
| `Keywords` | Multiline text / Text | No | Search terms |
| `RelatedKBArticleIds` | Multiline text / Text | No | IDs or references |
| `RelatedThreadIds` | Multiline text / Text | No | IDs or references |
| `RelatedIssueSolutionIds` | Multiline text / Text | No | IDs or references |
| `LastSeenDateTimeUtc` | DateTime | No | Most recent occurrence |
| `OccurrenceCount` | Whole number | No | Topic frequency |
| `CreatedOn` | System | Yes | Standard Dataverse |
| `ModifiedOn` | System | Yes | Standard Dataverse |

### Notes
- This table is optional in v1, but useful if you want topic clustering or trend analysis.

---

## Table 6: `Channel_Ingestion_Control`
Tracks monitored channels and checkpoints.

### Columns
| Column | Type | Required | Notes |
|---|---|---:|---|
| `IngestionControlId` | GUID / primary key | Yes | Unique record ID |
| `TeamId` | Text | Yes | Monitored team |
| `ChannelId` | Text | Yes | Monitored channel |
| `TeamName` | Text | No | Friendly name |
| `ChannelName` | Text | No | Friendly name |
| `IsActive` | Yes/No | Yes | Whether to ingest this channel |
| `LastSuccessfulIngestUtc` | DateTime | No | Checkpoint for delta loads |
| `LastAttemptedIngestUtc` | DateTime | No | Audit/monitoring |
| `LastRunStatus` | Choice | No | Success, Partial, Failed |
| `LastRunMessage` | Multiline text | No | Error details |
| `HoursBackOnFirstRun` | Whole number | No | Seed-backfill control |
| `CreatedOn` | System | Yes | Standard Dataverse |
| `ModifiedOn` | System | Yes | Standard Dataverse |

### Notes
- Use this table to drive one recurring flow across multiple channels.

---

## Table 7: `Agent_Feedback`
Captures user feedback on agent answers.

### Columns
| Column | Type | Required | Notes |
|---|---|---:|---|
| `FeedbackId` | GUID / primary key | Yes | Unique record ID |
| `ConversationId` | Text | No | Optional conversation/session ID |
| `QuestionText` | Multiline text | Yes | User question |
| `AnswerText` | Multiline text | Yes | Agent response |
| `FeedbackType` | Choice | Yes | Correct, Incorrect, Incomplete |
| `UserComment` | Multiline text | No | Freeform notes |
| `LinkedKBArticleId` | Text | No | Optional |
| `LinkedIssueSolutionId` | Text | No | Optional |
| `TeamId` | Text | No | Context |
| `ChannelId` | Text | No | Context |
| `SubmittedBy` | Text | No | User |
| `SubmittedAtUtc` | DateTime | Yes | Timestamp |
| `CreatedOn` | System | Yes | Standard Dataverse |
| `ModifiedOn` | System | Yes | Standard Dataverse |

---

## Relationships

### Recommended logical relationships
- `Channel_Message_Raw.ThreadId` → `Channel_Thread_Summary.ThreadId`
- `Channel_Issue_Solution.ThreadId` → `Channel_Thread_Summary.ThreadId`
- `Channel_Thread_Summary.TeamId + ChannelId` → `Channel_Ingestion_Control.TeamId + ChannelId`
- `Agent_Feedback.LinkedKBArticleId` → `KB_Article.KBArticleId`
- `Agent_Feedback.LinkedIssueSolutionId` → `Channel_Issue_Solution.IssueSolutionId`

### Practical note
You can implement these as lookups where useful, but text IDs are often simpler for flow-based upserts and cross-system references.

---

## Retrieval priority model

When the agent answers a question, search in this order:

1. `KB_Article`
2. `Channel_Issue_Solution`
3. `Channel_Thread_Summary`
4. `Channel_Message_Raw` only if more detail is needed

This preserves the hierarchy:
- official guidance first
- prior team resolution second
- noisy chat last

---

## Minimal v1 schema
If you want to move fast, start with only:

- `KB_Article`
- `Channel_Message_Raw`
- `Channel_Thread_Summary`
- `Channel_Issue_Solution`
- `Channel_Ingestion_Control`

Add `Channel_Topic_Index` and `Agent_Feedback` after the pilot proves value.

---

## Data retention recommendation

### Suggested defaults
- `Channel_Message_Raw`: 90–180 days
- `Channel_Thread_Summary`: 1 year or longer
- `Channel_Issue_Solution`: long-term, unless sensitive
- `KB_Article`: long-term with retirement state
- `Agent_Feedback`: at least 1 year for QA and tuning

Adjust based on tenant policy and privacy requirements.

---

## Security and governance
- Restrict who can edit `KB_Article` and change review status.
- Allow ingestion flows to create/update memory tables.
- Consider row ownership or team ownership for business-unit-specific channels.
- Do not automatically treat channel chatter as official process.
- Require human review before promoting extracted answers into `KB_Article`.

---

## Recommended choices
- Use **plain text fields** for content that will be searched and grounded.
- Store raw structured outputs as JSON only where necessary for audit/debug.
- Keep `ReviewStatus` on `Channel_Issue_Solution` so you can mature the memory into reliable knowledge over time.
