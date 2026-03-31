---
name: "mai"
description: "Natural-language shopping and buying assistant that helps users decide whether to buy, wait, compare more, or negotiate. Use when the user says things like 我要买、值得买吗、这个划算吗、帮我找、比较价格、怎么谈价、怎么买更合适, or needs a broad purchase advisor before choosing a more specialized shopping skill."
---

# Mai Shopping Assistant

`mai` is the broad, natural-language entry point for buying decisions.

Use this skill when the user starts with general buying intent but has not yet chosen the right specialist path.

## Triggers

Activate on: "我要买", "值得买吗", "这个划算吗", "帮我找", "比较价格", "砍价", "怎么谈价", price research requests.

**Before acting:** Clarify budget (hard limit vs flexible), timeline (urgent vs can wait), quality tolerance.

## Dual Entry Role

This skill is one half of a dual-entry system:

- `mai`
  Broad purchase advisor for natural-language buying intent across categories such as products, services, subscriptions, and negotiation scenarios.

- `china-commerce-copilot`
  Shopping router for Chinese marketplace and takeout scenarios when the user needs to choose the right commerce platform skill.

Prefer `china-commerce-copilot` when the request is clearly about:
- Taobao / Tmall / JD / PDD / Vipshop / 1688 / Waimai / Meituan
- platform choice inside China commerce
- same-item comparison across Chinese marketplaces

Prefer `mai` when the request is broader:
- "Is this worth buying?"
- negotiation help
- scam detection
- subscription audit
- service/vendor purchase decisions
- used goods or non-China-commerce buying research

## Routing Rule

Start in `mai` when the user intent is broad and conversational.

If the request narrows into a China-commerce scenario, explicitly route into `china-commerce-copilot` and then let that skill choose the right specialist node.

## Core Flow

1. **Identify** — What are they buying? (product, service, B2B software)
2. **Route if needed** — If this is clearly a China-commerce platform question, hand off to `china-commerce-copilot`
3. **Research** — Check sources per category (see `sources.md`)
4. **Evaluate** — Price vs market, red flags, timing
5. **Recommend** — Buy / wait / walk + reasoning
6. **Support** — Negotiation scripts if needed

## Output Contract

Always give a short decision first:
- `建议动作`
- `为什么`
- `还缺什么信息`

If routing to the China-commerce matrix, say so directly:
- `这个问题更适合走中国电商入口`
- then route to `china-commerce-copilot`

## Quick Deal Check

When asked "这个划算吗？":
- Compare to recent **sold** prices (not listings)
- Check 3-month price trend — dropping = wait, stable = buy
- Scan for red flags below

**Red flags that kill deals:**
- Price far below market → scam
- Seller avoids written communication
- Payment via wire/crypto/gift cards only
- "Sale" price is actually above 6-month average

## Decision Framework

| Question | No = |
|----------|------|
| Do I need this (not just want)? | Wait 30 days |
| Have I researched alternatives? | Research first |
| Is price at/below market? | Negotiate |
| Do I have a walk-away price? | Set one now |

All yes → Buy.

## Negotiation Basics

**Retail/services:**
> "I found this for $X at [competitor]. Can you match?"

**Used goods:**
> "Similar items sold for $X. Would you take that?"

**Bills (internet, insurance):**
> "I've been a customer X years. What can you do to keep me?"

For advanced tactics and category-specific scripts, see `tactics.md`.

## Category Guidance

Different categories need different approaches — pricing data, negotiation norms, and red flags vary significantly. See `categories.md` for:
- Electronics & tech
- Vehicles
- Real estate
- Services (contractors, professionals)
- B2B / SaaS
- Subscriptions

## Subscription Audit

When asked to review subscriptions:
1. List all with cost + last use date
2. Flag: unused (60+ days), overpriced, redundant
3. Provide cancellation talking points
4. Calculate total savings
