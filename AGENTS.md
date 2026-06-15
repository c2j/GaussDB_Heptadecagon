# AGENTS.md — GaussDB Heptadecagon

## What This Repo Is

This is an **umbrella/portal repository** for the GaussDB Heptadecagon open-source toolset. It contains only `README.md` and this file — no build artifacts, no source code. The 10 actual tools live in their own GitHub repos under the `c2j` org.

## Repo Structure

- `README.md` — Bilingual (中文/English) overview of all 10 tools, architecture diagrams, quick-start instructions. This IS the project's landing page.

## The 10 Sub-Projects (separate repos)

All repos are at `https://github.com/c2j/{name}`:

| Repo | Language | Build |
|------|----------|-------|
| `ogsql-parser` | Rust | `cargo build --release` |
| `ogexplain-analyzer` | Rust | `cargo build --release` |
| `metamorphosis` | Rust | `cargo build --release` |
| `codeweb` | Rust | `cargo build --release` |
| `grep-excel` | Rust | `cargo build --release` |
| `WDRProbe` | Rust + TypeScript (Tauri) | Tauri build chain |
| `flux-gauss` | Python | Python tooling |
| `SP-Complexity-Evaluator` | Java / Spring Boot | `./mvnw clean package` |
| `rust-opengauss` | Rust | `cargo build -p gaussdb-mcp` |
| `astgrep` | Rust | `cargo build --release` |

## Architecture Dependency

```
ogsql-parser (foundation AST)
  ├─ ogexplain-analyzer (consumes AST)
  ├─ metamorphosis (consumes AST)
  ├─ codeweb (consumes AST)
  └─ astgrep (consumes AST for GaussDB/OpenGauss SQL dialect)

rust-opengauss (native driver + MCP Server)
  └─ ogexplain-analyzer (can use driver for live EXPLAIN)

flux-gauss, grep-excel, WDRProbe, SP-Complexity-Evaluator — independent
```

Changes to `ogsql-parser` AST output can break the four downstream tools. Coordinate carefully.

## Conventions

- **License**: MIT / Apache-2.0 dual (rust-opengauss), MIT (all others)
- **README style**: Bilingual 中文/English, structured tables, architecture ASCII art
- **Primary language for docs**: Chinese with English translation below

## Working on This Repo

- Only documentation changes (`README.md`, `AGENTS.md`). No CI, no build, no tests.
- Commit message style: `docs: <description>`
- Push directly to `main` — no branch/PR workflow for this meta-repo
