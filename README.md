# Review Bench

Probe-by-probe review sweeps for pull requests, incident postmortems, design docs/RFCs and one-pagers.

**Live:** https://vundyalaavinash.github.io/review-bench/

Pick what you're reviewing and a depth (`core` or `full`), then work through one probe at a time. Every probe is cleared with evidence, logged as a finding, or skipped. You get at most three blockers. At the end you name the risk you're accepting, and the app builds a memo you can paste into the PR.

The History tab ranks which probes actually catch things for you, and lists the "cold" ones that have never produced a finding.

## Files

- `index.html` — the whole app. No build step and no dependencies. Open it directly or serve it statically.
- `review-sweep.devin-playbook.md` — the same sweep written as an agent playbook (`!review-sweep`). It runs the probes against a real checkout and ends with a JSON record.

## Agent sweeps → History

Paste the playbook's JSON record (or the whole memo) into **History → Import a sweep** and it is logged next to your manual sweeps.

## Storage

Reviews are saved in your browser's `localStorage`. They don't sync between browsers or devices, and clearing site data deletes them. When the page runs as a Claude artifact, it uses the artifact's shared database instead.

## License

MIT
