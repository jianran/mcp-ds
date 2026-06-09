# COORD — Coordination Protocol for Autonomous Agents v0.1

**The HTTP of the agent stack.**

A standard protocol for how agents coordinate work: task definition, lifecycle, state handoff, dependency resolution, escalation, and human-in-the-loop — framework-agnostic and transportable across any runtime.

---

## Where It Fits

```
L4  Application         CrewAI / LangGraph / Claude SDK
                          ↑ COORD ← this
L3  Communication       A2A (transport) + MCP (tools)
                          ↑ MCP-DS
L2  Runtime             LangGraph / OpenAI SDK / Smolagents
L1  Models              GPT-4o / Claude / Gemini
```

- **A2A** transports messages between agents (like TCP)
- **COORD** defines *what those messages mean* (like HTTP)
- Frameworks implement COORD and interoperate for free

---

## 1. Core Model

### Task — the atomic unit of work

A task is a sealed envelope. Any agent that understands COORD can accept it, process it, and return it.

```json
{
  "spec_version": "0.1",
  "task_id": "tsk_01j5c8x9y0",
  "goal": "Research competitor pricing",
  "meta": {
    "trace_id": "trc_abc123",
    "workflow_id": "wf_001",
    "created_at": "2026-06-09T21:00:00Z"
  },
  "inputs": {
    "companies": ["OpenAI", "Anthropic"],
    "detail_level": "pricing_tiers"
  },
  "dependencies": {
    "parent": "tsk_01j5c8x9wz",
    "blocked_by": [],
    "priority": "normal"
  },
  "state": {
    "pointer": "memory://wf_001/state",
    "scope": "workflow"
  },
  "constraints": {
    "deadline": "2026-06-10T10:00:00Z",
    "max_tokens": 50000,
    "allowed_tools": ["web_search", "filesystem_read"],
    "budget_max_usd": 0.50
  },
  "output_contract": {
    "format": "markdown",
    "schema_ref": "https://schemas.example.com/research-report.json",
    "required": ["summary", "table", "sources"]
  },
  "escalation": {
    "on": ["timeout", "error", "uncertainty"],
    "to": "agent:coordinator::workflow_001",
    "auto_retry": 2
  }
}
```

### Agent — a task processor

An agent is anything that speaks COORD:

- An LLM with tools
- A deterministic pipeline
- A human approval gate
- A sub-orchestrator (hierarchical)

An agent advertises its capabilities with an **Agent Card**:

```json
{
  "agent_id": "agent:researcher::v2",
  "name": "Research Agent",
  "description": "Web research and report generation",
  "protocols": ["COORD/0.1", "A2A/1.0"],
  "accepts": [
    { "goal_pattern": "Research *", "tags": ["research", "analysis"] },
    { "goal_pattern": "Find information about *", "tags": ["research"] }
  ],
  "state_access": ["memory://wf_*/state", "memory://wf_*/research_*"],
  "constraints": {
    "max_concurrent": 5,
    "max_duration_s": 300,
    "requires_mcp": ["web_search", "filesystem_read"]
  },
  "auth": {
    "required": ["coordinator::workflow_001"]
  }
}
```

---

## 2. Coordination Lifecycle

The lifecycle every agent agrees on:

```
                      ┌──────────────────────────┐
                      │       CANCELLED           │
                      └──────────────────────────┘
                                ▲
                                │
┌──────────┐    ┌──────────┐   │   ┌──────────┐    ┌──────────┐
│ CREATED  │───▶│ ASSIGNED │───┴──▶│ IN_FLIGHT│───▶│ COMPLETED│
└──────────┘    └──────────┘       └──────────┘    └──────────┘
                                       │  │              │
                                       │  ▼              │
                                       │  ┌──────────┐   │
                                       ├─▶│ NEED_HELP│───┤
                                       │  └──────────┘   │
                                       │                 │
                                       ▼                 ▼
                                  ┌──────────┐      ┌──────────┐
                                  │  FAILED  │      │ REVIEW   │──▶ COMPLETED
                                  └──────────┘      └──────────┘
                                       │                  │
                                       ▼                  ▼
                                  ┌──────────┐      ┌──────────┐
                                  │ESCALATED │      │ REJECTED │
                                  └──────────┘      └──────────┘
                                       │
                                       ▼
                                  ┌──────────┐
                                  │REASSIGNED│──▶ IN_FLIGHT
                                  └──────────┘
```

**State definitions:**

| State | Meaning | Emitted by |
|---|---|---|
| `CREATED` | Task exists, not yet assigned | Orchestrator or coordinator agent |
| `ASSIGNED` | Bound to a specific agent | Coordinator |
| `IN_FLIGHT` | Agent is actively working | Processing agent |
| `NEED_HELP` | Agent hit uncertainty, requesting guidance | Processing agent |
| `REVIEW` | Work complete, awaiting human/coordinator approval | Processing agent |
| `COMPLETED` | Successfully finished | Processing agent or reviewer |
| `FAILED` | Irrecoverable error | Processing agent |
| `ESCALATED` | Escalated to parent coordinator | System |
| `CANCELLED` | Aborted (user or parent coordinator) | Any authorized agent |
| `REJECTED` | Review failed, not salvageable | Reviewer |
| `REASSIGNED` | Re-routed to a different agent | Coordinator |
| `DELEGATED` | Passed to a sub-agent | Processing agent (hierarchical only) |

**State transitions are the key protocol constraint.** An agent receiving a task in `CREATED` knows exactly what to do with it. An agent receiving a `COMPLETED` task knows not to touch it. This is the TCP state machine for agents.

---

## 3. State Handoff Protocol

The most expensive mistake in current multi-agent systems: **copying the entire context** between agents.

COORD defines **durable state pointers** instead:

