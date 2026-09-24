# Safety Event Review Platform: Project Spec

## Overview

We're building a prototype platform that helps hospital staff review patient safety event reports. For each report, the system:

1. **Grades severity:** sorts it into `none`, `some`, or `serious` harm.
2. **Extracts key facts:** what happened, contributing factors, medications involved.
3. **Finds similar reports:** links related past cases and explains why they matched.
4. **Groups into themes:** clusters reports into topics, shows how those change over time, and flags unusual spikes and new kinds of events.

Everything is packaged in a Docker container so the sponsor can run it on real hospital data inside their secure environment. **Real data never leaves the hospital.** We develop only on the synthetic dataset.

## What already exists

| Repo | What it does | Status |
|---|---|---|
| `3-bucket-grader` | Severity grading (TF-IDF baseline + DistilBERT) | Works on the 80k dataset. `serious` recall is 0.82 and needs to go higher |
| `midas-pattern-phase-local` | Similar-case search, theme clustering, weekly/monthly trends, spike detection, novelty flag | Works end to end, but only tested on a 23-report sample |

Nothing exists yet for extraction, the backend, security, or the sponsor container. The frontend is in progress.

**First integration task:** Move both repos into one shared repo (layout below) so they use the same data loading, the same columns, and the same splits.

## How the pieces fit

```
Synthetic reports
      │
      ▼
[Ingest + validate] ──► [Database] ◄──► [Backend API] ◄──► [Frontend]
                             ▲
      ┌──────────────┬───────┴──────┬──────────────┐
  [Severity]   [Extraction]   [Similar cases]   [Themes + trends]
  (grader)        (new)       (pattern engine)  (pattern engine)
```

Every model reads a report and writes its results into one shared format (see below). The backend serves that format to the frontend.

## Shared data format ("enriched report")

Everyone builds against this shape. If you need to change it, open a PR and get the team to agree first. Several fields line up with what the pattern engine already outputs (`cluster`, `cluster_name`, `similarity`, `novelty_score`).

```json
{
  "report_id": "string",
  "event_date": "YYYY-MM-DD",
  "location": "string",
  "service": "string",
  "event_type": "string",
  "text": {
    "event_comments": "string",
    "manager_comments": "string",
    "unit_actions_taken": "string"
  },
  "severity": {
    "bucket": "none | some | serious",
    "confidence": 0.0,
    "model_version": "string",
    "explanation": ["top words or phrases that drove the prediction"]
  },
  "extracted": {
    "contributing_factors": ["string"],
    "medications": ["string"]
  },
  "similar_cases": [
    {
      "report_id": "string",
      "similarity": 0.0,
      "snippet": "string",
      "reasons": ["shared medication: heparin", "same event type: medication error"]
    }
  ],
  "theme": { "cluster_id": 0, "cluster_name": "string" },
  "novelty": { "score": 0.0, "threshold": 0.0, "is_novel": false }
}
```

## Repo layout

```
pipeline/          ingest, validation, shared data prep (one loader for everyone)
models/severity/   from 3-bucket-grader
models/extraction/ new
models/patterns/   from midas-pattern-phase-local (search, clusters, trends, novelty)
api/               backend service
frontend/          website
eval/              shared evaluation scripts + results table
deploy/            Dockerfile, docker-compose, sponsor container
docs/              architecture diagrams, setup guide
```

---

## Roles

### 1. Frontend

**Goal:** A clean, easy-to-use website for exploring safety reports.

**What to do:**
- Report detail page showing the text, severity, extracted facts, theme, and a "new kind of event" badge when the novelty flag is on.
- Similar-cases panel that lets users click from one report to a related one and shows *why* they're linked.
- Search, with filters the pattern engine already supports: date lookback window, location, and service.
- Theme and trend dashboard: themes over time (weekly/monthly), highlighted spikes, and filters by department and severity.
- The pattern engine already outputs a 2-D map of all reports (`embedding_projection.csv`). This could become an interactive "map of events" view.
- Run usability testing with a few people (think-aloud sessions plus a short survey such as SUS).

