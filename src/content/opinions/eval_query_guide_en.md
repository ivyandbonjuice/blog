---
title: "How to Build Verifiable Queries for Evaluating Complex Text Tasks"
lang: "en"
translationKey: "synthetic-query-construction"
date: 2026-05-29
summary: "A breakdown of query construction for non-code text tasks: real workflows, context environments, scoring constraints, difficulty tiers, and self-evolving mechanisms that turn evaluation prompts into reusable evaluation assets."
authors:
  - name: "Xinhui Huang"
    github: "ivyangbonjuice"
stance: "High-quality non-code evaluation sets cannot be built by piling up prompts; they require rewriting real user tasks into structured queries with environment, deliverables, checks, and failure modes."
tags: ["evaluation", "text-to-image", "llm-as-a-judge", "benchmark"]
---

Non-code tasks often create an illusion: since the output is all text, evaluation can only judge whether the writing is "good" or not.

But once you start building an evaluation set, you find that the hard part is not that text lacks a standard answer. The hard part is that the **query itself often fails to define the task boundary clearly**. Queries that are too broad reduce Agent capability to Chatbot capability; queries packed with constraints but lacking a scoring structure make it impossible to attribute failures.

Non-code tasks are not structureless. Their final deliverables usually take the form of scripts, reports, plans, or tables. What we need to evaluate is not just fluency, but whether the model can **actually get the job done** under complex constraints.

This article focuses on one question: **how should queries in non-code evaluation sets be constructed?**

Here, a query is not an ordinary prompt. An ordinary prompt only needs to make the model generate an answer; an eval query needs to **expose the boundary of model capability**. It should include the real task intent, contextual dependencies, hard constraints, expected path, scoring points, and failure modes.

A good eval query is not about "asking in a more complicated way". It is about "measuring more clearly".

---

## 1. First decide: is this task worth turning into an eval?

The first step in building a query is not writing the question. It is deciding which level the task belongs to, because the level determines its evaluation value.

| Level | Query example | Core capability | Evaluation value |
|------|---------------|-----------------|------------------|
| L1 General knowledge | Prepare a template for e-commerce return and exchange communication | Knowledge summarization, text generation | Suitable for basic smoke tests, not suitable as the core evaluation set |
| L2 Scenario-customized | I bought a down jacket last week and it is leaking down. It has been more than 7 days, but it is still under warranty. Help me design a negotiation plan | Scenario understanding, constraint handling, risk judgment | Suitable as the main body of non-code text-task evals |
| L3 Closed-loop execution | Based on the order, product-page snapshot, platform policy, and similar dispute cases, determine whether a partial refund is possible, and output talking points, evidence checklist, and escalation path | Data reading, evidence integration, multi-step planning, executable deliverables | Most suitable for Agent / complex text-task evals |

**The core problem with L1:** As long as the model is fluent and well-structured, it can easily get a high score. But this only tests whether the model can summarize; it cannot distinguish models that truly have planning and judgment capabilities.

**L2 starts to enter the value zone:** It requires the model to handle personalized constraints: time windows, product condition, user goals, likely merchant objections, and platform escalation paths. Each of these can become a potential scoring point.

**L3 is where high-value evaluation samples appear:** It does not merely ask the model to "write suggestions". It requires the model to complete key judgments based on environmental materials: has the order really exceeded 7 days? Does the product page contain misleading material claims? Does the platform policy support partial refunds? Is the evidence sufficient to escalate the complaint?

A simple test is: **after removing all environmental materials, can the model still complete the key judgment?**

If the answer is almost the same after removing the materials, the query is L1/L2. If the key judgment becomes impossible without the materials, it is truly L3.

The same topic across three levels:

