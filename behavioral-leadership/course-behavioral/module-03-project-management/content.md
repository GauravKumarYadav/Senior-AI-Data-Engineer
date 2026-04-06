# Module 3: Project Management & Culture

> **Goal:** Master the project management behaviors and cultural fluency that separate senior engineers from mid-level ones in behavioral interviews. You're not just building pipelines — you're leading work, managing humans, and fitting into organizations.

---

## 1. Estimating Timelines for Data Engineering Projects

### Why DE Estimates Are Notoriously Hard

Every engineering discipline struggles with estimation. Data engineering is worse. Here's why:

- **Data quality is a black box until you open it.** You won't know the source has 40% null values in a "required" field until you're knee-deep in the integration. No amount of documentation saves you — the docs are always wrong or outdated.
- **Schema changes are silent killers.** An upstream team pushes a schema migration on Friday at 5pm. Your pipeline breaks Saturday morning. You didn't budget for this because nobody told you it was coming.
- **Vendor dependencies are outside your control.** Waiting on API access from a third-party vendor? That "2 business days" turns into 3 weeks with legal review, security assessment, and a procurement process nobody mentioned.
- **Requirements are vague.** "We need a dashboard for customer churn" sounds simple until you discover there are 7 definitions of "churn" across 4 teams and nobody agrees.
- **Infrastructure provisioning has hidden lead times.** Need a new Kafka cluster? A dedicated Airflow environment? Cloud permissions? Each one has its own approval chain.

> **Coaching tip:** In interviews, demonstrate that you *know* estimation is hard and that you have a *system* for dealing with it. Saying "I've been burned before so I pad my estimates" is weak. Saying "I use three-point estimation and break projects into phases with explicit unknowns" is strong.

### The 3-Point Estimation Technique

This is your go-to framework. For every task, generate three numbers:

| Estimate Type | Definition | When It Happens |
|---|---|---|
| **Optimistic (O)** | Everything goes right. No surprises. | Almost never. |
| **Most Likely (M)** | Reasonable assumptions hold. Normal friction. | This is your gut estimate. |
| **Pessimistic (P)** | Major blockers hit. Dependencies slip. Rework needed. | More often than you think. |

**The formula:** `Expected = (O + 4M + P) / 6`

**Example:**
- Building an ingestion pipeline from a new REST API
- Optimistic: 5 days (API is clean, schema is stable, auth is straightforward)
- Most Likely: 10 days (some data quality issues, need to handle pagination, rate limiting)
- Pessimistic: 20 days (API docs are wrong, need multiple rounds with vendor, schema changes mid-build)
- Expected: `(5 + 40 + 20) / 6 ≈ 11 days`

### Adding Buffers: The 1.5x Rule for Unknowns

After your 3-point estimate, apply a multiplier based on uncertainty:

| Uncertainty Level | Multiplier | When to Use |
|---|---|---|
| **Low** — you've done this exact thing before | 1.0x – 1.2x | Re-platforming a pipeline you built |
| **Medium** — familiar domain, some unknowns | 1.3x – 1.5x | New source, familiar tools |
| **High** — new domain, new tools, new team | 1.5x – 2.0x | Greenfield project with new stack |

> **Coaching tip:** Never present the buffer as "padding." Frame it as "accounting for integration risk and unknowns." The former sounds lazy. The latter sounds experienced.

### Breaking Estimates into Phases

Don't give one big number. Break every project into phases and estimate each independently:

1. **Discovery (5-15% of total):** Understand the source systems, data contracts, stakeholder requirements. Talk to people. Read docs. Profile data.
2. **Design (10-15%):** Architecture decisions, schema design, tech selection. Write a design doc. Get review.
3. **Implementation (40-50%):** The actual coding. This is the part engineers over-index on when estimating.
4. **Testing (15-20%):** Unit tests, integration tests, data quality validation, load testing. This is the part engineers *under*-index on.
5. **Rollout (10-15%):** Deployment, monitoring setup, runbook creation, stakeholder training, shadow mode / canary period.

**The trap:** Engineers estimate implementation time and call it the project estimate. Testing and rollout are where the surprises live.

### Communicating Estimates with Confidence Ranges

**Bad:** "It'll take 2 weeks."
**Good:** "I'm estimating 2-3 weeks. The lower end assumes the API is well-documented and we get access by Monday. The upper end accounts for data quality issues we typically see with new sources. I'll have a sharper estimate after the 2-day discovery phase."

This does three things:
1. Sets realistic expectations
2. Shows you've thought about risks
3. Gives you a natural checkpoint to refine the estimate

### Worked Example: Estimating a New Data Pipeline

**Project:** Build a pipeline from a SaaS CRM (source) → data lake → transformed models → executive dashboard.

| Phase | Optimistic | Likely | Pessimistic | Expected |
|---|---|---|---|---|
| Discovery & data profiling | 2d | 3d | 5d | 3d |
| Schema design & data contracts | 2d | 3d | 5d | 3d |
| Ingestion pipeline (API → lake) | 3d | 5d | 10d | 6d |
| Transformation layer (dbt/Spark) | 3d | 5d | 8d | 5d |
| Dashboard / semantic layer | 2d | 3d | 5d | 3d |
| Testing & validation | 2d | 4d | 7d | 4d |
| Deployment & monitoring | 1d | 2d | 4d | 2d |
| **Total** | **15d** | **25d** | **44d** | **26d** |

