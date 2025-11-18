# AI Exposure Optimization (AEO) & LLM Visibility SaaS Blueprint

## 1. Product Overview
Clareo is a multi-tenant SaaS platform that gives service-based businesses full visibility into how large language models (LLMs) perceive, describe, and recommend them. The platform orchestrates automated scans across ChatGPT, Claude, Gemini, and Copilot, compares competitor positioning, mines audience questions, and delivers prioritized remediation plans. It combines AI-native telemetry, embeddings, and background jobs to help customers operationalize "AI search" as predictably as SEO.

### Target Customers
- Microsoft partners (ISVs, MSPs, CSPs, SIs)
- B2B SaaS vendors
- Marketing/SEO agencies
- Consultants and fractional executives
- IT services, cybersecurity, legal, accounting, and other professional services

### Goals
1. Reveal how LLMs describe a brand and its competitors.
2. Detect hallucinations, sentiment drift, and gaps in coverage.
3. Mine buyer-intent questions and content opportunities.
4. Generate AI-friendly sitemaps and schema to "teach" models about the business.
5. Deliver prioritized, recurring recommendations and shareable reports.

---

## 2. Core Modules & Capabilities

| Module | Purpose | Key Outputs |
| --- | --- | --- |
| AI Visibility Scanner | Query LLMs with 50–200 auto-generated buyer questions per scan. | Visibility score, sentiment tone map, hallucination alerts, "How AI Describes You" narrative. |
| Competitor Comparison | Run identical scans for selected/auto-discovered competitors. | Comparative scores, rankings, positioning deltas, keyword gaps. |
| AI Question Mining | Discover category-defining queries, objections, and thought-leadership prompts. | Ranked question clusters, pain/use-case lists, FAQ seeds. |
| AI Sitemap Builder | Produce AI-optimized sitemap with entities, summaries, and schema. | JSON/HTML sitemap, FAQ schema, AI-friendly content snippets. |
| Recommendation Engine | Prioritize remediation in red/yellow/green tiers. | Action plan with owners, impact/effort, links to evidence. |
| Reporting & Sharing | Package findings and trends. | PDF exports, share links, before/after visualizations, recurring scan scheduling. |

---

## 3. System Architecture

### Component Diagram (textual)
1. **Next.js Frontend**: Multi-tenant dashboard, onboarding, visualizations, report builders.
2. **API Gateway (Node.js or Python/FastAPI)**: Auth, request validation, job orchestration, REST + WebSocket for live updates.
3. **Worker Cluster (BullMQ or Celery)**: Runs scan jobs, integrates with OpenAI/Claude/Gemini, handles embeddings, sentiment analysis.
4. **Supabase (PostgreSQL)**: Core relational data, multi-tenant security policies, row-level security per workspace.
5. **Vector Store (Supabase pgvector or Pinecone)**: Stores embeddings for question mining, content similarity, competitor clustering.
6. **Object Storage (Supabase Storage or S3)**: PDF reports, shareable exports.
7. **Analytics Service (e.g., Metabase or custom)**: Trend dashboards, billing metrics.
8. **Billing (Stripe)**: Subscription management, usage metering, plan entitlements.

### Data Flow Summary
1. User submits domain + competitors via onboarding form.
2. Backend creates a scan job, persists metadata, enqueues tasks.
3. Workers generate buyer-intent prompts via embeddings + templates.
4. Workers query each LLM (ChatGPT, Claude, Gemini, Copilot) asynchronously and capture raw transcripts.
5. Post-processing pipeline extracts mentions, sentiment, hallucinations, and scores per model/per question.
6. Results stored in relational tables; embeddings saved for similarity search.
7. Recommendation engine runs rule-based + LLM reasoning passes to create prioritized actions.
8. Frontend subscribes to job status; when complete, dashboards render charts, comparisons, and exports.

---

## 4. Database Schema (Supabase/Postgres)

