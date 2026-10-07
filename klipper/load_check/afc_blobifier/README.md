# Example: check every AFC purge (AFC + Blobifier + bdwidth)

[purge_check.cfg](purge_check.cfg) wraps the Blobifier purge that AFC runs
after each filament load and checks it two ways:

1. **bdwidth motion:** mm the bdwidth saw moving past it, against mm
   extruded. Catches a grinding extruder, a tangle or a snapped filament.
2. **bd_pressure:** the longest continuous nozzle pressure during the
   purge. Catches filament that never reached the melt zone.

Either check is skipped when its sensor is missing, so it also works with
only a bdwidth (stock bd_pressure firmware) or only a bd_pressure with the
load-check firmware. Setting up the load check itself (firmware, module,
commands) is described in [../README.md](../README.md).

| File | Does |
|---|---|
| `purge_check.cfg` | `BLOBIFIER_CHECKED` macro and its settings |
| `bdwidth_status.py` | Klipper module that exposes bdwidth's motion counter and width to macros (bdwidth's own status only has `filament_detected` and `enabled`) |

## Install

1. Run `~/bd_pressure/klipper/install.sh`. It links `bdwidth_status.py` and
   `bdpressure_check.py` into `~/klipper/klippy/extras/`.
2. Copy `purge_check.cfg` to your config folder and add
   `[include purge_check.cfg]` to `printer.cfg`.
3. Check the two sensor names at the top of `purge_check.cfg`:
   `[bdwidth_status] sensor:` (name after `bdwidth` in `[bdwidth fila_width_0]`)
   and `[bdpressure_check] sensor:` (name after `bdpressure` in
   `[bdpressure bd_pa]`). Remove `[bdpressure_check]` if you have no
   load-check firmware.
4. In `[AFC]` (`AFC/AFC.cfg`) change `poop_cmd: BLOBIFIER` to
   `poop_cmd: BLOBIFIER_CHECKED` and restart Klipper.

Each purge then prints one line:

```
PURGE_CHECK lane1: motion 99.8/149.2 mm ratio 0.67, width 1.77; pressure longest 17883 ms, total 65535 ms, peak 148 -> OK
PURGE_CHECK ?: motion 0.0/149.2 mm ratio 0.00, width 0.00; pressure longest 31 ms, total 53 ms, peak 12 -> FAIL: filament not moving past bdwidth, no sustained nozzle pressure
```

## bdwidth outside a print

If your print start/end macros switch bdwidth on and off
(`SET_BDWIDTH NAME=... COMMAND=ENABLE_ALL` / `COMMAND=DISABLE`), it reads
nothing between prints. `BLOBIFIER_CHECKED` therefore switches it on for
the check when no print is running (or paused), waits `bdwidth_warmup_ms`
for the first readings (10-14 s were seen) and switches it off again
afterwards. During a print it leaves bdwidth alone.

With no filament loaded, `ENABLE_ALL` also arms bdwidth's own width
runout check, which then pauses the printer (`filament width is out of
range: 0.000mm`). That pause also moves the toolhead, which registers as
nozzle pressure (1589 ms in one test instead of the usual ~30 ms).

## Settings and rollout

The variables of `_PURGE_CHECK_CFG`:

| Variable | Default | Meaning |
|---|---|---|
| `mode` | `"report"` | `report` only prints, `enforce` pauses on a failure, `off` runs the plain purge |
| `min_ratio` | 0.4 | fail below this sensed/extruded ratio |
| `pressure_min_ms` | 5000 | fail below this longest continuous pressure |
| `min_extrude` | 20 | skip both checks for shorter purges (mm) |
| `bdwidth_warmup_ms` | 15000 | wait after switching bdwidth on outside a print |

1. Run in `report` mode for a few days and note the values of good purges.
2. Set `min_ratio` and `pressure_min_ms` well below the lowest good values.
3. Switch to `enforce`.

Reference values from the test printer (150 mm Blobifier purge):

| | ratio | longest pressure |
|---|---|---|
| good load, lane1 | 0.63-0.67 | 17-18 s |
| good load, lane4 (full `CHANGE_TOOL`) | 1.26 | 18 s |
| no filament | 0.00 | 31 ms, 1589 ms with a runout pause during the purge |

The ratio depends on the filament: the bdwidth's measuring wheel grips or
slips differently on different spools, so one `motion_linear_coefficient`
does not fit them all. That does not matter for the check, only the gap
between good and empty does; collect values per lane before raising
`min_ratio`.

The `width` in the result line is the last single reading. The bdwidth
sometimes reports about twice the real width for one sample (3.4-3.6 mm
instead of 1.75); the check does not use the width.

## Troubleshooting

| Message | Cause |
|---|---|
| `FAIL: no data from bdwidth since Klipper start` | bdwidth sends nothing. Is it connected and switched on? Klipper's view of it (`active`, `enabled`) can say on while the sensor itself is off. |
| `motion 0.0/...` with filament loaded | bdwidth was switched off (`SET_BDWIDTH ... COMMAND=DISABLE`) while a print was running or paused, where the check leaves it alone |
| `pressure skipped (no check firmware)` | stock bd_pressure firmware, or `[bdpressure_check]` missing |
| `PURGE_CHECK ?:` | no lane loaded according to AFC |
