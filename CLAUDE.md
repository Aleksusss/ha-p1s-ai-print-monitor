# Project rules

Home Assistant package and docs for camera-based print failure detection on a Bambu Lab P1S with a self-hosted Obico ML backend.

## Language

- Everything in the repository is in English: code, comments, docs, commit messages, notification and status texts.

## Git

- Commit with the repo-local identity (`git config --local user.name` / `user.email`, a GitHub noreply address). Never use a personal e-mail address.
- Ask the owner before every `git push` and before creating a release or tag.
- Never commit secrets, tokens, IP addresses or printer serials. Placeholders stay as they are: `p1s_yourserial`, `HA_HOST`, `ML_HOST`, `notify.mobile_app_your_phone`.
- Images in `docs/images/` must not show serials, IP addresses or other personal data.

## Scoring

- Never change the Obico constants (`ewm_alpha`, `win_short`, `win_long`, `t_low`, `t_high`, `safe_frames`, `short_mult`, `escalate`) without data from real prints. They are Obico's `FD_1ST_GEN_PARAMS`, tuned for one frame every 10 s.
- Local filters and new rules must be backed by a real case; record it in `docs/field-reports.md`, and mark what is verified and what is assumed.

## Checks and releases

- Run `yamllint -c .yamllint .` before every commit; there is no live Home Assistant to test against.
- Keep `CHANGELOG.md` (Keep a Changelog) and the docs in sync with behaviour changes.
- Create a GitHub release only after CI on `main` is green.
