# IO Guards and Respawn

This document describes the stability changes applied on branch
`feat/io-guards-and-respawn`. The intent is to keep the rest of the ROS 2 graph
alive when the u-blox device misbehaves (transient serial errors, missing power,
firmware in a bad state).

## Files Changed

- `ublox_serialization/include/ublox_serialization/serialization.hpp`
  - `Reader::read<T>()`: wrap `UbloxSerializer<T>::read(...)` in `try/catch`
    for `std::system_error`. On exception, return `false` instead of
    propagating.
  - `Writer::write<T>(message, ...)`: same guard around
    `UbloxSerializer<T>::write(...)`.
- `ublox_gps/src/node.cpp`
  - Added `<thread>` include.
  - `UbloxNode::initialize()`: replaced single-attempt `configureUblox()` with
    a bounded retry loop. On total failure, calls `shutdown()` and
    `rclcpp::shutdown()` so the launch system can respawn the process.
- `ublox_gps/launch/ublox_gps_node-launch.py`
  - `Node` action gets `respawn` and `respawn_delay`, controlled by env vars.
  - Removed the `OnProcessExit -> EmitEvent(Shutdown)` handler. With respawn
    enabled, that handler would cancel the respawn behavior on first exit.
- `ublox_gps/launch/ublox_gps_node-composed-launch.py`
  - `ComposableNodeContainer` gets the same `respawn` and `respawn_delay`
    options. The container is what the OS sees as a process, so respawn is
    applied there.

## Why

### Serializer try/catch
`UbloxSerializer::read` / `write` can throw `std::system_error` on malformed
buffers or short reads. Without a guard the exception unwinds out of the
async-worker callback and terminates the node. The guard converts these into
a soft failure (return `false`); the worker drops the bad packet and continues.

### Bounded configure retry
The original `initialize()` called `configureUblox()` once. If the device was
not enumerated yet (USB still settling, power-on race) the node would silently
sit with no timers and no subscriptions. The new behavior retries up to
`kMaxConfigureAttempts` (5) times with a 10 second delay, logging a warning on
each retry. If still unsuccessful, the node calls `rclcpp::shutdown()` so that
launch respawn can restart the entire process from scratch (which also
re-opens the serial port).

### Launch respawn
With the node now able to exit cleanly on init failure, `respawn=True` on the
launch action makes the node come back automatically. Previously the
`OnProcessExit` handler emitted a global shutdown event when the ublox node
exited, which would tear down every other node in the same launch — for a
multi-sensor robot that is the wrong default. Respawn keeps everything else
alive.

## Opting Out

Two environment variables:

| Variable        | Default | Effect                                         |
|-----------------|---------|------------------------------------------------|
| `RESPAWN_NODES` | `1`     | `0` disables respawn.                           |
| `RESPAWN_DELAY` | `5`     | Seconds between exit and restart.               |

Example:

```bash
RESPAWN_NODES=0 ros2 launch ublox_gps ublox_gps_node-launch.py
```

The retry-loop bounds inside `initialize()` are compile-time constants in
`node.cpp` (`kMaxConfigureAttempts`, `kConfigureRetryDelay`). They are
intentionally not exposed as ROS parameters; if you need to tune them, edit
the source.

## Source Attribution

The patterns originate from the KiwiCampus fork of `ublox`
(`kiwicampus/ublox`, branch `foxy-devel`). Specific upstream commits whose
ideas were ported by hand to ROS 2 Humble:

- `3809ef9` `[FEAT] try catch on read and write operations`
- `f54de7d` `[FEAT] Catch ublox in infinity loop`
- `c907f0f` `[FEAT]: shutdown node is ublox doesn't up`
- `4f3ad9c` `Avoid crashing whole system when crashing ublox / Add respawn with env var`

The ports are not direct cherry-picks; the diffs were re-applied against the
snslab humble tree and merged so that the "infinite retry" and
"shutdown if not up" ideas coexist as a bounded retry plus shutdown.