**Done when:** A user can find a report, understand it, jump to related cases, and spot a trend without help.

**Depends on:** Backend API. Until it's ready, build against mock data in the shared format, or the CSV outputs from the pattern engine.

---

### 2. Severity + Extraction

**Goal:** Reliable severity grading that catches serious cases, plus structured facts pulled from each report.

**Starting point:** `3-bucket-grader`. The model works; this role gets it ready to plug in and builds extraction (which doesn't exist yet).

**What to do:**
- **Severity:**
  - **Check for leakage first.** The grader uses `manager_comments` and `unit_actions_taken`, which are written *after* a report is reviewed. The pattern engine deliberately excludes these for that reason. If triage happens when a report first comes in, those fields won't exist yet, so also test the model using only `event_comments` and see how much performance drops. Ask the sponsor which setup matches real use.
  - Threshold sweep. Plot recall/precision against the cutoff and pick one that favors catching `serious` and `some` cases, since the sponsor wants high sensitivity.
  - Re-test with a time-based split (e.g. train Jan–Aug, validate Sep, test Oct).
  - Add a per-report explanation (the top words that drove the prediction).
  - Wrap the model in a simple `predict(report) -> severity` function the backend can call.
- **Extraction:**
  - Pull out contributing factors and medications from the narrative text.
  - These feed Similar Cases' "why linked" reasons and Themes' medication clustering, so agree on the output early.

**Done when:** Both models run through one function call, write to the shared format, and have results logged in `eval/`.

---

### 3. Similar Cases

**Goal:** Given a report, find the most related past reports and explain the match.

**Starting point:** `PatternEngine.search()` in the pattern engine. It already does meaning-based search (MiniLM embeddings), top-k results, lookback windows, location/service filters, and a novelty score. What's missing is the explanation of *why* reports matched, and testing at real scale.

**What to do:**
- Run the search on the full synthetic dataset (it has only been tested on 23 reports).
- Add "why linked" reasons to each match: shared medications, shared contributing factors, same event type or location, and overlapping key phrases. Right now each match only returns a similarity score and a text snippet.
- Consider hybrid search (keyword + meaning) so exact terms like drug names aren't missed.
- Build a small labeled set. For a sample of reports, mark whether each top-5 match is truly related, then measure Precision@5 / NDCG@5. The pattern engine README says the same thing: without labels, you can't claim one model is best.
- Compare MiniLM against one or two other local embedding models (e.g. a clinical or biomedical model) using that labeled set.
- Tune the novelty threshold on the full dataset, and check that "novel" reports actually look unusual.

**Done when:** Every report has top-k similar cases with readable reasons, retrieval quality is measured, and the model choice is backed by numbers.

**Depends on:** Extraction output for the richer reasons (phrase and event-type reasons work fine before that).

---

### 4. Themes + Trends (+ Stretch Goal)

**Goal:** Meaningful topics and clear trends over time.

**Starting point:** The pattern engine already clusters reports (HDBSCAN or KMeans), auto-names clusters from top keywords, builds weekly and monthly trend tables with zero-filled gaps, and flags spikes. It hasn't been judged on real-sized data yet.

**What to do:**
- Run clustering on the full synthetic dataset and check whether the themes make sense to a human reader.
- Improve theme names. Keyword lists like "pump / infusion / rate" are a start; aim for short readable labels, with a person able to rename them.
- Check spike detection on the full data. Are the flagged spikes real, and are obvious ones being missed?
- Medication clustering by department, to spot over-ordering or over-prescribing (a sponsor request). Uses Extraction's medication output.
- Make sure trend output is in the shape the frontend charts need.
- **Stretch goal (second half of project):** A feasibility study on finding safety events nobody reported by scanning clinical notes (public or synthetic data only), then drafting MIDAS-style summaries for human review. The novelty detector and similarity search can likely be reused here.

**Done when:** Themes are readable and checked by a person, trends and spikes show on the dashboard, and the stretch goal has at least a written feasibility assessment.

---

### 5. Backend

**Goal:** The engine that stores everything and connects the models to the website.

**What to do:**
- **One data loader for everyone.** The grader reads a parquet file and the pattern engine reads a CSV with its own column-name handling (`normalize_schema`). Merge these into one shared loader in `pipeline/` so every model sees the same reports and columns.
- Ingestion pipeline: load synthetic data, validate it (missing fields, bad dates, empty text), and reject bad rows with clear errors.
- Database schema (Postgres + pgvector suggested) based on the shared format.
- API endpoints (FastAPI suggested):
  - `GET /reports` with search and filters
  - `GET /reports/{id}` with full enriched report
  - `GET /reports/{id}/similar`
  - `GET /themes` and `GET /themes/trends`
  - `POST /reports` to ingest and run the models on a new report
- Wrap each model behind the API. The pattern engine is already a Python class with `fit()` and `search()`, so it should wrap cleanly.
- `docker compose up` should start the whole app locally.

**Done when:** A new report can be posted, runs through all models, gets saved, and shows up in the frontend.

---

### 6. Security + Packaging

**Goal:** Make the system safe for healthcare data and runnable inside the hospital.

**What to do:**
- **Access control:** Logins with roles (e.g. reviewer, admin) that limit who sees what.
- **Audit logging:** Record who viewed or changed what, and when.
- **Data separation:** Keep dev and eval data separate, with no mixing of training and test sets.
- **Data minimization:** Only use the columns we actually need. The pattern engine's leakage protections are a good model to follow.
- **Model versioning:** Track which model version produced each prediction (MLflow or similar).
- **Reproducibility:** Pin dependencies and fix random seeds so anyone can rerun results.
- **Offline models:** The pattern engine already runs fully offline with a cached embedding model. Keep that true for every model, since the hospital environment may not have internet access.
- **Sponsor container:**
  - A Docker image that runs our models on their data and outputs **aggregate results only** (metrics, counts, non-identifiable error examples).
  - Clear instructions for how the sponsor runs it.
  - Send the first version around week 5–6, even if the models are rough.
  - Note: some pattern engine outputs (`pattern_reports.csv`, `retrieval_neighbors.csv`, `embeddings.npy`) contain report text or data derived from it. Those must stay inside the hospital. The container should only send back summaries.

**Done when:** The sponsor can run the container with one command and send back results, and the privacy controls are documented.

---

## Shared responsibilities

- **Evaluation:** Everyone logs their model results in the shared table in `eval/`, using the same data and split.
- **Documentation:** Everyone writes their own section of the final report, setup guide, and handoff docs.
- **Final deliverables:** Technical report, demo, poster, and one-page summary. Split these up in week 10.

## Timeline

| Weeks | Focus |
|---|---|
| 1–2 | Merge the two repos, one shared data loader, agree on the shared format |
| 3–5 | Working end to end (ugly is fine) |
| 5–6 | First container sent to the sponsor |
| 6–9 | Improve models, explanations, UI, and dashboards |
| 10–12 | Full evaluation, report, poster, demo |

## Ground rules

- **Never commit data.** No parquet, CSV, embeddings, or output files that contain report text.
  - The pattern engine's `.gitignore` is good but doesn't block `*.parquet`, which is the format the grader uses. Add it when merging.
- **No hardcoded file paths.** Use a config file or environment variables (the grader currently points at a local Downloads folder).
- **Work on branches**, open PRs, and get one teammate's review before merging.
- **One issue per task** on the project board.
- **Weekly sync** to share progress and blockers.

## Open questions for the sponsor

- When would triage happen: as soon as a report is submitted, or after the manager adds comments? This decides which text fields the severity model is allowed to use.
- Some reports say "no harm noted" but carry a harm grade. Is that a labeling issue, or should the model learn it?
- Can reviewers help label whether similar-case matches are truly related? Even a small set would let us measure retrieval properly.
- What hardware will the container run on (CPU/GPU, memory, internet access)?
- What format do they want results returned in?
- Can we use their preliminary models as baselines?
