---
title: yze
---

**yze** is the gomatic analyzer family: a suite of [`go/analysis`](https://pkg.go.dev/golang.org/x/tools/go/analysis) analyzers that encode the gomatic Go standards (sentinel error constants, value receivers, named parameter types, the three-tier CLI layout, …), built on a shared framework so each analyzer is a small, uniform repository. The [`stickler`](https://github.com/gomatic/stickler) runner orchestrates this suite alongside `golangci-lint`; see [docs.stickler](https://github.com/gomatic/docs.stickler) for the runner.

## Architecture

```text
go-yze            framework library (diagnostic schema, registration, fix engine, checker driver)
   ▲
yze-<name>        one analyzer per repo (e.g. yze-errconst)
   ▲
yze               aggregator multichecker — runs the suite, emits stickler-json
```

Analyzers are flat-named `yze-<name>` (there is no language segment — the suite is all Go), and each emits the stable rule id `yze/<name>`. Analyzers carry **category** tags (`errors`, `naming`, `types`, `immutability`, `structure`, `patterns`, `testing`, …) as metadata, used for `--category` filtering and these docs.

## Components

- [**go-yze**](https://github.com/gomatic/go-yze) — the framework: the normalized `Diagnostic`/`Report` schema (stickler-json), `Registration`, `ToDiagnostic`, the shared `ApplyFixes` engine, the `checker`-based `Run` driver, and `ApplyConfig` (per-analyzer settings → analyzer flags). A pure library.
- [**yze**](https://github.com/gomatic/yze) — the aggregator: a `go/analysis` multichecker over every analyzer; `--category` filtering, stickler-json/text output, `--fix`, and `--config` (per-analyzer settings).
- **`yze-<name>`** — one analyzer per repository (standard `*analysis.Analyzer` + `Registration`, `analysistest` fixtures, standalone binary). The current suite:

| Rule | What it enforces | Category |
| --- | --- | --- |
| [`yze/errconst`](https://github.com/gomatic/yze-errconst) | no `errors.New` / `fmt.Errorf` — wrap via `errs.Const.With` (`exempt` defaults to `go-error` itself) | errors |
| [`yze/errlast`](https://github.com/gomatic/yze-errlast) | `error` is the last return value | errors |
| [`yze/gotostmt`](https://github.com/gomatic/yze-gotostmt) | no `goto` | patterns |
| [`yze/ctxfirst`](https://github.com/gomatic/yze-ctxfirst) | `context.Context` is the first parameter | patterns |
| [`yze/namedtypes`](https://github.com/gomatic/yze-namedtypes) | named domain types for parameters | types |
| [`yze/anonstruct`](https://github.com/gomatic/yze-anonstruct) | no anonymous struct types | types / structure |
| [`yze/emptyiface`](https://github.com/gomatic/yze-emptyiface) | `any`, not `interface{}` (autofix) | modern-go |
| [`yze/boolname`](https://github.com/gomatic/yze-boolname) | boolean predicate/flag naming | naming |
| [`yze/ptrrecv`](https://github.com/gomatic/yze-ptrrecv) | value receivers (no pointer receivers without a no-copy field) | immutability |
| [`yze/ptrparam`](https://github.com/gomatic/yze-ptrparam) | value parameters (no pointer params except idiomatic stdlib) | immutability |
| [`yze/stdlog`](https://github.com/gomatic/yze-stdlog) | `log/slog`, not the `log` package | data |
| [`yze/pkgstd`](https://github.com/gomatic/yze-pkgstd) | three-tier command-package standards (per-package) | structure |
| [`yze/layout`](https://github.com/gomatic/yze-layout) | three-tier command↔domain correspondence (cross-package) | structure |
| [`yze/testfile`](https://github.com/gomatic/yze-testfile) | unit-test files 1:1 with source | testing |

## Configuration

`ptrrecv` and `ptrparam` accept a configurable `-allow` list of additional types. Settings flow through `yze --config` (or [`stickler`](https://github.com/gomatic/stickler)'s layered config), keyed by analyzer name.

## Fixes

Analyzers attach native `analysis.SuggestedFix` edits when a safe, deterministic fix exists; the shared `go-yze` `ApplyFixes` engine applies them (`yze --fix`), and `gopls` surfaces them as editor quick-fixes for free.
