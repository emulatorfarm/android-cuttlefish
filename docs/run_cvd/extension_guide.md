# Extension Guide: Adding a New Virtual Device

This guide walks through the step-by-step process of adding a new virtual
device to the Cuttlefish `run_cvd` architecture.

---

## Prerequisites

Before starting, you need:

1. A host binary that implements your device emulation
2. A decision on the communication pattern (HVC FIFO, vsock, vhost-user,
   Unix socket, or no VMM communication)
3. Knowledge of whether the device must be ready before the VMM starts

---

## Step 1: Choose a Communication Pattern

| Pattern                | When to Use                                                 | Example                             |
| ---------------------- | ----------------------------------------------------------- | ----------------------------------- |
| **HVC FIFO**           | Bidirectional byte stream with a specific guest `/dev/hvcN` | SecureEnv, SensorsSimulator         |
| **Vsock**              | Socket-based communication (connect/accept)                 | ModemSimulator, TombstoneReceiver   |
| **Vhost-user**         | Native virtio device backend in a separate process          | VhostInputDevices, VhostDeviceVsock |
| **Unix socket / gRPC** | Host-only control plane, no guest communication             | EchoServer, CasimirControlServer    |
| **None**               | Standalone host service                                     | MetricsService                      |

### Allocating an HVC Port

If you need an HVC port, check the current allocations in
[vm_manager.h](../../base/cvd/cuttlefish/host/libs/vm_manager/vm_manager.h):

```
/dev/hvc0  = kernel console
/dev/hvc1  = serial console
/dev/hvc2  = serial logging (logcat)
/dev/hvc3  = keymaster
/dev/hvc4  = gatekeeper
/dev/hvc5  = bluetooth
/dev/hvc6  = gnss
/dev/hvc7  = location
/dev/hvc8  = confirmationui
/dev/hvc9  = uwb
/dev/hvc10 = oemlock
/dev/hvc11 = keymint
/dev/hvc12 = NFC
/dev/hvc13 = **vacant** (currently a sink — use this first)
/dev/hvc14 = MCU control
/dev/hvc15 = MCU UART
/dev/hvc16 = Ti50 TPM FIFO
/dev/hvc17 = jcardsimulator
/dev/hvc18 = sensors control
/dev/hvc19 = sensors data
```

**Port 13 is vacant** and available for use. If you need more ports, you'll
need to increase `kDefaultNumHvcs` (currently 20) — but this changes PCI
device numbering and requires SEPolicy updates.

---

## Step 2: Create the Launcher Files

### Option A: Simple Device (AutoCmd)

For a device that just needs to launch a single binary, use the `AutoCmd`
template. Create two files:

**`launch/new_device.h`**:

```cpp
#pragma once

#include "cuttlefish/host/libs/config/cuttlefish_config.h"
#include "cuttlefish/host/libs/feature/command_source.h"
#include "cuttlefish/result/result.h"

namespace cuttlefish {

// Returns a MonitorCommand to launch the new_device binary.
// The function name becomes the AutoCmd name and log identifier.
Result<MonitorCommand> NewDevice(
    const CuttlefishConfig& config,
    const CuttlefishConfig::InstanceSpecific& instance);

}  // namespace cuttlefish
```

**`launch/new_device.cpp`**:

```cpp
#include "cuttlefish/host/commands/run_cvd/launch/new_device.h"

#include "cuttlefish/common/libs/utils/known_paths.h"
#include "cuttlefish/host/libs/config/known_paths.h"

namespace cuttlefish {

Result<MonitorCommand> NewDevice(
    const CuttlefishConfig& config,
    const CuttlefishConfig::InstanceSpecific& instance) {
  // Return empty if the device is disabled
  // (use Result<std::optional<MonitorCommand>> if conditional)

  Command cmd(HostBinaryPath("new_device_binary"));
  cmd.AddParameter("--port=", instance.new_device_port());
  // ... add more parameters ...

  return MonitorCommand(std::move(cmd));
}

}  // namespace cuttlefish
```

### Option B: Complex Device (Manual Component)

For devices that need `VmmDependencyCommand`, multiple interfaces, or complex
wiring:

**`launch/new_device.h`**:

```cpp
#pragma once

#include <fruit/fruit.h>

#include "cuttlefish/host/libs/config/cuttlefish_config.h"
#include "cuttlefish/host/libs/vm_manager/vm_manager.h"

namespace cuttlefish {

fruit::Component<fruit::Required<const CuttlefishConfig,
                                 const CuttlefishConfig::InstanceSpecific,
                                 LogTeeCreator>>
NewDeviceComponent();

}  // namespace cuttlefish
```

**`launch/new_device.cpp`**:

```cpp
#include "cuttlefish/host/commands/run_cvd/launch/new_device.h"

#include "cuttlefish/host/libs/feature/command_source.h"

namespace cuttlefish {
namespace {

class NewDevice : public vm_manager::VmmDependencyCommand {
 public:
  INJECT(NewDevice(const CuttlefishConfig& config,
                   const CuttlefishConfig::InstanceSpecific& instance,
                   LogTeeCreator& log_tee))
      : config_(config), instance_(instance), log_tee_(log_tee) {}

  // SetupFeature
  std::string Name() const override { return "NewDevice"; }
  bool Enabled() const override { return instance_.enable_new_device(); }
  std::unordered_set<SetupFeature*> Dependencies() const override {
    return {static_cast<SetupFeature*>(&log_tee_)};
  }
  Result<void> ResultSetup() override {
    // Build command here
    return {};
  }

  // CommandSource
  Result<std::vector<MonitorCommand>> Commands() override {
    // Return built commands
  }

  // StatusCheckCommandSource (VmmDependencyCommand)
  Result<void> WaitForAvailability() override {
    // Wait for the device to be ready (e.g., socket file exists)
    // Typically: WaitForUnixSocketListeningWithoutConnect(path, 30s)
    return {};
  }

 private:
  const CuttlefishConfig& config_;
  const CuttlefishConfig::InstanceSpecific& instance_;
  LogTeeCreator& log_tee_;
  std::vector<MonitorCommand> commands_;
};

}  // namespace

fruit::Component<...> NewDeviceComponent() {
  return fruit::createComponent()
      .addMultibinding<CommandSource, NewDevice>()
      .addMultibinding<SetupFeature, NewDevice>()
      .addMultibinding<vm_manager::VmmDependencyCommand, NewDevice>();
}

}  // namespace cuttlefish
```

---

## Step 3: Register in main.cc

Add the include and install call in
[main.cc](../../base/cvd/cuttlefish/host/commands/run_cvd/main.cc):

**For AutoCmd**:

```cpp
#include "cuttlefish/host/commands/run_cvd/launch/new_device.h"

// In runCvdComponent():
.install(AutoCmd<NewDevice>::Component)
```

**For Manual Component**:

```cpp
#include "cuttlefish/host/commands/run_cvd/launch/new_device.h"

// In runCvdComponent():
.install(NewDeviceComponent)
```

### Install Order

Pay attention to the installation order:

- **Vhost-user devices** must be installed **before** `VmManagerComponent`
  (they must be stopped _after_ the VMM)
- **VmmDependencyCommand** devices can be installed anywhere (the prerequisite
  mechanism handles ordering)
- **Linux-only** devices should be guarded with `#ifdef __linux__`

---

## Step 4: Add HVC Port in crosvm_manager.cpp (if needed)

If your device uses an HVC port, add the FIFO wiring in
[crosvm_manager.cpp](../../base/cvd/cuttlefish/host/libs/vm_manager/crosvm_manager.cpp):

```cpp
// /dev/hvc13 = your new device (was vacant)
if (instance.enable_new_device()) {
  crosvm_cmd.AddHvcReadWrite(
      instance.PerInstanceInternalPath("new_device_fifo_vm.out"),
      instance.PerInstanceInternalPath("new_device_fifo_vm.in"));
} else {
  crosvm_cmd.AddHvcSink();
}
```

Also update the comment in `vm_manager.h`:

```cpp
// - /dev/hvc13 = new_device (was vacant)
```

**Important**: Always maintain the total HVC + disk count:

```cpp
CF_EXPECT(crosvm_cmd.HvcNum() + disk_num ==
            VmManager::kMaxDisks + VmManager::kDefaultNumHvcs, ...);
```

