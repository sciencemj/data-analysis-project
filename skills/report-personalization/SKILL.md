---
name: report-personalization
description: >-
  Set up or update YOUR personal identity preset for the data-analysis-project
  report — 리포트 개인화. WHEN to use: before (or after) running a data analysis,
  to make the generated HTML report carry *your* identity instead of the template
  author's — your portfolio back-link, author/display name, GitHub owner, and
  footer. The data-analysis-project report step (Stage 9) reads this preset and
  swaps the bundled defaults for yours. Triggers on "리포트 개인화",
  "내 포트폴리오/푸터 설정", "리포트에 내 정보 넣기", "리포트 프리셋 만들어줘",
  "set up my report preset", "personalize the data report".
  Do NOT use this for the analysis itself — that is the **data-analysis-project**
  skill. This skill ONLY writes the preset file; it does not source data, build a
  notebook, or generate a report.
---

# Report Personalization · 리포트 개인화

A small **setup utility**: it records *your* identity once into a preset file so the
**data-analysis-project** report stage personalizes the HTML report with your name,
portfolio link, GitHub owner, and footer — instead of shipping the template author's
defaults. **It does not analyze anything** and has no workflow gates.

## The preset file

- **Global (default — set once, reused everywhere):** `~/.claude/data-analysis-report-preset.json`
- **Project-local (optional override for one project):** `./report-preset.json` — if present,
  it takes precedence over the global file for that project.
- **JSON shape:**

```json
{
  "author": "<display name shown in the footer>",
  "portfolio_url": "<your portfolio URL, or null to omit the back-link>",
  "github_owner": "<your github username/org, used for repo links>",
  "footer": {
    "mode": "embed",
    "embed_url": "<your shared footer.js URL, only when mode=embed; else null>"
  }
}
```

### How each field maps to the report template

The bundled `assets/report-template.html` keeps the author's own defaults — a
`← 포트폴리오` back-link to `sciencemj.github.io`, a `github.com/sciencemj/…` repo link,
and a shared `embed/footer.js` `<script>`. Stage 9 replaces them from your preset:

| Field | Template target |
|---|---|
| `portfolio_url` | the nav **back-link** href (`← 포트폴리오 / Portfolio`). **`null` → the back-link is omitted** entirely. |
| `github_owner` | the report's **GitHub repo link owner** (`github.com/<owner>/<repo>`); the **repo name = the project's own repo**, not part of the preset. |
| `footer.mode` | `embed` → include **your** `footer.js` via `<script src="<embed_url>">`; `self-contained` → bake a small footer (**author + year + repo**) into the HTML; `none` → no footer. |
| `footer.embed_url` | the `src` of the footer `<script>` — **only** when `mode = embed` (else `null`). |
| `author` | the **name shown in the self-contained footer** (used when `mode = self-contained`). |

## Flow

Keep it to a few quick questions — confirm each value as you go.

1. **Check for an existing preset.** Look for `./report-preset.json` first, then
   `~/.claude/data-analysis-report-preset.json`. If one exists, **show its current values**
   and ask whether to **keep** them or **update** (개인화 유지/수정). If keeping, stop here.
2. **Collect the fields** (prefer `AskUserQuestion` or direct dialogue):
   - **author / display name** — e.g. "민준 / Minjun".
   - **portfolio URL** — your site, or **"none"** → `portfolio_url: null` (back-link omitted).
   - **GitHub owner** — your username/org for repo links (e.g. `minjun`).
   - **footer choice** — one of: your **shared embed** (ask for the `footer.js` URL) /
     **self-contained** (author + year + repo, no external script) / **none**.
3. **Write the JSON.** Default to the **global** path
   (`~/.claude/data-analysis-report-preset.json`); create `~/.claude/` if it doesn't exist.
   Offer a **project-local** write (`./report-preset.json`) if the user wants a different
   identity for this one project (예: 회사 프로젝트는 회사 계정).
4. **Confirm.** Print the **file path** written and a **one-line summary** of what was saved
   (author · portfolio · github owner · footer mode).

## How it's used

This skill **only writes the preset** — it runs no analysis. To produce a report, run the
**data-analysis-project** skill; its **Stage 9 (리포트 작성)** reads this preset and
personalizes the report (back-link / footer / repo owner / author name) instead of the
template author's defaults. If no preset exists and you are not the template author, Stage 9
will pause and either run **report-personalization** or ask you for these details inline —
it never ships someone else's identity.