| Topic | L1 (general knowledge) | L2 (scenario-customized) | L3 (closed-loop execution) |
|------|-------------------------|---------------------------|-----------------------------|
| Model evaluation retrospective | Summarize common types of model-evaluation misjudgments | Review misjudgment cases from the past month, classify causes, and calculate proportions | Read misjudgment details, rule documents, and weight configurations; determine attribution; output fix priorities and experiment schedule |
| E-commerce after-sales | Write return-communication talking points | Design a rights-protection plan for a down jacket leaking down after more than 7 days | Combine order, product page, platform rules, and precedents to output risk levels, talking points, evidence checklist, and escalation path |
| Flight ticket changes | Explain refund and change rules for discounted tickets | Design a lowest-loss plan for changing a Beijing-Sanya round-trip itinerary | Combine original order, airline rules, remaining seat prices, and time windows to calculate costs and rank change/refund/rebook options |

---

## 2. The structure of a good query: five indispensable elements

After identifying the level, we need to understand what an eval query is made of.

Public evaluation sets such as OpenAI GDPval, Anthropic Agent Evals, and Kimi Claw Bench all point toward the same trend: moving from a "single prompt" to a "task package".

The core idea of a task package is that the query is not just the sentence spoken by the user. It includes everything needed to complete the task.

I define the standard structure of a non-code eval query as:

```txt
query = user task
      + context package
      + expected deliverable
      + observable checks
      + failure modes
```

**User task:** A natural description of user intent. It should not sound like exam instructions; it should carry real-world pressure.

**Context package:** The external materials needed to complete the task, such as order JSON, product-page snapshots, platform policies, and historical precedents. If the model can answer almost the same way after these materials are removed, the query may not be valuable enough.

**Expected deliverable:** The output must be clear. "Give me some advice" is not a deliverable. "Output talking points by risk level, merchant-objection forecasts, and an evidence checklist" is.

**Observable checks:** These fall into two categories: **hard constraints** (whether the answer mentions that the 7-day period has passed, whether it asks for additional evidence) and **soft quality** (whether the wording feels collaborative rather than confrontational). Hard constraints can be checked automatically; soft quality needs human evaluation or an LLM judge.

**Failure modes:** Pre-label where this query is expected to make the model fail. For example: applying a generic return template, promising that the user will definitely get a refund, or only writing talking points without giving an escalation path. These labels are both grading criteria for the grader and directions for evaluation iteration.

> Agent-style evaluations also need to observe the **process trace**: whether the model actually called the right tools, whether it modified environmental state, and whether it recovered after tool failure. Saying "I completed it" and actually completing it are two different things.

---

## 3. Where queries come from: five sources with different value and cost

High-quality queries can come from five types of sources:

| Source | Strengths | Risks | Suitable stage |
|------|-----------|-------|----------------|
| Real user logs | Closest to the business, naturally representative | Requires anonymization; noisy | Core evaluation set, regression set |
| Online bad cases | Directly correspond to model weaknesses; highly discriminative | Can cluster around a few failure types | Self-evolution, version regression |
| Expert-written | Logically rigorous and controllable | Expensive; may not feel realistic enough | Cold-start MVE |
| LLM-synthesized | Fast to scale, broad coverage | Homogeneous, templated, may leak answer structure | Candidate expansion; not suitable for direct inclusion |
| Public benchmarks / industry cases | Useful for schema and coverage reference | May deviate from business scenarios | Building the initial taxonomy |

**Recommended ratios for cold start:**

| Stage | Recommended ratio |
|------|-------------------|
| 0-30 queries | 50% expert-written, 30% real logs / business feedback, 20% LLM-synthesized candidates |
| 30-100 queries | 40% real logs, 30% bad cases, 20% expert-written, 10% LLM-synthesized |
| After 100 queries | Mainly bad cases and production traces; experts handle cleaning, annotation, and gap filling |

The correct use of LLM synthesis is **candidate generation, not direct inclusion**: let the LLM generate query candidates in batches, then have humans filter, rewrite, add environmental materials, and write graders. If synthetic outputs are directly treated as gold data, the evaluation set will suffer from serious homogenization.

---

## 4. Write real task pressure into the query, not just more words

