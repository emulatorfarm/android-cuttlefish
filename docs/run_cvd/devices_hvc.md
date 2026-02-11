# HVC (Virtio-Console) Devices

These devices communicate with the Android guest via named FIFOs connected to
virtio-console (HVC) ports. Crosvm maps each FIFO pair to a `/dev/hvcN` device
inside the guest. The port assignments are fixed in
[vm_manager.h](../../base/cvd/cuttlefish/host/libs/vm_manager/vm_manager.h)
to maintain stable PCI device numbering required by SEPolicy.

---

## SecureEnv

**Binary**: `secure_env`
**Source**: [secure_env.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/secure_env.cpp)
**Registration**: `AutoCmd<SecureEnv>::Component`

### Communication Channels

| HVC Port | FIFO Names                    | HAL             |
| -------- | ----------------------------- | --------------- |
| hvc3     | `keymaster_fifo_vm.{in,out}`  | Keymaster (C++) |
| hvc4     | `gatekeeper_fifo_vm.{in,out}` | Gatekeeper      |
| hvc8     | `confui_fifo_vm.{in,out}`     | ConfirmationUI  |
| hvc10    | `oemlock_fifo_vm.{in,out}`    | OEMLock         |
| hvc11    | `keymint_fifo_vm.{in,out}`    | KeyMint (Rust)  |

### Description

SecureEnv is the security HAL multiplexer. It provides software implementations
of five Android security HALs, each communicating over a dedicated HVC FIFO
pair. The guest writes HAL requests to the `.in` FIFO and reads responses from
the `.out` FIFO.

### Dependencies

- `SnapshotControlFiles` — for snapshot control socket pair
- `GrpcSocketCreator` — for gRPC socket paths
- `KernelLogPipeProvider` — consumes kernel log pipe
- `LogTeeCreator` — (available, but not currently used)

### Host Resources

- ConfirmationUI unix socket (`confui_fifo_vm.{in,out}`)
- Snapshot control socket pair for freeze/restore coordination
- TPM state files (when using software TPM)

### Direction

Bidirectional — guest sends HAL requests, host sends responses.

### Configuration

- `--tpm_impl` flag selects TPM backend (software or ti50)
- `--keymint_impl` selects keymint backend (software, tpm, or rust-tpm)

---

## BluetoothConnector

**Binary**: `tcp_connector`
**Source**: [bluetooth_connector.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/bluetooth_connector.cpp)
**Registration**: `AutoCmd<BluetoothConnector>::Component`

### Communication Channels

| HVC Port | FIFO Names            | Bridge Target              |
| -------- | --------------------- | -------------------------- |
| hvc5     | `bt_fifo_vm.{in,out}` | TCP → `rootcanal_hci_port` |

### Description

Bridges the Bluetooth HVC FIFO to a TCP connection targeting the RootCanal or
Netsim Bluetooth emulator. Uses the `tcp_connector` binary which reads from the
guest-facing FIFO and forwards to/from a TCP socket.

### Dependencies

- `CuttlefishConfig` and `InstanceSpecific` for port configuration

### Direction

Bidirectional — guest Bluetooth HAL ↔ FIFO ↔ TCP ↔ root-canal/netsim.

### Condition

Only launched when `instance.has_bluetooth()` is true.

---

## NfcConnector

**Binary**: `tcp_connector`
**Source**: [nfc_connector.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/nfc_connector.cpp)
**Registration**: `AutoCmd<NfcConnector>::Component`

### Communication Channels

| HVC Port | FIFO Names             | Bridge Target                           |
| -------- | ---------------------- | --------------------------------------- |
| hvc12    | `nfc_fifo_vm.{in,out}` | Unix socket → `casimir_nci_socket_path` |

### Description

Bridges the NFC HVC FIFO to a unix socket connection targeting the Casimir NFC
emulator. Structurally identical to BluetoothConnector but for NFC NCI protocol.

### Dependencies

- `CuttlefishConfig` and `InstanceSpecific`

### Direction

Bidirectional — guest NFC HAL ↔ FIFO ↔ unix socket ↔ casimir.

### Condition

Only launched when `config.enable_host_nfc()` is true.

---

## UwbConnector

**Binary**: `tcp_connector`
**Source**: [uwb_connector.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/uwb_connector.cpp)
**Registration**: `AutoCmd<UwbConnector>::Component`

### Communication Channels

| HVC Port | FIFO Names             | Bridge Target         |
| -------- | ---------------------- | --------------------- |
| hvc9     | `uwb_fifo_vm.{in,out}` | TCP → `pica_uci_port` |

### Description

Bridges the UWB HVC FIFO to a TCP connection targeting the Pica UWB emulator.
Structurally identical to BluetoothConnector but for UWB UCI protocol.

### Dependencies

- `CuttlefishConfig` and `InstanceSpecific`

### Direction

Bidirectional — guest UWB HAL ↔ FIFO ↔ TCP ↔ pica.

### Condition

Only launched when `config.enable_host_uwb()` is true.

---

## GnssGrpcProxy

