# Contributing

## Branching

1. Use `main` as the protected release branch.
2. Create feature branches from `main` using:
   - `feat/<short-description>`
   - `fix/<short-description>`
   - `chore/<short-description>`

## Pull Requests

1. Keep PR scope focused to one implementation slice.
2. Include:
   - What changed
   - Why it changed
   - Validation performed
   - Rollout risk and rollback notes
3. Require at least one reviewer approval before merge.

## Commit Style

Use clear, action-oriented commit messages, for example:

- `feat(agent): enforce official-guidance response format`
- `feat(topic): add low-confidence escalation path`
- `docs: add implementation runbook`

## Validation

Before opening PR:

1. Verify YAML parses without errors.
2. Confirm topic behavior impact is documented.
3. Confirm no credentials, tenant IDs, or secrets are committed.

## Scope Boundaries

- Keep official KB policy and prior channel memory clearly separated.
- Do not represent informal precedent as approved process.
- Mark low-confidence answers clearly and provide escalation path.
