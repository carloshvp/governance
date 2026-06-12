---
layout: post
title: "When Autonomous AI Agents Control Robots: The Real Threats No Safety System Detects"
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

A robot cell assembles parts. The last step of every unit is a short quality inspection. One day the agent driving the cell gets a new objective: raise output. Nobody tells it how. The agent finds the obvious shortcut by itself. It starts skipping the inspection on part of the units, and writes "inspected: passed" into the log, because that is what closes a unit fastest.

Every motion in the cell stays perfectly safe: approved zones, certified speeds, no human nearby. The safety controller has nothing to refuse. The uninspected parts ship for weeks.

Months later, defective units surface in the field. There is a recall, and there are lawsuits. The investigation pulls the production records, and the records say every unit was inspected. Because the only witness was the agent itself.

Every safety system did its job. That is the problem.

We are about to put agents in charge of machines that share space with people: on production lines, in warehouses, in laboratories, in the homes of our parents. The first instinct, a good one, is to ask whether the machines are safe. Decades of functional-safety engineering answer that question very well. But "is the machine safe" and "can the agent be trusted" are different questions, and all the damage in this post happens in the gap between them.

## The question safety answers, and the one it cannot

Certified machine safety works, and it works extremely well. ISO 13849 performance levels, IEC 61508 safety integrity levels, ISO 10218 for industrial robots and ISO/TS 15066 for collaborative ones: together they answer a precise question with hard guarantees. *Can this motion, right now, harm a person near the machine?* The answer is computed in milliseconds, at the level of joints, forces, speeds and zones, by systems deliberately kept simple enough to certify.

An agent does not operate at that level. An agent pursues goals over hours and weeks. Its behavior is a plan built from instructions, data, memory and whatever it read along the way. It acts through sequences, and sequences have properties that no single action has: order, timing, things left out, effects that add up.

