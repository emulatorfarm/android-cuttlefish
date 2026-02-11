# Secondary VM Devices

## OpenWrt (WiFi Access Point VM)

**Binary**: crosvm (second VM instance)
**Source**: [open_wrt.cpp](../../base/cvd/cuttlefish/host/commands/run_cvd/launch/open_wrt.cpp)
**Registration**: `OpenWrtComponent` (manual Fruit component class)

### Communication Channels

| Channel                   | Type            | Purpose                            |
| ------------------------- | --------------- | ---------------------------------- |
| WiFi TAP device           | `wifi_tap_name` | Network bridge to main VM          |
| Vhost-user mac80211-hwsim | Socket          | WiFi medium (via wmediumd)         |
| Control socket            | Unix socket     | Crosvm control (shutdown/snapshot) |
| Serial console            | 8250 UART       | Console output (read-only)         |
| HVC log                   | FIFO            | OpenWrt system log                 |

### Description

Launches a second crosvm instance running an OpenWrt Linux image as a WiFi
access point. The OpenWrt VM bridges the Android guest's WiFi interface to a
host TAP device, providing realistic WiFi networking. Wmediumd simulates the
wireless medium between the two VMs' mac80211-hwsim interfaces.

### Boot Flow

Two boot modes are supported:

1. **Grub boot** (default): Uses a composite disk with Grub bootloader
2. **Legacy direct boot**: Directly loads kernel and rootfs images

The boot mode is determined by the presence of a composite disk image.

### Special Features

- **process_restarter**: **Yes** — wraps crosvm in `process_restarter` with
  exit code 32 (guest reboot)
- **log_tee**: **Yes** — captures crosvm stderr as "openwrt"
- **Snapshot support**: Supports save/restore alongside the main VM
  (uses `--restore=` on first boot after snapshot)

### Process Tree

```
process_restarter
└── crosvm run (OpenWrt)
log_tee (openwrt)
```

### Dependencies

- `CuttlefishConfig`, `InstanceSpecific`, `EnvironmentSpecific`
- `LogTeeCreator`
- `KernelLogPipeProvider`
- `WmediumdServer` — WiFi medium must be ready
- `VhostDeviceVsock` — vsock transport must be ready

### Crosvm Configuration

The OpenWrt crosvm is configured with:

- 1 vCPU, minimal memory
- 1 TAP device (wifi)
- Vhost-user mac80211-hwsim socket (from wmediumd)
- No USB, no balloon, no RNG, no sandbox
- Serial console for boot messages
- Optional PMem for persistent storage

### Direction

Bidirectional — WiFi AP serving the Android guest.

### Condition

Only launched when WiFi is enabled with mac80211-hwsim and OpenWrt is
configured.

### Platform

Linux-only (`#ifdef __linux__`).
