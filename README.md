# GitHub developers’ issue label vs. Gemini issue label prediction
## Tracing GitHub artifacts end-to-end and classifying UI issues with Gemini
## Case study on Mastodon web application

### Goal
Help GitHub developers label issues more consistently and act on them faster by using an LLM (Gemini) to predict UI vs. Other for real issues, then trace each labeled issue back to its PR(s) and commit(s). 
The pipeline turns a repo’s raw history into linked, reviewable artifacts (commit → PR → issue) so teams can compare human labels with model predictions, spot gaps, and streamline triage and fixes.

### GitHub artifacts tracing
This step turns raw GitHub history into clean, linked “capsules” so every change can be followed from the commit that made it, to the pull request that shipped it, to the issue it resolved.

Questions can be answered:
- How much code ships through PRs?
- What share of changes close issues?
- Which releases have more UI-related fixes?



<img width="2084" height="582" alt="conversion grouped" src="https://github.com/user-attachments/assets/2be76a7c-4338-43f3-a85d-3a4f277694a2" />


The series conversion charts shows, per major version, what percentage of commits are found in at least one PR and what percentage also link to at least one issue. 

- In a healthy PR-driven workflow we will typically see >90% commits → PR and a lower commits → PR → Issue rate, reflecting that not every change closes an issue. 



<img width="2084" height="582" alt="counts grouped" src="https://github.com/user-attachments/assets/0c09a260-a3df-4c44-98b5-5f3b3c4b97d4" />


The series totals chart shows, per version, total commits vs unique PRs vs unique issues, which makes it easy to spot unusually busy releases or mismatches between code volume and tracked work.

#### Sample commit_pr_issue.json

{
  "repo": "owner/name",
  "base": "v4.2.3",
  "compare": "v4.2.4",
  "counts": {
    "commits": 37,
    "prs": 36,
    "issues": 5,
    "commit_pr": 36,
    "commit_pr_issue": 5
  },
  "source_files": {
    "pairs": "pairs/v4.2/v4.2.3...v4.2.4.compare.json",
    "commits": "commits/v4.2/v4.2.3...v4.2.4.commits.json",
    "pulls": "pulls/v4.2/v4.2.3...v4.2.4.pulls.json",
    "issues": "issues/v4.2/v4.2.3...v4.2.4.issues.json"
  },
  "commit_pr_issue": [
    {
      "commit": [
        {
          "sha": "d7875a…",
          "html_url": "https://github.com/owner/name/commit/d7875a…",
          "api_url": "https://api.github.com/repos/owner/name/commits/d7875a…",
          "author_login": "ClearlyClaire",
          "commit_author_date": "2023-12-19T10:27:37Z",
          "commit_title": "Fix call to inefficient `delete_matched`…"
        }
      ],
      "prs": [
        {
          "number": 28367,
          "html_url": "https://github.com/owner/name/pull/28367",
          "api_url": "https://api.github.com/repos/owner/name/pulls/28367",
          "state": "closed",
          "merged_at": "2023-12-19T10:31:02Z",
          "pr_title": "Fix call to inefficient `delete_matched`…",
          "pr_body": "See #28374"
        }
      ],
      "issues": [
        {
          "number": 28330,
          "html_url": "https://github.com/owner/name/issues/28330",
          "state": "closed",
          "issue_title": "The text about … is cropped …",
          "issue_body": "### Steps to reproduce…"
        }
      ]
    }
  ]
}

### Prompt engineering and Gemini prediction

We use an LLM (Gemini) to predict whether each GitHub issue is UI or Other for other issue’s labels. Human labels fluctuate across projects/releases; getting a fast, consistent second opinion helps triage and prioritize.

#### Prompt:

You are an expert GitHub developer and contributor for a web application. Your job is to analyze a GitHub issue (title and body) and classify it into one of exactly two classes: "UI" or "Other". Follow the class definitions below precisely. Then output a SINGLE JSON object in the required schema. Do not add any extra text, explanations, code fences, or markdown—ONLY the JSON object.

DEFINITIONS (exactly two classes):

1) UI (User Interface)
Description:
- Issues that are purely visual or layout-related—the presentation layer the user directly sees.
Examples include (non-exhaustive):
- Readability / Cognitive Load: poor contrast, illegible fonts, dense text, inconsistent visual hierarchy.
- Aesthetics and Styling: incorrect colors, wrong font sizes, padding/margin mistakes, misaligned elements, excessive visual clutter.
- Layout and Responsiveness: overlapping elements, content bleeding off-screen, broken layouts on mobile/desktop, incorrect positioning.
- Visual State Errors: incorrect hover/pressed/disabled visual states; looks enabled when disabled, etc.
- Missing Visual Elements: expected icon/image/loader is visually missing.

2) Other
Description:
- The default for anything NOT strictly visual/presentation.
- Includes functional/behavioral UX, logic, performance, accessibility (interaction/ARIA/keyboard/screen reader), error handling, system feedback, data issues, or conceptual design.
Examples include (non-exhaustive):
- System Feedback/Function: no confirmation after save; forces memory/recall between steps.
- Logic and Behavior: wrong calculations, data loading errors, broken form submission, violates platform conventions.
- Error Handling: cryptic errors, no recovery, allows destructive actions too easily.
- User Control: missing undo/redo, cannot cancel a task.
- Accessibility (A11y): keyboard navigation failures, screen reader problems, missing ARIA labels (interaction failures are NOT purely visual).

OUTPUT FORMAT (strict):
Return ONLY a single JSON object with this exact schema and keys:
{
  "reason": "Explain why it is UI or Other, referencing specific words/phrases from the issue that match the definitions. Name the relevant heuristic/category when applicable (e.g., Readability, Consistency and Standards, Error Prevention).",
  "class": "UI or Other"
}

ADDITIONAL RULES:
- If any functional/behavioral aspect is involved, classify as "Other" (even if there are also minor visual complaints).
- If the issue is ambiguous or mixes concerns, prefer "Other" unless it is clearly and exclusively visual/presentation.
- Never invent details beyond the issue text.
- Output must be valid JSON (no trailing commas, no markdown, no extra keys).
- Keys must be exactly: "reason", "class".
- "class" must be exactly one of: "UI" or "Other".

ISSUE TO CLASSIFY:

sample_issue_title:
sample_issue_body

<img width="944" height="702" alt="issue_analysis_github_label_counts" src="https://github.com/user-attachments/assets/4e3237bc-ae26-4551-aaa8-ec259b70d688" />


This bar chart shows how maintainers labeled issues for the selected major version (here: v4.2).
•	Other: which means issues in this release were not labeled as UI.
•	None: those issues have text but no GitHub labels. Useful to track because unlabeled issues are hard to route or analyze later.
•	UI: indicating only issues were explicitly tagged as UI.



<img width="967" height="702" alt="issue_analysis_label_source_compare" src="https://github.com/user-attachments/assets/66789eec-9605-4b32-bae9-c8c83b091be0" />


This grouped chart compares maintainer labels (github_label) against Gemini model predictions (gemini_pred_label) for the same v4.2 set.


### Final analysis — model performance

- TP: GitHub=UI and Gemini=UI
- FP: GitHub=Other but Gemini=UI
- TN: GitHub=Other and Gemini=Other
- FN: GitHub=UI but Gemini=Other



<img width="628" height="84" alt="image" src="https://github.com/user-attachments/assets/ca39390f-b2f9-4355-b1d1-71abba1eee56" />



