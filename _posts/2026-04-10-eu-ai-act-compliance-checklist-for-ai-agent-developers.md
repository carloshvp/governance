---
layout: post
title: "EU AI Act for AI Agent Developers — A Practical Compliance Checklist"
date: 2026-04-10 09:00:00 +0000
categories: [compliance, eu-ai-act]
tags: [eu-ai-act, agent-governance, microsoft, compliance, python]
description: >-
  A practical compliance checklist for AI agent developers facing the August 2, 2026
  EU AI Act deadline, using Microsoft's Agent Governance Toolkit with real code examples.
---

**August 2, 2026 is fewer than four months away.** That is when EU AI Act obligations for high-risk AI systems — and the transparency requirements of Article 50 — become enforceable. If you are building AI agents, you need to know whether your system is in scope, what you are required to do, and how to get there without starting from scratch.

Before the checklist, one thing needs to be said clearly: **your model passing safety benchmarks does not make your agent compliant.**

Model safety and agent governance are different layers. Model safety focuses on what a model *generates* — training-time alignment, content filtering, red-teaming results. Agent governance focuses on what a system *executes* — runtime decisions, tool calls, audit records, and disclosure to users and deployers. The EU AI Act governs the execution layer. Your RLHF fine-tune and your toxicity filter say nothing about your audit trail, your risk management process, or your transparency disclosures.

