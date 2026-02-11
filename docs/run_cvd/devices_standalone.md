# Standalone Devices (No VMM Communication)

These devices don't directly communicate with the VMM or the guest. They
provide host-side services, telemetry, or serve as backends for other devices.

---

## MetricsService

**Binary**: `metrics`
**Source**: [metrics.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/metrics.cpp)
**Registration**: `AutoCmd<MetricsService>::Component`

### Communication Channels

None — standalone host service.

### Description

Collects and reports telemetry/metrics about the running virtual device
instance. Runs as a background process without any communication to the
VMM or guest.

### Direction

Host-only (telemetry collection).

### Condition

Only launched when `config.enable_metrics()` is `kYes`.

---

## AutomotiveProxy

**Binary**: `automotive_proxy`
**Source**: [automotive_proxy.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/automotive_proxy.cpp)
**Registration**: `AutoCmd<AutomotiveProxyService>::Component`

### Communication Channels

- Reads configuration from `etc/automotive/proxy_config.json`

### Description

Provides automotive-specific proxy services. Reads a JSON configuration file
and sets up proxy connections for automotive emulation scenarios (AAOS).

### Direction

Host-side proxy — no direct VMM communication.

### Condition

Only launched when `instance.enable_automotive_proxy()` is true and the
config file exists.

### Platform

Linux-only (`#ifdef __linux__`).

---

## Pica (UWB Emulator)

**Binary**: `pica`
**Source**: [pica.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/pica.cpp)
**Registration**: `AutoCmd<Pica>::Component`

### Communication Channels

| Channel         | Type       | Purpose                                 |
| --------------- | ---------- | --------------------------------------- |
| `pica_uci_port` | TCP        | UCI server (UwbConnector connects here) |
| PCAP output dir | Filesystem | Packet capture output                   |

### Description

Pica is a UWB (Ultra-Wideband) emulator that implements the UCI (UWB Command
Interface) protocol. It listens on a TCP port and the UwbConnector bridges
the guest's UWB HAL (via hvc9 FIFO) to this TCP port. Pica can capture
UWB packets to a PCAP directory for debugging.

### Special Features

- **log_tee**: **Yes** — captures stderr as "pica"

### Dependencies

- `CuttlefishConfig`, `InstanceSpecific`
- `LogTeeCreator`
- `GrpcSocketCreator`

### Direction

Host service — provides UCI server for UWB; UwbConnector bridges to guest.

### Condition

Only launched when `config.enable_host_uwb()` is true and the UWB connector
is `pica`.
