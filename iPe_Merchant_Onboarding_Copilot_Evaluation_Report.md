**iPe**

**iPe MERCHANT ONBOARDING COPILOT — EVALUATION REPORT**

*Updated results, replay evidence, failure taxonomy and observability plan*

Capstone Track B \| Updated 21 November 2026  
Repository snapshot: 596c50940152

Prepared for academic project submission  
Institution, course, team and mentor details to be completed by authors.

# Executive summary

The updated evaluation evidence supports the selected hybrid policy retrieval configuration and a constrained prompting approach. The analysis README records 50/50 development replay passes and 21/21 holdout replay passes for the 71-shop gold set. Because the copilot-svc raw case artifacts were not independently retrieved during this refresh, these rates are reported as repository-recorded evidence rather than an independent case-log audit.

Retrieval was measured on 24 fixed policy questions. Hybrid retrieval at 800/120 achieved hit@4 0.958, hit@8 1.000, mean recall@4 0.694 and mean recall@8 0.840. The prompt baseline recorded structured few-shot 6/10 clean; CoT/self-consistency 0/10; decomposition 0/10.

The main remaining evidence gaps are live trace coverage, an operational dashboard, a recorded live-web end-to-end scenario, production-grade document verification and deployment hardening.

# System under evaluation

The evaluated system is the Track B iPe Merchant Onboarding Copilot: Qwen/Qwen3-8B for chat, bge-small-en-v1.5 for embeddings, deterministic ONB Python service, authenticated onb-corpus search API and a closed five-tool MCP interface.

ONB owns application state and rules. The copilot plans, calls tools, retrieves passages and phrases responses. The corpus is 500 documents / 1,446 chunks at 800/120. The 71-shop gold set is excluded from the retrieval index.

Document verification is mocked using format checks and fixtures. The evaluation does not establish real OCR, authenticity or identity verification.

# Evaluation protocol and metric definitions

Three independent evidence layers are retained: retrieval (24 policy queries), prompting (10 diagnostic cases) and end-to-end workflow replay (71 full-shop cases). Their denominators must not be combined.

Retrieval hit@k = fraction of questions with at least one gold document among the top-k unique documents. Mean recall@k = mean of retrieved gold document count divided by gold document count. Workflow pass rate = cases whose required Correct assertions all pass divided by cases run.

Development set G-001–G-050 is used during build and iteration. Holdout G-051–G-071 is run once after tuning. Reuse of holdout examples for prompt or rule tuning invalidates the holdout claim.

# Retrieval results

At 800/120: dense hit@4 .792, hit@8 .833, mean recall@4 .458, mean recall@8 .625. Keyword hit@4 .583, hit@8 .750, mean recall@4 .333, mean recall@8 .479. Hybrid hit@4 .958, hit@8 1.000, mean recall@4 .694, mean recall@8 .840. Hybrid + MiniLM hit@4 .833, hit@8 1.000, mean recall@4 .500, mean recall@8 .722.

Hybrid therefore provides the strongest measured hit@4 and mean recall on the fixed query set. The reranker does not improve the measured result and is not retained on the product path.

| Mode            | hit@4 | hit@8 | mean recall@4 | mean recall@8 |
|-----------------|-------|-------|---------------|---------------|
| Dense           | 0.792 | 0.833 | 0.458         | 0.625         |
| Keyword         | 0.583 | 0.750 | 0.333         | 0.479         |
| Hybrid          | 0.958 | 1.000 | 0.694         | 0.840         |
| Hybrid + MiniLM | 0.833 | 1.000 | 0.500         | 0.722         |

# Chunking and deployment configuration

400/60: 2,667 chunks; hit@4 .875; hit@8 .958; mean recall@4 .597; mean recall@8 .729. 800/120: 1,446 chunks; .958; 1.000; .694; .840. 1200/180: 1,063 chunks; .958; 1.000; .701; .840.

800/120 is retained because it achieves the best hit@4 and matches the best hit@8, while the larger chunk setting offers no meaningful hit-rate improvement. Mean recall@8 is tied at .840.

# Prompting baseline results

On Qwen/Qwen3-8B, structured few-shot prompting produced 6/10 clean cases. CoT/self-consistency produced 0/10 clean cases. Decomposition produced 0/10 clean cases.

Failure patterns included invented intent labels, wrong next-step selection, false activation language and missing review IDs. The evidence supports structured outputs, constrained intent enums, authoritative ONB state and deterministic tool execution.

