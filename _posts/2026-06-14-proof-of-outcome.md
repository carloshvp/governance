---
layout: post
title: "Proof of Outcome: Did the Robot Actually Do It?"
date: 2026-06-14 09:00:00 +0200
categories: [runtime, embodied-ai]
tags: [embodied-ai, robotics, agentrust, proof-of-outcome, tamper-evident, sensor-attestation, flight-recorder, eu-ai-act]
description: >-
  An authorization log proves a robot was allowed to move, not that it did. Closing the response-hash gap with sensor-witnessed, tamper-evident outcome evidence.
canonical_url: "https://governance.ai-mvp.com/2026/06/14/proof-of-outcome/"
---

*Your audit proves the robot was allowed to move. It does not prove the robot moved — or that the world changed the way the agent claims. This is the gap that decides whether your evidence is worth anything to an insurer.*

> Series: The Engineering of AI Governance — Post 6
> Continues from [The Cloud Can Prove It. The Robot Can't.](https://governance.ai-mvp.com/2026/06/11/the-cloud-can-prove-it-the-robot-cant/)
> Interactive: this post has three widgets you can play with. They're the point — read with your hands.
> Status: this extends AgenTrust's existing primitives (manifest, cMCP, TRACE); the specific mechanisms proposed here are design proposals for discussion, not shipped features.

---

In Post 4 I admitted a gap and gave it a name: the **response-hash gap**. The audit bundle proves what was *authorized* — a `cMCP` `allow`, a signed manifest, a policy decision. It does not prove what *happened*. A robot that was authorized to place an object and then dropped it produces the same authorization log as one that placed it perfectly.

For a humanoid working among people — on a line next to a human, say — that gap is the whole game. After an incident, nobody is interested in what the policy engine permitted. They want to know what the arm did, when, and whether it stopped when the person stepped in. An `allow` is a statement of intent. **Intent is not what hurts people, and it is not what an insurer pays out on.**

So Post 6 is about closing the gap: turning a record of *decisions* into a record of *effects*.

## The gap, made concrete

Start by feeling it. Below, a three-step pick-and-place. The authorization log always says the same thing — approve, approve, approve, task complete. Change the *physical* scenario and toggle whether outcome evidence is being recorded, and watch what your audit can and cannot see.

<iframe src="/widgets/a6-authorization-vs-outcome.html" title="Interactive widget" loading="lazy" style="width:100%;max-width:720px;border:1px solid #e6e6e2;border-radius:12px;background:#fff;display:block" height="500"></iframe>

*Try this: pick "gripper slip" or "human entered zone," then flip outcome attestation off. The verdict still reads "complete" — because with only the authorization log, the failure is invisible. Turn attestation on and the divergence appears.*

The lesson is uncomfortable: an audit trail that logs only authorizations will certify a dropped object, a missed stop, and a clean run identically. That is not a logging bug. It is a category error — you are recording the wrong layer.

## Sensors as witnesses

The fix is to make the robot's own instrumentation testify. A motion already generates a stream of physical truth: end-effector pose, joint torques, gripper force, the vision frame, and — critically — the independent controller's **state token**, the same single-use token from Post 4 that says *the decision is now, not thirty seconds ago*.

Bind those readings into the evidence record, signed and time-stamped, and the audit stops describing permissions and starts describing reality. Scrub through a motion and watch the witnesses move together — including the moment a person enters the safeguarded zone and the controller forces a stop.

<iframe src="/widgets/a6-sensor-witness-scrubber.html" title="Interactive widget" loading="lazy" style="width:100%;max-width:720px;border:1px solid #e6e6e2;border-radius:12px;background:#fff;display:block" height="560"></iframe>

*Try this: drag the scrubber past the point where the person enters. Force collapses to zero, the end-effector freezes, and the controller's state token flips to STOP — all on the same timeline, all signed.*

There is an honest problem here, and Post 5 already named it: on the robot, the perception model runs in an integrated GPU with **no confidential computing**. The same compromised runtime that could forge a log could, in principle, forge a sensor frame. Signing readings close to the source, cross-checking redundant sensors, and binding every frame to the single-use state token raises the cost of lying enormously — but it does not make lying impossible. Outcome proof is a steep hill for an attacker, not a wall. Say that out loud; the credibility is in the caveat.

## The flight recorder

Witness data is only worth as much as its resistance to quiet editing after the fact. So the outcome record is structured the way an aircraft's is: a **tamper-evident, hash-chained ledger** where each record seals the one before it, and the final root is anchored with an outside party (a transparency log, an insurer's endpoint, the cloud step's attested record from Post 5). Alter any record and the chain breaks from there down; the anchored root stops matching. The witness is no longer the accused.

Edit the ledger below and watch it refuse to be rewritten.

<iframe src="/widgets/a6-flight-recorder-hashchain.html" title="Interactive widget" loading="lazy" style="width:100%;max-width:720px;border:1px solid #e6e6e2;border-radius:12px;background:#fff;display:block" height="680"></iframe>

*Try this: change any record — soften "object not released" to "object released," for instance. Every hash from that record down turns red, and the recomputed root no longer matches the anchored one. You cannot edit history without leaving the edit visible.*

This is the artifact an accident investigator, an insurer, or a court actually needs: not a list of what the robot was allowed to do, but a sealed, reconstructable account of what it did — one that an outside party can verify without trusting the operator.

## When the network drops

Robots keep working when connectivity doesn't. So outcome evidence cannot depend on a live link to the anchor. The pattern: buffer signed records locally in order, keep the hash chain intact offline, and anchor the accumulated root when the link returns. Ordering survives the outage because each record still seals its predecessor; only the *publication* of the root is deferred, not the integrity. An audit that falls apart the moment a robot leaves Wi-Fi is not an audit.

## Where this lands

For the EU AI Act, this is what makes **Article 12 / Article 19** logging meaningful rather than ceremonial. Those provisions want automatically generated records of the system's *operation* — and a list of granted permissions is not that. Outcome evidence is the difference between a log that satisfies the letter and one that survives the question an investigator actually asks.

It is also the foundation for the next two posts. Fleet revocation (Post 7) needs to know whether a skill *behaved*, not just whether it was *authorized*. The skill marketplace (Post 8) needs outcome evidence to catch a third-party skill that passed review and then misbehaved in the field. Proof of outcome is the layer everything downstream stands on.

The response-hash gap is closed — not perfectly, because the edge hardware won't allow perfect, but honestly, and to a standard an outside party can check. Next: making it hold across ten thousand robots that share what they learn.
