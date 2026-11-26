# iPe Merchant Onboarding Copilot — Evaluation Report

**Updated: 26 November 2026**  
**Capstone Track B**  
Repository snapshot reviewed: `Capstone-iPe/analysis` default branch; architecture decision record dated 4 November 2026.  
Prepared for academic project submission. Institution, course, team and mentor details to be completed by authors.

## Executive summary

This report consolidates the project's retrieval, prompting and end-to-end workflow evaluation evidence and updates the implementation-status assessment against the latest available analysis repository README and ADR-0001.

The strongest retrieval configuration remains hybrid search with 800-token chunks and 120-token overlap: on 24 fixed policy questions it records hit@4 **0.958**, hit@8 **1.000**, mean recall@4 **0.694** and mean recall@8 **0.840**. The prompting diagnostic records structured few-shot at **6/10** clean cases, compared with **0/10** for CoT/self-consistency and **0/10** for decomposition.

The analysis repository README reports **50/50 development** and **21/21 holdout** replay passes (71/71 in total). These are project-reported results; the raw per-case replay artifacts were not independently retrieved for this refresh, so they are not represented as an independent audit.

The implementation-status update is material: the README now marks the live web happy path, Postgres conversation history, Langfuse per-turn tracing, a recorded Lotus Tea House web run, Render deployment, per-merchant rate limiting and a five-minute policy-search cache as complete. These claims are distinguished from independent runtime verification. A production-grade document verifier remains a limitation: README and ADR-0001 describe different verifier modes, so the exact deployed mode should be confirmed before submission.

## 1. System under evaluation

The system is the Track B iPe Merchant Onboarding Copilot. The hosted chat model is `Qwen/Qwen3-8B`; embeddings use `bge-small-en-v1.5`. The web frontend communicates with the Copilot service, which is the only language-model service and the MCP client. The deterministic ONB Python service owns application state, pending-field order, file state, rule execution and review-ID allocation. The separate `onb-corpus` service provides authenticated policy retrieval.

The policy corpus contains **500 documents** and **1,446 chunks** using the retained 800/120 configuration. The 71-shop gold set is kept outside the retrieval index: G-001–G-050 are the development set and G-051–G-071 are the holdout set.

ONB exposes a closed set of five MCP tools: `next_step`, `save_answer`, `save_file`, `submit` and `get_review`. There is no activate, waive, change-volume or mark-complete tool. The copilot can explain and guide; a human iPe role remains responsible for activation.

## 2. Evaluation protocol and metric definitions

Three evidence layers are evaluated independently and their denominators must not be combined:

- **Retrieval:** 24 fixed policy questions. Hit@k is the fraction of questions with at least one gold document among the top-k unique documents. Mean recall@k is the mean fraction of gold documents retrieved.
- **Prompting:** 10 diagnostic cases comparing prompting strategies. Clean-case count is a small-sample diagnostic, not a general benchmark.
- **End-to-end replay:** 71 full-shop cases. A case passes when all required `Correct` assertions pass. Workflow pass rate is passing cases divided by cases run.

The intended split is G-001–G-050 for development and G-051–G-071 for a one-time holdout run after tuning. Reusing holdout cases for prompt or rule tuning would invalidate the holdout interpretation.

## 3. Retrieval results

The fixed retrieval evaluation contains 24 policy questions. The measured results are:

| Method | hit@4 | hit@8 | Mean recall@4 | Mean recall@8 |
|---|---:|---:|---:|---:|
| Dense | 0.792 | 0.833 | 0.458 | 0.625 |
| Keyword | 0.583 | 0.750 | 0.333 | 0.479 |
| Hybrid | 0.958 | 1.000 | 0.694 | 0.840 |
| Hybrid + MiniLM | 0.833 | 1.000 | 0.500 | 0.722 |

Hybrid is strongest on this fixed set for hit@4 and mean recall. The reranker does not improve the measured result.

## 4. Chunking experiment and selected configuration

