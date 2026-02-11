# Vhost-User Devices

Vhost-user devices run as separate processes that implement virtio device
backends. Crosvm connects to them via Unix domain sockets using the vhost-user
protocol, and the devices appear as native virtio PCI devices inside the guest.
This architecture allows device emulation to run in separate address spaces
(security isolation) and enables different process lifecycle management.

---

## VhostDeviceVsock

**Binary**: `vhost_device_vsock` (Rust)
**Source**: [vhost_device_vsock.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/vhost_device_vsock.cpp)
**Registration**: `VhostDeviceVsockComponent` (manual Fruit component class)

### Communication Channels

- **Vhost-user socket**: `<tmpdir>/vsock_<cid>_<uid>/vhost.socket`
  (connected to crosvm via `--vhost-user=vsock,socket=...`)
- **VM socket directory**: `<tmpdir>/vsock_<cid>_<uid>/vm.vsock`
  (host-side Unix socket for vsock connections)

### Description

Provides the virtio-vsock transport layer as a vhost-user device. This is the
foundation that enables all vsock-based devices (ModemSimulator,
TombstoneReceiver, VhalProxyServer, socket_vsock_proxy) to communicate with the
guest. When vhost-user vsock is enabled, other vsock-using devices connect to
Unix sockets in the VM socket directory instead of using kernel vsock.

### Special Features

- **VmmDependencyCommand**: **Yes** — must be ready before crosvm starts
- **process_restarter**: **Yes** — wraps binary in process_restarter
- **log_tee**: **Yes** — captures stderr via FIFO
- **WaitForAvailability()**: Waits for the vhost.socket Unix socket to appear
  (30-second timeout via `WaitForUnixSocketListeningWithoutConnect`)
- **Environment**: Sets `RUST_BACKTRACE=1` and `RUST_LOG=debug`
- **Shared instance**: Only the first instance launches the process; other
  instances share it

### Dependencies

- `CuttlefishConfig`, `InstanceSpecific`, `EnvironmentSpecific`
- `LogTeeCreator`

### Host Resources

- Vhost-user socket in `<tmpdir>`
- VM vsock directory

### Direction

Bidirectional — provides the vsock transport channel between host and guest.

### Platform

Linux-only (`#ifdef __linux__`).

---

## VhostInputDevices

**Binary**: `vhost_user_input` (one process per input device)
**Source**: [vhost_input_devices.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/vhost_input_devices.cpp)
**Registration**: `VhostInputDevicesComponent` (manual Fruit component class)

### Communication Channels

For each input device, a vhost-user Unix socket:

| Device Type      | Socket Path                            | Connects To                 |
| ---------------- | -------------------------------------- | --------------------------- |
| Touchscreen (×N) | `touch_socket_path(i)`                 | crosvm `--vhost-user=input` |
| Touchpad (×N)    | `touch_socket_path(display_count + i)` | crosvm `--vhost-user=input` |
| Mouse            | `mouse_socket_path()`                  | crosvm `--vhost-user=input` |
| Keyboard         | `keyboard_socket_path()`               | crosvm `--vhost-user=input` |
| Rotary           | `rotary_socket_path()`                 | crosvm `--vhost-user=input` |
| Gamepad          | `gamepad_socket_path()`                | crosvm `--vhost-user=input` |
| Switches         | `switches_socket_path()`               | crosvm `--vhost-user=input` |

Each `vhost_user_input` process also receives a socket pair: one end goes to
the WebRTC streamer (for browser input injection), and the other end is kept by
the input device process.

### Description

Each input device is a separate `vhost_user_input` process that implements the
virtio-input protocol via vhost-user. The WebRTC streamer forwards browser
input events through the socket pair to the input device, which then injects
them into the guest via the virtio-input vhost-user protocol.

### Process Tree

```
vhost_user_input (rotary)     + log_tee
vhost_user_input (keyboard)   + log_tee
vhost_user_input (switches)   + log_tee
vhost_user_input (touch 0)    + log_tee
vhost_user_input (touch 1)    + log_tee  (if multi-display)
vhost_user_input (touchpad 0) + log_tee  (if touchpad configured)
vhost_user_input (mouse)      + log_tee  (if enabled)
vhost_user_input (gamepad)    + log_tee  (if enabled)
```

### Special Features

- **VmmDependencyCommand**: No (but the vhost-user sockets must be available
  before crosvm fully initializes — crosvm handles the timeout via
  `--vhost-user-connect-timeout-ms`)
- **process_restarter**: No
- **log_tee**: **Yes** — one log_tee per input device, capturing stderr
- **InputConnectionsProvider**: Implements this interface to provide socket
  pairs to the WebRTC streamer

### Dependencies

- `CuttlefishConfig`, `InstanceSpecific`
- `LogTeeCreator`

### Direction

Host → Guest (input injection from browser/host tools into the VM).

### Platform

Linux-only (`#ifdef __linux__`).

---

## WmediumdServer

**Binary**: `wmediumd`
**Source**: [wmediumd_server.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/wmediumd_server.cpp)
**Registration**: `WmediumdServerComponent` (manual Fruit component class)

### Communication Channels

| Channel                      | Type              | Purpose                       |
| ---------------------------- | ----------------- | ----------------------------- |
| `vhost_user_mac80211_hwsim`  | Vhost-user socket | mac80211-hwsim WiFi medium    |
| `wmediumd_api_server_socket` | Unix socket       | API server for runtime config |
| gRPC UDS                     | Unix socket       | gRPC control endpoint         |

### Description

Wmediumd simulates the wireless medium for mac80211-hwsim virtual WiFi devices.
It controls packet delivery between multiple virtual WiFi interfaces (e.g.,
between the main Android VM and the OpenWrt AP VM), enabling realistic WiFi
emulation with configurable signal strength, packet loss, and medium access.

### Special Features

- **VmmDependencyCommand**: **Yes** — must be ready before both the main crosvm
  and the OpenWrt crosvm start
- **process_restarter**: No
- **log_tee**: **Yes** — captures stderr via FIFO
- **WaitForAvailability()**: Waits for the API server socket to appear
- **Config generation**: If no config file is provided, runs
  `wmediumd_gen_config` to generate a default configuration
- **Multi-instance awareness**: Instances that don't own the wmediumd process
  still register a `WaitForWmediumd` `VmmDependencyCommand` that blocks until
  the shared wmediumd is ready

### Dependencies

- `CuttlefishConfig`, `InstanceSpecific`, `EnvironmentSpecific`
- `LogTeeCreator`
- `GrpcSocketCreator`

### gRPC Endpoint

- `grpc_socket_path("WmediumdServer")`

### Direction

Bidirectional — simulates wireless medium between VMs; gRPC for external
control.

### Condition

Only launched when `config.virtio_mac80211_hwsim()` is true.

### Platform

Linux-only (`#ifdef __linux__`).
