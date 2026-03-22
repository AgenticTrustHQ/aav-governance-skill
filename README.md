# AAV Governance Skill for OpenClaw Agents

**Stop your AI agents from going rogue.** This skill integrates [Agent Authority Vault (AAV)](https://agentauthority.dev) into any OpenClaw-compatible agent, giving you real-time governance over what your agents can spend, who they can contact, and what actions they can take.

## What It Does

Before your agent takes any consequential action — spending money, contacting vendors, accessing sensitive data — it calls AAV's verify endpoint to check the action against human-defined policies. AAV runs an 11-step verification cascade covering:

- **Action permissions** — allowed and prohibited action lists
- **Vendor allowlists** — only approved vendors/services
- **Spending limits** — per-transaction and daily cumulative caps
- **Autonomy levels** — Observe, Suggest, Confirm, or Autonomous
- **Human approval thresholds** — auto-escalate high-value actions

If the action isn't authorized, the agent stops. No exceptions. Governance fails closed.

## Quick Start

### 1. Get AAV credentials

Sign up at [agentauthority.dev](https://agentauthority.dev) and:
- Create a **grant** (certificate) defining your agent's permissions
- Generate a **verify API key** (starts with `aav_verify_`)

### 2. Install the skill

Copy `aav-governance/SKILL.md` into your agent's skill directory, or install via ClawMart (coming soon).

### 3. Set environment variables

```bash
export AAV_API_KEY="aav_verify_your_key_here"
export AAV_BASE_URL="https://api.agentictrust.app"
export AAV_CERTIFICATE_ID="cert_your_certificate_id"
export AAV_AGENT_ID="agent_your_agent_id"
```

### 4. Your agent is now governance-aware

The skill instructs the agent to call `POST /api/v1/verify` before any action involving money, vendors, sensitive data, or irreversible operations. The agent handles all five response types:

| Response | Agent Behavior |
|---|---|
| `authorized` | Proceeds with the action |
| `denied` | Stops and explains why |
| `approval_required` | Pauses, sends approval link to admin |
| `observe_only` | Logs the action, takes no action |
| `suggestion_only` | Recommends the action, waits for human decision |

## How It Works

```
User: "Order supplies from Staples for $150"
         │
         ▼
┌─────────────────────┐
│  Agent builds        │
│  verify request      │
│  (action, amount,    │
│   vendor, etc.)      │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐     ┌──────────────────┐
│  POST /api/v1/verify │────▶│  AAV 11-Step     │
│  (AAV API)           │◀────│  Verification    │
└─────────┬───────────┘     └──────────────────┘
          │
          ▼
    ┌─────────────┐
    │  authorized? │
    └──────┬──────┘
     yes   │   no
      ▼    │    ▼
  Proceed  │  Stop + explain
           │
     approval_required?
           │
           ▼
     Pause + notify admin
```

## Autonomy Levels

Configure how much freedom your agent gets:

| Level | Name | Behavior |
|---|---|---|
| 1 | **Observe** | Agent logs what it would do. Zero autonomy. |
| 2 | **Suggest** | Agent recommends actions. Human decides. |
| 3 | **Confirm** | Agent requests approval per-action. Proceeds only after explicit approval. |
| 4 | **Autonomous** | Agent acts freely within policy limits. |

## Repository Structure

```
aav-governance-skill/
├── README.md                     ← You are here
└── aav-governance/
    └── SKILL.md                  ← The skill file (add to your agent)
```

## Use Cases

- **E-commerce agents** — Cap spending, restrict to approved suppliers
- **Customer service bots** — Limit refund authority, escalate high-value cases
- **DevOps agents** — Gate infrastructure changes behind approval workflows
- **Research agents** — Control API spend and data access
- **Any agent that handles money or sensitive operations**

## Requirements

- An [AAV account](https://agentauthority.dev) (free tier available)
- A verify API key
- An OpenClaw-compatible agent (or any agent that can read `.md` skill files)

## Links

- **AAV Dashboard**: [agentauthority.dev](https://agentauthority.dev)
- **API Documentation**: [agentauthority.dev/docs](https://agentauthority.dev/docs)
- **API Playground**: [agentauthority.dev/playground](https://agentauthority.dev/playground)
- **Agentic Trust**: [agentictrust.app](https://agentictrust.app)

## Support

Questions or issues? Email [support@agentictrust.app](mailto:support@agentictrust.app)

## License

MIT — use it however you want.
