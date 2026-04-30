# Verification Plan — `feat/io-guards-and-respawn`

This document defines how to verify the branch on a target machine that has
**ROS 2 Humble** installed and a **physical u-blox device** connected. The
patches were authored on a Jazzy host without hardware, so all runtime
acceptance criteria below are intentionally outstanding.

The branch ships three logically separate changes; each has its own test
section with explicit PASS/FAIL criteria.

## Prerequisites

| Item | Required state |
|------|----------------|
| OS | Ubuntu 22.04 |
| ROS | Humble installed and sourced (`source /opt/ros/humble/setup.bash`) |
| System packages | `libasio-dev` installed (`sudo apt-get install -y libasio-dev`) |
| Hardware | u-blox receiver enumerated as `/dev/ttyACM0` (or equivalent), powered, with antenna |
| Workspace | `/home/nev/dev/ros2_ws` with `src/ublox_snslab` checked out at `feat/io-guards-and-respawn` |

Adjust `device:=/dev/ttyACM0` and `frame_id`, baud rate, etc. to your setup
through the existing config files. None of the parameters were renamed by
this branch.

## Step 1 — Build

```bash
cd /home/nev/dev/ros2_ws
rm -rf build/ublox_serialization build/ublox_msgs build/ublox_gps \
       install/ublox_serialization install/ublox_msgs install/ublox_gps
colcon build --packages-select ublox_serialization ublox_msgs ublox_gps ublox
```

**PASS:** all four packages finish without errors. Warnings about deprecated
ROS API calls that were already present on `humble` are acceptable; new
warnings introduced by this branch are not.

**Known caveat:** `ublox_gps` requires `libasio-dev` at cmake time. If cmake
reports `Could NOT find asio (missing: ASIO_INCLUDE_DIR)`, install the package
and re-run. This is an environment issue, not a regression.

After the build, source the overlay:

```bash
source install/setup.bash
```

## Step 2 — Smoke test (device connected)

Goal: confirm the patched node still boots end-to-end on real hardware.

```bash
ros2 launch ublox_gps ublox_gps_node-launch.py
```

In another shell:

```bash
ros2 topic hz /ublox_gps_node/fix
ros2 topic echo /ublox_gps_node/fix --once
```

**PASS:** `/fix` publishes at the configured rate; `status` field is non-zero
once the receiver has a fix. No stack traces in the launch log. No respawn
events (`process started with pid` should appear exactly once).

**FAIL signals:** node exits within seconds, `OnProcessExit` event fires,
`/fix` never appears, or the launch starts respawn-looping with the device
plugged in.

## Step 3 — Serializer try/catch (T1)

Goal: confirm `std::system_error` thrown from `UbloxSerializer<T>::read` /
`write` is swallowed and the node keeps running.

This path is hard to provoke deterministically without an injection harness.
A practical proxy: yank the USB cable mid-stream.

1. Start the node with the device connected, wait for steady `/fix` publishing.
2. Unplug the device.
3. Observe the launch log for ~30 seconds.

**PASS:** the log shows transient warnings (read errors, lost packets) but
the process **does not exit**. Replug the device; reconnection may or may not
happen automatically depending on the existing driver behavior — that is out
of scope for this branch. The acceptance is just that the node didn't crash.

**FAIL:** unhandled `std::system_error` causes the process to terminate with
a stack trace before any retry loop kicks in. (If respawn is enabled the
process will then come back, which would falsely look like a pass — confirm
by checking the PID changed in the log.)

## Step 4 — Bounded configure retry (T2)

Goal: confirm `initialize()` retries 5 times and then calls
`rclcpp::shutdown()`.

Run with **no device connected** (or an invalid `device` path):

```bash
RESPAWN_NODES=0 ros2 launch ublox_gps ublox_gps_node-launch.py \
  device:=/dev/ttyDOES_NOT_EXIST
```

`RESPAWN_NODES=0` prevents respawn so the test is observable.

**PASS:**
- 5 warning-level log lines about retrying configuration, ~10 seconds apart.
- A final error indicating shutdown.
- The process exits with the launch reporting a clean exit (not a SIGABRT).
- Total elapsed time ≈ 50 seconds.

**FAIL:** the node exits after one attempt, retries forever, or aborts.

## Step 5 — Launch respawn (T3)

Goal: confirm the launch action restarts the node automatically after the
bounded retry shutdown.

Same scenario as Step 4 but with respawn enabled (the default):

```bash
ros2 launch ublox_gps ublox_gps_node-launch.py \
  device:=/dev/ttyDOES_NOT_EXIST
```

**PASS:**
- After the first ≈50 s retry cycle and shutdown, launch logs a respawn
  message and starts a new process with a different PID after `RESPAWN_DELAY`
  seconds (default 5).
- The cycle repeats indefinitely.
- Other nodes in the same launch (none in this minimal launch, but confirm
  in a multi-node setup if applicable) are not torn down at any point.

**FAIL:** the launch terminates after the first node exit (would indicate
that the removed `OnProcessExit -> Shutdown` handler is still active or that
respawn was not honored).

## Step 6 — Respawn opt-out

```bash
RESPAWN_NODES=0 ros2 launch ublox_gps ublox_gps_node-launch.py \
  device:=/dev/ttyDOES_NOT_EXIST
```

**PASS:** the node retries 5 times, exits, and the launch ends. No respawn.

```bash
RESPAWN_DELAY=20 ros2 launch ublox_gps ublox_gps_node-launch.py \
  device:=/dev/ttyDOES_NOT_EXIST
```

**PASS:** the gap between exit and the next process start is ≈20 seconds
instead of 5.

## Step 7 — Composed launch parity

Repeat Step 5 with the composed launch:

```bash
ros2 launch ublox_gps ublox_gps_node-composed-launch.py \
  device:=/dev/ttyDOES_NOT_EXIST
```

**PASS:** the `ComposableNodeContainer` process restarts on the same cadence.
Recall that respawn is on the container, not the composable node — so what
you should see is the entire container PID changing.

## Step 8 — Mid-cycle recovery

Goal: confirm the user-facing benefit. The node should self-heal once the
device appears.

1. Start the launch with `device:=/dev/ttyACM0` while the device is unplugged.
2. Observe at least one full retry-and-shutdown cycle (5 retries → respawn).
3. **During** the next retry window, plug the device in.
4. Within one retry attempt the configure should succeed, and the node should
   stop respawning.

**PASS:** after plugging, the next process iteration reaches steady-state and
publishes `/fix` without further respawns.

## Regression checks

Verify the following still work, since this branch touched serialization
and node init:

- `/diagnostics` topic still publishes the same diagnostic-status keys.
- `/ublox_gps_node/fix_velocity` (or whatever was published before) is still
  published.
- The composed launch composes into the existing container as before.
- All existing launch arguments (`device`, `frame_id`, parameter file paths)
  still work and override correctly.

If any of these regress, capture the full log and the exact `colcon build`
output and revert the most likely culprit (serialization patch first, since
its blast radius is widest).

## What to report back

A short note recording, per step, PASS / FAIL / N-A and any log excerpts that
were unexpected. If everything passes, the branch is ready to merge to
`humble`.
