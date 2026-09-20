# Spine Canvas / Swarm — Private Rebuild Technical Specification

Implementation-ready build spec for a single-user, privately-deployed clone on Next.js 14, Supabase, and Vercel with multi-provider model orchestration (OpenAI · Anthropic · Google · OpenRouter)
February 2025

**Document Type**
Technical build specification (implementation-ready)

**Target Deployment**
Single-user, private. Vercel (app + edge/serverless functions) + Supabase (Auth, Postgres, Storage, Realtime)

**Model Providers**
OpenAI, Anthropic, Google, OpenRouter (unified through the Vercel AI SDK provider abstraction)

**Evidence Base**
Consolidated research tables: platform schema findings [1] [5], per-block-type rebuild spec [2], integrations/MCP/canvas/tech-stack findings [3]

**Build Envelope**
4 phases / 15 weeks, single engineer, no multi-tenant billing surface

## Contents
1. Executive Summary
2. System Architecture
3. Postgres / Supabase Data Model
4. Block Type Specifications
5. Multi-Model Orchestration Layer
6. Swarm Mode Orchestrator
7. Deep Research Block
8. Infinite Canvas UI
9. Integrations and MCP Layer
10. Tech Stack and Environment Variables
11. 4-Phase Build Roadmap
12. Known Gaps and Confidence Ledger

(FULL CONTENT PENDING — see follow-up commit for complete document)
