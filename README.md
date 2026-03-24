# PreFlight — Guardrails for OpenClaw

Automated Reasoning guardrails for OpenClaw agents. Every consequential action gets checked against your policy before it executes. SAT = allowed. UNSAT = blocked. Under one second.

## The problem

OpenClaw agents can send emails, execute transactions, delete files, run shell commands, and modify their own behavior. The most popular skill on ClawHub — Capability Evolver (35K+ downloads) — injects "You are a Recursive Self-Improving System" into your agent and runs in "Mad Dog Mode" by default, executing changes immediately without review.

That same skill has been [flagged for data exfiltration](https://github.com/openclaw/clawhub/issues/95) to a Chinese cloud service, has [contradictory documentation](https://clawhub.ai/autogame-17/capability-evolver) about whether it modifies source code, and carries a "Suspicious" security rating from ClawHub's own scanner.

Meanwhile, [Cisco found 26% of 31,000 agent skills contain at least one vulnerability](https://blogs.cisco.com/ai/personal-ai-agents-like-openclaw-are-a-security-nightmare). [Microsoft recommends treating OpenClaw as "untrusted code execution with persistent credentials."](https://www.microsoft.com/en-us/security/blog/2026/02/19/running-openclaw-safely-identity-isolation-runtime-risk/) And [824 malicious skills have been found on ClawHub](https://www.koi.ai/blog/clawhavoc-341-malicious-clawedbot-skills-found-by-the-bot-they-were-targeting) so far.

Every existing security tool in the ecosystem — Skill Vetter, Clawdex, VirusTotal — uses pattern matching or database lookups. None of them can tell you whether a proposed action *provably violates your policy*. PreFlight can.

## How it works

Your rules are written in plain English. PreFlight translates them into formal logic (SMT-LIB) and checks every action with a mathematical solver. The solver gives you a definitive yes or no — not a confidence score, not a probability, not a best guess.

This is the same class of technology AWS uses internally to verify IAM policies across [a billion SMT queries a day](https://blog.icme.io/what-is-automated-reasoning/).

## Two tools

### checkLogic — free, no account

Catches logical contradictions in your agent's reasoning before it acts. No API key required.

```bash
curl -s -X POST https://api.icme.io/v1/checkLogic \
  -H "Content-Type: application/json" \
  -d '{"reasoning": "The budget is $10,000. I will spend $6,000 on marketing and $7,000 on engineering."}'
```

Returns `CONTRADICTION` — because $6K + $7K exceeds the $10K budget. An LLM would execute this plan confidently. The solver catches it in milliseconds.

**Use with self-evolving agents:** Before Capability Evolver applies a mutation, pipe its proposed changes through checkLogic. If the evolution contradicts existing constraints, you'll know before it takes effect.

### checkIt — paid, policy-based

The full guardrail. Compile your rules once, then check every action against them.

```bash
# Check an action against your policy
curl -s -N -X POST https://api.icme.io/v1/checkIt \
  -H 'Content-Type: application/json' \
  -H "X-API-Key: $ICME_API_KEY" \
  -d '{
    "policy_id": "YOUR_POLICY_ID",
    "action": "Send email to claims@lemonade.com with subject Formal Dispute citing policy clause 4.2"
  }'
```

SAT = proceed. UNSAT = blocked. Always fail closed.

**Use with self-evolving agents:** Define what your agent is and isn't allowed to evolve. "No skill may request shell access." "No evolution may modify authentication flows." "No outbound data to external services without explicit user approval." The solver enforces these mathematically — the agent can't talk its way around them.

## Install

From ClawHub:

```bash
clawhub install wyattbenno777/pre-flight
```

Or copy the `SKILL.md` into your OpenClaw skills directory manually.

## Why this matters for Capability Evolver users

Capability Evolver's own docs say `EVOLVE_ALLOW_SELF_MODIFY=true` is "catastrophic." Their recommended safeguard is a boolean flag and a `--review` mode. Those are config settings — an agent with file system access can change config settings.

ICME's policy lives on an external server your agent cannot modify. The rules are compiled into formal logic once by a human. Every proposed action or evolution is checked against that logic by PreFlight's solver. The agent receives SAT or UNSAT. There is nothing to override, no flag to flip, no prompt to inject around.

That's the difference between "please don't do this" and "you mathematically cannot do this."

## Tested against a real attack

We wrote a 6-rule policy for an OpenClaw agent running Capability Evolver and tested it against the actual Feishu exfiltration reported in [GitHub Issue #95](https://github.com/openclaw/clawhub/issues/95):

| Action | Destination | Result | Solvers |
|---|---|---|---|
| Send evolution logs to Feishu (ByteDance) | Not on approved list | **UNSAT** (blocked) | Unanimous |
| Send evolution logs to EvoMap | On approved list | **SAT** (allowed) | Unanimous |

Same data, same action. The solver caught the distinction mathematically. Both results include a ZK proof receipt for independent verification.

Full walkthrough with policy, battle testing, and results: [Guardrails for Self-Evolving OpenClaw Agents](https://docs.icme.io/documentation/openclaw/cryptographic-guardrails-for-your-openclaw-agent/guardrails-for-self-evolving-openclaw-agents)

## Links

- **ClawHub:** [clawhub.ai/wyattbenno777/pre-flight](https://clawhub.ai/wyattbenno777/pre-flight)
- **Docs:** [docs.icme.io](https://docs.icme.io)
- **MCP Server (npm):** [icme-preflight-mcp](https://www.npmjs.com/package/icme-preflight-mcp)
- **Paper:** [Succinctly Verifiable Agentic Guardrails (arxiv)](https://arxiv.org/abs/2602.17452)

## License

MIT