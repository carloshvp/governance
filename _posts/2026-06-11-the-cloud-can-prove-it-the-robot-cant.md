---
layout: post
title: "The Cloud Can Prove It. The Robot Can't."
date: 2026-06-11 09:00:00 +0200
categories: [runtime, embodied-ai]
tags: [embodied-ai, robotics, agentrust, confidential-computing, tee, nvidia-thor, attestation, edge-ai, eu-ai-act]
description: >-
  On a humanoid, the action happens where hardware trust is weakest. NVIDIA Jetson Thor has no GPU confidential computing, so agent trust must be engineered across an asymmetric cloud-to-edge boundary.
canonical_url: "https://governance.ai-mvp.com/2026/06/11/the-cloud-can-prove-it-the-robot-cant/"
---

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true, theme: 'neutral', securityLevel: 'loose' });
</script>

*Where to anchor agent trust on a humanoid robot when the on-board GPU (NVIDIA Thor) has no confidential computing — and the reasoning that drove the motion lives in a datacenter that does.*

> Series: The Engineering of AI Governance — Post 5
> Continues from [Software Promises, Hardware Proofs](https://governance.ai-mvp.com/2026/06/08/software-promises-hardware-proofs/) and [Ten Thousand Safe Motions](https://governance.ai-mvp.com/2026/06/06/ten-thousand-safe-motions/)
> Strategic companion (robotics.ai-mvp.com): [Trust Becomes the Next Moat in the Humanoid Race](https://robotics.ai-mvp.com/notes/010-the-trust-stack/) — the same problem framed for operators and investors.

---

In the last post I argued that software enforcement cannot prove itself to a party that does not trust the operator. Audit logs, policy decisions, compliance records — all of them are *promises*, and a compromised runtime can forge every one. The move from promises to proofs runs through hardware: anchor the signing keys and the enforcement outside the operator's reach, in a Trusted Execution Environment, and the witness is no longer the accused.

I closed with an honest admission — the **response-hash gap**: the audit bundle proves what was *authorized*, not what *happened*. This post is about a second gap, and it is worse, because it does not come from unfinished code. It comes from the silicon.

Here is the problem in one sentence:

> **On a humanoid robot, the place where the action physically happens is the place where you have the weakest hardware-rooted trust.**

The reasoning that decided to pick up the object runs in a datacenter GPU that can produce a hardware-signed proof of exactly what it did. The motion itself runs on a chip bolted into the robot's torso that cannot. Accountability, as I keep saying, has to be engineered — and this is the seam where it is hardest to engineer.

## The architecture nobody gets to opt out of

Strip a deployed humanoid down to its trust-relevant parts and you get two compute domains, not one.

<pre class="mermaid">
flowchart LR
    subgraph Cloud["Datacenter — confidential computing available"]
        LLM["Reasoning / planning<br/>(large VLA or LLM)<br/>H200 · B200 · GB200"]
    end
    subgraph Edge["Robot — NVIDIA Jetson Thor"]
        POL["On-board policy / fast VLA<br/>(reactive control)"]
        CTRL["Real-time controller<br/>(certified safety stack)"]
        ACT["Actuators"]
    end
    LLM -- "plan / goal" --> POL
    POL -- "command" --> CTRL
    CTRL --> ACT
    POL -. "evidence" .-> Ledger[(Trust ledger / TRACE)]
    LLM -. "attested evidence" .-> Ledger
</pre>

This is not an architectural choice you can refactor away. It is forced by physics and economics. The slow, heavy reasoning — the part that benefits from a 70B-plus model and a second of deliberation — commoditizes in the cloud, where you can rent the best hardware. The fast control loop — the part that must close in milliseconds and keep running when the network drops — stays on the robot. (Readers of the robotics side of this work will recognize this as the [*slow-reasoning-commoditizes, fast-control-stays-defensible* split](https://robotics.ai-mvp.com/notes/010-the-trust-stack/) — and why trust becomes the next axis of competition.) Trust has to cross that boundary, and the two sides of the boundary are not equals.

## The hardware reality

I want this section to be airtight, because the whole argument rests on it.

In the datacenter, **GPU Confidential Computing** is real and shipping. An NVIDIA H200 (Hopper) can run in CC mode: the model and its activations sit in **encrypted VRAM**, and the GPU's security processor emits a signed attestation report that a remote party can verify through NVIDIA's attestation service or Intel Trust Authority — in one composite workflow alongside the CPU TEE (Intel TDX or AMD SEV-SNP). Datacenter **Blackwell** (B200/GB200) goes further, adding **TEE-I/O (TDISP)** to secure the PCIe path between the CPU TEE and the GPU, removing most of the bounce-buffer overhead Hopper paid. You can hand an auditor cryptographic proof that *this exact model ran, unmodified and confidential, on genuine CC-mode hardware.*

Now the robot. Jetson AGX Thor uses a **Blackwell GPU** — the same architecture family. So you might assume the same proofs are available on board. They are not.

| Platform | GPU Confidential Computing (encrypted VRAM, GPU-signed attestation, TEE-I/O) | CPU / SoC TEE | What you can prove |
|---|---|---|---|
| **H200 (Hopper)** | ✅ CC mode; GPU-security-processor-signed attestation; remote-attestable (NVIDIA NRAS / Intel Trust Authority) | Pairs with Intel TDX / AMD SEV-SNP CVM | *This exact model ran unmodified and confidential on genuine CC-mode hardware* |
| **B200 / GB200 (datacenter Blackwell)** | ✅ Adds **TEE-I/O (TDISP)** — secures the CPU↔GPU link, removes most of Hopper's CC overhead | TDX / SEV-SNP | Same, at far lower performance cost |
| **Jetson AGX Thor (edge Blackwell)** | ❌ **Not exposed.** CC + TDISP are dGPU-only; no tooling in the Jetson Linux stack | ARM **TrustZone / OP-TEE**, secure boot (on-die BootROM + 16 PKC fuse keys), fTPM, disk + system-memory encryption, rollback protection | Boot/platform integrity and secrets held in the secure world — **not** the GPU inference |

The decisive fact: NVIDIA states plainly, in its own developer forum, that Confidential Computing and TDISP are *"for dGPU only"* and not supported on Jetson Thor. As of Jetson Linux r38 (mid-2026) there is no tooling in the Jetson software stack to enable them.

And this is not arbitrary market segmentation. Thor's GPU is **integrated** on the SoC, sharing **unified LPDDR memory** with the CPU. The entire datacenter CC model assumes a *discrete* GPU with its own encrypted HBM, attested across a *secured PCIe link* — which is exactly what TDISP protects. On an integrated SoC there is **no PCIe boundary to secure and no separate VRAM to encrypt.** The confidential-computing model does not structurally map onto the part. (Note the trap for the careless: Thor *does* encrypt system memory — but that is LPDDR-level encryption, not the encrypted-VRAM-plus-GPU-attestation primitive that defines datacenter CC. They sound alike. They are not the same protection.)

So the correct statement is not "Thor has a weaker GPU TEE." It is sharper:

> **Thor and the datacenter offer two different *classes* of trust. The cloud gives you GPU-confidential-compute-class proofs. The robot gives you TrustZone-class proofs — and only those.**

## The asymmetric trust chain

This turns the clean trust chain from Post 4 into a lopsided one. Walk the two halves.

**Cloud reasoning step.** Wrap it in a CC TEE. You get a hardware-attested proof binding the model identity, the inputs, and the outputs to genuine CC-mode silicon. This is the *strong* anchor — and it is exactly the kind of evidence the Agent Manifest was designed to seal: prompt, policy, tool schemas, model identity, delegation chain. In the cloud, the manifest's claims can be *proven*, not just asserted.

**Edge actuation step.** You cannot do any of that. The strongest anchor available on Thor is the secure world: OP-TEE holds a signing key that never leaves TrustZone, secure boot measures the boot chain into the fuses and the fTPM, and the secure world can emit **signed, ordered evidence** — "policy P evaluated; command C issued at time T by agent A; state token S consumed." That signed evidence feeds the hash-chained audit and the single-use state tokens we already built. It is real, hardware-rooted accountability.

But notice precisely what it covers and what it does not. TrustZone can prove the *platform booted as expected* and that *a key under hardware control signed this record.* It **cannot** prove that the vision-language-action model in the integrated GPU ran unmodified, or that its activations stayed confidential. There is no enclave around that GPU. The secure world is a few megabytes for small trusted apps; it cannot hold a multi-gigabyte policy model. So the on-robot inference — the thing closest to the moving hand — is *measured and signed at its boundaries*, not *confidential-compute-attested*.

That is the second gap, sitting right next to the response-hash gap:

- **Response-hash gap** (Post 4): the bundle proves authorizations, not outcomes.
- **Edge-inference gap** (this post): on the actuation device, the inference itself is signed at the edges but not attested as confidential or unmodified, because the hardware cannot.

I would rather name both gaps out loud than ship a diagram that implies a proof we cannot produce.

## What you can still do at the edge — and where I want to be honest

This is not a counsel of despair. TrustZone-class trust is enough to build something genuinely useful, as long as you do not oversell it:

1. **Anchor agent identity in the secure world.** The robot's signing key lives in OP-TEE, bound to secure boot. A forged runtime cannot impersonate the agent without breaking hardware.
2. **Measure the model at load.** Before the policy/VLA artifact is loaded into the integrated GPU, hash it — the weights, and ideally the runtime and config alongside them — and have a trusted app inside OP-TEE sign *"model with digest H was loaded at T on device D."* The signing key never leaves the secure world and is bound to secure boot, so a compromised normal-world runtime cannot forge the record; the fTPM can even hold the digest in a measurement register. Think of it as **measured boot applied to the model** rather than to the bootloader. It converts "trust the operator's word about which model is running" into a hardware-signed claim, and because the manifest already seals the *approved* model identity, any silent swap to a tampered or unapproved model shows up as a digest that no longer matches. Its limit is exactly the edge-inference gap: it proves *which artifact was loaded*, not that the GPU then executed it unmodified or kept its activations confidential — a load-time measurement, not continuous enforcement.
3. **Emit signed, ordered evidence from TrustZone**, hash-chained, so skipped steps leave holes no log entry can paper over.
4. **Bind the edge record to the cloud's strong attestation.** The plan that arrived from the datacenter carries a verifiable CC attestation; the edge evidence references it by hash. The chain is asymmetric, but it is *continuous*: you can follow one motion from a hardware-attested cloud decision to a hardware-signed edge actuation, and see exactly where the assurance steps down a class.
5. **Make the step-down explicit in TRACE.** Serialise an asymmetric chain naively and a consumer may read every link as equally strong — implying the actuation was GPU-CC-attested when it was not. So each claim in the record should carry an explicit **assurance class** — the kind of hardware trust backing *that link* — not a single flattened verdict. Concretely, a per-link tag such as `gpu-cc-attested | cpu-tee-attested | trustzone-signed | software-only`, plus what it protects (confidentiality / integrity / identity) and which authority attested it. The cloud reasoning link reads *GPU CC, remote-attested, model and I/O confidential*; the edge actuation link reads *TrustZone-signed, secure-boot-rooted, model measured at load, inference not confidential*. This keeps both classes of proof in one record without faking their equivalence, it lets an automated verifier enforce policy ("reject any chain whose actuation link is software-only"), and it maps onto the standards TRACE already composes — RATS/EAT claims already carry verifier-assessed trust. Honest evidence states its own limits, in a field a machine can read.

What I deliberately will **not** claim: that any of this makes the on-robot inference confidential, or that it proves the integrated GPU ran the measured model unmodified. It does not, and no amount of TrustZone signing changes that until the silicon does.

## Threat model, briefly

- **Defended:** forged agent identity at the edge; silent model substitution (detectable via the load measurement); tampered or reordered evidence; replayed stale state; a compromised software runtime forging its own audit; an operator asserting an unverifiable version of events to an investigator.
- **Not defended:** an attacker with physical control of the GPU memory bus extracting model weights or activations on the robot; a runtime that faithfully reports a measured model while the GPU silently miscomputes; nation-state-grade hardware attacks on the SoC. These require either confidential computing on the part (not available) or physical-security controls outside the trust layer.

Stating the second list is not weakness. It is the difference between an evidence record an investigator can rely on and a marketing claim that collapses under the first hard question.

## Where this lands for the EU AI Act

A humanoid carrying out tasks among people is, in regulatory terms, an AI system acting as a safety component of a regulated machine — high-risk under **Article 6(1)** via the **Annex I** product-safety route (the Machinery Regulation), not the Annex III use-case list. That routing matters for timing. Under the Digital Omnibus deferral agreed on 7 May 2026 (pending formal publication), Annex III high-risk obligations move to **2 December 2027** — but Annex I product-embedded systems, which is where robots sit, move to **2 August 2028**. You have runway; you do not have an exemption.

Two of the high-risk requirements bite directly on everything above. **Article 12 (record-keeping)** requires the system to automatically log events across its lifetime, and **Article 19** requires providers to keep those logs — the hash-chained, hardware-signed evidence in this post is exactly that kind of record, and the asymmetric chain lets you produce it across the cloud/edge boundary instead of only for the cloud half. **Article 15 (accuracy, robustness and cybersecurity)** wants assurance that the deployed system is the approved one and has not been silently altered; the cloud-side CC attestation and the edge-side load measurement are the technical controls that turn that requirement from a policy promise into a hardware-backed proof — fully for the cloud step, and an honestly-qualified one (carried in the TRACE assurance class) for the edge step. On the deployer side, **Article 26** adds the duty to keep those automatically generated logs — the same evidence, handed to whoever operates the fleet.

The regulation does not care that Thor lacks confidential computing. It cares whether you can show, independently of your own word, what your system did. The architecture in this post is how you show it *given* the hardware you actually have — and, just as importantly, how you disclose the one place where you can only show part of it.

## What I want from you

The example is public, in [`agentrust-io/examples`](https://github.com/agentrust-io/examples). The cloud-attested path and the TrustZone evidence path are where I am building next, and this is the part of the system where I most want to be wrong in interesting ways.

If you build humanoid platforms, edge perception stacks, or the confidential-computing infrastructure on either side of this boundary, I want your objections specifically on three things: whether the load-time model measurement on Thor is worth more than I think or less; whether anyone is getting *any* GPU-side attestation out of an integrated Tegra part that I have missed; and whether the cloud-to-edge evidence binding survives contact with a real fleet.

We are going to argue exactly this, in person. I have invited several robotics engineers and **Imran**, who leads AgenTrust, to a **GenAI Gurus** session on trust for embodied AI — the natural sequel to his Confidential Computing Summit talk. If you want the hardware-rooted-trust-for-robots conversation to be had by people who actually ship the hardware, this is the room.

Software promises. Hardware provides proofs. And on a robot, the most important hardware is the one that can prove the least — so we had better be precise about what we are proving, and where.

---

*Next in the series: closing the response-hash gap with hardware-attested runs, and a portable assurance-level profile for TRACE so a single trust record can carry both classes of proof without pretending they are the same.*
