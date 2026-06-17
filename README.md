# PV Limiter Control via Home Assistant & Node-RED


PV Limiter Control is a Home Assistant and Node-Red solution that gives control on (max 2) inverter ouput. 
Based on the core electricity price and grid power usage it controls the inverter output. 

[![Home Assistant](https://img.shields.io/badge/Home%20Assistant-Integration-3DDC84?logo=home-assistant&logoColor=#03A9F4)](https://www.home-assistant.io/) 
[![Node-RED](https://img.shields.io/badge/Node--RED-Flow-FF4A00?logo=node-red&logoColor=)](https://nodered.org/) 

# PV Limiter — Flow Documentation (v1.1)

> Version history / change log lives in the tab's **Info** panel (double-click the tab name). This node is the detailed, canonical how-it-works reference.

This flow throttles one or two PV inverters based on the 
- **current electricity price**, 
- **live grid (P1) power**
- **battery state-of-charge (SoC)**. 

The goal: export nothing (or charge the battery) when power is cheap, run flat-out when 
power is expensive, and in the normal-price band trim the limiter on battery SoC first
and the grid deadband second the rest of the time.

## Big picture

**Trigger:** every change of `sensor.gespot_current_price_be` (and once ~5 s
after deploy) runs the decision chain. A separate 5 s timer drives the P1
balance loop. **Master gate:** nothing acts unless `input_boolean.pv_limiter`
is ON.

**PV2 auto-detect:** `binary_sensor.pv2_config_present` is cached into flow
var `pv2_present`. Every PV2 write is gated on it and the P1 calc only needs
an PV2 reading when present, so the flow runs on **PV1 alone** when the
second inverter package is not installed.

Three price branches:
- **Low** (price < threshold_low) → both PVs to **0 %** + battery strategy
- **Charge**, loop stopped.
- **High** (price ≥ threshold_high) → both PVs to **100 %**, loop stopped.
- **Normal** (in between) → arm the **P1 balance loop**.

Watchdog: 3 consecutive bad sensor reads → safe **50 %** fallback.

---

## Groups

**00 Control (enable / disable)** — Manual ▶/⏹ inject buttons and the matching
HA `input_boolean` state listeners. A *Validate & debounce* function (2 s
debounce) rejects bad payloads and toggles `input_boolean.pv_limiter`; the
listener then mirrors the state into flow var `pv_enabled` (disabling also
clears `pv_loop_active`). Every toggle is logged. an *On start:
enable* inject fires once after deploy and turns `input_boolean.pv_limiter`
**ON**, so the flow comes up **enabled by default**.

**01 Price Trigger** — Fires the decision chain. *Price changed* listens to
`sensor.gespot_current_price_be`; *Deploy: re-evaluate* injects once 5 s after
deploy and *Read current price* feeds the same entry point so the flow settles
to the right state on restart. A genuine change to any **08 Config Monitoring**
helper (thresholds, deadband, step size) also routes through *Read current
price*, so editing config while the limiter is enabled restarts the decision
chain.

**02 Decision** — The gatekeeper chain: *Flow enabled?* halts unless
`input_boolean.pv_limiter` is ON → *Get threshold low* / *Get threshold high*
read the price bands → *Threshold sanity* parses/validates the numbers (NaN or
low ≥ high routes to the Watchdog) → *Price branch* switch routes to Low /
High / Normal. A healthy pass resets `pv_invalid_count`.

**03 Low Price → PV 0 % + Charge** — *Prepare 0 % + stop loop* clears
`pv_loop_active`, sets the desired setpoint to 0/0, then calls set both PV
sliders to 0 % and switches `input_select.house_battery_strategy` to **Charge**.
Logged as `cheap_zero`.

**04 High Price → PV 100 % + Self-consumption** — *Prepare 100 % + stop loop*
stops the loop, sets the desired setpoint to 100/100, and drives both PV
sliders to 100 % so the inverters run at full output. Logged as
`expensive_full`.

**05 Normal Price → Activate P1 Loop** — *Activate P1 loop* sets
`pv_loop_active = true`, clears the last-branch marker and nulls `pv_desired`
so the loop re-seeds from the live HA sliders on its next tick. Logged as
`normal_price`.

**06 Watchdog (bad-tick safety fallback)** — A shared bad-tick counter
(`pv_invalid_count`). Bad reads from the Decision chain or the P1 calc land
here; on the **3rd** consecutive strike it forces both PVs to **50 %**, stops
the loop, syncs the setpoint, and resets the counter (a fresh 3 strikes are
needed to fire again). Logged as `watchdog_fallback`.

**07 P1 Balance Loop — every 5 s in the normal-price band** — *Every 5 s* timer
→ *Gate* (runs only when `pv_enabled` and `pv_loop_active` are both true) → reads PV1, PV2, battery SoC and `sensor.p1_meter_power` →
*Calculate P1 adjustment*. The calc trims the PV output up/down by the
configured **step**, checking battery **SoC** first and the grid **deadband**
second:
- A **race guard** bails if the loop was stopped mid-tick (so it can't
  overwrite a 0/100/50 % write).
- It steps from a flow-maintained **desired setpoint** (`pv_desired`), not by
  reading the slider back each tick (avoids actuator-feedback stalls); a >5 %
  divergence or a stale (>30 s) setpoint re-syncs from the live HA value.
- **SoC < 95 % (charge):** drive the limiter straight to **100 %** (full
  output) to charge the battery, **regardless of the deadband**. The limiter is
  never trimmed down below 95 % SoC.
- **SoC > 95 %:** trim the limiter **down** (`reduce`), but only when grid
  power is outside the deadband; while |P1| ≤ deadband it holds (`balanced`,
  logged only on transition).
- **SoC = 95 %:** hold (`balanced`).
- Missing/invalid sensor values → `p1_invalid_input` → Watchdog.

**08 Config Monitoring** — Listeners on `input_number.pv_price_threshold_low`,
`…_high`, `…_step_size`, and `…_p1_deadband`. *Format config change* caches
each value into flow context (`cfg_*`) so the P1 loop reads config without
polling HA, and logs only genuine changes (the deploy-time emit and no-op
old == new changes are suppressed). Logged as `config_change`. a genuine
change also feeds *01 Price Trigger → Read current price*, so editing the
thresholds, deadband or step size **restarts** the flow (re-runs the price
evaluation) — routed through *Flow enabled?*, so nothing happens while the
limiter is off.

**09 PV2 Detection** — *PV2 config present changed* listens to
`binary_sensor.pv2_config_present` (with `outputInitially` so it also fires on
deploy); *Set pv2_present flag* caches the boolean into flow var
`pv2_present` and shows a node status ('PV2 detected' / 'PV1 only'). The
five *PV2 present?* gates and the P1 calc read this flag so PV2 is only
touched when its modbus package exists.

**99 Update UI: PV Limiter log** — Central logger. A *link in* collects events
from every branch; *Build log entry* formats a timestamped entry (icon, level,
group, node, payload), keeps a rolling **10-entry** buffer, and *Set
sensor.pv_limiter_log* writes it to HA — publishing both a JSON `entries`
attribute and a ready-to-render **Markdown table** (`markdown` attribute) for
the dashboard card. An *On deploy: seed log* inject feeds *Build startup event*
so a `startup` entry is written on every deploy, and a tab-wide *Catch all
errors* routes runtime errors into the same logger via *Filter transient HA
drops* — a guard that collapses repeated HA WebSocket dropouts
(`NoConnectionError` / `Connection lost`) into a **single log row per 60 s
window** (showing a live suppressed-count on its node status) while letting
genuine errors pass through untouched, so a brief reconnect no longer spams the
log.

---

## Required HA helpers
All helpers are defined in the
[`pv_limiter_control.yaml`](https://github.com/coderubbere/HA-PV-Limiter/blob/main/home%20assistant/packages/pv_limiter_control.yaml)
package:
1. `input_boolean.pv_limiter`
2. `input_number.pv_price_threshold_low`
3. `input_number.pv_price_threshold_high`
4. `input_number.pv_step_size` (1–10, def 2)
5. `input_number.pv_p1_deadband` (0–500, def 100)

Plus an auto-created `binary_sensor.pv2_config_present` (from the `command_line`
sensor in `pv_limiter_control.yaml`) that drives PV2 auto-detection.

## Key flow context vars
1. `pv_enabled`
2. `pv_loop_active`
3. `pv_desired` {pv1,pv2,ts}
4. `pv_p1_last_branch`
5. `pv_invalid_count`
6. `cfg_step`
7. `cfg_deadband`
8. `cfg_threshold_low`
9. `cfg_threshold_high`
10. `pv_log_buffer`
11. `pv2_present` (true when the PV2 package is detected)
12. `ha_conn_err_window` {start,count} — rate-limit state for *Filter transient HA drops*

## Dashboad yaml
1. [`dashboard.yaml`](https://github.com/coderubbere/PV-Limiter/blob/main/home%20assistant/dashboard.yaml)

## PV Modbus TCP yaml
1. ['modbus_tcp_pv1.yaml'](https://github.com/coderubbere/PV-Limiter/blob/main/home%20assistant/packages/modbus_tcp_pv1.yaml)
2. ['modbus_tcp_pv2.yaml'](https://github.com/coderubbere/PV-Limiter/blob/main/home%20assistant/packages/modbus_tcp_pv2.yaml)
