**iPe**

**iPe MERCHANT ONBOARDING COPILOT — TECHNICAL REPORT**

*Architecture, implementation, retrieval, agentic orchestration, evaluation and operations*

Capstone Track B \| Updated 21 November 2026  
Repository snapshot: 596c50940152

Prepared for academic project submission  
Institution, course, team and mentor details to be completed by authors.

# 1. Introduction and problem definition

The iPe Merchant Onboarding Copilot is a Track B conversational workflow for collecting information and supporting evidence for one merchant shop. It combines a hosted Qwen/Qwen3-8B language model, policy-grounded retrieval, a constrained MCP tool surface and a deterministic Python onboarding service. The core safety property is architectural: the language model may interpret, coordinate and explain, but it cannot decide eligibility, waive evidence or activate a merchant.

Updated project evidence records a 71-shop gold set with 50/50 development and 21/21 holdout replay passes; the repository summary reports these results, while the raw copilot-svc case logs were not independently re-audited for this document. Policy retrieval uses 500 documents and 1,446 chunks. On 24 fixed policy questions, hybrid retrieval at 800/120 chunking achieved hit@4 0.958 and hit@8 1.000. The document verifier remains mocked and the trace dashboard remains an integration starter rather than a live operational dashboard.

Keywords: merchant onboarding; RAG; hybrid retrieval; MCP; deterministic workflow; evaluation; observability; responsible AI.

Merchant onboarding requires structured data collection, evidence intake, eligibility checks and an explainable review outcome. Form-based experiences can make it difficult for applicants to understand why a document is needed, what is missing, or how to correct a rejected upload. A conversational interface can reduce friction, but a generative model must not become the source of truth for regulated or consequential decisions.

Project objective: guide a merchant through a single-shop application, request one pending item at a time, accept documents, present a review summary, and return a deterministic verdict with missing items, assignee, policy references and review ID. The thread remains open after submission so new evidence can update the outcome without creating a new review ID.

The engineering problem is therefore not simply question answering. It is controlled orchestration across state, files, rules, retrieval, tool execution and human review. The architecture is designed to fail closed when model output is malformed or an action is not supported.

# 2. Product scope, actors and requirements

Actors: the merchant provides details and uploads; the web chat presents the interaction; the copilot interprets the turn and coordinates services; ONB stores authoritative state and executes rules; onb-corpus returns policy passages; a human iPe reviewer owns activation.

Functional requirements include collecting shop_name, city, business_type, monthly_volume, payment_methods and stores_payer_phones; collecting business_registration, owner_id, bank_proof and shopfront_photo; conditionally requesting dpa_addendum when payer phones are stored; and allowing an optional note. The system asks for one pending item at a time and shows a summary when intake is complete.

Submit must be blocked while required document slots remain unresolved. A merchant may explicitly mark a missing document unavailable. The verdict includes outcome, missing items, assignee, policy citations and a stable review ID. A later accepted file reruns the deterministic rules without changing that ID.

Non-goals: automatic merchant activation, policy waiver, volume manipulation, document acceptance by language-model assertion, autonomous human-review bypass, or training/fine-tuning model weights.

# 3. Architecture overview

The system has two business services plus a separate corpus service. The copilot is the only LLM service. It prompts Qwen/Qwen3-8B, runs a planner/executor/critic loop, calls the policy search API, acts as MCP client and phrases user-facing replies. ONB is plain deterministic Python. It owns application state, file states, field order, rule execution and review identifiers. onb-corpus owns policy documents, indexing, query embeddings and hybrid search.

Data ownership is deliberately asymmetric: ONB is the single source of truth for answers, document states, policy version, phase, pending field and review ID. The copilot is stateless for durable business state between turns. This avoids hidden conversational memory becoming authoritative application data.

Service separation enables independent testing: rule correctness can be tested without an LLM; retrieval quality can be measured without submitting a shop; and prompt behaviour can be tested against fixed cases. The cost is network orchestration, authentication and additional operational failure modes.

# 4. Service responsibilities and interfaces

Web client: thin chat UI; it talks only to the copilot. Copilot: receives a turn, reads authoritative state, classifies intent, calls at most the permitted tool(s), retrieves policy passages for explanations, and returns a structured/user-friendly response. ONB: exposes a closed MCP surface and owns all state transitions. onb-corpus: exposes Bearer-authenticated POST /v1/search.

Model endpoints: hosted OpenAI-compatible chat endpoint Qwen/Qwen3-8B; hosted embedding endpoint bge-small-en-v1.5. No model weights are modified. Embeddings are used by the corpus service, not ONB.

ONB tool contract: next_step creates/returns the application and reports the next required field plus snapshot; save_answer writes or replaces an allowed answer before submit; save_file stores a file or marks a slot missing and invokes the verifier; submit runs the deterministic rule pipeline; get_review reads the existing review ID and never allocates a new one.

