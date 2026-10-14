# H-DOTS Architecture — Working Design

![H-DOTS Simplified Architecture](H-DOTS-Simplified-Architecture.png)

## 1. Purpose

H-DOTS is the persistent **Programme Manager** for the personal HAL environment. It owns the long-term goals, maintains the overall picture, coordinates long-running experts, commissions temporary project work, and preserves continuity across Hermes sessions.

The central design principle is:

> **The agent identity and operational state are persistent; individual Hermes sessions are replaceable.**

A session may continue for a long time, but when it approaches Hermes compression it should perform a **HANDOFF**, start a new session, reload its responsibilities plus the HANDOFF, and continue with a fresh context.

---

## 2. Main architectural layers

### H-DOT — Programme Manager

H-DOT sits at the centre of the architecture and owns the long-term view. It should remember:

- long-term goals and priorities
- the identities and responsibilities of long-running experts
- the current Hermes `session_id` for each long-running expert
- active transient projects
- important programme-level outcomes and status
- references to relevant HANDOFFs and artefacts

H-DOT is the initial entry point for new requests unless an external conversation has already been routed directly to an expert.

### Long-Running Experts

The next layer consists of persistent expert agents with clearly bounded responsibilities, tools and skills appropriate to their domain. Examples include:

- Tech PM
- CBR / Personal Researcher
- Finance
- Health
- Personal Assistant
- Smart Home
- Security
- other experts created as required

A long-running expert:

- has a stable identity and explicit responsibilities
- has a Hermes profile with relevant tools and skills
- owns a continuing Hermes session until rotation is needed
- may create and manage transient projects
- must reject or re-route requests that are outside its domain
- performs a HANDOFF before session compression
- returns its new `session_id` after rotation so H-DOT can continue to address it

H-DOT may create a new long-running expert when a genuinely recurring or strategically important domain appears.

### Transient Projects

Transient projects are temporary agents created to perform specific work on behalf of a long-running expert.

They exist for a finite objective and normally disappear when the work is complete. The long-running expert acts as the project manager for these agents.

A transient project should retain only the state needed to complete its objective, for example:

- project objective and success criteria
- assigned tasks
- current status
- dependencies and blockers
- outputs and artefacts
- final result / completion summary

#### Tech PM exception

The Tech PM is a long-running expert but has an established group of software-development agents beneath it. These agents build applications and manage the technical environment rather than being treated as ad-hoc general project agents.

---

## 3. External interfaces

### Home Assistant Voice PE

A second wake word on the Home Assistant Voice PE can route a request directly to H-DOTS.

Conceptually:

```text
Second HA wake word
        ↓
Session Registry
        ↓
current H-DOTS Hermes session
```

Ordinary Home Assistant voice commands can continue to use the normal HA routing path.

### OMI

For the time being, the OMI button-click workflow is set aside because of the current OMI bug.

The existing Mac mini OMI integration already extracts explicit commands using the `computer on` / `computer off` markers. That path can later be redirected from Home Assistant to the H-DOTS Session Registry.

Conceptually:

```text
OMI
 ↓
Mac mini OMI adaptor
 ↓
explicit command extracted
 ↓
Session Registry
 ↓
H-DOTS
```

---

## 4. Session Registry

The Session Registry sits in front of Hermes for external H-DOTS communication.

Its purpose is to hide raw Hermes session management from clients such as Voice PE and OMI.

The important distinction is between:

- **external conversation ID** — identifies the user's continuing dialogue
- **Hermes session ID** — identifies the current execution session of the agent handling that dialogue

A simple mapping might look like:

```text
conversation_id: C123
agent_id: Personal_Assistant
hermes_session_id: PA7
```

The registry should maintain at least:

```text
agent_id
profile
current_session_id
handoff_path
status
updated_at
```

External clients do not need to know Hermes session IDs. They simply send a request to a stable target such as H-DOTS, together with their external conversation ID.

---

## 5. Incoming request routing

### Case 1 — first request

A new conversation initially goes to H-DOTS.

```text
User request
   ↓
Session Registry
   ↓
H-DOTS
```

H-DOTS determines whether it should:

- answer directly
- route the user to an existing long-running expert
- use the General Knowledge Expert
- create a new specialist expert if justified

### Case 2 — continuing conversation with an expert

If H-DOTS routes the request to an expert, the Session Registry updates the external conversation mapping.

```text
conversation C123
       ↓
Personal Assistant
       ↓
Hermes session PA7
```

Subsequent utterances can then go directly to that expert without passing through H-DOTS each time.

### Case 3 — user's next utterance changes domain

Each expert must have a strict **domain contract**.

For every incoming utterance, the current expert checks:

> **Is this clearly within my responsibility?**

If yes, it continues the conversation.

If no, or if uncertain, it returns the utterance to H-DOTS for re-routing.

This prevents an expert from retaining a conversation simply because it handled the previous turn.

### Case 4 — no suitable expert exists

H-DOTS should not create a persistent expert for every unfamiliar question.

Use three levels:

1. **H-DOTS answers directly** for simple one-off questions.
2. **General Knowledge Expert** handles broader one-off queries that benefit from a generalist.
3. **Agent Provisioning / HR Agent** creates a new long-running expert only where the domain is recurring, strategically important, or requires specialist tools, skills or persistent memory.

