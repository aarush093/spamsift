# SPAMSIFT · CLAUDE.md

This is the master context file for Claude Code. Read it fully at the start of every session before touching anything. PLAN.md in the repo root holds the day by day execution gates and the exact scope for each session. Never work outside the current day's scope in PLAN.md.

## What this project is

SpamSift is an end to end spam email classification system built in one week as a placement portfolio project by Aarush (github.com/aarush093), a final year B.Tech IT student targeting ML Engineer fresher roles. Raw emails from the SpamAssassin public corpus are downloaded, parsed, cleaned and vectorized with TF-IDF, classified with linear models, evaluated with imbalance aware metrics, then served through a FastAPI service with a minimal web UI, containerized with Docker and deployed live on Hugging Face Spaces, with pytest coverage and GitHub Actions CI.

## Why it exists, optimize for these in order

1. Hard deadline: everything live and documented by Saturday 22 August 2026 night, Sunday 23 August is the only buffer day. Nothing pushes past Sunday.
2. Resume and GitHub proof of ML engineering fundamentals, not another notebook project.
3. A live demo URL plus Swagger docs a recruiter can click within 10 seconds of opening the README.
4. Interview readiness: every technical choice must be simple enough for a fresher to explain from first principles. If a choice cannot be defended in two plain sentences, pick the simpler alternative.

## Non goals, never build these

No deep learning, no transformers, no embeddings. No MLflow, DVC, Airflow, Kubernetes, databases, caching layers, auth, user accounts, monitoring dashboards, retraining jobs. No hyperparameter search beyond the tiny grid defined in the modeling contract. These appear only as one line mentions in the README future work section.

## Hard rules for all code

1. Zero comments and zero docstrings anywhere in the codebase. Names carry all meaning. If a line seems to need a comment, rename variables or split the function instead.
2. Simple direct code. Plain functions and scripts. No classes except pydantic request and response models in the API. No clever one liners, no decorators beyond FastAPI's own, no premature abstraction, no design patterns.
3. Target under 120 lines per file. If a file crosses that, split it by responsibility.
4. Only the dependencies pinned in requirements.txt. Adding any new dependency requires a one line justification appended to the progress log in PLAN.md.
5. Deterministic everywhere: seed 42, all tunable values live in config.yaml, no magic numbers inside function bodies.
6. Everything model related lives inside a single sklearn Pipeline so transformers are fitted only on training data. Never call fit or fit_transform on anything using test data.
7. Every script runs from repo root as `python -m src.<name>` with no arguments required for the default path.
8. Never delete or rewrite working committed code unless the current day's scope explicitly requires it. No drive by refactors.
9. Type hints only where they cost nothing: function signatures in src and app. Nothing exotic.

## Repo layout

```
spamsift/
  CLAUDE.md
  PLAN.md
  README.md
  LICENSE
  requirements.txt
  config.yaml
  Dockerfile
  .dockerignore
  .gitignore
  .github/workflows/ci.yml
  data/                 gitignored, rebuilt by src.ingest
    raw/
    processed/
  models/
    model.joblib        committed
  reports/
    data_summary.json
    metrics.json
    error_analysis.md
    figures/
  src/
    __init__.py
    ingest.py
    preprocess.py
    train.py
    evaluate.py
    predict.py
  app/
    __init__.py
    main.py
    static/index.html
  tests/
    test_preprocess.py
    test_predict.py
    test_api.py
  docs/
    INTERVIEW_PREP.md
    RESUME_BULLETS.md
    LINKEDIN_POST.md
```

## Stack

Python 3.12. Dependencies: pandas, scikit-learn, beautifulsoup4, fastapi, uvicorn, pydantic, joblib, pyyaml, matplotlib, pytest, httpx, ruff. Pin exact current stable versions in requirements.txt on Day 0 and never change pins after Day 2. Frontend is one static index.html with vanilla JS and inline CSS, no frameworks, no build step. Docker exists only as the Dockerfile that Hugging Face builds remotely, local Docker is never required.

## Dataset contract

Source: SpamAssassin public corpus at https://spamassassin.apache.org/old/publiccorpus/

Download exactly these five archives:

| file | label | approx count |
|---|---|---|
| 20030228_easy_ham.tar.bz2 | ham | 2500 |
| 20030228_easy_ham_2.tar.bz2 | ham | 1400 |
| 20030228_hard_ham.tar.bz2 | ham | 250 |
| 20030228_spam.tar.bz2 | spam | 500 |
| 20050311_spam_2.tar.bz2 | spam | 1397 |

Roughly 6047 messages at about 31 percent spam before deduplication. If any URL returns 404, fetch the directory listing at the base URL and adjust the date prefix, then log the change in the progress log.

Parsing rules in src/ingest.py:
1. Read each file with `email.message_from_bytes` using errors tolerant decoding.
2. Body extraction: walk the parts, prefer the last text/plain part, else take text/html and strip tags with BeautifulSoup using the built in html.parser.
3. Final text is subject plus a space plus body.
4. Drop exact duplicates on the final text, drop rows with empty text.
5. Output data/processed/emails.csv with columns text and label where label is 1 for spam and 0 for ham.
6. Write counts, spam ratio and mean text length per class into reports/data_summary.json.