The closed tool list intentionally has no activate, waive, change-volume or mark-complete operation. This is stronger than a prompt instruction: the capability is absent from the callable interface.

# 5. Intake workflow and state machine

The workflow has three phases: intake, review and verdict. ONB owns the pending field and sequence. The documented order is shop_name, city, business_type, monthly_volume, payment_methods, stores_payer_phones, then business_registration, owner_id, bank_proof and shopfront_photo; dpa_addendum is conditional on payer-phone storage; note is optional.

A why-question is an explanation request. It must not save the pending field or submit the application; the copilot retrieves policy and then re-asks the same pending field. Before submit, an already stored answer can be replaced when the new value is valid. After verdict, stored answers remain unchanged.

A document slot is satisfied only when a file is received and accepted by the verifier, or the merchant explicitly marks it unavailable. A text claim such as 'I attached it' without file bytes does not satisfy the slot. A rejected file leaves the slot open and the same item remains pending.

When nothing is pending, the copilot renders the application snapshot as a review summary. Submit is enabled only after all required slots are accepted or explicitly marked missing.

# 6. Deterministic decision pipeline

ONB evaluates the application snapshot using plain rules. Business types explicitly refused by policy (including money services, crypto and gambling) return REFUSED and do not receive a review ID. Supported categories such as restaurant, grocery and retail continue to evidence checks. An unlisted type remains IN_REVIEW rather than being inferred as eligible.

Missing or invalid required evidence produces IN_REVIEW. A complete valid pack may produce RECOMMEND_GO_LIVE. The monthly-volume threshold determines routing between Merchant Risk Lead and Head of Payments. The outcome is a recommendation for human review, not activation.

The rule inputs include business type, required fields, payment methods, payer-phone storage, document validity, conditional addendum and monthly volume. The copilot renders the structured verdict but cannot alter its fields. If the retrieval system returns a wrong passage, the citation/explanation may be wrong; the deterministic verdict remains unaffected.

# 7. Document lifecycle and verifier boundary

ONB calls a separate Document Verifier seam when save_file is invoked. The conceptual contract is verify(slot, shop_name, file, submit_date) returning accepted, reason_code and supporting metadata. The capstone implementation is mocked: it performs format checks and fixture-based accept/reject decisions.

Invalid or missing slots remain gaps and therefore prevent a clean recommendation. The model is not permitted to mark a document accepted. The mock does not establish PDF/OCR extraction quality, identity authenticity, bank-account ownership, tamper detection or production-grade document validation.

Production replacement should introduce file-type allowlists, malware scanning, size limits, OCR/extraction, legal-name matching, freshness checks for dated evidence, signed-addendum verification, confidence thresholds, human exception review and auditable reason codes.

# 8. Policy corpus and RAG design

The policy retrieval pack is maintained in Capstone-iPe/onb-corpus. The refreshed architecture records 500 documents and 1,446 chunks at 800-token target size and 120-token overlap. The pack contains policy documents and related annexes, circulars, briefings, opinions, FAQs, territorial notes, reviewer guidance and rejected drafts. The 71 practice shops in analysis/eval are not indexed, preventing test-set leakage.

The copilot calls POST /v1/search with Bearer authentication after ONB has acted, primarily to ground explanations and citations. Retrieval is not an ONB tool and is not used to calculate eligibility. The corpus service owns query embedding, dense similarity, keyword postings and ranking.

Selected product path: hybrid dense + keyword retrieval with Reciprocal Rank Fusion, folder-tier bonus and one chunk per document. The optional MiniLM cross-encoder is not retained on the product path because it reduced hit@4 in the fixed experiment and the free Render environment cannot host torch.

# 9. Retrieval evaluation protocol

The retrieval gold set contains 24 fixed why/cite questions with gold document_id lists spanning policy, annex, circular and FAQ material. Shop-level evaluation cases are separate and excluded from the corpus. Retrieval is scored independently from end-to-end workflow behaviour.

hit@k is the fraction of questions for which at least one gold document appears in the top-k unique documents. mean recall@k is the mean fraction of gold document IDs retrieved at the cutoff. The project reports hit@4 and hit@8 as the principal checklist measures, while retaining mean recall to capture multi-document coverage.

Results are tied to the corpus/index configuration. Any policy update or chunking change should be accompanied by a pinned corpus/index version and a fresh retrieval replay. The 24-question set is useful for controlled comparison but remains small relative to all possible policy questions and paraphrases.

# 10. Retrieval results and configuration decision

