---
layout: post
title: "Trust at Fleet Scale: the Skill That Spreads"
date: 2026-06-17 09:00:00 +0200
categories: [runtime, embodied-ai]
tags: [embodied-ai, robotics, agentrust, fleet, identity, revocation, trust-propagation, blast-radius, eu-ai-act]
description: >-
  Trust that works for one robot is a demo. Per-unit identity, fast revocation, and bounding the blast radius of a poisoned skill that spreads through a fleet's networked learning.
canonical_url: "https://governance.ai-mvp.com/2026/06/17/trust-at-fleet-scale/"
---

*Trust that works for one robot is a demo. A business is a fleet — and a fleet that shares what it learns can share a compromise just as fast. Identity, revocation, and the worm nobody designs for.*

> Series: The Engineering of AI Governance — Post 7
> Continues from [Proof of Outcome](https://governance.ai-mvp.com/2026/06/14/proof-of-outcome/)
> Interactive: three widgets. The blast-radius simulator is the one to play with.
> Status: this extends AgenTrust's existing primitives (manifest, cMCP, TRACE); the specific mechanisms proposed here are design proposals for discussion, not shipped features.

---

Everything so far has been about one robot: its manifest, its hardware, its outcome evidence. But nobody sells one robot. A robotics company ships *fleets* — and two trust problems appear only at scale, both invisible in a single-unit demo.

The first is mundane and unavoidable: **per-unit identity**, from the factory line to the scrapyard. The second is the one almost nobody designs for until it bites: **trust propagation**. When robots share what they learn — the way a skill-sharing fleet is designed to work, with "networked learning across robot types" — a poisoned or unsafe skill can spread through the fleet like a worm. The same mechanism that makes the fleet smart makes it fragile.

## A birth certificate on the line

A robot's identity cannot be assigned after the fact by a server that simply trusts whatever the robot claims. It has to be *born* on the manufacturing line: a hardware root of trust injected into the secure element and fuses, secure boot measured, and only then an identity the fleet will accept. Onboarding (FIDO Device Onboard / BRSKI-style) lets a unit *prove what it is* before it joins — not assert it.

Walk the sequence:

<iframe src="/widgets/a7-robot-onboarding-stepper.html" title="Interactive widget" loading="lazy" style="width:100%;max-width:720px;border:1px solid #e6e6e2;border-radius:12px;background:#fff;display:block" height="460"></iframe>

*Step through it: notice that each stage hands the next a signed artifact. The fleet identity at the end isn't a database row — it's the end of a chain that starts in silicon on the factory floor.*

Get this wrong and everything downstream is theatre: revocation, attestation, and skill provenance all assume the robot is who it says it is. That assumption is only as good as the moment of birth.

## Revocation that actually works

A fleet is only as trustworthy as its weakest unit. Robots get stolen, resold, jailbroken, or simply retired — and when one does, you need to revoke its identity *fleet-wide, fast, and provably*. "We sent an email to disable it" is not revocation. The metric that matters is **revocation latency**: how long between "this unit is compromised" and "no system in the fleet will accept its credentials," plus a transparency-log entry that proves it happened.

Compromise a few units below and revoke them:

<iframe src="/widgets/a7-fleet-revocation.html" title="Interactive widget" loading="lazy" style="width:100%;max-width:720px;border:1px solid #e6e6e2;border-radius:12px;background:#fff;display:block" height="640"></iframe>

*Try this: click several robots to mark them compromised, then hit revoke. Watch the revocation latency and the transparency-log entries. The point isn't that revocation is possible — it's that it's fast, fleet-wide, and leaves a proof.*

Decommissioning is the same discipline run forward: a retired robot must leave no usable keys behind, or your fleet's trust boundary quietly includes a machine in a scrapyard.

## The skill that spreads

Here is the problem that is genuinely new, and genuinely under-discussed. Networked learning is a superpower: one robot learns to handle a part better, and the whole fleet improves overnight. But run that mechanism without trust controls and you have built a **propagation channel** — and a poisoned, unsafe, or simply buggy skill rides it exactly as fast as a good one.

The defense is not to ban sharing. It's to make propagation *earn* its reach: every cross-robot update carries a **signed propagation manifest**, releases are staged to a bounded cohort, and a **blast-radius** bound ensures that if a skill turns out to misbehave — caught by the outcome evidence from Post 6 — it can only have reached a contained slice of the fleet, never all of it.

Watch the difference. Drop a poisoned skill into the fleet with containment off, then on:

<iframe src="/widgets/a7-trust-propagation-blast-radius.html" title="Interactive widget" loading="lazy" style="width:100%;max-width:720px;border:1px solid #e6e6e2;border-radius:12px;background:#fff;display:block" height="680"></iframe>

*Try this: hit play with containment off and the skill floods all 64 robots. Turn containment on and reset — the same poisoned skill is bounded to its staging cohort and halts. Same superpower, survivable failure mode.*

This is where the trilogy interlocks. Containment needs Post 6's outcome evidence to *notice* a skill is misbehaving, and Post 8's marketplace controls to decide what a skill was even allowed to do. No single layer is enough; the fleet survives because they back each other up.

## Fleet attestation

The question a regulator or an enterprise buyer will eventually ask is simple and brutal: *right now, is every robot in your fleet running approved software?* Not "was it approved at deployment" — *right now*. Answering it requires continuous fleet attestation: each unit periodically proves its measured state, and the fleet view is a verifiable rollup, not a spreadsheet maintained by hope. The honest version surfaces drift — the handful of units that have wandered from the approved baseline — rather than reporting a green dashboard that means nothing.

## The honest tension

Networked learning and attestation pull in opposite directions. The more autonomously robots adapt and share, the harder it is to say "every unit is running approved software," because the software is changing underneath you. There is no clean resolution — only a boundary you draw deliberately: what may propagate *autonomously* within a blast-radius bound, versus what requires a signed, reviewed release before it can reach the fleet. Pretending the tension doesn't exist is how you end up with a fleet that is both smart and unaccountable. State the boundary; don't hide it.

## Where this lands

For the EU AI Act, fleet identity and revocation are how Article 15's "the deployed system is the approved one" survives contact with ten thousand units that update in the field. Revocation latency and fleet attestation are the operational evidence behind it. And the transparency log of revocations and propagations is, in practice, the Article 12 / 19 record-keeping obligation applied to the fleet rather than the single robot.

One robot you can trust by hand. A fleet you have to trust by design — and the design has to assume that the channel which makes it intelligent is the same one an attacker would choose. Next, Post 8: the skills and models you didn't write, running on a machine that can hurt someone.
