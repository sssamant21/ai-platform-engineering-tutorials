# Chapter 089 — Model Quality and Release Regression Incidents

**Part 11 — Incident Response, HA and Disaster Recovery · Chapter 089 of 100**  
**Publication:** Chapter and offline lab published. Real staging evaluations, incident game day and independent safety/security review pending.

## Learning outcomes
Investigate LLM quality regressions after model, prompt, retrieval, tool, embedding, tokenizer, routing or policy releases; detect failures that HTTP 200 and latency dashboards miss; mitigate safely with a verified rollback; and produce auditable RCA/COA.

## 1. Failure domains and signal architecture
```text
request -> gateway -> model router -> serving revision -> response
                          |                  |
                          +-> retrieval ----+-> tools
                                |
           sampled privacy-safe outcomes and feedback
                                |
  evaluation pipeline: gold sets / slices / judge / human review
                                |
          quality SLO + safety gates + drift detection
                                |
       alert -> incident commander -> mitigation -> recovery
```
A successful HTTP request is not necessarily a correct, grounded, safe or useful answer. Use distinct operational and semantic health checks.

| Domain | Regression | Evidence |
|---|---|---|
| Model/adapter | factuality, refusal, formatting, instruction following | pinned eval suite, model digest |
| Prompt/template | changed instructions or variable interpolation | template diff, trace metadata |
| Retrieval | stale documents, poor recall, ACL filtering, embedding mismatch | corpus revision, recall@k, citations |
| Tools | invalid arguments, wrong action, permission errors | tool outcome, authorization audit |
| Routing | traffic silently shifted to different model | route decision, per-model score |
| Tokenizer/context | truncation or incompatible chat template | token lengths, truncation counters |
| Safety | harmful output or sensitive disclosure | red-team tests, security incident process |
| Evaluation | changed grader or dataset distribution | evaluator version, inter-rater agreement |

## 2. Quality SLOs and measurements
Select metrics matched to product risk: task success, grounded answer rate, citation correctness, structured-output validity, retrieval recall@k, tool success, refusal correctness, safety violations and human escalation. Segment by tenant tier, language, task, prompt length, document freshness, model revision and release cohort without exposing personal content.

Averages can conceal critical subgroup regressions. Report sample sizes, confidence intervals, judge drift and statistical uncertainty. LLM-as-judge scores are fallible: calibrate against human-labeled examples and use independent safety tests.

## 3. Release gates
1. Pin model weights, adapter, tokenizer, prompt, retrieval index, evaluator and routing config by immutable revision.
2. Establish golden and adversarial test sets, including known incident regressions.
3. Record pre-release baseline and slice-level pass/fail thresholds.
4. Run offline deterministic checks and probabilistic evaluations with repeated samples.
5. Verify safety, tenant isolation, tool authorization and privacy as hard gates.
6. Shadow traffic only when permitted; do not leak data to unapproved providers.
7. Canary a small approved cohort with explicit rollback thresholds.
8. Monitor semantic outcomes plus latency, TTFT, 5xx, GPU cost and spend.
9. Promote only after designated review; retain prior signed revision and recovery plan.
10. Reevaluate after corpus, routing, policy or model changes.

## 4. Triage in the first 15 minutes
- Confirm onset, affected task/tenant/language slices and customer-visible failure.
- Freeze releases and identify all correlated revision changes.
- Separate service errors from quality failures; inspect safe examples or synthetic reproductions.
- Compare baseline/canary scores and denominators; rule out evaluator drift.
- Check retrieval freshness, embedding/index compatibility, context truncation and tool broker behavior.
- Inspect router fallback, model quantization, temperature and max-output-token settings.
- Assess safety or data exposure; engage security/privacy immediately if indicated.
- Choose controlled rollback or containment, validate downstream dependencies, and communicate uncertainty.

## 5. Rollback decision matrix
| Finding | Preferred response | Verification |
|---|---|---|
| model revision regression | revert model and tokenizer together | golden set, p95, safety |
| prompt regression | restore pinned template | task/slice eval |
| retrieval mismatch | restore compatible embedding/index pair | recall, ACL and citation tests |
| tool failure | disable affected action under policy | authorization and business outcome |
| unsafe router fallback | restrict approved route | model identity and quality |
| evaluator-only drift | repair evaluation before broad rollback | human calibration |
| confirmed privacy breach | contain affected path and invoke security process | access review and disclosure controls |

Do not restore a model or corpus revision that violates current authorization, license or privacy policy. A rollback is a new production change with explicit validation.

## 6. Incident runbooks
**RB-089-A — Sudden answer-quality drop:** declare incident, isolate affected slices, compare signed revisions, reproduce with synthetic prompts, rollback approved component and validate quality/SLOs.

**RB-089-B — Retrieval grounding failure:** inspect index freshness, embeddings, reranker and ACL filtering; restore compatible components; test tenant isolation and citation fidelity.

**RB-089-C — Safety regression:** stop risky route, restrict tool permissions, page security/safety owner, preserve sanitized evidence and require sign-off before recovery.

**RB-089-D — False-positive evaluator alert:** verify evaluation data/version, sample size and human calibration; do not dismiss real user reports solely on judge score.

**RB-089-E — Post-incident RCA/COA:** document timeline, triggering change, contributing factors, detection gap, mitigation, customer impact, permanent test and named owner.

## 7. Executable staging game day
Use synthetic tenant data and a pinned evaluation corpus. Inject a faulty prompt template, a mismatched embedding index, unsafe router fallback and structured-output regression. For each, capture change ID, evaluation deltas, incident severity, communication, rollback and verified recovery. Validate that the incident response does not bypass gateway budgets or retrieval ACLs.

## 8. Acceptance rubric (100 points)
| Area | Points |
|---|---:|
| Baselines, task metrics and evaluation reliability | 20 |
| Slice-level regression and safety detection | 20 |
| Change correlation and root-cause investigation | 20 |
| Safe rollback, quality/SLO recovery | 20 |
| Tests, runbooks, evidence and RCA/COA | 20 |
| **Total** | **100** |

Target >=90/100, with **zero critical safety, privacy or cross-tenant failures**. Offline passing tests alone do not certify production readiness.

## 9. Companion lab
[Chapter 089 — Model Quality and Release Regression Incident Lab](../labs/089-model-quality-release-regression-incidents-check.md). The Python standard-library evaluator and pytest suite compare cohort quality, enforce critical safety gates and decide rollback based on synthetic evidence. They do not run a live LLM or replace statistical evaluation.

## References
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [OpenAI Evals](https://github.com/openai/evals)
- [Google SRE Workbook — Incident Response](https://sre.google/workbook/incident-response/)
- [OpenTelemetry](https://opentelemetry.io/docs/)

**Next:** Chapter 090 — TrueFoundry, Ray Serve and KServe Control Plane Failures.
