# Cuttlefish Virtual Device Architecture

This document provides a comprehensive reference to the `run_cvd` virtual device
architecture — how the Cuttlefish host tooling orchestrates crosvm, ~30 device
emulation processes, and the communication channels connecting them to the
Android guest VM.

## Table of Contents

- [Overview](#overview)
- [Architecture Diagram](#architecture-diagram)
- [Crosvm and the VM Manager](#crosvm-and-the-vm-manager)
- [Device Catalog](#device-catalog)
- [Architectural Patterns and Inconsistencies](patterns_and_inconsistencies.md)
- [Extension Guide: Adding a New Virtual Device](extension_guide.md)

---

## Overview

`run_cvd` is the core process that launches and supervises a single Cuttlefish
virtual device instance. It is invoked by `launch_cvd` after `assemble_cvd`
has prepared the configuration. `run_cvd` performs three main tasks:

1. **Dependency injection** — Uses Google's [Fruit](https://github.com/nicoa/fruit)
   framework to wire together ~30 device launchers, configuration fragments,
   and diagnostic providers into a single dependency graph.

2. **Feature setup** — Performs a topological sort of all `SetupFeature` nodes
   and executes them in dependency order. Each `CommandSource` feature produces
   one or more `MonitorCommand` objects (a `Command` + `is_critical` flag).

3. **Server loop** — Monitors all child processes, restarts non-critical ones
   on failure, and coordinates VM lifecycle events (snapshot, restore, shutdown).

### Process Tree

```
launch_cvd
 └── run_cvd  (orchestrator, reads config from stdin)
      ├── crosvm run ...                    (main VM, is_critical=true)
      │    └── [KVM guest: Android]
      ├── crosvm run ...                    (OpenWrt AP VM, optional)
      │    └── [KVM guest: OpenWrt]
      ├── kernel_log_monitor                (reads /dev/hvc0 via FIFO)
      ├── logcat_receiver                   (reads /dev/hvc2 via FIFO)
      ├── log_tee (crosvm)                  (captures crosvm stderr)
      ├── secure_env                        (keymaster/gatekeeper/keymint)
      ├── console_forwarder                 (serial console via /dev/hvc1)
      ├── tcp_connector (bt)                (FIFO↔TCP for Bluetooth HCI)
      ├── tcp_connector (nfc)               (FIFO↔TCP for NFC NCI)
      ├── tcp_connector (uwb)               (FIFO↔TCP for UWB UCI)
      ├── gnss_grpc_proxy                   (GNSS injection via FIFO + gRPC)
      ├── sensors_simulator                 (sensor data via FIFO)
      ├── root-canal / netsimd              (Bluetooth emulator)
      │    └── socket_vsock_proxy (×4)      (vsock↔TCP bridges)
      ├── casimir                           (NFC emulator)
      ├── pica                              (UWB emulator)
      ├── modem_simulator                   (RIL modem via vsock)
      ├── tombstone_receiver                (crash dumps via vsock)
      ├── vhal_proxy_server                 (Vehicle HAL via vsock)
      ├── vhost_device_vsock                (vhost-user vsock transport)
      │    └── log_tee
      ├── vhost_user_input (×N)             (touch/kb/mouse/rotary/etc.)
      │    └── log_tee (×N)
      ├── wmediumd                          (WiFi medium simulator)
      │    └── log_tee
      ├── MCU binary                        (microcontroller emulator)
      │    └── log_tee
      ├── ti50_emulator                     (Titan security chip)
      │    └── log_tee
      ├── webrtc                            (browser streaming)
      │    └── webrtc_sig_server_proxy
      ├── metrics                           (telemetry)
      ├── echo_server                       (gRPC test endpoint)
      ├── control_env_proxy_server          (gRPC control plane)
      ├── screen_recording_server           (gRPC screen recording)
      ├── casimir_control_server            (gRPC NFC control)
      ├── openwrt_control_server            (gRPC OpenWrt control)
      ├── automotive_proxy                  (automotive proxy)
      └── cvdalloc                          (resource allocation)
```

---

## Architecture Diagram

### Communication Topology

```
┌─────────────────────────────────── Host (run_cvd) ───────────────────────────────────┐
│                                                                                       │
│  ┌──────────── Virtio-Console (HVC FIFOs) ────────────┐                              │
│  │                                                     │                              │
│  │  hvc0  ←── kernel_log_monitor ──→ kernel.log        │                              │
│  │  hvc1  ←→  console_forwarder  ←→  screen/terminal   │                              │
│  │  hvc2  ←── logcat_receiver    ──→ logcat             │                              │
│  │  hvc3  ←→  secure_env (keymaster)                   │                              │
│  │  hvc4  ←→  secure_env (gatekeeper)                  │                              │
│  │  hvc5  ←→  tcp_connector (bt)  ←→ root-canal/netsim │                              │
│  │  hvc6  ←→  gnss_grpc_proxy (gnss)                   │                              │
│  │  hvc7  ←→  gnss_grpc_proxy (location)               │                              │
│  │  hvc8  ←→  secure_env (confirmationui)              │                              │
│  │  hvc9  ←→  tcp_connector (uwb) ←→ pica              │                              │
│  │  hvc10 ←→  secure_env (oemlock)                     │                              │
│  │  hvc11 ←→  secure_env (keymint)                     │                              │
│  │  hvc12 ←→  tcp_connector (nfc) ←→ casimir           │                              │
│  │  hvc13 ←── (vacant / sink)                          │                              │
│  │  hvc14 ←→  MCU (control)                            │                              │
│  │  hvc15 ←→  MCU (uart0)                              │                              │
│  │  hvc16 ←→  ti50_emulator (TPM FIFO)                 │                              │
│  │  hvc17 ←→  (jcardsim, if enabled)                   │                              │
│  │  hvc18 ←→  sensors_simulator (control)              │                              │
│  │  hvc19 ←→  sensors_simulator (data)                 │                              │
│  └──────────────────────────────────────────────────────┘                              │
│                                                                                       │
│  ┌──────────── Vsock ─────────────────────────────────┐                               │
│  │  modem_simulator     (RIL AT commands)             │                               │
│  │  tombstone_receiver  (crash dump upload)           │                               │
│  │  vhal_proxy_server   (Vehicle HAL)                 │                               │
│  │  socket_vsock_proxy  (root-canal HCI/test/link)    │                               │
│  └────────────────────────────────────────────────────┘                               │
│                                                                                       │
│  ┌──────────── Vhost-User ────────────────────────────┐                               │
│  │  vhost_device_vsock     (vsock transport layer)    │──┐                            │
│  │  vhost_user_input (×N)  (touch/kb/mouse/rotary)    │  │ unix sockets              │
│  │  wmediumd               (mac80211-hwsim WiFi)      │  │ to crosvm                 │
│  │  vhost_user_gpu         (optional GPU offload)     │  │                            │
│  └────────────────────────────────────────────────────┘  │                            │
│                                        ┌─────────────────┘                            │
│  ┌─────────────────────────────────────▼──────────────────────────┐                   │
│  │                         crosvm run                             │                   │
│  │  --mem=N --cpus=N --bios=bootloader                           │                   │
│  │  --serial/--hvc (×20)   --vsock=cid=N                         │                   │
│  │  --vhost-user=vsock,input,mac80211-hwsim,...                  │                   │
│  │  --gpu=...  --tap-name=...  --pmem=...  --pflash=...          │                   │
│  │  --socket=crosvm_control.sock                                 │                   │
│  └───────────────────────────────────────────────────────────────┘                    │
│                                                                                       │
│  ┌──────────── gRPC Control Plane (Unix Domain Sockets) ─────────┐                   │
│  │  echo_server             gnss_grpc_proxy                      │                   │
│  │  casimir_control_server  openwrt_control_server               │                   │
│  │  control_env_proxy_server  screen_recording_server            │                   │
│  │  wmediumd (API)                                               │                   │
│  └───────────────────────────────────────────────────────────────┘                    │
│                                                                                       │
└───────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Crosvm and the VM Manager

### Class Hierarchy

The VM manager abstraction is defined in
[vm_manager.h](../base/cvd/cuttlefish/host/libs/vm_manager/vm_manager.h):

```
VmManager (abstract)
 ├── CrosvmManager    — Production VM manager using crosvm
 └── QemuManager      — Alternative using QEMU (limited support)
```

`VmManager` defines three key responsibilities:

| Method                   | Purpose                                                       |
| ------------------------ | ------------------------------------------------------------- |
| `ConfigureGraphics()`    | Returns bootconfig key-value pairs for GPU mode               |
| `ConfigureBootDevices()` | Returns PCI path bootconfig for disk enumeration              |
| `StartCommands()`        | Builds and returns the crosvm CLI as `MonitorCommand` objects |

### Crosvm Launch Sequence

`CrosvmManager::StartCommands()` in
[crosvm_manager.cpp](../base/cvd/cuttlefish/host/libs/vm_manager/crosvm_manager.cpp)
constructs the full crosvm command line. The sequence is:

1. **Wait for VmmDependencyCommands** — A prerequisite closure iterates all
   `VmmDependencyCommand` instances (vhost_device_vsock, wmediumd, MCU,
   ti50_emulator, cvdalloc) and calls `WaitForAvailability()` on each.

2. **Apply process_restarter** — Wraps the crosvm binary in `process_restarter`
   with exit code 32 (`kCrosvmVmResetExitCode`) so guest-initiated reboots
   automatically restart crosvm. The `--first_time_argument` flag ensures
   snapshot restore (`--restore=`) is only passed on the first invocation.

3. **Configure core resources**:
   - `/dev/kvm` (or custom KVM path)
   - CPU count, memory, SMT, MTE settings
   - Virtual disks (up to `kMaxDisks = 3`)
   - Control socket at `crosvm_control.sock`

4. **Configure vhost-user devices** (sockets to external processes):
   - `--vhost-user=vsock,socket=...` (vhost_device_vsock)
   - `--vhost-user=input,socket=...` (×N for each input device)
   - `--vhost-user=mac80211-hwsim,socket=...` (wmediumd)
   - `--vhost-user=gpu,socket=...` (optional vhost-user GPU)
   - `--vhost-user=block,socket=...` (optional vhost-user block)

5. **Configure 20 HVC virtio-console ports** — Each port is either a
   read/write FIFO pair, a read-only FIFO, a socket, or a sink (no-op).
   The port assignments are fixed and documented in
   [vm_manager.h](../base/cvd/cuttlefish/host/libs/vm_manager/vm_manager.h#L60-L81).
   The total count of HVC ports + disk devices must equal
   `kDefaultNumHvcs + kMaxDisks = 23` to maintain stable PCI device numbering
   (required by SEPolicy).

6. **Configure networking**:
   - TAP devices for mobile, ethernet, and optionally WiFi
   - PCI addresses are hardcoded to prevent ordering issues

7. **Configure GPU** — Either inline (`--gpu=...`) or via vhost-user GPU
   device, with Wayland socket for frame delivery to the streamer.

8. **Configure storage** — PMem for hwcomposer, access_kregistry, pstore;
   pflash for x86_64 UEFI; virtiofs shared directory.

9. **Log capture** — Creates a FIFO at `crosvm.fifo` and a `log_tee` process
   to capture crosvm's stdout/stderr into the launcher log.

10. **Emit commands** — Returns a vector of `MonitorCommand` objects. The main
    crosvm command has `is_critical=true`; log_tee and vhost-user GPU processes
    are non-critical.

### Host Resources Consumed by Crosvm

| Resource           | Path / Details                                |
| ------------------ | --------------------------------------------- |
| KVM device         | `/dev/kvm` (or `config.kvm_path()`)           |
| Control socket     | `<instance>/crosvm_control.sock`              |
| TAP devices        | `cvd-mtap-NN`, `cvd-etap-NN`, `cvd-wtap-NN`   |
| HVC FIFOs          | 20 pairs in `<instance>/internal/`            |
| Disk images        | Up to 3 virtio-block devices                  |
| PMem images        | `access-kregistry`, `pstore`, hwcomposer pmem |
| Wayland socket     | `<instance>/frames.sock`                      |
| Audio socket       | `<instance>/audio_server.sock`                |
| Vhost-user sockets | vsock, input (×N), mac80211-hwsim, GPU, block |
| Log FIFO           | `<instance>/internal/crosvm.fifo`             |

---

## Device Catalog

The device catalog is split into per-device documents organized by
communication pattern:

| Communication Pattern             | Devices                                                                                                                                        | Document                                               |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| Virtio-Console (HVC FIFO)         | SecureEnv, BluetoothConnector, NfcConnector, UwbConnector, GnssGrpcProxy, SensorsSimulator, ConsoleForwarder, KernelLogMonitor, LogcatReceiver | [devices_hvc.md](devices_hvc.md)                       |
| Vsock                             | ModemSimulator, TombstoneReceiver, VhalProxyServer                                                                                             | [devices_vsock.md](devices_vsock.md)                   |
| Vsock + TCP Proxies               | RootCanal, NetsimServer                                                                                                                        | [devices_vsock_proxy.md](devices_vsock_proxy.md)       |
| Vhost-User                        | VhostDeviceVsock, VhostInputDevices, WmediumdServer                                                                                            | [devices_vhost_user.md](devices_vhost_user.md)         |
| Unix Socket / gRPC                | Casimir, CasimirControlServer, EchoServer, ControlEnvProxyServer, ScreenRecordingServer, OpenwrtControlServer                                  | [devices_grpc.md](devices_grpc.md)                     |
| VMM Dependency                    | VhostDeviceVsock, WmediumdServer, MCU, Ti50Emulator, Cvdalloc                                                                                  | [devices_vmm_dependency.md](devices_vmm_dependency.md) |
| Standalone / No VMM Communication | Metrics, AutomotiveProxy, Pica                                                                                                                 | [devices_standalone.md](devices_standalone.md)         |
| Streaming                         | WebRTC / Streamer                                                                                                                              | [devices_streaming.md](devices_streaming.md)           |
| Secondary VMs                     | OpenWrt                                                                                                                                        | [devices_secondary_vm.md](devices_secondary_vm.md)     |

### Quick Reference Table

| #   | Device                | Binary                                 | Channel              | AutoCmd? | VmmDep? | process_restarter? | log_tee? | gRPC? |
| --- | --------------------- | -------------------------------------- | -------------------- | -------- | ------- | ------------------ | -------- | ----- |
| 1   | SecureEnv             | `secure_env`                           | HVC 3,4,8,10,11      | ✅       |         |                    |          |       |
| 2   | BluetoothConnector    | `tcp_connector`                        | HVC 5 → TCP          | ✅       |         |                    |          |       |
| 3   | NfcConnector          | `tcp_connector`                        | HVC 12 → TCP         | ✅       |         |                    |          |       |
| 4   | UwbConnector          | `tcp_connector`                        | HVC 9 → TCP          | ✅       |         |                    |          |       |
| 5   | GnssGrpcProxy         | `gnss_grpc_proxy`                      | HVC 6,7 + gRPC       | ✅       |         |                    |          | ✅    |
| 6   | SensorsSimulator      | `sensors_simulator`                    | HVC 18,19            | ✅       |         |                    |          |       |
| 7   | ConsoleForwarder      | `console_forwarder`                    | HVC 1                | ✅       |         |                    |          |       |
| 8   | KernelLogMonitor      | `kernel_log_monitor`                   | HVC 0                |          |         |                    |          |       |
| 9   | LogcatReceiver        | `logcat_receiver`                      | HVC 2                | ✅       |         |                    |          |       |
| 10  | ModemSimulator        | `modem_simulator`                      | vsock                | ✅       |         |                    |          |       |
| 11  | TombstoneReceiver     | `tombstone_receiver`                   | vsock                | ✅       |         |                    |          |       |
| 12  | VhalProxyServer       | `vhal_proxy_server`                    | vsock                | ✅       |         |                    |          |       |
| 13  | RootCanal             | `root-canal` + `socket_vsock_proxy` ×4 | vsock→TCP            |          |         | ✅                 | ✅       |       |
| 14  | NetsimServer          | `netsimd` + `socket_vsock_proxy` ×2    | vsock→TCP + FIFO     |          |         |                    |          |       |
| 15  | VhostDeviceVsock      | `vhost_device_vsock`                   | vhost-user           |          | ✅      | ✅                 | ✅       |       |
| 16  | VhostInputDevices     | `vhost_user_input` ×N                  | vhost-user           |          |         |                    | ✅       |       |
| 17  | WmediumdServer        | `wmediumd`                             | vhost-user mac80211  |          | ✅      |                    | ✅       | ✅    |
| 18  | Casimir               | `casimir`                              | unix socket          | ✅       |         | ✅                 | ✅       |       |
| 19  | CasimirControlServer  | `casimir_control_server`               | gRPC UDS             | ✅       |         |                    |          | ✅    |
| 20  | EchoServer            | `echo_server`                          | gRPC UDS             | ✅       |         |                    |          | ✅    |
| 21  | ControlEnvProxyServer | `control_env_proxy_server`             | gRPC UDS             |          |         |                    |          | ✅    |
| 22  | ScreenRecordingServer | `screen_recording_server`              | gRPC UDS             | ✅       |         |                    |          | ✅    |
| 23  | OpenwrtControlServer  | `openwrt_control_server`               | gRPC UDS             |          |         |                    |          | ✅    |
| 24  | MCU                   | configurable                           | unix socket          |          | ✅      |                    | ✅       |       |
| 25  | Ti50Emulator          | `ti50_emulator`                        | unix socket + HVC 16 |          | ✅      |                    | ✅       |       |
| 26  | Cvdalloc              | `cvdalloc`                             | unix socket pair     |          | ✅      |                    |          |       |
| 27  | WebRTC                | `webrtc`                               | frames/audio sockets |          |         |                    |          |       |
| 28  | OpenWrt               | crosvm (2nd VM)                        | TAP + vhost-user     |          |         | ✅                 | ✅       |       |
| 29  | Metrics               | `metrics`                              | none                 | ✅       |         |                    |          |       |
| 30  | AutomotiveProxy       | `automotive_proxy`                     | none                 | ✅       |         |                    |          |       |
| 31  | Pica                  | `pica`                                 | TCP                  | ✅       |         |                    | ✅       |       |
