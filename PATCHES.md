# Patches on top of `anerdins/node-red-contrib-nibepi#1.2.1`

Branch `arva`. The `nibepi` dependency points at the matching fork,
`github:martinarva/nibepi#arva`.

Note that `1.2.1` upstream is a **branch, not a tag**.

## indoor.js  (`NIBEPI_PATCHED_INDOOR`)

**A startup race silently disabled indoor control.** `startUp()` calls
`await server.nibe.reqData(...)` before the Nibe core is up. That throws
`Error: Core is not started`, `.catch(console.log)` swallows it and returns
`undefined`, and the very next line —

```js
if (nibe_enabled === undefined || nibe_enabled.data === undefined) { arr = []; }
```

— empties the sensor array and **permanently disables the plugin for that
session**, even though `config.indoor.enable_s1` is `true`. Nothing ever
retried.

`startUp` now retries after 15 seconds when the config says the system should be
enabled. Confirmed by the log line `Indoor Ss1: core was not ready, retrying in
15s` appearing exactly once and not repeating.

## config_node.js

This file carries **custom price-control logic**: `fetchHAEntity()` around line
31 and `runPrice()` around line 1804 (`runPriceOld()` is the superseded earlier
version). Background: nibepi's author used to run a cloud service called
"priceai" to supply electricity prices, and when that shut down the logic was
reimplemented to read the price from Home Assistant's REST API instead.

**The token is no longer in the source.** It used to be hardcoded in clear
text; it now reads:

```js
const HA_TOKEN = process.env.HA_TOKEN || "";
```

Supply it to the container as an environment variable — for example a compose
`env_file:` pointing at a file kept out of version control. Without it the price
logic simply never queries Home Assistant.

> Forks of public repositories cannot be made private, so anything committed
> here is public the moment it is pushed. Scan for secrets before every push.

**This logic is currently inactive, and that is deliberate.** Two independent
breakages: `/etc/nibepi/config.json` has `price.enable: false` with
`source: "priceai"`, and the hardcoded `HA_ENTITY = "sensor.nordpool_import"` no
longer exists. Reviving it needs both fixed — and before that, a decision about
whether price control belongs here at all, rather than on the Home Assistant
side where the Nordpool sensors and EMHASS already live.

## Plugin registry  (`NIBEPI_PATCHED_REGISTRY`, `NIBEPI_PATCHED_SNAPSHOT`)

Files: `config_node.js`, `rmu.js`, `indoor.js`, `weather.js`.

**Whether RMU, indoor and forecast control survived a restart was a coin toss.**
Each plugin registers once, at startup, through `initiatePlugin()`, which decides
whether climate system S1 exists from a single live read of `supply_s1`. That read
happened while nibepi fired one read per configured register at a pump that
answers one register at a time (see `nibepi` patch 9), so it regularly timed out —
and nothing ever retried. `getList` stayed empty, `updateData()` looped over
nothing, and every S1 plugin was silently dead until the next restart. Observed:
24 hours without a single RMU write after one restart, against 505 writes and 24
forecast runs in the 24 hours before it.

**The RMU node stayed green through all of it.** `checkRMU` resolved outside its
`if (regN !== -1)`, so it reported success whenever register 10020 was readable,
registered or not. It now resolves only when S1 really is in the registry.

- **Self-healing.** `checkPluginRegistry()` runs at the end of every `updateData()`
  cycle. If RMU, indoor or forecast control is enabled but missing from the
  registry, it emits `pluginReinit`, and `rmu.js`, `indoor.js` and `weather.js`
  register again. At most once per 5 minutes, and not in the first 5 minutes after
  start, so the normal startup registration gets to finish first. It is a dedicated
  event rather than `ready`, which would also restart `runDiagnostic()`.
- **Plugins run in isolation** (`runPluginSafe()`). `runIndoor` is synchronous and
  throws on a missing register (`inside_set.data` after one read timed out); that
  used to abort the whole cycle, so `runRMU` never ran.
- **Values could be filed under the wrong name.** `updateData()` re-read
  `item.registers[i]` inside its callbacks, i.e. after a read that can take
  seconds. If plugins re-registered in the meantime the list had shifted, and a
  value got another entry's name: register 40025 (exhaust air, always −3276.8 on a
  pump without that sensor) came out labelled as the room sensor, and `runRMU`
  wrote −3276.8 to the pump as the room temperature. The loop now walks a snapshot
  and binds each entry before the `await`, and `runRMU` refuses anything outside
  −30…60 °C.

Measured afterwards: the watchdog repaired two real startup failures on its own,
RMU writes resumed with plausible values, and nothing was mislabelled when a
re-registration coincided with two concurrent cycles.

## Node ≥ 15 and unhandled rejections

**Node-RED kept restarting itself.** This code was written for Node 8–10, where an
unhandled promise rejection was only a warning. Node ≥ 15 terminates the process
instead, and Node-RED installs no `unhandledRejection` handler. Twice in three days
a read of 47402 timed out, `runIndoor` threw, the rejection escaped from the cron
callback and Node-RED exited — and every exit re-ran the coin toss above.

`updateData()` no longer lets plugin errors escape, but the codebase has many more
fire-and-forget promises. Run it with

```
NODE_OPTIONS=--unhandled-rejections=warn
```

Checked on Node 24.20: without the flag a bare `Promise.reject()` exits with code 1;
with it the process keeps running.

## config_node.js — forecast control  (`NIBEPI_PATCHED_SNOW1G`)

**SMHI retired the `pmp3g/v2` point forecast.** Every URL returns 404, the API root
included (checked 2026-09-26), so forecast control could not work at all. It failed
quietly, too: the error branch just keeps the curve offset at 0.

It now uses `snow1g/v1`. Rather than rewrite the forecast logic, `snow1gToPmp3g()`
converts the new response into the old shape as soon as it arrives:

| pmp3g | snow1g |
|---|---|
| `validTime` | `time` |
| `t` | `air_temperature` |
| `ws` / `wd` / `gust` | `wind_speed` / `wind_from_direction` / `wind_speed_of_gust` |
| `Wsymb2` | `symbol_code` (SMHI documents it as `Wsymb2`, same 27 codes) |

snow1g marks a missing value as `9999`, which the adapter rejects, and so is any
response that is too short or not JSON. Each case returns `undefined`, which takes
the existing "provider not responding" path. The whole response handler is also
wrapped in `try`/`catch`: it runs in an `https` callback, where a throw is an
uncaught exception that restarts Node-RED.

Checked for 59.40 N, 24.82 E, which is outside Sweden: 57 hourly steps (forecast
control needs 49), and the computed curve offset matches the formula worked by hand.

## Dashboard flow (not part of this repository)

The stock dashboard button "Starta om Node-RED" runs `sudo service nodered restart`,
which silently does nothing in a container. Under Docker use `kill -TERM 1` instead:
the image's entrypoint traps SIGTERM, stops Node-RED cleanly and exits, and a
`restart: unless-stopped` policy starts the container again about two seconds later.
