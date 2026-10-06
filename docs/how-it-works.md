# How it works

## Pipeline

Every 10 s, while the print status is `running`, the automation "P1S AI - Score frame (every 10 s)" does the following:

1. **Skip unusable frames.** If the chamber light is off, if the light changed less than 20 s ago, or if the current stage is not `printing`, the frame is skipped and the status shows why.
2. **Snapshot.** `camera.snapshot` writes the frame to `/config/www/obico/latest.jpg`.
3. **ML request.** `rest_command.p1s_ai_ml_predict` sends the image URL to the Obico ML container, which downloads the image and returns a list of detections `[label, confidence, [xc, yc, w, h]]`.
4. **Score and smooth** the result (below).
5. **Alert** if needed, then store the state in helpers so it survives restarts.

## Scoring

The ML API returns every box with confidence ≥ 0.08 (`THRESH` in `ml_api/server.py`). Filtering and smoothing are the client's job. This package ports Obico's client-side logic from `backend/lib/prediction.py` with the default parameters (`FD_1ST_GEN_PARAMS`), which are tuned for one frame every 10 s.

| Value | Definition |
| --- | --- |
| `p` | Sum of the confidences of all boxes ≥ `min_conf` whose centre is outside every ignore zone |
| EWM | Exponentially weighted mean of `p`, span 12 frames (about 2 min) |
| Print mean | Rolling mean of `p` over up to 310 frames (about 50 min); reset at the start of every print |
| Baseline | Rolling mean of `p` over up to 7200 frames (about 20 h of printing); kept across prints |

```
score = (EWM - baseline) × sensitivity
```

| Level | Condition |
| --- | --- |
| Warm-up | The first 30 scored frames of a print never alert |
| Warning | `score ≥ 0.38` and (`score > 0.78` or `score > 3.8 × (print mean - baseline)`) |
| Pause | The same condition applied to `score / 1.75` |

The baseline is what keeps false alarms down: a feature of your camera view that the model always flags (a cable, a reflection, a logo) raises the baseline instead of the score. In a simulation, a constant artefact with `p = 0.3` never alerts, while a real failure with `p = 1.6` warns after about 30 s and pauses after about 60 s.

## Alerting and review

- A warning sets `input_boolean.p1s_ai_alert_active`, saves the analysed frame to `alert.jpg` and sends "P1S alert". While the alert is active, further warnings are suppressed.
- A pause presses the printer's pause button, waits up to 30 s for the printer to leave `running`, and sends "P1S paused", or "PAUSE FAILED" if it did not stop.
- The review buttons clear the alert. Resume and Acknowledge False Alarm start a cooldown (15 min by default) during which no new alerts or pauses fire. The cooldown is stored in an `input_datetime`, so it survives restarts.
- Resuming from the printer or Bambu Handy after an alert is treated as a review.

## Print lifecycle

"P1S AI - Print lifecycle" resets the per-print state (frame counter, EWM, print mean) when a new print starts. A resume or a reconnect after Wi-Fi or HA downtime is not treated as a new print. The long-term baseline is only reset with `script.p1s_ai_reset_baseline`.

## Chamber light

"P1S - Chamber light control" keeps the light on during prints while the AI is enabled, and switches it off 5 min after the print only if it switched it on itself (`input_boolean.p1s_ai_light_owned`). A light switched off from Home Assistant is respected until the next print; one switched off by the firmware, the printer screen or Bambu Handy is turned back on while the AI needs it. With the AI disabled, `input_boolean.p1s_auto_light_off_on_print` keeps a dark-print mode.

## Implementation notes

- Smoothed values are rendered with fixed-point formatting (`'%.6f' | format(...)`). Home Assistant keeps template results such as `8e-05` as strings, which would break the arithmetic when the averages approach zero.
- The automation runs in `single` mode with `max_exceeded: silent`, so a slow ML response skips a tick instead of queueing frames.
- The ML request uses `rest_command` with `response_variable`, so errors are visible in the status text instead of being read as "no detections".