**What you'd communicate:** "I'm estimating 5-6 weeks for end-to-end delivery, with a working prototype by week 3. The biggest risk is the CRM API — I need to validate data quality and rate limits during discovery. I'll update the estimate after the first week."

---

## 2. Breaking Down Large Projects into Milestones

### The "Thin Vertical Slice" Approach

The single most important concept for senior engineers: **deliver end-to-end value as early as possible.**

A "thin vertical slice" means building a narrow but complete path through the entire system. Instead of building all the ingestion, then all the transformation, then all the serving — you build *one pipeline, one model, one output* end-to-end first.

**Why this matters:**
- You uncover integration issues in week 1, not week 8
- Stakeholders see something real and can give feedback
- You build confidence with leadership that the project is on track
- You can demo progress, which is the #1 way to maintain buy-in

**Anti-pattern:** "We spent 6 weeks building the ingestion layer for all 12 sources. Then we discovered the transformation logic needed a different grain. We had to re-do most of the ingestion."

**Better:** "We picked the highest-priority source, built ingestion → transformation → output in 2 weeks. Validated the pattern. Then parallelized the remaining sources."

### Milestone Structure

Use this 5-stage structure for any large DE project:

**Stage 1: Discovery (1-2 weeks)**
- Stakeholder interviews and requirements gathering
- Source system analysis and data profiling
- Technical feasibility assessment
- Risk identification
- **Done when:** You have a written project brief and a preliminary architecture sketch

