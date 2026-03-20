# PRD: Lead Hand — Autonomous Lead Generation for OpenClaw

**Status:** Draft
**Date:** 2026-03-20
**Author:** Claude (analysis of OpenFang's Lead Hand)
**Target:** OpenClaw core

---

## 1. Overview

The Lead Hand is an autonomous agent that discovers, enriches, scores, and delivers sales leads without manual prompting. Inspired by OpenFang's "Hands" architecture, this PRD defines how OpenClaw can implement equivalent functionality as a first-class autonomous capability.

OpenFang's key insight: shift agents from reactive (respond to user queries) to proactive (run on schedules, build knowledge, deliver results). The Lead Hand is the highest-value demonstration of this paradigm — it replaces manual prospecting with an always-on pipeline.

---

## 2. Problem Statement

Sales teams and solo founders spend 30-60% of their time on manual prospecting:
- Searching LinkedIn, directories, and databases for prospects
- Manually researching companies and contacts
- Qualifying leads against ideal customer profiles
- Deduplicating against existing CRM/pipeline
- Formatting and routing qualified leads

An autonomous lead agent eliminates this entirely by running daily, learning the user's ICP over time, and delivering scored, enriched leads to their preferred channel.

---

## 3. Goals

| Goal | Metric |
|------|--------|
| Autonomous daily lead discovery | Runs on cron schedule without user prompting |
| ICP-based targeting | Configurable industry, company size, role, geography filters |
| Lead enrichment | Each lead includes company data, contact info, web research |
| Lead scoring | 0-100 score based on ICP fit |
| Deduplication | Zero duplicate leads delivered across runs |
| Multi-format output | CSV, JSON, Markdown delivery |
| Channel delivery | Results pushed to Telegram, Discord, Slack, email, or dashboard |
| ICP learning | Profile refines over time based on user feedback |

---

## 4. Architecture

### 4.1 Hand Manifest (Declarative Config)

Adopt a TOML-based manifest similar to OpenFang's `HAND.toml`:

```toml
[hand]
id = "lead"
name = "Lead Generator"
description = "Discovers prospects matching your ICP, enriches with research, scores 0-100"
category = "sales"
version = "1.0.0"

[hand.requires]
tools = ["web_search", "web_scrape", "csv_write", "memory_store"]

[hand.settings]
target_industry = { type = "string", default = "", description = "Target industry vertical" }
target_role = { type = "string", default = "", description = "Target job titles/roles" }
company_size = { type = "string", default = "10-500", options = ["1-10", "10-50", "50-200", "200-500", "500+"] }
geography = { type = "string", default = "", description = "Target geography/region" }
min_score = { type = "int", default = 60, min = 0, max = 100, description = "Minimum score to deliver" }
leads_per_run = { type = "int", default = 20, min = 5, max = 100 }
output_format = { type = "string", default = "markdown", options = ["csv", "json", "markdown"] }
delivery_channel = { type = "string", default = "dashboard", description = "Channel to deliver results" }

[hand.agent]
model = "claude-sonnet-4-6"
temperature = 0.2
max_iterations = 50

[hand.schedule]
cron = "0 9 * * 1-5"  # Weekdays at 9 AM

[hand.dashboard]
metrics = ["leads_found", "leads_qualified", "avg_score", "top_industries"]
```

### 4.2 Pipeline Stages

The Lead Hand runs a 5-stage pipeline on each execution:

```
┌─────────────┐    ┌─────────────┐    ┌──────────┐    ┌──────────────┐    ┌──────────┐
│  Discovery   │───▶│ Enrichment  │───▶│ Scoring  │───▶│    Dedup     │───▶│ Delivery │
│              │    │             │    │          │    │              │    │          │
│ Web search   │    │ Company     │    │ ICP fit  │    │ Against      │    │ Format   │
│ Directory    │    │ Contact     │    │ 0-100    │    │ existing DB  │    │ Route    │
│ Social       │    │ Research    │    │ Rank     │    │ Previous     │    │ Notify   │
└─────────────┘    └─────────────┘    └──────────┘    └──────────────┘    └──────────┘
```

#### Stage 1: Discovery
- Query web search (Brave/Perplexity/Tavily) for prospects matching ICP criteria
- Scrape relevant directories, job boards, company listings
- Extract candidate companies and contacts from search results
- Sources: LinkedIn (public profiles), Crunchbase, industry directories, company blogs, job postings

#### Stage 2: Enrichment
- For each discovered prospect, research:
  - Company: size, funding, tech stack, recent news, hiring signals
  - Contact: role, seniority, public email, social profiles
  - Signals: recent funding rounds, job postings (growth indicator), tech blog activity
- Store enrichment data in structured format

#### Stage 3: Scoring
- Score each lead 0-100 based on weighted ICP criteria:
  - **Industry match** (0-25): exact match vs adjacent
  - **Company size fit** (0-20): within target range
  - **Role/seniority match** (0-20): target title alignment
  - **Geography match** (0-10): location alignment
  - **Signal strength** (0-15): funding, hiring, growth indicators
  - **Data completeness** (0-10): how much enrichment data was found
- Filter leads below `min_score` threshold

#### Stage 4: Deduplication
- Compare against previously delivered leads (stored in agent memory)
- Match on: company domain, contact email, company name (fuzzy)
- Skip duplicates, flag "re-emerged" leads with new signals

#### Stage 5: Delivery
- Format output per user preference (CSV/JSON/Markdown)
- Route to configured delivery channel
- Update dashboard metrics
- Store delivered leads in memory for future dedup
- Log run summary with counts and top leads

### 4.3 ICP Learning

Over time, the Lead Hand refines its targeting:
- Track which delivered leads get positive user feedback (thumbs up, "contacted", "converted")
- Weight discovery queries toward patterns that produced high-feedback leads
- Surface ICP drift: "Your best leads are trending toward Series A fintech companies"
- Store learned ICP as structured profile in agent memory

---

## 5. Data Model

### Lead Record

```typescript
interface Lead {
  id: string;
  discoveredAt: string; // ISO 8601
  runId: string;

  // Company
  companyName: string;
  companyDomain: string;
  companySize: string;
  industry: string;
  location: string;
  fundingStage?: string;
  techStack?: string[];
  recentNews?: string[];

  // Contact
  contactName?: string;
  contactRole?: string;
  contactEmail?: string;
  contactLinkedIn?: string;

  // Scoring
  score: number; // 0-100
  scoreBreakdown: {
    industryMatch: number;
    companySizeFit: number;
    roleMatch: number;
    geographyMatch: number;
    signalStrength: number;
    dataCompleteness: number;
  };

  // Signals
  signals: string[]; // e.g. "Recently raised Series A", "Hiring 3 engineers"

  // Lifecycle
  status: "discovered" | "delivered" | "contacted" | "converted" | "rejected";
  feedback?: "positive" | "negative" | "neutral";
}
```

### ICP Profile

```typescript
interface ICPProfile {
  targetIndustries: string[];
  targetRoles: string[];
  companySizeRange: [number, number];
  geographies: string[];
  keywords: string[];
  excludeCompanies: string[];
  excludeDomains: string[];

  // Learned weights (updated over time)
  learnedWeights: {
    industryPreferences: Record<string, number>;
    rolePreferences: Record<string, number>;
    signalPreferences: Record<string, number>;
  };
}
```

---

## 6. OpenClaw Implementation Plan

### Phase 1: Hand Framework (Foundation)

Build the autonomous Hand runtime into OpenClaw:

1. **Hand manifest parser** — parse `HAND.toml` configs (or equivalent YAML/JSON in OpenClaw style)
2. **Hand lifecycle manager** — activate/pause/resume/deactivate state machine
3. **Scheduler** — cron-based execution engine that triggers Hand runs
4. **Hand CLI** — `openclaw hand activate lead`, `openclaw hand status`, `openclaw hand config lead --set target_industry=SaaS`
5. **Dashboard metrics** — store and display per-hand run metrics

Implementation location: `src/hands/` (new module)

### Phase 2: Lead Hand Core

1. **Discovery engine** — integrate with existing web search tools (Brave, Perplexity, Tavily) to find prospects
2. **Enrichment pipeline** — multi-step web research per prospect using existing scraping tools
3. **Scoring engine** — weighted ICP scoring with configurable thresholds
4. **Dedup store** — SQLite or agent memory-based deduplication
5. **Output formatters** — CSV, JSON, Markdown renderers

Implementation location: `src/hands/lead/` (new module)

### Phase 3: Delivery & Integration

1. **Channel delivery** — route lead reports to any configured OpenClaw channel (Telegram, Discord, Slack, etc.)
2. **Dashboard UI** — lead pipeline view in web dashboard
3. **Feedback loop** — user can react to leads (thumbs up/down) to train ICP
4. **Export API** — REST endpoints for lead CRUD and pipeline status

### Phase 4: ICP Learning

1. **Feedback ingestion** — capture user reactions from channels
2. **Weight adjustment** — Bayesian update of ICP weights based on feedback
3. **Drift detection** — surface changing patterns in qualified leads
4. **Profile recommendations** — suggest ICP refinements based on conversion data

---

## 7. Competitive Differentiation vs OpenFang

| Aspect | OpenFang Lead Hand | OpenClaw Lead Hand (Proposed) |
|--------|-------------------|-------------------------------|
| Runtime | Rust binary, WASM sandbox | TypeScript/Node, existing OpenClaw runtime |
| Channels | 40 adapters | All OpenClaw channels (core + extensions) |
| LLM support | 27 providers | All OpenClaw-supported providers |
| Security | 16 security layers | Existing OpenClaw security + channel auth |
| Install | Separate binary | Built into OpenClaw — zero extra install |
| Existing users | Greenfield | Leverage existing OpenClaw user base + channel configs |
| Ecosystem | FangHub marketplace | OpenClaw plugin system / Clawhub |
| ICP learning | Basic | Feedback-loop driven with drift detection |

### OpenClaw advantages to leverage:
- **Existing channel infrastructure** — leads delivered to any already-configured channel
- **Existing web search integration** — Brave, Perplexity, Tavily already wired up
- **Plugin ecosystem** — Lead Hand ships as a first-party plugin or built-in hand
- **Gateway architecture** — runs alongside existing agents without new infrastructure
- **Pi sessions** — lead research can leverage existing conversation/memory infrastructure

---

## 8. User Stories

1. **As a founder**, I want to configure my ICP once and receive qualified leads daily so I can focus on selling instead of prospecting.
2. **As a sales rep**, I want leads scored 0-100 so I can prioritize my outreach by fit quality.
3. **As a team lead**, I want leads deduplicated against our existing pipeline so we don't double-contact prospects.
4. **As a marketer**, I want lead data exported as CSV so I can import into our CRM.
5. **As a power user**, I want the Lead Hand to learn from my feedback so targeting improves over time.
6. **As a user**, I want lead reports delivered to my Telegram/Slack so I see them in my existing workflow.

---

## 9. CLI Interface

```bash
# Activate the lead hand
openclaw hand activate lead

# Configure ICP
openclaw hand config lead \
  --set target_industry="SaaS,fintech" \
  --set target_role="CTO,VP Engineering" \
  --set company_size="50-500" \
  --set geography="US,EU" \
  --set min_score=70 \
  --set leads_per_run=25 \
  --set output_format=csv \
  --set delivery_channel=telegram

# Check status
openclaw hand status lead

# View recent leads
openclaw hand leads --last-run
openclaw hand leads --score-above 80
openclaw hand leads --export csv > leads.csv

# Pause/resume
openclaw hand pause lead
openclaw hand resume lead

# Give feedback
openclaw hand feedback lead --lead-id abc123 --positive
openclaw hand feedback lead --lead-id def456 --negative

# View ICP profile (including learned weights)
openclaw hand icp lead
```

---

## 10. Success Criteria

| Criteria | Target |
|----------|--------|
| Daily autonomous execution | Runs without user prompting on schedule |
| Lead quality | >60% of scored leads rated useful by user feedback |
| Discovery coverage | 15-30 new prospects per run |
| Enrichment depth | Company + contact data for >80% of leads |
| Dedup accuracy | <2% duplicate leads across runs |
| Delivery latency | Results delivered within 5 min of run completion |
| ICP convergence | Scoring accuracy improves by 10%+ after 2 weeks of feedback |

---

## 11. Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Web scraping rate limits | Rotate search providers, respect robots.txt, cache results |
| Low-quality discovery results | Multi-source cross-referencing, minimum enrichment threshold |
| LLM cost per run | Use fast/cheap models for scoring, expensive models only for enrichment |
| Data privacy (GDPR) | Only use publicly available data, provide data deletion, document sources |
| ICP cold start | Require minimum ICP config before first run, use sensible defaults |
| Stale leads | Track lead age, re-score periodically, surface freshness in output |

---

## 12. Open Questions

1. **Hand framework scope**: Should the Hand framework be generic (supporting all 7 OpenFang-style hands) or start Lead-only?
2. **Storage backend**: Use existing OpenClaw agent memory, or introduce a dedicated leads SQLite table?
3. **Pricing/quotas**: Should autonomous runs count against any usage quota?
4. **Plugin vs built-in**: Ship as a core feature or as an installable plugin via Clawhub?
5. **CRM integrations**: Should Phase 1 include direct CRM export (HubSpot, Salesforce) or defer to CSV?