The biggest danger for non-code evaluation sets is a "textbook flavor": the query is written in a clean, orderly way, and the model can get a high score by generating a smooth paragraph.

Textbook-flavored version:

> Help me write a livestream sales script.

This query is too clean. Real business livestream scripts involve product selling points, prohibited words, duration limits, conversion goals, platform rules, and target audiences. Write those pressures into the query:

> I need to write a 60-second livestream spoken script for a foundation aimed at oily-skin commuters. It must create a pain-point conflict in the first 3 seconds and include an emotional turn every 15 seconds. It must naturally mention the three selling points: "8-hour wear", "no dullness", and "no caking". It must not include absolute claims such as "the strongest", "number one", or "medical-grade beauty". Output the script by timeline segment and mark host action cues.

The latter is not better because it is longer. It is better because it writes out the **task pressure**:

- It has a clear deliverable: a 60-second livestream spoken script
- It has structural constraints: segment by timeline and mark action cues
- It has content constraints: three selling points must appear
- It has experience goals: hook the audience in the first 3 seconds and create a turn every 15 seconds
- It has safety constraints: no absolute claims
- It has checkable items: selling points, duration, format, and prohibited words can all be evaluated

**Good queries often come from friction in real scenarios:**

- Users complain that "this cannot be used directly"
- Business stakeholders say "this is not our tone"
- Reviewers point out "there is a compliance risk here"
- Operations teams report that "the conversion point appears too late"

These are not noise; they are raw materials for queries. **Before adding a query to the evaluation set, you should be able to answer: which capability is this expected to trip the model on?** If you cannot answer that, do not rush it into the set.

---

## 5. Difficulty scale: 6 dimensions, 12 points

A healthy evaluation set needs a difficulty gradient. Below is a 6-dimensional difficulty scale for non-code queries. Each dimension is scored 0-2, for a total of 0-12:

| Dimension | 0 points | 1 point | 2 points |
|------|----------|---------|----------|
| **Context dependency** | No external materials needed | Requires 1-2 materials | Requires cross-verification across multiple materials |
| **Constraint density** | Only one goal | 2-3 hard constraints | Multiple hard constraints with priorities |
| **Goal conflict** | Goals are aligned | Minor trade-offs | Multiple goals cannot all be satisfied at once |
| **Judgment openness** | Has a clear standard answer | Requires explaining the judgment | Requires decisions under uncertainty |
| **Execution chain** | Single-step generation | Multi-step planning | Read, classify, calculate, rank, and output a closed loop |
| **Multi-turn disturbance** | Completed in one turn | User adds constraints | User changes the goal / tool fails / context must be reused |

**Difficulty ranges and uses:**

| Score | Difficulty | Use |
|------|------------|-----|
| 0-3 | Easy | Smoke tests; check basic instruction following |
| 4-7 | Medium | Main body of the evaluation set; tests combined capabilities |
| 8-10 | Hard | Separates model performance; tests planning and judgment |
| 11-12 | Adversarial | Keep a small number for boundary testing; should not take too large a share |

**Example comparison:**

> "Help me write a template for e-commerce after-sales communication."
> -> Context dependency 0, constraint density 0, goal conflict 0, judgment openness 0, execution chain 1, multi-turn disturbance 0 -> **Total score: 1 (easy)**

> "I bought a down jacket last week and it is leaking down. It has been more than 7 days, but it is still under warranty. I want a partial refund while keeping the item, and I also want to complain that the product page misrepresented the material. Please give me negotiation talking points, likely merchant objections, the platform intervention process, and an evidence checklist."
> -> Context dependency 1, constraint density 2, goal conflict 2, judgment openness 1, execution chain 1, multi-turn disturbance 0 -> **Total score: 7 (upper-medium)**

> Same as above, but add order JSON, product-page snapshot, platform policy, and similar precedents; require risk levels and priority ranking; in the second turn, the user adds: "The merchant says the warranty does not cover human-caused damage. How should I respond?"
> -> **Total score: 10-11 (hard/adversarial)**

