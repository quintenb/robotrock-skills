# robotrock-skills

Agent skill for integrating the [robotrock](https://www.npmjs.com/package/robotrock) npm SDK — human-in-the-loop approvals for scripts, webhooks, Trigger.dev, Vercel Workflow, and Vercel AI SDK agents.

## Install

```bash
npx skills add quintenb/robotrock-skills --skill robotrock
```

```bash
# Global (all projects)
npx skills add quintenb/robotrock-skills --skill robotrock -g -y

# Cursor only
npx skills add quintenb/robotrock-skills --skill robotrock -a cursor -y
```

## What it covers

- `createClient` setup and environment variables
- `sendToHuman` task payloads, actions, and assignment
- Polling vs webhook modes
- Trigger.dev durable waits (`sendToHumanTask`, `approveByHumanTask`)
- Vercel Workflow durable waits (`sendToHumanInWorkflow`)
- Vercel AI SDK tools and tool-approval bridge (`robotrock/ai`)

## Documentation

- SDK README: https://github.com/quintenb/robotrock/tree/main/packages/sdk
- Official docs: https://robotrock.dev/docs (when available)

## License

MIT
