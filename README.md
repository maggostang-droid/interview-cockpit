# HR Interview Cockpit

A single-file, client-side tool for running structured hiring interviews:
job-ad/CV intake, an importable question pool (xlsx), a scheduling calendar,
a live interview cockpit with timer/phase tracking and 4-point
behavioral-anchor ratings, and a KPI/radar-chart summary.

No backend, no build step — everything (including the "AI copilot" question
suggestions, which call `api.anthropic.com` directly from the browser using a
key you paste in yourself and that is stored only in your browser) runs
client-side. Question data, candidate data, and saved interviews stay in
your browser's local storage or a local folder you pick — nothing is sent
anywhere except the optional direct Anthropic API call.

Originally built as a private project while going through an application
process with Festo (no employment relationship) — this is a sanitized,
rebranded version with only synthetic example content (fictional job
posting, fictional example candidate, self-authored generic question bank).
No real candidate data, job postings, or third-party competency frameworks
are included.

## Try it

Open `index.html` in a browser, or click "Beispiel-Auswertung ansehen" on
the start screen for a filled-in example.

## Stack

Vanilla JS/HTML/CSS. SheetJS (xlsx import), Chart.js (radar chart), pdf.js
(CV text extraction) via CDN.
