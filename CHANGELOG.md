# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

This project is a fork of [`finnboeger/samsung-scantopc`](https://github.com/finnboeger/samsung-scantopc),
originally derived from [`kleest/samsung-scantopc`](https://github.com/kleest/samsung-scantopc)
and [`suld-scantopc`](https://www.bchemnet.com/suldr/) under the GNU General Public License v3.0.

## [0.7.0] - 2026-11-11

### Added
- **Automatic ADF & Flatbed Feeder Detection**:
  - Scanning preset `doc_source` can now be set to `"Auto"` (or left unset).
  - Probes the feeder using `ADF Simplex`; if documents are loaded, scans from the feeder, otherwise falls back to the `Flatbed` glass.
- **SANE Session Handle Reset**:
  - Automatically re-initializes and cleans the SANE session handle after feeder probing, resolving a timeout/freeze issue on flatbed scans.
- **Graceful Offline Reconnection & Silent Retries**:
  - Catches network timeouts, connection refused, and unreachable printer status when the printer is powered off or in deep sleep.
  - Replaces repetitive 15-line stack trace logging every 10 seconds with a concise single-line status notification.
- **Automated GitHub Container Registry (GHCR) Publishing**:
  - Added GitHub Actions workflow (`.github/workflows/ci.yml`) to build multi-tag `linux/amd64` Docker images (`ghcr.io/misterx2000/samsung-scantopc`).
- **Modern Python Tooling & Packaging**:
  - Added `pyproject.toml` with PEP 621 metadata, dependencies, and `ruff` linting/formatting configuration.
  - Added support for `uv` package manager and automated linting in CI.

### Fixed
- Fixed `TypeError: SANE_STRING requires a string` when `doc_source` is `None`.
- Fixed Dockerfile warning regarding undefined `$PYTHONPATH`.
- Fixed type comparisons in multipart form encoding (`src/http_comms.py`).

### Changed
- Formatted and linted all codebase files with `ruff`.
