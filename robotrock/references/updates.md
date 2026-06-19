# Updates

Send short status updates (1-2 sentences) to a thread. The newest update shows in a status bar at the top of the inbox task detail, with an icon and color from an optional `status`. Every update is logged and viewable in an expandable history.

Updates are **thread-scoped** — they attach to a `threadId`, not a single task.

## Send a standalone update

`client.sendUpdate({ threadId, message, status? })` logs an update against an existing thread and returns it.

```typescript
import { robotrock } from "@/lib/robotrock";

const update = await robotrock.sendUpdate({
  threadId: "thread_123", // from task.threadId
  message: "Deployment started, running smoke tests.",
  status: "running",
});

// update: { id, threadId, message, status, source, createdAt }
```

The thread must already have at least one task in your tenant, otherwise the API returns `404`.

## Send an update at task creation

Pass `update` to `sendToHuman` (or `createTask`). It logs an initial update against the new task's thread.

```typescript
await robotrock.sendToHuman({
  type: "deploy-approval",
  name: "Approve production rollout",
  threadId: `deploy_${deploymentId}`,
  update: { message: "Build finished, awaiting approval.", status: "waiting" },
  actions: [{ id: "approve", title: "Approve" }],
});
```

`update` is a **top-level field** — like `threadId` and `assignTo`, it is not inside `context`.

## Status values

`status` is optional and defaults to `info`:

| Status | Meaning |
|--------|---------|
| `info` | Neutral informational message (default) |
| `queued` | Queued, not started |
| `running` | In progress |
| `waiting` | Blocked / waiting on input |
| `succeeded` | Completed successfully |
| `failed` | Errored / failed |
| `cancelled` | Aborted / cancelled |

## Field reference

| Field | Type | Notes |
|-------|------|-------|
| `threadId` | `string` | Required for `sendUpdate`. |
| `message` | `string` | 1-2 sentences. Max 500 characters. |
| `status` | enum (optional) | One of the values above. Defaults to `info`. |

## Typical pattern

Report progress against the same thread an agent created:

```typescript
const { task } = await robotrock.createTask({
  type: "job",
  name: "Nightly export",
  actions: [{ id: "ack", title: "Acknowledge" }],
  update: { message: "Export queued.", status: "queued" },
});

const threadId = task.threadId;

await robotrock.sendUpdate({ threadId, message: "Export running.", status: "running" });
// ... later
await robotrock.sendUpdate({ threadId, message: "Export complete.", status: "succeeded" });
```