---

## Step 5: Add Configuration Fields (if needed)

If your device needs configurable parameters, add them to `CuttlefishConfig`:

1. Add fields to the config class (port numbers, enable flags, paths)
2. Add a `ConfigFragment` if the device has its own config section
3. Register the fragment in the Fruit component

---

## Step 6: Add Observability

### Log Capture (log_tee)

Use `LogTeeCreator` to capture your process's output:

```cpp
Result<MonitorCommand> NewDevice(
    const CuttlefishConfig::InstanceSpecific& instance,
    LogTeeCreator& log_tee) {
  // Create log_tee first (it must be started before the device)
  auto [log_tee_cmd, log_pipe] = CF_EXPECT(
      log_tee.CreateLogTee(instance, "new_device"));

  Command cmd(HostBinaryPath("new_device_binary"));
  cmd.RedirectStdIO(Subprocess::StdIOChannel::kStdOut, log_pipe);
  cmd.RedirectStdIO(Subprocess::StdIOChannel::kStdErr, log_pipe);

  // Return both commands — log_tee first so it starts first
  std::vector<MonitorCommand> commands;
  commands.emplace_back(std::move(log_tee_cmd));
  commands.emplace_back(std::move(cmd));
  return commands;
}
```

### Diagnostic Information

Register a diagnostic message using `AutoDiagnostic`:

```cpp
// In new_device.h:
Result<std::vector<std::string>> NewDeviceInfo(
    const CuttlefishConfig::InstanceSpecific& instance);

// In new_device.cpp:
Result<std::vector<std::string>> NewDeviceInfo(
    const CuttlefishConfig::InstanceSpecific& instance) {
  return std::vector<std::string>{
      "New device log: " + instance.PerInstancePath("new_device.log"),
  };
}

// In main.cc:
.install(AutoDiagnostic<NewDeviceInfo>::Component)
```

---

## Step 7: Add process_restarter (if needed)

For devices that should auto-restart on failure:

```cpp
CrosvmBuilder builder;
builder.ApplyProcessRestarter(
    HostBinaryPath("new_device_binary"),
    /*first_time_argument=*/"",      // extra arg only on first run
    /*exit_code=*/-1);               // or specific exit code to restart on

// Or manually:
Command cmd(HostBinaryPath("process_restarter"));
cmd.AddParameter("--");
cmd.AddParameter(HostBinaryPath("new_device_binary"));
cmd.AddParameter("-when_killed");
cmd.AddParameter("-when_dumped");
cmd.AddParameter("-when_exited_with_failure");
```

---

## Step 8: Add BUILD Rules

Add the new source files to the BUILD.bazel in the launch directory:

```python
# In base/cvd/cuttlefish/host/commands/run_cvd/launch/BUILD.bazel
cc_library(
    name = "new_device",
    srcs = ["new_device.cpp"],
    hdrs = ["new_device.h"],
    deps = [
        "//base/cvd/cuttlefish/host/libs/config:cuttlefish_config",
        "//base/cvd/cuttlefish/host/libs/feature:command_source",
        # ... other deps
    ],
)
```

And add the dependency to the `run_cvd` binary target.

---

## Checklist

- [ ] Choose communication pattern (HVC/vsock/vhost-user/unix socket/none)
- [ ] Create `launch/new_device.h` and `launch/new_device.cpp`
- [ ] Implement launcher function or class
- [ ] Register in `main.cc` via `.install()`
- [ ] If HVC: add FIFO wiring in `crosvm_manager.cpp`
- [ ] If HVC: update port comment in `vm_manager.h`
- [ ] If VmmDependency: implement `WaitForAvailability()`
- [ ] Add configuration fields to `CuttlefishConfig` if needed
- [ ] Add `log_tee` for output capture
- [ ] Add `DiagnosticInformation` for observability
- [ ] Add `process_restarter` if crash recovery is needed
- [ ] Add BUILD.bazel rules
- [ ] Guard with `#ifdef __linux__` if Linux-only
- [ ] Verify HVC+disk count invariant is maintained
- [ ] Test install order (vhost-user before VMM)