The chunking sweep compared 400/60, 800/120 and 1200/180 (chunk size / overlap). The retained 800/120 setting provides the best hit@4 jointly with the top hit@8 result, while keeping the index smaller than 400/60. The 1200/180 setting ties hit@8 and gives a marginally higher mean recall@4, but does not improve hit rates; 800/120 remains the documented product setting.

The optional MiniLM reranker did not improve the fixed-set result and is not retained on the product path.

| Chunk / overlap | Chunks | hit@4 | hit@8 | Mean recall@4 | Mean recall@8 |
|---|---:|---:|---:|---:|---:|
| 400 / 60 | 2,667 | 0.875 | 0.958 | 0.597 | 0.729 |
| 800 / 120 | 1,446 | 0.958 | 1.000 | 0.694 | 0.840 |
| 1200 / 180 | 1,063 | 0.958 | 1.000 | 0.701 | 0.840 |

## 5. Prompting baseline

On `Qwen/Qwen3-8B`, structured few-shot prompting produced **6/10** clean cases. CoT/self-consistency produced **0/10**, and decomposition produced **0/10**. Observed failure patterns included invented intent labels, wrong next-step selection, false activation language and missing review IDs.

The evidence supports structured outputs, constrained intent enums, authoritative ONB state and deterministic tool execution. Because n=10 is small, these results should be presented as a diagnostic comparison rather than a statistically generalisable ranking of prompting methods.

## 6. End-to-end replay results

The analysis repository README reports **50/50** development cases passing and **21/21** holdout cases passing, or **71/71** repository-reported passes. The development cases are G-001–G-050; holdout cases are G-051–G-071.

**Evidence qualification:** the README points to `copilot-svc/docs/results/` for replay outputs, but the raw case-level artifacts were not independently retrieved during this report refresh. Include the raw run summary and per-case artifacts in the final submission so reviewers can verify the denominators, assertions and holdout procedure.

Expected assertions cover field collection, document state, conditional addendum requirements, next-step sequence, submit gating, outcome, missing items, assignee and review-ID stability.

## 7. Failure taxonomy F01–F15

Use one primary failure code per failed case and optional secondary codes. Capture expected versus actual behaviour, severity, trace ID, test-case ID, model/prompt/corpus version and whether application state changed.

| Code | Failure class |
|---|---|
| F01 | Retrieval miss |
| F02 | Retrieval contamination |
| F03 | Intent classification error |
| F04 | Next-step/state error |
| F05 | Tool selection/argument error |
| F06 | Tool execution/integration failure |
| F07 | Persistence error |
| F08 | Document verification error |
| F09 | Decision-rule error |
| F10 | Unsafe action/boundary breach |
| F11 | Citation/grounding error |
| F12 | Privacy/sensitive-data exposure |
| F13 | Response rendering error |
| F14 | Trace/observability gap |
| F15 | Latency/capacity degradation |

## 8. Failure analysis method

Reproduce the failure with pinned inputs and versions. Locate the first divergent stage using the trace timeline. Assign primary and secondary F-codes. Assess impact on the user-visible response, persisted state, verdict, privacy and safety. Record root cause and corrective action, then add a regression case before closing the issue.

Do not classify every wrong answer as a retrieval failure. A correct passage with a bad paraphrase is a generation/grounding issue; a correct verdict with a wrong explanation is not a decision-rule failure; an incorrect deterministic outcome is F09 even if the copilot phrased it faithfully.

## 9. Observability and tracing status

The latest analysis README states that each live turn is exported to **Langfuse**, with the session ID corresponding to the ONB application ID. It also identifies a public Langfuse session for the Lotus Tea House live-web demonstration. This supersedes the earlier status in which trace export and the dashboard were open.

Recommended trace coverage remains request, intent, `next_step`, retrieval, MCP tool call, ONB result, critic/guard and response. Useful fields include trace/span/parent IDs, application ID, turn ID, stage, status, duration, tool name, safe summaries, model/prompt version and policy document IDs.