## Preprocessing contract

src/preprocess.py exposes clean_text(text) that lowercases, replaces every URL with the token urltoken, every email address with emailtoken, every standalone number with numtoken, collapses all whitespace to single spaces and strips leading and trailing space. It does nothing else. Punctuation handling is left to the TF-IDF tokenizer. clean_text is applied inside the sklearn Pipeline via FunctionTransformer so it runs identically at training and serving time.

## Modeling contract

1. Split: stratified 80/20 train test split with seed 42, done once in train.py.
2. Features: FeatureUnion of two TfidfVectorizers, one with word 1 to 2 grams with min_df 2 and sublinear_tf true, one with char_wb 3 to 5 grams with min_df 2.
3. Models compared with stratified 5 fold cross validation on the training set only: MultinomialNB, LogisticRegression with max_iter 2000, LinearSVC. Record mean and std of F1 for the spam class for each.
4. The deployed model must expose predict_proba, so choose between LogisticRegression and MultinomialNB for the final artifact even if LinearSVC scores highest. Default to LogisticRegression unless MultinomialNB beats it by more than 0.01 F1 in cross validation. State this reasoning in the README.
5. Threshold: get out of fold probabilities on the training set with cross_val_predict, then pick the lowest threshold that achieves spam precision of at least 0.98, maximizing recall under that constraint. Fall back to 0.5 if 0.98 is unattainable and log it.
6. Fit the chosen pipeline on the full training set, evaluate exactly once on the held out test set.
7. Report on test: accuracy, precision, recall and F1 for the spam class at the chosen threshold, ROC AUC, PR AUC and the confusion matrix. Accuracy is never the headline number.
8. Acceptance bar: spam F1 on test of at least 0.95. If missed, try in order: C grid {0.1, 1, 10} for LogisticRegression, then min_df 1. Stop after that and report honestly.
9. Save one artifact at models/model.joblib: a dict with keys pipeline, threshold, sklearn_version, trained_at_utc. Commit it to git.
10. Write reports/metrics.json with the cross validation table and all test metrics. Save confusion matrix and precision recall curve figures into reports/figures.

## API contract

app/main.py, FastAPI:

- POST /predict accepts {"text": str}, text must be 1 to 50000 chars, returns {"label": "spam" or "ham", "spam_probability": float rounded to 4 places, "threshold": float}.
- GET /health returns {"status": "ok", "model_loaded": true}.
- GET / serves app/static/index.html.
- Swagger stays enabled at /docs with one example request in the schema.
- The artifact loads exactly once at startup. Uvicorn binds 0.0.0.0 port 7860 because Hugging Face Docker Spaces expect 7860.

The UI is a single centered card: heading, textarea, one Classify button, a result panel showing the label in green for ham or red for spam with the probability as a percentage. Clean and minimal, dark background, system font stack.

## Canonical verification examples

Use these exact strings in tests and in live smoke checks after deployment:

- spam: "CONGRATULATIONS! You have been selected to receive a FREE $1000 Walmart gift card. Click http://claim-prize-now.example.com to verify your account and claim your reward before midnight!!!"
- ham: "Hi team, attaching the minutes from Tuesday's review meeting. Please send me your edits by Friday so I can circulate the final version before the sprint planning call."

The deployed model must label these correctly. If it does not, the day is not done.

## Git rules

Conventional commit prefixes: feat, fix, test, docs, chore. Small commits per logical unit, push to origin at least once per session end. Repo is public at github.com/aarush093/spamsift. The Hugging Face Space is a second git remote named hf added on Day 4. Never commit data/, secrets or tokens. models/model.joblib and everything in reports/ are committed on purpose.

## Definition of done for the whole project

- ruff check passes and all pytest tests pass locally and in GitHub Actions.
- The live Hugging Face Space URL answers /health and classifies both canonical examples correctly.
- README contains: one paragraph problem statement, dataset description, a mermaid pipeline diagram, the model comparison table and test metrics with real numbers from reports/metrics.json, local run instructions, API usage with a curl example, the live demo link, a CI badge and a short future work section that mentions drift monitoring, retraining and transformer baselines as deliberate exclusions.
- docs/INTERVIEW_PREP.md, docs/RESUME_BULLETS.md and docs/LINKEDIN_POST.md exist with real numbers, no placeholders.
- Repo tagged v1.0.0.

## Working style with Aarush

Execute only the current day from PLAN.md. Make reasonable decisions autonomously and record every notable decision, deviation or fallback in the progress log at the bottom of PLAN.md. Ask a question only when truly blocked, meaning a missing credential or a destructive ambiguity. If a single bug consumes more than 25 minutes, implement the simplest working fallback and log it. End every session with tests run, work committed and pushed, the progress log updated and a five line summary of what a recruiter can now see that they could not see yesterday.
