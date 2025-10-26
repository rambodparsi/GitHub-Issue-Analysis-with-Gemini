# GitHub Release Diff → Commits → PRs → Issues (Notebook-first)

This repo is a tiny, notebook-first pipeline that:
1) fetches tags for a repo,  
2) builds adjacent tag pairs and diffs,  
3) maps commits → PRs,  
4) fetches PRs (with labels) and linked Issues (with labels),  
5) builds **capsules** per pair consolidating commit ↔ PR ↔ Issue.

Outputs are written under `/<name>/...` where `<name>` is the repo name (e.g., `mastodon`).

## Notebooks (run in order)

- `notebooks/m1_fetch_tags.ipynb` → writes `<name>/tags/tags.all.json`  
- `notebooks/m2_compare_vA_vB.ipynb` → writes `<name>/pairs/<series>/<base...compare>.compare.json`  
- `notebooks/m3_commits.ipynb` → writes `<name>/commits/<series>/<pair>.commits.json`  
- `notebooks/m4_pulls.ipynb` → writes `<name>/pulls/<series>/<pair>.pulls.json` (**adds PR labels**)  
- `notebooks/m5_issues.ipynb` → writes `<name>/issues/<series>/<pair>.issues.json` (**adds Issue labels**)  
- `notebooks/m6_commit_pr_issue.ipynb` → writes `<name>/commits_pr_issue/<series>/<pair>.tarce_artifacts.json`  

> All notebooks read a single shared config: `config/config.yaml`, and a local `.env` for the token.

## Quick start

```bash
# 1) set up Python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# 2) create your local .env from the example and add your token
cp .env.example .env
# then edit .env and set GITHUB_TOKEN=...

# 3) open the notebooks in Jupyter / VS Code and run each top-to-bottom