### Core Tables
- **accounts**: id (uuid), name, plan_tier, stripe_customer_id, created_at.
- **users**: id, account_id (FK), email, role, hashed_password, created_at.
- **workspaces**: id, account_id, domain, industry, primary_contact, created_at.
- **competitors**: id, workspace_id, name, domain, auto_discovered (bool).
- **scans**: id, workspace_id, initiated_by, status (queued/running/complete/failed), started_at, completed_at, scan_type (baseline/competitor/question-mining), llm_question_count.
- **scan_models**: id, scan_id, llm_vendor (gpt4o/claude_sonnet/gemini/copilot), model_name, visibility_score, sentiment_score, hallucination_count, raw_transcript_url.
- **scan_questions**: id, scan_id, question_text, question_cluster, buyer_stage, priority.
- **scan_answers**: id, scan_model_id, scan_question_id, mentioned_brand (enum: self/competitor/none), mention_confidence, sentiment_label, hallucination_flag, summary.
- **competitor_scores**: id, scan_id, competitor_id, score, sentiment, ranking_position.
- **question_opportunities**: id, workspace_id, question_text, cluster, volume_estimate, difficulty, last_seen_scan_id.
- **ai_sitemaps**: id, workspace_id, sitemap_json, schema_markup, created_at.
- **recommendations**: id, workspace_id, scan_id, severity (red/yellow/green), title, description, impact, effort, owner, status.
- **reports**: id, workspace_id, scan_id, report_type (pdf/sharelink), storage_path, expires_at.
- **billing_usage**: id, account_id, period_start, period_end, scans_run, tokens_used, overage_fee.

### Indexing & Policies
- Row-Level Security by account/workspace on all workspace-scoped tables.
- GIN indexes on question text and competitor names for search.
- pgvector indexes on embeddings (question mining, content matching).

---

## 5. API Specification (REST + WebSocket)

| Method | Endpoint | Description | Auth |
| --- | --- | --- | --- |
| POST | `/api/auth/login` | Email/password login; returns JWT. | None |
| POST | `/api/auth/signup` | Creates account, workspace, initial scan quota. | None |
| GET | `/api/workspaces` | List workspaces for account. | JWT |
| POST | `/api/workspaces` | Create new workspace. | JWT |
| POST | `/api/workspaces/{id}/scans` | Launch AI visibility scan (domain + competitors + question depth). | JWT |
| GET | `/api/scans/{id}` | Fetch scan status, summary metrics, transcripts. | JWT |
| GET | `/api/workspaces/{id}/scans` | Paginated history with trend metrics. | JWT |
| GET | `/api/workspaces/{id}/competitors` | Fetch competitor profiles & scores. | JWT |
| POST | `/api/workspaces/{id}/competitors` | Add competitor or trigger auto-detect. | JWT |
| GET | `/api/workspaces/{id}/question-opportunities` | Question mining results. | JWT |
| POST | `/api/workspaces/{id}/ai-sitemaps` | Generate sitemap from latest scan. | JWT |
| GET | `/api/workspaces/{id}/recommendations` | Prioritized action items. | JWT |
| POST | `/api/reports` | Generate shareable PDF/URL. | JWT |
| WS | `/ws/scans/{id}` | Live updates on scan progress/events. | JWT |

All endpoints enforce plan entitlements via middleware.

---

## 6. Backend Workflows

### AI Visibility Scan Workflow
1. Validate plan limits, create `scan` row, enqueue job in BullMQ/Celery.
2. Job step: crawl provided domain to extract core pages, metadata, and structured data.
3. Generate buyer-intent question set using embeddings + industry templates (50–200 questions).
4. For each LLM vendor, run asynchronous prompt batches with rate-limit/backoff; persist raw responses.
5. Post-processing pipeline extracts mentions, sentiment (VADER or OpenAI), hallucination detection (cross-check vs knowledge graph & website data), and calculates scores.
6. Compute competitor ranking by comparing mention frequencies and sentiment.
7. Persist results, trigger recommendation engine, update scan status to complete, emit WebSocket events.

### Competitor Auto-Detection Workflow
- Use Clearbit/SimilarWeb APIs + embeddings to infer competitors from industry keywords and site text.
- Store top suggestions, allow user confirmation before scanning.

