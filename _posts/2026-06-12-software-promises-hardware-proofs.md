---
layout: post
title: "Software Promises, Hardware Proofs: Governing an Industrial Robot Agent with AgenTrust"
date: 2026-06-12 00:30:00 +0200
categories: [runtime, embodied-ai]
tags: [embodied-ai, robotics, agentrust, cmcp, agent-manifest, trace, cedar, confidential-computing, tee, agent-governance, functional-safety, industrial, eu-ai-act]
description: >-
  The creator of the Microsoft Agent Governance Toolkit invited me to bring agent
  governance to embodied AI. This is the first working example: an industrial
  robot-cell agent governed by a signed manifest, default-deny policy and a
  verifiable audit trail, and an honest map of where software governance ends
  and machine safety begins.
canonical_url: "https://governance.ai-mvp.com/2026/06/12/software-promises-hardware-proofs/"
---

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true, theme: 'neutral', securityLevel: 'loose' });
</script>

# Software Promises, Hardware Proofs: Governing an Industrial Robot Agent with AgenTrust

*On June 23, Imran Siddique presents ["Governing AI Agents at the Hardware Boundary"](https://ccsummit2026.sched.com/event/2NKcF/governing-ai-agents-at-the-hardware-boundary-imran-siddique-microsoft) at the Confidential Computing Summit. His thesis fits in one line: software governance makes promises, hardware governance provides proofs. I spent the past days building the first industrial embodied-AI example for AgenTrust, the open-source stack behind that talk. This post walks through it: what gets proven, what stays a promise, and the exact line where agent governance must hand over to machine safety.*

## TL;DR

- **AgenTrust** is a new open-source agent-governance stack that builds on the lessons of the [Microsoft Agent Governance Toolkit](https://github.com/microsoft/agent-governance-toolkit): a signed **Agent Manifest** that binds an agent's prompt, policy and tool declarations to hashes; **cMCP**, a policy-enforcing MCP runtime with Cedar authorization and hash-chained audit; and **TRACE**, runtime-issued trust records you can verify offline. The full stack goes public around the June 23 summit; the [examples repository](https://github.com/agentrust-io/examples) is public today.
- I contributed an [industrial embodied-AI example](https://github.com/agentrust-io/examples/pull/16): a material-movement agent requesting motion from a (simulated) robot cell. It runs three paths: a declared workflow that completes, an undeclared workflow that dies at the policy gate, and the important one — **a request the policy authorizes and the controller still refuses, because a person entered the safeguarded area.**
- The central boundary, stated in the example itself: a cMCP `allow` means the software request was authorized. It does **not** mean a physical action is safe, accepted by the controller, or completed by a machine. Functional safety (ISO 13849, IEC 61508, ISO 10218) keeps owning the cell; agent governance sits above it and proves what was attempted, by whom, under which policy.
- Why hardware proofs matter: today's agent controls — policy engines, identity, audit logs — live *inside* the agent's trust boundary. If the runtime is compromised, policies can be bypassed and logs forged. Trusted execution environments move enforcement and evidence outside the blast radius. For agents with actuators, that is the difference between an audit log and admissible evidence.
- If verification is your thing: this Thursday, **June 18**, GenAI Gurus hosts [Marius Hobbhahn, CEO of Apollo Research, on what thousands of real agent traces reveal when coding agents go rogue](https://www.meetup.com/genai-gurus/events/314881167/). This post is about proving what an agent *may* do; that session is about watching what agents *actually* do. They are two halves of the same problem.

## Why this post exists

[Post #2 in this series](/2026/05/28/coding-agents-safely/) closed by crediting a May GenAI Gurus session with [Imran Siddique](https://www.linkedin.com/in/imransiddique1986/) on the Microsoft Agent Governance Toolkit and its policy-as-code approach. Two things happened since. Imran left Microsoft to become Chief Platform Officer at OPAQUE, working at the junction of agent governance and confidential computing. And he invited me to contribute to **AgenTrust**, the open-source project he is presenting at the Confidential Computing Summit — specifically to build out its coverage of *embodied AI systems in industrial environments*, working toward external maintainership of that area once the repositories are public.

This series has been walking toward that subject for a while, whether I planned it or not. [Post #1](/2026/04/10/eu-ai-act-compliance-checklist-for-ai-agent-developers/) was the paper layer: what the EU AI Act actually requires from agent developers before August 2, 2026. [Post #2](/2026/05/28/coding-agents-safely/) was the containment layer: how to run agents whose tools touch repositories and secrets. This post is the next step on the same road: agents whose tools move steel.

The contribution is [PR #16 on agentrust-io/examples](https://github.com/agentrust-io/examples/pull/16), and everything below describes that public example. One disclosure up front: I contribute in a personal capacity, and the example is fully synthetic — no robot hardware, no vendor SDK, no production endpoint, no proprietary industrial data. What is real is the governance path: every request in this post went through a live cMCP runtime, and the committed evidence files were captured from a real run.

## The scenario: a material-movement agent asks a robot to move

The setting is the most ordinary thing in any factory: an agent that schedules material movement asks a robot cell to move a pallet. Ordinary is the point. If governance cannot handle "move this pallet," it has nothing to say about humanoids.

Here is what the example contains, artifact by artifact:

| Artifact | What it is | Why it matters |
|---|---|---|
| `agent-manifest.json` (signed) | Binds the agent's system prompt, policy and tool declarations to hashes | The agent that ran is provably the agent you reviewed — prompt swaps and tool-list drift become detectable |
| `policy/allow.cedar` | Workflow-scoped Cedar permits, **default deny** | Anything not explicitly declared never reaches the robot side |
| `catalog.json` | Attested tool catalog for the robot-cell MCP server | The agent only sees tools that were declared and pinned |
| `server/mock_robot_controller.py` | An **independent** controller with fresh, authenticated, single-use state tokens | Stale, modified or replayed cell state is rejected; speed and zone limits are enforced outside the agent's reach |
| `trace-output/example-*.json` | Runtime-issued TRACE Trust Record + signed audit bundle from a real run | The decision trail verifies offline — you do not have to trust my laptop |
| `tests/` (7 passing) | Controller regression tests: stale, modified and replayed state, speed limits, approved zones | The safety-relevant refusals are tested behavior, not narrative |

## The architecture: two gates, one record

The design has two decision points on purpose, and they answer different questions. cMCP answers *"is this request allowed?"* The controller answers *"is this motion acceptable right now?"* Neither can answer the other's question, and the example treats that as a feature to be engineered, not a bug to be abstracted away.

<pre class="mermaid">
flowchart TB
    M["Agent Manifest (signed)<br/>binds prompt, policy and tool hashes"] -.-> B
    A["Material-movement agent<br/>(LLM planner)"] -->|"MCP tools/call"| B["cMCP Runtime<br/>attested tool catalog<br/>Cedar authorization, default deny<br/>hash-chained audit"]
    B -->|"denied"| X["Request stops here.<br/>Controller never invoked"]
    B -->|"authorized request"| C["Independent controller<br/>fresh single-use state token<br/>recheck of live cell state<br/>speed and zone limits"]
    C -->|"rejected"| Y["Motion never starts"]
    C -->|"accepted command"| D["Simulated robot execution"]
    B ==>|"session close"| T["TRACE Trust Record<br/>+ signed audit bundle"]
</pre>

Three details deserve attention because they generalize beyond this example:

1. **The manifest is signed and the catalog is attested.** Before any policy question is asked, the runtime can check that the agent's prompt, its policy and its tool declarations are exactly the ones that were reviewed. In a world of skill marketplaces and fleet-wide updates — where a behavior trained in one place executes in thousands of others — *"which agent is this, exactly?"* is the first governance question, not an afterthought.
2. **Policy is default deny and workflow-scoped.** The Cedar permits do not say what the robot may not do; they say the only things the agent may ask for. The undeclared request in run 2 below is not caught by a clever rule — it fails because nothing permits it.
3. **The controller does not trust the conversation.** It demands a fresh, authenticated, single-use state token and rechecks the physical state at decision time. A cell state observed thirty seconds ago, or replayed from an earlier request, is not evidence about now — and the regression tests verify that stale, modified and replayed state all get rejected.

## Three runs, one boundary

The agent runs three paths through the live runtime. Here they are as a sequence:

<pre class="mermaid">
sequenceDiagram
    participant A as Agent
    participant P as cMCP Runtime
    participant K as Controller
    participant R as Robot cell (simulated)
    rect rgb(232,245,233)
    Note over A,R: Run 1 — declared workflow, clear cell
    A->>P: move request (declared workflow)
    P->>P: manifest verified, Cedar: allow
    P->>K: authorized request + fresh state token
    K->>K: state, speed, zone: accept
    K->>R: execute
    R-->>A: completed
    end
    rect rgb(255,243,224)
    Note over A,P: Run 2 — undeclared workflow
    A->>P: request outside declared workflows
    P->>P: Cedar: default deny
    P-->>A: denied. Controller never invoked
    end
    rect rgb(255,235,238)
    Note over A,R: Run 3 — authorized, then a person enters the cell
    A->>P: move request (declared workflow)
    P->>P: Cedar: allow
    P->>K: authorized request + fresh state token
    K->>K: live state: human detected in safeguarded area
    K-->>A: rejected. Motion never starts
    end
</pre>

And here is the actual summary the agent prints, from a real run:

```text
SUCCESS
  cMCP policy: authorized
  controller: accepted
  execution: completed

POLICY DENY
  cMCP policy: denied
  controller: not invoked

SAFETY REJECT
  cMCP policy: authorized
  controller: rejected
  reason: human_detected
  execution: not_started

TRACE VERIFICATION
  schema/signature/hashes/freshness: verified
  audit bundle: verified
  runtime platform: software-only
  hardware attestation: not verified (development mode)
```

| Run | cMCP policy | Controller | Execution | What the audit trail proves |
|---|---|---|---|---|
| 1 — declared workflow, clear cell | authorized | accepted | completed | Which policy authorized which request, under which signed manifest |
| 2 — undeclared workflow | **denied** | not invoked | never started | The request was structurally stopped before any robot-side call existed |
| 3 — person in the safeguarded area | authorized | **rejected** (`human_detected`) | not started | Authorization happened *and* the machine layer refused — both on the record |

Run 3 is the whole post in one row. The policy engine did its job perfectly and said yes. The world changed — a person walked into the cell — and the layer that watches the world said no. If your governance design cannot represent "the software was right to allow it and the machine was right to refuse it," it will either block everything or, much worse, treat an `allow` as permission to move.

The example states this boundary explicitly, and it is the sentence I most want robotics engineers to read:

> A cMCP `allow` decision means the software request is authorized. It does not mean that a physical action is safe, accepted by the controller, or completed by a machine.

## Who owns "safe": the three-layer split

That boundary generalizes into a layered model. Each layer answers one question, produces its own evidence, and defers downward.

<svg viewBox="0 0 760 400" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Three-layer responsibility model for embodied AI governance" style="max-width:100%;height:auto;">
  <style>
    .lt { font: 600 17px -apple-system, Helvetica, Arial, sans-serif; fill: #1a1a1a; }
    .lq { font: italic 13.5px -apple-system, Helvetica, Arial, sans-serif; fill: #444; }
    .lp { font: 13px -apple-system, Helvetica, Arial, sans-serif; fill: #333; }
    .hd { font: 700 13px -apple-system, Helvetica, Arial, sans-serif; fill: #666; letter-spacing: 1px; }
    .ar { font: 600 12px -apple-system, Helvetica, Arial, sans-serif; fill: #777; }
  </style>
  <text x="20" y="28" class="hd">LAYER</text>
  <text x="545" y="28" class="hd">ITS EVIDENCE</text>

  <rect x="20" y="42" width="500" height="92" rx="10" fill="#E3F2FD" stroke="#90CAF9"/>
  <text x="40" y="76" class="lt">Mission governance — AgenTrust</text>
  <text x="40" y="100" class="lq">"Is this request allowed, and which agent made it?"</text>
  <text x="40" y="120" class="lp">Signed Agent Manifest · Cedar default-deny permits · cMCP runtime</text>
  <text x="545" y="86" class="lp">Signed audit bundle +</text>
  <text x="545" y="104" class="lp">TRACE Trust Record</text>

  <text x="255" y="158" class="ar">defers to ▼</text>

  <rect x="20" y="172" width="500" height="92" rx="10" fill="#FFF8E1" stroke="#FFE082"/>
  <text x="40" y="206" class="lt">Machine control — independent controller</text>
  <text x="40" y="230" class="lq">"Is this motion acceptable right now?"</text>
  <text x="40" y="250" class="lp">Single-use state tokens · live state recheck · speed and zone limits</text>
  <text x="545" y="216" class="lp">Accept / reject decisions</text>
  <text x="545" y="234" class="lp">on fresh, authenticated state</text>

  <text x="255" y="288" class="ar">defers to ▼</text>

  <rect x="20" y="302" width="500" height="92" rx="10" fill="#FFEBEE" stroke="#EF9A9A"/>
  <text x="40" y="336" class="lt">Functional safety — certified and hardware-enforced</text>
  <text x="40" y="360" class="lq">"How is harm physically prevented?"</text>
  <text x="40" y="380" class="lp">ISO 13849 · IEC 61508 · ISO 10218 · safety PLC, e-stop, force limiting</text>
  <text x="545" y="346" class="lp">Safety certification,</text>
  <text x="545" y="364" class="lp">outside the agent's reach</text>
</svg>

| Layer | Question it answers | Owner in the example | Failure handled |
|---|---|---|---|
| Mission governance | Is this request allowed, by which agent, under which policy? | cMCP + Agent Manifest + Cedar | Undeclared workflows, prompt/tool drift, unprovable decisions |
| Machine control | Is this motion acceptable against the *current* physical state? | Independent mock controller | Stale/replayed state, speed and zone violations, humans in the cell |
| Functional safety | How is harm physically prevented when everything above fails? | Certified systems (out of scope, on purpose) | The cases no software policy should ever be trusted with |

A governance layer written in Python evaluating Cedar policies is not a safety function and never will be: it is not certified to any Performance Level or SIL, and it sits orders of magnitude above real-time control loops. The honest claim — and the one this example is built around — is that agent governance adds a *provable decision layer above* the certified stack, and explicitly defers to it. Anyone who tells you agentic policy can replace ISO 13849 has never stood next to a moving robot.

## Promises vs. proofs: why the hardware boundary matters

So far, everything described could run on any laptop — and that is precisely the limitation Imran's summit talk targets. The controls we rely on today (policy engines, identity checks, audit logs, credential vaults) all live *inside* the agent's trust boundary. If the runtime is compromised, policies can be bypassed and logs can be forged after the fact. Software governance, on its own, makes promises.

The AgenTrust design pushes toward proofs, and the example shows the trajectory honestly:

| Control | Software-only (this example, dev mode) | TEE-backed (where the stack is going) |
|---|---|---|
| Policy decision | A promise: Cedar evaluated the policy — assuming the runtime was intact | A proof: the decision executed inside an attested environment |
| Audit trail | Hash-chained and signed — verifiable offline, forgeable only before signing | Anchored to hardware attestation: forging requires breaking the TEE, not the app |
| Agent identity | Signed manifest binds prompt, policy and tool hashes | The same binding, attested at runtime, not just at review time |
| Controller outcome | **The open gap:** the audit bundle binds request hashes and policy decisions, but does not yet populate a response hash for what the controller reported back | The contribution I want to make next: close the loop so *outcomes*, not just authorizations, are bound into the evidence |

Note the last row. The example can currently *prove what was authorized* but not yet *prove what the controller said happened* — the bundle has no response hash for the controller outcome. I put that gap in the PR description on purpose. A governance example that hides its own audit gap would be a strange way to argue for auditability.

The committed TRACE fixture is deliberately labeled `software-only`: it validates signatures, hashes and audit integrity without claiming hardware provenance, and the agent supports a `--require-hardware` flag for rehearsal on a TEE host. When the full stack is public, that flag is where "promise" turns into "proof."

Why does this matter more for embodied agents than for chatbots? Because the consequences are irreversible and the questions arrive in legal form. When a robot cell does something unexpected, the people asking — the works council, the insurer, the accident investigator — are not asking whether your policy engine *probably* worked. They are asking what you can prove. For software agents, a forged audit log is a bad day for the security team. For physical agents, an unforgeable record of *which agent, which policy, which authorization, which refusal* is the difference between an incident review and a liability lottery.

## What this means under EU law

Readers of [post #1](/2026/04/10/eu-ai-act-compliance-checklist-for-ai-agent-developers/) will recognize where each artifact lands. A sketch, not legal advice:

| Obligation | Deadline | What the example demonstrates |
|---|---|---|
| EU AI Act Art. 12 — record-keeping | Aug 2, 2026 | Hash-chained audit + signed TRACE bundle, verifiable offline by a third party |
| EU AI Act Art. 14 — human oversight | Aug 2, 2026 | Default-deny, workflow-scoped permits give oversight an actual enforcement point, not a dashboard |
| EU AI Act Art. 15 — robustness & cybersecurity | Aug 2, 2026 | Signed manifest against prompt/tool tampering; single-use state tokens against replay |
| Machinery Regulation 2023/1230 | Jan 14, 2027 | Deliberately *not* claimed: machine safety stays with the certified layer, and the example's architecture encodes that deference |

The pattern from post #1 holds: the Act's words ("logging," "oversight," "robustness") only become real when there is a specific artifact a notified body or an auditor can check. This example is my current best answer to what those artifacts look like when the agent's tools are actuators.

## Where this goes next

Three dates, in order:

**Thursday, June 18** — GenAI Gurus hosts **Marius Hobbhahn**, CEO and founder of Apollo Research, on *"When Claude Code Goes Rogue: Real-Time Monitoring"*: what thousands of real coding-agent traces reveal (dangerous commands, data exfiltration attempts, instruction drift, scope creep), and a live demo of Watcher, Apollo's real-time oversight tool. If this post is about proving what an agent is *allowed* to do before it acts, that session is about catching what agents *actually* do while they act — prevention and detection, the two halves of any control system that deserves the name. It is free and online: **[RSVP on Meetup](https://www.meetup.com/genai-gurus/events/314881167/)**.

**Tuesday, June 23** — Imran presents [Governing AI Agents at the Hardware Boundary](https://ccsummit2026.sched.com/event/2NKcF/governing-ai-agents-at-the-hardware-boundary-imran-siddique-microsoft) at the Confidential Computing Summit, where the AgenTrust stack — including this example — goes on stage.

**After that** — the response-hash gap, hardware-attested runs of this example, and the deeper question this series keeps circling: what a portable governance profile for physical agents should standardize. If you build robot cells, fleets, or the skill marketplaces that feed them — or if you are a robotics-safety engineer convinced an AI-governance person has no business near your controller — I want your objections: [the example is public](https://github.com/agentrust-io/examples/pull/16), and the comment section at the [GenAI Gurus call](https://www.meetup.com/genai-gurus/events/314881167/) works too.

## The series

- **Post 1:** [EU AI Act for AI Agent Developers: A Practical Compliance Checklist](/2026/04/10/eu-ai-act-compliance-checklist-for-ai-agent-developers/)
- **Post 2:** [How to Run Coding Agents Safely in the Enterprise](/2026/05/28/coding-agents-safely/)
- **Post 3:** Software Promises, Hardware Proofs: Governing an Industrial Robot Agent with AgenTrust *(this post)*
- **Post 4:** Code Is No Longer the Bottleneck. Verification Is *(coming soon)*
- **Post 5:** From NIM to Jetson: A NeMo Guardrails Configuration Pack for Production Inference
- **Post 6:** Open Weights, Real Obligations: Governing GPAI Models You Deploy but Didn't Train
- **Post 7:** Sovereign AI Infrastructure: Governance Patterns for On-Prem and European Cloud
- **Post 8:** The Contributor Journey: Building an Open-Source Agent Governance Layer

The full series lives at [governance.ai-mvp.com](https://governance.ai-mvp.com).

If you found this useful, the companion resource is **[Awesome EU AI Act](https://github.com/GenAI-Gurus/awesome-eu-ai-act)**, a community-maintained list of 200+ official sources, open source tools, templates, and guides for EU AI Act compliance. A GitHub star helps other developers find it.

---

*Written by [Carlos Hernandez](https://www.linkedin.com/in/carloshvp), founder of [GenAI Gurus](https://genai-gurus.com), Europe's GenAI practitioner community. I contribute to [AgenTrust](https://github.com/agentrust-io/examples) in a personal capacity. AgenTrust is in developer preview: the example pins specific cMCP and TRACE commits until the public release stack is on PyPI, and APIs may change. The scenario is fully synthetic — no robot hardware, vendor SDK, production endpoint, or proprietary industrial data — and nothing in this post is a substitute for certified functional safety or legal advice.*
