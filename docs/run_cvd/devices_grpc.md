# gRPC and Unix Socket Devices

These devices communicate via Unix Domain Sockets (UDS), typically exposing
gRPC endpoints for external control. They don't directly communicate with the
VMM but provide control-plane services to test automation, the WebRTC frontend,
or other host-side tools.

---

## Casimir (NFC Emulator)

**Binary**: `casimir`
**Source**: [casimir.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/casimir.cpp)
**Registration**: `AutoCmd<Casimir>::Component`

### Communication Channels

| Channel                   | Type        | Purpose                                         |
| ------------------------- | ----------- | ----------------------------------------------- |
| `casimir_nci_socket_path` | Unix socket | NCI protocol (connected by NfcConnector)        |
| `casimir_rf_socket_path`  | Unix socket | RF protocol (connected by CasimirControlServer) |

### Description

Casimir is the NFC emulator that implements the NCI (NFC Controller Interface)
protocol. The guest NFC HAL communicates with it via the NfcConnector
(hvc12 → tcp_connector → casimir NCI socket). The CasimirControlServer
connects to the RF socket for external NFC tag/device emulation.

### Special Features

- **process_restarter**: **Yes** — wraps binary in process_restarter
- **log_tee**: **Yes** — captures stderr via FIFO

### Dependencies

- `CuttlefishConfig`, `InstanceSpecific`
- `LogTeeCreator`
- `GrpcSocketCreator`

### Direction

Bidirectional — NCI protocol from guest; RF protocol from control server.

### Condition

Only launched when `config.enable_host_nfc()` is true.

---

## CasimirControlServer

**Binary**: `casimir_control_server`
**Source**: [casimir_control_server.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/casimir_control_server.cpp)
**Registration**: `AutoCmd<CasimirControlServer>::Component`

### Communication Channels

| Channel                  | Type        | Purpose                              |
| ------------------------ | ----------- | ------------------------------------ |
| gRPC UDS                 | Unix socket | External NFC control (tag emulation) |
| `casimir_rf_socket_path` | Unix socket | Connects to Casimir RF interface     |

### Description

Provides a gRPC control interface for the Casimir NFC emulator. Test automation
tools can use this to emulate NFC tags, simulate tap events, and control the
NFC RF environment.

### gRPC Endpoint

- `grpc_socket_path("CasimirControlServer")`

### Dependencies

- `CuttlefishConfig`, `InstanceSpecific`
- `GrpcSocketCreator`

### Direction

External → Host (gRPC control plane for NFC).

### Condition

Only launched when `config.enable_host_nfc()` is true.

---

## EchoServer

**Binary**: `echo_server`
**Source**: [echo_server.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/echo_server.cpp)
**Registration**: `AutoCmd<EchoServer>::Component`

### Communication Channels

| Channel  | Type        | Purpose                               |
| -------- | ----------- | ------------------------------------- |
| gRPC UDS | Unix socket | Echo service for connectivity testing |

### Description

A simple gRPC echo server used for testing the gRPC infrastructure. Echoes
back any message sent to it. Useful for verifying that the gRPC UDS path
works correctly.

### gRPC Endpoint

- `grpc_socket_path("EchoServer")`

### Dependencies

- `CuttlefishConfig::InstanceSpecific`
- `GrpcSocketCreator`

### Direction

External → Host (gRPC echo for testing).

---

## ControlEnvProxyServer

**Binary**: `control_env_proxy_server`
**Source**: [control_env_proxy_server.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/control_env_proxy_server.cpp)
**Registration**: `ControlEnvProxyServerComponent` (manual Fruit component class)

### Communication Channels

| Channel            | Type        | Purpose                                  |
| ------------------ | ----------- | ---------------------------------------- |
| gRPC UDS           | Unix socket | Control environment proxy                |
| `grpc_socket_path` | Unix socket | Connects to instance's own gRPC services |

### Description

Acts as a proxy for the control environment, forwarding gRPC requests to the
appropriate backend services. Used by `cvd` command-line tool and the web
frontend to interact with the running instance.

### gRPC Endpoint

- `grpc_socket_path("ControlEnvProxyServer")`

### Dependencies

- `CuttlefishConfig`, `InstanceSpecific`
- `GrpcSocketCreator`

### Direction

External → Host → Guest (proxies control commands).

---

## ScreenRecordingServer

**Binary**: `screen_recording_server`
**Source**: [screen_recording_server.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/screen_recording_server.cpp)
**Registration**: `AutoCmd<ScreenRecordingServer>::Component`

### Communication Channels

| Channel  | Type        | Purpose                  |
| -------- | ----------- | ------------------------ |
| gRPC UDS | Unix socket | Screen recording control |

### Description

Provides a gRPC interface to start/stop screen recording and take screenshots
of the running virtual device.

### gRPC Endpoint

- `grpc_socket_path("ScreenRecordingServer")`

### Dependencies

- `CuttlefishConfig::InstanceSpecific`
- `GrpcSocketCreator`

### Direction

External → Host (gRPC control for screen recording).

---

## OpenwrtControlServer

**Binary**: `openwrt_control_server`
**Source**: [openwrt_control_server.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/openwrt_control_server.cpp)
**Registration**: `OpenwrtControlServerComponent` (manual Fruit component class)

### Communication Channels

| Channel  | Type        | Purpose               |
| -------- | ----------- | --------------------- |
| gRPC UDS | Unix socket | OpenWrt AP VM control |

### Description

Provides a gRPC control interface for the OpenWrt access point VM. Allows
external tools to configure the WiFi AP, manage network settings, and
interact with the OpenWrt instance.

### gRPC Endpoint

- `grpc_socket_path("OpenwrtControlServer")`

### Dependencies

- `CuttlefishConfig`, `InstanceSpecific`, `EnvironmentSpecific`
- `GrpcSocketCreator`

### Direction

External → Host (gRPC control for OpenWrt AP).

### Condition

Only launched when the OpenWrt VM is enabled and `virtio_mac80211_hwsim` is
configured.
