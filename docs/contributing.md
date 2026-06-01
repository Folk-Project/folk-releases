# Contributing

Folk is an open-source project and we welcome contributions from the community — whether it's a bug report, a feature idea, or a code fix.

All issues are tracked in the [folk-releases](https://github.com/Folk-Project/folk-releases/issues) repository.

## Reporting a Bug

Before filing a bug, please check [existing issues](https://github.com/Folk-Project/folk-releases/issues?q=is%3Aissue+label%3Abug) to avoid duplicates.

### Bug report structure

```
Title: <component>: short description of the problem

## Component

Which part of Folk is affected (e.g. `folk-plugin-http`, `folk-core`, `folk-builder`).
Include the file and line number if you know it.

## Problem

What happens? Include steps to reproduce and a minimal code snippet or config.

## Expected behavior

What should happen instead.

## Environment

- Folk extension version (`php -r "echo folk_version();"`)
- PHP version (`php -v`)
- OS (e.g. Ubuntu 24.04, macOS 15, Alpine 3.20)
- Framework and version (e.g. Laravel 11, Symfony 7.2)
```

### Example

```
Title: folk-plugin-http: multipart file upload corrupts binary data

## Component

folk-plugin-http — src/payload.rs

## Problem

Uploading a binary file via multipart/form-data results in corrupted data.
The uploaded JPEG has different bytes than the original.

Steps to reproduce:
1. Create a route that saves an uploaded file
2. `curl -F "file=@photo.jpg" http://localhost:8080/upload`
3. Compare `md5sum` of original and saved file — they differ

## Expected behavior

Uploaded files should be byte-identical to the originals.

## Environment

- Folk 0.2.6
- PHP 8.3.12 ZTS
- Ubuntu 24.04
- Laravel 11
```

### Tips for good bug reports

- **Minimal reproduction** — the smaller the example, the faster the fix
- **One issue per bug** — don't combine multiple problems in one issue
- **Include config** — share your `folk.toml` (redact passwords)
- **Include logs** — run with `RUST_LOG=debug` for verbose output

## Proposing an Idea

Have a feature request or an improvement suggestion? We'd love to hear it.

Use the **idea** label when creating an issue.

### Idea structure

```
Title: short description of the feature

## Problem

What problem does this solve? Why is it needed?

## Proposed solution

How do you imagine this working? Include config examples,
API sketches, or workflow descriptions.

## Alternatives considered

Have you considered other approaches? Why is this one better?

## Use case

Describe a real-world scenario where this feature would help.
```

### Example

```
Title: Hot reload for development

## Problem

Every PHP code change requires a manual server restart,
slowing down the development cycle.

## Proposed solution

Add a `--watch` flag or `[dev] watch = true` config option
that monitors PHP files and automatically restarts workers
when changes are detected.

## Alternatives considered

- `max_jobs = 1` — restarts workers after each request,
  but much slower than targeted reload
- External file watcher (e.g. `entr`) — works but requires
  extra tooling and doesn't integrate with graceful restart

## Use case

During active feature development, I edit controllers and
want to see changes immediately without restarting the server.
```

### Tips for good ideas

- **Focus on the problem** — explain *why* before *how*
- **Be specific** — "better performance" is vague; "reduce memory usage for idle workers" is actionable
- **Consider scope** — small, focused features are more likely to be implemented quickly

## Labels

| Label | Description |
|-------|-------------|
| `bug` | Something isn't working |
| `idea` | Future feature to consider |
| `enhancement` | Improvement to an existing feature |
| `critical` | Critical bug or security issue |
| `important` | Important issue |
| `core` | folk-core / folk-ext |
| `api` | folk-api |
| `plugin-http` | folk-plugin-http |
| `plugin-grpc` | folk-plugin-grpc |
| `plugin-jobs` | folk-plugin-jobs |
| `plugin-metrics` | folk-plugin-metrics |
| `plugin-process` | folk-plugin-process |
| `builder` | folk-builder |