A useful rule is:

> **Create a Long-Running Expert only when there is an ongoing responsibility, not simply because a query is unfamiliar.**

---

## 6. Session continuity and HANDOFF

A long-running expert should remain in the same Hermes session while practical.

When the session approaches the Hermes compression threshold:

```text
normal session
     ↓
context approaches threshold
     ↓
HANDOFF plugin detects condition
     ↓
agent creates HANDOFF
     ↓
important/relevant state preserved
     ↓
new Hermes session created
     ↓
responsibilities + HANDOFF loaded
     ↓
new session_id returned to H-DOTS / Registry
```

The HANDOFF is the **durable reconstructed working context**. It is not intended to be a transcript.

It should contain only what is needed to continue effectively, including:

- responsibilities
- current goals
- active work
- important decisions and rationale
- dependencies and blockers
- key facts currently relevant
- artefact references
- open actions
- next steps

As it grows, the agent should periodically consolidate it so that it remains bounded and useful.

A useful principle is:

> **Session grows freely; HANDOFF grows cautiously; consolidation keeps HANDOFF bounded.**

---

## 7. Detecting imminent compression

The existing Kanban HANDOFF plugin provides a useful starting point, but ordinary sessions need a separate session-oriented implementation.

Preferred structure:

```text
shared HANDOFF core
       ├── kanban-handoff adapter
       └── session-handoff adapter
```

The ordinary-session version needs to:

1. inspect the real outgoing model context
2. measure context/token use
3. trigger before Hermes' own compression threshold
4. force HANDOFF creation
5. create a replacement session
6. update the Session Registry
7. load responsibilities + HANDOFF into the new session
8. resume transparently

`llm_request` middleware is the likely interception point because it sees the actual outbound request before the model call.

---

## 8. Wake mechanisms

Different persistent entities may be woken for different reasons.

```text
Explicit human input       → H-DOTS
Periodic supervisory timer → H-DOTS / PA
Research timer             → CBR
Project timer              → relevant PM / expert
Watch condition satisfied  → Scout → original requester
A2A message                → target expert
```

### Timers

Timers should belong to the entity whose responsibility requires them.

Examples:

- H-DOTS may periodically gather status from experts.
- CBR may run scheduled research sweeps.
- a Project Manager may wake to review deadlines or stalled work.

### Scout

Scout is a monitoring service, not a decision-maker.

H-DOTS, PMs and CBR decide:

- what to monitor
- where to look
- the condition to detect
- why it matters
- who receives the result

Scout maintains the watch list, detects the requested condition, and returns the finding to the original requester, which has the context to decide relevance.

---

## 9. What each layer should remember

### H-DOTS should remember

- programme goals and priorities
- expert identities and responsibilities
- current expert session IDs
- active and recently completed transient projects
- cross-domain dependencies
- important programme-level outcomes
- agent creation/retirement decisions
- references to expert HANDOFFs

### Long-Running Experts should remember

- their domain contract and responsibilities
- relevant tools and skills
- domain-specific goals
- active project portfolio
- current working context
- important domain decisions
- dependencies and open actions
- HANDOFF state

### Transient Projects should remember

- project objective
- assigned work
- required inputs
- task/status state
- blockers
- artefacts/results
- completion summary

They do **not** need to become permanent memory once the work has been absorbed by the responsible long-running expert.

---

## 10. Learning is a separate concern

H-DOTS persistence and session continuity should not be confused with the learning pipelines already being developed.

There are two separate learning streams:

### Agent-learning infrastructure

Kanban activity, chat sessions, tool use, decisions and outcomes should eventually be captured so they can be reviewed and potentially used as training material with Unsloth.

The `conductor-workflow-controller` provides a starting point, but ordinary chat/session and Kanban capture still need extending.

### Home Assistant behavioural learning

The existing Home Assistant reflection and memory architecture is a separate learning system. It can later be extended beyond media to human-linked device behaviour such as motion, switches and other household interactions.

Neither learning stream should be made part of H-DOTS session persistence itself.

---

## 11. Implementation model

The current working model is:

```text
External client
(Voice PE / OMI / other)
        ↓
Session Registry
        ↓
external conversation → agent/session mapping
        ↓
Hermes API
        ↓
H-DOTS or selected Long-Running Expert
        ↓
reason / work / delegate
        ↓
Transient Projects / specialist agents as required
        ↓
HANDOFF before compression
        ↓
new Hermes session
        ↓
Session Registry updated
```

The important architectural outcome is that the user experiences a continuous agent, while Hermes is free to rotate the underlying execution sessions whenever necessary.

---

## 12. Core design principles

1. **H-DOTS is the Programme Manager, not the universal worker.**
2. **Long-Running Experts own continuing responsibilities.**
3. **Transient Projects perform finite work on behalf of experts.**
4. **Agent identity persists independently of Hermes sessions.**
5. **Session IDs are implementation details hidden behind the Session Registry.**
6. **External conversation IDs provide user-dialogue continuity.**
7. **Experts have strict domain boundaries and re-route out-of-domain requests.**
8. **New experts are created only for genuine ongoing responsibilities.**
9. **HANDOFF replaces uncontrolled compression for persistent agents.**
10. **Learning pipelines remain separate from operational persistence.**