### Question Mining Workflow
- Combine scraped content, competitor mentions, and vector search of industry corpora.
- Use LLM reasoning to cluster into Pain, Use Case, Objection, Feature, Trend categories.
- Score questions by novelty, buyer-stage coverage, and frequency.

### AI Sitemap Builder Workflow
- Map key entities/pages, create summaries, generate FAQ schema and JSON-LD.
- Provide both downloadable file and API endpoint to push to CMS.

### Recommendation Engine Workflow
- Rules engine + LLM reasoning: evaluate sentiment dips, hallucination flags, missing questions.
- Output prioritized items with severity, rationale, suggested assets, estimated effort.

---

## 7. Frontend Wireframes & UI/UX Flows

### Onboarding Flow
1. **Step 1**: Enter domain & company name.
2. **Step 2**: Select industry + target audience.
3. **Step 3**: Add competitors or auto-detect suggestions.
4. **Step 4**: Choose scan depth (questions per model) and frequency.
5. **Confirmation**: Show estimated credits usage, start scan, display progress modal.

### Dashboard Wireframes (descriptions)
- **Overview Tab**: Hero metric cards (Visibility Score, Sentiment Score, Hallucinations, Question Coverage). Trend spark-lines, "Latest AI Quotes" carousel, scan status timeline.
- **Competitors Tab**: Radar chart comparing scores, stacked bar for share-of-voice per LLM, table of positioning statements.
- **Questions Tab**: Heatmap of buyer-stage coverage, clustered cards of top questions, CTA to publish FAQs.
- **AI Sitemap Tab**: Tree view of sitemap, entity chips, copy-to-clipboard schema, CMS integration button.
- **Recommendations Tab**: Kanban lanes (Red/Yellow/Green), filters by owner/impact, quick-mark as done.
- **Reports Tab**: Download/export list with before/after graphs.

### UI Patterns
- Use Next.js App Router, Tailwind, Tremor/Recharts for visualizations.
- Provide skeleton loaders during scan processing.
- Live toast notifications for scan progress & completion.

---

## 8. Example Dashboards & Visualizations
1. **Visibility Over Time**: Line chart per LLM, toggles for self vs competitors.
2. **AI Sentiment Tone Map**: Quadrant chart (positive/negative vs certainty).
3. **Hallucination Heatmap**: Matrix of LLM vs question cluster with severity color.
4. **Competitive Ranking Table**: Shows total mentions, net sentiment, recommended actions.
5. **Question Funnel**: Sankey diagram from Awareness → Consideration → Decision coverage.
6. **Recommendation Impact Matrix**: Bubble chart plotting impact vs effort.

---

## 9. Sample Prompt Chains

### Buyer-Intent Question Generation
- Input: industry, ICP, crawled page summaries, competitor names.
- Prompt: "Generate 150 unique buyer-intent questions a {{industry}} buyer would ask AI when researching {{solution}}. Tag each with buyer stage (Awareness/Consideration/Decision) and intent tags (Pain, Use Case, Objection). Return JSON." (Run via GPT-4.1.)

### Visibility Scan Prompt
- Prompt Template (per question & LLM):
```
You are evaluating solution providers for {{industry_problem}}.
Question: {{question_text}}
Please answer as you would to a buyer. Mention specific vendors only if you are confident. Cite why you recommend them.
```
- Post-process to detect whether target brand appears and extract quotes.

### Hallucination Detection
- Prompt: "Given the official company knowledge base below, does the AI answer contain inaccurate statements about {{brand}}?" Provide structured JSON with `hallucination_flag`, `reason`, `severity`.

### Recommendation Generation
- Prompt: "Here are scan metrics, question gaps, and hallucinations. Propose prioritized actions grouped by severity (Red/Yellow/Green) with impact & effort scores." Run via Claude 3.7 Sonnet for nuanced strategy.

---

## 10. User Stories & Acceptance Criteria

1. **Workspace Owner Runs Initial Scan**
   - *Given* I have entered a domain and competitors
   - *When* I click "Run Scan"
   - *Then* the system should queue a job, show progress, and within SLA provide visibility, sentiment, and hallucination metrics.

