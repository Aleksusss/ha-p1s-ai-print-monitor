# Contributing

Thanks for your interest. This is a small, experimental project, so please keep changes focused.

## Reporting problems

- Use the issue templates. For false alarms, attach the Logbook entry named "P1S AI" and the archived frame.
- Remove tokens, IP addresses, URLs and your printer serial before posting.

## Pull requests

- One change per pull request, with a short description of what and why.
- Keep everything in English: code comments, log messages and notification texts.
- Do not change the Obico scoring constants without data from real prints.
- YAML must pass `yamllint -c .yamllint .` (the CI runs the same check).
- Add a line to `CHANGELOG.md` under an `Unreleased` heading.
