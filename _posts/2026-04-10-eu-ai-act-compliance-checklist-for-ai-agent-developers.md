---
layout: post
title: "EU AI Act for AI Agent Developers: A Practical Compliance Checklist"
date: 2026-04-10 09:00:00 +0000
categories: [compliance, eu-ai-act]
tags: [eu-ai-act, agent-governance, microsoft, compliance, python]
description: >-
  A practical compliance checklist for AI agent developers facing the August 2, 2026
  EU AI Act deadline, using Microsoft's Agent Governance Toolkit with real code examples.
---

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true, theme: 'neutral', securityLevel: 'loose' });
</script>

*Update, June 12, 2026: the Digital Omnibus political agreement moves the EU AI Act's high-risk deadlines referenced in this post from August 2, 2026 to December 2, 2027 (stand-alone high-risk systems) and August 2, 2028 (high-risk systems embedded in products), subject to final adoption. AI-enabled machinery is expected to be handled through the Machinery Regulation route instead of the AI Act's direct high-risk regime. The controls and artifacts in this checklist are unchanged; the dates are not. See the regulatory note in [post #3](/2026/06/12/software-promises-hardware-proofs/).*

<style>
.terminal-window {
  background: #1e1e1e;
  border-radius: 8px;
  margin: 1.5em 0;
  overflow: hidden;
  font-size: 0.85em;
  box-shadow: 0 4px 12px rgba(0,0,0,0.3);
}
.terminal-header {
  background: #3a3a3a;
  padding: 8px 14px;
  display: flex;
  align-items: center;
  gap: 6px;
}
.terminal-dot {
  width: 12px; height: 12px;
  border-radius: 50%;
  display: inline-block;
}
.dot-red   { background: #ff5f56; }
.dot-amber { background: #ffbd2e; }
.dot-green { background: #27c93f; }
.terminal-title {
  color: #999;
  font-family: monospace;
  font-size: 0.9em;
  margin-left: 8px;
}
.terminal-body {
  padding: 16px 20px;
  color: #d4d4d4;
  font-family: 'SFMono-Regular', Consolas, monospace;
  line-height: 1.6;
  white-space: pre;
  overflow-x: auto;
  background: #1e1e1e;
  margin: 0;
  border: none;
}
.t-ok   { color: #27c93f; }
.t-warn { color: #ffbd2e; }
.t-fail { color: #ff5f56; }
.t-dim  { color: #6a6a6a; }
.t-bold { color: #ffffff; font-weight: bold; }
</style>

**August 2, 2026 is fewer than four months away.** That is when EU AI Act obligations for high-risk AI systems (including the transparency requirements of Article 50) become enforceable. If you are building AI agents, you need to know whether your system is in scope, what you are required to do, and how to get there without starting from scratch. The European Commission's [Navigating the AI Act FAQ](https://digital-strategy.ec.europa.eu/en/faqs/navigating-ai-act) is a good orientation if you are new to the regulation.

Before the checklist, one thing needs to be said clearly: **your model passing safety benchmarks does not make your agent compliant.**

Model safety and agent governance are different layers. Model safety focuses on what a model *generates*: training-time alignment, content filtering, red-teaming results. Agent governance focuses on what a system *executes*: runtime decisions, tool calls, audit records, and disclosure to users and deployers. AGT addresses the execution layer of compliance; the Act itself reaches much further, into documentation, data governance, transparency, instructions for use, monitoring, and organisational measures. Your RLHF fine-tune and your toxicity filter say nothing about your audit trail, your risk management process, or your transparency disclosures.

This checklist uses **[Microsoft's Agent Governance Toolkit](https://github.com/microsoft/agent-governance-toolkit)** (AGT) as the practical tooling reference. We will use an **HR screening agent** as our running example: an agent that parses CVs, scores candidates, and generates shortlists for a hiring workflow.

```bash
# Full toolkit in one step (see QUICKSTART.md for detailed setup)
# https://github.com/microsoft/agent-governance-toolkit/blob/main/QUICKSTART.md
pip install "agent-governance-toolkit[full]"

# Or install individual packages
pip install agent-os-kernel agentmesh-platform agentmesh-runtime agent-sre
```

> **Package naming warning:** On PyPI, the bare `agentmesh` package is an unrelated 2024 placeholder, not Microsoft's AgentMesh. Use `agentmesh-platform` for the AgentMesh component. Verify all package names against the [repository's installation guide](https://github.com/microsoft/agent-governance-toolkit) before running in a production environment.

### How the toolkit maps to the law

Before diving into the checklist, here is how AGT's components align to the articles you need to satisfy:

<pre class="mermaid">
graph LR
    subgraph AGT["Agent Governance Toolkit"]
        OS["Agent OS\nPolicy Engine"]
        MESH["AgentMesh\nIdentity + Trust"]
        SRE["Agent SRE\nSLOs + Reliability"]
        COMP["Agent Compliance\nAttestation CLI"]
    end
    OS -->|runtime enforcement| A9["Art. 9\nRisk Mgmt"]
    OS -->|audit trail| A12["Art. 12\nLogging"]
    OS -->|kill switch + approvals| A14["Art. 14\nHuman Oversight"]
    OS -->|disclosure interceptor| A50["Art. 50\nTransparency"]
    MESH -->|DID identity| A13["Art. 13\nTransparency\nto Deployers"]
    SRE -->|SLOs + thresholds| A15["Art. 15\nAccuracy"]
    COMP -->|dossier export| A11["Art. 11\nTech Docs"]
</pre>

---

## Step 0: Are you in scope?

Not every AI agent triggers the full obligation stack. The Act creates a risk hierarchy.

**High-risk AI systems** ([Annex III](https://artificialintelligenceact.eu/annex/3/)) face the heaviest obligations. These are systems operating in eight domains: biometrics, critical infrastructure, education, employment, essential services (credit scoring, healthcare, emergency triage), law enforcement, migration/border control, and justice/democracy.

**Limited-risk systems** (AI that interacts with users without falling in Annex III) face only Article 50 transparency obligations.

<pre class="mermaid">
flowchart TD
    A["Your AI Agent"] --> B{"Operates in an\nAnnex III domain?"}
    B -->|No| C["Limited Risk\nArt. 50 transparency only"]
    B -->|Yes| D{"Profiles\nnatural persons?"}
    D -->|Yes| E["HIGH RISK\nFull Arts. 9 to 15 obligations"]
    D -->|No| F{"Art. 6(3) exemption\napplies?"}
    F -->|Yes: narrow procedural task| G["Not high-risk\nArt. 50 still applies"]
    F -->|No| E
    style E fill:#f8d7da,stroke:#dc3545
    style C fill:#d4edda,stroke:#28a745
    style G fill:#fff3cd,stroke:#ffc107
</pre>

Let us classify the HR screening agent using the toolkit's `EUAIActRiskClassifier`:

```python
from agentmesh.governance.eu_ai_act import (
    EUAIActRiskClassifier,
    AgentRiskProfile,
    RiskLevel,
)

profile = AgentRiskProfile(
    name="hr-screening-agent",
    description="Automated CV screening, candidate scoring, and shortlist generation",
    domain="employment",
    capabilities=["resume_parsing", "candidate_scoring", "shortlist_generation"],
    involves_profiling=True,   # evaluates personal characteristics
    is_safety_component=False,
)

classifier = EUAIActRiskClassifier()
result = classifier.classify(profile)

print(result.risk_level)          # RiskLevel.HIGH
print(result.profiling_override)  # True
print(result.triggers)
# ['annex_iii_domain_employment', 'involves_profiling']
```

Notice `profiling_override=True`. For systems that already fall within an Annex III use case, involving profiling of natural persons blocks the [Article 6(3) exemption](https://artificialintelligenceact.eu/article/6/). That exemption lets some Annex III systems escape the high-risk classification when they perform only narrow procedural or preparatory tasks, but it explicitly does not apply once profiling is in scope (cross-referenced to GDPR Article 4(4)). An agent that evaluates CV content, infers competencies, and ranks candidates is profiling within an Annex III domain, which is why `profiling_override` fires here.

> **Gap:** `EUAIActRiskClassifier` currently lives in the toolkit's `examples/` directory, not yet a production library export. Domain sets are static YAML; if the EU updates Annex III, you will need to update your config file manually. Use it as a well-structured starting point, not a certified compliance tool.

If your agent scores minimal risk, the remaining checklist items below are optional best practice rather than legal requirements. Note that [Article 4 AI literacy obligations](https://digital-strategy.ec.europa.eu/en/faqs/ai-literacy-questions-answers) entered into application before August 2026 and apply regardless of risk tier: you are already required to ensure your team has appropriate AI literacy for the systems they deploy and use.

---

## 1. Risk management system (Article 9)

**What the law requires:** A continuous, iterative risk management process throughout the AI system's lifecycle, not a one-time pre-deployment assessment. You must identify known and foreseeable risks (including under misuse), implement mitigation measures, document residual risks for deployers, and test before market placement. The process must specifically assess impacts on vulnerable populations.

**What the toolkit provides:** The Agent OS policy engine intercepts every tool call and agent action before execution at sub-millisecond latency. Policies are written in YAML, OPA Rego, or Cedar:

```yaml
# policy.yaml: require human approval before generating shortlists
- id: hr-shortlist-human-approval
  description: Block shortlist generation without human sign-off
  scope: ["shortlist_generation"]
  action: require_approval
  conditions:
    candidate_count_threshold: 10
    escalation_on_timeout: block
  audit: true
```

Agent SRE adds SLO-based risk containment: when your safety SLI drops below 99% (more than 1% policy violations in the measurement window), agent capabilities are automatically restricted via circuit breaker.

> **Gap:** Article 9 requires *lifecycle* risk management including post-market monitoring. The toolkit handles runtime enforcement well but has no built-in feedback loop from production observation back into your risk policies. You need to build that connection by periodically reviewing audit logs, identifying new failure modes, and updating your policy set. Treat this as a scheduled maintenance task, not a one-time configuration.

---

## 2. Technical documentation (Article 11 and Annex IV)

**What the law requires:** Technical documentation must be prepared *before* market placement, kept continuously updated, and retained for 10 years. It must contain nine sections specified in Annex IV, covering system description, development process, monitoring, performance metrics, risk management, lifecycle changes, standards applied, declaration of conformity, and post-market monitoring plan. The preparation effort for a complex system is substantial, particularly if design decisions were not documented as the system was built.

**What the toolkit provides:** `TechnicalDocumentationExporter` auto-generates the documentation sections it can infer from your governance artifacts:

```python
from agentmesh.governance.annex_iv import TechnicalDocumentationExporter, to_markdown

exporter = TechnicalDocumentationExporter(
    system_name="hr-screening-agent",
    provider="Acme Corp",
    system_description="Automated CV screening, scoring, and shortlist generation for recruitment",
    system_version="1.2.0",
)

# Feed in artifacts the toolkit has already collected
exporter.add_compliance_report(compliance_report)   # from agent-compliance verify
exporter.add_policies(active_policies)               # from Agent OS
exporter.add_audit_entries(recent_audit_log)         # from Agent OS audit trail
exporter.add_slo_data(sre_metrics)                   # from Agent SRE

doc = exporter.export()
print(to_markdown(doc))   # Annex IV dossier, ready for review and filing
```

Sections 1 through 5 (general description, development process, monitoring, performance metrics, risk management) are auto-populated from toolkit artifacts. Sections 6 through 9 are marked as `placeholder` fields requiring human input.

> **Gap:** Roughly half the Annex IV content cannot be auto-generated. Design rationale, training data provenance (datasheets), third-party evaluation results, and your post-market monitoring plan are things only you can write. Start this documentation *now*, before market placement. The Act requires it to exist before you ship, and the 10-year retention clock starts at that point.

---

## 3. Record-keeping and logging (Article 12)

**What the law requires:** High-risk AI systems must technically allow automatic event recording throughout their lifetime. Logs must support post-market monitoring and risk identification. Deployers must be able to access, collect, store, and interpret them.

**What the toolkit provides:** This is the toolkit's strongest area of coverage. Agent OS logs every policy decision automatically: tool call requests, evaluation outcomes (allowed, blocked, or modified), reasons, timestamps, agent identity, and session context. Every audit entry is structured and immutable.

For the HR screening agent, every candidate scoring request, every shortlist generation attempt, every human approval trigger, and every policy violation is recorded with a complete decision trace.

**Action required:** Ensure your deployers can access and export these logs. The toolkit emits structured OpenTelemetry traces and integrates natively with Datadog, Prometheus, Langfuse, and PagerDuty. Wire the audit trail to your logging infrastructure and document how deployers can query it. This forms part of your Article 13 instructions for use.

---

## 4. Transparency to deployers (Article 13)

**What the law requires:** Systems must be designed to give deployers sufficient transparency to understand and correctly use outputs. Instructions for use must cover: provider identity, performance characteristics and limitations (accuracy levels, known failure modes, input data specifications), human oversight requirements, and log collection guidance.

**What the toolkit provides:** Every AGT agent has a cryptographically verifiable identity via Decentralized Identifiers (DIDs) with Ed25519 key pairs. Every action the agent takes is signed and attributable. The compliance report from `agent-compliance verify` gives deployers a structured view of what the system covers and where gaps remain.

**Action required:** Write your instructions for use document. The toolkit gives you the audit infrastructure and identity layer; the documentation is still on you. It should include: the performance thresholds you have declared (see checklist item 6), known failure cases, bias risks specific to your training data, and how deployers access and interpret audit logs.

---

## 5. Human oversight (Article 14)

**What the law requires:** Systems must include tools enabling effective human oversight. Operators must be able to understand system capabilities, detect automation bias, interpret outputs correctly, choose *not to use* an output, and *interrupt or stop* the system. Requirements scale with the agent's level of autonomy.

**What the toolkit provides:** Three mechanisms cover the Article 14 requirements directly.

**Kill switch:** AgentMesh Runtime includes a system-level kill switch that immediately halts agent execution across all active sessions.

**Approval workflows with quorum logic:**

```yaml
# Two reviewers must approve before the final shortlist is delivered
- id: shortlist-final-approval
  scope: ["shortlist_delivery"]
  action: require_approval
  conditions:
    quorum: 2
    timeout_minutes: 120
    escalation_on_timeout: block
  audit: true
```

Here is how that flow looks at runtime when the agent reaches the shortlist delivery step:

<pre class="mermaid">
sequenceDiagram
    participant A as HR Screening Agent
    participant OS as Agent OS
    participant R1 as Reviewer 1
    participant R2 as Reviewer 2
    A->>OS: shortlist_delivery request
    OS->>OS: Policy check: quorum=2 required
    OS-->>R1: Approval request sent
    OS-->>R2: Approval request sent
    R1->>OS: ✓ Approved
    R2->>OS: ✓ Approved
    OS->>A: ✓ Action permitted
    Note over OS: Art. 12 audit entry logged
</pre>

**Human-in-the-loop gates:** Capability boundaries that pause execution pending human confirmation before high-stakes actions (contacting candidates, writing to HR systems, making external API calls).

> **Gap:** Article 14 requires oversight measures "commensurate with the level of autonomy." As your agent gains new capabilities (new tools, new domains, new integrations), your oversight policies need to be updated to match. The toolkit has no mechanism to flag when a policy set may no longer be adequate for an expanded agent scope. Build a policy review cadence into your release process.

---

## 6. Accuracy, robustness, and transparency

### Article 15: Accuracy thresholds

`AccuracyDeclaration` lets you formally declare and validate your Article 15 accuracy commitments against live SLI data:

```python
from agent_sre.accuracy_declaration import AccuracyDeclaration

declaration = AccuracyDeclaration.for_high_risk("hr-screening-agent")
# Sets threshold commitments for a high-risk system:
#   tool_call_accuracy >= 95% minimum  (99% recommended)
#   hallucination_rate <= 5%  maximum  (2%  recommended)
#   task_success_rate  >= 90% minimum  (95% recommended)
#   calibration_delta  <= 10% maximum  (5%  recommended)

# Validate against live SLI metrics from Agent SRE
ok, msg = declaration.validate_against_sli("task_success_rate", 0.92)
print(ok, msg)   # True  "task_success_rate: 0.92 >= 0.90 ✓"

ok, msg = declaration.validate_against_sli("hallucination_rate", 0.08)
print(ok, msg)   # False "hallucination_rate: 0.08 > 0.05 ✗"
```

Wire this into your CI/CD pipeline. A failing threshold should block deployment.

### Article 50: Transparency for all AI systems

Article 50 covers two separate obligations that apply to different categories of systems. They are not the same duty.

**Article 50(1) applies to interactive systems.** Any AI system intended to interact directly with natural persons must notify the user they are interacting with an AI at first contact, unless this is obvious from the context. The Commission's [guidelines on transparent AI systems](https://digital-strategy.ec.europa.eu/en/faqs/guidelines-and-code-practice-transparent-ai-systems) explain how this is expected to work in practice.

**Article 50(2) applies to generative systems.** Providers of AI systems that generate synthetic audio, image, video, or text must mark that output as artificially generated in a machine-readable format. This obligation applies to the content itself, not to the interaction. The Commission's [code of practice on marking AI-generated content](https://digital-strategy.ec.europa.eu/en/policies/code-practice-ai-generated-content) is the voluntary implementation framework currently being developed for this track.

An HR screening agent that converses with candidates owes the first obligation. If it also produces AI-generated written outputs delivered to those candidates, it owes the second as well. Not every interactive system generates synthetic content, and not every generative system interacts directly with people.

The August 2, 2026 enforcement date applies to Article 50 obligations as written in the Act. The Commission's ongoing guidance process continues to develop practical implementation detail, so treat the Act text as the current baseline and monitor delegated acts as they are published.

`TransparencyInterceptor` handles the first obligation:

```python
from agent_os.transparency import TransparencyInterceptor, TransparencyLevel

interceptor = TransparencyInterceptor(
    default_level=TransparencyLevel.ENHANCED,
    require_disclosure_confirmation=True,
)

# At session start: deliver disclosure text and record confirmation
print(interceptor.get_disclosure_text(TransparencyLevel.ENHANCED))
# "You are interacting with an AI system governed by policy enforcement rules.
#  All interactions are logged and subject to human oversight..."

interceptor.confirm_disclosure(session_id="candidate-session-001")

# All subsequent tool calls are validated for disclosure status
result = interceptor.intercept(tool_call_request)
# result.allowed = True (disclosure confirmed)
# result.modified_arguments includes _ai_disclosure metadata marker
```

**The multi-agent transparency chain problem:** When your HR agent calls a background enrichment agent, which calls an external data API, disclosure ownership becomes ambiguous. The Act says the *provider of the human-facing system* is responsible. Design your disclosure flow at the outermost boundary:

<pre class="mermaid">
graph LR
    H["👤 Candidate"] -->|session starts| A["HR Screening Agent\n✓ Art. 50 disclosure here\nTransparencyInterceptor active"]
    A -->|internal call| B["Enrichment Agent\nno direct human contact\ndisclosure not required"]
    B -->|API call| C["External\nData Source"]
    style A fill:#d4edda,stroke:#28a745
    style B fill:#fff3cd,stroke:#ffc107
    style H fill:#cce5ff,stroke:#0056b3
    style C fill:#f8f9fa,stroke:#6c757d
</pre>

For fully autonomous pipelines where no single agent is clearly "human-facing," this remains an unresolved question in the regulation.

> **Gap:** `TransparencyInterceptor` handles disclosure confirmation and metadata injection but does not implement cryptographic watermarking of generated text (Art. 50(2) machine-readable markers). This requires a separate solution: evaluate C2PA-compatible tools or your LLM provider's native watermarking API.

---

## Running the compliance report

Once the toolkit components are wired up, run a compliance attestation with `agent-compliance verify`:

<div class="terminal-window">
  <div class="terminal-header">
    <span class="terminal-dot dot-red"></span>
    <span class="terminal-dot dot-amber"></span>
    <span class="terminal-dot dot-green"></span>
    <span class="terminal-title">agent-compliance verify --agent hr-screening-agent</span>
  </div>
  <pre class="terminal-body"><span class="t-dim">$</span> agent-compliance verify --agent hr-screening-agent

<span class="t-bold">Agent Governance Toolkit: Compliance Report</span>
<span class="t-dim">────────────────────────────────────────────────────</span>
System:   hr-screening-agent v1.2.0
Provider: Acme Corp
Profile:  <span class="t-fail">HIGH RISK</span> (Annex III: Employment, profiling override)

<span class="t-bold">Article    Coverage    Conformity Risk   Status</span>
<span class="t-dim">────────────────────────────────────────────────────</span>
Art. 6     Partial     <span class="t-fail">HIGH</span>              <span class="t-warn">⚠</span>  examples/ only, not library
Art. 9     Partial     <span class="t-fail">HIGH</span>              <span class="t-warn">⚠</span>  no lifecycle feedback loop
Art. 11    Partial     MEDIUM            <span class="t-warn">○</span>  manual sections required
Art. 12    Covered     <span class="t-ok">LOW</span>               <span class="t-ok">✓</span>  full audit trail active
Art. 13    Partial     MEDIUM            <span class="t-warn">○</span>  instructions for use needed
Art. 14    Covered     <span class="t-ok">LOW</span>               <span class="t-ok">✓</span>  kill switch + approvals wired
Art. 15    Partial     MEDIUM            <span class="t-warn">○</span>  thresholds declared, not validated
Art. 50    Partial     MEDIUM            <span class="t-warn">○</span>  watermarking not configured
<span class="t-dim">────────────────────────────────────────────────────</span>
Overall conformity risk: <span class="t-fail">HIGH</span>
Signed attestation:      <span class="t-dim">sha256:a3f9c2...8d721</span>

<span class="t-dim">Run with --json to pipe output to CI/CD.</span></pre>
</div>

Integrate into CI/CD to fail builds on unmitigated high-risk findings:

```bash
agent-compliance verify --json | python -c "
import json, sys
report = json.load(sys.stdin)
failures = [
    f for f in report.get('findings', [])
    if f.get('conformity_risk') == 'HIGH' and not f.get('mitigated')
]
if failures:
    for f in failures: print(f'FAIL: {f[\"article\"]}: {f[\"gap\"]}')
    sys.exit(1)
print('Compliance check passed')
"
```

---

## What to do with the gaps

Every section above flagged at least one gap. This is expected. The Agent Governance Toolkit covers the runtime governance layer (policy enforcement, audit trails, identity, human oversight) but was never designed to be a complete EU AI Act compliance solution on its own.

**Prioritised gap list for an HR screening agent:**

1. **Post-market monitoring feedback loop (Art. 9):** Schedule quarterly policy reviews using production audit logs. Define what constitutes a risk event that triggers a policy update.
2. **Annex IV manual sections (Art. 11):** Write design rationale, training data documentation, and your post-market monitoring plan before you ship. The 10-year clock starts at market placement.
3. **Content watermarking (Art. 50(2)):** Evaluate C2PA tools or your LLM provider's native watermarking for AI-generated text delivered to candidates.
4. **[AI literacy obligations (Art. 4)](https://digital-strategy.ec.europa.eu/en/faqs/ai-literacy-questions-answers):** Train your team on the AI system. Entirely outside the toolkit's scope, and already in application before August 2026.
5. **Data governance (Art. 10):** Training data practices, bias testing, and dataset governance are not covered by AGT. You need a separate data governance process.

---

## Further reading

**The law**
- [EU AI Act — Official text (OJ L 2024/1689)](https://eur-lex.europa.eu/legal-content/EN/TXT/PDF/?uri=OJ%3AL_202401689) — the primary source
- [Article 6: Classification rules for high-risk AI systems](https://artificialintelligenceact.eu/article/6/) — annotated explainer
- [Annex III: High-risk AI system categories](https://artificialintelligenceact.eu/annex/3/) — annotated explainer

**European Commission guidance**
- [Navigating the AI Act FAQ](https://digital-strategy.ec.europa.eu/en/faqs/navigating-ai-act) — scope, high-risk classification, enforcement
- [Implementation timeline](https://ai-act-service-desk.ec.europa.eu/en/ai-act/timeline/timeline-implementation-eu-ai-act) — phased rollout through August 2027
- [AI literacy Q&A (Article 4)](https://digital-strategy.ec.europa.eu/en/faqs/ai-literacy-questions-answers) — what the literacy obligation requires in practice
- [Guidelines on transparent AI systems (Article 50)](https://digital-strategy.ec.europa.eu/en/faqs/guidelines-and-code-practice-transparent-ai-systems) — Art. 50(1) disclosure guidance
- [Code of practice on marking AI-generated content](https://digital-strategy.ec.europa.eu/en/policies/code-practice-ai-generated-content) — Art. 50(2) machine-readable marking framework

**Microsoft Agent Governance Toolkit**
- [Repository](https://github.com/microsoft/agent-governance-toolkit)
- [Quickstart guide](https://github.com/microsoft/agent-governance-toolkit/blob/main/QUICKSTART.md)
- [Architecture overview](https://github.com/microsoft/agent-governance-toolkit/blob/main/docs/ARCHITECTURE.md)
- [Threat model](https://github.com/microsoft/agent-governance-toolkit/blob/main/docs/THREAT_MODEL.md)

**Community resources**
- [Awesome EU AI Act](https://genai-gurus.com/eu-ai-act) — 200+ curated resources: official sources, open source tools, templates, standards, and practical guides for EU AI Act compliance. Maintained by the [GenAI Gurus](https://genai-gurus.com) community.

---

## What's next

This is the first post in a series on practical AI governance for AI agent developers using Microsoft's Agent Governance Toolkit and related open-source primitives:

- **Post 2:** [How to Run Coding Agents Safely in the Enterprise](/2026/05/28/coding-agents-safely/), a reference architecture for adopting Codex, Claude Code, Copilot, and Cursor without burning your secrets, your IP, or your audit story
- **Post 3:** [Agents Can Restart. Accountability Cannot: Verifiable Trust for Embodied AI](/2026/06/12/software-promises-hardware-proofs/), where the Article 12 logging and Article 14 oversight artifacts from this checklist run against a robot cell, and machine safety stays with the certified layer
- **Post 4:** Code Is No Longer the Bottleneck. Verification Is: A Survey of AI-Native Software Delivery in 2026
- **Post 5:** From NIM to Jetson: A NeMo Guardrails Configuration Pack for Production Inference
- **Post 6:** Open Weights, Real Obligations: Governing GPAI Models You Deploy but Didn't Train
- **Post 7:** Sovereign AI Infrastructure: Governance Patterns for On-Prem and European Cloud
- **Post 8:** The Contributor Journey: Building an Open-Source Agent Governance Layer

The full series lives at [governance.ai-mvp.com](https://governance.ai-mvp.com).

If you found this useful, the companion resource is **[Awesome EU AI Act](https://github.com/GenAI-Gurus/awesome-eu-ai-act)** — a community-maintained list of 200+ official sources, open source tools, templates, and guides for EU AI Act compliance. A GitHub star helps other developers find it.

---

*Written by [Carlos Hernandez](https://www.linkedin.com/in/carloshvp), founder of [GenAI Gurus](https://genai-gurus.com) — Europe's GenAI practitioner community. This post was written as a contribution to [microsoft/agent-governance-toolkit issue #849](https://github.com/microsoft/agent-governance-toolkit/issues/849). The toolkit is open source under MIT at [github.com/microsoft/agent-governance-toolkit](https://github.com/microsoft/agent-governance-toolkit). The example code references source files in the repository; verify import paths and package names against your installed version, as the toolkit is under active development.*
