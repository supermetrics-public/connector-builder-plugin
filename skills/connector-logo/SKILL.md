---
name: connector-logo
description: Use when the user wants to upload, replace, or view the logo image for a Supermetrics Connector they have access to.
---

# Connector logo

Upload, replace, or fetch the logo for a saved Connector.

## Image rules

- **Formats:** PNG or JPG.
- **Max size:** 5 MB.
- **Recommended:** square, ≥ 256×256, transparent background for PNG.

## Upload / replace

```bash
supermetrics connector-builder-logo upload \
  --team-id <team-id> \
  --connector-identifier <ds-id> \
  --file <path-to-image>
```

The command overwrites any existing logo for that Connector. There
is no separate "delete" — upload a different image if needed.

## Fetch current

```bash
supermetrics connector-builder-logo get \
  --team-id <team-id> \
  --connector-identifier <ds-id> \
  --output <save-path>
```

(Output flag name per `--help`; defaults may vary.)

## When in the workflow

Logo upload is **optional** and **independent** of all other phases.
Reasonable times to invoke:

- After the Connector is validated (`connector-validation` passed),
  as a finishing touch before sharing with users.
- Any time later, on user request.

The build → validate workflow does not depend on the logo. Never
block other phases on this skill.

## Common mistakes

| Mistake | Fix |
|---|---|
| Trying to upload a file > 5 MB | Resize first. The CLI rejects oversize uploads. |
| Using a non-PNG / non-JPG (e.g. SVG, GIF) | Convert to PNG or JPG. |
| Forgetting `--team-id` because the user is on multiple teams | Read it from `<project>/.team-id`. |

## Deeper reference

- `${CLAUDE_PLUGIN_ROOT}/docs/cli-reference.md §7` — `connector-builder-logo` commands.
