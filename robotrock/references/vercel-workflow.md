# Vercel Workflow integration

Use `robotrock/workflow` for **durable waits** inside Vercel Workflow runs.

```bash
npm install robotrock workflow
```

`workflow` is an optional peer dependency of `robotrock/workflow`.

## How it works

1. `sendToHumanInWorkflow()` calls `createWebhook()` and gets `webhook.url`
2. A step creates the RobotRock task with that URL as the client webhook
3. The workflow suspends on `await webhook` until RobotRock POSTs the result
4. `validUntil` is enforced with `Promise.race` against `sleep(timeout)`

Do **not** configure a client `webhook` yourself — the SDK sets it automatically.

## Environment variables

- `ROBOTROCK_API_KEY` (required)
- `ROBOTROCK_BASE_URL` or `ROBOTROCK_API_URL` (optional)
- `ROBOTROCK_APP` (optional default inbox bucket)

## `sendToHumanInWorkflow`

Call from a function whose **first statement** is `"use workflow"`:

```typescript
import { sendToHumanInWorkflow } from "robotrock/workflow";

export async function deploymentApproval(version: string) {
  "use workflow";

  const result = await sendToHumanInWorkflow({
    type: "deploy-approval",
    name: `Deploy ${version}`,
    actions: [
      { id: "approve", title: "Deploy now" },
      { id: "reject", title: "Block deploy" },
    ] as const,
    context: {
      data: { version },
    },
  });

  if (result.actionId === "reject") {
    throw new Error("Deploy blocked");
  }

  return { deployed: true, version, handledBy: result.handledBy };
}
```

## `approveByHumanInWorkflow`

Fixed **approve** / **decline** actions:

```typescript
import { approveByHumanInWorkflow } from "robotrock/workflow";

export async function releaseGate(name: string) {
  "use workflow";

  const result = await approveByHumanInWorkflow({
    type: "release-gate",
    name,
    description: "Confirm release to production",
  });

  return result.actionId === "approve";
}
```

## Webhook security note

`createWebhook()` exposes a public URL at `/.well-known/workflow/v1/webhook/:token`. For stricter control, use `createHook()` + `resumeHook()` from a verified API route after `verifyRobotRockWebhook()`.

## Vercel AI SDK in workflows

```typescript
import { approveByHumanTool } from "robotrock/ai/workflow";

const approveByHuman = approveByHumanTool({
  mode: "workflow",
  app: "my-agent",
});
```

See [vercel-ai.md](vercel-ai.md) for tool approval bridge with `mode: "workflow"`.

## Workflow vs step level

| Capability | `"use step"` | `"use workflow"` |
|------------|--------------|------------------|
| `createWebhook()` | No | Yes |
| `sendToHumanInWorkflow` | No (call from workflow) | Yes |
| `approveByHumanTool({ mode: "workflow" })` | No | Yes |
