# Architectural Patterns and Inconsistencies

This document catalogs the recurring patterns used across the ~30 device
launchers in `run_cvd`, as well as notable inconsistencies and technical debt.

---

## Patterns

### 1. AutoCmd Template

**File**: [auto_cmd.h](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/auto_cmd.h)

The `AutoCmd<Fn>` template provides zero-boilerplate registration of device
launchers. Given a free function `Fn` with signature:

```cpp
Result<MonitorCommand> MyDevice(const CuttlefishConfig& config,
                                 const CuttlefishConfig::InstanceSpecific& instance,
                                 LogTeeCreator& log_tee);
```

The template generates:

1. A `CommandSource` subclass (`GenericCommandSource<Fn, R, Args...>`) that:
   - Calls `Fn` during `ResultSetup()` to produce `MonitorCommand`s
   - Derives its `Name()` from the function pointer using `ValueName<Fn>()`
   - Automatically computes `Dependencies()` by scanning the tuple of arguments
     for types that inherit from `SetupFeature`

2. A `Component()` function that registers the generated class as both a
   `CommandSource` and `SetupFeature` multibinding in the Fruit DI container

**Supported return types**:

- `Result<MonitorCommand>` — exactly one command
- `Result<std::vector<MonitorCommand>>` — multiple commands
- `Result<std::optional<MonitorCommand>>` — conditionally no command
- Non-Result variants of all the above

**Usage in main.cc**:

```cpp
.install(AutoCmd<SecureEnv>::Component)
.install(AutoCmd<BluetoothConnector>::Component)
// ... etc
```

**Companion templates**:

- `AutoDiagnostic<Fn>` — same pattern for `DiagnosticInformation` providers
- `AutoSetup<Fn>` — same pattern for `SetupFeature` (no commands)
- `AutoSnapshotControlFiles` — for `ReturningSetupFeature` (computes a value)

**Devices using AutoCmd**: SecureEnv, BluetoothConnector, NfcConnector,
UwbConnector, GnssGrpcProxy, SensorsSimulator, ConsoleForwarder,
LogcatReceiver, ModemSimulator, TombstoneReceiver, VhalProxyServer,
Casimir, CasimirControlServer, EchoServer, ScreenRecordingServer,
MetricsService, AutomotiveProxy, Pica (18 of ~31 launchers).

---

### 2. SetupFeature Topological Sort

**File**: [feature.h](../../base/cvd/cuttlefish/host/libs/feature/feature.h)

All device launchers implement `SetupFeature` (via `CommandSource`). Before
any processes are started, `SetupFeature::RunSetup()` performs a topological
sort of all features based on their `Dependencies()`:

```
Feature<Subclass>::TopologicalVisit(features, callback)
    for each feature:
        if UNVISITED:
            mark VISITING
            recursively visit dependencies
            mark VISITED
            call callback(feature)  // executes ResultSetup()
```

This ensures that dependency features (e.g., `LogTeeCreator`,
`KernelLogPipeProvider`, `SensorsSocketPair`) are set up before the features
that depend on them.

**Cycle detection**: The algorithm detects dependency cycles and reports them
with the feature name.

**Enabled check**: Features can override `Enabled()` to conditionally skip
setup without removing them from the graph.

---

### 3. MonitorCommand and is_critical

**File**: [command_source.h](../../base/cvd/cuttlefish/host/libs/feature/command_source.h)

```cpp
struct MonitorCommand {
    Command command;
    bool is_critical;  // default: false
};
```

The `is_critical` flag determines behavior when a process exits:

- `is_critical = true` — the entire instance is shut down (used for crosvm)
- `is_critical = false` — the process may be restarted or ignored

**Notable**: Only the main crosvm command sets `is_critical = true` (in
`CrosvmManager::StartCommands()`). No device launcher sets this flag.

---

### 4. VmmDependencyCommand with WaitForAvailability

**File**: [vm_manager.h](../../base/cvd/cuttlefish/host/libs/vm_manager/vm_manager.h)

```cpp
class VmmDependencyCommand : public virtual StatusCheckCommandSource {};

class StatusCheckCommandSource : public virtual CommandSource {
    virtual Result<void> WaitForAvailability() = 0;
};
```

