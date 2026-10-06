# Installation guide

This guide adds camera-based print failure detection to a Bambu Lab P1S that is already integrated in Home Assistant through ha-bambulab. Setup takes about an hour.

- [Placeholders](#placeholders)
- [Step 1: Deploy the ML backend](#step-1-deploy-the-ml-backend)
- [Step 2: Prepare Home Assistant](#step-2-prepare-home-assistant)
- [Step 3: Install the package](#step-3-install-the-package)
- [Step 4: Verify the pipeline without printing](#step-4-verify-the-pipeline-without-printing)
- [Step 5: Add the dashboard view](#step-5-add-the-dashboard-view)
- [Step 6: First prints, then go live](#step-6-first-prints-then-go-live)
- [Daily operation](#daily-operation)
- [Tuning](#tuning)
- [Troubleshooting](#troubleshooting)
- [Uninstall](#uninstall)

## Placeholders

The files use four placeholders. Replace all of them before the first restart; nothing else needs editing.

| Placeholder | Replace with | Where to find it |
| --- | --- | --- |
| `p1s_yourserial` | Your ha-bambulab entity prefix, for example `p1s_01p00a123456789` | Developer Tools → States, search `print_status` |
| `HA_HOST` | IP address of Home Assistant, reachable from the ML host | Settings → System → Network |
| `ML_HOST` | IP address of the Docker host running the ML container | Your Docker host |
| `notify.mobile_app_your_phone` | Your Companion app notify service | Developer Tools → Actions, type `notify.mobile_app` |

In `ml-backend/compose.yaml`, also set `TZ` to your timezone (it only affects container logs).

Check that these printer entities exist with your prefix. If Home Assistant was set up in a language other than English, ha-bambulab may have created translated entity IDs; then replace each one by hand instead of a single find-and-replace.

| Entity | Used for |
| --- | --- |
| `sensor.<prefix>_print_status` | Print state: `running`, `pause`, `finish`, `failed` |
| `sensor.<prefix>_current_stage` | Skipping calibration and prep stages; must read `printing` during a real print |
| `sensor.<prefix>_current_layer` | Logged with archived frames |
| `camera.<prefix>_camera` | Snapshots for the model |
| `light.<prefix>_chamber_light` | Chamber light control |
| `button.<prefix>_pause_printing`, `_resume_printing`, `_stop_printing` | Pause, resume and stop |

## Step 1: Deploy the ML backend

The Obico ML container does the image analysis. It runs on an x86-64 Docker host on the LAN and needs no GPU. The files are in [`ml-backend/`](../ml-backend/).

1. On the Docker host, generate a token: `openssl rand -hex 20`.
2. Copy `compose.yaml` and `.env.example` to a stack folder, then `cp .env.example .env` and put your token in `ML_API_TOKEN`.
3. Start it with `docker compose up -d`, or deploy it from Dockge or Portainer.
4. Check that it answers: `curl -i http://ML_HOST:3333/p/`. Any HTTP reply, usually 401 because no token was sent, means it is up. "Connection refused" means it is not running or port 3333 is blocked.

The ML host must be an x86-64 (amd64) machine. Obico ML does not run on ARM, so Home Assistant Green, Home Assistant Yellow and Raspberry Pi cannot host it, not even as a Home Assistant add-on. Keep port 3333 reachable from the LAN only and never commit your `.env`. If the token ever leaks, generate a new one and update both `.env` and the HA secret from Step 2.

## Step 2: Prepare Home Assistant

Four one-time changes, all before installing the package. Commands are for the Terminal & SSH add-on, where the config folder is `/homeassistant`; inside Home Assistant the same folder is `/config`.

**2.1 Enable packages.** In `configuration.yaml`, add this, or merge it into an existing `homeassistant:` block. Then run `mkdir -p /homeassistant/packages`.

```yaml
homeassistant:
  packages: !include_dir_named packages
```

**2.2 Create the folders.** `www` must exist before HA starts, otherwise `/local/` URLs are not served until the next restart. `obico_history` stays outside `www`, so archived frames are not publicly reachable.

```bash
mkdir -p /homeassistant/www/obico /homeassistant/obico_history
```

**2.3 Add the secret** to `/homeassistant/secrets.yaml`, using the token from Step 1:

```yaml
p1s_ml_auth_header: "Bearer <your token>"
```

**2.4 Remove conflicts.** Disable any existing automation that switches the P1S chamber light around prints; the package takes over the light. If you already have an automation that notifies when a print pauses, add this condition to it, so an AI pause is not announced twice:

```yaml
- condition: template
  value_template: >-
    {{ not (is_state('input_boolean.p1s_ai_alert_active', 'on')
            and as_timestamp(now())
                - (state_attr('input_datetime.p1s_last_ai_alert', 'timestamp') | float(0)) < 60) }}
```

Security note: files in `/config/www` are served at `/local/` without login. If HA is reachable from the internet, block `/local/obico/` on your reverse proxy or tunnel.

## Step 3: Install the package

After this step every switch is off (AI, Auto Pause, Notify Only), so prints are not affected until Step 6.

**3.1 Copy the package.** Copy [`homeassistant/packages/p1s_ai.yaml`](../homeassistant/packages/p1s_ai.yaml) to `/homeassistant/packages/p1s_ai.yaml`.

**3.2 Replace the placeholders.** Then confirm none are left; this command must print nothing:

```bash
grep -nE "yourserial|HA_HOST|ML_HOST|your_phone" /homeassistant/packages/p1s_ai.yaml
```

**3.3 Optional: cut database writes.** These entities update every 10 s. Add them to the existing `recorder:` block in `configuration.yaml`, or create one there; a second `recorder:` block in the package would conflict.

```yaml
recorder:
  exclude:
    entities:
      - input_number.p1s_ai_frame_num
      - input_number.p1s_ai_lifetime_frames
      - input_datetime.p1s_last_ai_check
      - input_text.p1s_last_ai_result_raw
```

**3.4 Check and restart.** Run Developer Tools → YAML → Check configuration; a typo in the secret name is the usual error. Then Settings → System → Restart → Restart Home Assistant. A full restart is required, because `shell_command` and new helpers are not picked up by a reload.

**3.5 Apply defaults.** Run Developer Tools → Actions → `script.p1s_ai_apply_defaults`. Afterwards `input_number.p1s_ai_sensitivity` should be 1.0 and `input_number.p1s_ai_cooldown_minutes` 15.

## Step 4: Verify the pipeline without printing

Three calls in Developer Tools → Actions check every link of the chain before a real print. If a print is already running, you can skip this step: the status in Step 6 shows the same result.

| # | Action | Expected result |
| --- | --- | --- |
| 1 | `camera.snapshot`, entity `camera.<prefix>_camera`, filename `/config/www/obico/latest.jpg` | `http://HA_HOST:8123/local/obico/latest.jpg` opens the current camera frame in a browser |
| 2 | `rest_command.p1s_ai_ml_predict` | Response shows `status: 200` and `content.detections`; an empty list is normal |
| 3 | `shell_command.p1s_ai_archive_frame` in YAML mode with `data: {frame_id: test}` | `/homeassistant/obico_history/test.jpg` appears; delete it afterwards |

If call 2 fails:

- **401**: the token in `secrets.yaml` does not match `.env`.
- **400**: the ML container cannot download the image from HA. Check `HA_HOST`, the firewall, and that HA answers plain HTTP on port 8123. If HA serves HTTPS only, change the image URL in `rest_command` to your `https://` address; it needs a valid certificate.
- **No response**: the ML container is down, or `ML_HOST` or the port is wrong.

## Step 5: Add the dashboard view

The view in [`homeassistant/dashboard/p1s_ai_view.yaml`](../homeassistant/dashboard/p1s_ai_view.yaml) uses only built-in cards. Replace `p1s_yourserial` in the camera and light entities, and add it after the restart, when all its entities exist.

1. Open the dashboard you want to use and click the pencil icon to edit it.
2. Click + to add a view, then in the view dialog open ⋮ → Edit in YAML.
3. Replace everything with the view YAML and save.
4. Check that no tile says "Entity not available". If one does, a placeholder was missed or the package did not load.

The tiles to watch are Status and Score. The AI signal graph shows the last 6 hours of the frame score, the smoothed value, the printer baseline and the final score.

## Step 6: First prints, then go live

Run the first one or two prints in Notify Only mode, where the AI warns but never pauses. Turn on AI and Notify Only; leave Auto Pause off.

| Check | Expected during the print |
| --- | --- |
| Chamber light | On for the whole print. If it was off before the start, it turns on at `prepare` and off 5 min after the print ends |
| Status sequence | `Skipped: <stage>` during heating and calibration, then `Warming up (1/30)` for about 5 min, then `OK` |
| Current stage | `sensor.<prefix>_current_stage` reads `printing` during actual printing; otherwise the status stays on `Skipped` |
| Last result | `input_text.p1s_last_ai_result_raw` mostly shows small `p` values, often `n=0/0` |
| AI signal graph | Score stays well below 0.38 |

Light tests can run on the same print, in this order:

1. Switch the light off from the HA dashboard. It stays off and the status reads `Skipped: chamber light off`. Switch it on: `Skipped: light settling` for about 20 s, then `OK`.
2. Switch it off from the printer screen or Bambu Handy. HA turns it back on within about 5 s.
3. Turn AI off, then turn on Light Off on Print. The light goes off within about 5 s. Turn the toggle off again; the light stays off.
4. Turn AI on. The light turns on at once.

Finish with the light toggle off and AI plus Notify Only on.

**Controlled failure test (recommended).** On a cheap test print in Notify Only mode, pause after a few layers, open the door, knock the part off the plate and resume. The printer now prints into the air and produces real spaghetti. The head is parked while paused, so this is safe. A "P1S alert" should arrive within about a minute.

After one or two clean prints, go live: AI on, Auto Pause on, Notify Only off. Leave Alert off; it switches on by itself when the AI raises an alert.

Home Assistant can be restarted mid-print without harm. Afterwards, press Reset Alert State once, because the per-print reset only runs when a new print starts.

## Daily operation

The AI has two levels: a warning keeps the print running, a pause needs a signal 1.75× stronger.

| Level | When | What happens |
| --- | --- | --- |
| Warning | Score reaches 0.38 and is clearly above this print's normal level | "P1S alert" notification with the analysed frame; the print continues |
| Pause | Signal 1.75× stronger, Auto Pause on, Notify Only off | Print paused, time-sensitive "P1S paused" notification with the frame; status `Paused - review required` |

| Button | Use it when | Effect |
| --- | --- | --- |
| Resume After Review | The print is fine after a pause | Clears the alert and resumes; no new alerts during the cooldown (15 min) |
| Stop After Review | The print is ruined | Clears the alert and stops the print |
| Acknowledge False Alarm | A warning or pause was wrong | Counts a false alarm, clears the alert, starts the cooldown. It does not resume, so press Resume After Review too if paused |
| Reset Alert State | You want a clean slate without counting anything | Clears the alert and the cooldown |
| Reset Baseline | Camera moved, different build plate, new lighting inside the chamber | The long-term printer baseline relearns from zero |

The Alert tile is an indicator, not a setting. It switches on when the AI raises an alert, and off with the buttons above or when a print starts or ends. Never switch it on by hand: while it is on, new warnings are suppressed.

Resuming from the printer screen or Bambu Handy after an alert also counts as reviewed and starts the cooldown. Do not reset the baseline for day and night: it averages about 20 hours of printing and absorbs those changes.

| AI | Light Off on Print | Chamber light |
| --- | --- | --- |
| On | Any | On for the whole print; off 5 min after the print, only if the automation switched it on |
| Off | On | Off 5 s after printing starts or resumes |
| Off | Off | Not touched |

Switching the light off from HA is respected until the next print. Switching it off from the printer screen or Handy is undone while the AI needs the light.

## Tuning

Change settings only after a real false alarm, and decide from the recorded data. Every frame whose strongest box has confidence 0.2 or higher is copied to `/homeassistant/obico_history/<YYYYMMDD-HHMMSS>.jpg` and kept for 14 days. A Logbook entry named "P1S AI" lists its boxes as `[conf, xc, yc, w, h]`: confidence, then box centre and size in snapshot pixels.

| Setting | Where | Default | Effect |
| --- | --- | --- | --- |
| Sensitivity | Dashboard slider | 1.0 | Lower means fewer alerts, higher means earlier alerts; move in steps of 0.05–0.1 |
| Cooldown | Dashboard slider | 15 min | Quiet time after Resume or Acknowledge |
| `ignore_zones` | Variables of the score automation | `[]` | Ignores boxes whose centre is inside a rectangle |
| `min_conf` | Same | 0.08 | Ignores boxes below this confidence |
| `archive_min_conf` | Same | 0.2 | Archive threshold |
| `light_settle_s` | Same | 20 s | Frames skipped after the light switches |

Leave the Obico constants (`ewm_alpha`, `win_short`, `win_long`, `t_low`, `t_high`, `safe_frames`, `short_mult`, `escalate`) unchanged; they are tuned for the 10 s interval. After editing variables, reload automations in Developer Tools → YAML.

Zone example: false alarms keep coming from a box like `[0.31, 412, 280, 60, 44]`. It spans x 382–442 and y 258–302; with a 20 px margin the zone is:

```yaml
ignore_zones: [[362, 238, 462, 322]]
```

A zone also hides real failures in that area, so keep zones small and only on spots the part never occupies, such as the wiper, a cable or the plate edge.

## Troubleshooting

The Status tile (`input_text.p1s_ai_status`) names most problems directly.

| Status or symptom | Meaning | Fix |
| --- | --- | --- |
| `Idle` during a print | AI is off, or the text is left over from before | Turn AI on; the status changes within 10 s |
| `Disabled` | AI is off | Turn AI on |
| `Skipped: chamber light off` | Light is off, so frames are not analysed | Turn the light on; look for another automation that controls it |
| `Skipped: light settling` | First 20 s after a light change | Wait |
| `Skipped: <stage>` | Heating, levelling, filament change and similar prep stages | Normal during prep; if it stays while actually printing, check what `current_stage` reads |
| `Warming up (n/30)` | First 30 scored frames of a print, about 5 min | Normal |
| `ML error (401)` | Token mismatch | Make `p1s_ml_auth_header` match `.env` |
| `ML error (400)` | The ML container cannot download the snapshot | Check `HA_HOST`, HTTP vs HTTPS and the firewall; `www` must have existed when HA started |
| `ML error (no response)` | ML container unreachable | Check the container, `ML_HOST` and port 3333 |
| Status never changes | The score automation is not running | Automations → "P1S AI - Score frame (every 10 s)" → ⋮ → Traces shows the failed condition or error |
| Frequent false warnings | Something in view looks like a failure to the model | Collect Logbook entries and archived frames, then add a zone or lower sensitivity (see [Tuning](#tuning)) |
| Light keeps switching back | Another automation controls the chamber light | Disable it (Step 2.4) |
| Two notifications per AI pause | Your own pause notification fires as well | Add the condition from Step 2.4 |
| "Entity not available" on the dashboard | A placeholder was missed or the package did not load | Re-run the check from Step 3.2, then Check configuration |

When you open an issue, remove IP addresses, tokens and your printer serial from logs and screenshots first.

## Uninstall

Turning the AI toggle off stops all analysis, alerts and pauses at once. To remove the setup completely:

1. Delete `/homeassistant/packages/p1s_ai.yaml` and restart Home Assistant.
2. In Settings → Devices & services → Entities, remove leftover `p1s_ai` entities marked "Not provided". HA often cleans these up by itself on restart.
3. Delete the P1S AI dashboard view.
4. Remove `p1s_ml_auth_header` from `secrets.yaml`, and the recorder lines if you added them.
5. Delete the folders: `rm -r /homeassistant/www/obico /homeassistant/obico_history`.
6. On the Docker host, run `docker compose down` in the ML stack folder.
