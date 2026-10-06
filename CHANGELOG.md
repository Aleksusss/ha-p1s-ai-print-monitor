# Changelog

All notable changes to this project are documented here. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the project uses [Semantic Versioning](https://semver.org/).

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

[0.1.0]: ../../releases/tag/v0.1.0
