# Teensy 4.1 sensor bring-up

This PlatformIO firmware polls a BMP390 at 25 Hz and an ADXL375 at 800 Hz on
the Teensy 4.1's default `Wire` bus. It checks both device IDs, retries failed
initialization once per second, reports sensor/I2C errors without stopping the
loop, captures a ground-pressure reference on the first good BMP390 sample,
and outputs relative altitude and acceleration.

## Wiring

Use 3.3 V logic and connect the grounds together.

| Sensor | Sensor pin | Teensy 4.1 |
|---|---|---|
| Both | GND | GND |
| Both | VIN/VCC | 3.3 V (confirm breakout's input requirements) |
| Both | SDA | pin 18 (SDA0) |
| Both | SCL | pin 19 (SCL0) |

The BMP390 address is `0x77`; the ADXL375 address is `0x53`. If a breakout has
pull-ups, confirm that the combined bus has suitable pull-ups to 3.3 V.

## Build and run

Open this directory in PlatformIO and build/upload the `teensy41` environment.
Open the serial monitor at 921600 baud. The startup banner appears after sensor
initialization; the built-in LED toggles at 1 Hz. No application sampling loop
uses `delay()`.

Serial output is CSV-like:

* `A,sequence,x_g,y_g,z_g` — one ADXL375 sample; sensitivity is 0.049 g/LSB.
* `B,micros,pressure_hPa,temperature_C,altitude_m` — BMP390 data. Altitude is
  relative to the first valid pressure (or the most recent `r` command).
* `ERROR,flags=...` — latched error flags: bit 0 BMP390 unavailable, bit 1
  ADXL375 unavailable, bit 2 BMP390 error register, bit 3 accelerometer
  overrun/possible FIFO loss, bit 4 barometer schedule overrun.
* `I2C,...,avg_us=...,max_us=...` — measured sensor read duration over the
  preceding reporting interval. These timings include software/API overhead;
  use a scope or logic analyzer for wire-only transaction duration.
* `RATE,...` — per-second `accel_hz`/`baro_hz`, cumulative missed schedule
  counts, FIFO near-full events, and the current error flags.

Send `f` to enable ADXL375 stream FIFO mode (drain every 5 ms), `b` for direct
read mode (poll every 1.25 ms), or `r` to capture a new ground reference.
Compare measured rates, max service times, and FIFO near-full events
between modes. The FIFO test measures buffering/jitter tolerance; the ADXL375
still requires one six-byte burst read for each sample, so FIFO does not
intrinsically reduce the sensor data bytes or per-sample I2C transactions.

## Bus budget

The estimates below assume 400 kHz I2C and count address, register, and data
bytes at 9 SCL clocks per byte. START/STOP timing and clock stretching are not
included. Replace the measured columns with the `I2C` output from the actual
board, or with logic-analyzer measurements if wire-only time is required.

| Transfer | Target rate | Bytes per transaction | Estimated wire time | Estimated bus use | Measured avg / max |
|---|---:|---:|---:|---:|---|
| ADXL375 data burst (register pointer + six data bytes) | 800/s | 9 | 203 us | 16.2% | Run firmware |
| BMP390 pressure/temperature read (register pointer + six bytes) | 25/s | 9 | 203 us | 0.51% | Run firmware |
| BMP390 error-register read | 25/s | 4 | 90 us | 0.23% | Run firmware |
| FIFO status in stream mode | 200/s | 4 | 90 us | 1.8% | Run firmware |
| **Direct-mode total, excluding setup** | — | — | — | **about 16.9%** | Run firmware |
| **FIFO-mode total, excluding setup** | — | — | — | **about 18.7%** | Run firmware |

The `bmp_data` value measures the BMP library's complete read operation, not
strictly a single I2C transaction. The ADXL375 data timing measures the
register-pointer plus six-byte burst operation; `fifo_status` includes its
separate FIFO status read. `micros()` resolution and software overhead make
these suitable for a practical budget, not a substitute for a scope.

## Bench validation

The firmware provides telemetry and error detection but cannot establish
hardware pass results by itself. Record the board, supply, and observed values
when performing these tests:

1. **Still test:** run for two minutes with the board stationary. Verify
   `missed_accel=0`, `missed_baro=0`, no error flags, roughly 800 acceleration
   records/s and 25 barometer records/s. Note pressure/altitude drift.
2. **Height test:** keep the board still, then move it to a known higher
   location (elevator or stairwell). Confirm reported relative altitude
   increases and record the known and observed height change. Avoid rapid
   pressure changes from handling or airflow.
3. **Shake test:** shake the board safely by hand and confirm the acceleration
   output responds on all axes without persistent overrun flags.
4. **FIFO comparison:** run a still/shake interval in direct mode, then send
   `f` and repeat. Compare throughput, maximum service time, schedule misses,
   and FIFO near-full events. FIFO mode should be judged by measured results;
   its status polling adds bus traffic and the per-sample data reads remain.
5. **Disconnect test:** unplug each sensor separately while powered only if
   safe for the specific breakout. Confirm the corresponding error flag appears,
   firmware continues running, and reconnecting clears the unavailable flag
   after successful retry.
6. **Ten-minute run:** leave both sensors connected and verify the cumulative
   missed counts remain zero. Save serial output as the run record.

| Test | Result / measurements | Pass |
|---|---|---|
| Two-minute stationary run | Not run (requires hardware) | — |
| Known height change | Not run (requires hardware) | — |
| Shake test | Not run (requires hardware) | — |
| Ten-minute rate run | Not run (requires hardware) | — |
| Individual sensor disconnect/reconnect | Not run (requires hardware) | — |
