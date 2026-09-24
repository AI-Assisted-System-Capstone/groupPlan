# Safety Event Review Platform: Project Spec

## Overview

1. **Grades severity:** sorts it into `none`, `some`, or `serious` harm.
2. **Extracts key facts:** what happened, contributing factors, medications involved.
3. **Finds similar reports:** links related past cases and explains why they matched.
4. **Groups into themes:** clusters reports into topics, shows how those change over time, and flags unusual spikes and new kinds of events.



## Roles

### 1. Frontend


**What to do:**
- Report detail page showing the text, severity, extracted facts, theme, and a "new kind of event" badge when the novelty flag is on.
- Similar-cases panel that lets users click from one report to a related one and shows *why* they're linked.
- Search, with filters the pattern engine already supports: date lookback window, location, and service.
- Theme and trend dashboard: themes over time (weekly/monthly), highlighted spikes, and filters by department and severity.
- The pattern engine already outputs a 2-D map of all reports (`embedding_projection.csv`). This could become an interactive "map of events" view.
- Run usability testing with a few people (think-aloud sessions plus a short survey such as SUS).


**Depends on:** Backend API.

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
- Database schema (Postgres + pgvector suggested)
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

