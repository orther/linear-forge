# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] - 2026-04-03

### Added

- Plugin manifest (`.claude-plugin/plugin.json`) with Linear MCP server and `userConfig` schema
- **Skills**:
  - `/linear-forge:work` -- Full issue-to-merge lifecycle for a single issue
  - `/linear-forge:status` -- Team-wide status overview with priorities and blockers
  - `/linear-forge:project-status` -- Deep project view with dependency graph and actionable breakdown
  - `/linear-forge:work-parallel` -- Concurrent multi-issue work in isolated worktrees
  - `/linear-forge:project-work` -- Full project automation with dependency-ordered batching
  - `/linear-forge:setup` -- Interactive configuration wizard with tooling auto-detection
- **Agents**:
  - `pr-reviewer` -- Automated code review with lint, type checks, and auto-fix for trivial issues
  - `test-runner` -- Test plan execution, full suite runner, and minor test fix automation
- Per-project configuration via `userConfig` with 10 configurable options
- Auto-detection for test and lint commands across JS/TS, Go, Rust, Python, Ruby, PHP, and more
- Bundled Linear MCP server for OAuth-based Linear API access

[0.1.0]: https://github.com/orther/linear-forge/releases/tag/v0.1.0