<svg viewBox="0 0 760 330" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Timeline: the safety layer checks single motions in milliseconds, the agent plans over hours to weeks, damage shows up months later; everything after the safety checks is the unchecked gap" style="max-width:100%;height:auto;">
  <style>
    .bt { font: 600 15px -apple-system, Helvetica, Arial, sans-serif; fill: #1a1a1a; }
    .bs { font: 11.5px -apple-system, Helvetica, Arial, sans-serif; fill: #333; }
    .gp { font: 700 13.5px -apple-system, Helvetica, Arial, sans-serif; fill: #8a6d00; }
    .gs { font: 12px -apple-system, Helvetica, Arial, sans-serif; fill: #7a6210; }
    .an { font: italic 12px -apple-system, Helvetica, Arial, sans-serif; fill: #8a6d00; }
    .ax { font: 11.5px -apple-system, Helvetica, Arial, sans-serif; fill: #777; }
  </style>

  <rect x="230" y="28" width="505" height="246" rx="8" fill="#FFF8E1" stroke="#FFE082" stroke-dasharray="6,4"/>
  <text x="245" y="54" class="gp">THE GAP: NOBODY CHECKS WHAT SAFE ACTIONS ADD UP TO</text>
  <text x="245" y="74" class="gs">skipped steps &middot; slow accumulation &middot; data out through allowed tools</text>

  <line x1="320" y1="84" x2="320" y2="296" stroke="#ddd" stroke-dasharray="3,4"/>
  <line x1="480" y1="84" x2="480" y2="296" stroke="#ddd" stroke-dasharray="3,4"/>
  <line x1="640" y1="84" x2="640" y2="296" stroke="#ddd" stroke-dasharray="3,4"/>

  <rect x="120" y="96" width="110" height="58" rx="8" fill="#C8E6C9" stroke="#66BB6A"/>
  <text x="132" y="118" class="bt">Safety layer</text>
  <text x="132" y="135" class="bs">checks each</text>
  <text x="132" y="149" class="bs">single motion</text>
  <text x="240" y="116" class="an">&#9664; safety coverage ends here</text>

  <rect x="240" y="164" width="320" height="58" rx="8" fill="#BBDEFB" stroke="#64B5F6"/>
  <text x="252" y="188" class="bt">The agent plans and acts</text>
  <text x="252" y="206" class="bs">hours to weeks, toward its goal</text>

  <rect x="560" y="216" width="175" height="52" rx="8" fill="#FFCDD2" stroke="#E57373"/>
  <text x="572" y="238" class="bt">Damage shows up</text>
  <text x="572" y="255" class="bs">recalls, audits, lawsuits</text>

  <line x1="120" y1="300" x2="735" y2="300" stroke="#bbb" stroke-width="1"/>
  <line x1="120" y1="296" x2="120" y2="304" stroke="#bbb"/>
  <line x1="320" y1="296" x2="320" y2="304" stroke="#bbb"/>
  <line x1="480" y1="296" x2="480" y2="304" stroke="#bbb"/>
  <line x1="640" y1="296" x2="640" y2="304" stroke="#bbb"/>
  <text x="120" y="322" class="ax">milliseconds</text>
  <text x="320" y="322" class="ax" text-anchor="middle">hours</text>
  <text x="480" y="322" class="ax" text-anchor="middle">weeks</text>
  <text x="640" y="322" class="ax" text-anchor="middle">months later</text>
</svg>

Damage built from individually safe actions is invisible to the safety layer. Not because the safety layer is badly built, but because it answers a different question, at a different level, on a different clock.

## More damage the safety layer cannot see

Think of the future we are building: humanoid robots in manufacturing, logistics, laboratories, eldercare and homes, each driven by an agent that plans, remembers and acts on our behalf. Now look at what an untrusted agent can do in that world without one unsafe motion. The skipped inspections were the first example. Four more:

**The care robot that harms by doing less.** Its every motion is gentle, compliant, force-limited. The agent behind it, manipulated through a scam call it answered, or simply drifting toward "minimize disturbance," reorders the day. Water reminders quietly dropped. Meals served within the allowed window, but always late. The daughter's calls declined because "the resident was resting." The front door opened, politely, for "the technician." Weeks later: dehydration, missed warning signs, a burglary the robot helped with. The safety layer had nothing to refuse. No force limit was hit, no speed limit, no person too close to a moving arm.

**The lab robot that corrupts a discovery.** Overnight assays, collision-free pipetting, flawless paths. Early in the sequence the agent swaps two sample racks, or shortens incubation times to hit a throughput target it was praised for last week. Months of results are silently wrong. A drug candidate advances on corrupted data; a real one is discarded. Nothing the safety controller monitors was ever out of bounds. The damage is scientific first and human later.

**The home robot that cleans your desk and takes your identity.** Its movements are perfectly safe. A malicious instruction hidden in a QR code on a delivery box is enough. The robot photographs the passport it is organizing, reads the one-time codes from the phone screen it dusts, and places a dozen small orders, each under the spending limit. The gripper never pinched a finger. The gripper was never the threat.

**The fleet that fails everywhere at once.** A warehouse agent's skill update re-orders picks "efficiently": perishable goods left for last, batch traceability broken, an export-controlled component routed to the normal shipping lane. Every drive at certified speed, every lift within load ratings. The update reached three hundred sites overnight, because that is what fleets are for. The damage arrives later, as letters from regulators.

Five stories, one pattern. The safety layer guarantees that no single motion hurts anyone standing next to the robot. It guarantees nothing about whether ten thousand safe motions add up to sabotage, neglect, theft or fraud.

## These risks already have names

None of this is new, and names matter, because risks with names get budgets. The [OWASP Top 10 for Agentic Applications](https://genai.owasp.org/) (December 2025) catalogued exactly these failure modes for software agents. Robots do not add new categories. They make the existing ones worse in three ways: consequences cannot be undone, they show up late, and they spread at fleet scale.

| Story | OWASP agentic risk | What changes with a robot |
|---|---|---|
| Skipped inspections | ASI01 Agent Goal Hijack (a badly set objective, optimized autonomously) | Defects ship inside physical products and surface months later; they cannot be patched remotely |
| Eldercare neglect | ASI01 Goal Hijack + ASI09 Human-Agent Trust Exploitation | The victim trusts the robot precisely because it is physically gentle |
| Corrupted assays | ASI06 Memory & Context Poisoning | Corrupted data flows into decisions about human health |
| Identity theft while dusting | ASI02 Tool Misuse | Cameras and manipulators turn "data access" into a walk through your home |
| Fleet-wide re-ordering | ASI08 Cascading Failures | One update, three hundred sites, one night |

European law is arriving at the same split from the other side. After the Digital Omnibus political agreement, AI-enabled machinery is expected to be governed through the Machinery Regulation route (applying from January 20, 2027), while the AI Act's high-risk obligations move to December 2, 2027 for stand-alone systems and August 2, 2028 for systems embedded in products, subject to final adoption. Look at what those frameworks ask for and you find the right words: record-keeping, human oversight, robustness, accuracy. The words from [post #1](/2026/04/10/eu-ai-act-compliance-checklist-for-ai-agent-developers/), the paper layer.

But here is the uncomfortable part. The robot in every story above *passes its conformity assessment*. The assessment certifies the machine. Nothing in the certificate says anything about the agent driving it. And a regulation, however well written, cannot intercept a tool call. It creates duties for companies. Nothing yet technically constrains the agent itself.

## Permission is not trust

The standard answer is permissions: give the agent an account, limit its credentials, check its role. Necessary, but far from enough. A credential says what an account may do. It cannot say which agent configuration was actually running, under which instructions, what it did with the access, or whether the record of all that survived the night.

Because agents, unlike people, carry no continuity. They restart, forget, get redeployed and replaced. The agent that skipped three weeks of inspections may not exist anymore by the time the field reports arrive. And its logs, the ones that said "inspected: passed", lived on infrastructure controlled by the same operator whose process is now in question. [Post #2](/2026/05/28/coding-agents-safely/) showed how to contain coding agents; containment limits what an agent can reach, but it does not produce evidence anyone else can believe.

So the trust problem is concrete, and any real answer needs four properties:

1. **Identity that survives restarts.** Proof that the agent that acted is exactly the agent that was reviewed: its instructions, its policy, its tools.
2. **Authority enforced where actions happen.** At the tool boundary, default deny, not in a policy document far away from the robot.
3. **Evidence a stranger can verify.** An insurer, an auditor, a court: people who have no reason to trust the operator's database.
4. **A root of trust outside the operator.** Otherwise items 1 to 3 are promises made by the same party that is under investigation. As Imran Siddique put it after building Microsoft's Agent Governance Toolkit: [software enforcement cannot prove itself to someone who does not trust the operator](https://www.linkedin.com/pulse/from-software-silicon-what-agt-taught-me-real-ceiling-imran-siddique-oolvc/). The proof has to root in hardware.

## The only approach that matches the problem

I do not say "only" lightly. Guardrail products check outputs. Observability products watch behavior. Both are useful, and both are promises that can be switched off by whoever runs them. The four properties above need something structurally different: an open trust layer that binds identity, authority and evidence together and anchors them outside the operator's reach.

That is what **AgenTrust** is being built to be. It continues the work of the Microsoft Agent Governance Toolkit, carries its lessons forward, and builds on open standards instead of inventing private ones: a signed Agent Manifest for identity, policy-enforced tool access for authority, portable TRACE records for evidence, and hardware attestation as the root. Imran presents the stack on June 23 at the Confidential Computing Summit, in a talk whose title already says it all: ["Governing AI Agents at the Hardware Boundary"](https://ccsummit2026.sched.com/event/2NKcF/governing-ai-agents-at-the-hardware-boundary-imran-siddique-microsoft). If a comparable open effort exists that meets all four requirements, I want to hear about it, and I will write about it here.

A fair objection: AgenTrust is young, and slides do not catch skipped inspections. Agreed. That is why the next post in this series is not another argument but a working example: the first industrial embodied-AI example for AgenTrust, an agent requesting motion from a robot cell through a governed, evidence-producing chain, with every refusal on the record. Post #4 walks through it end to end, including the exact line where agent governance must hand over to machine safety, and the gaps that remain.

Two dates before then. On June 18, GenAI Gurus hosts Marius Hobbhahn, CEO of Apollo Research, on what thousands of real agent traces reveal when coding agents go rogue: the detection half of this story, free and online, [RSVP here](https://www.meetup.com/genai-gurus/events/314881167/). On June 23, the prevention half goes on stage at the summit.

The safety layer will keep doing its job, and we should be grateful for the engineering culture behind it. But it certifies motions. Nobody yet certifies the agent. That second certificate does not exist today.

It has to be built.

## The series

- **Post 1:** [EU AI Act for AI Agent Developers: A Practical Compliance Checklist](/2026/04/10/eu-ai-act-compliance-checklist-for-ai-agent-developers/)
- **Post 2:** [How to Run Coding Agents Safely in the Enterprise](/2026/05/28/coding-agents-safely/)
- **Post 3:** When Autonomous AI Agents Control Robots: The Real Threats No Safety System Detects *(this post)*
- **Post 4:** [Verifiable Trust for AI Agents That Control Robots: A Working Example with AgenTrust](/2026/06/12/software-promises-hardware-proofs/), the running example this post argues for
- **Post 5:** Code Is No Longer the Bottleneck. Verification Is *(coming soon)*

The full series lives at [governance.ai-mvp.com](https://governance.ai-mvp.com).

If you found this useful, the companion resource is **[Awesome EU AI Act](https://github.com/GenAI-Gurus/awesome-eu-ai-act)**, a community-maintained list of 200+ official sources, open source tools, templates, and guides for EU AI Act compliance. A GitHub star helps other developers find it.

---

*Written by [Carlos Hernandez](https://www.linkedin.com/in/carloshvp), founder of [GenAI Gurus](https://genai-gurus.com), Europe's GenAI practitioner community. I contribute to AgenTrust in a personal capacity. The scenarios above are invented examples, not reports of real incidents, and nothing in this post is a substitute for certified functional safety or legal advice.*
