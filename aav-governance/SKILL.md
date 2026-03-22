---
name: aav-governance
description: >
  Agent governance and authorization via Agent Authority Vault (AAV). Use this skill before any
  agent action that involves spending money, accessing sensitive data, contacting vendors, or
  performing operations that should be gated by human-defined policies. Integrates with the AAV
  verify API to enforce spending limits, vendor allowlists, action permissions, and autonomy
  levels. Triggers include: purchase, payment, spend, authorize, approve, vendor, transaction,
  budget, governance, compliance, and any action requiring human approval.
metadata:
  author: agentic-trust
  version: '1.0'
  homepage: https://agentauthority.dev
  support: support@agentictrust.app
---

# AAV Governance Skill

## Purpose

This skill makes your agent governance-aware. Before taking any consequential action — spending
money, contacting a vendor, accessing sensitive data, or performing an irreversible operation —
your agent calls the Agent Authority Vault (AAV) verify endpoint to check whether the action is
authorized under the human-defined grant policies.

AAV enforces an 11-step verification cascade:

1. Grant lookup by certificate ID
2. Grant active status (not expired or revoked)
3. Agent identity match
4. Requested action in allowed actions
5. Requested action not in prohibited actions
6. Vendor in approved vendor list
7. Amount within per-transaction spend limit
8. Amount within daily cumulative spend limit
9. Autonomy level check (Observe / Suggest / Confirm / Autonomous)
10. Human approval threshold check
11. All checks pass → authorized

## When to Use This Skill

Use this skill when your agent is about to:

- **Spend money** — purchasing, subscribing, tipping, paying invoices
- **Contact a vendor or external service** — sending emails, placing orders, calling APIs
- **Access sensitive resources** — reading private data, downloading documents, accessing databases
- **Perform irreversible actions** — deleting data, publishing content, signing agreements
- **Execute any action governed by organizational policy**

If the agent's action doesn't involve spending, external contact, sensitive data, or irreversible
consequences, you can skip verification.

## Prerequisites