An evaluation set is not a pile of exam questions. It is a design for **capability resolution**. Truly good queries should separate models instead of producing a flat scoreboard. If all models get full marks, the task is too easy. If all models collapse, the task may be too hard or the grader may be misconfigured.

---

## 6. Pre-inclusion checklist: self-check before adding to the set

**Basic validity (5 items; all must pass before inclusion is considered)**

| Check item | Passing standard |
|-----------|------------------|
| Realism | The sentence sounds like something a real user would say, not an exam question or textbook example |
| Task orientation | It completes a concrete task rather than broadly summarizing general knowledge |
| Clear deliverable | The output has a clear form: report, table, talking points, plan, checklist, ranking |
| Scorable | At least 2 hard constraints can be checked automatically or semi-automatically |
| Attributable | After model failure, you can tell whether the cause is missing information, omitted constraints, wrong judgment, or expression problems |

**Structural completeness (5 items)**

| Field | What to check |
|------|---------------|
| prompt | Whether the user task is natural and does not read like exam instructions |
| environment | Whether the materials needed to complete the task are provided, and whether the materials are truly bound to the task |
| hidden assumptions | Whether missing information and assumption-only information are marked |
| hard_checks / soft_checks | Whether must-pass hard constraints are separated from gradable soft quality |
| grader | Whether there is clear Pass/Fail logic, a scoring rubric, or human review rules |

**Discriminative-power checks (5 items; revise if any is triggered)**

| Signal | Action |
|------|--------|
| Strong models all get full marks | Increase constraint-conflict density or add multi-turn disturbance |
| Most models cannot complete it at all | Check whether the task wording is unclear, the environmental materials are insufficient, or the grader is too strict |
| Human experts cannot judge because of insufficient information | Add materials or reduce adversarial intensity |
| It obviously looks like a standard LLM-synthesized question | Rewrite it and add real business friction |
| Evaluation can only conclude "well written / average" | Rewrite hard_checks and add checkable hard constraints |

---

## 7. A complete query example

Below is a non-code eval query suitable for inclusion in an Agent scenario. Every field has a reason to exist: `environment` prevents the model from answering only with general knowledge; `hard_checks` makes evaluation attributable; `failure_modes` gives direction for the next iteration.

```yaml
id: ecommerce_refund_down_jacket_001
scenario: ecommerce_after_sales
difficulty_score: 9
difficulty_tags:
  - incomplete_information   # Some information must be extracted from materials and cannot be assumed
  - policy_trap              # There is a policy trap caused by user misunderstanding: partial refund is not the default path
  - multi_goal_conflict      # Refund, keeping the item, and complaint goals are in conflict
  - risk_ranking             # Requires prioritized risk judgment

prompt: >
  I bought a down jacket last week. After wearing it once, I found that it leaks down.
  It has already passed the 7-day no-reason return period, but it is still under warranty.
  I want to apply for a partial refund while keeping the item, and also complain that the merchant's product-detail page misrepresented the material.
  Please help me prepare talking points for negotiating with customer service, anticipate possible excuses from the merchant and provide rebuttals.
  If negotiation fails, also outline the platform intervention process and the evidence checklist I should prepare in advance.
  Please label all suggestions as high / medium / low risk.

environment:
  - order_detail.json          # Order time, product information, payment amount
  - product_page_snapshot.html # Product-detail material description, including suspected misrepresentation
  - platform_refund_policy.md  # Platform return/refund policy, including partial-refund conditions
  - dispute_precedent.md       # Historical precedents for similar disputes
  - evidence_checklist.xlsx    # Template for evidence that can be submitted

hard_checks:
  - Whether it proactively points out that "partial refund while keeping the item" may not be a path supported by default on the platform
  - Whether it asks the user to supplement or verify key evidence, such as order time, material description on the product page, and photos/videos of down leakage
  - Whether it distinguishes negotiation, complaint, and platform intervention as three stages and gives action items for each
  - Whether it outputs likely merchant excuses and corresponding rebuttal strategies
  - Whether it labels all suggestions as high / medium / low risk
  - Whether it avoids promising that "partial refund will definitely succeed"

soft_checks:
  - Whether the talking points sound collaborative rather than escalating confrontation
  - Whether the plan is truly executable and directly usable by the user
  - Whether the risk warnings are tied to the sufficiency of evidence
  - Whether it reasonably explains trade-offs between the user's goal and platform rules

failure_modes:
  - Directly applying a generic return template and ignoring the specifics of this case
  - Ignoring the key constraint that the 7-day period has passed and assuming the user is still within the no-reason return window
  - Promising that the user can definitely obtain a partial refund
  - Failing to ask the user to prepare key evidence, such as videos/photos of down leakage
  - Only outputting talking points without providing an escalation path and evidence checklist
  - Missing risk labels or using labels that do not differentiate risk levels
```

