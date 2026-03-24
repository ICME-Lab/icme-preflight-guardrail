---
name: pre-flight
description: Stop your agent from doing things you didn't authorize. ICME checks every consequential action against your policy before it executes — email, transactions, file ops, external calls. Define your rules in plain English. Get SAT or UNSAT back in under a second. Includes a free logic contradiction detector (no account required).
homepage: https://icme.io
metadata:
  clawdbot:
    emoji: "🔐"
    requires:
      env:
        - ICME_API_KEY
        - ICME_POLICY_ID
      primaryEnv: ICME_API_KEY
---

# ICME Guardrails

Two tools. One free, one paid. Both use Automated Reasoning — your rules get translated into math and checked by a solver that gives you a definitive yes or no. No confidence scores, no guessing.

## Tool 1: checkLogic (free, no account)

Check any reasoning for logical contradictions before acting on it. No API key required.

### When to use

Call checkLogic when the agent is:
- **Planning multi-step actions** — verify the plan is internally consistent before executing step 1
- **Combining information from multiple sources** — catch conflicting facts before they produce wrong decisions
- **Working with numbers** — budgets, schedules, quantities, limits — where arithmetic contradictions hide in natural language

### How to use

POST to `/v1/checkLogic` with the reasoning as a single string:

```bash
curl -s -X POST https://api.icme.io/v1/checkLogic \
  -H "Content-Type: application/json" \
  -d '{"reasoning": "<the reasoning, plan, or statements to check>"}'
```

No streaming. Response is immediate JSON.

### Interpreting the result

| `result` | Meaning | What to do |
|---|---|---|
| `CONSISTENT` | No contradictions found | Proceed |
| `CONTRADICTION` | Statements are logically incompatible | **Stop. Report the conflicting claims to the user.** |

The response includes a `claims` array showing exactly which statements conflict and a `detail` field with a human-readable explanation.

### Example

```bash
curl -s -X POST https://api.icme.io/v1/checkLogic \
  -H "Content-Type: application/json" \
  -d '{"reasoning": "The budget is $10,000. I will spend $6,000 on marketing and $7,000 on engineering."}'
```

Returns `CONTRADICTION` — $6,000 + $7,000 exceeds the $10,000 budget.

### When to skip

If the agent is doing something simple with no multi-step reasoning (answering a factual question, formatting text, reading a file), skip the check.

---

## Tool 2: checkIt (paid, policy-based)

Check any proposed action against a custom policy compiled from your plain English rules. This is the full guardrail — every consequential action gets verified against your specific constraints.

### When to use

Call checkIt before any action that is:
- **Irreversible** — sending email, executing a transaction, deleting files, creating accounts
- **External** — outbound API calls, messages to third parties, financial operations
- **Privileged** — anything touching credentials, billing, or user data

If the action is low-stakes and fully reversible (reading a file, fetching a URL, formatting text), skip the check. If in doubt, check.

### How to check an action

POST to `/v1/checkIt` with your `policy_id` and the proposed action as a plain English string. Describe the action specifically — include amounts, recipients, subjects, and any other relevant details.

```bash
curl -s -N -X POST https://api.icme.io/v1/checkIt \
  -H 'Content-Type: application/json' \
  -H "X-API-Key: $ICME_API_KEY" \
  -d "{
    \"policy_id\": \"$ICME_POLICY_ID\",
    \"action\": \"<describe the action in plain English>\"
  }"
```

The endpoint streams SSE. Read until you receive a `"step":"done"` event. Parse the final event's JSON for the result.

### Interpreting the result

| `result` | Meaning | What to do |
|---|---|---|
| `SAT` | Action satisfies all policy rules | Proceed with the action |
| `UNSAT` | Action violates one or more rules | **Do not execute. Stop and report to the user.** |
| `ERROR` | Verification failed | Treat as UNSAT. Do not proceed. Fail closed. |

**Always fail closed.** If the API is unreachable, the response is malformed, or the result is anything other than an explicit `SAT`, do not execute the action.

### What to tell the user when blocked

When a result is `UNSAT`, report clearly:

```
Action blocked by policy.
Action: <what the agent tried to do>
Reason: <detail field from the response>
Check ID: <check_id from the response>
```

Do not attempt to rephrase the action and retry. A blocked action is blocked.

### What to do when credits run out

If the API returns `INSUFFICIENT_CREDITS` (HTTP 402), stop immediately. Do not attempt the action. Tell the user:

```
ICME guardrail check failed — out of credits.
The action has not been executed.
To continue: top up at https://icme.io or run:

curl -s -X POST https://api.icme.io/v1/topUpCard \
  -H 'Content-Type: application/json' \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{"amount_usd": 10}' | jq .

Open the checkout_url in your browser to pay by card.
```

Credits must be topped up by a human via the browser. Do not attempt to proceed without a successful guardrail check.

### Action description guidelines

Write action strings the way you would describe them to a human — specific, honest, complete.

```
# ✅ Good — specific and complete
"Send email to claims@lemonade.com with subject 'Formal Dispute: Claim #LM-2024-8821' citing policy coverage clause 4.2 to contest rejection of Bruno's veterinary claim."

# ✅ Good — includes amount and recipient
"Transfer 250 USDC from main wallet to 0xABC123 for freelancer invoice #47."

# ❌ Bad — vague
"Send an email about the claim."

# ❌ Bad — omits the key detail
"Make a payment."
```

Be specific. The policy checker extracts values (amounts, recipients, subjects) from your action string — vague descriptions produce less reliable results.

---

## Setup

checkLogic requires no setup. It works immediately with no account or API key.

checkIt requires a one-time setup by a human (not the agent):

### Required environment variables

| Variable | Description |
|---|---|
| `ICME_API_KEY` | Your ICME API key — starts with `sk-smt-` |
| `ICME_POLICY_ID` | UUID of your compiled policy from `/v1/makeRules` |

### 1. Create an account

```bash
curl -s -X POST https://api.icme.io/v1/createUserCard \
  -H 'Content-Type: application/json' \
  -d '{"username": "YOUR_USERNAME"}' | jq .
# Open checkout_url in your browser — $5.00 by card
# Then retrieve your API key:
curl -s https://api.icme.io/v1/session/SESSION_ID | jq .
```

### 2. Top up credits

```bash
curl -s -X POST https://api.icme.io/v1/topUpCard \
  -H 'Content-Type: application/json' \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{"amount_usd": 10}' | jq .
# Open checkout_url in your browser — $10 = 1,050 credits
```

### 3. Compile your policy

Write your rules in plain English — one constraint per numbered line:

```bash
curl -s -N -X POST https://api.icme.io/v1/makeRules \
  -H 'Content-Type: application/json' \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{
    "policy": "1. No email may be sent to an external party without a confirmation token.\n2. No financial transaction may exceed $100.\n3. File deletions are not permitted outside the /tmp directory."
  }'
# Copy the policy_id from the response and set it as ICME_POLICY_ID
```

Policy compilation costs 300 credits ($3.00), one-time. Each `checkIt` call costs 1 credit ($0.01).

### 4. Set environment variables

```bash
export ICME_API_KEY=sk-smt-...
export ICME_POLICY_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

## Further reading

- [ICME Documentation](https://docs.icme.io/documentation)
- [Writing Effective Policies](https://docs.icme.io/documentation/basics/writing-effective-policies)
- [API Reference](https://docs.icme.io/api-reference)
- [Paper: Succinctly Verifiable Agentic Guardrails](https://arxiv.org/abs/2602.17452)
- [MCP Server (npm)](https://www.npmjs.com/package/icme-preflight-mcp)