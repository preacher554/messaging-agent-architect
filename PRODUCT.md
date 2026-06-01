# Messaging Agent — Product Definition

## One-line positioning

A Messaging Agent is a managed AI frontdesk employee that helps businesses respond to customers, qualify requests, and coordinate human handoff inside the messaging channels customers already use.

## What we sell

We do **not** sell raw AI, prompts, vector databases, or automation tooling.

We sell a configured AI employee capability:

- **Basic:** AI Receptionist.
- **Pro:** AI Sales / Support Receptionist.
- **Custom/Add-on:** custom workflows, booking, payment, catalog, CRM, marketplace, or channel-specific integrations.

## Delivery model

1. Client provides channel access and business information.
2. Provider configures the messaging agent.
3. Agent answers inbound messages from approved knowledge and package scope.
4. Agent escalates to human admin when needed.
5. Provider monitors, optimizes, and maintains the setup.

## Strategic positioning

The v1 wedge is narrow on purpose:

- Messaging-native interaction.
- Human-AI collaboration, not AI-only automation.
- Clear package boundaries: Basic, Pro, Custom/Add-on.
- Reusable runtime architecture across channels.
- Channel-specific adapters only where needed.

## Value proposition

For the client:

- Fewer missed inquiries.
- Faster first response.
- Cleaner qualified leads or support requests.
- Human admins receive better context before taking over.

For the provider/operator:

- Recurring monthly revenue per managed agent.
- Upsell path from Basic to Pro to Custom workflows.
- Reusable playbook and runtime pattern across multiple clients and channels.

## Terminology

| Term | Meaning |
|---|---|
| Agent | The AI receptionist/sales/support assistant instance. |
| Client | The business that pays for the managed service. |
| Customer / end user | The person messaging the client. |
| Tenant | One isolated client configuration and data namespace. |
| Admin | Human operator who can take over conversations. |
| Provider / operator | The team delivering and maintaining the service. |
| Channel | WhatsApp, Instagram DM, Telegram, webchat, email, etc. |

## Scalability notes

- Phase 1: one managed runtime per client for simple operations.
- Phase 2: shared runtime with tenant isolation for better unit economics.
- Phase 3: self-serve onboarding with templates and review gates.
