---
name: robotrock
description: Integrates the robotrock npm SDK for human-in-the-loop approvals — sendToHuman, webhooks, polling, Trigger.dev, Vercel Workflow, and Vercel AI SDK tools. Use when the user mentions RobotRock, human approval, send to human, HITL, robotrock SDK, approveByHuman, or wiring approval gates into agents/workflows.
license: MIT
metadata:
  author: RobotRock
  version: "1.2.0"
  homepage: https://github.com/quintenb/robotrock/tree/main/packages/sdk
  source: https://github.com/quintenb/robotrock-skills
---

# RobotRock

Integrate the `robotrock` npm package for human-in-the-loop approval workflows.

## When to use

- Gate deployments, releases, or sensitive operations behind human approval
- Pause AI agents until a person reviews a plan or tool call
- Collect structured form data from reviewers (JSON Schema actions)
- Durable waits in Trigger.dev or Vercel Workflow without holding HTTP connections

## Prerequisites

```bash
npm install robotrock
# or: bun add robotrock
```

Environment:

- `ROBOTROCK_API_KEY` — required (create in RobotRock dashboard → Settings → API Keys)
- `ROBOTROCK_WEBHOOK_SECRET` — required when verifying inbound webhooks
- `ROBOTROCK_APP` — optional default inbox bucket (overridable per task)
- `ROBOTROCK_BASE_URL` or `ROBOTROCK_API_URL` — optional, for self-hosted or local dev

## Quick start

Create a shared client module — never instantiate per request:

```typescript
// lib/robotrock.ts
import { createClient } from "robotrock";

export const robotrock = createClient({
  app: "my-service",
  polling: {
    intervalMs: 2_000,
    timeoutMs: 30 * 60_000,
  },
});
```

```typescript
import { robotrock } from "@/lib/robotrock";

const result = await robotrock.sendToHuman({
  type: "budget-approval",
  name: "Approve Q1 budget",
  actions: [
    { id: "approve", title: "Approve" },
    { id: "reject", title: "Reject" },
  ],
});
```

## Choose an integration path

| Scenario | Package entry | Wait mechanism |
|----------|---------------|----------------|
| Script / long-lived server | `robotrock` | Polling (default when no webhook) |
| Fire-and-forget + your HTTP handler | `robotrock` + client `webhook` | Webhook POST callback |
| Trigger.dev background task | `robotrock/trigger` | Wait tokens via `triggerAndWait()` |
| Vercel Workflow | `robotrock/workflow` | `createWebhook()` + suspend |
| Vercel AI SDK agent | `robotrock/ai` | Polling, or durable `mode: "trigger"` / `mode: "workflow"` |

**Decision rule:** If the runtime is short-lived serverless (Vercel function < 60s), do **not** poll in-process — use Trigger.dev or Vercel Workflow.

## Critical gotchas

1. **`webhook` and `polling` are mutually exclusive** on `createClient()` — TypeScript enforces one mode.
2. **`app` and `version` belong on the client**, not on individual `sendToHuman()` calls.
3. **Webhooks/handlers are configured on the client**, not per task or per action input.
4. **Trigger.dev:** always check `waitResult.ok` before accessing `waitResult.output`.
5. **Trigger.dev:** never use `Promise.all` with `triggerAndWait()` or `wait.*` calls.
6. **AI tools block in `execute`** until a human acts — set `polling.timeoutMs` or use durable modes.
7. **Webhook verification:** use `verifyRobotRockWebhook(request)` — do not roll your own crypto.
8. **`assignTo` is a top-level task field**, not inside `context`.
9. **`threadId` is a top-level task field**, not inside `context`. Omit it to start a new thread (the server returns one on `task.threadId`); reuse it to group related tasks.
10. **Updates are thread-scoped.** Use `client.sendUpdate({ threadId, message, status? })` for an existing thread, or pass `update: { message, status? }` to `sendToHuman`. `status` defaults to `info`; valid values are `info | queued | running | waiting | succeeded | failed | cancelled`.

## What do you need?

| Task | Reference |
|------|-----------|
| Client setup and env vars | [references/client-setup.md](references/client-setup.md) |
| `sendToHuman` payloads and assignment | [references/send-to-human.md](references/send-to-human.md) |
| Group related tasks into threads | [references/task-threads.md](references/task-threads.md) |
| Send status updates to a thread | [references/updates.md](references/updates.md) |
| Block until handled (no webhook) | [references/polling.md](references/polling.md) |
| HTTP callbacks when handled | [references/webhooks.md](references/webhooks.md) |
| Rich inbox UI for reviewers | [references/task-context.md](references/task-context.md) |
| Trigger.dev durable waits | [references/trigger.md](references/trigger.md) |
| Vercel Workflow durable waits | [references/vercel-workflow.md](references/vercel-workflow.md) |
| Vercel AI SDK tools and approval bridge | [references/vercel-ai.md](references/vercel-ai.md) |

## Companion skills

When the user is also setting up the host framework, load the relevant framework skill:

- **Trigger.dev** → `trigger-tasks`, `trigger-setup` (`npx skills add triggerdotdev/skills`)
- **Vercel Workflow** → `workflow` (`npx skills add vercel/workflow`)

## SDK exports

| Import | Use for |
|--------|---------|
| `robotrock` | `createClient`, `sendToHuman`, `sendUpdate`, `getTask`, `cancelTask`, webhook verification |
| `robotrock/trigger` | `sendToHumanTask`, `approveByHumanTask` |
| `robotrock/workflow` | `sendToHumanInWorkflow`, `approveByHumanInWorkflow` |
| `robotrock/ai` | `approveByHumanTool`, `createSendToHumanTool`, tool-approval bridge |
| `robotrock/ai/trigger` | AI tools with `mode: "trigger"` |
| `robotrock/ai/workflow` | AI tools with `mode: "workflow"` |

## Implementation checklist

When integrating RobotRock into a project:

- [ ] Install `robotrock` (and peers: `@trigger.dev/sdk`, `workflow`, or `ai` as needed)
- [ ] Create `lib/robotrock.ts` with `createClient`
- [ ] Set `ROBOTROCK_API_KEY` in environment
- [ ] Pick integration path (polling / webhook / trigger / workflow / ai)
- [ ] Define task `type`, `name`, and at least one `action`
- [ ] Add webhook route with `verifyRobotRockWebhook` if using client webhooks
- [ ] For Trigger.dev: re-export SDK tasks from `trigger/robotrock.ts`
- [ ] For Workflow: call `sendToHumanInWorkflow` from `"use workflow"` functions
- [ ] Handle `TaskExpiredError` / `TaskTimeoutError` when polling
