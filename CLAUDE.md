# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

會考試題連結抓取器 — a zero-dependency, single-file client-side tool that scrapes Taiwan's 心測中心 (rcpet) 歷屆試題 pages and extracts the Google Drive file IDs of every exam PDF into a JSON list.

The entire application is `index.html` (markup + CSS + JS inline). There is no build system, no package manager, no test suite, no linter, and no backend. Adding any of those is a structural change — ask first.

## Running it

```sh
python3 -m http.server 8000    # then open http://localhost:8000/
```

Opening `index.html` via `file://` also works, but serving over HTTP better matches the deployed environment.

Verification is manual, in a browser with devtools open: click 開始讀取 and watch the `#log` pane and the OMR grid. The tool depends on live third-party CORS proxies and a live upstream site, so a failing run is often an upstream/proxy outage rather than a code defect — check whether a different 中繼站 from the dropdown succeeds before treating it as a bug.

## Architecture

A four-stage pipeline, all in the `<script>` block:

1. **Fetch through a CORS proxy** — `grab(url, mode)` walks the `PROXIES` array (allorigins, codetabs, corsproxy, jina reader) with a 9s `AbortController` timeout each, in `auto` mode falling through to the next proxy on HTTP error, timeout, or a suspiciously short (<300 char) body. Only the last error surfaces. The browser cannot fetch rcpet directly, so every network read goes through here.
2. **Discover year pages** — `findYearPages()` parses the start page with `DOMParser` and collects `<a href>` plus `<option value>` candidates, then regex-scrapes the raw HTML for `*examination*.html` filenames as a fallback for JS-driven navigation. The click handler then optionally filters out 基測 pages (keeping only 3-digit years matching `1[0-2]\d`), sorts newest-year-first, and caps at 40 pages.
3. **Extract Drive links** — `extract()` parses each page and keeps anchors pointing at drive/docs.google whose href yields an ID via `driveId()` (handles `/file/d/`, `?id=`, `/document/d/`, `/drive/folders/`; folder links are then discarded). Each row's year and subject are resolved by widening scope: the anchor's own text → its surrounding row context → the page-level year. If DOM extraction finds nothing, a markdown-link regex pass handles the plain-text output of the jina reader proxy.
4. **Render** — dedupe by `fileId`, group by year, emit JSON into `#out` for copy/download.

### Classification heuristics

`SUBJECTS` is an **ordered** regex→label table and `subjectOf()` returns the first match, so ordering is load-bearing: 聽力/聽測 and 閱讀 must precede the generic 英語|英文 rule, or English listening papers get mislabeled. `yearOf()` accepts a 3-digit ROC year only within 90–130 — widen that range rather than removing it if newer years appear.

### Progress UI state machine

Each year page owns one `.bub` cell whose `data-s` attribute drives the CSS in the `.bub[data-s=...]` rules. Four states: absent (queued, empty circle) → `scan` (in flight, hatched) → `done` (rows found, filled) or `fail` (zero rows or a thrown error, red cross). Set in the scan loop of the `#go` handler; `fail` also increments the 讀不到 tally, and the first two failures dump that page's real hrefs into the log for diagnosis. Any new status must be added in both places — the CSS block and the loop.

## Conventions

- All user-facing text is Traditional Chinese, written for non-technical users: plain wording, no jargon (e.g. 中繼站 for proxy, 劃記 for the progress cells).
- The visual design is a deliberate OMR answer-sheet / 准考證 metaphor; the palette and fonts live in the `:root` custom properties at the top of the `<style>` block. Reuse those variables instead of hardcoding colors.
- Keep everything in the one file — no external scripts or stylesheets beyond the Google Fonts link.
