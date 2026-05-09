# Mai CLI Design Spec

## Goal

Build `mai-cli`, an independent AI marketplace agent runtime for local commerce. Buyers interact with buyer agents, merchants run merchant agents, and both sides communicate through a marketplace service without requiring OpenClaw or Hermes.

The first product wedge is nearby local-life commerce: tea shops, restaurants, flowers, gifts, and other same-city merchants where real-time consultation, stock checks, delivery feasibility, and human confirmation matter more than pure SKU search.

## Chosen Approach

Use an independent platform that remains compatible with the existing Mai core.

- Create a new standalone runtime named `mai-cli`.
- Keep OpenClaw and Hermes as optional adapters, not required hosts.
- Reuse the current Mai catalog, conversation, order, risk, and marketplace concepts.
- Upgrade the current JSON/HTTP registry shape into a service-oriented API, database-backed state, and explicit buyer/merchant agent runtimes.

This avoids a full rewrite while removing the main limitation of the current package: it can only behave as a skill or plugin inside another agent host.

## Product Boundary

`mai-cli` is a two-sided AI marketplace system:

- Buyer agents help buyers express needs, search nearby merchants, compare options, ask follow-up questions, and request order drafts.
- Merchant agents help merchants answer product, stock, delivery, and price questions using shop data and merchant-defined rules.
- The marketplace API stores trusted state: merchants, products, conversations, order drafts, order confirmations, moderation decisions, and external payment evidence.
- Merchant agents may create order drafts, but merchant confirmation is required before inventory is reserved and an order becomes confirmed.

Core principle: AI can consult, compare, negotiate within rules, and draft orders; final merchant commitments require merchant confirmation.

## MVP Flow

1. A merchant creates a local shop profile with city, service area, contact details, hours, and delivery rules.
2. The merchant publishes products with price, stock, category, tags, and delivery attributes.
3. The merchant starts a merchant agent.
4. A buyer starts a buyer agent and asks for a nearby product or service.
5. The buyer agent searches marketplace inventory and merchants.
6. The buyer agent opens one or more conversations with merchant agents.
7. Merchant agents answer stock, price, delivery range, estimated timing, and substitution questions.
8. The buyer agent summarizes options and risks for the buyer.
9. The buyer chooses one option and asks to proceed.
10. The merchant agent creates an order draft.
11. The merchant reviews and confirms the order draft through CLI or console.
12. The marketplace changes the order to `confirmed` and reserves inventory.

Example demo:

```text
Buyer: 我在西湖附近，今天想买两盒龙井礼盒，能送吗？
Buyer Agent: searches nearby tea merchants and asks the selected merchant agent.
Merchant Agent: checks stock=5 and 5km delivery rules, then replies with price and ETA.
Buyer Agent: summarizes the option.
Buyer: 确认想买。
Merchant Agent: creates an order draft.
Merchant: confirms the draft.
System: stock changes from 5 to 3 and order status becomes confirmed.
```

## Architecture

`mai-cli` should be a standalone Python project with five main modules.

```text
Buyer <-> Buyer Agent <-> Marketplace API <-> Merchant Agent <-> Merchant Console
                              |
                              v
                         Database / Events
```

### Marketplace API

The API is the trusted state boundary. It owns:

- merchant profiles
- products and inventory
- conversations and messages
- agent heartbeat records
- order drafts and confirmed orders
- payment status evidence
- moderation and risk flags

Agents call this API through typed tools. They do not mutate trusted state directly.

### Buyer Agent Runtime

The buyer runtime handles buyer-facing conversation.

Responsibilities:

- parse buyer needs into structured search criteria
- search nearby merchants and products
- compare candidate products and merchants
- open merchant conversations
- ask stock, delivery, price, and substitution questions
- summarize options, warnings, and next actions
- request order draft creation when the buyer confirms intent

### Merchant Agent Runtime

The merchant runtime handles merchant-side automation.

Responsibilities:

- answer public product questions
- check stock and delivery feasibility
- apply public price and delivery rules
- suggest substitutions when stock is low
- create order drafts
- flag conversations for human review

Human review is required for bargaining, low stock, unclear delivery area, high order value, unusual refund promises, unsupported products, or low model confidence.

### Merchant Console

The first version can be CLI-first, with a web console later. It must allow merchants to:

- create and edit the shop profile
- manage products and stock
- configure delivery rules
- configure automation limits
- review conversations
- confirm or reject order drafts

### Operator Console

The first version can expose operator actions through CLI/API. It should support:

- product moderation queue
- suspicious conversation review
- payment status evidence review
- dispute notes
- agent health monitoring

## Mai Agent Protocol

The first internal protocol should be explicit and typed, but not overdesigned as a public standard.

### Agent

```text
id
type: buyer | merchant
owner_id
status: online | away | human_required
capabilities
last_seen_at
```

### Conversation

```text
id
buyer_id
merchant_id
sku optional
status: open | waiting_buyer | waiting_merchant | draft_order_created | closed
messages
```

### Message

```text
sender: buyer | buyer_agent | merchant_agent | merchant | operator
intent: ask_product | ask_stock | ask_delivery | negotiate | create_order_draft | support
text
structured_payload optional
created_at
```

### Order Draft

```text
buyer_id
merchant_id
items
quantity
delivery_address_area
requested_time
quoted_price
delivery_fee
merchant_confirmation_required: true
status: draft
```

## Order Model