**Binary**: `gnss_grpc_proxy`
**Source**: [gnss_grpc_proxy.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/gnss_grpc_proxy.cpp)
**Registration**: `AutoCmd<GnssGrpcProxyServer>::Component`

### Communication Channels

| HVC Port | FIFO Names                     | Purpose       |
| -------- | ------------------------------ | ------------- |
| hvc6     | `gnsshvc_fifo_vm.{in,out}`     | GNSS data     |
| hvc7     | `locationhvc_fifo_vm.{in,out}` | Location data |

Additionally exposes a gRPC endpoint on a Unix Domain Socket for external
control (injecting GNSS fixes from test automation), and optionally a TCP port
(`gnss_grpc_proxy_server_port`).

### Dependencies

- `CuttlefishConfig`, `InstanceSpecific`, `GrpcSocketCreator`

### Direction

Bidirectional — GNSS data flows guest ↔ host; external gRPC clients inject
location fixes host → guest.

### Condition

Only launched when `instance.enable_gnss_grpc_proxy()` is true.

### Observability

- gRPC UDS: `grpc_socket_path("GnssGrpcProxyServer")`
- Optional TCP port: `gnss_grpc_proxy_server_port`

---

## SensorsSimulator

**Binary**: `sensors_simulator`
**Source**: [sensors_simulator.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/sensors_simulator.cpp)
**Registration**: `AutoCmd<SensorsSimulator>::Component`

### Communication Channels

| HVC Port | FIFO Names                         | Purpose                 |
| -------- | ---------------------------------- | ----------------------- |
| hvc18    | `sensors_control_fifo_vm.{in,out}` | Sensor control commands |
| hvc19    | `sensors_data_fifo_vm.{in,out}`    | Sensor data stream      |

Also uses a unix socket pair (from `SensorsSocketPair`) to exchange sensor
data with the WebRTC streamer, allowing browser-based sensor injection.

### Dependencies

- `CuttlefishConfig`, `InstanceSpecific`
- `SensorsSocketPair` — provides socket FD to/from WebRTC
- `KernelLogPipeProvider` — consumes kernel log pipe

### Direction

Bidirectional — sensor data flows guest ↔ host; WebRTC browser can inject
sensor events.

---

## ConsoleForwarder

**Binary**: `console_forwarder`
**Source**: [console_forwarder.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/console_forwarder.cpp)
**Registration**: `AutoCmd<ConsoleForwarder>::Component`

### Communication Channels

| HVC Port | FIFO Names               | Purpose                    |
| -------- | ------------------------ | -------------------------- |
| hvc1     | `console_{in,out}` pipes | Interactive serial console |

### Description

Forwards the guest's serial console (hvc1) to a host-side PTY that can be
attached to via `screen`. When `kgdb` or `use_bootloader` is enabled, the
serial console is instead routed through an 8250 UART (ISA serial port) for
early boot access, and hvc1 becomes a sink.

### Dependencies

- `CuttlefishConfig::InstanceSpecific`

### Diagnostics

The companion `ConsoleInfo` diagnostic (registered via `AutoDiagnostic`)
prints: `"To access the console run: screen <path>"`

### Direction

Bidirectional — user input → guest; guest output → terminal.

### Condition

Only launched when `instance.console()` is true.

---

## KernelLogMonitor

**Binary**: `kernel_log_monitor`
**Source**: [kernel_log_monitor.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/kernel_log_monitor.cpp)
**Registration**: `KernelLogMonitorComponent` (manual Fruit component)

### Communication Channels

| HVC Port | FIFO Name         | Purpose               |
| -------- | ----------------- | --------------------- |
| hvc0     | `kernel-log-pipe` | Kernel console output |

### Description

Reads the kernel console output from hvc0 and writes it to `kernel.log`. It
also implements `KernelLogPipeProvider`, distributing kernel log pipe FDs to
multiple consumers (WebRTC streamer, SensorsSimulator, SecureEnv) via the
`KernelLogPipeConsumer` interface.

### Interfaces Implemented

- `CommandSource` — provides the `kernel_log_monitor` command
- `KernelLogPipeProvider` — distributes kernel log FDs
- `DiagnosticInformation` — reports kernel log file path

### Dependencies

- `CuttlefishConfig::InstanceSpecific`

### Direction

Guest → Host (unidirectional kernel log stream).

### Diagnostics

Reports: `"Kernel log: <instance>/kernel.log"`

---

## LogcatReceiver

**Binary**: `logcat_receiver`
**Source**: [logcat_receiver.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/logcat_receiver.cpp)
**Registration**: `AutoCmd<LogcatReceiver>::Component`

### Communication Channels

| HVC Port | FIFO Name     | Purpose               |
| -------- | ------------- | --------------------- |
| hvc2     | `logcat-pipe` | Android logcat stream |

### Description

Receives the Android logcat stream from hvc2 and writes it to
`logcat-pipe` on the host. Simple unidirectional receiver.

### Dependencies

- `CuttlefishConfig::InstanceSpecific`

### Diagnostics

The companion `LogcatInfo` diagnostic (registered via `AutoDiagnostic`)
reports: `"Logcat output: <instance>/logcat"`

### Direction

Guest → Host (unidirectional logcat stream).
