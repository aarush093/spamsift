# SPAMSIFT · PLAN.md

Execution gates. One day equals one Claude Code session of roughly 2 to 3 focused hours. Claude Code must read CLAUDE.md fully before starting any day and must never touch a future day's scope.

Hard deadline: live and fully documented by Saturday 22 August 2026 night. Sunday 23 August is the only planned buffer day, use it exclusively if something slipped. The longer buffer before 9 September exists only for genuine emergencies, never plan into it.

Schedule note: if Day 0 slips today, it merges into Tuesday morning before Day 1 and the deadline still holds.

## Prerequisites Aarush does manually once, about 15 minutes, before the Day 0 prompt

1. Python 3.12 installed and on PATH. The venv must be created with py -3.12 -m venv .venv. Local Python and the Docker base image must stay on the same minor version.
2. git configured, `gh auth login` completed.
3. Hugging Face account created, a write access token generated at huggingface.co/settings/tokens, exported as HF_TOKEN in the shell profile. Note the HF username, it may differ from the GitHub username.
4. An empty folder named spamsift opened in Claude Code.

## Day 0 · Monday 17 August · Scaffold · 60 to 90 minutes

Scope:
1. Create the full repo layout from CLAUDE.md with empty placeholder modules where needed.
2. requirements.txt with exact pinned current stable versions of every dependency in the stack list. Create .venv and install.
3. config.yaml holding: seed 42, data URLs and filenames from the dataset contract, tfidf params, model params, precision target 0.98, threshold fallback 0.5, artifact path, test size 0.2.
4. .gitignore covering data/, .venv, __pycache__, .pytest_cache, .ruff_cache, *.pyc. models/ and reports/ stay tracked.
5. Minimal ruff config in pyproject.toml, line length 100.
6. MIT LICENSE, README stub with project one liner and a Work in progress note.
7. git init, first commit, `gh repo create spamsift --public --source . --push`.

Acceptance checklist:
- [x] `pip install -r requirements.txt` completes clean inside .venv
- [x] `ruff check .` passes
- [x] Repo live at github.com/aarush093/spamsift with the initial commit
- [x] CLAUDE.md and PLAN.md committed at repo root

Prompt to paste:

```
Read CLAUDE.md fully, then PLAN.md. Execute Day 0 only. Verify every item on the Day 0 acceptance checklist by actually running the commands, tick them in PLAN.md, append a progress log entry, commit and push. Do not start Day 1. Finish with a five line summary.
```

## Day 1 · Tuesday 18 August · Data ingestion and preprocessing

Scope:
1. src/ingest.py per the dataset contract: download the five archives to data/raw with urllib, extract, parse every message, build data/processed/emails.csv, write reports/data_summary.json. Idempotent, skips downloads that already exist.
2. src/preprocess.py per the preprocessing contract with clean_text.
3. tests/test_preprocess.py with at least six behavior tests: url replacement, email replacement, number replacement, lowercasing, whitespace collapse, empty string safety.
4. Run ingestion end to end once and sanity check the output.

Acceptance checklist:
- [ ] emails.csv has at least 5000 rows after dedupe with spam ratio between 25 and 40 percent
- [ ] data_summary.json written with counts, spam ratio and mean lengths
- [ ] All tests pass, ruff clean
- [ ] Committed and pushed

Prompt to paste:

```
Read CLAUDE.md fully, then PLAN.md. Execute Day 1 only. Follow the dataset contract and preprocessing contract exactly. Run the full ingestion for real and verify the acceptance numbers from the actual emails.csv, not from assumptions. Tick the checklist, append a progress log entry, commit and push. Do not start Day 2. Finish with a five line summary.
```

## Day 2 · Wednesday 19 August · Training, evaluation and error analysis