At 800/120, dense-only achieved hit@4 0.792 and hit@8 0.833, with mean recall@4 0.458 and mean recall@8 0.625. Keyword-only achieved 0.583 / 0.750 and mean recall 0.333 / 0.479. Hybrid achieved 0.958 / 1.000 and mean recall 0.694 / 0.840. Hybrid plus MiniLM achieved 0.833 / 1.000 and mean recall 0.500 / 0.722.

Chunking sweep: 400/60 generated 2,667 chunks and achieved hybrid hit@4 0.875, hit@8 0.958, mean recall@4 0.597, mean recall@8 0.729. 800/120 generated 1,446 chunks and achieved 0.958, 1.000, 0.694 and 0.840. 1200/180 generated 1,063 chunks and achieved 0.958, 1.000, 0.701 and 0.840.

Decision: retain 800/120. The larger chunk setting matches hit rate but does not provide enough measured benefit to justify rebuilding the deployment index. Do not deploy the cross-encoder on the product path based on the current measured results.

# 11. Prompting baseline

The prompting baseline compared three strategies on Qwen/Qwen3-8B using ten cases. Structured few-shot prompting produced six clean cases out of ten. Chain-of-thought/self-consistency produced zero clean cases out of ten. Decomposition produced zero clean cases out of ten.

Observed errors included invented intent labels, incorrect next-step selection, missing review identifiers and false activation language. These results support a constrained, structured classifier and a deterministic executor rather than asking the small model to reason freely over the entire workflow.

The baseline is diagnostic, not a statistically powered model comparison. The sample is small and should be expanded with stratified adversarial cases. Internal reasoning traces should not be exposed or logged; evaluation should score observable structured outputs, tool choices and state transitions.

# 12. Agent orchestration and MCP controls

The copilot follows a planner → executor → critic pattern. The planner selects an allowed intent and next action; the executor validates arguments and invokes the narrow MCP tool surface; the critic checks policy support, citation alignment and sensitive-digit leakage before response rendering.

The copilot uses a closed intent schema and bounded steps. Invalid labels or malformed outputs should be rejected or retried under a fixed policy; they must not be translated into arbitrary tools. ONB remains authoritative after every tool call. A tool error or timeout should lead to a safe retry/clarification or a transparent failure response, not fabricated success.

Five tools are exposed: next_step, save_answer, save_file, submit and get_review. Policy search is internal to the copilot's RAG path. The Document Verifier is internal to ONB. This separation reduces the number of ways a prompt injection can reach a business action.

# 13. End-to-end workflow evaluation

The full-shop gold set contains 71 cases. G-001–G-050 form the development/build set; G-051–G-071 are the holdout set and should be run once after tuning. The analysis README records replay pass rates of 50/50 on development and 21/21 on holdout.

These figures imply a repository-recorded 71/71 pass outcome for the documented replay. In this report, the rates are explicitly attributed to the analysis README's reference to copilot-svc/docs/results. The raw copilot-svc artifacts and individual case logs were not independently retrieved during this report refresh, so no claim is made that those case-level outputs were re-audited here.

Pass/fail should be judged against each case's Correct assertions: required fields, evidence state, next step, deterministic outcome, missing items, assignee and review-ID behaviour. A single aggregate pass rate can conceal category-specific weaknesses; future reports should publish a confusion matrix and per-case failure labels.

# 14. Failure taxonomy F01–F15

F01 Retrieval miss — relevant in-force policy passage absent from top-k. F02 Retrieval contamination — rejected, superseded or non-effective material treated as current. F03 Intent classification error — user turn mapped to wrong or invented intent. F04 Next-step/state error — wrong field, skipped item or stale snapshot. F05 Tool selection/argument error — wrong tool or invalid arguments. F06 Tool execution/integration failure — timeout, auth, transport or server error.

F07 Persistence error — accepted answer/file or state not durably stored. F08 Document verification error — incorrect accept/reject/missing classification. F09 Decision-rule error — ONB output differs from the approved deterministic rule. F10 Unsafe action/boundary breach — attempted or claimed activation, waiver, volume change or unsupported completion. F11 Citation/grounding error — citation absent, mismatched or unsupported. F12 Privacy exposure — sensitive data appears in output, trace or logs.

F13 Rendering error — structured result rendered incorrectly or incompletely. F14 Trace/observability gap — missing or uncorrelated spans prevent diagnosis. F15 Latency/capacity degradation — timeout, cold start, queueing or resource pressure. Every incident should have a primary code, optional secondary code, severity, trace ID and reproducible case ID.

# 15. Observability and trace dashboard

The project README states that a per-turn trace schema exists in copilot-svc, but exported traces and the dashboard remain open. The updated dashboard package therefore provides a collector, SQLite store and UI starter with an event contract aligned to the architecture. It is not presented as a deployed dashboard or as evidence of live user traffic.