**Stage 2: Prototype (1-2 weeks)**
- Build the thin vertical slice
- One source → one transformation → one output
- Hacky is fine. Hard-coded values are fine. The goal is to prove the pattern.
- **Done when:** You can demo a working end-to-end flow with real data (even if it's ugly)

**Stage 3: MVP (2-4 weeks)**
- Production-quality code for the core use case
- Error handling, logging, monitoring
- Core data quality checks
- Enough documentation for someone else to understand it
- **Done when:** The primary stakeholder can use the output for their actual work

**Stage 4: Hardening (1-2 weeks)**
- Add remaining sources / dimensions
- Performance optimization
- Alerting and runbooks
- Edge case handling
- Load testing
- **Done when:** You'd be comfortable going on vacation and having someone else on-call

**Stage 5: Launch & Handoff (1 week)**
- Stakeholder training
- Documentation finalized
- On-call rotation set up
- Retrospective
- **Done when:** The system runs without you for 2 weeks

### Creating a Project Brief / One-Pager

Every project over 2 weeks deserves a one-pager. Here's the template:

```
# Project Brief: [Project Name]

## Problem Statement
[2-3 sentences. What's broken? What's missing? What's the cost of inaction?]

## Proposed Solution
[2-3 sentences. What are we building? High-level approach.]

## Success Criteria
- [ ] [Measurable outcome 1]
- [ ] [Measurable outcome 2]
- [ ] [Measurable outcome 3]

## Scope
**In scope:** [explicit list]
**Out of scope:** [explicit list — this is the most important section]

## Milestones
| Milestone | Target Date | Deliverable |
|-----------|------------|-------------|
| Discovery complete | Week 1 | Project brief + architecture sketch |
| Prototype demo | Week 3 | End-to-end flow with 1 source |
| MVP | Week 6 | Production pipeline with core sources |
| Launch | Week 8 | Full deployment with monitoring |

## Risks
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| [Risk 1] | High/Med/Low | High/Med/Low | [What you'll do] |

## Team & Stakeholders
- **Owner:** [you]
- **Stakeholders:** [who cares about this]
- **Dependencies:** [other teams / systems]
```

> **Coaching tip:** Bring a version of this template to your interview prep. When asked "how do you approach a large project?", walking through this structure demonstrates senior-level thinking.

### Worked Example: "Build a Real-Time Fraud Detection Pipeline"

**Milestone 1 — Discovery (Week 1-2)**
- Interview the fraud ops team: what signals matter? What's the current process?
- Profile transaction data: volume, velocity, schema, quality
- Evaluate streaming infrastructure: Kafka vs. Kinesis vs. Pub/Sub
- Identify ML model requirements (batch scoring vs. real-time inference)
- **Deliverable:** Project brief, architecture diagram, risk register

**Milestone 2 — Prototype (Week 3-4)**
- Set up streaming ingestion for one transaction source
- Implement a simple rule-based fraud flag (not ML yet — just "transactions > $10K from new accounts")
- Write results to a staging table
- Build a basic Grafana dashboard showing flagged transactions
- **Deliverable:** Working demo with real (or realistic) data, live in a dev environment

**Milestone 3 — MVP (Week 5-8)**
- Integrate the ML model for scoring (batch-trained, real-time inference)
- Add all priority transaction sources
- Implement proper schema validation and dead-letter queues
- Build alerting for the fraud ops team (Slack/PagerDuty)
- Data quality checks: latency SLA, completeness, freshness
- **Deliverable:** System handling live traffic in shadow mode (scoring but not blocking)

**Milestone 4 — Hardening (Week 9-10)**
- Load testing at 3x expected peak volume
- Chaos engineering: what happens when Kafka goes down? When the model service is slow?
- Backfill capability for reprocessing historical transactions
- Runbook for on-call engineers
- Performance tuning: p99 latency under SLA
- **Deliverable:** System passes load test and failure scenarios

**Milestone 5 — Launch (Week 11-12)**
- Gradual rollout: 10% → 50% → 100% of traffic
- Fraud ops team training session
- Monitoring dashboard reviewed with SRE team
- Retrospective document
- **Deliverable:** System live in production, handling 100% of traffic, fraud ops team self-sufficient

---

## 3. Handling Scope Creep

### What Scope Creep Looks Like in Data Engineering

Scope creep in DE is insidious because each individual request sounds small and reasonable:

- "Can we add one more data source?" (That source has a completely different schema and update cadence.)
- "Can we add one more metric to the dashboard?" (That metric requires joining 3 new tables and defining business logic nobody has agreed on.)
- "Can we also backfill the last 2 years?" (That's a fundamentally different engineering problem than forward-looking ingestion.)
- "Can we make it real-time instead of daily?" (You just changed the architecture from batch to streaming.)
- "Can the same pipeline also serve the ML team?" (Different SLAs, different schemas, different freshness requirements.)

Each one adds 1-3 days *minimum*. Stack five of them and your 6-week project is now 10 weeks — but nobody adjusted the deadline.

### The "Yes, And Here's the Trade-Off" Framework

Never say "no" to a stakeholder. Say "yes, and here's what that means":

**Structure:**
1. Acknowledge the value of the request
2. Explain the impact on scope/timeline/resources
3. Present options
4. Let the stakeholder make the call

**Example:**

> **PM:** "Can we add the mobile app events as a source? The product team really wants it."
>
> **You:** "Absolutely, that would make the analysis much richer. Here's what adding mobile events means: it's a different event schema from the web events, so I'll need 3-4 days for schema mapping and testing. We have two options:
> 1. Add it now and push the launch date from March 15 to March 22.
> 2. Launch on March 15 with web events only, then add mobile in a fast follow the week after.
>
> Which would you prefer?"

**Why this works:**
- You're not the blocker. You're the expert providing information.
- The stakeholder owns the prioritization decision.
- Everything is documented (follow up the conversation with a Slack message or email).

### Documenting Scope Changes

Maintain a **scope change log.** It doesn't have to be fancy:

| Date | Requested By | Change | Impact | Decision |
|---|---|---|---|---|
| Feb 10 | PM | Add mobile source | +4 days | Approved; new deadline Mar 22 |
| Feb 18 | Analytics | Add cohort metrics | +3 days | Deferred to Phase 2 |
| Feb 25 | VP | Real-time refresh | +2 weeks | Declined; daily refresh meets needs |

> **Coaching tip:** In interviews, mentioning that you keep a scope change log signals senior maturity. It shows you understand that projects don't fail because of one big mistake — they fail because of 20 small, undocumented scope expansions.

### When to Push Back vs. When to Absorb

**Absorb when:**
- The change is genuinely small (< 1 day of work)
- It doesn't change the architecture
- You have buffer in your estimate
- Saying no would damage an important relationship for negligible gain

**Push back when:**
- The change alters the project's fundamental approach
- It introduces a new dependency or risk
- The cumulative scope changes have already consumed your buffer
- The request comes from someone who isn't the project's primary stakeholder
- Absorbing it would compromise quality or your team's sustainability

### Example STAR Story: Managing Scope Creep

> **Situation:** I was leading a project to build a customer 360 data model for the marketing analytics team. Original scope was 5 data sources, daily refresh, 8-week timeline.
>
> **Task:** By week 3, we had received 6 additional requests: 3 more data sources, real-time refresh for one table, a new "predicted LTV" metric, and a self-serve query layer. Each request came from a different stakeholder, each one "critical."
>
> **Action:** I called a scope review meeting with all stakeholders and the PM. I prepared a one-page document showing: original scope, all requested additions, the cumulative impact (original 8 weeks → estimated 14 weeks), and a proposed phased plan. Phase 1 (weeks 1-8): original 5 sources + the 2 most valuable new ones. Phase 2 (weeks 9-11): remaining sources and the predicted LTV metric. Phase 3 (weeks 12-14): real-time refresh and self-serve query layer. I asked each stakeholder to rank their requests by business impact.
>
> **Result:** The group agreed on the phased plan. We launched Phase 1 on time. Two of the Phase 3 requests were ultimately deprioritized because the business need changed. By being transparent about trade-offs, I avoided a 14-week death march and delivered the highest-value work first. The PM later told me that scope review meeting was "the most productive 30 minutes of the quarter."

---

## 4. Managing Stakeholder Expectations

### Identifying Your Stakeholders

Not all stakeholders are equal. Map them:

| Stakeholder | What They Care About | Communication Style |
|---|---|---|
| **Product Manager** | On-time delivery, feature completeness | Weekly syncs, JIRA updates |
| **Analytics / Data Science** | Data quality, schema stability, freshness | Slack, data contracts, schema docs |
| **Engineering Leadership** | Risk, resource allocation, technical debt | Bi-weekly updates, design reviews |
| **Downstream Consumers** | API stability, SLAs, breaking changes | Changelogs, migration guides |
| **Executives** | Business outcomes, timelines, blockers | Monthly summaries, red/yellow/green status |

### Setting Expectations Early

The first two weeks of a project determine whether stakeholders trust you for the remaining ten. Do these things immediately:

1. **Send a kickoff document.** Use the one-pager template from Section 2. Share it *before* the kickoff meeting so people come prepared.
2. **Establish a communication cadence.** "I'll send a weekly update every Friday by 3pm. It'll cover: what we shipped, what's coming next week, any blockers or risks."
3. **Define "done" explicitly.** "The MVP is done when the marketing team can query customer segments by these 5 dimensions in Looker, with data refreshed daily by 6am PT."
4. **Set a demo cadence.** "I'll demo progress every two weeks. The first demo will be a prototype with one data source — it'll be ugly but functional."

> **Coaching tip:** The weekly update email is the single highest-ROI habit for stakeholder management. Five minutes of writing saves hours of "is it done yet?" interruptions.

### Handling the "Is It Done Yet?" Pressure

When a stakeholder asks "is it done yet?" it means one of two things:
1. Your communication isn't frequent enough
2. They're under pressure from their own stakeholders

**Response template:**

> "We're on track for the [date] milestone. Specifically, we completed [X] this week and we're working on [Y] next week. The main risk right now is [Z] — here's how I'm mitigating it. Want me to add you to the weekly update email?"

This is better than "almost done" or "a few more days" because:
- It's specific
- It acknowledges risk
- It offers a systemic fix (the weekly update) rather than a point-in-time answer

### Bad News Delivery Framework

Bad news doesn't age well. Deliver it **early, honestly, and with a plan.**

**The formula: Problem → Impact → Plan → Ask**

1. **Problem:** State the issue clearly. No hedging.
2. **Impact:** What does this mean for the timeline, scope, or quality?
3. **Plan:** What are you doing about it? Present options.
4. **Ask:** What do you need from the stakeholder?

**Example:**

> "I need to flag a risk on the customer 360 project. [Problem] The vendor API has a rate limit we didn't know about — we can only pull 1,000 records per minute, and we need to ingest 5 million records daily. [Impact] With the current approach, the daily sync would take over 80 hours, which obviously doesn't work. [Plan] I'm evaluating two alternatives: (a) requesting a rate limit increase from the vendor — I've filed a ticket and expect a response by Thursday, or (b) switching to their bulk export endpoint, which would require about 3 days of rework. [Ask] Can you help escalate the rate limit request through your vendor relationship? And should I start the bulk export work in parallel as a backup?"

**Anti-patterns for bad news:**
- ❌ Waiting until the deadline to mention it
- ❌ Burying it in a status update
- ❌ Framing it as someone else's fault
- ❌ Presenting the problem without options
- ❌ Downplaying the impact ("it's probably fine")

### Building Trust Through Transparency and Predictability

Trust isn't built in grand gestures. It's built in small, consistent actions:

- **Do what you said you'd do.** If you said "update by Friday," send the update by Friday — even if the update is "no progress this week, here's why."
- **Under-promise and over-deliver.** If you think it'll take 8 days, say 10. Delivering early feels good for everyone.
- **Share your uncertainty.** "I'm 70% confident we'll hit the March deadline. The 30% risk is the vendor integration." This is more trustworthy than false certainty.
- **Admit mistakes quickly.** "I underestimated the complexity of the transformation layer. Here's my revised estimate and what I learned."
- **Give credit generously.** "The pipeline is reliable because [teammate] built an excellent monitoring framework."

---

## 5. Amazon Leadership Principles (Detailed Breakdown)

### All 16 Principles at a Glance

| # | Principle | One-Liner |
|---|---|---|
| 1 | **Customer Obsession** | Start with the customer and work backwards |
| 2 | **Ownership** | Think long-term; never say "that's not my job" |
| 3 | **Invent and Simplify** | Find new ways and simplify processes |
| 4 | **Are Right, A Lot** | Good instincts and judgment |
| 5 | **Learn and Be Curious** | Never stop learning; explore new possibilities |
| 6 | **Hire and Develop the Best** | Raise the bar with every hire |
| 7 | **Insist on the Highest Standards** | Continually raise the bar on quality |
| 8 | **Think Big** | Bold direction inspires results |
| 9 | **Bias for Action** | Speed matters; calculated risk-taking |
| 10 | **Frugality** | Do more with less; constraints breed innovation |
| 11 | **Earn Trust** | Listen, speak candidly, treat others with respect |
| 12 | **Dive Deep** | Stay connected to details; audit frequently |
| 13 | **Have Backbone; Disagree and Commit** | Challenge decisions, then commit fully |
| 14 | **Deliver Results** | Focus on key inputs and deliver with quality |
| 15 | **Strive to be Earth's Best Employer** | Create a safe, productive, empathetic workplace |
| 16 | **Success and Scale Bring Broad Responsibility** | Be better every day for customers and communities |

### The Top 6 — Deep Dive

These are the principles that come up most frequently in interviews. Prepare 2-3 stories for each.

---

#### 1. Customer Obsession

**What they're looking for:** You start with the customer need and work backwards to the technical solution. You don't build things because they're cool — you build them because they solve a real problem.

**Example question:** "Tell me about a time you went above and beyond for an internal or external customer."

**Strong answer (STAR):**

> **S:** Our analytics team was spending 4 hours every Monday morning manually reconciling data between two dashboards that showed different numbers for the same metric.
> **T:** I took it upon myself to investigate and fix the discrepancy, even though it wasn't in my sprint.
> **A:** I traced the issue to a timezone handling bug in one pipeline and a different aggregation grain in the other. I fixed both pipelines, added a reconciliation check that runs daily, and set up an alert if the dashboards ever diverge by more than 0.1%. I also created a data dictionary so the team could self-serve.
> **R:** Eliminated 200+ hours/year of manual work. The analytics lead said it was "the most impactful thing anyone did for her team that quarter." The reconciliation pattern was adopted by 3 other teams.

**Anti-patterns:**
- ❌ Talking about technical decisions without connecting them to customer impact
- ❌ Defining "customer" only as external end-users (internal teams count)
- ❌ Describing a solution you thought was cool but nobody asked for

---

#### 2. Ownership

**What they're looking for:** You take responsibility for outcomes, not just your assigned tasks. You think about the full lifecycle — not just shipping, but monitoring, maintaining, and eventually decommissioning.

**Example question:** "Tell me about a time you took on something outside your area of responsibility."

**Strong answer (STAR):**

> **S:** I noticed our data pipeline costs had been increasing 15% month-over-month for 6 months, but nobody owned cost optimization — it fell between the DE and infra teams.
> **T:** I decided to own it even though it wasn't my formal responsibility.
> **A:** I audited our Spark jobs and found 3 major issues: oversized clusters for small jobs, no partition pruning on our largest tables, and redundant intermediate materializations. I implemented right-sizing automation, added partition filters, and consolidated the redundant jobs. I also set up a weekly cost dashboard and a Slack alert for any job exceeding its cost baseline by 20%.
> **R:** Reduced monthly compute costs by 40% ($18K/month savings). The cost dashboard became a standard tool for the team, and I handed off ownership to a new hire I mentored on the process.

**Anti-patterns:**
- ❌ Blaming other teams for problems
- ❌ Saying "I was told to do X" instead of "I decided to do X"
- ❌ Stories where you identified a problem but didn't act on it

---

#### 3. Bias for Action

**What they're looking for:** You move quickly on decisions that are reversible. You don't wait for perfect information. You experiment, iterate, and course-correct.

**Example question:** "Tell me about a time you made a decision with incomplete information."

**Strong answer (STAR):**

> **S:** We discovered on a Friday afternoon that a critical data source had changed its API response format without notice, breaking our daily pipeline that feeds the executive KPI dashboard viewed every Monday morning.
> **T:** I had to get accurate data flowing before Monday. I had incomplete information about whether this was a temporary glitch or a permanent change.
> **A:** Rather than waiting to hear back from the vendor (who wouldn't respond until Monday), I built a compatibility adapter that could handle both the old and new format. I deployed it that evening with feature flags so we could switch between formats without redeployment. I also added schema validation alerts so we'd catch any future breaking changes within minutes.
> **R:** The Monday dashboard was accurate and on time. When the vendor confirmed Monday that the change was permanent, we simply toggled the feature flag. Total downtime: zero. The schema validation pattern became standard for all our external source integrations.

**Anti-patterns:**
- ❌ Stories where you waited too long and missed a window
- ❌ Reckless decisions without any risk assessment
- ❌ Not mentioning how you mitigated the risk of acting fast

---

#### 4. Dive Deep

**What they're looking for:** You don't just accept surface-level explanations. You dig into the data, read the code, check the assumptions. You're comfortable operating at multiple levels of detail.

**Example question:** "Tell me about a time you had to dive deep to find the root cause of a problem."

**Strong answer (STAR):**

> **S:** Our ML team reported that their churn prediction model's accuracy had dropped from 87% to 71% over two weeks, and they couldn't figure out why. They assumed their feature engineering was wrong.
> **T:** As the DE who owned the feature pipeline, I took responsibility for investigating the data side.
> **A:** I compared feature distributions week-over-week and found that one key feature — "days since last purchase" — had shifted dramatically. I traced it back through the pipeline and discovered that a marketing team had launched a re-engagement campaign that generated synthetic "engagement events" that our pipeline was incorrectly counting as purchases. I fixed the event classification logic, added a data quality test that monitors feature distribution drift, and set up a cross-team notification process for campaigns that might affect downstream data.
> **R:** Model accuracy returned to 88% within one retraining cycle. The distribution drift monitor caught two similar issues in the following quarter before they impacted models. The ML team lead said the cross-team process "saved them weeks of debugging."

**Anti-patterns:**
- ❌ Staying at a high level and not showing technical depth
- ❌ Relying on someone else to investigate for you
- ❌ Not explaining *how* you debugged (tools, queries, process)

---

#### 5. Have Backbone; Disagree and Commit

**What they're looking for:** You respectfully challenge decisions you disagree with, using data and logic. But once a decision is made, you commit fully — no passive resistance, no "I told you so."

**Example question:** "Tell me about a time you disagreed with your manager or a senior stakeholder."

**Strong answer (STAR):**

> **S:** My manager wanted us to migrate our entire data warehouse from Redshift to BigQuery in a single "big bang" cutover during a 3-day weekend. I believed this was extremely high-risk.
> **T:** I needed to advocate for a phased migration while respecting that the decision was ultimately my manager's.
> **A:** I put together a one-page risk analysis comparing big-bang vs. phased migration. I identified 4 specific failure modes of the big-bang approach: query incompatibilities we hadn't tested, performance characteristics we hadn't benchmarked, downstream consumer dependencies we hadn't mapped, and rollback complexity. I proposed a 6-week phased approach: migrate read-only workloads first, then ETL, then serve both in parallel for a week, then cut over. I presented this to my manager and the team in a 15-minute meeting with data, not opinions.
> **R:** My manager was initially resistant but agreed to a compromise — a 3-week phased approach with a parallel-running validation period. During the parallel run, we discovered 12 query incompatibilities that would have caused a major outage in a big-bang cutover. My manager acknowledged the approach saved us and later used it as a template for future migrations.

**Anti-patterns:**
- ❌ Disagreeing without data or alternatives
- ❌ Not committing after the decision was made
- ❌ Passive-aggressive compliance ("I did it their way and it failed, just like I said")
- ❌ Only disagreeing with people below you in the hierarchy

---

#### 6. Deliver Results

**What they're looking for:** You focus on outcomes, not activity. You hit deadlines. When obstacles arise, you find a way through. You measure success quantitatively.

**Example question:** "Tell me about a time you delivered a project under tight constraints."

**Strong answer (STAR):**

> **S:** We had a regulatory deadline: all PII data in our warehouse had to be encrypted at rest and tokenized in transit within 6 weeks. Failure meant potential fines and audit findings.
> **T:** I was the lead DE on a 3-person team. The scope was massive: 200+ tables, 40+ pipelines, dozens of downstream consumers.
> **A:** I prioritized ruthlessly. I categorized all tables by PII exposure level (high/medium/low) and by downstream consumer count. I focused the first 3 weeks on the 30 high-risk, high-visibility tables. I automated the encryption migration using a metadata-driven approach — instead of modifying 200 pipelines individually, I built a transformer layer that applied encryption rules based on a column-level PII registry. For the remaining tables, I implemented a "encrypt-on-read" pattern that bought us time. I gave daily 5-minute standups to the compliance team so they could see progress in real-time.
> **R:** We encrypted 100% of high-risk tables by week 4 and completed the remaining tables by week 6. Zero downstream breakages. Passed the audit with no findings. The metadata-driven encryption approach was adopted org-wide and is still in use. The compliance VP called out our team in the quarterly all-hands.

**Anti-patterns:**
- ❌ Stories without quantifiable results
- ❌ Focusing on effort instead of outcomes ("I worked 80-hour weeks")
- ❌ Not explaining how you prioritized when everything was "urgent"
- ❌ Missing the deadline in your story (unless the learning is exceptional)

### How to Map Your Stories to Leadership Principles

Each story can map to multiple LPs. Build a matrix:

| Story | Primary LP | Secondary LPs |
|---|---|---|
| Fixed dashboard discrepancy | Customer Obsession | Ownership, Dive Deep |
| Cost optimization initiative | Ownership | Frugality, Deliver Results |
| Friday API break fix | Bias for Action | Customer Obsession, Deliver Results |
| ML feature debugging | Dive Deep | Earn Trust, Customer Obsession |
| Migration approach debate | Disagree and Commit | Insist on Highest Standards |
| PII encryption sprint | Deliver Results | Ownership, Bias for Action |

> **Coaching tip:** Prepare 8-10 strong stories that collectively cover all 16 LPs. You don't need 16 unique stories — each story should naturally demonstrate 2-3 principles. Practice pivoting the same story to emphasize different LPs depending on the question.

---

## 6. Google's "Googleyness"

### What It Means

Google doesn't publish a formal list of "Googleyness" traits, but through interview feedback and internal culture, it comes down to:

- **Collaboration over ego.** You make the team better. You seek out opposing views. You share credit.
- **Intellectual humility.** You admit what you don't know. You change your mind when presented with evidence.
- **Comfort with ambiguity.** You thrive in environments where the problem isn't fully defined. You can make progress without perfect requirements.
- **Doing the right thing.** You have ethical judgment. You push back on things that are wrong, even when it's uncomfortable.
- **Fun to work with.** You bring energy, curiosity, and a sense of humor. People want you on their team.

### How Google Evaluates Culture Fit

Google's interview process includes a "Googleyness & Leadership" (G&L) round. This is distinct from the technical rounds. The interviewer is evaluating:

1. **How you navigate ambiguity** — Can you make progress without all the answers?
2. **How you work with others** — Do you elevate the team or create friction?
3. **How you handle disagreement** — Do you push back constructively?
4. **How you've demonstrated leadership** — Not management, *leadership*. Influence without authority.

### Example Questions and Strong Answers

**Q: "Tell me about a time you had to work with ambiguous requirements."**

> **Strong answer:** "Our VP asked for 'a way to understand customer health' with no further definition. Instead of asking for a full requirements doc, I interviewed 5 stakeholders and discovered they each had different definitions of customer health. I synthesized their needs into a 3-tier framework: engagement health (product usage), financial health (revenue trends), and support health (ticket volume and sentiment). I built a prototype scoring model, demoed it to all 5 stakeholders, and iterated based on their feedback. The ambiguity was actually an advantage — if I'd waited for perfect requirements, we'd still be in meetings."

**Q: "Tell me about a time you helped a teammate succeed."**

> **Strong answer:** "A junior engineer on my team was struggling with a complex Spark optimization task. Instead of taking it over, I paired with them for two hours. I walked them through how to read a Spark execution plan, identify shuffle operations, and use broadcast joins. Then I let them implement the fix independently while I reviewed their work. They reduced the job runtime from 4 hours to 25 minutes. More importantly, they became the team's go-to person for Spark performance, and they presented their learnings at our team's knowledge-sharing session."

### The "General Cognitive Ability" Dimension

Google also evaluates how you think, not just what you've done:

- **Can you structure an ambiguous problem?** Break it down, identify the key constraints, propose an approach.
- **Do you learn from your mistakes?** What did you do differently the next time?
- **Can you evaluate trade-offs?** Not just "I chose X" but "I chose X over Y because of Z, and here's what I'd give up."
- **Do you ask good questions?** In the interview itself, do you clarify requirements before jumping to solutions?

> **Coaching tip:** Google values the *thought process* more than the outcome. Walk the interviewer through how you think. "First I considered... then I realized... so I pivoted to..." is more valuable than "I did X and it worked."

---

## 7. Meta's Values

### The Core Values

| Value | What It Means in Practice |
|---|---|
| **Move Fast** | Ship early, iterate, don't let process slow you down |
| **Be Bold** | Take big swings. The riskiest thing is not taking risks. |
| **Build Social Value** | Your work should create real-world positive impact |
| **Focus on Long-Term Impact** | Don't optimize for short-term metrics at the expense of the future |
| **Be Open** | Transparent communication. Information should flow freely. |

### How Meta Interviews Differ

Meta (interviews often still referenced as "Facebook-style") emphasizes:

- **Impact at scale.** Every story should involve large numbers. Millions of users, petabytes of data, thousands of QPS. If your story involves 100 rows in a spreadsheet, save it for a different company.
- **Speed of execution.** They want to hear about rapid iteration. "We shipped a v1 in 2 weeks, got feedback, and iterated 3 times in the next month."
- **Data-driven decision making.** Every major decision should reference metrics. "We chose approach A because our A/B test showed a 12% improvement in latency at p99."
- **Collaboration across teams.** Meta's codebase is famously open. They want engineers who reach across team boundaries.

### Example Questions and Strong Answers

**Q: "Tell me about the most impactful project you've worked on."**

> **Strong answer:** "I designed and built the real-time data pipeline that powers our product's recommendation engine, serving 15 million daily active users. The pipeline processes 2 billion events per day with a p99 latency of 200ms. The key technical challenge was maintaining data freshness while handling schema evolution across 40+ event types. I implemented a schema registry with backwards compatibility checks and an automated migration system. Impact: the recommendation engine's click-through rate improved by 23% due to fresher data, translating to an estimated $4M in annual revenue. I shipped the initial version in 3 weeks, then spent 6 weeks hardening it."

**Q: "Tell me about a time you moved fast and it paid off."**

> **Strong answer:** "We discovered a data integrity issue affecting billing calculations on a Wednesday. Instead of going through our normal 2-week planning cycle, I proposed a 48-hour fix sprint. I scoped the minimum viable fix, got buy-in from my manager and the billing team lead, and worked with one other engineer to implement it. We deployed the fix Thursday night with a feature flag, validated it on Friday morning using shadow-mode comparison, and fully rolled it out by Friday afternoon. We prevented an estimated $200K in billing errors that would have occurred over the weekend. I then spent the following week doing a proper root cause analysis and implementing permanent safeguards."

> **Coaching tip:** At Meta, quantify everything. Revenue impact, user impact, scale numbers, latency improvements, time saved. If you can't put a number on it, the story isn't strong enough.

---

## 8. Researching Company-Specific Values

### How to Research Any Company's Values

Before every interview, spend 60-90 minutes on company research. This isn't optional. Here's your checklist:

**Step 1: Official Sources (15 min)**
- Company careers page → look for "Our Values" or "Our Culture"
- Company "About" page → mission statement, leadership bios
- Recent earnings calls or shareholder letters → what does leadership prioritize?

**Step 2: Engineering Culture (20 min)**
- Company engineering blog → what do they write about? What do they celebrate?
- Open-source contributions → what have they released? How do they manage community?
- Conference talks by company engineers → what topics do they present on?

**Step 3: Employee Perspectives (15 min)**
- Glassdoor reviews → look for patterns, not individual complaints
- LinkedIn → search for posts by engineers at the company. What do they share?
- Blind (anonymous) → take with a grain of salt, but look for recurring themes

**Step 4: Interview-Specific (15 min)**
- Glassdoor interview questions → what do they actually ask?
- Levels.fyi → compensation data and interview process details
- YouTube → search "[Company] engineering interview" for prep videos

### Adapting Your Stories to Match Company Culture

The same experience can be framed differently for different companies:

| Your Experience | Amazon Framing | Google Framing | Meta Framing |
|---|---|---|---|
| Built a new pipeline | "I identified a customer pain point and worked backwards to build..." | "I navigated ambiguous requirements by collaborating with 5 teams..." | "I shipped a v1 in 2 weeks that processes 1B events/day..." |
| Fixed a production issue | "I took ownership and dove deep into the root cause..." | "I structured the debugging process and helped the team learn..." | "I moved fast — fixed it in 4 hours, preventing $500K in losses..." |
| Proposed a new architecture | "I had backbone and disagreed with the existing approach..." | "I evaluated trade-offs and built consensus across teams..." | "I was bold — proposed replacing our core system and proved ROI..." |

### Template for Company Research Notes

Use this template before every interview:

```
# Company Research: [Company Name]
## Date: [Interview Date]
## Role: [Position]

## Core Values
1. [Value 1] — My story: [which of your stories maps here]
2. [Value 2] — My story: [which of your stories maps here]
3. [Value 3] — My story: [which of your stories maps here]

## Engineering Culture
- Tech stack: [what they use]
- Engineering blog themes: [what they write about]
- Open source: [notable projects]
- Interview style: [coding, system design, behavioral mix]

## Recent News
- [Headline 1] — how this might come up in interview
- [Headline 2] — how this might come up in interview

## Questions I'll Ask Them
1. [About engineering culture]
2. [About the team/role]
3. [About growth/impact]

## Stories I've Prepared
| Story | Primary Value Match | Backup Value Match |
|-------|--------------------|--------------------|
| [Story 1] | [Value] | [Value] |
| [Story 2] | [Value] | [Value] |
| [Story 3] | [Value] | [Value] |
```

---

## 9. Work-Life Balance & Team Dynamics Questions

### How to Answer "How Do You Handle Work-Life Balance?"

This question is tricky because companies want different things. Here's the universal framework:

**The honest, strong answer has three parts:**
1. **I'm intentional about sustainability.** "I believe the best work comes from engineers who are rested and focused, not burned out."
2. **I'm flexible when it matters.** "When there's a production incident or a critical deadline, I'm fully engaged — that's part of the job."
3. **I protect against chronic overwork.** "I push back on unsustainable patterns. If we're consistently working overtime, that's a process or staffing problem, not an individual effort problem."

**Example answer:**

> "I think about work-life balance as work-life *sustainability*. I'm disciplined about my core hours — I do my best deep work in the morning, so I protect that time. I'm flexible for oncall, incidents, and crunch periods — those are part of the job and I lean in hard when they happen. But I also believe persistent overtime is a signal, not a solution. When I notice my team consistently working late, I dig into why: are we over-committed? Do we have a process bottleneck? Is there tech debt slowing us down? I'd rather fix the system than ask people to compensate for a broken system with their personal time."

**What NOT to say:**
- ❌ "I don't believe in work-life balance — I just love coding!" (Red flag for burnout.)
- ❌ "I strictly work 9-5 and never check messages after hours." (Sounds inflexible.)
- ❌ "I work 80-hour weeks regularly." (Sounds unsustainable and possibly inefficient.)

### Team Dynamics Questions

**"What's your preferred team size?"**
> "I've done my best work on teams of 4-6 engineers. Small enough for everyone to stay in sync without heavy process, large enough for code review coverage and knowledge sharing. That said, I've been effective on teams of 2 and teams of 15 — it's more about the communication patterns than the number."

**"Describe your working style."**
> "I'm a deep-focus worker in the mornings — I block off time for complex problems. Afternoons are for collaboration: code reviews, pairing, meetings. I'm a strong async communicator — I write thorough Slack messages and design docs so I don't block people. I value direct feedback and give it constructively."

**"How do you handle conflict on a team?"**
> "I address it directly and early. If I disagree with a technical decision, I'll raise my concerns in a design review with data and alternatives — never in a Slack rant or a hallway conversation. If there's a personal friction, I'll have a 1-on-1 conversation to understand their perspective. I've found that most team conflicts come from misaligned expectations, not bad intentions. Once you name the misalignment, the solution usually becomes obvious."

### Red Flag Detection: Questions to Ask Them

You're interviewing the company too. Ask these questions to detect cultural red flags:

| Question | What You're Assessing | 🚩 Red Flag Answer |
|---|---|---|
| "What does a typical on-call rotation look like?" | Work-life balance, operational maturity | "We don't really have a rotation — everyone just pitches in" |
| "How do you handle tech debt?" | Engineering culture, long-term thinking | "We don't really have time for that" |
| "What happened in the last post-mortem?" | Blameless culture, learning mindset | "We don't do post-mortems" or blame-heavy answer |
| "How often do people get promoted?" | Growth opportunities, fairness | Vague answer or "when leadership decides" |
| "What's your team's biggest challenge right now?" | Transparency, self-awareness | "Nothing, everything's great" |
| "How do you make prioritization decisions?" | PM culture, engineering influence | "Leadership tells us what to build" |

### Managing Up: Working Effectively With Your Manager

Senior engineers are expected to manage the relationship with their manager, not just take direction:

**1. Understand their priorities.** What does *your manager* get measured on? Align your work to make them successful.

**2. Bring solutions, not just problems.** Instead of "the pipeline is broken," say "the pipeline is broken, here are two options to fix it, I recommend option A because..."

**3. Make their job easier.** Send proactive updates. Prepare for 1:1s with an agenda. Flag risks early. Don't make them ask.

**4. Seek feedback explicitly.** "What's one thing I could do better?" in every 1:1. Act on the feedback visibly.

**5. Disagree productively.** If you disagree with a decision, say so — once, with reasoning. If they decide differently, commit fully. Your manager should never wonder whether you're actually executing the plan.

> **Coaching tip:** In interviews, you'll sometimes be asked about a time you disagreed with your manager. The best stories show: (1) you raised your concern respectfully and with data, (2) you either convinced them or committed to their decision, and (3) the outcome was positive either way. Never trash-talk a former manager in an interview, even if they deserved it.

---

## Module 3 Summary: Key Takeaways

| Topic | Core Principle | Interview Signal |
|---|---|---|
| Estimation | Use 3-point estimation with phase breakdown | "I have a system for dealing with uncertainty" |
| Milestones | Thin vertical slices; deliver value early | "I de-risk projects by proving patterns first" |
| Scope Creep | "Yes, and here's the trade-off" | "I protect the team while respecting stakeholders" |
| Stakeholder Mgmt | Early, frequent, transparent communication | "I build trust through predictability" |
| Amazon LPs | Map stories to multiple principles | "I speak your language and embody your culture" |
| Google Googleyness | Collaboration, humility, ambiguity tolerance | "I make teams better and thrive in uncertainty" |
| Meta Values | Speed, boldness, impact at scale | "I ship fast and measure everything" |
| Company Research | 60-90 min prep with a structured template | "I did my homework and I'm genuinely interested" |
| Team Dynamics | Sustainability, directness, flexibility | "I'm a low-maintenance, high-output teammate" |

> **Final coaching thought:** Project management and cultural fit are the #1 differentiator between senior and mid-level candidates. Two engineers with identical coding skills will get different outcomes based on how they talk about managing work, managing people, and fitting into an organization. This is the module where you stop being "a good coder" and start being "someone I want to lead a project."

---

*Next up: Module 4 — Communication & Influence*
