# Practical AI Governance for Builders

Technical controls and runtime governance for teams deploying AI agents and
embodied systems.

**Read the publication:** [governance.ai-mvp.com](https://governance.ai-mvp.com/)

This project connects AI governance requirements to engineering practice:
sandboxing, policy enforcement, least-privilege identity, agent telemetry,
human approval, and verifiable evidence. The EU AI Act is an important
reference point, but the focus is broader: governance that can be implemented,
tested, and operated.

## Published series

1. [EU AI Act for AI Agent Developers: A Practical Compliance Checklist](https://governance.ai-mvp.com/2026/04/10/eu-ai-act-compliance-checklist-for-ai-agent-developers/)
2. [How to Run Coding Agents Safely in the Enterprise](https://governance.ai-mvp.com/2026/05/28/coding-agents-safely/)
3. [When Autonomous AI Agents Control Robots](https://governance.ai-mvp.com/2026/06/06/ten-thousand-safe-motions/)
4. [Verifiable Trust for AI Agents That Control Robots](https://governance.ai-mvp.com/2026/06/12/software-promises-hardware-proofs/)

## Related open-source work

- [Microsoft Agent Governance Toolkit contributions](https://github.com/microsoft/agent-governance-toolkit/pulls?q=is%3Apr+is%3Amerged+author%3Acarloshvp):
  fail-closed sandboxing, action-bound approval, physical-agent risk mapping,
  and compliance evidence
- [AgenTrust industrial embodied-AI example](https://github.com/agentrust-io/examples/pull/16)
- [AgenTrust evidence-continuity guidance](https://github.com/agentrust-io/examples/pull/18)
- [Awesome EU AI Act](https://github.com/GenAI-Gurus/awesome-eu-ai-act)

## Repository structure

```text
_posts/       Published articles
_drafts/      Work in progress
_includes/    Reusable page fragments
index.md      Publication home page
_config.yml   Jekyll and site metadata
```

## Run locally

Requirements: Ruby and Bundler.

```bash
bundle install
bundle exec jekyll serve
```

Open `http://localhost:4000`.

## About the author

[Carlos Hernandez](https://github.com/carloshvp) is an AI strategy lead and
open-source contributor focused on governable autonomy across enterprise
agents and embodied systems. He is the founder of
[GenAI Gurus](https://genai-gurus.com/).
