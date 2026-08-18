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
