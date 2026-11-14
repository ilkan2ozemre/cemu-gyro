<img src="assets/icon.svg" width="64" height="64" alt="">

# Cemu Gyro

A Cemu patch that makes the **MSI Claw 8 AI+**'s built-in gyro work as Wii U GamePad motion.

## Why the motion option is greyed out

Cemu turns on "Use motion" only when the selected controller has its own gyro and
accelerometer (`SDLController::has_motion()`). On the Claw, the gamepad is a plain
XInput device with no motion sensors. The IMU is a separate Windows sensor called
"Physical Gyrometer" / "Physical Accelerometer", and it reports through the *custom*
sensor value fields, not the standard ones. SDL's Windows sensor backend can't read
those fields, so Cemu never sees a gyro.

## What the patch does

[`patches/0001-system-imu-motion.patch`](patches/0001-system-imu-motion.patch), against Cemu `d4f7205`:

- Adds `src/input/api/SDL/SystemIMU.cpp` (Windows only). It finds the built-in
  gyrometer and accelerometer through the Windows Sensor API (custom fields first, then
  the standard fields) and polls them at 250 Hz.
- When an SDL gamepad has no motion sensors of its own, `SDLController` reports motion
  anyway and takes its samples from the system IMU. Those samples go through the same
  motion integration code SDL gamepads already use.
- Linux and macOS builds are not affected.

The data fields and the axis mapping come from Handheld Companion's Claw A2VM device
config (`Resources/Devices/ClawA2VM.json`).

## Build

**Easiest: build on GitHub using Cemu's own CI**

1. Fork https://github.com/cemu-project/Cemu. Open the fork's **Actions** tab and
   enable workflows. GitHub turns them off on new forks.
2. Apply the patch on top of your fork's `main` and push it:
   ```sh
   git clone --recurse-submodules https://github.com/<you>/Cemu && cd Cemu
   git am -3 /path/to/0001-system-imu-motion.patch
   git push origin main
   ```
3. In your fork, open Actions → "Build check" → the latest run, then download
   `cemu-bin-windows-x64`.

**Local build:** follow Cemu's `BUILD.md` (Visual Studio 2022 + vcpkg) after running
`git am`.

## Use it

1. Check that Windows sees the IMU. Run this in PowerShell:
   `Get-PnpDevice -Class Sensor | ft FriendlyName, Status`.
   You should see a gyrometer and an accelerometer.
2. In Cemu, go to Input settings → Wii U GamePad and set the API to **SDLController**,
   not XInput. Pick the Claw's controller, open its settings, and tick **Use motion**.
3. Lay the Claw flat on a table, screen up, *before* starting Cemu, and leave it there for
   about 5 seconds after launch. Cemu logs a reading at startup and another one 5 seconds
   later. Then look in `log.txt` for lines starting with `SystemIMU:`. They show which
   sensors were found and the sample readings. The "mapped accel" line should read about
   `0 1 0`. A `reading ... failed` line means Windows found the sensor but won't hand
   over its data.

### If an axis is wrong

These environment variables set how the raw axes map to SDL's axes. Set them before
starting Cemu, then restart Cemu after each change:

| Variable | Default | Purpose |
|---|---|---|
| `CEMU_SYSTEM_IMU_AXES` | `x,z,-y` | Mapping for both sensors |
| `CEMU_SYSTEM_IMU_GYRO_AXES` | – | Gyro-only override |
| `CEMU_SYSTEM_IMU_ACCEL_AXES` | – | Accel-only override |
| `CEMU_SYSTEM_IMU_RATE` | `250` | Polling rate in Hz (30–1000) |
| `CEMU_SYSTEM_IMU=0` | – | Turns the fallback off |

To test, lay the Claw flat and check the log. Then tilt it toward you and turn it left
and right, and watch the GamePad view (for example in Mario Kart 8 or Splatoon). If one
axis moves the wrong way, flip its sign, for example `x,z,y`. If two axes are swapped,
swap their letters.

## Status

The patch has **not been compiled or tested on a Claw**. It only applies cleanly to
Cemu `d4f7205`. If the IMU doesn't show up on your unit, use this fallback with the
stock Cemu build: run Handheld Companion's DSU server and point Cemu's DSU client at it.
