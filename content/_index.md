---
title: yze
---

**yze** is the gomatic analyzer family: a suite of [`go/analysis`](https://pkg.go.dev/golang.org/x/tools/go/analysis) analyzers that encode the gomatic Go standards (sentinel error constants, named parameter types, no `goto`, …), built on a shared framework so each analyzer is a small, uniform repository. The [`stickler`](https://github.com/gomatic/stickler) runner orchestrates this suite alongside `golangci-lint`; see [docs.stickler](https://github.com/gomatic/docs.stickler) for the runner.

## Architecture

```text
go-yze            framework library (diagnostic schema, registration, fix engine, checker driver)
   ▲
yze-<group>-<name>   one analyzer per repo (e.g. yze-go-errconst)
   ▲
yze               aggregator multichecker — runs the suite, emits stickler-json
```

## Components

- [**go-yze**](https://github.com/gomatic/go-yze) — the framework library every analyzer is built on: the normalized `Diagnostic`/`Report` schema (stickler-json), `Registration` with the `group`/`category` taxonomy, the `ToDiagnostic` conversion, the shared `ApplyFixes` engine, and the `checker`-based `Run` driver. A pure library (carries the `library.go` marker).
- **`yze-<group>-<name>`** — one analyzer per repository, each exporting a standard `*analysis.Analyzer` plus a `Registration`, with `analysistest` fixtures and a standalone `singlechecker` binary. The current suite:
  - [**yze-go-errconst**](https://github.com/gomatic/yze-go-errconst) — forbid `errors.New` and non-wrapping `fmt.Errorf` (category `errors`).
  - [**yze-go-gotostmt**](https://github.com/gomatic/yze-go-gotostmt) — deny the `goto` statement (category `patterns`).
  - [**yze-go-namedtypes**](https://github.com/gomatic/yze-go-namedtypes) — flag bare primitive parameter types (category `types`).
- [**yze**](https://github.com/gomatic/yze) — the aggregator: a `go/analysis` multichecker that fans in every analyzer, filters by `group`/`category`, and emits the normalized report (`stickler-json` by default, or `text`), with `--fix`.

## Two organizing axes

- **group** — a single, stable segment in the repo/module name (`yze-<group>-<name>`); the default axis is the language/target (`go`). Few groups, by design.
- **category** — a many-to-many semantic tag held in metadata (`errors`, `patterns`, `types`, …), used for filtering and these docs. Categories are decoupled from groups and are deliberately kept out of repo names.

## Fixes

Analyzers attach native `analysis.SuggestedFix` edits when a safe, deterministic fix exists; the shared `go-yze` `ApplyFixes` engine applies them (`yze --fix`), and `gopls` surfaces them as editor quick-fixes for free.