Scope:
1. src/train.py per the modeling contract: stratified split, FeatureUnion features, 5 fold CV comparison of the three models, threshold selection via out of fold probabilities, final fit, single test evaluation, write reports/metrics.json, save confusion matrix and precision recall curve pngs to reports/figures, dump the artifact dict to models/model.joblib.
2. src/predict.py exposing load_artifact() and predict_text(text) returning label, probability and threshold using the saved artifact.
3. src/evaluate.py: top 20 most spam indicating and top 20 most ham indicating features from the deployed model's coefficients plus the 10 highest confidence wrong predictions on the test set, written to reports/error_analysis.md.
4. tests/test_predict.py: artifact loads, both canonical examples from CLAUDE.md classify correctly, probability is between 0 and 1.

Acceptance checklist:
- [ ] Spam F1 on the held out test set is at least 0.95, real number recorded in metrics.json
- [ ] models/model.joblib committed and loads clean in a fresh Python process
- [ ] error_analysis.md written with real examples
- [ ] All tests pass, ruff clean, committed and pushed

Prompt to paste:

```
Read CLAUDE.md fully, then PLAN.md. Execute Day 2 only. Follow the modeling contract exactly, in particular the predict_proba requirement, the threshold procedure and the single use of the test set. Train for real, report the real numbers, never fabricate metrics. If the F1 bar is missed after the two allowed fallbacks, report honestly and log it. Tick the checklist, append a progress log entry, commit and push. Do not start Day 3. Finish with a five line summary.
```

## Day 3 · Thursday 20 August · API, UI and Dockerfile

Scope:
1. app/main.py per the API contract.
2. app/static/index.html per the UI description in CLAUDE.md.
3. tests/test_api.py with httpx and FastAPI TestClient: health check, correct classification of both canonical examples through the endpoint, 422 on empty text, 422 on missing field, response schema keys present.
4. Dockerfile: python:3.12-slim base, install requirements, copy src app models config, run uvicorn on 0.0.0.0:7860. Plus .dockerignore excluding data, .venv, tests, .git.
5. Verify locally with uvicorn, classify both canonical examples through the real running server and through the browser UI once.

Acceptance checklist:
- [ ] Local uvicorn serves the UI and both canonical examples classify correctly end to end
- [ ] /docs renders with the example request
- [ ] All tests pass, ruff clean, committed and pushed
- [ ] Local Docker build was not required and was not attempted

Prompt to paste:

```
Read CLAUDE.md fully, then PLAN.md. Execute Day 3 only. Follow the API contract exactly including port 7860. Test against the real running server, not only the TestClient. Do not build the Docker image locally, Hugging Face builds it remotely on Day 4. Tick the checklist, append a progress log entry, commit and push. Do not start Day 4. Finish with a five line summary.
```

## Day 4 · Friday 21 August · CI, README and live deployment

Scope:
1. .github/workflows/ci.yml: on push and pull request, Python 3.12, install requirements, ruff check, pytest. Must go green on the actual GitHub run.
2. Full README per the definition of done in CLAUDE.md, all metric numbers copied from reports/metrics.json by hand, mermaid diagram of the pipeline, CI badge, curl example, screenshots optional.
3. Add Hugging Face Space YAML frontmatter to the top of README.md: title SpamSift, sdk docker, app_port 7860. GitHub renders this block as a small table at the top, acceptable tradeoff, log it.
4. Create the Space with huggingface_hub using HF_TOKEN: repo type space, sdk docker, under the HF username from prerequisites. Add it as git remote hf, push main.
5. Wait for the Space build, then run live smoke checks with curl: /health plus both canonical examples through /predict. Put the live URL at the very top of the README.

Acceptance checklist:
- [ ] GitHub Actions run is green on the latest commit
- [ ] Space status is Running and both canonical examples classify correctly on the live URL
- [ ] README complete per definition of done with the live link at the top
- [ ] Committed and pushed to both origin and hf

Prompt to paste:

```
Read CLAUDE.md fully, then PLAN.md. Execute Day 4 only. First confirm HF_TOKEN is present in the environment, if it is missing stop and ask, that is the one allowed question. Deploy, then verify the live URL with real curl calls against /health and /predict using the canonical examples. A day where the live checks were not actually run is not done. Tick the checklist, append a progress log entry, push to origin and hf. Do not start Day 5. Finish with a five line summary.
```