Recommended span sequence: request received; intent classification; ONB next_step; policy retrieval (when required); MCP tool call; ONB result; critic/guard; response rendered. Correlate spans with trace_id, span_id, parent_span_id, application_id and turn_id. Record timestamps, duration, status, tool name, safe result summaries, model/prompt version and policy document IDs.

Dashboard metrics should derive only from ingested real events: distinct traces, span volume, error rate with denominator, p50/p95/p99 latency, trace completeness and F01–F15 counts. Empty data must display 'No traces received', not fabricated zero traffic or synthetic sample rows.

# 16. Security, privacy and responsible AI

Threats include prompt injection in merchant text or uploaded content, tool misuse, policy contamination, incorrect citations, PII exposure, forged uploads and unsafe claims of activation. The strongest control is capability restriction: the tool interface contains no activate or waiver action. ONB's deterministic rules remain authoritative.

Trace and application logs must not include raw uploaded files, identity numbers, bank details, phone numbers, credentials, API keys or full unredacted prompts. Store only redacted summaries and identifiers required for diagnosis. Apply service authentication, least privilege, TLS, encryption at rest, access controls, retention limits and deletion procedures.

The critic should remove or mask sensitive digit sequences in user-facing replies and logs. Policy citations should be checked against retrieved document identifiers and relevant passages. Merchant-provided instructions and text inside documents are data, not trusted system instructions.

# 17. Deployment and operational status

The analysis README describes a live web happy path in which the frontend talks to the copilot and the copilot talks to ONB. Conversation-history persistence and UX polish remain open. A recorded end-to-end Lotus Tea House run through the live web chat remains open.

Containerised deployment with rate limiting and caching is still marked open in the project status. The CLI demo is ready; it should be labelled CLI-only where appropriate. The current trace dashboard starter requires deployment, durable storage, authentication, retention controls and instrumentation in the real copilot execution paths before it can be called operational.

Operational readiness should be assessed separately from functional correctness. Passing replay tests does not prove availability, resilience, latency under load, secure secret management or production-grade document verification.

# 18. Limitations and validity threats

The 24-query retrieval set is small and may not cover rare policies, paraphrases, multi-hop questions or policy conflicts. The 10-case prompting baseline is diagnostic and sensitive to case selection. The 71-shop replay result is repository-recorded; raw per-case artifacts were not independently re-audited in this refresh.

The document verifier is mocked. The trace dashboard is a starter, not live telemetry. The web happy path is not equivalent to a complete production UX, and the recorded live-web Lotus Tea House scenario remains open. Containerisation, rate limiting and caching are not marked complete.

Corpus drift can invalidate retrieval scores; holdout leakage can inflate workflow scores; hosted model behaviour can vary; and deployment cold starts can affect latency. Future reports must pin commit, model, prompt, corpus/index version, evaluation date and test-set denominator.

# 19. Roadmap and release gates

P0: wire trace emission into actual copilot and ONB request paths; verify end-to-end trace correlation and redaction; deploy the dashboard with persistent storage and authentication; publish per-case 71-shop replay evidence and failure labels; record the live Lotus Tea House journey.

P1: replace the mocked verifier with a production-grade validation seam; complete containerised deployment, rate limiting, caching and history persistence; expand retrieval questions and adversarial cases; publish citation-support and per-category workflow metrics.

Release gates should include exact deterministic rule-test agreement; zero successful prohibited actions in adversarial tests; acceptable retrieval and citation support on a versioned set; verified PII redaction; trace coverage of all critical stages; operational latency/error budgets; and explicit human ownership of activation. Thresholds must be agreed with the product owner rather than invented in this report.

# 20. Conclusion and references

The updated project demonstrates a clear separation between probabilistic conversation and deterministic onboarding decisions. The strongest measured retrieval setting is hybrid 800/120. The structured few-shot baseline is preferable to the tested free-form prompting strategies. The repository records successful 71-shop replay, while the dashboard and several production-readiness tasks remain open.

References: Capstone-iPe/analysis README and ADR-0001; Capstone-iPe/onb-corpus eval/retrieval.md and eval/results; prompting-baseline materials in the analysis repository; IPE-PRD-001; Capstone Track B templates. Snapshot used: analysis main commit 596c50940152e1fd625390679e0f23f749893c6e (18 Nov 2026).

Repository links: https://github.com/Capstone-iPe/analysis · https://github.com/Capstone-iPe/analysis/blob/main/ADR-0001-architecture.md · https://github.com/Capstone-iPe/onb-corpus · https://github.com/Capstone-iPe/onb-corpus/blob/main/eval/retrieval.md · https://capstone-ipe.github.io/analysis/ · https://capstone-ipe.github.io/onb-corpus/
