# NextMission Navigator · VetNavi.AI

> AI-Powered Veteran Transition Platform · Capstone Project [Indy-1] · AI 7993 Spring 2026

https://msmetamorphosis.github.io/nextmission-capstone/

**Live Deployment:** [vetnavi.ai](https://vetnavi.ai)  
**Developer:** Crystal Tubbs · KSU MSAI · Metamorphic Curations LLC  
**Advisor:** Dr. Arthur Choi · Kennesaw State University CCSE  

---

## Overview

NextMission Navigator is a full-stack, production-grade web application that helps U.S. military veterans navigate the transition to civilian life. Over 200,000 service members separate from the military each year into a fragmented, overwhelming landscape of benefits, career resources, housing options, and healthcare services. This platform bridges that gap using AI-driven action planning grounded in verified regional resource data.

Users describe their goals in plain language. The platform classifies that goal, retrieves relevant regional resources, and generates a structured, prioritized, step-by-step action plan via the Anthropic Claude API. A dual-interface design combines a persistent text chatbot with an ElevenLabs voice agent for 24/7 accessible guidance.

This is a complete from-scratch rebuild (v3), replacing a prior low-code prototype with a production TypeScript implementation across 200+ hours and 5 development milestones (January through April 2026).

---

## Key Features

- **AI Action Plan Generation** — Personalized, schema-validated plans tailored to individual goals (career, education, housing, healthcare, disability claims, financial planning, general transition)
- **RAG Architecture** — Regional resource hints are injected directly into the LLM prompt, grounding outputs in verified data and eliminating hallucinated resource recommendations
- **Zod Schema Validation** — Both request inputs and LLM JSON outputs are validated; malformed responses trigger graceful fallback to mock plans, so users always receive guidance
- **Goal Classification Engine** — Keyword-based classifier routes each goal into one of 7 categories, providing focused generation context to the model
- **Text Chatbot** — Floating chat widget with full conversation history, voice input (Web Speech API), and a 180-word response cap enforced by the system prompt
- **ElevenLabs Voice Agent** — Embedded ConvAI web component providing a separate spoken-language interaction modality
- **Regional Resource Catalog** — 20+ manually verified resources across 6 categories with regional matching for national, state (TX, FL, GA, TN), and metro-level (Tampa) coverage
- **Graceful Degradation** — API key absence or Claude API failure falls back to contextually appropriate mock plans; voice service unavailability does not impact core plan generation
- **Veterans Crisis Line Integration** — 988 prominently displayed on all pages; chat system prompt mandates crisis line surfacing for urgent mental health queries

---

## Technology Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 15.5 (App Router), React 19, TypeScript 5.9 |
| Styling | Tailwind CSS 3.4 + custom CSS variable military design system |
| AI / LLM | Anthropic Claude API (`claude-3-5-sonnet-20241022`, temp 0.7, max_tokens 3000) |
| Schema Validation | Zod 3.25 (input + LLM output normalization) |
| Voice AI | ElevenLabs ConvAI web component |
| Icons | Lucide React 0.468 |
| Deployment | Vercel (serverless edge functions, global CDN) |
| Build Tools | PostCSS, Autoprefixer, ESLint |

---

## Architecture

The platform uses a layered, stateless serverless architecture deployed on Vercel. There is no session storage or database in v1.0; all context is passed per-request, enabling automatic horizontal scaling.

### System Components

**Frontend (Next.js 15 App Router)**  
Pages: Home, Resources, Veterans, About, Contact. Military-themed design system using CSS variables (`globals.css`): deep army green `#333C2C`, sandstone beige `#C2BA9A`, coyote tan `#A37E4C`, muted gold `#BCA55A`. Card hover-lift effects, gradient backgrounds, military texture overlays. WCAG 2.1 AA contrast compliance.

**Backend API Layer**  
Two Next.js API routes handle all AI interaction:
- `POST /api/action-plan` — Validates request body (goal: min 5 / max 8000 chars, optional userContext), runs goal classification, resource injection, and LLM generation
- `POST /api/chat` — Validates message history (1–30 messages, each max 12,000 chars, roles `user`/`assistant`), returns a single chat reply

**Intelligence Engine (4 server-side TypeScript modules)**  
- `anthropic.ts` — Minimal fetch-based wrapper around the Anthropic Messages REST API with AbortSignal timeout control. API key is read from `process.env` at runtime and never bundled into client code.
- `goalClassify.ts` — Classifies user goals into 7 categories via keyword matching: Housing, Disability Benefits, Healthcare, Education, Career, Financial, General Transition. Returns a label, matched keywords, and a planner blurb injected into the LLM prompt.
- `generateActionPlan.ts` — Orchestration layer. Builds the full prompt combining goal text, classification, veteran context (branch, years served, location, target industry), and up to 4 resource hints. Calls Claude with a 55-second timeout. Parses and normalizes the JSON response via Zod. Falls back to `mockPlans.ts` on any failure.
- `actionPlanOutput.ts` — Defines the `ActionPlanResponse` Zod schema: `why_this_plan` (optional), `categories` (array of named step groups), `follow_up` (string). Each step has `title`, `description`, optional `link` (markdown), `timeframe`, `priority` (high/medium/low), and `additionalInfo`. Includes `normalizePlan()` for coercing priorities and `extractJsonObject()` for stripping markdown fences from LLM output.

**Knowledge Curation Layer**  
`src/data/resources/catalog.ts` — 20+ manually verified resources across 6 categories: Career Transition, Education Benefits, Housing Assistance, Healthcare & Wellness, Financial Support, Community & Support. Each resource includes name, URL, description, type (Government/Organization/Program), keyword tags, and regional codes (ALL, TN, GA, TX, FL, TAMPA).

`resourceMatch.ts` — Region-aware scoring engine. Normalizes user location strings to region tokens (e.g., `macdill` → FL/FLORIDA/TAMPA), scores resources by keyword overlap with goal text, boosts regional over national matches, deduplicates by URL, and returns top 4 hints.

**Voice Module**  
ElevenLabs ConvAI web component dynamically loaded on mount from unpkg CDN (duplicate injection prevention). Rendered as a fixed CSS overlay, independent of the text chatbot.

### Action Plan Request Lifecycle

1. User submits goal + optional context via `ActionPlanGenerator`
2. Frontend POSTs to `/api/action-plan` with Zod-validated body
3. `goalClassify.ts` maps goal to one of 7 categories
4. `resourceMatch.ts` scores catalog resources against goal and location; top 4 injected as RAG hints
5. `generateActionPlan.ts` constructs full structured prompt
6. Anthropic Messages API called with 55-second timeout
7. Response JSON extracted, normalized via `actionPlanOutput.ts` Zod schema
8. `attachCatalogNotes()` appends verified links to the first step
9. Validated `ActionPlanResponse` returned to frontend and rendered as categorized step cards

---

## Frontend Components

**`ActionPlanGenerator.jsx`**  
Primary interactive component. Manages goal text, four optional context fields (branch, years of service, location, target industry), loading state, and plan display. Five example prompts for onboarding. Parses markdown links from `step.link` fields via regex helper. Implements a follow-up refinement loop: user additions are appended to the original goal and the plan is regenerated. Steps render as categorized card groups with priority color-coding and timeframe display.

**`Chatbot.jsx`**  
Floating chat widget. Sends full message history to `/api/chat` on each turn. Integrates browser Web Speech API for voice input with graceful fallback. Auto-scrolls to latest message. Shows animated typing indicator. System prompt enforces 180-word response cap and mandates crisis line surfacing for urgent mental health queries.

**`ElevenLabsWidget.jsx`**  
Dynamically loads ElevenLabs ConvAI web component on mount. Fixed-position overlay separate from text chatbot, providing two distinct voice and text interaction modalities.

**`ResourcesAccordions.tsx`**  
Accordion-based resource directory with 6 category sections, distinct category background colors, two-column card grids with type badges, descriptions, and external links. Veterans Crisis Line in a prominent red alert banner at the top.

---

## Prompt Engineering

**Action Plan System Prompt** (`src/prompts/actionPlan.ts`)  
Enforces strict JSON-only output (no markdown fences), defines the response schema inline, requires 3–5 steps across categories, mandates HTTPS markdown links, and requires a follow-up question. Prompt is intentionally minimal to maximize schema adherence.

**Chat System Prompt** (`src/prompts/chat.ts`)  
Enforces 180-word response cap. Mandates Veterans Crisis Line (988) inclusion for any urgent or mental health-adjacent queries.

---

## Development Milestones

| Milestone | Date | Scope |
|---|---|---|
| M1 | Feb 7, 2026 | Repo setup, Next.js 15 scaffold, Tailwind military design system |
| M2 | Feb 28, 2026 | Anthropic wrapper, Zod validation, `/api/action-plan`, goal classifier |
| M3 | Mar 15, 2026 | Resource catalog, regional matching, RAG injection, ElevenLabs widget, `/api/chat` |
| M4 | Mar 30, 2026 | `ActionPlanGenerator` UI with follow-up loop, Chatbot with voice, ResourcesAccordions, Veterans and About pages |
| M5 | Apr 15, 2026 | Vercel production deployment, UI polish, mock plan fallbacks, crisis line integration |

Total: 200+ hours across 5 milestones.

---

## Version Control

Branch strategy: `main` (stable production), `dev` (integration), `feature/*` (per-feature), `bugfix/*` (hotfixes).

Commit convention: `<type>: <description>` (e.g., `feat: add Zod schema validation to action-plan route`).

Semantic version tags: `v1.0-alpha` (prototype), `v1.0-beta` (feature complete), `v1.0-final` (C-Day submission).

---

## Non-Functional Requirements

| Requirement | Implementation |
|---|---|
| Performance | Plan generation target under 15 seconds; stateless API enables horizontal scaling |
| Security | All API keys server-side only (`process.env`); HTTPS enforced |
| Safety | 988 Veterans Crisis Line on all pages; AI output disclaimers throughout |
| Accessibility | WCAG 2.1 AA contrast; voice AI; clear non-technical error messaging |
| Reliability | Mock plan fallbacks ensure users always receive a response even during API failures |

---

## Repository Notice

This repository is **private**. Source code at `github.com/Msmetamorphosis` represents potentially protected intellectual property developed under Metamorphic Curations LLC. Full implementation details are documented in the Final Report linked below.

---

## Project Documents

- [Final Report (PDF)](https://msmetamorphosis.github.io/nextmission-capstone/Indy-1-NextMission-FinalReport.pdf)
- [Software Requirements Specification](https://msmetamorphosis.github.io/nextmission-capstone/Indy-1-NextMission%20Navigator-Requirements.pdf)
- [Software Design Document](https://msmetamorphosis.github.io/nextmission-capstone/Indy-1-NextMission%20Navigator-Design.pdf)
- [Project Plan & Gantt Chart](https://msmetamorphosis.github.io/nextmission-capstone/Indy-1-NextMissionNavigator-GanttChart-Estimate%20(1).pdf)
- [Capstone Project Site](https://msmetamorphosis.github.io/nextmission-capstone/)
- [Narrated Demo & Presentation](https://youtu.be/k1pcMwI1snM)

---

## Target Users

- U.S. military veterans transitioning from active duty to civilian life
- Transitioning service members approaching separation
- Military spouses assisting with transition planning
- Workforce support coordinators using the platform as a guidance tool

---

*[Indy-1] NextMission Navigator · AI 7993 · KSU CCSE · Spring 2026*  
*© 2026 Metamorphic Curations LLC*