2. **Marketer Reviews Competitor Comparison**
   - *Given* a scan is complete
   - *When* I open the Competitors tab
   - *Then* I see ranked scores per LLM, quotes highlighting competitor positioning, and keyword gaps with export options.

3. **Content Lead Mines Questions**
   - *Given* question mining results exist
   - *When* I filter by buyer stage = Decision
   - *Then* I receive a prioritized list of objections with suggested content assets.

4. **Consultant Shares Report**
   - *Given* I have run at least one scan
   - *When* I click "Share Report"
   - *Then* the system generates a secure link + PDF with visibility trends, competitor charts, and recommendations.

5. **Ops Admin Sets Recurring Scans**
   - *Given* I am on Growth plan or higher
   - *When* I schedule monthly scans
   - *Then* jobs auto-run, usage is logged, and notifications summarize deltas vs prior scan.

Acceptance criteria include role-based access, error states (insufficient credits), and instrumentation (audit logs).

---

## 11. Roadmap Phases

1. **Phase 1 – MVP (Months 0-2)**
   - Core onboarding, single LLM (GPT-4.1), 50-question scans, basic scoring, PDF export.
2. **Phase 2 – Multi-LLM & Competitors (Months 2-4)**
   - Add Claude/Gemini/Copilot, competitor module, sentiment analysis, shareable dashboards.
3. **Phase 3 – Question Mining & AI Sitemap (Months 4-6)**
   - Embedding store, question clustering, sitemap builder, FAQ schema outputs.
4. **Phase 4 – Recommendation Engine & Automation (Months 6-8)**
   - Rule-based + LLM recommendations, recurring scans, Slack/email alerts.
5. **Phase 5 – Enterprise & Ecosystem (Months 8-12)**
   - SSO, granular RBAC, agency multi-client portals, integrations (HubSpot, Webflow), marketplace for service partners.

---

## 12. Engineering Task Backlog (Sample)

1. **Implement Multi-Tenant Auth (NextAuth + Supabase)**
   - Tasks: Configure RLS policies, JWT session handling, workspace switching UI.
2. **Scan Job Orchestrator**
   - Tasks: Define scan state machine, BullMQ queues, retry/backoff logic, token accounting.
3. **LLM Connector Library**
   - Tasks: Abstractions for OpenAI, Anthropic, Google APIs; streaming support; per-model prompt templates.
4. **Question Mining Pipeline**
   - Tasks: Crawl site, embed content, cluster with HDBSCAN, LLM labeling, persistence to `question_opportunities`.
5. **Competitor Comparison Charts**
   - Tasks: API endpoints for aggregated stats, frontend visual components, download/export.
6. **Recommendation Engine**
   - Tasks: Rule definitions, LLM summarizer, action item persistence, Kanban UI interactions.
7. **Billing & Usage Tracking**
   - Tasks: Stripe plans, webhook handler, plan enforcement middleware, usage dashboard.
8. **Reporting Service**
   - Tasks: Generate PDF via React PDF, signed URLs for share links, before/after graph rendering.

---

## 13. Pricing & Packaging Suggestions
- **Starter ($29/mo)**: 1 domain, 1 competitor, monthly scans, 50 questions, single LLM.
- **Growth ($99/mo)**: 3 domains, 3 competitors each, bi-weekly scans, 100 questions, multi-LLM, question mining.
- **Pro ($299/mo)**: 10 domains, 10 competitors, weekly scans, 200 questions, AI sitemap, recommendation engine, white-label PDF.
- **Enterprise ($999+/mo)**: Unlimited domains, hourly priority queue, custom integrations, SSO, dedicated success manager.
- Free trial: 1 scan, limited competitors, partial dashboard.

---

## 14. Compliance & Observability
- SOC2-ready logging via OpenTelemetry, retention 30 days.
- Data encryption in transit (TLS) and at rest (KMS-managed keys).
- Audit logs for every scan + report generation.
- Rate-limiting + anomaly detection on scan requests to prevent abuse.

---

This blueprint serves as the master reference for implementation across product, design, and engineering teams delivering the AEO / LLM Visibility SaaS platform.
