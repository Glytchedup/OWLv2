# Tenant Implementation Runbook

This runbook covers the manual Microsoft platform steps required to complete OWLv2.

## 1. Dataverse Schema Deployment

Create the following tables in your target environment:

1. `KB_Article`
2. `Channel_Message_Raw`
3. `Channel_Thread_Summary`
4. `Channel_Issue_Solution`
5. `Channel_Topic_Index`
6. `Channel_Ingestion_Control`
7. `Agent_Feedback`

Reference: `Build Plans/teams-channel-memory-agent-dataverse-schema.md`

### Required key setup

- `Channel_Message_Raw`: alternate key on `MessageId`
- `Channel_Thread_Summary`: alternate key on `ThreadId`
- `Channel_Issue_Solution`: thread-centric keying for v1 (one record per thread)

### Governance baseline

- Restrict KB edit rights to owners/admins
- Enable auditing for KB and review status changes
- Apply retention policy for raw message table

## 2. Seed Ingestion Control

Create one pilot channel row in `Channel_Ingestion_Control` with:

- `IsActive = true`
- `TeamId`, `ChannelId` populated
- first-run lookback configured (24h recommended)

## 3. Power Automate Flows

Build these flows exactly:

1. Teams Channel Memory - Hourly Ingest
2. Teams Channel Memory - Search Knowledge and Memory
3. Teams Channel Memory - Submit Correction
4. Teams Channel Memory - Promote to KB

Reference: `Build Plans/teams-channel-memory-agent-power-automate-plan.md`

### Critical flow behavior

- Hourly ingest updates checkpoint only on successful completion
- Search flow returns packaged payload, not final user wording
- Correction flow writes to `Agent_Feedback`
- Promote flow creates curated KB entries from reviewed records

## 4. Copilot Studio Wiring

In Copilot Studio:

1. Register the Power Automate actions
2. Bind retrieval action to question answering behavior
3. Publish the agent to pilot channel
4. Validate sign-in and Dataverse access end-to-end

## 5. Pilot Validation Checklist

- Ingest flow captures new and edited messages
- Thread summaries refresh for changed threads
- KB results outrank discussion memory when both exist
- Low-confidence responses offer escalation path
- Corrections are captured and reviewable

## 6. Operations and Hardening

- Alert if no successful ingest for 3+ hours
- Track answer quality and correction rate weekly
- Promote only reviewed issue/solution records to KB
- Run quarterly access and retention review
