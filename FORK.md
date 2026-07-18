# sqly Fork Documentation (`FORK.md`)

This repository is a fork of [nao1215/sqly](https://github.com/nao1215/sqly). It maintains local optimizations, code quality upgrades, linter compliance, and bug fixes.

---

## 1. Summary of Differences by Category

The differences between this fork and the upstream repository are grouped into the following categories:

### Category A: Atomic File Writing & Data Loss Prevention (Issue #753)
* **Description**: Prevents `sqly` from truncating or destroying destination files when an export or save operation fails mid-write.
* **Justification**: The original implementation opened output files directly with `O_TRUNC` before validating the table records. If validation failed (e.g., due to invalid characters in LTSV formatting), the target file was left at `0` bytes. With `--save --force`, this deleted the user's source data. We introduced a temp-file-write-then-rename staging pattern.

### Category B: Error-Propagation Corrections
* **Description**: Ensures that output rendering errors are checked and returned rather than silently ignored.
* **Justification**: Output formats like Markdown and LTSV previously discarded write errors and returned `nil` (success) to their callers. If the output stream became full or broken, this resulted in silent truncation. We refactored these renderers to return errors.

### Category C: Gosec Security Scan Exclusions
* **Description**: Updated gosec comment annotations to the `#nosec` standard recognized by the `gosec` CLI tool.
* **Justification**: The codebase previously used `//nolint:gosec` comments, which are honored by `golangci-lint` but ignored by the direct `gosec` CLI scanner used in automated audits. These were updated to `// #nosec` with detailed inline justifications explaining why file opening, path creation, and concatenation for SQL DDL are benign.

### Category D: Benign Linter Warning Suppressions (`errcheck`)
* **Description**: Added `//nolint:errcheck` and `_, _ =` ignores to diagnostic terminal prints.
* **Justification**: Discards unchecked write errors when printing to `Stdout`/`Stderr` for interactive output, CLI help dialogs, version requests, or exit-path error logging where errors cannot be resolved or recovered.

---

## 2. File-by-File Details and Justification

| File / Component | Category | Change Type | Detailed Justification |
| :--- | :--- | :--- | :--- |
| **[interactor/export.go](./interactor/export.go)** | A | Refactored `withCompressedWriter` | Writes format outputs (CSV, TSV, LTSV, MD, JSON, NDJSON) to a temp file in the target directory and renames it atomically on success. Prevents truncating target files on failure. |
| **[interactor/errorpath_test.go](./interactor/errorpath_test.go)** | A | Added `TestDumpTable_PreservesExistingFileOnFailure` | Regression test verifying that target files are preserved if formatting validation fails during export. |
| **[domain/model/table.go](./domain/model/table.go)** | B | Refactored `printMarkdownTable`, `printLTSV`, and `Print` | Refactored Markdown and LTSV printers to return write errors instead of ignoring them, ensuring correct status codes. |
| **[infrastructure/filesql/adapter.go](./infrastructure/filesql/adapter.go)** | C | Updated line 258 to `// #nosec G202` | Concatenation for SQLite `VACUUM INTO` is safe because SQLite lacks a parameterized option for it, and the path is single-quote escaped. |
| **[infrastructure/filesql/parquet.go](./infrastructure/filesql/parquet.go)** | C | Updated lines 140, 146 to `// #nosec` | Bypasses path traversal warnings since filesql temp read and user-specified export destination writes are expected behaviors. |
| **[shell/import.go](./shell/import.go)** | C | Updated lines 951, 963 to `// #nosec` | Bypasses path traversal warnings for reading user-chosen imports and staging them under secure temp directories. |
| **[shell/shell.go](./shell/shell.go)** | C / D | Updated line 644 to `// #nosec G304`, line 1090 to `//nolint:errcheck` | Bypasses temp file creation warnings and silences stdout print warnings on query results. |
| **[shell/cache.go](./shell/cache.go)** | C / D | Updated lines 303, 331 to `// #nosec G304`, lines 171, 175, 210, 215 to `//nolint:errcheck` | Bypasses cache file reads/hashes and silences terminal logs to stderr on cache hits/failures. |
| **[shell/batch.go](./shell/batch.go)** | C / D | Updated line 703 to `// #nosec G304`, line 83 to `//nolint:errcheck` | Bypasses --sql-file read warnings and silences batch failure messages to stderr. |
| **[main.go](./main.go)** | D | Updated lines 30, 36 to `//nolint:errcheck` | Silences error output warnings to stderr during fatal startup crashes before calling `os.Exit(1)`. |
| **[config/argument.go](./config/argument.go)** | D | Updated line 514 to `//nolint:errcheck` | Silences stdout print warning on `sqly --version` requests immediately before exiting. |
| **[shell/clear.go](./shell/clear.go)** | D | Updated line 29 to `//nolint:errcheck` | Silences terminal clearing ANSI escape sequences print warnings. |
| **[shell/compare.go](./shell/compare.go)** | D | Updated lines 137, 144 to `//nolint:errcheck` | Silences comparison stdout outputs where SIGPIPE is handled at the OS level. |
| **[shell/help.go](./shell/help.go)** | D | Updated lines 64, 66, 70 to `//nolint:errcheck` | Silences interactive help listings print warnings. |
| **[shell/profile.go](./shell/profile.go)** | D | Updated lines 102, 109 to `//nolint:errcheck` | Silences profile stats stdout print warnings where SIGPIPE is handled by OS. |
| **[shell/inspect.go](./shell/inspect.go)** | D | Updated line 120 to `//nolint:errcheck` | Silences inspect JSON output stdout print warnings. |
| **[shell/ls.go](./shell/ls.go)** | D | Updated lines 46, 64 to `//nolint:errcheck` | Silences in-shell file listings stdout print warnings. |