```
state.pointer → a URI that resolves to durable state
state.scope   → how visible the state is
```

### State Scopes

| Scope | Visibility | Lifetime | Example |
|---|---|---|---|
| `task` | This task only | Task lifecycle | Intermediate results |
| `workflow` | All agents in this workflow | Workflow lifecycle | Shared research notes |
| `user_session` | Across workflows for a user | User session | User preferences |
| `global` | All agents (careful!) | Permanent | Registered capabilities |

### State operations

```json
{
  "state_op": "read",
  "pointer": "memory://wf_001/state",
  "version": 3
}
→ { "company_focus": "OpenAI", "depth": "deep" }

{
  "state_op": "write",
  "pointer": "memory://wf_001/research_openai",
  "value": { "pricing": "$0.15/1M input", "models": ["GPT-4o", "o3"] },
  "scope": "workflow"
}
```

### Why pointers beat history-passing

- Agent A writes result to `memory://wf_001/research_openai`
- Agent B reads from the same pointer
- No shared context on A's side
- B can read in parallel without coupling
- Coordinator can see all writes without any agent forwarding them
- You get git-like versioning — roll back to "before Agent D wrote to state"

---

## 4. Coordination Graph

A workflow is a DAG of tasks with dependency edges:

```json
{
  "workflow_id": "wf_001",
  "goal": "Generate competitive analysis report",
  "tasks": [
    { "task_id": "tsk_01", "goal": "Research OpenAI pricing" },
    { "task_id": "tsk_02", "goal": "Research Anthropic pricing" },
    { "task_id": "tsk_03", "goal": "Research Google pricing" },
    { "task_id": "tsk_04", "goal": "Write report",
      "depends_on": ["tsk_01", "tsk_02", "tsk_03"] },
    { "task_id": "tsk_05", "goal": "Review and approve",
      "depends_on": ["tsk_04"],
      "hitl": true }
  ],
  "state": {
    "pointer": "memory://wf_001/state",
    "shared_keys": {
      "research_*": "any agent can read findings",
      "report_draft": "only writer + reviewer"
    }
  }
}
```

The coordinator resolves dependencies: tsk_01, tsk_02, tsk_03 run in parallel. tsk_04 waits for all three. tsk_05 waits for tsk_04 and requires human approval.

---

## 5. Human-in-the-Loop

A standard format for any agent to request human intervention:

```json
{
  "task_id": "tsk_05",
  "state": "NEED_HELP",
  "hitl_request": {
    "type": "approval",
    "message": "The draft report recommends lowering prices by 15%. Approve?",
    "context": {
      "summary": "OpenAI charges $0.15/1M, Anthropic $0.12/1M...",
      "recommendation": "Drop to $0.10/1M to undercut",
      "tradeoffs": "Margin drops from 60% to 55%, but volume-estimated +40%"
    },
    "options": [
      { "id": "approve", "label": "Approve recommendation" },
      { "id": "modify", "label": "Suggest different price", "input_required": true },
      { "id": "reject", "label": "Keep current pricing" }
    ],
    "timeout_s": 86400
  }
}
```

HITL is just another state transition: `IN_FLIGHT → NEED_HELP`. The coordinator handles routing it to a human interface. The agent doesn't know or care if the human uses Slack, email, or a web dashboard.

---

## 6. Error Handling & Escalation

Agents *will* fail. The protocol defines standard failure modes:

| Error Type | Meaning | Default Action |
|---|---|---|
| `timeout` | Exceeded `max_duration_s` | Retry once, then escalate |
| `tool_failure` | MCP tool returned error | Retry with backoff |
| `uncertainty` | Agent lacks confidence to proceed | Escalate with `NEED_HELP` |
| `invalid_input` | Task envelope doesn't match capabilities | Fail immediately |
| `resource_exhausted` | Hit rate limits, budget limits, or token limits | Escalate |
| `state_conflict` | Version conflict on state write | Retry with newer version |
| `coordinator_unreachable` | Can't report state back | Buffer and retry |

Escalation always goes back to the coordinator, which decides: retry, reassign, skip, or cancel.

---

## 7. Transport Mapping

COORD is transport-agnostic. It defines semantics, not delivery.

| Transport | How COORD rides on it |
|---|---|
| **A2A** | COORD task envelope wraps A2A `Task.message`, `state` field maps to A2A state pointer |
| **HTTP** | POST `{task}` → `{status, task_id}` on `/coord/v1/tasks` |
| **gRPC** | rpc SubmitTask(CoordTask) returns (CoordStatus) |
| **MCP tools** | A `coord_execute` tool resource |

Default recommendation: **COORD over A2A.** A2A handles transport guarantees (delivery, retry, auth). COORD defines what's inside the envelope.

---

## 8. Framework Interop Example

```yaml
# WORKFLOW definition (portable)
workflow:
  id: wf_001
  goal: "Weekly competitive analysis"
  tasks:
    researcher:
      goal: "Research competitors"
      accepts_pattern: "Research *"
      tools: [web_search, filesystem]
    writer:
      goal: "Draft report"
      accepts_pattern: "Draft *report*"
      depends_on: [researcher]
      tools: [filesystem_write]
    reviewer:
      goal: "Approve report"
      depends_on: [writer]
      hitl: true
```

This same workflow could run on:

- **CrewAI**: maps `researcher`/`writer`/`reviewer` to CrewAI agents
- **LangGraph**: maps to graph nodes
- **Claude SDK**: maps to sequential agent calls
- **Custom**: maps to function calls

The protocol guarantees the task envelope, state format, and lifecycle are identical. Only the routing differs.

---

## Next

- v0.1 — Core model + lifecycle + state handoff
- v0.2 — Dependency graph + coordination
- v0.3 — HITL + escalation + error handling
- v1.0 — Production spec + reference implementations
