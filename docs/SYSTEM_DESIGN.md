# Nox Metals Knowledge Base — System Design Document

**Author:** Anirudh K. — MS CS, University of Michigan
**Date:** June 11, 2026
**Status:** Proposal (V1 MVP + Ambitious Version)
**Deployment target:** Fully local / on-prem (vLLM-served open-weight models)

---

## 0. Executive Summary

Nox Metals is scaling from a 4-person core team to 45+ people while operating highly automated metal factories. The scaling bottleneck is not headcount — it is that the company's most valuable knowledge lives at the **seam between vendor hardware and Nox's proprietary orchestration software**, and that seam currently exists only in four people's heads. Every new hire converts that knowledge gap into founder interrupts.

This document proposes an on-prem, citation-first knowledge and onboarding system:

- **V1 (MVP):** A single-GPU, vLLM-served RAG system with a three-tier memory architecture, a deterministic query router that answers ~30–40% of queries without touching the GPU, KV-cache-resident static context for fast Time-to-First-Token (TTFT), and full Langfuse tracing from day one.
- **Ambitious Version:** A multi-GPU platform with a larger tensor-parallel model, prefix-cache-aware request routing across replicas, multimodal ingestion (wiring diagrams, floor photos), voice access for hands-busy operators, and agentic knowledge-capture loops that turn Slack threads and resolved tickets into reviewed KB articles.

Everything runs locally. Process recipes, machine configurations, and orchestration code are Nox's moat; none of it leaves the building.

---

## 1. The Core Problem

### 1.1 What actually breaks when a hard-tech company goes 4 → 45

A generic chatbot solves "search is annoying." That is not Nox's problem. Nox's problem is specific to automated manufacturing:

**The knowledge that matters most is interface knowledge.** A vendor manual tells you what fault `E-217` means on the press brake. The git history tells you what the orchestration service does when it receives that fault. But *only a founder knows* that `E-217` fires spuriously when the hydraulic oil is below 18°C, that the correct response is a warm-up cycle rather than the vendor's prescribed reset, and that the software has a hardcoded retry that masks the first two occurrences. That three-way intersection — vendor hardware quirk × physical reality × proprietary software behavior — is exactly the knowledge that:

1. Is written down **nowhere** (it spans three documentation domains that don't reference each other),
2. Is the **highest-value** knowledge in the company (it *is* the automation moat),
3. Generates the most **founder interrupts** as new hires arrive.

### 1.2 The three failure modes to design against


| Failure mode              | Mechanism                                                                                                                                                        | Cost                                                                                                                     |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| **Founder interrupt tax** | Every question a new hire can't answer becomes a Slack ping or shoulder tap to one of 4 people                                                                   | Founder throughput collapses precisely when the company needs them on product; answers are given verbally and lost again |
| **Onboarding latency**    | New controls engineer or operator can't safely touch equipment until they've absorbed unwritten context                                                          | Weeks of shadowing per hire × 40 hires; physical equipment means mistakes are expensive and dangerous                    |
| **Knowledge divergence**  | Without a canonical source, hires re-derive solutions and create conflicting local versions ("Site B does the anneal cycle differently because someone guessed") | Process drift in a business whose entire pitch is repeatable automated production                                        |


### 1.3 The safety constraint that shapes everything

A metals factory is an OSHA-regulated environment: lockout/tagout (LOTO), hot work, crane operations, press safety. This imposes a hard design constraint that generic RAG systems ignore:

> **A wrong answer about a safety procedure is not a quality problem — it is a liability and injury risk. The system must be citation-first and refusal-aware: every safety-relevant answer must quote and link its source document verbatim, and the system must refuse to synthesize safety procedures it cannot ground.**

This constraint is why the design below leans hard on deterministic retrieval, structured lookups, and faithfulness evaluation rather than maximizing "answer rate."

### 1.4 What Nox actually needs (design center)

A **machine-aware troubleshooting and onboarding assistant** that:

- Answers "what does this fault mean *on our line, with our software*" — not just what the vendor manual says
- Gets a new hire to their first safe, unsupervised task measurably faster
- Captures founder answers *once* and makes them canonical
- Treats safety content as a separate, stricter class of knowledge

Generic chat over documents is the baseline it must beat, not the goal.

---

## 2. Players


| Player                              | Role in the system                                                                          | Primary interaction                                          |
| ----------------------------------- | ------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| **Founders / core 4**               | Knowledge source & review authority. Approve promoted knowledge, curate the golden eval set | Review queue (~30 min/week each); "answer once" capture flow |
| **Software engineers** (new hires)  | Consumers of architecture docs, ADRs, code context, hardware-interface behavior             | Slack bot + web UI, IDE-adjacent                             |
| **Controls / electrical engineers** | Consumers of PLC docs, alarm tables, wiring context, vendor manuals × software behavior     | Slack bot + web UI                                           |
| **Operators / technicians**         | Consumers of SOPs, fault lookups, safety procedures. Hands-busy, on the floor               | Floor terminal / tablet; (Ambitious: voice)                  |
| **Ops & safety lead**               | Owns the safety document class; gatekeeps anything tagged `safety_critical`                 | Review queue with veto                                       |
| **KB admin** (rotating eng role)    | Owns ingestion health, index freshness, eval dashboards                                     | Grafana + Langfuse                                           |


---

## 3. The Pre-LLM Knowledge Base

The LLM is the *last* component to add. The knowledge base must be valuable as a search and lookup system before a single token is generated — if retrieval is bad, no model fixes it.

### 3.1 Data sources to ingest


| Source                                                               | Format                   | Volume (est.)                | Update cadence        | Notes                                                                                 |
| -------------------------------------------------------------------- | ------------------------ | ---------------------------- | --------------------- | ------------------------------------------------------------------------------------- |
| Vendor equipment manuals                                             | PDF, many scanned        | 50–200 docs, 100–800 pp each | Rare                  | OCR required; tables (fault codes, specs) are the high-value content                  |
| PLC alarm/fault code tables                                          | CSV exports, manual PDFs | Thousands of rows            | Per firmware rev      | **Extract to structured store, not vectors**                                          |
| Internal SOPs & safety docs (LOTO, hot work, press ops)              | Docs/PDF                 | 30–100 docs                  | Versioned, controlled | Tagged `safety_critical`; verbatim-quote-only class                                   |
| Proprietary software: repos, READMEs, ADRs, API docs                 | Git                      | Continuous                   | Per merge             | Ingest docs + docstrings + ADRs; full code search is a later phase                    |
| Slack/chat history                                                   | Export API               | Years of threads             | Continuous            | The richest source of tribal knowledge; noisy — needs distillation, not raw embedding |
| Maintenance logs / CMMS tickets                                      | DB or CSV                | Continuous                   | Continuous            | Resolved tickets = troubleshooting case studies                                       |
| Incident postmortems & runbooks                                      | Docs                     | Dozens                       | Per incident          | High value per token                                                                  |
| Onboarding checklists & training material                            | Docs                     | Small                        | Quarterly             | Seeds the role-tier memory                                                            |
| Equipment registry (machine ID, model, firmware, site, install date) | Spreadsheet → DB         | ~100s rows                   | Per change            | Becomes the **metadata spine** — every chunk links to a machine where applicable      |


Deliberately out of V1 scope: CAD models, raw sensor/telemetry streams, video. (Telemetry queries appear in the Ambitious Version.)

### 3.2 The key ingestion insight: don't vectorize what you can SELECT

A large fraction of factory knowledge is **tabular and exact**: fault codes, torque specs, part numbers, firmware compatibility matrices. Embedding these into prose chunks destroys their precision and invites hallucinated numbers. Instead:

- Alarm/fault tables, spec tables, and the equipment registry are **extracted into Postgres** during ingestion.
- Queries that match these patterns (see §5, Deterministic Bypasses) are answered by **SQL lookup with the source row cited** — zero LLM involvement, zero hallucination surface, ~0 ms of GPU time.
- The vector index holds *prose*: procedures, explanations, postmortems, distilled Slack threads.

### 3.3 Ingestion pipeline

```
 Sources                Parse & Extract              Enrich                    Index
┌──────────┐   ┌──────────────────────────┐   ┌──────────────────┐   ┌─────────────────────┐
│ PDFs      │──▶│ Docling (layout-aware)   │──▶│ Metadata tagging │──▶│ Qdrant (dense+sparse│
│ Git repos │   │ + OCR for scans          │   │  machine_id      │   │ hybrid index)        │
│ Slack     │   │ Table extraction ────────│──▶│  doc_type        │   ├─────────────────────┤
│ CMMS      │   │  → Postgres (structured) │   │  safety_critical │   │ Postgres (tables,   │
│ Drive     │   │ Chunking:                │   │  site / role     │   │ registry, versions) │
└──────────┘   │  headings-aware, 400–800 │   │  version, date   │   └─────────────────────┘
               │  tok, tables kept intact │   └──────────────────┘
               └──────────────────────────┘
```

Pipeline properties:

- **Layout-aware parsing** (Docling or equivalent): vendor manuals are useless if tables and procedure step-numbering are flattened. Chunking respects heading hierarchy; a chunk carries its breadcrumb (`Manual > Press Brake X7 > Hydraulics > Fault Recovery`).
- **Metadata spine:** every chunk is tagged with `machine_id` (joined against the equipment registry), `doc_type`, `site`, `safety_critical`, `version`, `effective_date`. These power retrieval filters, the memory tiers (§4), and role-based access.
- **Hybrid retrieval:** dense embeddings (BGE-M3, runs on CPU) + sparse/BM25 in the same Qdrant collection, fused with RRF. Factory vocabulary is full of exact identifiers ("X7-2 anneal recipe", "E-217") where lexical match beats semantic similarity.
- **Versioning & freshness:** documents are upserted by content hash; superseded SOP versions stay queryable but are excluded from default retrieval. A nightly job re-syncs Git and CMMS; manuals are ingested on upload.
- **Slack distillation queue (human-in-the-loop):** raw Slack is too noisy to embed wholesale. Resolved threads with ≥N replies and a marked answer are summarized into draft KB articles and queued for founder review. Approval promotes them into the corpus (this is the Tier-3 → Tier-2 promotion path in §4).

### 3.4 Pre-LLM milestone

Before any generation is enabled, ship **search-only mode**: hybrid retrieval with filters + the structured fault-code lookup, exposed in Slack and a minimal web UI. This validates ingestion quality, builds the click/feedback data that later powers evaluation, and delivers value in week ~2 instead of week ~8.

---

## 4. Memory & Knowledge System: Three-Tier Architecture

Adapted from the three-tier memory architecture I worked with during my AI-infrastructure co-op, mapped onto Nox's actual structure. The tiers differ in **lifetime, write authority, and where they live in the prompt** — which is what makes the KV-cache strategy in §5 work.

```
┌───────────────────────────────────────────────────────────────────┐
│ TIER 1 — ORGANIZATION MEMORY            lifetime: quarters–years  │
│ write: founder/safety-lead approved      prompt: static prefix     │
│        + curated corpus                  (KV-cache resident)      │
├───────────────────────────────────────────────────────────────────┤
│ TIER 2 — SITE / ROLE MEMORY             lifetime: weeks–months    │
│ write: promotion pipeline + admins       prompt: stable role block │
│                                          + retrieval filters       │
├───────────────────────────────────────────────────────────────────┤
│ TIER 3 — SESSION MEMORY                 lifetime: hours (TTL)     │
│ write: automatic                         prompt: suffix (dynamic)  │
└───────────────────────────────────────────────────────────────────┘
```

### Tier 1 — Organization Memory (the canon)

What lives here for Nox:

- **Safety policy preamble:** the rules the assistant itself follows — quote-verbatim-or-refuse for `safety_critical` content, always cite, never improvise a procedure.
- **Nox glossary:** company-specific vocabulary (line names, machine nicknames, recipe terminology, internal acronyms). This single block dramatically improves answer quality for a company whose language is 50% jargon.
- **Equipment registry summary:** the fleet at a glance — machines, models, sites, firmware major versions.
- **Software architecture overview:** the 2-page "how the orchestration stack fits together" every engineer needs.
- **Org-wide curated corpus:** approved manuals, SOPs, ADRs, postmortems (retrieved, not in-prompt).

Properties: slow-changing, founder-approved, versioned like a release. The in-prompt portion (~2–4K tokens) is **byte-identical across all requests**, which is what keeps it resident in the vLLM KV-cache (§5).

### Tier 2 — Site / Role Memory

What lives here for Nox:

- **Per-site:** machine configurations and line layout for that site, site-specific SOP variants, local network/equipment quirks ("Site B's furnace 2 runs the legacy firmware").
- **Per-role context blocks:** a stable ~500-token block per role — operators get safety-and-procedure orientation; controls engineers get PLC/firmware orientation; software engineers get codebase orientation. Stable per role ⇒ also prefix-cacheable per role.
- **Retrieval ACL filters:** role + site scope what's retrievable. Operators don't get codebase internals; early-tenure hires don't get full process recipes (the moat). Enforced as Qdrant metadata filters, not prompt instructions.
- **Promoted knowledge:** the output of the Tier-3 distillation pipeline — founder-reviewed troubleshooting articles, distilled Slack threads, resolved-ticket case studies, scoped to the site/role they concern.

Write authority: the promotion pipeline (human-approved) and the KB admin. Lifetime: weeks–months, reviewed on a cadence.

### Tier 3 — Session Memory

What lives here for Nox:

- Rolling conversation history (token-budgeted, oldest-first eviction).
- **Active machine context:** when a user says "the press is faulting again," the session pins `machine_id=X7-2`, and subsequent retrieval is filtered to it. This is the single biggest relevance win for floor troubleshooting.
- Retrieved-chunk cache for the session (avoid re-retrieving identical context turn over turn).
- Scratchpad state for multi-step procedures ("you're on step 4 of the LOTO checklist").

Lifetime: TTL of hours; stored in Redis. **End-of-session distillation:** sessions that resolved a real problem (positive feedback + a resolution pattern) are summarized into a draft article and pushed to the Tier-2 promotion queue. This is the mechanism that converts founder interrupts into permanent organizational memory — answer once, capture forever.

---

## 5. Engineering Constraints (Internship Best Practices, Applied)

### 5.1 KV-Cache-Friendly Context Engineering

Nox's context is dominated by heavy static material (safety preamble, glossary, role blocks). vLLM's **Automatic Prefix Caching (APC)** lets us pay the prefill cost for that material once per cache lifetime instead of once per request — but only if the prompt is engineered for it. APC matches on exact token prefixes, so the rules are strict:

**Prompt layout — ordered by stability, most static first:**

```
┌─────────────────────────────────────────────┐
│ [1] System + safety policy        (Tier 1)  │  identical for ALL requests
│ [2] Nox glossary + fleet summary  (Tier 1)  │  identical for ALL requests
│ [3] Role context block            (Tier 2)  │  identical per ROLE (one of ~4)
├─ ── ── ── prefix cache boundary ── ── ── ──┤
│ [4] Retrieved chunks (deterministic order)  │  varies per query
│ [5] Conversation history (append-only)      │  varies per turn
│ [6] Current user query                      │  varies per turn
└─────────────────────────────────────────────┘
```

Hard rules to keep the prefix byte-identical:

- **No timestamps, request IDs, or user names** anywhere in segments [1]–[3]. The classic mistake is `Current time: 10:47 AM` at the top of the system prompt — it invalidates the entire cache on every request.
- **Role blocks are an enum, not a template.** Four roles ⇒ four cached prefixes, not 45 per-user variants.
- **Retrieved chunks are ordered deterministically** (by stable chunk ID after score-ranking ties), and conversation history is strictly append-only, so multi-turn requests re-hit the longest possible prefix.
- **Cache warming:** on deploy and on a schedule, fire one request per role to re-populate the four prefixes before users arrive.

Expected effect: segments [1]–[3] are ~3–5K tokens. On a single GPU, skipping their prefill cuts TTFT for a typical request from ~1.5–2 s to ~300–500 ms — the difference between "tool" and "toy" for an operator standing at a faulting machine. We monitor `vllm:prefix_cache_hit_rate` in Grafana and treat a drop as a regression (it almost always means someone added dynamic text to the prefix).

In the Ambitious Version this extends to **prefix-cache-aware routing**: the router hashes the prompt prefix (effectively, the role) and routes to the replica whose KV-cache already holds it, so cache hit rates stay high across a multi-replica fleet instead of being diluted by random load balancing.

### 5.2 Deterministic Bypasses

LLMs are the most expensive and least reliable component in the stack; they should be the path of last resort. The gateway routes every query through escalating tiers:

```
Query ──▶ TIER 0: Regex / exact match              (~0 ms, no GPU)
          │  • fault-code patterns (E-\d{3}, vendor formats) → Postgres lookup, cited row
          │  • part numbers / machine IDs → equipment registry lookup
          │  • "where is/find the <doc>" → direct search-result list, no generation
          │  • greetings, "help" → canned responses
          ▼
          TIER 1: Lightweight intent classifier     (~5 ms, CPU)
          │  • keyword rules + embedding-similarity match against intent exemplars
          │  • routes: doc-lookup vs. troubleshooting vs. how-to vs. out-of-scope
          │  • semantic answer cache: near-duplicate of a recently answered query
          │    (similarity > threshold, same role/site scope) → serve cached answer
          ▼
          TIER 2: LLM (RAG synthesis)               (GPU)
             • only for queries that genuinely need synthesis across sources
```

No LLM is ever used for routing or intent classification. Expected effect, based on the query mix a factory generates (fault lookups and doc-finding dominate): **30–40% of queries never touch the GPU**, which on a single-GPU MVP is effectively a capacity multiplier. Tier-0 answers are also *better* than LLM answers for their query class — exact, instant, and impossible to hallucinate.

Every routing decision is logged to Langfuse with the tier that served it, so we can verify the bypass rate empirically rather than assume it.

### 5.3 Tracking & Evaluation (Langfuse + eval discipline)

**Tracing (from day one, not retrofitted):** self-hosted Langfuse. Every request produces one trace:

```
trace: { user_role, site, session_id }
 ├─ span: router          { tier_served, rule_matched }
 ├─ span: retrieval       { query, filters, candidates + scores, reranked order }
 ├─ span: generation      { model, prompt_tokens, completion_tokens,
 │                          TTFT, prefix_cache_hit, finish_reason }
 └─ event: feedback       { thumbs, optional comment }   ← Slack reactions + UI buttons
```

Paired with vLLM's Prometheus metrics (queue time, prefix cache hit rate, tokens/s) in Grafana. The KB admin's dashboard answers: bypass rate, cache hit rate, TTFT p50/p95, deflection rate, top unanswered queries.

**Evaluation:**

- **Golden set (the founding investment):** ~100–150 Q/A pairs curated *with the founders* — they are the only ground truth that exists. Stratified by: fault lookups, hardware-software seam questions, SOP/safety questions, software architecture questions. Includes **must-cite** cases (answer is only correct with the right source) and **must-refuse** cases (safety questions the corpus can't ground — the correct behavior is refusal + escalation pointer).
- **Metrics:** RAGAS-style — faithfulness (claims grounded in retrieved context), answer relevancy, context precision/recall — scored by an LLM-as-judge. In V1 the judge is the local model on off-hours batches; in the Ambitious Version, a dedicated larger judge model. Faithfulness on `safety_critical` queries is a **hard gate: required 1.0** — any regression blocks deploy.
- **Regression gating in CI:** any change to prompts, chunking, retrieval parameters, or the model runs the golden set; faithfulness/citation regressions block the merge. Prompts and retrieval configs are versioned artifacts, not strings in code.
- **Online signals:** thumbs ratio, "escalated to human anyway" rate, and the weekly **top-10 unanswered queries** report — which doubles as the ingestion backlog (unanswerable queries are missing documents, not model failures).
- **Feedback → data flywheel:** thumbs-down traces are auto-queued for review; confirmed failures become new golden-set entries. The eval set grows with the company.

---

## 6. Version 1 — MVP on a Single Local GPU

**Goal:** prove the loop — *ingest → retrieve → answer with citations → capture feedback* — for one site, with founders' real documents, in ~6–8 weeks. Optimize for trustworthiness and TTFT, not capability breadth.

### 6.1 Hardware & model


| Component      | Choice                                                       | Rationale                                                                           |
| -------------- | ------------------------------------------------------------ | ----------------------------------------------------------------------------------- |
| GPU            | 1× RTX 4090 (24 GB) — or L40S/A6000 (48 GB) if budget allows | Workstation-class, deployable in the factory office, no cloud dependency            |
| LLM            | Qwen-class 14B instruct, AWQ 4-bit (≈9 GB weights)           | Best quality/VRAM at this size; strong citation-following. 48 GB card ⇒ 32B-class   |
| Context budget | 16K serving window                                           | ~14 GB left for KV-cache ⇒ ample room for APC-pinned prefixes + concurrent sessions |
| Embeddings     | BGE-M3 on **CPU**                                            | Hybrid dense+sparse from one model; keeps GPU exclusively for generation            |
| Reranker       | BGE-reranker, CPU, top-20 → top-5                            | Biggest retrieval-quality win per dollar; CPU latency fine at this scale            |


Concurrency reality check: 45 employees ⇒ realistically <10 concurrent sessions at peak; with 30–40% of queries bypassing the GPU (§5.2), a single card serves this comfortably.

### 6.2 V1 architecture

```
 Slack bot ─┐                                       ┌─ vLLM (APC on) ── Qwen-14B-AWQ
 Web UI ────┤                                       │       [GPU]
 Floor      ├──▶ FastAPI Gateway ──▶ Tiered Router ─┤
 terminal ──┘      │                  (§5.2)        ├─ Postgres ── fault tables, registry,
                   │                                │             doc versions, sessions
                   ├──▶ Langfuse (traces)           ├─ Qdrant ─── hybrid prose index
                   └──▶ Prometheus/Grafana          └─ Redis ──── Tier-3 session memory

 Ingestion (offline): Docling + OCR ─▶ chunk/extract ─▶ tag ─▶ Qdrant + Postgres
 All of it: Docker Compose on one box.
```

### 6.3 V1 scope

**In:** one site; manuals + SOPs + ADRs + equipment registry + fault tables; search-only mode first (§3.4); tiered router; three-tier memory with role blocks for 3–4 roles; citations on every generated answer; verbatim-or-refuse for `safety_critical`; Langfuse + golden-set eval + CI gate; Slack bot + minimal web UI.

**Out (deferred to Ambitious):** multimodal, voice, agentic capture, fine-tuning, multi-site, SSO/RBAC beyond role enum, code-level search.

### 6.4 V1 success metrics


| Metric                                                                      | Target                                            |
| --------------------------------------------------------------------------- | ------------------------------------------------- |
| Founder interrupt deflection (questions answered without pinging a founder) | >50% of eligible questions by week 12             |
| TTFT p50 / p95 (generated answers)                                          | <500 ms / <2 s                                    |
| Prefix cache hit rate                                                       | >80% of LLM-tier requests                         |
| GPU bypass rate (Tier 0/1 served)                                           | >30%                                              |
| Faithfulness on golden set                                                  | >0.9 overall; **1.0 on safety class (hard gate)** |
| Time-to-first-safe-task for new operator                                    | Baseline in month 1, then −30%                    |


---

## 7. Ambitious Version — Scaled Multi-GPU Local Platform

**Goal:** from "assistant that answers" to **the system of record for how Nox runs its factories** — multi-site, multimodal, and self-feeding.

### 7.1 Hardware & serving topology

Target: one 4–8× GPU node (e.g., 4–8× L40S 48 GB or equivalent), still fully on-prem.


| GPUs     | Workload                                                                                                                                                        |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2–4      | Primary LLM: 70B-class instruct, AWQ, tensor-parallel (TP=2–4) — the quality jump that handles genuine multi-source synthesis across the hardware-software seam |
| 1–2      | vLLM replicas of the 14B model for high-volume/simple queries (**model-tier routing:** the deterministic router now also picks model size by query class)       |
| 1        | Multimodal VLM (Qwen-VL-class): wiring diagrams, manual figures, operator photos of faulting equipment                                                          |
| shared/1 | Embeddings + reranker (GPU-accelerated now), Whisper for voice, LLM-as-judge for continuous eval                                                                |


**Prefix-cache-aware routing:** the gateway routes by prompt-prefix hash (effectively role) so each replica's KV-cache specializes in a subset of role prefixes — sustaining the >80% cache hit rate that random load balancing would destroy. This is the multi-GPU continuation of the §5.1 discipline.

### 7.2 Capability roadmap

1. **Multimodal ingestion & queries.** Diagrams and manual figures become first-class retrievable objects. An operator photographs a fault screen or a wiring panel; the VLM grounds the answer in both the image and the manuals. This is the single highest-value upgrade for floor users.
2. **Voice on the floor.** Local Whisper STT + TTS at floor terminals. Operators mid-task have gloved, busy hands; voice is the difference between "used during work" and "used after work."
3. **Agentic knowledge capture.** The Tier-3 → Tier-2 promotion pipeline becomes proactive: agents draft KB articles from resolved Slack threads and CMMS tickets, diff new SOP versions against old and flag affected articles, and detect contradiction between sources ("manual says X, postmortem says we do Y") — with founders always as the approval gate. The system starts *closing* its own knowledge gaps instead of just reporting them.
4. **Telemetry-aware answers ("ask the fleet").** Read-only connectors to machine telemetry/historians let the deterministic router answer "what's the current state of furnace 2" via direct queries — extending the don't-vectorize-what-you-can-SELECT principle to live data. Strictly read-only: this system never actuates equipment.
5. **Domain LoRA adapters.** With ~6 months of feedback data and the grown golden set: LoRA fine-tunes for Nox vocabulary and citation format, served via vLLM multi-LoRA (one base model in VRAM, per-role adapters). Cheap to train locally, reversible, and evaluated against the same CI gate as every other change.
6. **Multi-site + full RBAC/SSO.** Tier-2 memory shards per site; SSO-backed roles; audit logs on every retrieval of `safety_critical` or recipe-class content — the compliance posture a 45-person regulated manufacturer needs.
7. **Equipment knowledge graph.** The metadata spine matures into an explicit graph (machine → firmware → SOPs → known faults → postmortems → owning code modules), enabling impact queries no vector search can answer: "what procedures are affected if we upgrade the X7 firmware?"

### 7.3 What deliberately stays the same

The disciplines are scale-invariant — this is the thesis of the proposal:

- Deterministic tiers still front the LLM (they get *more* valuable as query volume grows).
- The prompt-stability contract still governs context layout (now enforced across a fleet by the cache-aware router).
- Every capability addition ships with golden-set coverage and the same CI regression gate.
- Humans (founders, then designated experts) remain the write authority on Tier-1/Tier-2 knowledge. The system drafts; people canonize.

---

## 8. Risks & Open Questions


| Risk                                            | Mitigation                                                                                                                                  |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| OCR quality on old scanned vendor manuals       | Search-only milestone surfaces parse failures early; manual-priority re-scans for the worst offenders                                       |
| Founder review becomes the new bottleneck       | Cap review queue at ~30 min/person/week; batch by topic; track queue latency as a system metric                                             |
| Safety answers trusted beyond their reliability | Verbatim-quote-or-refuse policy + 1.0 faithfulness gate + visible "source: LOTO-SOP v4 §3.2" citations + safety-lead veto on that doc class |
| Adoption (45 people ignore it)                  | Slack-first (meet users where they ask today); search-only value in week 2; founders redirect pings to the bot ("ask Nox-KB first")         |
| Recipe/IP leakage to early-tenure hires         | Tier-2 ACL filters enforced at retrieval, audit-logged; not enforced by prompt                                                              |
| Single-GPU capacity surprise                    | Bypass tiers + semantic cache as capacity multipliers; Grafana queue-time alert as the upgrade trigger                                      |


**Open questions for Nox:** Which CMMS/ticketing system (affects connector priority)? One site or two during V1? Are process recipes in-scope for retrieval at all in V1, or registry-metadata only? What's the realistic floor hardware (shared terminal vs. tablets)?

---

## 9. Delivery Plan (V1)


| Weeks | Milestone                                                                                                             |
| ----- | --------------------------------------------------------------------------------------------------------------------- |
| 1–2   | Infra up (Compose stack); equipment registry + fault tables in Postgres; first manuals parsed                         |
| 2–4   | **Search-only mode live in Slack** (§3.4); hybrid retrieval + Tier-0 fault lookup; Langfuse tracing on                |
| 4–6   | Generation enabled: three-tier prompt assembly, APC verified (cache-hit dashboards), citations, safety refusal policy |
| 6–8   | Golden set (founder sessions) + CI regression gate; feedback capture; promotion queue v0; success-metric baselines    |
| 8+    | Iterate on top-unanswered-queries backlog; begin Ambitious Version sequencing by observed demand                      |