`mai-cli` should preserve the existing Mai order state discipline.

Recommended statuses:

```text
draft -> confirmed -> payment_pending -> paid_external -> fulfilled -> completed
```

Dispute path:

```text
disputed -> resolved | refunded | cancelled
```

Rules:

- Creating an order draft does not reserve stock.
- Merchant confirmation reserves stock.
- Cancellation or refund releases reserved stock.
- The agent must not claim payment, escrow, refund, or delivery success without external evidence.
- Live money movement must go through a licensed payment service provider; `mai-cli` records PSP references and status only.

## Technical Stack

Use Python first because the existing Mai implementation is already Python and contains useful business logic.

Recommended stack:

```text
Python 3.11+
FastAPI for Marketplace API and webhooks
Typer or argparse for CLI entry points
SQLite for local development and demos
Postgres for hosted deployment
SQLAlchemy for persistence and migrations
Pydantic for typed protocol/tool schemas
APScheduler or a small worker loop for agent polling
OpenAI-compatible LLM provider abstraction
```

## Project Shape

```text
mai-cli/
  pyproject.toml
  README.md
  docs/
    architecture.md
    agent-protocol.md
    migration-from-mai.md
  mai_cli/
    cli.py
    config.py
    main.py
    core/
      catalog.py
      orders.py
      conversations.py
      risk.py
      pricing.py
      delivery.py
    api/
      app.py
      routes_merchants.py
      routes_marketplace.py
      routes_conversations.py
      routes_orders.py
      routes_agents.py
    agents/
      base.py
      buyer_agent.py
      merchant_agent.py
      tools.py
      prompts.py
      runtime.py
    db/
      models.py
      session.py
      migrations/
    integrations/
      llm.py
      payments.py
      notifications.py
    adapters/
      mai_legacy.py
      openclaw.py
      hermes.py
  tests/
    test_buyer_agent.py
    test_merchant_agent.py
    test_marketplace_flow.py
    test_order_drafts.py
```

## Migration From Existing Mai

Move or adapt these current pieces:

- `search_products`, `product_summary`, and merchant rating logic into `core/catalog.py`.
- Order state and inventory reservation logic into `core/orders.py`.
- Registry risk scoring and moderation into `core/risk.py`.
- Message and conversation concepts into `core/conversations.py`.
- Registry endpoints into FastAPI route modules.
- Existing CLI smoke workflows into end-to-end tests.

Upgrade these pieces:

- JSON file store to SQL database.
- Dependency-free HTTP server to FastAPI.
- API-key-only auth to user, merchant, and agent tokens.
- Host-specific skill prompts to reusable agent prompts.
- CLI subcommands to typed runtime tools.
- Pull-only merchant inbox to worker-driven merchant agent processing.

Keep these compatibility surfaces:

- `adapters/mai_legacy.py` imports existing `mai.json` files.
- `adapters/openclaw.py` and `adapters/hermes.py` expose optional bridges later.
- Existing Mai registry data can be migrated through a script instead of abandoned.

## MVP Features

### Merchant

- Create merchant profile.
- Create and update products.
- Adjust inventory.
- Configure delivery area, delivery fee, and estimated delivery time.
- Configure automation boundaries.
- Start merchant agent runtime.
- View conversations.
- Confirm or reject order drafts.

### Buyer

- Start buyer agent runtime.
- Express demand in natural language.
- Search nearby merchants and products.
- Compare one to three candidates.
- Ask merchant agents about stock, delivery, price, and substitutions.
- Request an order draft.

### Marketplace

- Merchant and product CRUD.
- Product and merchant search.
- Conversation and message APIs.
- Agent heartbeat APIs.
- Order draft create/list/confirm APIs.
- Basic risk and moderation flags.
- External payment status recording only.

## Explicit Non-Goals For MVP

- Automatic formal order confirmation by agents.
- Real payment charging or fund custody.
- Full KYC/KYB and tax handling.
- Advertising or bidding.
- Multi-merchant automatic auctioning.
- Automatic refunds.
- Courier or rider dispatch.
- Large-scale recommendation ranking.
- Native mobile apps.

## Acceptance Criteria

- `mai-cli` can run without OpenClaw or Hermes.
- A buyer agent and merchant agent can run as independent processes.
- A merchant can create a shop, product, inventory, and delivery rule.
- A buyer can ask for a nearby product in natural language.
- The buyer agent can search the marketplace and contact a merchant agent.
- The merchant agent can answer stock, price, and delivery questions from structured shop data.
- The buyer agent can summarize options and request an order draft.
- The merchant agent can create an order draft but cannot confirm it automatically.
- A merchant can confirm an order draft through CLI or console.
- Confirming the order reserves inventory.
- Risky or unsupported situations are marked for human review.
- Payment state is recorded only as external/PSP evidence.
- Existing Mai data can be imported through a legacy adapter.

## First Implementation Milestone

The first milestone should build the smallest runnable vertical slice:

1. Project scaffold and CLI entry point.
2. SQLite-backed data models for merchants, products, conversations, messages, agents, and order drafts.
3. FastAPI marketplace service.
4. Deterministic merchant agent tools for catalog, inventory, delivery, and order draft creation.
5. Simple buyer agent tools for search, merchant conversation, and order draft request.
6. End-to-end demo test for the Hangzhou Longjing gift box scenario.

Do not implement full hosted production deployment until the local vertical slice is reliable.
