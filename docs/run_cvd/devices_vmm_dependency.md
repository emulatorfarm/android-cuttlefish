# VMM Dependency Devices

These devices implement the `VmmDependencyCommand` interface (which extends
`StatusCheckCommandSource`). They **must be fully initialized before crosvm
starts**. The crosvm launch sequence includes a prerequisite that calls
`WaitForAvailability()` on every enabled `VmmDependencyCommand`, blocking
the VM start until all dependencies report readiness.

This pattern is used for devices that provide vhost-user backends (which crosvm
connects to at startup) or devices that must initialize hardware state before
the guest boots.

```
CrosvmManager::StartCommands() {
    crosvm_cmd.AddPrerequisite([&dependencyCommands]() {
        for (auto dep : dependencyCommands) {
            if (dep->Enabled()) dep->WaitForAvailability();
        }
    });
    // ... build crosvm command ...
}
```

---

## VhostDeviceVsock

See [devices_vhost_user.md](devices_vhost_user.md#vhostdevicevsock) for full
documentation.

**Summary**: Provides the vsock transport as a vhost-user device.
`WaitForAvailability()` blocks until the vhost.socket file appears (30s timeout).

---

## WmediumdServer

See [devices_vhost_user.md](devices_vhost_user.md#wmediumdserver) for full
documentation.

**Summary**: Simulates the wireless medium for mac80211-hwsim.
`WaitForAvailability()` blocks until the API server socket appears.

---

## MCU (Microcontroller Emulator)

**Binary**: Configurable via JSON config (`instance.mcu()`)
**Source**: [mcu.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/mcu.cpp)
**Registration**: `McuComponent` (manual Fruit component class)

### Communication Channels

| Channel            | Type        | HVC Port               | Purpose               |
| ------------------ | ----------- | ---------------------- | --------------------- |
| MCU control socket | Unix socket | hvc14 (if type=serial) | MCU control interface |
| MCU uart0          | Unix socket | hvc15 (if type=serial) | MCU UART data         |

### Description

Launches a microcontroller emulator binary specified in the instance
configuration JSON. The MCU config specifies the binary path (resolved via
`HostBinaryPath`), working directory, and socket paths. The MCU communicates
with the guest via HVC ports 14 and 15 when configured as serial type.

### Special Features

- **VmmDependencyCommand**: **Yes**
- **process_restarter**: No
- **log_tee**: **Yes** — captures stderr as "mcu"
- **WaitForAvailability()**: Waits for both the control socket file and uart0
  file to appear in the MCU working directory (30-second timeout)

### Dependencies

- `CuttlefishConfig`, `InstanceSpecific`
- `LogTeeCreator`
- `KernelLogPipeProvider`

### Direction

Bidirectional — MCU emulation communicates with guest firmware.

### Condition

Only launched when `instance.mcu()["path"]` is non-empty.

### Platform

Linux-only (`#ifdef __linux__`).

---

## Ti50Emulator (Titan Security Chip)

**Binary**: `ti50_emulator` (path from config)
**Source**: [ti50_emulator.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/ti50_emulator.cpp)
**Registration**: `Ti50EmulatorComponent` (manual Fruit component class)

### Communication Channels

| Channel           | Type        | HVC Port | Purpose             |
| ----------------- | ----------- | -------- | ------------------- |
| `control_sock`    | Unix socket | —        | Ti50 control/status |
| `gpioPltRst`      | Unix socket | —        | GPIO platform reset |
| `direct_tpm_fifo` | Unix socket | hvc16    | TPM FIFO data       |

### Description

Emulates Google's Titan security chip (Ti50/CR50), providing TPM functionality
to the guest. The TPM FIFO is connected to hvc16, allowing the guest's TPM
driver to communicate with the emulator.

### Special Features

- **VmmDependencyCommand**: **Yes**
- **process_restarter**: No
- **log_tee**: **Yes** — captures stderr as "ti50"
- **WaitForAvailability()**: Complex multi-step initialization:
  1. Wait for `control_sock` to appear (30s timeout)
  2. Connect to control socket, wait for `READY` message
  3. Assert GPIO platform reset via `gpioPltRst` socket
  4. Send TPM2_Startup(SU_CLEAR) command to `direct_tpm_fifo` socket
  5. Validate TPM response (up to 5 retries with 1s timeout each)

### Dependencies

- `CuttlefishConfig`, `InstanceSpecific`
- `LogTeeCreator`

### Direction

Bidirectional — guest TPM driver ↔ Ti50 emulator.

### Condition

Only launched when `instance.ti50_emulator()` path is non-empty.

### Platform

Linux-only (`#ifdef __linux__`).

---

## Cvdalloc (Resource Allocator)

**Binary**: `cvdalloc`
**Source**: [cvdalloc.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/cvdalloc.cpp)
**Registration**: `CvdallocComponent` (manual Fruit component class)

### Communication Channels

| Channel          | Type       | Purpose                                 |
| ---------------- | ---------- | --------------------------------------- |
| Unix socket pair | socketpair | Semaphore protocol (ACQUIRE/RELEASE/OK) |

### Description

Manages host resource allocation for the virtual device instance. Uses a
socket-pair-based protocol where `run_cvd` sends `ACQUIRE` and the allocator
responds with `OK` after resources are secured. This ensures no resource
conflicts between multiple concurrent instances.

### Special Features

- **VmmDependencyCommand**: **Yes**
- **process_restarter**: No
- **log_tee**: No
- **WaitForAvailability()**: Sends "ACQUIRE" on the socket pair and waits for
  "OK" response (30-second timeout)
- **Custom stopper**: Sends "RELEASE" on teardown, waits for "OK", then
  terminates the process
- **Privilege check**: Validates the binary has the correct capabilities

### Dependencies

- `CuttlefishConfig::InstanceSpecific`

### Direction

Host-only (resource coordination, no guest communication).

### Condition

Only launched when `instance.enable_cvdalloc()` is true.
