# OpenTelemetry + RobotRock feedback loop

RobotRock does not ingest OTLP. Export full traces to your observability stack and snapshot compact telemetry on `sendToHuman`.

## Dual export

1. **Full traces** → OTel collector (Jaeger, Datadog, Trigger.dev dashboard, etc.)
2. **Compact snapshot** → `agent` field on `sendToHuman` → Statistics + feedback analysis

## SDK helper

```bash
bun add robotrock @opentelemetry/api
```

```typescript
import { createClient, agentTelemetryFromOtel } from "robotrock";

const runStartedAt = Date.now();
// ... instrumented agent work ...

await robotrock.sendToHuman({
  type: "deploy-approval",
  name: "Approve production deploy",
  actions: [
    { id: "approve", title: "Approve" },
    { id: "reject", title: "Reject" },
  ],
  agent: agentTelemetryFromOtel({
    version: process.env.AGENT_VERSION,
    runStartedAt,
    toolCalls: { grep: 4, readFile: 2 },
    cost: { tokens: { total: 5000 } },
    extraInfo: { model: "claude-sonnet-4" },
    otel: {
      spans: [
        { name: "tool.grep", durationMs: 100, status: "ok" },
        { name: "tool.readFile", durationMs: 50, status: "error" },
      ],
    },
  }),
});
```

`agentTelemetryFromOtel()` fills:

- `agent.version`, `agent.toolCalls`, `agent.cost` from your options
- `agent.info.traceId`, `spanId`, `durationMs`, `errorCount`
- `agent.otel` structured span summary (`traceId`, `rootDurationMs`, `spans`)

Optional: `toolCallsFromOtelSpans(spans)` to derive `toolCalls` from span names.

## MCP improvement loop

Before changing agent code:

1. Call `get_feedback_analysis` with the same `app` and `type` as `send_to_human`
2. When `status` is `completed` and `isHealthy` is `false`, apply `agentInstructions`
3. Trigger new analyses from RobotRock dashboard → Statistics → Analyze feedback

The analysis LLM correlates human action data (`handled.action.id`, feedback form fields) with agent telemetry (version, cost, tool calls, OTel duration/errors) to produce concrete improvement instructions.

## Trigger.dev

Trigger.dev exports OTel to its platform automatically but does **not** fill RobotRock `agent`. Pass `agent` on your `sendToHumanTask` payload or wrap the SDK call:

```typescript
import { agentTelemetryFromOtel } from "robotrock";

await client.sendToHuman({
  ...taskInput,
  agent: agentTelemetryFromOtel({ version: process.env.AGENT_VERSION, runStartedAt }),
});
```
