# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.0.0] — 2026-05-02

Initial release.

### Added

- `SKILL.md` dispatcher with scoring rubric, rule index, review workflow, and structured output format.
- Twelve self-contained reference files under `references/`, one per rule:
  - **01 — Transactional management:** `@Transactional` and `runTransaction` discipline; no competing transactions per request.
  - **02 — Repository definition:** all `createQueryBuilder` and raw SQL stay out of services.
  - **03 — Service definition:** abstract `<Domain>Service` plus `<Domain>ServiceImpl`, registered with `provide`/`useClass`.
  - **04 — Event emission:** `entity.action` naming describing the committed transaction, never the subscriber's job.
  - **05 — Decorator design pattern:** `BaseDecorator` extending the abstract service, registered via `useFactory` with a `Symbol` token.
  - **06 — Folder structure:** per-domain module layout with `<name>.<type>.ts` filenames.
  - **07 — Strategy design pattern:** abstract strategy plus named-variant implementations; global vs domain-specific registration.
  - **08 — Adapter design pattern:** vendor-neutral abstract plus named provider implementations with typed request/response shapes.
  - **09 — Error handling:** `ErrorTypes` casting at the service layer; `HttpException` preservation; `InternalServerError` only for genuinely unhandled errors.
  - **10 — Logging:** request-scoped logging at start, end, around I/O boundaries, with correct severity levels.
  - **11 — Circular dependencies:** zero tolerance, `forwardRef` not accepted; resolution patterns documented.
  - **12 — Dependency vs Composition:** when to inject a service vs pass an entity; canonical `Rule`/`Condition` example.
- `README.md` with install instructions for Claude Code, Cursor, and other Agent Skills-compatible hosts, including manual install fallback and `gh` CLI install instructions.
- MIT license.

### Critical rules

- Rule **#1** (Transactional management) and rule **#11** (Circular dependencies) block merges regardless of net score.

### Compatibility

- Pure markdown skill — no shell injection, no forked-subagent execution. Verified portable across Claude Code and Cursor; expected to work on any Agent Skills–compatible host.
- The `allowed-tools` frontmatter field is intentionally omitted so the agent's permission settings stay authoritative.

[Unreleased]: https://github.com/<owner>/backend-pr-review/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/<owner>/backend-pr-review/releases/tag/v1.0.0