---

## 8. Evaluation sets are not one-off question banks; they need a data flywheel

Non-code tasks change quickly, and models will also "learn" old task types through iteration. A healthy evaluation set needs continuous evolution.

An executable data flywheel has six steps:

1. **Online capture:** Record sessions where users downvote, repeatedly ask follow-up questions, require human intervention, encounter tool failures, or heavily rewrite the output
2. **Failure attribution:** Classify bad cases into categories such as omitted constraints, factual errors, invalid format, wrong risk judgment, failure to ask clarifying questions, overpromising, and expression that does not fit the scenario
3. **Rewrite into eval queries:** After anonymization, preserve real task pressure and rewrite into a standard structure: prompt + environment + expected behavior + grader
4. **Add to the regression set:** Every time the model, prompt, or toolchain changes, run this batch of queries to ensure old problems do not recur
5. **Retire regularly:** When all mainstream models pass a type of query stably across multiple rounds, downgrade it from the core evaluation set to the smoke-test set
6. **Add new difficulties:** When new failure modes appear online, immediately add corresponding queries, such as "missing key selling points after multi-turn compression", "refusing to provide compliant rights-protection advice", or "fabricating results directly after tool timeout"

**Maintenance signal table:**

| Signal | Action |
|------|--------|
| Full-score rate for a query type exceeds 80% | Add constraint conflicts or multi-turn disturbance |
| All queries of a type fail | Check task wording, sufficiency of environmental materials, and grader strictness |
| Online bad cases appear frequently | Add them to the core regression set |
| A task has no discriminative power for a long time | Downgrade it to a smoke test |
| Human reviewers and LLM judges continue to diverge significantly | Rewrite the rubric and add anchor examples |

The key is not "accumulating more and more". It is keeping the evaluation set's **resolution**: as model capabilities improve, the difficulty boundary of the evaluation set must move upward as well.

---

## Final thoughts

To decide whether a query is worth adding to the evaluation set, look at five things:

1. **Does it come from a real user scenario,** rather than an exam question designed only for testing?
2. **Does it have a clear main test point,** so that failures can be attributed to a specific capability dimension?
3. **Does it contain verifiable hard constraints,** such as text requirements, quantity limits, format rules, or prohibited items?
4. **Does it distinguish between models,** instead of letting all models pass easily or making all models fail?
5. **Does it expose a business-meaningful failure,** rather than creating a rare and contrived trick question?

A good query is not necessarily long or complex. **Its value lies in this: when a model fails, we know why it failed; when a model improves, we also know which capability improved.**

Every good query should be like a probe, piercing the places where models most easily pretend they have "already learned it".

---

**References:**
- OpenAI GDPval
- Anthropic Demystifying evals for AI agents
- Kimi K2.6 Tech Blog
- [LangChain Agent Evals](https://docs.langchain.com/oss/python/langchain/test/evals)
- [LangSmith Evaluations](https://www.langchain.com/langsmith/evaluation)
