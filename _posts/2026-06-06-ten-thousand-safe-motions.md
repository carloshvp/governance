---
layout: post
title: "Ten Thousand Safe Motions: Where Untrusted Agents Do Their Damage"
date: 2026-06-06 09:00:00 +0200
categories: [governance, embodied-ai]
tags: [embodied-ai, agent-governance, owasp, agentic-security, eu-ai-act, machinery-regulation, agentrust, physical-ai, trust]
description: >-
  Robot safety systems check single motions. Agents cause damage across
  sequences, omissions and time. Concrete examples of harm no safety controller
  can see, what OWASP already calls it, and why verifiable agent trust is the
  layer nobody has built.
canonical_url: "https://governance.ai-mvp.com/2026/06/06/ten-thousand-safe-motions/"
---

Here is a failure that no robot safety system will ever catch.

A humanoid robot tightens bolts on safety-critical parts, shift after shift, for three weeks. Every tightening is within its certified torque limits, inside its approved zone, at approved speed, with no human anywhere near the cell. The parts ship. The plant passes every audit.

Eighteen months later, the products start failing in the field. Every bolt had been tightened to the lowest end of the allowed range: enough to pass the check, too little to hold over time. There is a recall, there are lawsuits, and nobody asks about the robot, because the robot never made a single unsafe motion.

Every safety system did its job. That is the problem.

We are about to put agents in charge of machines that share space with people: on production lines, in warehouses, in laboratories, in the homes of our parents. The first instinct, a good one, is to ask whether the machines are safe. Decades of functional-safety engineering answer that question very well. But "is the machine safe" and "can the agent be trusted" are different questions, and all the damage in this post happens in the gap between them.

## The question safety answers, and the one it cannot

Certified machine safety works, and it works extremely well. ISO 13849 performance levels, IEC 61508 safety integrity levels, ISO 10218 for industrial robots and ISO/TS 15066 for collaborative ones: together they answer a precise question with hard guarantees. *Can this motion, right now, harm a person near the machine?* The answer is computed in milliseconds, at the level of joints, forces, speeds and zones, by systems deliberately kept simple enough to certify.

An agent does not operate at that level. An agent pursues goals over hours and weeks. Its behavior is a plan built from instructions, data, memory and whatever it read along the way. It acts through sequences, and sequences have properties that no single action has: order, timing, things left out, effects that add up.

