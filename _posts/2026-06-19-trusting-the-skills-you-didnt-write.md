---
layout: post
title: "Trusting the Skills You Didn't Write"
date: 2026-06-19 09:00:00 +0200
categories: [runtime, embodied-ai]
tags: [embodied-ai, robotics, agentrust, marketplace, provenance, slsa, cedar, default-deny, eu-ai-act]
description: >-
  A robot's brain is a foundation model you didn't train and skills from a marketplace anyone can publish to. Provenance, default-deny capability scoping, and a fleet-wide kill switch.
canonical_url: "https://governance.ai-mvp.com/2026/06/19/trusting-the-skills-you-didnt-write/"
---

*The first two posts assumed you wrote the software. You didn't. The brain is a foundation model you didn't train; the skills come from a marketplace anyone can publish to. The ecosystem is your moat and your largest attack surface — at the same time.*

> Series: The Engineering of AI Governance — Post 8
> Continues from [Proof of Outcome](https://governance.ai-mvp.com/2026/06/14/proof-of-outcome/) and [Trust at Fleet Scale](https://governance.ai-mvp.com/2026/06/17/trust-at-fleet-scale/). Final post of the trust trilogy.
> Interactive: three widgets. The capability explorer is the one to push on.
> Status: this extends AgenTrust's existing primitives (manifest, cMCP, TRACE); the specific mechanisms proposed here are design proposals for discussion, not shipped features.

---

Posts 6 and 7 quietly assumed something convenient: that you wrote the software running on your robot. You almost certainly didn't. A full-stack humanoid builder runs a foundation model it didn't train — its robot is powered by **NVIDIA's GR00T** — and on top of that, the skills come from a marketplace where third parties publish, share, and monetize robot capabilities. The marketplace *is* the business model. It is also the largest attack surface in the entire system.

So the question this post answers is the uncomfortable one underneath every robot app store: **how do you let code you didn't write run on a machine that can hurt someone — without trusting whoever wrote it?**

## The supply chain of a robot's mind

A robot's behaviour is not one artifact. It's a stack of them, each from a different author: a foundation model, a domain fine-tune, first-party skills, and third-party skills from the marketplace. Every link in that stack is a trust decision — and today most of them are taken on faith.

The discipline is the same one software supply chains learned the hard way: **provenance**. A signed manifest per artifact, **SLSA** provenance on the model and its fine-tunes, a model bill-of-materials so you know what went into the brain. An artifact without provenance isn't "probably fine" — it's unaccountable, and on a safety-critical robot that means blocked.

Inspect the stack and see where trust actually comes from:

<iframe src="/widgets/a8-supply-chain.html" title="Interactive widget" loading="lazy" style="width:100%;max-width:720px;border:1px solid #e6e6e2;border-radius:12px;background:#fff;display:block" height="560"></iframe>

*Try this: click each link to see its signer and provenance level, then toggle "require signed provenance" off. The unsigned promo skill, which was blocked, is suddenly admitted — that toggle is the difference between a supply chain and a hope.*

Notice the foundation model sits at the root. You didn't train GR00T, so you cannot audit its weights line by line — but you *can* require it to arrive signed, with provenance, from an identifiable party, and pin exactly which version you run. Trust you can't derive from inspection, you derive from provenance and identity instead.

## Default-deny at the skill boundary

Provenance tells you *where a skill came from*. It says nothing about *what the skill may do once it runs*. That is a separate control, and it is the most important one on this page: every skill is scoped to exactly what its signed manifest declares, and **anything undeclared fails structurally** — the Cedar default-deny posture from Post 4, applied at the marketplace boundary.

The consequence is the property you want to be able to say out loud to a regulator: a weather-display skill *cannot* command an actuator. Not "shouldn't," not "is unlikely to" — *cannot*, because moving the arm was never in its manifest, and you cannot grant a capability that the signed manifest doesn't contain.

Push on it:

<iframe src="/widgets/a8-capability-scoping.html" title="Interactive widget" loading="lazy" style="width:100%;max-width:720px;border:1px solid #e6e6e2;border-radius:12px;background:#fff;display:block" height="680"></iframe>

*Try this: select the weather skill, then attempt "move arm" or "actuate gripper." Both are denied — not by a rule someone remembered to write, but structurally, because they aren't in the skill's signed manifest. Switch to the arm-handoff skill and the same calls go through. Capability follows declaration.*

This is what turns a marketplace from a liability into something operable. The blast radius of a malicious skill is bounded *before it runs* by what it was allowed to declare — and actuator capabilities are exactly the declarations a marketplace reviews hardest.

## Running untrusted code on a machine that moves

Scoping bounds what a skill can *request*. You still have to bound what it can *do* if it misbehaves within its scope. That's the three-layer containment from Post 4 doing real work: the third-party skill lives in the **mission layer**, sandboxed away from the certified functional-safety stack, under motion-envelope and rate limits it cannot exceed. A skill that was authorized to move the arm still cannot drive it outside the safe envelope, because the layer that enforces the envelope is not the layer the skill runs in. Untrusted code on a robot is survivable only when the thing keeping people safe is architecturally below the thing you don't trust.

## Review, attestation, and the kill switch

Before a skill ships, the marketplace has to check it — signature valid, provenance present, declared capabilities within policy, sandbox manifest attached, static scan clean — and that review has to become a **verifiable claim** a buyer can inspect later, not an internal rubber stamp. An attestation record in TRACE says *this skill passed this review at this version*, signed.

And because review is never perfect, the marketplace needs a kill switch: when a published skill turns out to be unsafe, you revoke it **fleet-wide**, fast, and provably — the exact revocation machinery from Post 7, now pointed at a marketplace artifact instead of a compromised robot.

<iframe src="/widgets/a8-skill-review-revocation.html" title="Interactive widget" loading="lazy" style="width:100%;max-width:720px;border:1px solid #e6e6e2;border-radius:12px;background:#fff;display:block" height="640"></iframe>

*Try this: run the review pipeline, publish the skill, then report misbehaviour. The skill is pulled across the fleet and its attestation revoked — a published, monetized capability can be un-shipped in one motion.*

## The honest gap

Here is what static review and capability scoping cannot do: catch a skill that behaves perfectly through review and then misbehaves in the field at the worst possible moment. Provenance proves origin; scoping bounds capability; neither reads intent. An adversary who stays inside its declared scope until the one instant it matters passes every check on this page.

That is precisely why this is the *third* post and not the first. The marketplace controls are necessary and insufficient on their own — they only hold because **Post 6's outcome evidence notices** a skill doing something its authorization didn't predict, and **Post 7's fleet containment bounds** how far that skill could have spread before it was caught and revoked. No single layer is sufficient. The trilogy is one argument: trust an embodied agent the way you'd trust a stranger you've given the keys to — verify where it came from, bound what it can touch, watch what it actually does, and be able to take the keys back instantly.

## Where this lands

For the EU AI Act, a skill marketplace is a supply chain, and supply chains have named responsibilities. The provenance, signed manifests, and attestation records here are how a provider demonstrates that the system in the field is the approved one (Article 15) and keeps the records to prove it (Articles 12 / 19); the capability scoping and sandboxing are the technical measures behind the safety obligations for AI as a safety component of machinery. The marketplace operator and the robot manufacturer will argue for years about who owns which obligation — but neither can discharge theirs without exactly this evidence.

That closes the trilogy. Post 6 made the robot prove what it did. Post 7 made trust survive a fleet that shares what it learns. Post 8 made the marketplace safe enough to be a business. None of it makes a robot trustworthy on its own — together, they make a robot *accountable*, which is the only kind of trust that survives contact with an incident.
