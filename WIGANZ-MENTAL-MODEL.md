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