This checklist uses **[Microsoft's Agent Governance Toolkit](https://github.com/microsoft/agent-governance-toolkit)** (AGT) as the practical tooling reference. We will use an **HR screening agent** as our running example — an agent that parses CVs, scores candidates, and generates shortlists for a hiring workflow. Install the toolkit first:

```bash
pip install agent-governance-toolkit agentmesh agent-sre
```

---

## Step 0: Are you in scope?

Not every AI agent triggers the full obligation stack. The Act creates a risk hierarchy.

**High-risk AI systems** (Annex III) face the heaviest obligations. These are systems operating in eight domains: biometrics, critical infrastructure, education, employment, essential services (credit scoring, healthcare, emergency triage), law enforcement, migration/border control, and justice/democracy.

**Limited-risk systems** — AI that interacts with users without falling in Annex III — face only Article 50 transparency obligations.

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

Notice `profiling_override=True`. The Act has a hard rule: **any AI system that profiles natural persons is always high-risk**, regardless of the Article 6(3) exemptions. Article 6(3) lets some Annex III systems escape the high-risk classification if they perform only narrow procedural or preparatory tasks — but that exemption explicitly does not apply to systems that profile people (cross-referenced to GDPR Article 4(4)). An agent that evaluates CV content, infers competencies, and ranks candidates is profiling.

> **Gap:** `EUAIActRiskClassifier` currently lives in the toolkit's `examples/` directory, not yet a production library export. Domain sets are static YAML — if the EU updates Annex III, you will need to update your config file manually. Use it as a well-structured starting point, not a certified compliance tool.

If your agent scores minimal risk, Article 50 (transparency) is your only obligation. The checklist below still applies to any system you want to govern responsibly.

---

## 1. Risk management system — Article 9

**What the law requires:** A continuous, iterative risk management process throughout the AI system's lifecycle — not a one-time pre-deployment assessment. You must identify known and foreseeable risks (including under misuse), implement mitigation measures, document residual risks for deployers, and test before market placement. The process must specifically assess impacts on vulnerable populations.

**What the toolkit provides:** The Agent OS policy engine intercepts every tool call and agent action before execution at sub-millisecond latency. Policies are written in YAML, OPA Rego, or Cedar:

```yaml
# policy.yaml — require human approval before generating shortlists
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

> **Gap:** Article 9 requires *lifecycle* risk management including post-market monitoring. The toolkit handles runtime enforcement well but has no built-in feedback loop from production observation back into your risk policies. You need to build that connection — periodically reviewing audit logs, identifying new failure modes, and updating your policy set. Treat this as a scheduled maintenance task, not a one-time configuration.

---

## 2. Technical documentation — Article 11 and Annex IV

**What the law requires:** Technical documentation must be prepared *before* market placement, kept continuously updated, and retained for 10 years. It must contain nine sections specified in Annex IV — covering system description, development process, monitoring, performance metrics, risk management, lifecycle changes, standards applied, declaration of conformity, and post-market monitoring plan. Industry estimates put preparation time at 40–80 hours for a complex system, assuming design decisions were documented from the start.

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

> **Gap:** Roughly half the Annex IV content cannot be auto-generated. Design rationale, training data provenance (datasheets), third-party evaluation results, and your post-market monitoring plan are things only you can write. Start this documentation *now*, before market placement — the Act requires it to exist before you ship, and the 10-year retention clock starts at that point.

---

## 3. Record-keeping and logging — Article 12

**What the law requires:** High-risk AI systems must technically allow automatic event recording throughout their lifetime. Logs must support post-market monitoring and risk identification. Deployers must be able to access, collect, store, and interpret them.

**What the toolkit provides:** This is the toolkit's strongest area of coverage. Agent OS logs every policy decision automatically: tool call requests, evaluation outcomes (allowed, blocked, or modified), reasons, timestamps, agent identity, and session context. Every audit entry is structured and immutable.

For the HR screening agent, every candidate scoring request, every shortlist generation attempt, every human approval trigger, and every policy violation is recorded with a complete decision trace.

**Action required:** Ensure your deployers can access and export these logs. The toolkit emits structured OpenTelemetry traces and integrates natively with Datadog, Prometheus, Langfuse, and PagerDuty. Wire the audit trail to your logging infrastructure and document how deployers can query it — this forms part of your Article 13 instructions for use.

---

## 4. Transparency to deployers — Article 13

**What the law requires:** Systems must be designed to give deployers sufficient transparency to understand and correctly use outputs. Instructions for use must cover: provider identity, performance characteristics and limitations (accuracy levels, known failure modes, input data specifications), human oversight requirements, and log collection guidance.

**What the toolkit provides:** Every AGT agent has a cryptographically verifiable identity via Decentralized Identifiers (DIDs) with Ed25519 key pairs. Every action the agent takes is signed and attributable. The compliance report from `agent-compliance verify` gives deployers a structured view of what the system covers and where gaps remain.

**Action required:** Write your instructions for use document. The toolkit gives you the audit infrastructure and identity layer; the documentation is still on you. It should include: the performance thresholds you have declared (see checklist item 6), known failure cases, bias risks specific to your training data, and how deployers access and interpret audit logs.

---

## 5. Human oversight — Article 14

**What the law requires:** Systems must include tools enabling effective human oversight. Operators must be able to understand system capabilities, detect automation bias, interpret outputs correctly, choose *not to use* an output, and *interrupt or stop* the system. Requirements scale with the agent's level of autonomy.

**What the toolkit provides:** Three mechanisms cover the Article 14 requirements directly.

**Kill switch** — AgentMesh Runtime includes a system-level kill switch that immediately halts agent execution across all active sessions.

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

**Human-in-the-loop gates** — Capability boundaries that pause execution pending human confirmation before high-stakes actions (contacting candidates, writing to HR systems, making external API calls).

> **Gap:** Article 14 requires oversight measures "commensurate with the level of autonomy." As your agent gains new capabilities — new tools, new domains, new integrations — your oversight policies need to be updated to match. The toolkit has no mechanism to flag when a policy set may no longer be adequate for an expanded agent scope. Build a policy review cadence into your release process.

---

## 6. Accuracy, robustness, and transparency

### Article 15 — Accuracy thresholds

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

Wire this into your CI/CD pipeline — a failing threshold should block deployment.

### Article 50 — Transparency for all AI systems

This article applies regardless of risk classification: every AI system interacting with natural persons must (1) disclose it is an AI at first contact and (2) mark AI-generated content in machine-readable format.

`TransparencyInterceptor` handles the first obligation:

```python
from agent_os.transparency import TransparencyInterceptor, TransparencyLevel

interceptor = TransparencyInterceptor(
    default_level=TransparencyLevel.ENHANCED,
    require_disclosure_confirmation=True,
)

# At session start — deliver disclosure text and record confirmation
print(interceptor.get_disclosure_text(TransparencyLevel.ENHANCED))
# "You are interacting with an AI system governed by policy enforcement rules.
#  All interactions are logged and subject to human oversight..."

interceptor.confirm_disclosure(session_id="candidate-session-001")

# All subsequent tool calls are validated for disclosure status
result = interceptor.intercept(tool_call_request)
# result.allowed = True (disclosure confirmed)
# result.modified_arguments includes _ai_disclosure metadata marker
```

**The multi-agent transparency chain problem:** When your HR agent calls a background enrichment agent, which calls an external data API, Article 50(1) disclosure is still required at the point of human interaction. The Act says the *provider of the human-facing system* is responsible — even if subordinate agents do most of the work. Design your disclosure flow at the outermost human-facing boundary. For fully autonomous pipelines where no single agent is clearly "human-facing," this remains an unresolved question in the regulation.

> **Gap:** `TransparencyInterceptor` handles disclosure confirmation and metadata injection but does not implement cryptographic watermarking of generated text (Art. 50(2) machine-readable markers). This requires a separate solution — evaluate C2PA-compatible tools or your LLM provider's native watermarking API.

---

## Running the compliance report

Once the toolkit components are wired up, run a compliance attestation:

```bash
agent-compliance verify --agent hr-screening-agent --json > compliance-report.json
```

The report grades each EU AI Act article with a coverage level and conformity risk rating. Integrate it into CI/CD to fail builds on unmitigated high-risk findings:

```bash
agent-compliance verify --json | python -c "
import json, sys
report = json.load(sys.stdin)
failures = [
    f for f in report.get('findings', [])
    if f.get('conformity_risk') == 'HIGH' and not f.get('mitigated')
]
if failures:
    for f in failures:
        print(f'FAIL: {f[\"article\"]} — {f[\"gap\"]}')
    sys.exit(1)
print('Compliance check passed')
"
```

---

## What to do with the gaps

Every section above flagged at least one gap. This is expected. The Agent Governance Toolkit provides the runtime governance layer — policy enforcement, audit trails, identity, human oversight — but it was never designed to be a complete EU AI Act compliance solution on its own.

**Prioritised gap list for an HR screening agent:**

1. **Post-market monitoring feedback loop (Art. 9)** — Schedule quarterly policy reviews using production audit logs. Define what constitutes a risk event that triggers a policy update.
2. **Annex IV manual sections (Art. 11)** — Write design rationale, training data documentation, and your post-market monitoring plan before you ship. The 10-year clock starts at market placement.
3. **Content watermarking (Art. 50(2))** — Evaluate C2PA tools or your LLM provider's native watermarking for AI-generated text delivered to candidates.
4. **AI literacy obligations (Art. 4)** — Train your team on the AI system. Entirely outside the toolkit's scope.
5. **Data governance (Art. 10)** — Training data practices, bias testing, and dataset governance are not covered by AGT. You need a separate data governance process.

---

## What's next

This is the first post in a series on EU AI Act compliance for AI agent developers using Microsoft's Agent Governance Toolkit:

- **Post 2:** [Introducing the Agent Governance Toolkit — architecture and setup](/coming-soon)
- **Post 3:** [Building your Annex IV dossier with `annex_iv.py`](/coming-soon)
- **Post 4:** [Article 50 in agentic pipelines — the multi-agent transparency chain problem](/coming-soon)
- **Post 5:** [Contributing to Microsoft's open-source governance toolkit](/coming-soon)

The full series lives at [eu-ai-act.ai-mvp.com](https://eu-ai-act.ai-mvp.com).

---

*This post was written as a contribution to [microsoft/agent-governance-toolkit issue #849](https://github.com/microsoft/agent-governance-toolkit/issues/849). The toolkit is open source under MIT at [github.com/microsoft/agent-governance-toolkit](https://github.com/microsoft/agent-governance-toolkit). The example code in this post references actual source files in the repository — all imports are accurate as of v3.0.0.*
