---
layout: post
title: "How to Run Coding Agents Safely in the Enterprise"
date: 2026-05-28 09:00:00 +0000
categories: [runtime, coding-agents]
tags: [coding-agents, codex, claude-code, copilot, cursor, sandboxing, prompt-injection, mcp, agent-governance, eu-ai-act]
description: >-
  OpenAI just published how it runs Codex internally. Here is what every enterprise
  should copy, what to add on top, and a reference architecture for adopting Codex,
  Claude Code, Copilot, and Cursor without burning your secrets, your IP, or your
  audit story.
canonical_url: "https://governance.ai-mvp.com/2026/05/28/coding-agents-safely/"
---

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true, theme: 'neutral', securityLevel: 'loose' });
</script>

# How to Run Coding Agents Safely in the Enterprise

*OpenAI just published how it runs Codex internally. Here is what every enterprise should copy, what to add on top, and a reference architecture for adopting Codex, Claude Code, Copilot, and Cursor without burning your secrets, your IP, or your audit story.*

## Why this post exists

A couple of weeks ago I wrote [a short LinkedIn note](https://www.linkedin.com/posts/carloshvp_aigovernance-agentgovernance-aisafety-activity-7459639476135407616-nfQY) about OpenAI's then-new paper "Running Codex safely at OpenAI." The point of the note was small: five major vendors have converged on essentially the same coding-agent safety architecture, which means we now have a public reference design and no good excuse for not deploying it.

The DMs that followed surprised me. Various known and first-met security leads, platform engineers, and AI product managers reached out to discuss further details. This post is the answer.

It also builds directly on two GenAI Gurus community sessions that shaped my thinking: an April session with **[Maxim Vovshin](https://www.linkedin.com/in/maxim-vovshin-a5a0a5164/)** on personal agents and the safety patterns he uses to keep them bounded, and a May session with **[Imran Siddique](https://www.linkedin.com/in/imransiddique1986/)** (Principal Engineering Leader at Microsoft) on the Microsoft Agent Governance Toolkit (AGT) and its policy-as-code approach. Both are linked at the end. The architecture below would not exist without those conversations.

## The post that changed the conversation

On May 8, 2026, OpenAI published "Running Codex safely at OpenAI." It is the most useful piece of public documentation we have on how a frontier lab actually runs an autonomous coding agent against its own production codebase. It is also a rare artifact: a vendor admitting that "control is only half the job" and that "once agents are deployed, security teams need visibility into what these agents are doing and why."

If you lead AI strategy, security, or platform engineering, read it twice. It is a blueprint, and most enterprises are nowhere close to it.

This post does three things:

1. Distills what OpenAI actually does for Codex into a small set of patterns.
2. Compares those patterns with how Anthropic ships Claude Code, how GitHub ships Copilot Enterprise, and how Cursor has evolved through a hard year of CVEs.
3. Hands you a reference architecture and a 12-control checklist you can take to a security review on Monday.

The pattern below is what I would build today if I were standing up a coding-agent platform inside a regulated European enterprise.

## Why coding agents break traditional AppSec

Traditional AppSec assumes that source code is mostly written by humans and that automation runs deterministic pipelines. Coding agents break both assumptions. They:

- Read your repo, your dotfiles, and any document a developer drops into the context window.
- Decide which shell commands to run, then run them.
- Call MCP servers, GitHub APIs, package managers, and the open internet.
- Generate code that gets committed, often under a developer's identity rather than the agent's.

The threat model is closer to "an enthusiastic but credulous junior contractor with `sudo`" than "an IDE plugin." That mental model, not legal disclaimers, is what should drive your controls.

A 2026 arXiv survey of agentic coding assistants documented 30+ CVEs across major tools, including remote code execution via MCP prompt injection (CurXecute), persistent team compromise through poisoned MCP configurations (MCPoison), and shell-builtin sandbox escapes (NomShub / CVE-2026-22708) that the authors describe as "deterministic, 100% reliable." NIST has called prompt injection "generative AI's greatest security flaw," OWASP ranks it #1 in the LLM Top 10, and its agent-native form, Agent Goal Hijack, tops the 2026 [OWASP Top 10 for Agentic Applications](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) as ASI01. The risk is not theoretical. GitGuardian's State of Secrets Sprawl report finds that 6.4% of repositories where Copilot is active leaked at least one secret in their sample of about 20,000 repositories, 40% higher than the 4.6% baseline across all public repositories. Whatever you think of the headline number, the direction is clear.

Maxim Vovshin put it bluntly during a [GenAI Gurus session on personal agents](https://www.youtube.com/live/TA04giWhM-g) in April 2026: "even if you use Claude Code you can still be prompt injected. It's just probably a bit safer. There is no silver bullet." That posture, more than any single control, is the right starting point. And on the relative effectiveness of controls, Imran Siddique, Principal Engineering Leader at Microsoft, shared a benchmark from the Microsoft Agent Governance Toolkit (AGT) work during a [follow-up GenAI Gurus session in May 2026](https://youtu.be/uDQBqp9Om5s): under prompt-based governance, agents violated stated policy 27.7% of the time on identical tasks; under policy-based governance enforced as code, the same agents stayed within bounds. That gap, in his words "the difference between asking someone to follow the rules and making it physically impossible to break them," is the entire argument for the architecture in this post.

## What OpenAI actually does (and why it works)

The Codex post organizes its approach around four layers. I will keep OpenAI's framing because it is unusually clean.

**1. Bounded execution.** Codex runs inside an OS-level sandbox. The sandbox modes are `read-only`, `workspace-write` (default), and `danger-full-access`. On macOS the sandbox is enforced by Seatbelt via `sandbox-exec`. On Linux it uses bubblewrap plus seccomp by default, with a managed proxy that exposes only allowed domains via a TCP-to-Unix-socket-to-TCP bridge inside an unshared network namespace. On Windows it uses a native restricted-token sandbox. In Codex Cloud, each session runs in an isolated OpenAI-managed container, with a setup phase that has network access for dependency install, then an agent phase that runs offline by default. Critically: "secrets configured for cloud environments are available only during setup and are removed before the agent phase starts."

**2. Approval policy.** Sandbox mode controls what is technically possible; approval policy controls what requires human review. Modes are `untrusted`, `on-request` (default), and `never`. OpenAI couples this with command rules so that "common benign commands that engineers use in day-to-day development are allowed without approval outside of the sandbox and specific dangerous commands can be blocked or require approval." A rule like `prefix_rule(pattern = ["gh", "pr", ["view", "list"]], decision = "allow")` captures the pattern.

**3. Network policy.** Quote: "We do not run Codex with open-ended outbound access. Our managed network policy allows expected destinations, blocks destinations we do not want Codex reaching, and requires approval for unfamiliar domains." This is not the same as your corporate proxy. It is an agent-aware allow/deny/approve list applied at the bubblewrap egress.

**4. Admin-enforced configuration.** "Requirements" are admin-controlled settings that users cannot override, deployed via cloud-managed requirements, macOS managed preferences, and local requirements files. Authentication is pinned: "CLI and MCP OAuth credentials are stored in the secure OS keyring, login is forced through ChatGPT, and access is pinned to our ChatGPT enterprise workspace." Codex activity then flows into the ChatGPT Compliance Logs Platform automatically.

**5. Agent-native telemetry.** This is the layer most enterprises miss. Codex emits OpenTelemetry events covering "the original request, tool activity, approval decisions, tool results, and any relevant network policy decision or block." OpenAI feeds this into "an AI-powered security triage agent" that "surfaces its analysis to our security team for review to distinguish between expected agent behavior, benign mistakes, and activity that truly warrants escalation." Same telemetry is used operationally to track adoption, MCP server usage, and sandbox blocks.

The reason this works is that no single layer is required to be perfect. Sandbox failures get caught by approvals; approval-fatigue mistakes get caught by network policy; network policy blind spots get caught by telemetry; telemetry firehose gets triaged by an AI agent. It is defense in depth, with the explicit assumption that prompt injection will sometimes succeed.

## How the four major coding agents compare

| Capability | OpenAI Codex | Anthropic Claude Code | GitHub Copilot Enterprise | Cursor |
|---|---|---|---|---|
| OS-level sandbox | Seatbelt / bubblewrap+seccomp / Windows native | Seatbelt / bubblewrap+socat (Bash tool only) | None client-side | Seatbelt / Landlock+seccomp |
| Default mode | `workspace-write` + `on-request` | read-only, asks before edits | suggestion-only in IDE; agent mode adds runtime | sandbox + auto-run unless disabled |
| Network policy | Allow/deny/approve list, agent-aware | Bash sandbox `allowedDomains` + WebFetch rules | Pre-inference proxy, vulnerability filter | OS firewall recommended; not built in |
| Admin enforcement | Requirements (cloud + MDM + file) | Managed settings (admin console + MDM + file + drop-in dir) | Org/enterprise policies in github.com | MDM, Enterprise plan |
| Audit logs | OpenTelemetry to SIEM + Compliance Logs Platform | Transcripts (configurable retention), admin console | 180-day audit log; client prompts NOT included | Enterprise plan adds audit logs |
| MCP governance | Allowed servers in requirements | `enableAllProjectMcpServers` flag, allow rules | Admin allowlist for IDE-side MCP | Documented CVEs (MCPoison, CurXecute) |
| Secret isolation | OS keyring; cloud secrets removed pre-agent | git proxy on web; config-level deny rules | Content exclusions (not for CLI/agent mode) | `.cursorignore` (overlay-mounted) |
| IP indemnification | Per OpenAI enterprise terms | Per Anthropic enterprise terms | $500K cap; requires public-code filter | Enterprise terms; check |
| Notable hard lessons | Internal triage agent reveals adoption gaps | "Sandboxing safely reduces permission prompts by 84%" (Anthropic engineering blog) | CamoLeak: prompt injection extracted private code | 7 CVEs in 2025; sandbox redesign removed user allowlists |

The convergence is striking. Every major vendor now ships some flavor of: OS-level sandbox + permission rules + admin-managed settings + audit log streaming + MCP governance. The differences are in defaults, scope, and which surfaces are actually covered. Copilot's audit-log gap on client prompts and on CLI/agent-mode content exclusions is meaningful. Cursor's removal of user allowlists is meaningful. Codex's agent-native telemetry layer is, today, the most complete.

## The architectural principle: governance as a layer, not a rewrite

Two ideas from the May 2026 Microsoft AGT session anchor everything that follows. First, in Imran Siddique's framing: "Governance is a layer, not a rewrite. You don't have to change your agents. You add governance around them." Whatever framework your team is on (LangChain, CrewAI, Semantic Kernel, OpenAI Agents SDK, Microsoft Agent Framework), the goal is to wrap it, not replace it. Second, the kernel-space metaphor: "the agent runs in user space, governance runs in kernel space." Every action passes through the governance layer; the agent cannot bypass it; policy evaluation happens before execution, not as a filter on output. AGT itself targets sub-5 ms policy evaluation per call, which is the right order of magnitude for an agent making 100+ tool invocations per minute.

The [Microsoft Agent Governance Toolkit](https://github.com/microsoft/agent-governance-toolkit) is the cleanest open-source reference implementation of this layer that exists today. It is MIT-licensed, framework-agnostic, and structured around four components that map directly to the boundaries this post argues for:

- **Agent OS:** the policy engine. Sub-millisecond YAML, OPA Rego, or Cedar policies intercept every tool call and agent action before execution. This is the kernel-space layer.
- **AgentMesh:** zero-trust identity. Every agent gets a Decentralized Identifier (DID) with an Ed25519 key pair, and every action it takes is cryptographically signed and attributable. This is what makes the "identity boundary" below enforceable in practice.
- **Agent SRE:** reliability and SLOs. SLO-based circuit breakers, accuracy thresholds, and degradation triggers. This is the operational layer that turns governance from a compliance artifact into something a platform team can actually run.
- **Agent Compliance:** attestation CLI. Generates the signed conformity reports you need for audit, mapped article-by-article to the EU AI Act (Articles 9, 11, 12, 13, 14, 15, 50) and to ISO/IEC 42001.

The repo's own positioning is worth reading literally: AGT covers **10 out of 10 of the OWASP Agentic Top 10**, which is the most comprehensive single answer to that threat list shipping today. For the rest of this post, when I say "policy engine" or "governance layer," AGT is what I have in mind as the concrete tool, though every primitive below works with Open Policy Agent, Cedar, or a hand-rolled equivalent.

The practical consequence: your security team writes policy in YAML or Rego, your agent team writes agent code, and the two repositories are reviewed independently. Same pattern as Open Policy Agent for Kubernetes. The novelty is applying it to non-deterministic systems.

## A reference architecture for enterprise coding agents

Here is the architecture I would deploy. It works for Codex, Claude Code, Copilot, or Cursor with minor adapter changes.

<pre class="mermaid">
flowchart TB
    subgraph EP["1 · Developer endpoint, MDM-enforced, OS keyring"]
        direction TB
        AGENT["IDE / CLI agent\nCodex, Claude Code, Copilot, Cursor"]
        SANDBOX["OS-level sandbox\nSeatbelt / bubblewrap+seccomp / restricted token"]
        APPROVAL["Approval policy\nrule files, pre-tool hooks"]
        EGRESS["Egress proxy + DNS\nallow / deny / approve, domain-aware"]
        AGENT --> SANDBOX
        AGENT --> APPROVAL
        AGENT --> EGRESS
    end
    subgraph CP["2 · Enterprise control plane"]
        direction TB
        MDM["MDM: Jamf / Intune\nmanaged settings on every endpoint"]
        IDP["Identity: SSO / SCIM\npinned enterprise workspace"]
        VAULT["Secret broker: Vault\nshort-lived, scoped tokens"]
        MCP["MCP gateway\nsigned allowlist of servers"]
        REVIEW["Code review gates\nrequired reviewers, signed commits"]
    end
    subgraph TT["3 · Telemetry and triage"]
        direction TB
        OTEL["OpenTelemetry collector\nprompt, tool calls, approvals, network events"]
        SIEM["SIEM\nSplunk / Sentinel"]
        TRIAGE["AI triage agent\nseparates noise from incidents"]
        OTEL --> SIEM
        SIEM --> TRIAGE
    end
    EP -->|"governed by"| CP
    EP -->|"emits telemetry"| TT
    CP -->|"policy + identity events"| TT
    style EP fill:#cce5ff,stroke:#0056b3,color:#000
    style CP fill:#d4edda,stroke:#28a745,color:#000
    style TT fill:#fff3cd,stroke:#ffc107,color:#000
</pre>

Three boundaries matter most:

- **The sandbox boundary** keeps a compromised agent on the endpoint.
- **The egress boundary** keeps an injected agent from phoning home.
- **The identity boundary** keeps agent activity attributable, auditable, and revocable.

If any one boundary fails open, the others contain the blast radius. This is the most important property to engineer for, and the easiest to lose during "we just want it to work" pilots.

## Twelve technical controls, ranked by ROI

You can ship these in order, and you should not ship the next one until the previous one is enforced (not just documented).

1. **Block free tiers on managed devices.** No coding agent on a personal account in a corporate network. SSO + SCIM + MDM + DLP. The shadow-AI ceiling here is real: SAP's 2025 data and the Microsoft and LinkedIn Work Trend Index both put unauthorized AI-tool usage at 78% of knowledge workers.

2. **MDM-deploy managed settings** for every coding agent in use. For Codex, that is `requirements.toml`. For Claude Code, `managed-settings.json` plus the OS-level policy paths. For Cursor, the Enterprise plan's MDM policies. For Copilot, organization policies in github.com.

3. **Default to read-only or workspace-write, never `bypassPermissions` / `danger-full-access` / "YOLO."** When you need full access, run inside a disposable container or microVM.

4. **Egress allowlist at the agent level.** Domain allow/deny/approve, not just an HTTP proxy. Codex's bubblewrap + UDS bridge is the reference design.

> **The Jensen Huang rule.** NVIDIA's CEO has argued repeatedly that an agent should not be allowed three capabilities at the same time: access to sensitive information, code execution, and external communication. The combination is what turns a prompt-injection success into a data-exfiltration incident. In practice this means: separate the agent that reads your inbox from the agent that runs shell commands from the agent that calls external APIs. Compose them through a governance layer, not through shared credentials. For most enterprise coding-agent setups today, this is the single architectural decision with the highest leverage.

5. **Secret broker, never `.env`. And separate identities for the agent.** Short-lived, scoped credentials issued by Vault, AWS Secrets Manager, or an equivalent. Agents authenticate to a broker; the broker decides whether to mint a token. Codex Cloud handles cloud secrets exactly this way: available during setup, removed before agent execution. A second, complementary pattern from Maxim Vovshin's GenAI Gurus walkthrough is worth stealing wholesale. Rather than giving an agent access to your personal Gmail, give the agent its own Gmail account, set up Gmail's auto-forward to relay relevant messages to it, and have a daily cron purge messages older than 48 hours. "Worst case is if something happens, I get a leakage of two days worth of emails," Maxim explained. "If I had given the agent read access to my Gmail account, this attacker can take all of my emails and just use them all." The pattern generalizes: agents get their own scoped identities, the blast radius of a compromise is bounded by what that identity can see today, and you keep the door open to revoke the entire identity rather than rotate dozens of credentials.

6. **`.cursorignore` / Read-deny / content-exclusion** for `~/.ssh`, `~/.aws`, `~/.docker`, `.npmrc`, `.git`, `node_modules` of any installed-from-untrusted package. Treat the dotfile zone as actively hostile.

7. **Pre-tool hooks** to reject patterns the LLM should never touch (writes to `/etc`, `curl ... | sh`, package installs from non-allowed registries, push to protected branches). Claude Code's `PreToolUse` and Cursor's hook system both support this. For Codex, use rule files.

8. **MCP gateway.** Centralize MCP server registration and signing. Block unsigned servers. Prefer the Microsoft Agent Governance Toolkit pattern: AGT's Agent OS evaluates every tool invocation against a policy set before the agent sees the result, so a malicious or compromised MCP server cannot smuggle an action past the governance layer. Enforce policy on the agent *action*, not just the agent prompt. Prompt-level filtering is a defense-in-depth control, not a primary one.

9. **Required code review for AI-authored PRs**, with attribution preserved (Claude Code's `Co-Authored-By` trailer, Copilot's commit attribution). Codeowners enforced on protected branches. Tests required to merge.

10. **CI/CD provenance.** Sign commits and artifacts (Sigstore/cosign). SBOM on every build. Dependency review with hard-fail on suspicious package age, ownership transfer, or postinstall scripts.

11. **Agent-native telemetry to SIEM.** OpenTelemetry export from each agent. Capture prompt, tool calls, approval decisions, tool results, network policy events. Without this, your audit log answers "what" but not "why." This is also where deterministic and non-deterministic controls compose cleanly: keep policy enforcement deterministic and inline (sub-5 ms, no LLM in the hot path), and run a separate, asynchronous semantic analysis over the logs to catch what static rules miss. As Imran framed it in the AGT session: "you also need a non-deterministic system to actually look at the logs and keep making decisions, that hey, out of all the last 100 operations performed by this agent which got approved, do you see any problem?" That second loop is what catches multi-turn social engineering, proxy discrimination patterns (an HR agent filtering by zip code to evade an age-discrimination rule), and slow-burn capability drift.

12. **AI-assisted triage with a kill switch.** OpenAI runs an AI triage agent over Codex logs. You can do the same with a small classification model, a rules engine, and on-call escalation. The point is to keep the firehose reviewable. Pair it with an actual kill switch (Article 14 of the EU AI Act calls for one explicitly): the ability to pause or terminate any agent identity in seconds, propagated through the same identity layer that issued its credentials.

## Mapping the controls to the OWASP Agentic Top 10

The [OWASP Top 10 for Agentic Applications (2026)](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) is the right risk vocabulary for this post, because a coding agent is an agent first and an LLM app second. The older OWASP LLM Top 10 still applies to the model layer, but the threats that actually bite a coding-agent platform (goal hijack, tool misuse, privilege abuse, sandbox escape) are agent-native, and the 2026 agentic list names them directly.

Here is each agentic risk, how it shows up for a coding agent, and which of the twelve controls above contain it. (Control numbers refer to the list in the previous section.)

| OWASP Agentic risk (2026) | How it shows up for a coding agent | Controls that contain it |
|---|---|---|
| **ASI01: Agent Goal Hijack** | Prompt injection in a repo file, issue, dependency, or MCP tool output redirects the agent's task | 4 egress allowlist, 7 pre-tool hooks, 11 telemetry, 12 triage + kill switch |
| **ASI02: Tool Misuse and Exploitation** | Agent runs a destructive shell command or chains tools in unintended ways | 2 managed settings, 3 conservative sandbox mode, 7 pre-tool hooks, 8 MCP gateway |
| **ASI03: Identity and Privilege Abuse** | Agent inherits a developer's broad token and acts beyond its intended scope | 1 block free tiers, 5 secret broker + scoped identities, 9 preserved attribution |
| **ASI04: Agentic Supply Chain** | Poisoned MCP server, trojanized package, or tampered update channel (MCPoison) | 8 signed MCP allowlist, 10 CI/CD provenance + SBOM + dependency review |
| **ASI05: Unexpected Code Execution** | Agent-generated or agent-invoked code escapes the intended boundary (NomShub) | 3 sandbox mode, 4 egress boundary, 6 content exclusion of the dotfile zone |
| **ASI06: Memory and Context Poisoning** | Hostile content in context or RAG store biases later reasoning and actions | 6 content exclusion, 7 pre-tool hooks, 11 telemetry for drift detection |
| **ASI07: Insecure Inter-Agent Communication** | Sub-agent or A2A messages spoofed or intercepted (weak auth between agents) | 5 scoped per-agent identities, 8 MCP gateway with server signing |
| **ASI08: Cascading Failures** | One compromised agent or tool propagates failure across the workflow | The three boundaries (blast-radius containment), 11 telemetry, 12 kill switch |
| **ASI09: Human-Agent Trust Exploitation** | Approval fatigue and authority bias get an unsafe action waved through | 9 required human code review, 12 AI triage, literacy program (org control) |
| **ASI10: Rogue Agents** | An agent drifts or is compromised into harmful autonomy, the insider threat | 5 revocable identity, 11 telemetry, 12 kill switch + triage |

This is not a coincidence of taxonomy. The Microsoft Agent Governance Toolkit positions itself as covering **all ten** of the OWASP Agentic Top 10, and the architecture in this post is what that coverage looks like assembled from the four major coding agents plus the enterprise control plane around them. The same controls also satisfy the regulatory regimes a governance committee cares about: NIST AI RMF (Govern, Map, Measure, Manage), ISO/IEC 42001 (Annex B controls), and the EU AI Act (Art. 12 logging, Art. 14 human oversight and kill switch, Art. 15 robustness, Art. 26 deployer obligations). The cost of multi-regime compliance is in the evidence, not in duplicated controls.

## Organizational controls that matter as much as technical ones

- **A named owner for each agent surface.** "Coding agent platform owner" is a real job, not a side gig. Codex, Claude Code, Copilot, and Cursor each carry a different threat model and need a directly responsible individual.
- **A change-management policy that treats agent settings as code.** Settings JSON in version control, peer reviewed, deployed by MDM. This is how you survive `--dangerously-bypass-approvals-and-sandbox` regressions.
- **An incident playbook.** What happens when an agent exfiltrates a token? Who rotates? Who reviews repo history? Who notifies the regulator under NIS2 or DORA? Decide before you need to know.
- **A literacy program.** EU AI Act Article 4 expects this. Frame it as developer enablement, not compliance training: a 90-minute internal session on "how Codex actually works on your laptop" reduces both incident rate and approval-fatigue.

## What you should do this quarter

Three concrete moves:

1. **Inventory every coding agent in your environment**, including the ones on personal accounts. Block the personal accounts. SCIM + DLP + MDM.
2. **Pick one agent, deploy the 12 controls end-to-end.** Don't try to harden four agents at once. Codex is the easiest to start with because it has the most complete set of admin-enforced primitives and the cleanest telemetry path.
3. **Stand up the telemetry pipeline before you scale.** OpenTelemetry from every agent, into SIEM, with one rule that fires on egress to an unallowed domain. That single rule will pay for the entire program.

## Where to go deeper

Three places to take this further. The first two are the conversations that produced this post; the third is where the conversation goes next.

- **The April 2026 GenAI Gurus session on personal agents** with Maxim Vovshin walks through the inner architecture of an always-on agent harness (models, skills, harness, tools), the front-matter-first context engineering pattern that keeps token costs sane, and a specific set of opinionated safety patterns: separate identities, separate machines, deny-by-default for credentials that touch anything you cannot afford to lose. The Jensen Huang capability-separation rule above came up in the live Q&A. [Watch on YouTube.](https://www.youtube.com/live/TA04giWhM-g)
- **The May 2026 GenAI Gurus session on the Microsoft Agent Governance Toolkit** with Imran Siddique covers the kernel-space architecture, the policy-vs-prompt benchmark, the EU AI Act article-by-article mapping (Articles 5, 12, 13, 14, 15), and a CI/CD deployment-gate pattern in six lines of Python that fails the build if your agent does not meet the compliance bar. [Watch on YouTube.](https://youtu.be/uDQBqp9Om5s)
- **The next GenAI Gurus session, June 18, 2026, with Marius Hobbhahn (CEO and founder of Apollo Research)** picks up exactly where this post ends. This post argues that agent-native telemetry is the most under-built layer in most enterprises; Marius will present findings from *tens of thousands of real coding agent traces* (dangerous commands, data exfiltration attempts, insecure code modifications, instruction drift, scope creep, and overclaiming behavior), plus a working demo of Apollo's **Watcher** tool for real-time oversight, tied back to the OWASP Top 10 for Agentic Applications. If you only attend one community session this quarter, make it this one. [Details and RSVP on LinkedIn.](https://www.linkedin.com/posts/carloshvp_openclaw-agenticengineering-owasp-activity-7463195258156167169-J4Ra)

The GenAI Gurus community itself, 2,500+ practitioners, technical, European-rooted, is at [genai-gurus.com](https://genai-gurus.com).

---

## What's next

This is post #2 in a series on practical AI governance for AI agent developers using Microsoft's Agent Governance Toolkit and related open-source primitives:

- **Post 1:** [EU AI Act for AI Agent Developers: A Practical Compliance Checklist](/2026/04/10/eu-ai-act-compliance-checklist-for-ai-agent-developers/)
- **Post 2:** How to Run Coding Agents Safely in the Enterprise *(this post)*
- **Post 3:** From NIM to Jetson: A NeMo Guardrails Configuration Pack for EU AI Act Compliance
- **Post 4:** Open Weights, Real Obligations: EU AI Act for GPAI Providers and the Developers Who Deploy Them
- **Post 5:** Sovereign AI Infrastructure: EU AI Act Compliance for On-Prem and European Cloud Deployments
- **Post 6:** The Contributor Journey: Building the Governance Layer the EU AI Act Requires

The full series lives at [governance.ai-mvp.com](https://governance.ai-mvp.com).

If you found this useful, the companion resource is **[Awesome EU AI Act](https://github.com/GenAI-Gurus/awesome-eu-ai-act)**, a community-maintained list of 200+ official sources, open source tools, templates, and guides for EU AI Act compliance. A GitHub star helps other developers find it.

---

*Written by [Carlos Hernandez](https://www.linkedin.com/in/carloshvp), founder of [GenAI Gurus](https://genai-gurus.com), Europe's GenAI practitioner community. The Microsoft Agent Governance Toolkit is open source under MIT at [github.com/microsoft/agent-governance-toolkit](https://github.com/microsoft/agent-governance-toolkit). Vendor capability statements in this post reflect public documentation and primary security research as of May 2026; defaults and primitives in this space change fast, so verify against your installed versions before relying on the comparison table for a security review.*
