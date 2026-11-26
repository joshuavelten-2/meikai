# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

`meikai` is a newly initialized repository. It contains only a placeholder `README.md`. There is no source code, build system, test suite or tooling configuration yet.

Once the project has a language, framework and structure, update this file with:
- Build, lint and test commands, including how to run a single test
- A high-level architecture overview covering what spans multiple files or modules

## Agent skills

### Issue tracker

Issues are tracked in GitHub Issues on `joshuavelten-2/meikai` via the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

Default vocabulary: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `GLOSSARY.md` and `docs/adr/` at the repo root. See `docs/agents/domain.md`.
