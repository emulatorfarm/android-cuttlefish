# Vsock + TCP Proxy Devices

These devices use a combination of vsock and TCP, with `socket_vsock_proxy`
bridges translating between the two transports. They are the most complex
launchers, spawning multiple helper processes alongside the main emulator binary.

---

## RootCanal (Bluetooth Emulator)

**Binary**: `root-canal` + 4× `socket_vsock_proxy`
**Source**: [root_canal.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/root_canal.cpp)
**Registration**: `RootCanalComponent` (manual Fruit component class)

### Communication Channels

| Channel                   | Type                   | Purpose                       |
| ------------------------- | ---------------------- | ----------------------------- |
| `rootcanal_hci_port`      | TCP (bridged to vsock) | HCI data to/from guest BT HAL |
| `rootcanal_test_port`     | TCP (bridged to vsock) | Test/control interface        |
| `rootcanal_link_port`     | TCP (bridged to vsock) | BT link-layer simulation      |
| `rootcanal_link_ble_port` | TCP (bridged to vsock) | BLE link-layer simulation     |

Each TCP port is bridged to a vsock port via a `socket_vsock_proxy` process
with `--server=tcp` listening on the TCP side and connecting to the guest's
vsock CID.

### Description

RootCanal is a Bluetooth HCI emulator that provides a virtual Bluetooth
controller. The guest's Bluetooth HAL connects via vsock (proxied through TCP)
to exchange HCI commands and events. RootCanal simulates the Bluetooth
link layer, allowing multiple virtual devices to communicate.

### Process Tree

```
root-canal
├── log_tee (root-canal)
├── socket_vsock_proxy (HCI)
├── socket_vsock_proxy (test)
├── socket_vsock_proxy (link)
└── socket_vsock_proxy (link_ble)
```

### Special Features

- **process_restarter**: Yes — restarts on kill, dump, or exit with failure
  (`-when_killed -when_dumped -when_exited_with_failure`)
- **log_tee**: Yes — full log capture via FIFO
- **Diagnostics**: Reports test channel port for external BT test tools

### Dependencies

- `CuttlefishConfig`, `InstanceSpecific`
- `LogTeeCreator` — for log capture
- `KernelLogPipeProvider` — (available but usage depends on wiring)

### Direction

Bidirectional — guest BT HAL ↔ vsock proxy ↔ TCP ↔ root-canal.

### Condition

Only launched when `instance.enable_host_bluetooth()` is true and the
connectivity connector is `rootcanal`.

---

## NetsimServer (Network Simulator)

**Binary**: `netsimd` + 2× `socket_vsock_proxy`
**Source**: [netsim_server.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/netsim_server.cpp)
**Registration**: `NetsimServerComponent` (manual Fruit component class)

### Communication Channels

| Channel               | Type                   | Purpose                                            |
| --------------------- | ---------------------- | -------------------------------------------------- |
| BT HCI FIFOs          | FIFO (via JSON config) | Bluetooth chip data (passed in device config JSON) |
| UWB UCI FIFOs         | FIFO (via JSON config) | UWB chip data (passed in device config JSON)       |
| `rootcanal_hci_port`  | TCP (bridged to vsock) | HCI vsock proxy                                    |
| `rootcanal_test_port` | TCP (bridged to vsock) | Test vsock proxy                                   |

### Description

Netsim is a newer, more comprehensive network simulator that replaces RootCanal
for Bluetooth and also handles UWB simulation. It receives device chip data
paths via a JSON configuration file that specifies FIFO paths for each radio
technology.

### Process Tree

```
netsimd
├── socket_vsock_proxy (HCI)
└── socket_vsock_proxy (test)
```

### Special Features

- **process_restarter**: No
- **log_tee**: No
- **Device config**: Passes chip FIFOs via `--device_config=<json>` with per-chip
  `bt` and `uwb` sections

### Dependencies

- `CuttlefishConfig`, `InstanceSpecific`
- `LogTeeCreator` — (available)

### Direction

Bidirectional — guest BT/UWB HALs ↔ FIFOs/vsock ↔ netsim.

### Condition

Only launched when the connectivity connector is `netsim`.