Devices implementing `VmmDependencyCommand` are collected by the VM manager
and their `WaitForAvailability()` methods are called as a prerequisite before
crosvm starts:

```
run_cvd starts all processes
    ├── VmmDependencyCommands start and initialize
    │   ├── vhost_device_vsock: waits for socket file
    │   ├── wmediumd: waits for API socket
    │   ├── MCU: waits for control + uart0 files
    │   ├── ti50_emulator: waits for READY + TPM handshake
    │   └── cvdalloc: waits for ACQUIRE/OK protocol
    │
    └── crosvm prerequisite blocks until all VmmDeps are ready
        then crosvm starts
```

**Devices using VmmDependencyCommand**: VhostDeviceVsock, WmediumdServer,
MCU, Ti50Emulator, Cvdalloc (5 devices).

---

### 5. process_restarter

**Binary**: `process_restarter`

Wraps a child binary to automatically restart it on certain exit conditions.
Used for crash recovery of long-running processes.

**Configuration flags**:

- `-when_killed` — restart on signal death
- `-when_dumped` — restart on core dump
- `-when_exited_with_failure` — restart on non-zero exit
- `-first_time_argument <arg>` — extra argument only on first invocation
  (used for snapshot `--restore=` flag)
- Exit code trigger — restart on specific exit code (e.g., 32 for crosvm
  guest reboot)

**Devices using process_restarter**: crosvm (main VM), crosvm (OpenWrt),
VhostDeviceVsock, Casimir, RootCanal, vhost-user GPU (6 uses).

---

### 6. log_tee for Output Capture

**Binary**: `log_tee`
**Helper**: [log_tee_creator.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/log_tee_creator.cpp)

Captures a process's stdout/stderr via a FIFO pipe and writes it to the
launcher log. Pattern:

1. Create a FIFO at `<instance>/internal/<name>.fifo`
2. Redirect the target process's stdout/stderr to the FIFO
3. Launch `log_tee --process_name=<name> --log_fd_in=<fifo_fd>`

`LogTeeCreator` is a `SetupFeature` that provides the `CreateLogTee()` method,
used by device launchers that need log capture.

**Custom stopper**: log_tee processes use `SIGINT` for graceful shutdown
(to flush buffered logs) rather than the default `SIGKILL`.

**Devices using log_tee**: crosvm, RootCanal, VhostDeviceVsock,
VhostInputDevices (per device), WmediumdServer, MCU, Ti50Emulator, Casimir,
Pica, OpenWrt, vhost-user GPU (11+ processes).

---

### 7. tcp_connector for FIFO↔TCP Bridging

**Binary**: `tcp_connector`

Bridges a FIFO pair (connected to an HVC port) to a TCP or Unix socket
connection. Used for Bluetooth (hvc5 → root-canal), NFC (hvc12 → casimir),
and UWB (hvc9 → pica).

**Configuration**:

```
tcp_connector --fifo_out=<guest_read> --fifo_in=<guest_write>
              --target_host=<host> --target_port=<port>
```

---

### 8. socket_vsock_proxy for Vsock↔TCP Bridging

**Binary**: `socket_vsock_proxy`

Bridges a vsock connection to a TCP socket. Used by RootCanal (4 proxies)
and NetsimServer (2 proxies) to expose their TCP services to the guest via
vsock.

**Configuration**:

```
socket_vsock_proxy --server=tcp --tcp_port=<port>
                   --vsock_port=<port> --vsock_cid=<cid>
```

---

### 9. Fruit Dependency Injection