The separate HTML/PDF dashboard starter created for this project is not proof that a production dashboard is deployed. Treat Langfuse as the repository-reported live trace destination and the starter dashboard as a separate visualization/collector artifact unless the team has connected it to the live event stream. Dashboard metrics should include trace count, event count, failure rate with denominator, latency percentiles, critical-stage completeness and F01–F15 counts.

## 10. Deployment and live-demo status

According to the latest analysis README, the frontend is wired to the live copilot happy path; conversation history is stored in Postgres; the services are deployed on Render; a per-merchant limit of 30 messages per minute is configured; and policy search is cached for five minutes using Render Key Value (`copilot-cache`). The README also marks the live Lotus Tea House web run and public Langfuse session as complete.

These are repository-recorded implementation claims, not independent uptime, load or security tests. The live demo scenario is a restaurant in Pune with monthly volume ₹12,00,000, registration and owner ID uploaded, and bank proof, shopfront photo and addendum missing. The expected result is `IN_REVIEW`, assigned to Merchant Risk Lead, with review ID MR-2041. A request to activate must not change the outcome or review ID.

## 11. Safety, privacy and document-verification limitations

The closed MCP tool surface prevents the copilot from directly activating a merchant, waiving a document or changing a volume ceiling. Human review remains part of the go-live boundary.

Do not log raw uploads, bank details, identity numbers, phone numbers, credentials, API keys or unredacted prompts. Apply authentication, least privilege, encryption, retention and deletion controls; test redaction at both ingestion and presentation.

**Document-verifier qualification:** the current README describes an LLM checker as the default (`DOCUMENT_VERIFIER_MODE=llm`) with deterministic rules as fallback. ADR-0001 describes the capstone verifier as mocked using format checks and fixtures, and says production OCR/authenticity checks are deferred. These descriptions are not fully aligned. Confirm the actual deployed configuration and test evidence, then update the ADR or README so the submission makes one consistent claim. In any case, this evaluation does not establish production-grade OCR, document authenticity or identity verification.

## 12. Limitations

- Retrieval evaluation is limited to 24 fixed policy questions.
- Prompting comparison is limited to 10 diagnostic cases.
- Replay pass rates are repository-reported; raw case-level logs were not independently audited in this refresh.
- The verifier's documented implementation status is inconsistent between README and ADR-0001.
- Repository claims of deployment and live tracing do not establish production availability, load resilience, security certification or complete user-experience quality.
- No latency distribution or capacity benchmark is included in the evidence summarized here.

## 13. Recommendations and next steps

**Priority 0 — evidence and correctness**
1. Publish raw development and holdout replay summaries plus per-case labels and pinned versions.
2. Resolve the document-verifier mode discrepancy and document exactly what has been exercised in the deployed path.
3. Export a reproducible Langfuse trace for the Lotus Tea House scenario, with sensitive values redacted.
4. Add automated regression checks for review-ID stability, submit gating, refusal boundaries and conditional addendum logic.

**Priority 1 — evaluation depth and operations**
1. Expand the policy-query set and add citation-support scoring, not only document hit rate.
2. Publish p50/p95/p99 latency, tool failure rate and load-test results.
3. Verify trace completeness and failure taxonomy assignment against known injected failures.
4. Document retention, access control, deletion and redaction tests for trace data.

## References

- Analysis repository: https://github.com/Capstone-iPe/analysis
- Architecture decision record: https://github.com/Capstone-iPe/analysis/blob/main/ADR-0001-architecture.md
- Retrieval evaluation: https://github.com/Capstone-iPe/onb-corpus/blob/main/eval/retrieval.md
- Retrieval results: https://github.com/Capstone-iPe/onb-corpus/tree/main/eval/results
- Analysis project dashboard: https://capstone-ipe.github.io/analysis/
- Corpus project dashboard: https://capstone-ipe.github.io/onb-corpus/
- Lotus Tea House demo notes: https://github.com/Capstone-iPe/Reports/blob/main/lotus-tea-house.md
- Public Langfuse session (as linked from the analysis README): https://us.cloud.langfuse.com/project/cmwwc06zv025bad0djdu06aup/sessions/app_3020d278483c
