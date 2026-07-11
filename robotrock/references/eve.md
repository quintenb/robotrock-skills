# Eve agents (dashboard chat)

Integration for [Eve](https://eve.dev) deployments that chat in the RobotRock inbox.

**There is no `npx eve init robotrock` template.** Eve only scaffolds a generic agent:

```bash
npx eve init my-agent
cd my-agent
bun add robotrock eve
```

`eve init robotrock` would create an agent **named** `robotrock`, not a RobotRock-ready template. Wire RobotRock with the SDK steps below (new agent or existing deployment).

Load the **eve** skill (`node_modules/eve/docs/`) for Eve authoring; use this reference for RobotRock-specific wiring.

## SDK entry points (v1.1+)

| Import | Purpose |
|--------|---------|
| `robotrock/eve` | Input mapping, `buildTaskHandledResumeMessage` (no `eve` peer) |
| `robotrock/eve/agent` | Channel, auth, hooks, session client, inbox task APIs |
| `robotrock/eve/tools` | `createRobotrockTools()` bundle |
| `robotrock/eve/tools/inbox/create-task` | Inbox delegation (`create_robotrock_task`) |
| `robotrock/eve/tools/identity/whoami` | Dashboard user identity |
| `robotrock/eve/tools/identity/my-access` | Role and group capabilities |

## Minimal file layout

After `eve init`, add thin re-exports (one line each):

```typescript
// agent/hooks/robotrock-ready.ts
export { robotrockReadyHook as default } from "robotrock/eve/agent";
```

```typescript
// agent/hooks/robotrock-chat-audit.ts
export { robotrockChatAuditHook as default } from "robotrock/eve/agent";
```

```typescript
// agent/channels/eve.ts
import { robotrockEveChannel } from "robotrock/eve/agent";
import { myWebappAuth } from "../lib/auth.js"; // optional — existing webapp

export default robotrockEveChannel({
  auth: [myWebappAuth()], // stack after RobotRock user-context auth
});
```

```typescript
// agent/tools/create_robotrock_task.ts
export { createInboxTaskTool as default } from "robotrock/eve/tools/inbox/create-task";
```

Optional tools:

```typescript
export { whoamiTool as default } from "robotrock/eve/tools/identity/whoami";
export { myAccessTool as default } from "robotrock/eve/tools/identity/my-access";
```

Customize inbox delegation:

```typescript
import { defineCreateInboxTaskTool } from "robotrock/eve/tools/inbox/create-task";

export default defineCreateInboxTaskTool({
  resolveDelegationReason(input, caller) {
    // optional business rules
    return "Routed to finance because …";
  },
});
```

## RobotRock-ready detection

On connect, RobotRock reads `GET /eve/v1/info`. The deployment is **RobotRock-ready** when the manifest includes the **`robotrock-ready`** hook slug (from `robotrockReadyHook` above). No manual toggle in Settings.

## Environment

### Self-hosted (one workspace per deployment)

Set after connecting in **Settings → Agents**:

| Variable | Purpose |
|----------|---------|
| `ROBOTROCK_API_KEY` | Workspace `ll_*` key |
| `ROBOTROCK_USER_CONTEXT_SECRET` | Verify dashboard user-context HMAC JWTs |
| `ROBOTROCK_BASE_URL` | API base (optional; default production) |
| `ROBOTROCK_APP` | Default inbox app bucket |

### Multi-tenant (developer registry)

| Variable | Purpose |
|----------|---------|
| `ROBOTROCK_AGENT_SERVICE_TOKEN` | `ras_*` from **Developer → Agent deployments** |
| `ROBOTROCK_BASE_URL` | Optional; local dev (`http://localhost:4001/v1`) |

Platform user-context JWT verification fetches the public key from `/.well-known/robotrock-user-context-public-key` automatically.

## Existing Eve agent + custom webapp

RobotRock does **not** add a second channel. Extend the single `agent/channels/eve.ts` with `robotrockEveChannel({ auth: [existingAuth] })`. RobotRock uses `X-RobotRock-User-Token`; your webapp keeps its own `Authorization` / session auth. Sessions stay separate unless you share `sessionId` intentionally.

## Dashboard chat flow

1. Connect deployment URL in **Settings → Agents**.
2. Users chat via dashboard proxy (`/{tenant}/api/eve/{connectionId}`).
3. Gated tools and `ask_question` park with `input.requested` → inline buttons in chat.
4. `robotrockChatAuditHook` logs HITL choices to the audit trail (RobotRock sessions only).

For inbox tasks delegated to other people, use `create_robotrock_task` — see [send-to-human.md](send-to-human.md).

## `ask_question` (mandatory for choices)

Inline buttons appear only when Eve emits `input.requested` from `ask_question`. Prose option lists do **not** render buttons.

```ts
await ask_question({
  prompt: "Which path?",
  options: [
    { id: "polling", label: "Polling" },
    { id: "webhooks", label: "Webhooks" },
  ],
});
```

- ≤6 options → one button each
- After calling `ask_question`, do not repeat options in assistant text

## Eve input display modes

| Eve `display` | Meaning | Chat UI |
|---------------|---------|---------|
| `confirmation` | Gated tool approval | Approve / Deny |
| `select` | `ask_question` with options | Buttons or radio form |
| `text` | Open `ask_question` | Text field |

## Checklist

- [ ] `bun add robotrock` (and `eve` if not already)
- [ ] `robotrock-ready` hook re-export
- [ ] `robotrockEveChannel` in `agent/channels/eve.ts`
- [ ] Env vars for deployment kind (self-hosted vs multi-tenant)
- [ ] Optional: `robotrock-chat-audit` hook, inbox/identity tools
- [ ] Use `ask_question` for user choices; gate dangerous tools in Eve
- [ ] Connect deployment in RobotRock **Settings → Agents**

Reference implementation: [robotrock-agent](https://github.com/quintenb/robotrock/tree/main/apps/robotrock-agent) (`https://robotrock-agent.vercel.app`).
