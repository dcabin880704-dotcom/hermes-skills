# hermes-skills

Reusable Hermes skill repository.

Contents:
- `skills/amazon-ozon-draft-workflow/SKILL.md`
- `skills/ozon-1688-upload-pipeline/SKILL.md`

Use on another machine:

```bash
hermes skills tap add dcabin880704-dotcom/hermes-skills
hermes skills list | grep -E 'amazon-ozon-draft-workflow|ozon-1688-upload-pipeline'
```

Current focus:
- Amazon -> Ozon draft workflow development under `~/ozon_pipeline`
- 1688 -> Ozon upload workflow with image translation and Ozon API publishing

Security rule:
- Do not commit real API keys, client IDs, tokens, bucket URLs, fixed resource URLs, or other live credentials into this repo.
- Public skills must reference local private configuration for all sensitive values.

This repo is intentionally generic so more Hermes skills can be added over time.
