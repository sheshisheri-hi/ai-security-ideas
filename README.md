# AI security ideas (private backlog)

Private weekday log of papers, protocol notes, and tools. I pick from here later. Each pick becomes its own private implementation repo.

## How this works

- One markdown page per month: `ideas/YYYY-MM.md`
- Every weekday around 8:25am ET, Gitcoder appends three new candidates to that month's page
- Each entry says what the paper/tech/tool is, how it works, and why it would matter in a real product
- `INDEX.md` is the running list. Status stays `queued` until I pick one to implement
- Duplicates are not re-logged. If a day has nothing new, skip the append
- Ad-hoc research (not a daily candidate) goes under `research/`

## Layout

```
INDEX.md              running list across months
ideas/YYYY-MM.md      that month's page (daily entries)
research/             one-off evals (not implementation picks)
```

## Ad-hoc research

- [Legacy test generation vs Agentic-QE](research/legacy-test-generation-agentic-qe.md) (2026-09-02)
- [AQE + GitHub Copilot how-to (from source)](research/aqe-github-copilot-howto.md) (2026-09-02)
- [Legacy test-gen papers and GitHub repos](research/legacy-testgen-papers-and-repos.md) (2026-09-02)

## Status values

| Status | Meaning |
| --- | --- |
| queued | logged, waiting for me to pick |
| skip | interesting, not worth building |
| building | I asked for an implementation; private repo in progress |
| shipped | private implementation repo exists |
