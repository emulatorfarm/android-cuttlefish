# Vsock Devices

These devices communicate with the Android guest via virtio-vsock, a
socket-based transport that provides stream connections between the host and
guest identified by a Context ID (CID) and port number. Unlike HVC FIFOs,
vsock provides a true socket abstraction with connect/accept semantics.

---

## ModemSimulator

**Binary**: `modem_simulator`
**Source**: [modem.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/modem.cpp)
**Registration**: `AutoCmd<ModemSimulator>::Component`

### Communication Channels

- **Vsock server** on `modem_simulator_ports` (one or more ports)
- When vhost-user vsock is enabled, binds to a Unix domain socket instead
- **Unix socket** for monitoring/control (`modem_simulator_host_id`)

### Description

Emulates an AT-command-based cellular modem for the guest's Radio Interface
Layer (RIL). The guest RIL connects via vsock and exchanges AT commands. The
modem simulator supports multiple SIM slots and provides basic telephony
emulation (calls, SMS, data).

### Custom Stopper

Uses a custom shutdown protocol: sends `STOP` on the monitoring Unix socket
and waits for `OK` response before terminating the process. This ensures
clean state serialization.

### Dependencies

- `CuttlefishConfig::InstanceSpecific`

### Host Resources

- Vsock ports (one per SIM slot)
- Monitoring Unix socket

### Direction

Bidirectional — guest RIL sends AT commands, host sends responses and
unsolicited events (incoming calls, SMS).

### Condition

Only launched when `instance.enable_modem_simulator()` is true and
modem simulator ports are configured.

---

## TombstoneReceiver

**Binary**: `tombstone_receiver`
**Source**: [tombstone_receiver.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/tombstone_receiver.cpp)
**Registration**: `AutoCmd<TombstoneReceiver>::Component`

### Communication Channels

- **Vsock server** on `tombstone_receiver_port`
- When vhost-user vsock is enabled, binds to a Unix domain socket instead

### Description

Receives native crash tombstone files from the Android guest. When a process
crashes in the guest, the tombstone daemon connects via vsock and uploads
the crash dump. The receiver writes these to the host's
`<instance>/tombstones/` directory.

### Dependencies

- `CuttlefishConfig::InstanceSpecific`

### Host Resources

- `<instance>/tombstones/` directory (created if not exists)
- Vsock port

### Direction

Guest → Host (unidirectional crash dump upload).

---

## VhalProxyServer

**Binary**: `vhal_proxy_server`
**Source**: [vhal_proxy_server.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/vhal_proxy_server.cpp)
**Registration**: `AutoCmd<VhalProxyServer>::Component`

### Communication Channels

- **Vsock** on `vhal_proxy_server_port`, or
- **Vhost-user vsock** Unix socket when vhost-user vsock is enabled
- Also receives `ethernet_mac` address for VHAL network configuration

### Description

Proxies the Android Vehicle HAL (VHAL) interface between the guest automotive
stack and host-side vehicle property servers. Used for automotive (AAOS)
emulation to enable external tools to read/write vehicle properties.

### Dependencies

- `CuttlefishConfig::InstanceSpecific`

### Direction

Bidirectional — external tools and guest VHAL exchange vehicle property
get/set/subscribe messages.

### Condition

Only launched when `instance.enable_vehicle_hal_grpc_server()` is true.
Linux-only (`#ifdef __linux__`).
