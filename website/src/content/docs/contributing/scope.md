---
title: Scope
description: What belongs in this repo, what doesn't, and the borderline rule.
---

This repo teaches **building on Cardano**. It is a generic knowledge base for
AI coding agents and developers.

## In scope

- SDKs, frameworks, validator libraries, design patterns, language tooling
- Infrastructure (nodes, indexers, chain providers)
- Protocol and standard specs (CIPs, ledger specs)
- Reference implementations of *patterns* (e.g. multisig oracle contracts as
  a pattern, not as a product manual)
- Generic dApp categories: DEX, lending, NFT marketplace, oracle consumer,
  governance tool

## Out of scope

- Product docs for specific deployed dApps. These belong on each project's
  own site; users who need them can ask their agent to search the web.
- Closed-source content
- Marketing material

## Borderline rule

If the upstream repo's primary purpose is *"use OUR product"*, it's out.
If it's *"here's how X pattern works, here's the reference code"*, it's in.

## Skill content

The same rule applies to skills:

- Teach categories generically (*"how to write a vesting validator"*), not
  product mechanics (*"how to use Product X's deposit endpoint"*).
- A skill is a neutral community contribution. No branded promotion, no
  grant/treasury context, no proposal framing.

If you're unsure, open a discussion before writing code.
