# Load-check extension (I2C)

An optional extension of the I2C register table that lets the host measure
nozzle pressure over a time window while the sensor stays in endstop
(probe) mode. Typical use: confirm that filament really reached the nozzle
after a filament swap, by checking for sustained pressure during the purge.

It does not use PA mode, so the host does not need `SET_BDPRESSURE START`,
which switches the X/Y stepper drivers and needs a re-home afterwards.

## Compatibility

- Registers 0-53 are unchanged. Hosts that do not know the extension never
  touch bytes 54 and up and see exactly the old behaviour.
- PA mode, the endstop output and the UART protocol are unchanged.
- The automatic baseline update in endstop mode only pauses while a check
  is running.
- Hosts must read the marker at 54-55 before writing byte 56. On firmware
  without the extension that byte is outside the register table.

## Registers

All values little-endian. Read with a single read starting at 54, 12 bytes.

| Offset | Size | Name | Access | Meaning |
|---|---|---|---|---|
| 54 | 2 | ext_magic | R | `0xBD 0x01`: extension present, version 1 |
| 56 | 1 | chk_ctrl | R/W | write 1 to start a check, 0 to stop it |
| 57 | 1 | chk_state | R | bit0: check running, bit1: above threshold now |
| 58 | 2 | chk_peak | R | largest \|delta\| since start |
| 60 | 2 | chk_above | R | ms above `THRHOLD_Z` since start (saturates at 65535) |
| 62 | 2 | chk_longest | R | longest continuous ms above `THRHOLD_Z` |
| 64 | 2 | chk_delta | R | current delta (signed), also updated outside a check |

`delta` is the same value the endstop trigger uses: the average of the last
4 samples minus the baseline.

## Behaviour

On start (`chk_ctrl` 0 -> 1) the firmware:

1. sets the baseline to the median of 16 block averages over the last 256
   samples (about 0.2 s), so a short burst just before the start is ignored,
2. clears peak and timers,
3. stops the automatic baseline update until the check is stopped, so a
   steady extrusion pressure keeps counting instead of being learned as the
   new zero.

Vibration gives short runs above the threshold, a sustained extrusion gives
one long run, so `chk_longest` separates the two. The host should start the
check while the toolhead is still (after `M400` and a short dwell).

## Klipper

`klipper/bdpressure_check.py` adds `BDP_CHECK_START`, `BDP_CHECK_STOP` and
`BDP_CHECK_QUERY` on top of an existing `[bdpressure <name>]` I2C section,
and reports `available, running, peak, above_ms, longest_ms, delta` in its
status. On firmware without the extension it reports `available: False` and
writes nothing.

```
[bdpressure_check]
sensor: bd_pa
```