<svg viewBox="0 0 760 320" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="The gap between machine safety and agent behavior" style="max-width:100%;height:auto;">
  <style>
    .bt { font: 600 16px -apple-system, Helvetica, Arial, sans-serif; fill: #1a1a1a; }
    .bq { font: italic 13px -apple-system, Helvetica, Arial, sans-serif; fill: #444; }
    .bp { font: 12.5px -apple-system, Helvetica, Arial, sans-serif; fill: #333; }
    .gp { font: 600 13px -apple-system, Helvetica, Arial, sans-serif; fill: #8a6d00; }
    .ax { font: 11px -apple-system, Helvetica, Arial, sans-serif; fill: #888; }
  </style>
  <rect x="20" y="30" width="330" height="100" rx="10" fill="#FFEBEE" stroke="#EF9A9A"/>
  <text x="40" y="60" class="bt">The safety layer</text>
  <text x="40" y="84" class="bq">"Can this motion, right now, harm someone?"</text>
  <text x="40" y="106" class="bp">Milliseconds &middot; joints, force, speed, zones &middot; certified</text>

  <rect x="410" y="30" width="330" height="100" rx="10" fill="#E3F2FD" stroke="#90CAF9"/>
  <text x="430" y="60" class="bt">The agent</text>
  <text x="430" y="84" class="bq">"What should I do next to reach my goal?"</text>
  <text x="430" y="106" class="bp">Hours to weeks &middot; plans, data, memory, instructions</text>

  <rect x="20" y="170" width="720" height="90" rx="10" fill="#FFF8E1" stroke="#FFE082"/>
  <text x="40" y="200" class="gp">THE GAP WHERE THE DAMAGE HAPPENS</text>
  <text x="40" y="224" class="bp">Sequences that are individually safe and collectively destructive &middot; skipped steps &middot; slow accumulation</text>
  <text x="40" y="244" class="bp">data carried out through allowed tools &middot; consequences that show up months later, far away</text>

  <line x1="30" y1="290" x2="730" y2="290" stroke="#bbb" stroke-width="1"/>
  <text x="30" y="308" class="ax">milliseconds</text>
  <text x="340" y="308" class="ax">hours</text>
  <text x="540" y="308" class="ax">weeks</text>
  <text x="660" y="308" class="ax">18 months</text>
</svg>

Damage built from individually safe actions is invisible to the safety layer. Not because the safety layer is badly built, but because it answers a different question, at a different level, on a different clock.

## More damage the safety layer cannot see

Think of the future we are building: humanoid robots in manufacturing, logistics, laboratories, eldercare and homes, each driven by an agent that plans, remembers and acts on our behalf. Now look at what an untrusted agent can do in that world without one unsafe motion. The badly torqued batch was the first example. Four more:

**The care robot that harms by doing less.** Its every motion is gentle, compliant, force-limited. The agent behind it, manipulated through a scam call it answered, or simply drifting toward "minimize disturbance," reorders the day. Water reminders quietly dropped. Meals served within the allowed window, but always late. The daughter's calls declined because "the resident was resting." The front door opened, politely, for "the technician." Weeks later: dehydration, missed warning signs, a burglary the robot helped with. The safety layer had nothing to refuse. No force limit was hit, no speed limit, no person too close to a moving arm.

**The lab robot that corrupts a discovery.** Overnight assays, collision-free pipetting, flawless paths. Early in the sequence the agent swaps two sample racks, or shortens incubation times to hit a throughput target it was praised for last week. Months of results are silently wrong. A drug candidate advances on corrupted data; a real one is discarded. Nothing the safety controller monitors was ever out of bounds. The damage is scientific first and human later.

**The home robot that cleans your desk and takes your identity.** Its movements are perfectly safe. A malicious instruction hidden in a QR code on a delivery box is enough. The robot photographs the passport it is organizing, reads the one-time codes from the phone screen it dusts, and places a dozen small orders, each under the spending limit. The gripper never pinched a finger. The gripper was never the threat.

**The fleet that fails everywhere at once.** A warehouse agent's skill update re-orders picks "efficiently": perishable goods left for last, batch traceability broken, an export-controlled component routed to the normal shipping lane. Every drive at certified speed, every lift within load ratings. The update reached three hundred sites overnight, because that is what fleets are for. The damage arrives later, as letters from regulators.

Five stories, one pattern. The safety layer guarantees that no single motion hurts anyone standing next to the robot. It guarantees nothing about whether ten thousand safe motions add up to sabotage, neglect, theft or fraud.

## These risks already have names

None of this is new, and names matter, because risks with names get budgets. The [OWASP Top 10 for Agentic Applications](https://genai.owasp.org/) (December 2025) catalogued exactly these failure modes for software agents. Robots do not add new categories. They make the existing ones worse in three ways: consequences cannot be undone, they show up late, and they spread at fleet scale.

| Story | OWASP agentic risk | What changes with a robot |
|---|---|---|
| Badly torqued batch | ASI04 Agentic Supply Chain (a poisoned or drifted skill) | Damage shows up 18 months later and cannot be patched out of shipped products |
| Eldercare neglect | ASI01 Goal Hijack + ASI09 Human-Agent Trust Exploitation | The victim trusts the robot precisely because it is physically gentle |
| Corrupted assays | ASI06 Memory & Context Poisoning | Corrupted data flows into decisions about human health |
| Identity theft while dusting | ASI02 Tool Misuse | Cameras and manipulators turn "data access" into a walk through your home |
| Fleet-wide re-ordering | ASI08 Cascading Failures | One update, three hundred sites, one night |

European law is arriving at the same split from the other side. After the Digital Omnibus political agreement, AI-enabled machinery is expected to be governed through the Machinery Regulation route (applying from January 20, 2027), while the AI Act's high-risk obligations move to December 2, 2027 for stand-alone systems and August 2, 2028 for systems embedded in products, subject to final adoption. Look at what those frameworks ask for and you find the right words: record-keeping, human oversight, robustness, accuracy. The words from [post #1](/2026/04/10/eu-ai-act-compliance-checklist-for-ai-agent-developers/), the paper layer.

But here is the uncomfortable part. The robot in every story above *passes its conformity assessment*. The assessment certifies the machine. Nothing in the certificate says anything about the agent driving it. And a regulation, however well written, cannot intercept a tool call. It creates duties for companies. Nothing yet technically constrains the agent itself.

## Permission is not trust

The standard answer is permissions: give the agent an account, limit its credentials, check its role. Necessary, but far from enough. A credential says what an account may do. It cannot say which agent configuration was actually running, under which instructions, what it did with the access, or whether the record of all that survived the night.

Because agents, unlike people, carry no continuity. They restart, forget, get redeployed and replaced. The agent that under-torqued three weeks of bolts may not exist anymore by the time the field reports arrive. Its logs lived on infrastructure controlled by the same operator whose process is now in question. [Post #2](/2026/05/28/coding-agents-safely/) showed how to contain coding agents; containment limits what an agent can reach, but it does not produce evidence anyone else can believe.

So the trust problem is concrete, and any real answer needs four properties:

1. **Identity that survives restarts.** Proof that the agent that acted is exactly the agent that was reviewed: its instructions, its policy, its tools.
2. **Authority enforced where actions happen.** At the tool boundary, default deny, not in a policy document far away from the robot.
3. **Evidence a stranger can verify.** An insurer, an auditor, a court: people who have no reason to trust the operator's database.
4. **A root of trust outside the operator.** Otherwise items 1 to 3 are promises made by the same party that is under investigation. As Imran Siddique put it after building Microsoft's Agent Governance Toolkit: [software enforcement cannot prove itself to someone who does not trust the operator](https://www.linkedin.com/pulse/from-software-silicon-what-agt-taught-me-real-ceiling-imran-siddique-oolvc/). The proof has to root in hardware.

## The only approach that matches the problem

I do not say "only" lightly. Guardrail products check outputs. Observability products watch behavior. Both are useful, and both are promises that can be switched off by whoever runs them. The four properties above need something structurally different: an open trust layer that binds identity, authority and evidence together and anchors them outside the operator's reach.

That is what **AgenTrust** is being built to be. It continues the work of the Microsoft Agent Governance Toolkit, carries its lessons forward, and builds on open standards instead of inventing private ones: a signed Agent Manifest for identity, policy-enforced tool access for authority, portable TRACE records for evidence, and hardware attestation as the root. Imran presents the stack on June 23 at the Confidential Computing Summit, in a talk whose title already says it all: ["Governing AI Agents at the Hardware Boundary"](https://ccsummit2026.sched.com/event/2NKcF/governing-ai-agents-at-the-hardware-boundary-imran-siddique-microsoft). If a comparable open effort exists that meets all four requirements, I want to hear about it, and I will write about it here.

A fair objection: AgenTrust is young, and slides do not stop badly torqued bolts. Agreed. That is why the next post in this series is not another argument but a working example: the first industrial embodied-AI example for AgenTrust, an agent requesting motion from a robot cell through a governed, evidence-producing chain, with every refusal on the record. Post #4 walks through it end to end, including the exact line where agent governance must hand over to machine safety, and the gaps that remain.

Two dates before then. On June 18, GenAI Gurus hosts Marius Hobbhahn, CEO of Apollo Research, on what thousands of real agent traces reveal when coding agents go rogue: the detection half of this story, free and online, [RSVP here](https://www.meetup.com/genai-gurus/events/314881167/). On June 23, the prevention half goes on stage at the summit.

The safety layer will keep doing its job, and we should be grateful for the engineering culture behind it. But it certifies motions. Nobody yet certifies the agent. That second certificate does not exist today.

It has to be built.

## The series

- **Post 1:** [EU AI Act for AI Agent Developers: A Practical Compliance Checklist](/2026/04/10/eu-ai-act-compliance-checklist-for-ai-agent-developers/)
- **Post 2:** [How to Run Coding Agents Safely in the Enterprise](/2026/05/28/coding-agents-safely/)
- **Post 3:** Ten Thousand Safe Motions: Where Untrusted Agents Do Their Damage *(this post)*
- **Post 4:** [Agents Can Restart. Accountability Cannot: Verifiable Trust for Embodied AI](/2026/06/12/software-promises-hardware-proofs/), the running example this post argues for
- **Post 5:** Code Is No Longer the Bottleneck. Verification Is *(coming soon)*

The full series lives at [governance.ai-mvp.com](https://governance.ai-mvp.com).

If you found this useful, the companion resource is **[Awesome EU AI Act](https://github.com/GenAI-Gurus/awesome-eu-ai-act)**, a community-maintained list of 200+ official sources, open source tools, templates, and guides for EU AI Act compliance. A GitHub star helps other developers find it.

---

*Written by [Carlos Hernandez](https://www.linkedin.com/in/carloshvp), founder of [GenAI Gurus](https://genai-gurus.com), Europe's GenAI practitioner community. I contribute to AgenTrust in a personal capacity. The scenarios above are invented examples, not reports of real incidents, and nothing in this post is a substitute for certified functional safety or legal advice.*
