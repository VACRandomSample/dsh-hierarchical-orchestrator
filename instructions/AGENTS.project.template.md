# Project AGENTS.md template (Hierarchical Orchestrator)

Copy this file to `AGENTS.md` in the root of a project repository and edit the
marked sections. The harness loads it automatically (project instruction chain)
for every agent working in that repository, in addition to the global
`~/.dsh/AGENTS.md` rules. Never overwrite an existing project `AGENTS.md` —
merge this content into it instead.

```markdown
# Project Instructions — <project name>

## Orchestration scopes (fill before delegating)

- Write-owned paths and their single owners are assigned per task by the lead.
  Default split:
  - Backend: src/server/**, src/api/**
  - Frontend: src/client/**, src/ui/**
  - Tests: tests/**, fixtures/**
  - Read-only roles (Explorer, Code Reviewer, Security Auditor) never edit.
- Forbidden for every agent without explicit user approval: deploy scripts,
  production configuration, secrets files (.env, credentials), CI release jobs.

## Verification commands (fill with real commands)

- Unit tests: <command>
- Lint: <command>
- Build: <command>

## Acceptance

- A change is accepted only when: listed checks pass with real output,
  an independent review (ho-code-reviewer) approved the diff, and changed
  files match the assigned allowed_paths exactly.

## House rules

- <project-specific rules>
```
