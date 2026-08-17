# LocalGPT Agent specifications

## Canonical implementation source

For all new implementation planning and Codex work, use only:

- [`2026-08-17-localgpt-agent-design-canonical.md`](./2026-08-17-localgpt-agent-design-canonical.md)

This file consolidates the current architecture, security/approval model, personal-use ergonomics, Phase 1 tool catalog, secure-remote milestone and post-Core capability roadmap.

## Historical design records

The following files are retained as design history and review context:

- `2026-08-17-localgpt-agent-design.md`
- `2026-08-17-localgpt-agent-approval-authority-amendment.md`
- `2026-08-17-localgpt-agent-personal-use-ergonomics-amendment.md`

They are **superseded for new implementation work** by the canonical file above. Do not implement from a historical file in isolation and do not attempt to resolve conflicting old wording when the canonical design provides the current rule.

## Precedence

```text
Canonical design
  > historical main design
  > historical amendments / review snapshots
```

If a future approved design changes behavior, update the canonical design (or create its explicit successor) and update this index in the same change.
