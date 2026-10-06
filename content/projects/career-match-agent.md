---
title: "CareerMatch Agent"
description: "Explainable ML job-matching system combining deterministic constraints, multilingual semantic ranking, hybrid retrieval and evidence-grounded LLM evaluation."
---

# CareerMatch Agent

[← Back to projects](/#projects)

**Context:** Independent Machine Learning Engineering Project · 2026–Present  
**Status:** Active technical alpha  
**Focus:** ML systems · Semantic retrieval · Hybrid ranking · LLM evaluation · Bounded agentic workflows

CareerMatch Agent is an end-to-end AI-assisted job-search and recommendation system that turns a **CV PDF and natural-language preferences** into ranked job opportunities with structured, evidence-grounded suitability reports.

I built the system around a deliberate principle: **LLMs should assist with interpretation, planning and explanation, but they should not silently override explicit user constraints or invent unsupported candidate/job claims.**

Instead of relying on a single prompt to decide whether a job is a good match, CareerMatch combines:

- deterministic filtering for hard constraints,
- multilingual SentenceTransformer retrieval,
- hybrid ranking across several matching signals,
- bounded LangGraph search and replanning,
- structured LLM evaluation over explicit evidence bundles,
- deterministic grounding checks that reject unsupported findings.

[View source on GitHub →](https://github.com/LittleBigPluton/career-match-agent)

## Why I built it

Job matching is a useful test case for a broader AI-engineering problem: **how do you combine probabilistic models with deterministic rules when some requirements must remain authoritative?**

A fully LLM-driven matcher can produce fluent explanations while still:

- accepting a job that violates location or seniority requirements,
- overlooking explicit exclusions,
- overstating candidate experience,
- inventing strengths or gaps unsupported by the CV or job description,
- producing unstable rankings between runs.

CareerMatch separates these responsibilities. Hard constraints remain deterministic; semantic models provide retrieval and similarity signals; LLMs handle tasks where language understanding adds value; generated findings are then checked against the evidence supplied to the evaluator.

The result is an AI workflow designed to be **inspectable, testable and replaceable at each stage** rather than one opaque model call.

## End-to-end architecture

The current workflow is:

```text
CV PDF + Natural-Language Preferences
            +
 Optional HackerRank Evidence
            +
 LLM / Job-Provider Selection
            ↓
 Candidate Profile Extraction
            ↓
 Preference Extraction + Validation
            ↓
 Search Planning
            ↓
 Multi-Provider Job Retrieval
            ↓
 Deduplication + Normalization
            ↓
 Deterministic Filtering
            ↓
 Semantic / Hybrid Ranking
            ↓
 Evidence Bundle Construction
            ↓
 Grounded LLM Evaluation
            ↓
 Ranked Job Recommendations
```

The workflow is orchestrated through a **bounded LangGraph agent**. If too few suitable jobs survive filtering, the system can broaden and rerun the search, but the number of attempts is explicitly limited to prevent uncontrolled agent loops.

The FastAPI backend owns the workflow and domain logic, while the Streamlit frontend acts as a user-facing client. This keeps the core system reusable outside the UI.

## Deterministic constraints before AI judgment

Explicit user preferences are treated as authoritative.

Before ranking, jobs can be rejected for constraints such as:

- role mismatch,
- seniority mismatch,
- location mismatch,
- required-keyword mismatch,
- excluded-keyword presence,
- language mismatch,
- language-level mismatch.

Natural-language preferences are first converted into a structured schema and then deterministically validated. Missing information is kept unknown or unrestricted rather than silently replaced with assumptions.

This prevents a semantically attractive job from being promoted simply because an LLM decides it is “close enough” to a requirement the user explicitly provided.

## Multi-provider retrieval with a common job model

CareerMatch currently integrates:

- **Arbeitnow**
- **Adzuna**
- **Jooble**

A composite provider layer allows several sources to participate in the same run.

Each provider implements a common interface and its response is normalized into a shared job representation before filtering and ranking. This isolates provider-specific retrieval logic from downstream matching logic and makes the retrieval layer extensible.

Users can select the active providers for each workflow run.

## Semantic and hybrid ranking

Jobs that pass deterministic filtering are ranked using a combination of signals rather than semantic similarity alone.

The current hybrid score combines:

- multilingual semantic similarity,
- candidate/job skill overlap,
- required-keyword matching,
- role alignment,
- warning quality.

Semantic matching uses:

`sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`

Candidate and job text are represented through bounded evidence chunks so that ranking and later explanation can retain interpretable supporting context.

### Embedding-model selection

I evaluated a larger multilingual MPNet model during calibration, but rejected it because it **reduced ranking quality while increasing runtime**.

During an earlier embedding-model calibration experiment:

| Model / ranking | nDCG@5 | nDCG@10 | Precision@10 |
|---|---:|---:|---:|
| MiniLM hybrid | **0.794** | **0.814** | **0.900** |
| MPNet hybrid | 0.703 | 0.759 | 0.800 |

This was an important engineering result: a larger embedding model was not assumed to be better simply because it was larger.

## Evidence-grounded LLM evaluation

The evaluator does not receive an unconstrained prompt asking whether a candidate is suitable.

Instead, CareerMatch constructs an explicit **evidence bundle** containing items such as:

- candidate summary and atomic skills,
- experience and project evidence,
- job-description excerpts,
- matched roles and skills,
- required-keyword matches,
- semantic comparison evidence,
- deterministic warnings,
- optional external assessment evidence.

Generated findings must reference supplied evidence IDs.

A deterministic validation layer checks:

- whether referenced evidence IDs exist,
- whether candidate evidence is actually used,
- whether job evidence is actually used,
- whether findings are grounded at the individual-claim level.

Unsupported strengths, gaps and risks are removed rather than presented as valid findings.

The LLM is therefore responsible for **structured interpretation of known evidence**, not for inventing the evidence itself.

## Multi-LLM support without vendor lock-in

CareerMatch implements a common structured LLM interface supporting:

- **Ollama**
- **Google Gemini**
- **OpenAI**

The provider and model can be selected at runtime through the UI or API.

The selected provider is reused consistently across LLM-dependent stages, including candidate-profile extraction, preference extraction, search planning/replanning and grounded suitability evaluation.

Non-LLM stages—including PDF extraction, job retrieval, deterministic filtering, SentenceTransformer ranking, LangGraph orchestration and grounding validation—remain independent of the selected LLM vendor.

This separation allows the same workflow to run with local or hosted models without coupling the core architecture to one provider.

## Benchmarking instead of subjective inspection

I built a benchmark framework so changes to the filtering and ranking pipeline can be evaluated quantitatively rather than judged only by manually inspecting recommendations.

The benchmark measures:

- deterministic filtering,
- rejection-reason quality,
- Precision@K and Recall@K,
- nDCG@5 and nDCG@10,
- Mean Reciprocal Rank,
- ranking latency.

LLM-generated suitability reports are evaluated separately and are not part of the filtering/ranking benchmark.

## Development benchmark suite

The frozen development suite contains **four synthetic candidate scenarios** with **20 labelled job postings each**, for a total of **80 job cases**:

- junior ML / AI candidate,
- mid-level backend engineer,
- junior data scientist,
- manufacturing professional transitioning into data / ML.

Each scenario is evaluated with the same three ranking configurations:

- `hybrid_default`,
- `semantic_only`,
- `deterministic_only`.

All three configurations achieved **1.000 filtering F1** and **1.000 rejection-reason F1** across the four development scenarios.

For ranking:

| Configuration | Mean nDCG@5 | Mean nDCG@10 |
|---|---:|---:|
| Hybrid | **0.8767** | **0.8976** |
| Semantic only | 0.8277 | 0.8737 |
| Deterministic only | 0.9129 | 0.9309 |

The development suite was used for calibration and regression analysis, so these values should not be interpreted as independent estimates of real-world performance.

## Frozen holdout evaluation

After development, I evaluated the unchanged filtering and ranking pipeline on a separate frozen holdout suite containing **40 labelled job postings** across two previously unscored synthetic candidate scenarios:

- junior multilingual NLP candidate,
- junior ML serving / platform candidate.

The holdout datasets, labels and checksums were frozen before the benchmark was executed. The ranking configurations were not retuned in response to the holdout results.

Filtering performance across the two holdout scenarios was:

| Metric | Macro mean |
|---|---:|
| Filtering F1 | **0.955** |
| Rejection-reason F1 | **0.947** |

Ranking performance was:

| Configuration | Mean nDCG@5 | Mean nDCG@10 |
|---|---:|---:|
| Hybrid | **0.901** | **0.948** |
| Semantic only | 0.897 | 0.940 |
| Deterministic only | 0.959 | 0.936 |

![Development and frozen holdout ranking benchmark across hybrid, semantic-only and deterministic-only configurations](/images/projects/career-match/career_match_benchmark_graph.png)


The results show that ranking behaviour depends on both the scenario and the evaluation cutoff. Deterministic-only achieved the highest mean nDCG@5 on the holdout suite, while hybrid ranking achieved the highest mean nDCG@10.

These results measure agreement with the frozen synthetic relevance labels. They do **not** establish that one configuration is generally superior or represent real-world hiring accuracy.

## What the evaluation revealed

The holdout evaluation exposed two role-based filtering disagreements: one unexpected acceptance in each of the two scenarios.

That matters because the purpose of the benchmark is not to produce perfect-looking metrics. It is to expose concrete failure cases that can be preserved as regression tests and addressed in later engineering work without rewriting the original benchmark.

The benchmark therefore serves as both a performance-measurement framework and a mechanism for identifying weaknesses in filtering and ranking logic.

Ranking latency measurements remain exploratory because model initialization and embedding warm-up are not yet standardized across repeated runs.

## Reusable workflow state

CV parsing and structured preference extraction can be relatively expensive, especially when LLMs are involved.

CareerMatch can export intermediate workflow state as structured JSON, including:

- PDF extraction results,
- structured candidate profiles,
- structured job preferences,
- optional assessment evidence,
- prepared workflow state,
- agent search requests and responses.

A prepared workflow can later be reloaded while changing the active LLM, job providers and search configuration.

This avoids unnecessarily repeating candidate-side preprocessing during experimentation and makes provider/model comparisons easier.

## User-facing automated workflow

The Streamlit interface exposes the complete pipeline.

A user can:

1. upload a CV,
2. describe job preferences in natural language,
3. optionally provide external assessment evidence,
4. select an LLM provider/model,
5. choose one or more job providers,
6. configure bounded search behavior,
7. run the complete workflow,
8. inspect the interpreted profile and preferences,
9. review search and agent metrics,
10. inspect ranked jobs and grounded suitability reports.

![Career Match UI](/images/projects/career-match/career-match-ui.gif)

## Engineering quality

The repository is structured as an application rather than a notebook-only prototype.

It separates:

- API routes and typed domain models,
- domain services,
- LLM providers,
- job providers,
- filtering and ranking logic,
- LangGraph orchestration,
- frontend workflow,
- benchmark datasets and runners,
- unit/integration tests,
- documentation and scripts.

Automated GitHub workflows include:

- continuous integration,
- CodeQL analysis,
- dependency review.

The development workflow uses:

- **pytest** for automated testing,
- **mypy** for static type checking,
- **Ruff** for linting and code quality.

The project also explicitly keeps CV-derived artifacts and API credentials outside public version control because workflow outputs may contain personal information.

## Key engineering decisions

**Deterministic where requirements are hard.**  
Location, seniority, language and explicit inclusion/exclusion rules should not depend on model judgment.

**Semantic where wording varies.**  
Embeddings help match related skills and responsibilities even when job descriptions use different terminology.

**LLMs where interpretation adds value.**  
Candidate extraction, search planning and explanation benefit from language understanding, but outputs remain structured.

**Grounding after generation.**  
LLM findings must point back to known evidence and unsupported claims are rejected.

**Provider interfaces over vendor-specific logic.**  
Both LLM and job-retrieval layers can change without rewriting downstream workflow logic.

**Bounded agents over uncontrolled loops.**  
Search replanning is useful, but agent behavior needs explicit limits.

**Benchmarks over intuition.**  
Ranking configuration and embedding-model choices are evaluated quantitatively instead of selected from subjective examples.

## Current limitations

CareerMatch remains a **technical-alpha project**, not a production job-search platform.

Current limitations include:

- benchmark scenarios are synthetic and intentionally limited in scale,
- job-provider coverage is still limited,
- local LLM evaluation can be slow,
- external provider behavior and listing availability can change,
- persistence, authentication and deployment are not yet complete,
- recommendations should support—not replace—human career decisions.

These limitations define the next engineering work rather than being hidden behind the AI layer.

## What I contributed

I designed and implemented the system end to end, including:

- FastAPI application architecture and typed domain models,
- CV-to-candidate-profile extraction,
- natural-language preference extraction and deterministic validation,
- LangGraph search planning and bounded replanning,
- Arbeitnow, Adzuna and Jooble integrations,
- provider normalization and composite retrieval,
- deterministic filtering and reason codes,
- multilingual SentenceTransformer ranking,
- hybrid score design and embedding-model comparison,
- evidence-bundle construction,
- structured multi-LLM provider abstraction,
- evidence-grounded LLM evaluation and deterministic grounding validation,
- Streamlit end-to-end workflow,
- reusable workflow-state export/import,
- frozen development and holdout benchmark design, ablation evaluation and regression analysis,
- automated tests, CI and code-quality tooling.

## What this project demonstrates

CareerMatch is my strongest example of **machine-learning and AI systems engineering**.

It demonstrates that I can work across the boundary between ML and software engineering:

- design a typed backend API,
- integrate external data providers,
- use embeddings for multilingual retrieval,
- combine deterministic and learned ranking signals,
- orchestrate bounded agentic workflows,
- abstract multiple LLM providers,
- build evidence-grounded generation pipelines,
- evaluate retrieval/ranking quantitatively,
- expose the system through a usable frontend,
- test and maintain the implementation as a modular application.

Most importantly, the project is built around the idea that **useful AI systems need explicit constraints, measurable evaluation and verifiable evidence—not only good prompts.**

**Core technologies:** Python · FastAPI · Pydantic · LangGraph · SentenceTransformers · Hugging Face · OpenAI · Gemini · Ollama · Streamlit · pytest · mypy · Ruff · GitHub Actions

[View source on GitHub →](https://github.com/LittleBigPluton/career-match-agent)

[← Back to projects](/#projects)
