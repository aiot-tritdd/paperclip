# WHAT IS PAPERCLIP ?

Paperclip is a platform that runs AI agents for you

> You have a pile of work tickets (bugs, features, tasks). Instead of human managing them. You assign AI to do that. Paperclip is system that manages those agents - spawing them, tracking their work, stopping them when they fail, let you watch they work

**AI worker management systems**

---

# Overview

It's a platform to defining, running, monitoring autonomous agents against issues and projects and company

## The Three Big Concepts

### 1. Company = Workspace

Everything in Paperclip lives inside a company.

* Issues
* Agents (every agent is scoped into a company)
* Projects
* Settings
* Url
* Piece of data

### 2. Issues = Work Items

It's like Ticket I think.

An Issues is the task that an AI agent works on. It's a Github issue or Jira Ticket - and AI agents pick up it

* Well you (the BOARD) can assign it
* Or an AI agents can create it and assign it to others

The agent reads the issue, does the work (writes code, searches files, call APIs), and reports back.


  server/src/services/issues.ts

  An Issue is Paperclip's fundamental unit of work. It's not just a ticket — it has a state machine with guarded transitions.

  Key function: assertTransition → proves issues move through states (open → in_progress → done, etc.) with rules.

  Also key: enrichCommentsWithDerivedAgentAttribution — comments on issues are how agents communicate. Every agent interaction is a comment
  thread.

  ▎ Why this matters: Understand issues = understand how agents receive tasks, report progress, and signal completion.

### 3. Agents = AI Workers

An Agent is the AI that does the work. You define it (what model to use, what plugins it has, how it behaves v.v.) and Paperclip can assigns it to issues

  Here's the beautiful part — agents don't push work to the server. The server claims them via a heartbeat loop:

  Server: "Any agent ready to work?" → claims an issue → runs the agent
  Server: "Any agent ready to work?" → claims another → runs it
  ...repeat forever

  This design (from heartbeat.ts) prevents two agents from grabbing the same issue simultaneously. Smart, right?

---

| Layer              | Path          | Purpose                                                         |
| ------------------ | ------------- | --------------------------------------------------------------- |
| **Server**   | `server/`   | Node.js API — route handlers, business services, agent runtime |
| **UI**       | `ui/`       | React 19 SPA — control plane dashboard                         |
| **CLI**      | `cli/`      | Command-line client for company/export operations               |
| **Packages** | `packages/` | Shared types, plugin SDK, and built-in plugins                  |

The system is designed around **company-scoped workspaces**. Every entity (agent, issue, project, routine) belongs to a company, and all routing — both HTTP and client-side — is prefixed with a company slug.

  ★ Insight ─────────────────────────────────────

- The server/src/services/heartbeat.ts file is massive — it's the core engine of the entire platform. Over 9,000+ lines. That's the heartbeat that drives
  everything.
- The recovery service (recovery/service.ts) exists because agents CAN stall or crash. It detects this and re-queues or cancels the stuck run — like a safety net.
- Plugins run in a sandboxed Node VM on the server — meaning even if a plugin has bad code, it can't crash the whole system.
  ─────────────────────────────────────────────────

  So now you actually understand the full picture:

1. You create an issue → it sits in the waiting room (queued)
2. The heartbeat (triage nurse) wakes up every few seconds, looks at the waiting room, and says "okay, Agent A — take this issue"
3. The agent picks it up and starts working

---

  🎹 The Simplest Mental Model

  Imagine a music conductor (the heartbeat) standing in front of an orchestra (the agents). The conductor:

- Picks which musicians play (claims a run)
- Keeps time (heartbeat interval)
- Stops the music if something goes wrong (recovery service)
- Lets you plug in new instruments (plugins)

  You sit in the audience (the UI), watching the performance unfold in real time.

---

---

## 3. Execution Pipeline — The Engine

>   server/src/services/workspace-runtime.ts
>   packages/adapter-utils/src/server-utils.ts
>   server/src/services/environment-execution-target.ts

  This is how a task actually runs. Flow:

  Heartbeat detects issue ready
    → resolveEnvironmentExecutionTarget()
    → buildPaperclipEnv()          ← builds the agent's environment
    → Adapter executes the run
    → releaseIssueExecutionAndPromote()  ← promotes result back

  ExecutionWorkspaceInput, RuntimeServiceRef — these are the contracts for spinning up an isolated agent workspace.


##  4. Adapters — The Brains

  packages/adapters/codex-local/
  packages/adapters/openclaw-gateway/
  packages/adapter-utils/

  Paperclip is model-agnostic. Adapters are the plugins that connect to actual AI execution backends. Think of them as drivers — same
  interface, different engines.

- codex-local → runs against local Codex
- openclaw-gateway → runs via a WebSocket gateway (GatewayWsClient)

  ▎ The wisdom: Paperclip doesn't care who runs the task. It just hands the workspace to the adapter.

---

## 🔁 5. Routines — The Scheduler

  server/src/services/routines.ts
  server/src/services/plugin-managed-routines.ts

  Routines are recurring tasks — cron-style triggers that auto-create issues on a schedule.

  tickScheduledTriggers()   ← called by heartbeat
  createTrigger()
  updateTrigger()

  Plugin-managed routines let plugins register their own recurring behaviors.

---

## 🧩 6. Plugin System — The Extensibility Layer

  server/src/services/plugin-host-services.ts
  packages/plugins/sdk/src/types.ts

  Plugins can:

- Create and comment on issues (createComment, requestWakeup)
- Pause and resume execution
- Manage their own routines
- Inject UI slots (ui/src/plugins/slots.tsx)

  PluginDatabaseClient gives plugins scoped DB access. ToolRunContext is what a plugin sees when it runs a tool.

## 7. Auth — The Gatekeeper

  server/src/auth/better-auth.ts
  server/src/routes/auth.ts
  cli/src/commands/client/auth.ts

  Uses better-auth library. resolveBetterAuthSessionFromHeaders is the entry point — every API request passes through here.

  The CLI also has its own auth flow (registerClientAuthCommands).

---

  🗺️ The Full Mental Map

  ┌─────────────────────────────────────────────┐
  │                  PAPERCLIP                  │
  │                                             │
  │  Auth ──→ Routes ──→ Services               │
  │                         │                   │
  │              ┌──────────┴──────────┐        │
  │              ▼                     ▼        │
  │          Heartbeat ──────────→  Issues      │
  │              │                  (state)     │
  │              ▼                     │        │
  │          Routines            Execution      │
  │         (schedule)           Pipeline       │
  │                                   │         │
  │                          Adapters (AI)      │
  │                                             │
  │              Plugin System (hooks in ↑)     │
  └─────────────────────────────────────────────┘
