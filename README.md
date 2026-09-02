# AI security ideas (private backlog)

This repo is a weekday log of implementation candidates: papers, protocol notes, and tools worth building later.

It is **not** the implementation repo. When I pick one, that work goes into its own private repo.

## How this works

- Weekdays around 8:25am ET, Gitcoder adds a dated file under `ideas/` with three candidates.
- Each note is a bit more than a title: what the paper/tech/tool actually is, how it works, and why it would matter in a real product.
- Status lives in `INDEX.md`: `queued` until I ask to implement, then `building` / `shipped` with a link to that private repo.
- Duplicates should not be re-logged. If a day has nothing new, skip the file.

## Layout

```
INDEX.md              running list
ideas/YYYY-MM-DD.md   that day's three candidates
```

## Status values

| Status | Meaning |
| --- | --- |
| queued | logged, waiting for me to pick |
| skip | interesting, not worth building |
| building | I asked for an implementation; private repo in progress |
| shipped | private implementation repo exists |
