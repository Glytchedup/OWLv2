# Security Guidance

## Data Classification

Treat Teams channel ingestion data as internal and potentially sensitive.

- `Channel_Message_Raw` may contain user identifiers, operational details, or sensitive free text.
- Official guidance should only come from approved `KB_Article` records.

## Required Controls

1. Restrict edit access to `KB_Article` and review status fields.
2. Enable Dataverse auditing for KB changes and promotion events.
3. Apply retention policy for raw messages (recommended 90 to 180 days).
4. Avoid exposing internal IDs, connector payloads, or raw JSON to end users.

## Secrets and Credentials

- Never commit secrets, tokens, connection strings, or tenant-specific private identifiers.
- Store connection configuration in Power Platform environment settings.

## AI Extraction Safety

1. Mark extracted issue/solution records as `AI Extracted` by default.
2. Require human review before promotion to official KB.
3. Add confidence thresholds and escalation behavior for weak grounding.

## Incident Handling

If sensitive data is accidentally committed:

1. Revoke and rotate impacted credentials immediately.
2. Remove exposed values from source history following security policy.
3. Notify repository owners and security stakeholders.