## Day 5 · Saturday 22 August · Interview pack, resume assets and release

Scope:
1. docs/INTERVIEW_PREP.md covering, in plain fresher friendly language with the project's real numbers: two minute project pitch, TF-IDF intuition and formula, why linear models suit sparse text, MultinomialNB assumptions versus LogisticRegression, why LinearSVC was excluded from deployment, stratified splitting, why the whole flow lives in one Pipeline and how that prevents leakage, class imbalance and why precision recall F1 beat accuracy here, how the threshold was chosen and the precision recall tradeoff, what char_wb n-grams catch that word grams miss, three findings from error_analysis.md, the serving architecture from request to response, what breaks in production covering drift, adversarial spam and retraining, then 25 likely interview questions with short answers.
2. docs/RESUME_BULLETS.md: three resume bullets with real numbers covering data scale, metrics, deployment and CI, plus one two line project blurb for the resume projects section.
3. docs/LINKEDIN_POST.md: two post variants built from Aarush's existing drafts, keeping his voice. The storytelling variant must lead with working from raw email files rather than a clean CSV and the scoping before modeling angle. Include the live demo and repo links and his existing hashtag set.
4. Final read of the README top to bottom, fix anything stale, tag v1.0.0 and push the tag.

Acceptance checklist:
- [ ] All three docs exist with real numbers, zero placeholders
- [ ] v1.0.0 tag pushed
- [ ] Live URL, GitHub Actions and tests all green on final check

Prompt to paste:

```
Read CLAUDE.md fully, then PLAN.md. Execute Day 5 only. Pull every number in the docs from reports/metrics.json and reports/error_analysis.md, never invent figures. Write for a fresher explaining to an interviewer, simple and confident. Tick the checklist, append a progress log entry, tag v1.0.0, push everything. Finish with a five line summary of the final state of the project.
```

## Stretch items, only if a day finishes early, never at the cost of the schedule

- Measure average /predict latency locally over 50 requests and put the number in the README.
- One paragraph in error_analysis.md on how the model performs on the hard_ham subset specifically.
- A GitHub Actions job that pushes main to the hf remote automatically using an HF_TOKEN repo secret.

## Progress log

Claude Code appends one entry at the end of every session: date, what shipped, decisions taken, deviations and fallbacks with reasons.

### Day 0 · 17 August 2026 · Scaffold

Shipped: full repo layout from CLAUDE.md with empty placeholder modules (src/*.py, app/main.py); requirements.txt with exact pins of current stable versions (pandas 3.0.5, scikit-learn 1.9.0, beautifulsoup4 4.15.0, fastapi 0.141.1, uvicorn 0.52.3, pydantic 2.13.4, joblib 1.5.3, PyYAML 6.0.3, matplotlib 3.11.1, pytest 9.1.1, httpx 0.28.1, ruff 0.16.3); config.yaml with seed, data URLs/filenames, tfidf/model params, precision target 0.98, threshold fallback 0.5, artifact path, test size 0.2; .gitignore; pyproject.toml with ruff line length 100; MIT LICENSE; README stub. Installed deps into pre-existing .venv (Python 3.12.10).

Verified: `pip install -r requirements.txt` clean, `ruff check .` passes, repo live at github.com/aarush093/spamsift with initial commit, CLAUDE.md and PLAN.md at repo root.

Decisions/deviations:
- Renamed default branch master -> main so it matches the `push main` references in Day 4. GitHub default branch updated accordingly.
- Added `.claude/` to .gitignore (machine-local Claude Code settings, not project content). This is beyond the .gitignore items listed in the Day 0 scope; logged here per the dependency/deviation rule.
- Used .gitkeep files in app/static, tests, reports/figures and models so these tracked directories survive the commit while still empty.
- No new dependencies added beyond the stack list.
