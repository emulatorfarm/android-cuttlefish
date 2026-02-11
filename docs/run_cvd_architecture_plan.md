## Plan: Document run_cvd Virtual Device Architecture

The deliverable is a set of Markdown documents answering the five questions about the crosvm-based Cuttlefish virtual device architecture. The document will synthesize findings from the `run_cvd` orchestration layer, ~35 device launchers, the crosvm VM manager integration, and the Fruit DI framework.

### Steps

1. **Create the document folder and documents** at a location like [docs/run_cvd/virtual_device_architecture.md](docs/virtual_device_architecture.md) with the five sections corresponding to the user's questions.

2. **Section 1 — Crosvm overview**: Document how `run_cvd` launches crosvm via `process_restarter` (auto-restart on exit code 32 for guest reboots), the full CLI argument construction in [crosvm_manager.cpp](base/cvd/cuttlefish/host/libs/vm_manager/crosvm_manager.cpp), host resource consumption (`/dev/kvm`, TAP devices, control socket at `crosvm_control.sock`, 20 HVC FIFOs, disk images, PMem/PStore, Wayland socket, audio socket), and the `VmManager` → `CrosvmManager` class hierarchy defined in [vm_manager.h](base/cvd/cuttlefish/host/libs/vm_manager/vm_manager.h).

3. **Section 2 & 3 — Virtual device catalog**: Create a comprehensive table for all ~30 device launchers in [launch/](base/cvd/cuttlefish/host/commands/run_cvd/launch/), organized by communication pattern:
   - **Virtio-console (HVC FIFO)** devices: `SecureEnv` (5 HAL pairs), `BluetoothConnector`, `NfcConnector`, `UwbConnector`, `GnssGrpcProxy`, `SensorsSimulator`, `ConsoleForwarder`, `KernelLogMonitor`, `LogcatReceiver`
   - **Vsock** devices: `ModemSimulator`, `TombstoneReceiver`, `VhalProxyServer`, `RootCanal` (4 vsock proxies), `NetsimServer`
   - **Vhost-user** devices: `VhostDeviceVsock`, `VhostInputDevices` (touch/mouse/keyboard/gamepad/rotary/switches), `WmediumdServer` (mac80211-hwsim)
   - **Unix socket only**: `MCU`, `Ti50Emulator`, `Casimir`, gRPC control servers
   - **No VMM communication**: `Metrics`, `EchoServer`, `AutomotiveProxy`, `ScreenRecordingServer`
   - For each, document the binary, dependencies, host resources, VMM communication channel, directionality, observability (log_tee, diagnostics, gRPC), and whether it's a `VmmDependencyCommand`.

4. **Section 4 — Patterns and inconsistencies**: Document the key architectural patterns:
   - `AutoCmd<Fn>` template in [auto_cmd.h](base/cvd/cuttlefish/host/commands/run_cvd/launch/auto_cmd.h) (zero-boilerplate free-function → `CommandSource`)
   - `SetupFeature` topological sort for dependency ordering in [feature.h](base/cvd/cuttlefish/host/libs/feature/feature.h)
   - `MonitorCommand` with `is_critical` flag in [command_source.h](base/cvd/cuttlefish/host/libs/feature/command_source.h)
   - `VmmDependencyCommand` / `StatusCheckCommandSource` with `WaitForAvailability()` for pre-VMM readiness
   - `process_restarter` wrapper for crash recovery
   - `log_tee` + FIFO for stdout/stderr capture
   - `tcp_connector` for FIFO↔TCP bridging (BT, NFC, UWB)
   - `socket_vsock_proxy` for vsock↔TCP bridging
   - **Inconsistencies**: some devices use `AutoCmd` template while others define manual `Component` functions; log_tee is inconsistently applied; no launcher sets `is_critical=true`; the `tcp_connector` pattern is identical across BT/NFC/UWB but each has its own launcher file; some devices use vsock while others use HVC for similar bidirectional communication.

5. **Section 5 — Extension guide**: Document the step-by-step process to add a new virtual device:
   - Create `launch/new_device.h` and `launch/new_device.cpp` with a free function returning `Result<MonitorCommand>`
   - Register in [main.cc](base/cvd/cuttlefish/host/commands/run_cvd/main.cc) via `.install(AutoCmd<NewDevice>::Component)` (or manual `Component` for `VmmDependencyCommand`)
   - Choose communication pattern: allocate an HVC port in [vm_manager.h](base/cvd/cuttlefish/host/libs/vm_manager/vm_manager.h) (currently port 13 is vacant), use vsock, or implement vhost-user
   - Add config fields to `CuttlefishConfig` if needed
   - Add observability via `LogTeeCreator` and `DiagnosticInformation`
   - Add `process_restarter` wrapper if crash recovery is needed
   - If the device must be ready before VMM, implement `VmmDependencyCommand` with `WaitForAvailability()`

### Further Considerations

1. **Document location**: New documentation for this task should be placed [docs/](docs/) alongside existing docs like [cuttlefish_config_schema.md](docs/cuttlefish_config_schema.md). Use multiple files if needed.
2. **Depth and breadth**: The device catalog has ~30 entries. Every device should get a full subsection. Each device should get a dedicated markdown file, alongside a main document that synthesizes patterns and the extension guide.
3. **Diagrams**: Include new ASCII architecture diagrams (like the process tree and communication topology).
