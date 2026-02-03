# x402 Five Proxy — Next.js Gateway for x402 Payment Integration

**A production-ready Next.js proxy that implements the full x402 gateway integration flow, enabling payment-gated access to APIs and resources using on-chain USDC settlements on Solana.**

[![TypeScript](https://img.shields.io/badge/TypeScript-45%25-3178C6?style=flat-square)](#)
[![CSS](https://img.shields.io/badge/CSS-55%25-264653?style=flat-square)](#)
[![Next.js](https://img.shields.io/badge/Next.js-App%20Router-000000?style=flat-square)](#)
[![x402 Protocol](https://img.shields.io/badge/x402-Protocol-6366f1?style=flat-square)](https://x402.org)

---

## Table of Contents

- [Overview](#overview)
- [The Problem](#the-problem)
- [How x402 Gateway Integration Works](#how-x402-gateway-integration-works)
- [Architecture & Request Flow](#architecture--request-flow)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Installation & Setup](#installation--setup)
- [Configuration](#configuration)
- [Usage](#usage)
- [Use Cases](#use-cases)
- [Security Considerations](#security-considerations)
- [Roadmap & Extensibility](#roadmap--extensibility)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

x402 Five Proxy is a **Next.js-based x402 gateway integration** that acts as a payment-aware reverse proxy between incoming HTTP clients and your protected backend resources. It implements the five-step x402 protocol handshake natively — from the initial `402 Payment Required` challenge through on-chain verification and settlement — entirely within a single Next.js deployment.

The proxy layer (`proxy.ts`) handles all x402 payment middleware logic: parsing `PaymentRequirements`, validating incoming `X-PAYMENT` headers, coordinating with an upstream facilitator for verification and settlement, and forwarding authenticated requests to your origin. The `app/` directory delivers a modern, responsive frontend for interacting with payment-gated resources directly in the browser.

This makes x402 Five Proxy suitable as both a **standalone payment gateway** and a **composable middleware module** you can drop into any Next.js infrastructure stack.

---

## The Problem

Traditional API monetization relies on account registration, API key provisioning, credit-based billing, and manual payment reconciliation. This creates compounding friction — especially for programmatic consumers and AI agents that need to acquire resources autonomously at scale.

x402 Five Proxy solves this by replacing the entire account-and-key lifecycle with a single HTTP exchange. A client that wants access to a protected resource simply receives a `402` status with structured payment requirements, settles the payment on-chain in seconds, and retries the request with a cryptographic proof of payment. No accounts. No API keys. No chargeback risk.

For developers and teams building **Web3 payment integration** into their services, this proxy provides the gateway layer that makes that exchange operationally trivial to deploy and maintain.

---

## How x402 Gateway Integration Works

x402 Five Proxy implements the canonical x402 protocol flow as defined by the [x402 specification](https://x402.org). The "Five" in the project name refers to the five discrete stages of the payment handshake that this proxy orchestrates end-to-end:

**Stage 1 — Request.** A client sends a standard HTTP request to a resource endpoint routed through the proxy.

**Stage 2 — Challenge.** If no valid `X-PAYMENT` header is present, the proxy responds with `HTTP 402 Payment Required`. The response body contains a `PaymentRequirements` object that specifies the accepted payment scheme (e.g., `exact`), the target network (e.g., `base` or `base-sepolia`), the required amount in atomic units, the recipient wallet address (`payTo`), and the ERC-20 asset contract address (typically USDC).

**Stage 3 — Payment.** The client constructs a signed payment payload using EIP-3009 `transferWithAuthorization` and retries the request with the payload encoded in the `X-PAYMENT` header.

**Stage 4 — Verification.** The proxy forwards the payment payload and the original `PaymentRequirements` to the upstream facilitator's `/verify` endpoint. The facilitator validates the signature, checks allowance limits, and confirms the payment has not been replayed.

**Stage 5 — Settlement & Access.** Upon successful verification, the proxy calls the facilitator's `/settle` endpoint to execute the on-chain transfer. Once confirmed, the proxy forwards the original request to the protected resource and returns the response to the client, along with an `X-PAYMENT-RESPONSE` header containing the transaction hash and settlement details.

This five-stage flow is the core of x402 gateway integration — and it runs entirely within the proxy without requiring the downstream resource server to understand x402 at all.

---

## Architecture & Request Flow

```
┌──────────┐        ┌─────────────────────────────┐        ┌─────────────┐
│  Client  │        │     x402 Five Proxy         │        │  Facilitator│
│ (Browser │  HTTP  │  ┌─────────┐  ┌──────────┐  │  REST  │  Server     │
│  / Agent)│◄──────►│  │ Next.js │  │ proxy.ts │  │◄──────►│  /verify    │
└──────────┘        │  │  app/   │  │ Gateway  │  │        │  /settle    │
                    │  └─────────┘  └────┬─────┘  │        └─────────────┘
                    │                    │        │                │
                    │                    ▼        │                │ On-Chain
                    │          ┌──────────────┐   │                ▼
                    │          │  Origin API  │   │        ┌─────────────┐
                    │          │  (upstream)  │   │        │  Solana Dev │
                    │          └──────────────┘   │        │  (USDC)     │
                    └─────────────────────────────┘        └─────────────┘
```

**Component breakdown:**

`proxy.ts` is the x402 payment gateway middleware. It intercepts every inbound request, evaluates whether a valid payment proof is attached, and either issues a `402` challenge or proxies the request upstream after settlement confirmation.

`app/` is the Next.js App Router frontend. It provides a browser-native interface for viewing payment requirements, connecting a wallet, and interacting with payment-gated content — useful for human-facing UIs built on top of the same proxy.

`public/` holds static assets served by Next.js alongside the application.

The upstream **facilitator** (e.g., the Coinbase-hosted facilitator at `x402.org`, or a self-hosted instance) handles the cryptographic verification and on-chain settlement. The proxy never holds custody of funds; it acts purely as an orchestration layer.

---

## Project Structure

```
x402-Five-Proxy/
├── app/                  # Next.js App Router — pages and UI components
├── public/               # Static assets (icons, images, stylesheets)
├── proxy.ts              # x402 gateway middleware — core payment proxy logic
├── next.config.js        # Next.js configuration (rewrites, env, headers)
├── next-env.d.ts         # Next.js TypeScript environment declarations
├── package.json          # Dependencies and scripts
├── tsconfig.json         # TypeScript compiler options
└── .gitignore
```

---

## Prerequisites

- **Node.js** v18 or higher (v20+ recommended)
- **npm** or **pnpm** (pnpm preferred for monorepo-style dependency resolution)
- A wallet address on **Solana** (mainnet or devnet) to receive payments
- Access to a facilitator endpoint compatible with the x402 `/verify` and `/settle` REST interface

---

## Installation & Setup

Clone the repository and install dependencies:

```bash
git clone https://github.com/ElliteAnts/x402-Five-Proxy.git
cd x402-Five-Proxy
npm install
```

Create a `.env` file in the project root and populate it with your configuration values (see [Configuration](#configuration) below):

Start the development server:

```bash
npm run dev
```

The proxy will be available at `http://localhost:3000` by default.

---
FACILITATOR_URL=https://x402.org/facilitator
RESOURCE_EVM_ADDRESS=
RESOURCE_SVM_ADDRESS=4uJdBto9HJytdHuaoavx7XKS3GU5ejSqxb7QGskjrQcN
FACILITATOR_EVM_PRIVATE_KEY=
FACILITATOR_SVM_PRIVATE_KEY
## Configuration

x402 Five Proxy is driven by environment variables. Set the following in `.env.local`:

| Variable | Description | Example |
|---|---|---|
| `FACILITATOR_URL` | Base URL of your x402 facilitator instance | `https://facilitator.x402.org` |
| `RESOURCE_SVM_ADDRESS` | Destination Solana address | `4uJd...rQcN` |

## Use Cases

**API monetization without accounts.** Turn any existing REST or GraphQL endpoint into a pay-per-request service. Clients pay exactly what they consume — no subscriptions, no credit top-ups.

**AI agent commerce.** AI agents operating autonomously can discover payment-gated resources, settle micropayments on-chain, and acquire data or compute without human intervention. x402 Five Proxy is purpose-built for this agentic flow.

**Content paywalls.** Serve premium articles, datasets, or media behind a per-access payment wall. The Next.js frontend layer makes it straightforward to build a polished reader experience alongside the payment logic.

**Internal service meshes.** Use the proxy as an internal gateway to enforce payment or cost-allocation policies between microservices within a single organization — useful for chargeback and usage tracking in multi-tenant architectures.

**Prototyping and experimentation.** The single-deployment Next.js model means you can stand up a fully functional x402 payment gateway in minutes, with no infrastructure dependencies beyond a facilitator endpoint. Ideal for hackathons, MVPs, and proof-of-concept work.

## Roadmap & Extensibility

x402 Five Proxy is designed to grow alongside the x402 ecosystem. Planned and community-driven extensions include:

- **Multi-scheme support** — extending beyond `exact` to `upto` (consumption-based) and other emerging x402 schemes as they are ratified.
- **Multi-chain routing** — adding support for Solana, Polygon, and other networks supported by the x402 facilitator layer.
- **Route-level pricing configuration** — declarative, per-route payment requirements defined in `next.config.js` or a dedicated config file, without custom middleware code.
- **Analytics dashboard** — a built-in admin view (within the `app/` layer) that surfaces settlement history, revenue, and per-route usage metrics.
- **Webhook settlement notifications** — outbound webhooks that fire on successful payment settlement, enabling event-driven downstream workflows.

---

## Contact

[Telegram](https://t.me/WebFiveFingers)

---

## References

- [x402 Protocol Specification — x402.org](https://x402.org)
- [coinbase/x402 — Reference Implementation](https://github.com/coinbase/x402)
- [EIP-3009 — transferWithAuthorization](https://eips.ethereum.org/EIPS/eip-3009)