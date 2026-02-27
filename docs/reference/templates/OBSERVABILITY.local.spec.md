# OpenClaw native and lobster local observability specification

## Purpose

Define a reproducible, JavaScript-first local deployment profile for deep observability across OpenClaw runtime surfaces, using native OpenClaw capabilities and lobster-compatible terminal flow.

## Scope

This specification covers:

1. Fork-first setup workflow
2. Host-level observability pipeline for process, shell, files, commits, and network
3. Native hook instrumentation using `hooks.mappings`
4. User-defined project queue and autonomous 24x7 execution loop
5. JavaScript declaration inventory with file and line metadata
6. Unified timeline for issue-level triage and validation gates

## Requirements

### R1. Fork and workspace bootstrap

- The workflow MUST begin from a GitHub fork of `openclaw/openclaw`.
- The workflow MUST initialize a dedicated observability root directory with:
  - `process`, `shell`, `files`, `git`, `network`, `hooks`, `events`, `manifest`, `instructions`, `projects`, `coverage`.

### R2. Process visibility

- The profile MUST produce both an initial process snapshot and a periodic process poll log.
- Process filters MUST include `openclaw`, `openclaw-gateway`, `node`, and `bun` process names.

### R3. Terminal and shell visibility

- The profile MUST expose a wrapper command (`openclaw-observe`) that captures full TTY transcripts.
- Terminal flow MUST remain lobster-compatible and use native CLI execution paths.

### R4. File operation visibility

- Linux profile MUST support `auditd` as primary mode for read and write visibility.
- Linux profile MUST provide `inotifywait` fallback mode.

### R5. Commit visibility

- The profile MUST define a repository-local `post-commit` hook.
- Hook output MUST include UTC timestamp, commit SHA, and patch summary.

### R6. Network visibility

- The profile MUST include packet capture guidance (`tcpdump`) and periodic socket snapshots (`ss`).
- Captured data MUST be directed into the observability root tree.

### R7. Native hook instrumentation

- The profile MUST document at least one `hooks.mappings` example that normalizes inbound events.
- The profile MUST define a local hook event sink script to persist hook-derived payloads.

### R8. User-defined project execution model

- The profile MUST include a project queue file controlled by user-defined project instructions.
- The profile MUST include a long-running worker loop that can continue 24x7 without interactive prompting.
- The model MUST define two phases:
  - **Implementation and errands phase**: high activity until required work is sufficiently performed.
  - **Maintenance-only phase**: lower activity once feature/spec completion and validation thresholds are met.

### R9. Source declaration inventory

- The profile MUST define a JavaScript declaration manifest generator.
- Generator output MUST include file path, line number, and declaration snippet.
- Output format MUST be TSV for machine processing, with optional markdown rendering.

### R10. Validation and assistant-equivalent gates

- The profile MUST define local validation checks for implementation completion.
- The profile MUST define assistant-task-equivalent validation checkpoints for non-coding errands.
- Coverage and gate outputs MUST be logged under `coverage/` for auditability.

### R11. Correlated timeline and safety controls

- The profile MUST define a timeline builder that merges logs from all observability surfaces.
- Timeline output MUST be sortable and suitable for issue attachment.
- The profile MUST include explicit guidance to avoid committing secrets or private identifiers.
- The profile MUST include least-privilege guidance for packet capture.

## Deliverables

Implementation of this specification requires:

1. `docs/reference/templates/OBSERVABILITY.local.md` with executable setup blocks
2. Coverage of all requirement groups R1 through R11
3. A triage checklist aligned to collected artifact paths and validation gates