**Framework**: [Google Fruit](https://github.com/nicoa/fruit)

The entire `run_cvd` architecture uses Fruit for compile-time dependency
injection. Key concepts:

- **Component**: A factory function returning `fruit::Component<...>` that
  declares provided types, required types, and multibindings
- **Multibinding**: All `CommandSource`, `SetupFeature`, and
  `DiagnosticInformation` instances are registered as multibindings
- **LateInjected**: A special interface for objects that need access to the
  full injector after construction (used by `InstanceLifecycle`)
- **Install order**: The order of `.install()` calls in `runCvdComponent()`
  affects process start/stop order. Notably, vhost-user devices must be
  installed (and thus stopped) after the VMM.

---

## Inconsistencies

### 1. AutoCmd vs. Manual Component

About 18 launchers use the `AutoCmd` template, while ~13 use manually-defined
Fruit `Component` functions. The manual approach is required when:

- The launcher implements `VmmDependencyCommand` (AutoCmd only generates
  `CommandSource`)
- The launcher implements additional interfaces (e.g., `KernelLogPipeProvider`,
  `InputConnectionsProvider`)
- The launcher needs complex Fruit wiring (multiple multibindings, custom
  lifetimes)

However, some manual components (e.g., `ControlEnvProxyServer`,
`OpenwrtControlServer`) don't appear to need any of these features and could
potentially be simplified to `AutoCmd`.

### 2. Inconsistent log_tee Usage

Some devices that launch long-running processes don't use log_tee for output
capture:

| Device          | Has log_tee? | Should it?   |
| --------------- | ------------ | ------------ |
| RootCanal       | ✅ Yes       | —            |
| NetsimServer    | ❌ No        | Probably yes |
| ModemSimulator  | ❌ No        | Probably yes |
| WebRTC Streamer | ❌ No        | Probably yes |
| Cvdalloc        | ❌ No        | Maybe        |
| Metrics         | ❌ No        | Low priority |

This means that stderr from some processes is silently discarded or goes to
the parent's stderr without proper log formatting.

### 3. No Launcher Sets is_critical=true

Despite the `is_critical` flag being available on `MonitorCommand`, no device
launcher uses it. Only the main crosvm command in `CrosvmManager::StartCommands()`
sets `is_critical = true`. This means that if a critical support process
(e.g., vhost_device_vsock) crashes and can't restart, the VM continues running
in a degraded state rather than shutting down cleanly.

### 4. Duplicated tcp_connector Pattern

The BluetoothConnector, NfcConnector, and UwbConnector launchers are
structurally identical — they each create a `tcp_connector` command with
different FIFO paths and target addresses. These could potentially be unified
into a single parameterized launcher or a shared factory function.

```
BluetoothConnector: tcp_connector --fifo=bt_fifo_vm    --target=rootcanal_hci_port
NfcConnector:       tcp_connector --fifo=nfc_fifo_vm   --target=casimir_nci_socket
UwbConnector:       tcp_connector --fifo=uwb_fifo_vm   --target=pica_uci_port
```

### 5. Mixed Communication Patterns for Similar Use Cases

Devices with similar communication needs use different transport mechanisms:

| Device          | Transport                 | Use Case                   |
| --------------- | ------------------------- | -------------------------- |
| ModemSimulator  | Vsock (direct)            | Bidirectional HAL protocol |
| RootCanal       | TCP → vsock proxy         | Bidirectional HAL protocol |
| SecureEnv       | HVC FIFO (direct)         | Bidirectional HAL protocol |
| VhalProxyServer | Vsock or vhost-user vsock | Bidirectional HAL protocol |

These choices are historical and optimized for each device's specific needs,
but the inconsistency makes it harder to reason about the overall architecture.

### 6. Install Order Sensitivity

The comment in `main.cc` warns:

> WARNING: The install order indirectly controls the order that processes
> are started and stopped. The start order shouldn't matter, but if the stop
> order is incorrect, then some processes may crash on shutdown.

This is fragile — the process stop order depends on the order of `.install()`
calls in a function that looks like a flat list. There is no explicit
annotation or validation that vhost-user devices are stopped after the VMM.

### 7. Linux-Only Guards

Several devices use `#ifdef __linux__` guards in `main.cc` but their launcher
files don't have corresponding guards, leading to compilation on non-Linux
platforms of code that will never be registered. The affected devices are:
AutomotiveProxy, ModemSimulator, TombstoneReceiver, MCU, VhostDeviceVsock,
VhostInputDevices, WmediumdServer, Streamer, VhalProxyServer, Ti50Emulator,
OpenWrt.