The sample is small and should be treated as diagnostic rather than a general benchmark of prompting methods.

# End-to-end replay results

The analysis README records 50/50 development cases passing and 21/21 holdout cases passing, for 71/71 repository-recorded passes. Development cases are G-001–G-050; holdout cases are G-051–G-071.

Important provenance qualification: the README points to copilot-svc/docs/results for the replay outputs, but those raw files were not independently retrieved during this report refresh. The pass rates are therefore reproduced as project-reported results and should be accompanied in the final submission by the raw run summary and per-case artifacts.

Expected assertions include field collection, document state, conditional addendum, next-step sequence, submit gating, outcome, missing items, assignee and review-ID stability.

# Failure taxonomy F01–F15

F01 Retrieval miss; F02 retrieval contamination; F03 intent classification error; F04 next-step/state error; F05 tool selection/argument error; F06 tool execution/integration failure; F07 persistence error; F08 document verification error; F09 decision-rule error; F10 unsafe action/boundary breach; F11 citation/grounding error; F12 privacy/sensitive-data exposure; F13 response rendering error; F14 trace/observability gap; F15 latency/capacity degradation.

Each failed case should receive one primary category and optional secondary categories. Record expected vs actual behaviour, severity, trace ID, test case ID, model/prompt/corpus version, and whether application state changed.

| Code | Failure class                      |
|------|------------------------------------|
| F01  | Retrieval miss                     |
| F02  | Retrieval contamination            |
| F03  | Intent classification error        |
| F04  | Next-step/state error              |
| F05  | Tool selection/argument error      |
| F06  | Tool execution/integration failure |
| F07  | Persistence error                  |
| F08  | Document verification error        |
| F09  | Decision-rule error                |
| F10  | Unsafe action/boundary breach      |
| F11  | Citation/grounding error           |
| F12  | Privacy/sensitive-data exposure    |
| F13  | Response rendering error           |
| F14  | Trace/observability gap            |
| F15  | Latency/capacity degradation       |

# Failure analysis method

Reproduce the failure with pinned inputs and versions. Localise the first divergent stage using the trace timeline. Assign primary and secondary F-codes. Assess impact on user-visible response, persisted state, verdict, privacy and safety. Identify root cause and corrective action. Add a regression case before closing the incident.

Do not classify every wrong answer as a retrieval failure. A correct passage with a bad paraphrase is a generation/grounding issue; a correct verdict with a wrong explanation is not a decision-rule failure; and an incorrect deterministic outcome is F09 even if the copilot phrased it faithfully.

# Observability and trace plan

The analysis README states that a per-turn trace schema is defined, but trace export and the dashboard remain open. The updated starter package supports ingestion and visualization of actual events but contains no fabricated sample traffic.

Recommended spans: request, intent, next_step, retrieval, MCP tool call, ONB result, critic/guard and response. Store trace_id, span_id, parent_span_id, application_id, turn_id, stage, status, duration, tool name, safe summaries, model/prompt version and policy document IDs.

Dashboard metrics should show trace count, event count, failure rate with denominator, latency percentiles, critical-stage trace completeness and failure taxonomy counts. Empty event stores should display an explicit no-data state.

# Safety, privacy and limitations

Do not log raw uploads, bank details, identity numbers, phone numbers, credentials, API keys or unredacted prompts. Apply authentication, least privilege, encryption, retention and deletion controls. Verify redaction at ingestion and presentation.

Limitations: 24-query retrieval set; 10-case prompt set; replay evidence not independently audited at case-log level in this refresh; mocked document verifier; trace dashboard not yet wired to production; live-web Lotus Tea House run remains open; containerisation/rate limit/cache remain open.

Repository-reported pass rates do not prove production availability, robustness under load, document authenticity or end-user UX completeness.

# Recommendations and references

Priority 0: publish raw replay results and per-case labels; wire and validate real traces; deploy dashboard with persistent storage and access control; record a live-web end-to-end scenario. Priority 1: expand policy-query coverage, add citation-support scoring, replace the mocked verifier, complete deployment hardening and publish latency/error budgets.

References: https://github.com/Capstone-iPe/analysis · https://github.com/Capstone-iPe/analysis/blob/main/ADR-0001-architecture.md · https://github.com/Capstone-iPe/onb-corpus/blob/main/eval/retrieval.md · https://github.com/Capstone-iPe/onb-corpus/tree/main/eval/results · https://capstone-ipe.github.io/analysis/
