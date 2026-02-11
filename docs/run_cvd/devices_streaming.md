# Streaming Devices

## WebRTC Streamer

**Binary**: `webrtc` + `webrtc_sig_server_proxy`
**Source**: [streamer.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/streamer.cpp)
**Registration**: `launchStreamerComponent` (manual Fruit component class)

### Communication Channels

| Channel             | Type                               | Purpose                                          |
| ------------------- | ---------------------------------- | ------------------------------------------------ |
| Frames socket       | Unix socket (`frames_socket_path`) | Receives GPU frames from crosvm (via Wayland)    |
| Audio socket        | Unix socket (`audio_server_path`)  | Receives audio from crosvm                       |
| ConfUI FIFOs        | FIFO pair                          | ConfirmationUI frames                            |
| Input socket pairs  | Socket pairs (×N)                  | Touch/kb/mouse/rotary/gamepad input from browser |
| Sensors socket pair | Socket pair                        | Sensor injection from browser                    |
| Kernel log pipe     | Pipe FD                            | Kernel log streaming to browser console          |
| Command channel     | Socket pair                        | Control commands from WebRtcController           |

### Description

The WebRTC streamer is the primary user interface for Cuttlefish. It captures
the virtual device's display output (via the crosvm Wayland socket), audio
output, and kernel logs, then streams them to a web browser via WebRTC. It also
receives input events (touch, keyboard, mouse, rotary, gamepad) from the browser
and injects them into the guest via the VhostInputDevices socket pairs.

### Process Tree

```
webrtc
└── webrtc_sig_server_proxy
```

The signaling server proxy handles WebRTC session negotiation and can be
configured to connect to an external signaling server.

### Supporting Components

- **WebRtcController** ([webrtc_controller.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/webrtc_controller.cpp)):
  Creates a socket pair command channel to send control commands (start/stop
  recording, screenshot) to the streamer process. Not a device launcher itself.

- **InputConnectionsProvider** ([input_connections_provider.h](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/input_connections_provider.h)):
  Abstract interface implemented by VhostInputDevices to provide input socket
  pairs to the streamer.

- **EnableMultitouch** ([enable_multitouch.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/enable_multitouch.cpp)):
  Helper that returns a boolean indicating whether multi-touch is supported.

### Diagnostics

Reports: `"Point your browser to https://localhost:<port>"` with the WebRTC
server URL.

### Dependencies

- `CuttlefishConfig`, `InstanceSpecific`
- `KernelLogPipeProvider` — for kernel log streaming
- `InputConnectionsProvider` — socket pairs for input devices
- `SensorsSocketPair` — for sensor injection
- `WebRtcController` — for command channel
- `SnapshotControlFiles` — for snapshot coordination

### Direction

Bidirectional:

- Guest → Browser: display frames, audio, kernel logs
- Browser → Guest: touch, keyboard, mouse, rotary, gamepad, sensor events

### Condition

Only launched when `instance.enable_webrtc()` is true.

### Platform

Linux-only (`#ifdef __linux__`).

### Custom Action Servers

The streamer can also launch custom action server binaries specified in the
instance configuration, providing extensible browser-accessible actions.