1. An AAV account at [agentauthority.dev](https://agentauthority.dev)
2. A verify API key (starts with `aav_verify_`) — generate one from the API Keys page
3. A grant (certificate) configured for this agent with:
   - `certificate_id` — unique identifier for the grant
   - `agent_id` — must match the agent's identity
   - Allowed/prohibited actions, approved vendors, spend limits, and autonomy level

Store the API key and certificate ID securely. Never hard-code them in prompts visible to
end users.

## Configuration

Set these environment variables or pass them as parameters:

| Variable | Description | Example |
|---|---|---|
| `AAV_API_KEY` | Verify API key | `aav_verify_abc123...` |
| `AAV_BASE_URL` | API base URL | `https://api.agentictrust.app` |
| `AAV_CERTIFICATE_ID` | Grant certificate ID for this agent | `cert_xyz789...` |
| `AAV_AGENT_ID` | This agent's registered identity | `agent_mybot_001` |

## Verification Flow

### Step 1: Build the Verify Request

Before executing the action, construct a verification payload:

```json
{
  "certificate_id": "<your-certificate-id>",
  "agent_id": "<your-agent-id>",
  "requested_action": "purchase_supplies",
  "amount": 149.99,
  "currency": "USD",
  "vendor": "staples.com",
  "description": "Office supplies order — 3 boxes of paper, toner cartridge"
}
```

**Field reference:**

| Field | Type | Required | Description |
|---|---|---|---|
| `certificate_id` | string | Yes | The grant's certificate ID |
| `agent_id` | string | Yes | The agent's registered ID |
| `requested_action` | string | Yes | What the agent wants to do (e.g., `purchase`, `send_email`, `delete_record`) |
| `amount` | float | No | Dollar amount if the action involves spending |
| `currency` | string | No | Currency code (default: `USD`) |
| `vendor` | string | No | Vendor or service being interacted with |
| `description` | string | No | Human-readable description of the action |

### Step 2: Call the Verify Endpoint

```
POST {AAV_BASE_URL}/api/v1/verify
Authorization: Bearer {AAV_API_KEY}
Content-Type: application/json
```

**Example using curl:**

```bash
curl -X POST https://api.agentictrust.app/api/v1/verify \
  -H "Authorization: Bearer $AAV_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "certificate_id": "'"$AAV_CERTIFICATE_ID"'",
    "agent_id": "'"$AAV_AGENT_ID"'",
    "requested_action": "purchase_supplies",
    "amount": 149.99,
    "currency": "USD",
    "vendor": "staples.com",
    "description": "Office supplies — paper and toner"
  }'
```

**Example using Python:**

```python
import os
import requests

response = requests.post(
    f"{os.environ['AAV_BASE_URL']}/api/v1/verify",
    headers={
        "Authorization": f"Bearer {os.environ['AAV_API_KEY']}",
        "Content-Type": "application/json",
    },
    json={
        "certificate_id": os.environ["AAV_CERTIFICATE_ID"],
        "agent_id": os.environ["AAV_AGENT_ID"],
        "requested_action": "purchase_supplies",
        "amount": 149.99,
        "currency": "USD",
        "vendor": "staples.com",
        "description": "Office supplies — paper and toner",
    },
)

result = response.json()
```

### Step 3: Handle the Response

The verify endpoint returns a JSON response with these key fields:

| Field | Type | Description |
|---|---|---|
| `authorized` | bool | `true` if the action is allowed |
| `result` | string | One of: `authorized`, `denied`, `approval_required`, `observe_only`, `suggestion_only` |
| `reason` | string | Human-readable explanation of the decision |
| `verification_id` | string | Unique ID for this verification (use for audit trail) |
| `approval_id` | string | Present when `result` is `approval_required` — use to check approval status |
| `approval_url` | string | URL where a human can approve/deny the pending request |
| `details` | object | Additional context about the verification decision |

### Step 4: Act on the Result

Follow this decision tree based on the `result` field:

**`authorized`** — Proceed with the action.

```
✅ Action authorized. Proceeding with purchase of $149.99 from staples.com.
   Verification ID: ver_abc123 (logged for audit trail)
```

**`denied`** — Do NOT proceed. Inform the user why.

```
🚫 Action denied by governance policy.
   Reason: "Amount $149.99 exceeds per-transaction limit of $100.00"
   The agent cannot complete this purchase. Please adjust the grant
   limits or complete this purchase manually.
```

**`approval_required`** — Pause and wait for human approval.

```
⏳ This action requires human approval.
   Approval URL: https://agentauthority.dev/approvals/apr_xyz789
   A notification has been sent to the grant administrator.
   The agent will check back for approval status before proceeding.
```

When you receive `approval_required`:
1. Store the `approval_id` from the response
2. Present the `approval_url` to the user or notify the administrator
3. Poll or wait for the approval decision:
   - `POST /api/v1/approvals/{approval_id}/approve` — administrator approves
   - `POST /api/v1/approvals/{approval_id}/deny` — administrator denies
4. Only proceed with the action after receiving explicit approval

**`observe_only`** — Log the action but do NOT execute it.

```
👁️ Observe mode: This grant is set to autonomy level 1 (Observe).
   Action logged: purchase_supplies, $149.99, staples.com
   Verification ID: ver_abc123
   No action taken. The agent is in observation mode for this grant.
```

**`suggestion_only`** — Suggest the action to the user but do NOT execute it.

```
💡 Suggestion mode: This grant is set to autonomy level 2 (Suggest).
   Recommended action: Purchase $149.99 of office supplies from staples.com.
   Verification ID: ver_abc123
   Awaiting human decision — the agent will not act autonomously.
```

## Autonomy Levels Reference

| Level | Name | Agent Behavior |
|---|---|---|
| 1 | Observe | Agent logs what it would do. Takes no action. |
| 2 | Suggest | Agent recommends the action. Human decides. |
| 3 | Confirm | Agent requests approval. Proceeds only after explicit human approval. |
| 4 | Autonomous | Agent acts immediately if all other checks pass. |

## Error Handling

If the verify endpoint returns an HTTP error or is unreachable:

- **401 Unauthorized** — API key is invalid or expired. Do not proceed. Alert the user.
- **404 Not Found** — Certificate ID doesn't exist. Do not proceed. Check configuration.
- **422 Validation Error** — Request payload is malformed. Check required fields.
- **500 Server Error** — AAV service issue. Do not proceed with the action. Retry once after 5 seconds; if still failing, inform the user.
- **Network timeout** — AAV is unreachable. Do not proceed. Fail closed (deny by default).

**Critical rule: When in doubt, deny.** If the verify call fails for any reason, the agent
must NOT proceed with the action. Governance fails closed, never open.

## Audit Trail

Every verification call is logged in AAV's audit system. You can query the audit log:

```
GET {AAV_BASE_URL}/api/v1/audit
Authorization: Bearer {AAV_API_KEY}
```

This returns a history of all verify calls, approvals, denials, and grant changes — useful for
compliance reporting and reviewing agent behavior.

## Quick Integration Checklist

- [ ] AAV account created at agentauthority.dev
- [ ] Verify API key generated (`aav_verify_` prefix)
- [ ] Grant created with appropriate policies for this agent
- [ ] Environment variables set (`AAV_API_KEY`, `AAV_BASE_URL`, `AAV_CERTIFICATE_ID`, `AAV_AGENT_ID`)
- [ ] Verify call added before every consequential action
- [ ] All five response types handled (`authorized`, `denied`, `approval_required`, `observe_only`, `suggestion_only`)
- [ ] Error handling implements fail-closed behavior
- [ ] Tested with AAV Playground before going live

## Example: Full Agent Action Flow

Here's a complete example of an agent that needs to purchase office supplies:

```
User: "Order 3 boxes of paper and a toner cartridge from Staples."

Agent thinking:
  1. This involves spending money → must verify with AAV
  2. Build verify request:
     - action: "purchase_supplies"
     - amount: 149.99
     - vendor: "staples.com"
     - description: "3 boxes of paper, toner cartridge"
  3. Call POST /api/v1/verify
  4. Response: { "authorized": true, "result": "authorized", "verification_id": "ver_abc123" }
  5. Action is authorized → proceed with the purchase
  6. Complete the order on staples.com
  7. Log verification_id for audit trail

Agent response: "I've placed the order for 3 boxes of paper and a toner cartridge
from Staples ($149.99). This was authorized under your governance policy.
Verification ID: ver_abc123."
```

## Support

- Dashboard: [agentauthority.dev](https://agentauthority.dev)
- Documentation: [agentauthority.dev/docs](https://agentauthority.dev/docs)
- API Playground: [agentauthority.dev/playground](https://agentauthority.dev/playground)
- Email: support@agentictrust.app
