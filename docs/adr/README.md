# Architecture Decision Records

Each significant, hard-to-reverse decision gets a short record here, named
`NNNN-short-title.md` (e.g. `0001-go-backend.md`).

Template:

```markdown
# NNNN. Title

- Status: proposed | accepted | superseded by NNNN
- Date: YYYY-MM-DD

## Context
What forces are at play.

## Decision
What we chose.

## Consequences
What becomes easier or harder as a result.
```

Candidate first ADRs (decisions already made in `01-architecture.md`):

- Go backend, React + TypeScript frontend
- Split into in-cluster Helm chart + local single-binary app
- No new inbound ports; reach the gateway through the Kubernetes API proxy
- Other tools are headless sensors only; all UI is our own
