# Changelog

All notable changes to this project are documented here. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the project uses [Semantic Versioning](https://semver.org/).

## [0.2.1] - 2026-10-11

### Fixed

- Notification images were missing on iOS: the Companion app returns 404 for an image URL with a query string, and every alert used `alert.jpg?v=<timestamp>`. Each alert now saves its own `alert-<YYYYMMDD-HHMMSS>.jpg` and links it without a query string.

### Added

- `shell_command.p1s_ai_cleanup_alerts`, run by the daily 04:30 cleanup, deletes alert frames older than 7 days.

### Docs

- Alert frame location and retention, a troubleshooting row for notifications without an image, and a note that the alert frames are publicly reachable under `/local/obico/`.

## [0.2.0] - 2026-10-11

Changes from the first real failures on a P1S, see [docs/field-reports.md](docs/field-reports.md).

### Added

- Automation "P1S AI - Auto-clear stale alert": an alert nobody reviewed is cleared once the score has stayed below 0.38 for 10 minutes while printing. No cooldown and no notification, so the next warning notifies again.

### Changed

- Frames on layers below `min_layer` (default 3) are skipped with the status `Skipped: first layers`, because first-layer outlines look like strings to the model. The frame counter does not advance, so the warm-up starts after these layers. A non-numeric layer sensor never skips a frame.

### Docs

- New field report with the first two real failures and their timelines.
- README: limitation on late detection of small failures far from the camera.
- Installation guide: `min_layer` tuning row, auto-clear, and pressing a review button after a warning.
- How it works: first-layer skip and auto-clear.

## [0.1.1] - 2026-10-06

### Changed

- CI: `actions/checkout` v4 → v6 and `gitleaks/gitleaks-action` v2 → v3, which run on Node 24 and remove the Node 20 deprecation warnings.
- Docs: the ML backend requires an x86-64 machine; HA Green, HA Yellow and Raspberry Pi cannot run it, not even as an add-on.

## [0.1.0] - 2026-10-06

First public release. Experimental: tested on one Bambu Lab P1S.

### Added

- Home Assistant package `p1s_ai.yaml`: snapshot every 10 s, Obico ML request, Obico-style scoring (EWM, per-print mean, long-term printer baseline), warning and pause levels, cooldown, review scripts.
- Stage-aware analysis: frames are scored only in the `printing` stage, with the chamber light on and settled.
- Chamber light control with a dark-print mode when the AI is disabled.
- Archive of suspicious frames (14 days) and Logbook entries with raw boxes for tuning; ignore zones and a sensitivity slider.
- Dashboard view with built-in cards only.
- Docker Compose stack for the Obico ML backend.
- Installation guide and design notes.

[0.2.1]: ../../releases/tag/v0.2.1
[0.2.0]: ../../releases/tag/v0.2.0
[0.1.1]: ../../releases/tag/v0.1.1
[0.1.0]: ../../releases/tag/v0.1.0
