# Cuttlefish-on-Kubernetes: Design Considerations

> **Status:** Draft  
> **Date:** 2026-02-12  
> **Scope:** Running one Cuttlefish Virtual Device (CVD) per Kubernetes pod without resource conflicts across co-located pods on a node.

---

## Table of Contents

1. [Resource Inventory](#1-resource-inventory)
2. [Device Nodes & Privilege Requirements](#2-device-nodes--privilege-requirements)
3. [Instance-Numbering & Isolation Model](#3-instance-numbering--isolation-model)
4. [Kubernetes Resource Mapping](#4-kubernetes-resource-mapping)
5. [Isolation Boundaries](#5-isolation-boundaries)
6. [Networking Strategy](#6-networking-strategy)
7. [Storage Strategy](#7-storage-strategy)
8. [Operational Concerns](#8-operational-concerns)
9. [Decision Matrix](#9-decision-matrix)

---

## 1. Resource Inventory

Every CVD instance launched by `run_cvd` consumes the following host resources. These were catalogued from the Cuttlefish codebase — primarily `crosvm_manager.cpp`, `flags.cc`, `config_utils.cpp`, and `cuttlefish-base.cuttlefish-host-resources.init`.

### 1.1 Compute

| Resource             | Default              | Flag          | Notes                          |
| -------------------- | -------------------- | ------------- | ------------------------------ |
| vCPUs                | 4                    | `--cpus`      | Passed to crosvm as `--cpus=N` |
| RAM (MB)             | dynamic (auto-sized) | `--memory_mb` | Passed to crosvm as `--mem=N`  |
| SMT / HyperThreading | enabled              | `--smt`       | `--no-smt` disables            |
| KVM acceleration     | required             | `--kvm_path`  | `/dev/kvm` must be available   |

### 1.2 Storage — Disk Images Per Instance

Each CVD creates or references multiple disk images. With `--use_overlay=true` (default), qcow2 overlay files are layered on top of shared read-only base images.

| Image                           | Purpose                                       | Typical Size                                |
| ------------------------------- | --------------------------------------------- | ------------------------------------------- |
| OS composite disk               | Android system, vendor, product partitions    | 4–8 GB (base, read-only shareable)          |
| Persistent composite disk       | Survives powerwash                            | variable                                    |
| Userdata (blank_data_image)     | App data, `/data` partition                   | 8 GB default (`--blank_data_image_mb=8192`) |
| Super image                     | Dynamic partitions super image                | ~4 GB                                       |
| Boot image                      | Kernel + ramdisk                              | ~64 MB                                      |
| Vendor boot image               | Vendor ramdisk                                | ~32 MB                                      |
| Verified boot metadata (vbmeta) | AVB metadata                                  | < 1 MB                                      |
| Bootloader / BIOS               | `--bios=` flag                                | ~4 MB                                       |
| PFLASH (x86_64 only)            | UEFI variable store                           | ~256 KB                                     |
| ESP image                       | EFI system partition                          | ~64 MB                                      |
| U-Boot env image                | U-Boot environment                            | < 1 MB                                      |
| HWComposer pmem                 | Pmem for hardware composer                    | ~4 MB                                       |
| Access key registry (pmem)      | TEE key storage                               | ~2 MB                                       |
| Pstore                          | Persistent store (crash logs)                 | ~256 KB                                     |
| SD card (optional)              | Virtual SD card                               | `--use_sdcard`                              |
| qcow2 overlays                  | Per-instance CoW overlays on all of the above | grows dynamically                           |

### 1.3 Network — TAP Interfaces, Bridges, and IP Subnets

Each instance requires up to **4 TAP interfaces** and associated IP addressing:

| Interface          | Naming            | IP Subnet                                       | Purpose         |
| ------------------ | ----------------- | ----------------------------------------------- | --------------- |
| Mobile TAP         | `cvd-mtap-<NN>`   | `192.168.97.0/30` per-instance (point-to-point) | Cellular / RIL  |
| Ethernet TAP       | `cvd-etap-<NN>`   | `192.168.98.0/24` (bridged, shared)             | Ethernet        |
| WiFi TAP (bridged) | `cvd-wtap-<NN>`   | `192.168.96.0/24` (bridged, shared)             | Legacy WiFi     |
| WiFi AP TAP        | `cvd-wifiap-<NN>` | `192.168.94.0/30` per-instance                  | OpenWrt WiFi AP |

**Bridges** (shared across instances on a host):

| Bridge          | Name      | Subnet            |
| --------------- | --------- | ----------------- |
| Ethernet bridge | `cvd-ebr` | `192.168.98.0/24` |
| WiFi bridge     | `cvd-wbr` | `192.168.96.0/24` |

Each mobile and WiFi-AP TAP gets a `/30` subnet with the formula:

- Gateway: `<base>.(4*i - 3)`
- Network: `<base>.(4*i - 4)/30`

An `iptables -t nat POSTROUTING -s <network> -j MASQUERADE` rule is created for each mobile and WiFi-AP interface.

### 1.4 Port Ranges

All port formulas use the instance number `N` (starting at 1):

| Port / Range       | Formula            | Default (N=1) | Purpose                           |
| ------------------ | ------------------ | ------------- | --------------------------------- |
| ADB host port      | `6520 + N - 1`     | 6520          | ADB TCP forwarding                |
| Fastboot host port | same as ADB        | 6520          | Fastboot                          |
| WebRTC TCP range   | `--tcp_port_range` | `15550:15599` | WebRTC ICE (TCP)                  |
| WebRTC UDP range   | `--udp_port_range` | `15550:15599` | WebRTC ICE (UDP)                  |
| Sig-server proxy   | `8443 + N - 1`     | 8443          | WebRTC signaling proxy            |
| GNSS gRPC proxy    | `7200 + N - 1`     | 7200          | GNSS proxy                        |
| Tombstone receiver | `6600 + (CID - 3)` | 6600          | Crash dump receiver (vsock-based) |
| Lights server      | `6900 + (CID - 3)` | 6900          | Lights HAL (vsock-based)          |
| RootCanal HCI      | `7300 + inst`      | 7300          | Bluetooth HCI                     |
| RootCanal Link     | `7400 + inst`      | 7400          | Bluetooth Link                    |
| RootCanal Test     | `7500 + inst`      | 7500          | Bluetooth Test                    |
| RootCanal BLE      | `7600 + inst`      | 7600          | Bluetooth BLE                     |
| Casimir NCI        | `7800 + inst`      | 7800          | NFC NCI                           |
| Casimir RF         | `7900 + inst`      | 7900          | NFC RF                            |
| Pica UCI           | `7000 + inst`      | 7000          | UWB control                       |
| QEMU VNC           | `544 + N - 1`      | 544           | VNC (QEMU only)                   |

### 1.5 IPC — Vsock, FIFOs, Unix Sockets

#### Vsock CID

- Default: `3 + instance - 1` (instance 1 → CID 3)
- Flag: `--vsock_guest_cid`
- Device: `/dev/vhost-vsock` (node-wide, not namespaced)
- Port formula: `GetVsockServerPort(base, cid) = base + (cid - 3)`

#### Named FIFOs (HVC Channels) — ~20 pairs per instance

Each FIFO pair carries a `.in` and `.out` file for bidirectional communication:

| HVC Port   | FIFO Pair                   | Purpose            |
| ---------- | --------------------------- | ------------------ |
| /dev/hvc0  | `kernel.log`                | Kernel console log |
| /dev/hvc1  | `console_out`, `console_in` | Serial console I/O |
| /dev/hvc2  | `logcat`                    | Logcat serial      |
| /dev/hvc3  | `keymaster_fifo_vm`         | Keymaster (C++)    |
| /dev/hvc4  | `gatekeeper_fifo_vm`        | Gatekeeper         |
| /dev/hvc5  | `bt_fifo_vm`                | Bluetooth          |
| /dev/hvc6  | `gnsshvc_fifo_vm`           | GNSS               |
| /dev/hvc7  | `locationhvc_fifo_vm`       | Location           |
| /dev/hvc8  | `confui_fifo_vm`            | ConfirmationUI     |
| /dev/hvc9  | `uwb_fifo_vm`               | UWB                |
| /dev/hvc10 | `oemlock_fifo_vm`           | OEM Lock           |
| /dev/hvc11 | `keymint_fifo_vm`           | KeyMint (Rust)     |
| /dev/hvc12 | `nfc_fifo_vm`               | NFC                |
| /dev/hvc13 | _(vacant)_                  | —                  |
| /dev/hvc14 | `mcu/control`               | MCU control        |
| /dev/hvc15 | `mcu/uart0`                 | MCU UART           |
| /dev/hvc16 | `direct_tpm_fifo`           | Ti50 TPM           |
| /dev/hvc17 | `jcardsim_fifo_vm`          | JCard Simulator    |
| /dev/hvc18 | `sensors_control_fifo_vm`   | Sensors control    |
| /dev/hvc19 | `sensors_data_fifo_vm`      | Sensors data       |

Additional FIFOs: `crosvm.fifo` (crosvm log output), `crosvm_vhost_user_gpu.fifo` (GPU device logs).

#### Unix Domain Sockets — 10+ per instance

| Socket Path                      | Purpose                              |
| -------------------------------- | ------------------------------------ |
| `CrosvmSocketPath()`             | Crosvm control socket                |
| `OpenwrtCrosvmSocketPath()`      | OpenWrt crosvm control               |
| `touch_socket_path(N)`           | Vhost-user input (touch per display) |
| `mouse_socket_path()`            | Vhost-user input (mouse)             |
| `gamepad_socket_path()`          | Vhost-user input (gamepad)           |
| `rotary_socket_path()`           | Vhost-user input (rotary)            |
| `keyboard_socket_path()`         | Vhost-user input (keyboard)          |
| `switches_socket_path()`         | Vhost-user input (switches)          |
| `frames_socket_path()`           | Wayland frames (GPU → streamer)      |
| `vhost-user-gpu-socket`          | Vhost-user GPU device                |
| `audio_server_path()`            | Audio (`--sound=`)                   |
| `grpc_socket_path()`             | gRPC control socket                  |
| `launcher_monitor_socket_path()` | Launcher monitor                     |
| `vhost-user-vsock-socket`        | Vhost-user vsock device              |

### 1.6 Companion Host Processes — 15–37 per instance

The following processes are spawned per CVD instance (from the 37 launch files under `host/commands/run_cvd/launch/`):

| Process                                     | Condition                              | Category      |
| ------------------------------------------- | -------------------------------------- | ------------- |
| **crosvm** (VM manager)                     | Always                                 | Core          |
| **log_tee** (crosvm logs)                   | Always                                 | Core          |
| **kernel_log_monitor**                      | Always                                 | Core          |
| **logcat_receiver**                         | Always                                 | Core          |
| **tombstone_receiver**                      | Always                                 | Core          |
| **secure_env** (KeyMint/Gatekeeper/OEMLock) | Always                                 | Security      |
| **adb_connector** (via socket_vsock_proxy)  | Always                                 | Connectivity  |
| **console_forwarder**                       | When `--console=true`                  | Debug         |
| **webrtc** (streaming server)               | When GPU enabled                       | Streaming     |
| **webrtc_sig_server_proxy**                 | With WebRTC                            | Streaming     |
| **modem_simulator**                         | When `--enable_modem_simulator=true`   | Connectivity  |
| **gnss_grpc_proxy**                         | When `--start_gnss_proxy=true`         | Connectivity  |
| **bluetooth_connector**                     | When BT enabled (non-netsim)           | Connectivity  |
| **root_canal**                              | First instance, BT enabled             | Shared        |
| **casimir** + control server                | First instance, NFC enabled            | Shared        |
| **pica**                                    | First instance, UWB enabled            | Shared        |
| **netsim**                                  | First instance, netsim enabled         | Shared        |
| **wmediumd**                                | First instance, WiFi enabled           | Shared        |
| **openwrt** (crosvm) + control server       | When AP configured                     | WiFi          |
| **screen_recording_server**                 | When `--record_screen=true`            | Debug         |
| **sensors_simulator**                       | Always                                 | HAL           |
| **vhost_user_input**                        | Per display + peripherals              | Input         |
| **vhost_device_vsock**                      | When vhost-user vsock enabled          | IPC           |
| **control_env_proxy_server**                | Always                                 | Control       |
| **echo_server**                             | Always                                 | Test          |
| **metrics**                                 | Optional                               | Telemetry     |
| **cvdalloc**                                | When `--use_cvdalloc=true`             | Resource mgmt |
| **mcu**                                     | When MCU config present                | Emulation     |
| **ti50_emulator**                           | When Ti50 configured                   | Security      |
| **vhal_proxy_server**                       | When `--enable_vhal_proxy_server=true` | Automotive    |
| **automotive_proxy**                        | When `--enable_automotive_proxy=true`  | Automotive    |

A typical non-automotive CVD without optional features runs **~15–20 host processes**.

---

## 2. Device Nodes & Privilege Requirements

### 2.1 Required Device Nodes

| Device Node        | Purpose                        | Required By                                    |
| ------------------ | ------------------------------ | ---------------------------------------------- |
| `/dev/kvm`         | KVM hardware virtualization    | crosvm (`--kvm-path`)                          |
| `/dev/vhost-vsock` | Vsock host-guest communication | crosvm (`--vsock=cid=N`)                       |
| `/dev/vhost-net`   | Vhost-net acceleration         | crosvm (`--vhost-net`), when enabled           |
| `/dev/net/tun`     | TAP interface creation         | `ip tuntap add`, for mobile/wifi/ethernet TAPs |

### 2.2 Linux Capabilities

| Capability             | Reason                                                           |
| ---------------------- | ---------------------------------------------------------------- |
| `CAP_NET_ADMIN`        | Creating/configuring TAP interfaces, bridges, and iptables rules |
| `CAP_NET_RAW`          | Raw socket operations for network configuration                  |
| `CAP_NET_BIND_SERVICE` | Binding to privileged ports (e.g., DHCP on port 67)              |

### 2.3 Kernel Modules

| Module                             | Purpose                             |
| ---------------------------------- | ----------------------------------- |
| `kvm` (+ `kvm_intel` or `kvm_amd`) | Hardware virtualization             |
| `vhost-vsock`                      | Vsock device support                |
| `vhost-net`                        | Vhost-net acceleration              |
| `bridge`                           | Linux bridge for ethernet/wifi TAPs |
| `tun`                              | TAP device creation                 |

### 2.4 System Groups

| Group        | Purpose                                                       |
| ------------ | ------------------------------------------------------------- |
| `kvm`        | Access to `/dev/kvm`                                          |
| `cvdnetwork` | Access to `/dev/vhost-net`, `/dev/vhost-vsock`, TAP ownership |

### 2.5 Security Policy

The existing container setup (from `Containerfile` and `cuttlefish-base.cuttlefish-host-resources.init`) uses:

- **Privileged container mode** or explicit device access — the init script detects `/.dockerenv` and adjusts ownership of `/dev/kvm`, `/dev/vhost-net`, `/dev/vhost-vsock`.
- **Seccomp**: Crosvm has its own seccomp policy directory (`usr/share/crosvm/<arch>-linux-gnu/seccomp`). The `--enable_sandbox` flag (default: `false`) controls whether crosvm applies seccomp; when disabled, `--disable-sandbox` is passed. For Kubernetes, the container-level seccomp profile must be `Unconfined` or a custom profile that permits the syscalls crosvm needs.

---

## 3. Instance-Numbering & Isolation Model

### 3.1 Instance Number Resolution

The instance number is resolved in the following priority order (from `config_utils.cpp`):

1. `CUTTLEFISH_INSTANCE` environment variable
2. `USER` environment variable (if it matches `vsoc-<N>`)
3. Default: `1` (`kDefaultInstance`)

### 3.2 Per-Instance Resource Derivation

Every derived resource uses the instance number `N`:

| Resource            | Formula                                  | Conflict Domain                              |
| ------------------- | ---------------------------------------- | -------------------------------------------- |
| Vsock CID           | `3 + N - 1`                              | **Node-wide** (`/dev/vhost-vsock` is global) |
| TAP interface names | `cvd-{type}tap-<NN>`                     | **Host network namespace**                   |
| Mobile IP subnet    | `192.168.97.(4*N - 4)/30`                | **Host network namespace**                   |
| WiFi-AP IP subnet   | `192.168.94.(4*N - 4)/30`                | **Host network namespace**                   |
| ADB port            | `6520 + N - 1`                           | **Host (or pod) network namespace**          |
| WebRTC port range   | `15550:15599` (configurable)             | **Host (or pod) network namespace**          |
| All other ports     | `<base> + N - 1`                         | **Host (or pod) network namespace**          |
| Instance directory  | `$HOME/cuttlefish/instances/cvd-<N>`     | **Filesystem**                               |
| UUID                | `699acfc4-c8c4-11e7-882b-5065f31dc1<NN>` | Informational                                |
| MAC addresses       | Per-instance derived                     | **Layer-2 within bridges**                   |

### 3.3 Conflict-Prone Resources

When two pods on the same node pick overlapping instance numbers:

| Resource                       | Conflict Risk                                                                                          | Severity |
| ------------------------------ | ------------------------------------------------------------------------------------------------------ | -------- |
| **Vsock CID**                  | **Critical** — two VMs with the same CID on a shared `/dev/vhost-vsock` will cause connection failures | 🔴       |
| **TAP interface names**        | **Critical** if using host network namespace — duplicate names are rejected by the kernel              | 🔴       |
| **IP subnets**                 | **Critical** under host networking — overlapping `/30` subnets cause routing conflicts                 | 🔴       |
| **Port numbers** (ADB, WebRTC) | **High** under host networking — bind failures                                                         | 🟡       |
| **iptables NAT rules**         | **High** — overlapping MASQUERADE rules can cause misrouted traffic or cleanup failures                | 🟡       |
| **Instance directories**       | **Low** if per-pod filesystem isolation is used (ephemeral volumes)                                    | 🟢       |
| **FIFOs / Unix sockets**       | **Low** — created under per-instance directory paths                                                   | 🟢       |

---

## 4. Kubernetes Resource Mapping

### 4.1 Mapping Table

| CVD Resource               | Kubernetes Mechanism                                                   | Notes                                                                                                   |
| -------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| **vCPUs**                  | `resources.requests.cpu` / `resources.limits.cpu`                      | 4 cores default; request ≥ 4 to avoid noisy-neighbor contention                                         |
| **RAM**                    | `resources.requests.memory` / `resources.limits.memory`                | Dynamic default; for production, explicitly set (e.g., `4Gi`) including overhead for ~20 host processes |
| **`/dev/kvm`**             | Kubernetes Device Plugin (`devices.kubevirt.io/kvm` or custom)         | Exposes `/dev/kvm` to pods; each pod needs exactly 1                                                    |
| **`/dev/vhost-vsock`**     | Kubernetes Device Plugin (custom) with CID allocation                  | See [§5.2 Vsock CID Allocation](#52-vsock-cid-uniqueness)                                               |
| **`/dev/vhost-net`**       | Kubernetes Device Plugin                                               | Similar to KVM device plugin                                                                            |
| **`/dev/net/tun`**         | Kubernetes Device Plugin or `securityContext.capabilities`             | Needed for TAP creation within pod                                                                      |
| **TAP interfaces**         | Per-pod network namespace (default) **or** `hostNetwork: true`         | See [§6 Networking Strategy](#6-networking-strategy)                                                    |
| **Bridges**                | DaemonSet-managed on host **or** per-pod setup                         | Design choice — see [§9 Decision Matrix](#9-decision-matrix)                                            |
| **iptables / NAT**         | `securityContext.capabilities: [CAP_NET_ADMIN]`                        | Required inside pod for NAT/masquerade rules                                                            |
| **Disk images (base)**     | `ReadOnlyMany` PersistentVolumeClaim (shared) or node-local hostPath   | Base images are read-only; share across pods to save storage                                            |
| **Disk images (overlays)** | `emptyDir` (ephemeral) or `ReadWriteOnce` PVC                          | Per-pod qcow2 overlays; ephemeral is preferred for stateless                                            |
| **Userdata**               | `emptyDir` or `ReadWriteOnce` PVC                                      | 8 GB default; PVC if data persistence across pod restarts is needed                                     |
| **Vsock CID**              | Custom Device Plugin **or** init-container based allocation            | Must be globally unique per node; see [§5.2](#52-vsock-cid-uniqueness)                                  |
| **Port ranges**            | `containerPort` declarations; `hostPort` only if using host networking | Under CNI, pods get unique IPs, so port conflicts are eliminated                                        |
| **Seccomp**                | `securityContext.seccompProfile.type: Unconfined`                      | Crosvm requires syscalls not allowed by default Docker/containerd profile                               |
| **Capabilities**           | `securityContext.capabilities.add: [NET_ADMIN, NET_RAW]`               | For TAP + iptables management                                                                           |

### 4.2 Pod `securityContext` Template

```yaml
securityContext:
  privileged: false # avoid full privilege; use fine-grained controls
  capabilities:
    add:
      - NET_ADMIN
      - NET_RAW
    drop:
      - ALL
  seccompProfile:
    type: Unconfined # required for crosvm syscall needs
  runAsUser: 0 # root required for device access and TAP creation
```

### 4.3 Device Plugin Requirements

Three device plugins are recommended:

1. **KVM Device Plugin** — Exposes `/dev/kvm` as an allocatable resource (e.g., `devices.kubevirt.io/kvm: 1`). Well-established; KubeVirt community provides one.

2. **Vsock CID Device Plugin** (custom) — Manages a pool of vsock CIDs per node. When a pod requests `cuttlefish.dev/vsock-cid: 1`, the plugin:
   - Allocates a unique CID from the node-local pool (e.g., CIDs 3–1024)
   - Passes the CID to the pod as a device environment variable (`VSOCK_CID`)
   - Mounts `/dev/vhost-vsock` into the pod
   - Deallocates the CID on pod termination

3. **TUN/TAP Device Plugin** — Exposes `/dev/net/tun` (or simply allow via capability + security context).

---

## 5. Isolation Boundaries

### 5.1 Instance-Number Uniqueness

**Problem:** Two pods on the same node choosing the same `CUTTLEFISH_INSTANCE` will produce identical vsock CIDs, TAP names, and port numbers.

**Solutions (choose one):**

| Approach                         | Mechanism                                                                                     | Pros                                               | Cons                                                                  |
| -------------------------------- | --------------------------------------------------------------------------------------------- | -------------------------------------------------- | --------------------------------------------------------------------- |
| **Device Plugin CID allocation** | Vsock CID Device Plugin assigns unique CID; pod derives instance from CID                     | Single source of truth; no race conditions         | Requires custom device plugin                                         |
| **Init container**               | Init container calls a node-level allocator service (DaemonSet) to reserve an instance number | Flexible; can integrate with any assignment scheme | Requires a DaemonSet + coordination service                           |
| **StatefulSet ordinal**          | Use StatefulSet pod ordinal as instance number: `CUTTLEFISH_INSTANCE=$((ORDINAL + 1))`        | Simple; Kubernetes-native ordering                 | Only works within one StatefulSet; cross-StatefulSet overlap possible |
| **Pod UID hash**                 | Hash the pod UID to derive a unique instance number                                           | No coordination needed                             | Hash collisions possible; ranges must be large                        |

**Recommendation:** Use the **Vsock CID Device Plugin** approach. It solves both the CID uniqueness problem and instance number assignment in a single mechanism. The allocated CID can be used to derive the instance number (`instance = CID - 2`), which in turn drives all other per-instance resource naming.

### 5.2 Vsock CID Uniqueness

**The core problem:** `/dev/vhost-vsock` is a **node-wide device**. Two pods on the same node with the same CID will cause vsock connection failures. CIDs are not namespaced by Linux.

**Proposed solution — Vsock CID Device Plugin:**

```
Node CID Pool: [3, 4, 5, ..., 1024]
                 │
                 ▼
    ┌──────────────────────┐
    │  Vsock CID Device    │
    │  Plugin (DaemonSet)  │
    │                      │
    │  Pool: [3..1024]     │
    │  Allocated: {3→PodA} │
    │  Allocated: {4→PodB} │
    └──────────────────────┘
         │            │
    CID=3 env     CID=4 env
         │            │
    ┌────▼────┐  ┌────▼────┐
    │  Pod A  │  │  Pod B  │
    │ CVD N=1 │  │ CVD N=2 │
    └─────────┘  └─────────┘
```

Each pod reads the assigned CID from environment variable `VSOCK_CID` and starts CVD with `--vsock_guest_cid=$VSOCK_CID`.

### 5.3 TAP / Bridge Namespace Isolation

Under the default Kubernetes networking model (each pod gets its own network namespace), TAP interfaces created inside a pod are **isolated by default**. Two pods can both create `cvd-mtap-01` without conflict, because they are in separate network namespaces.

However, `hostNetwork: true` breaks this isolation. See [§6 Networking Strategy](#6-networking-strategy).

### 5.4 Filesystem Isolation

All FIFOs, Unix sockets, and per-instance directories live under the instance home directory (typically `$HOME/cuttlefish/instances/cvd-<N>`). With per-pod ephemeral storage (`emptyDir`), these are naturally isolated.

---

## 6. Networking Strategy

### 6.1 Option A: Per-Pod Network Namespace (CNI-Managed) — **Recommended**

Each pod gets its own network namespace via the Cilium CNI. TAP interfaces are created **inside** the pod's namespace.

```
┌─────────────────── Node ───────────────────┐
│                                             │
│  ┌─── Pod A netns ───┐  ┌─── Pod B netns ──┐
│  │ cvd-mtap-01       │  │ cvd-mtap-01      │
│  │ cvd-etap-01       │  │ cvd-etap-01      │
│  │ cvd-wtap-01       │  │ cvd-wtap-01      │
│  │ cvd-wifiap-01     │  │ cvd-wifiap-01    │
│  │ 192.168.97.1/30   │  │ 192.168.97.1/30  │
│  │ bridges: local    │  │ bridges: local   │
│  │ iptables: local   │  │ iptables: local  │
│  └────────┬──────────┘  └────────┬─────────┘
│           │ veth                  │ veth
│           └────────┬─────────────┘
│                    │ Cilium CNI
│              ┌─────▼─────┐
│              │ eth0 (host)│
│              └────────────┘
└─────────────────────────────────────────────┘
```

**Advantages:**

- Complete TAP/bridge/iptables isolation per pod — no naming conflicts
- All pods can use the same `--base_instance_num=1` for these resources
- Standard Kubernetes networking; Cilium manages pod-to-pod communication
- No need for unique TAP names or IP subnet partitioning across pods
- iptables rules are scoped to the pod's network namespace

**Disadvantages:**

- Guest-to-host and guest-to-internet connectivity requires NAT within the pod
- ADB, WebRTC, and other services require either `NodePort` / `LoadBalancer` services or an ingress controller for external access
- Slightly more complex ingress routing for WebRTC (UDP)

**Cilium-Specific Considerations:**

- Cilium in **tunnel mode** (VXLAN/Geneve) adds overhead; consider **native routing** mode for lower latency
- `CiliumNetworkPolicy` can restrict pod-to-pod traffic if multi-tenancy is required
- Cilium does **not** interfere with in-pod TAP devices; TAP creation via `/dev/net/tun` + `CAP_NET_ADMIN` works within the pod namespace
- eBPF-based datapath does not conflict with in-pod iptables rules (iptables apply only to traffic within the pod's namespace)

### 6.2 Option B: Host Networking (`hostNetwork: true`)

Pods share the host's network namespace. This mirrors the traditional bare-metal Cuttlefish setup.

**Advantages:**

- Simplest migration path from bare-metal; TAP setup scripts work as-is
- Direct ADB access at `host_ip:6520+N`
- WebRTC works without ingress complexity

**Disadvantages:**

- **All conflict-prone resources become critical:** TAP names, IP subnets, port numbers must be unique across all pods on the node
- Requires strict instance-number coordination per node
- iptables rules from all pods share the host's iptables, requiring careful lifecycle management (cleanup on pod termination)
- Reduced security isolation

**If choosing host networking**, enforce unique instance numbers via the Vsock CID Device Plugin (§5.1) and limit pods per node using `podAntiAffinity` or resource quotas.

### 6.3 External Access for Per-Pod Networking

For Option A (CNI-managed), expose CVD services externally:

| Service          | External Access Method                             | Notes                                             |
| ---------------- | -------------------------------------------------- | ------------------------------------------------- |
| ADB              | `NodePort` service (TCP) or `kubectl port-forward` | Port `6520` in pod → NodePort                     |
| WebRTC HTTP/WS   | `Ingress` (Cilium Ingress or Nginx)                | Path-based routing per CVD                        |
| WebRTC ICE (UDP) | `NodePort` (UDP) or `hostPort` for specific range  | UDP ingress is complex; `hostPort` may be simpler |
| gRPC control     | `ClusterIP` service                                | Internal access for orchestration                 |

### 6.4 Bridge Configuration

| Strategy                                | When to Use                                                                |
| --------------------------------------- | -------------------------------------------------------------------------- |
| **Per-pod bridges** (inside pod netns)  | Option A — each pod creates its own `cvd-ebr`, `cvd-wbr`                   |
| **DaemonSet-managed bridges** (on host) | Option B — a DaemonSet creates shared bridges on each node; pods join them |

---

## 7. Storage Strategy

### 7.1 Base Images — Read-Only Sharing

Android system images (super, boot, vendor_boot, vbmeta) are large (~6–10 GB combined) and identical across CVDs running the same Android build.

**Strategy:** Store base images on a shared `ReadOnlyMany` volume or a node-local hostPath, and use qcow2 overlays per pod.

```
┌──────────────── Node ─────────────────┐
│                                        │
│  ┌─ Shared RO Volume (hostPath) ────┐  │
│  │ super.img                        │  │
│  │ boot.img                         │  │
│  │ vendor_boot.img                  │  │
│  │ vbmeta.img                       │  │
│  └──────────┬──────────┬────────────┘  │
│             │          │               │
│  ┌──── Pod A ────┐  ┌──── Pod B ────┐  │
│  │ overlay_A.qcow2│  │ overlay_B.qcow2│ │
│  │ userdata_A.img │  │ userdata_B.img │ │
│  │ (emptyDir)     │  │ (emptyDir)     │ │
│  └────────────────┘  └────────────────┘ │
└────────────────────────────────────────┘
```

### 7.2 Volume Configuration

| Volume           | Type                                | Access Mode     | Lifecycle                   | Typical Size               |
| ---------------- | ----------------------------------- | --------------- | --------------------------- | -------------------------- |
| Base images      | `hostPath` or `PVC (ReadOnlyMany)`  | `ReadOnlyMany`  | Persist across pod restarts | 6–10 GB per Android build  |
| qcow2 overlays   | `emptyDir`                          | N/A (ephemeral) | Deleted with pod            | 1–4 GB (grows with writes) |
| Userdata         | `emptyDir` or `PVC (ReadWriteOnce)` | `ReadWriteOnce` | Ephemeral or persistent     | 8 GB default               |
| FIFOs + sockets  | `emptyDir`                          | N/A (ephemeral) | Deleted with pod            | < 1 MB                     |
| Instance runtime | `emptyDir`                          | N/A (ephemeral) | Deleted with pod            | < 100 MB                   |

### 7.3 Pre-Populating Base Images

Options for making base images available on nodes:

1. **Baked into node image** — Fast startup; requires node image rebuild for updates
2. **DaemonSet + init container** — DaemonSet downloads images to a `hostPath`; pods mount it read-only
3. **Network filesystem (NFS / CephFS)** — `ReadOnlyMany` PVC; highest flexibility, potential latency
4. **OCI image layer** — Include base images in the CVD container image; increases image size but simplifies deployment

**Recommendation:** Use a **DaemonSet** that manages base images on a `hostPath` volume. It can pull new images on demand and pods reference the local path. This balances speed (local I/O) with flexibility (remote updates).

---

## 8. Operational Concerns

### 8.1 Scaling & Density

| Consideration          | Guidance                                                                                                                                                                    |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Max pods per node**  | Limited by: `min(available_vCPUs / 4, available_RAM / 4Gi, vsock_CID_pool_size)`. For a 96-core / 384 GB node, practical limit ≈ 20–24 CVDs (accounting for host overhead). |
| **Horizontal scaling** | Use `Deployment` or `StatefulSet` with `podAntiAffinity` to spread across nodes.                                                                                            |
| **Resource quotas**    | Set `LimitRange` to enforce minimum 4 CPU / 4 Gi per pod to prevent overcommit.                                                                                             |
| **Node selectors**     | Label nodes with `cuttlefish.dev/kvm-enabled: "true"` and use `nodeSelector` in pod specs.                                                                                  |

### 8.2 Cleanup on Pod Termination

| Resource       | Cleanup Mechanism                                                                                |
| -------------- | ------------------------------------------------------------------------------------------------ |
| TAP interfaces | Pod network namespace teardown (auto-cleaned by kernel when netns is destroyed)                  |
| iptables rules | Same — scoped to pod netns in Option A. In Option B (host networking), requires `preStop` hook.  |
| Vsock CID      | Device Plugin deallocation on pod deletion                                                       |
| Disk overlays  | `emptyDir` deletion on pod removal                                                               |
| Host processes | Container runtime sends SIGTERM → SIGKILL; crosvm and companions terminate                       |
| Bridges        | In Option A, per-pod bridges are destroyed with netns. In Option B, DaemonSet manages lifecycle. |

**Risk:** If using `hostNetwork` and a pod is force-killed (OOMKill, node eviction), iptables rules and TAP interfaces on the host may leak. A **cleanup DaemonSet** should periodically reconcile and remove orphaned resources.

### 8.3 Observability

| Signal               | Source                      | Collection Method                                                     |
| -------------------- | --------------------------- | --------------------------------------------------------------------- |
| CVD boot state       | `kernel_log_monitor` output | Sidecar → stdout → Kubernetes log pipeline                            |
| Guest logcat         | `logcat_receiver`           | File or stdout; mount `emptyDir` for log files                        |
| Host process health  | Process exit codes          | `run_cvd` restarts subprocesses by default (`--restart_subprocesses`) |
| Resource utilization | Node-level CPU/mem/disk     | Prometheus node-exporter; Cilium Hubble for network                   |
| Vsock CID allocation | Device Plugin metrics       | Custom Prometheus exporter on device plugin                           |
| WebRTC quality       | WebRTC stats                | Expose via signaling server metrics endpoint                          |

### 8.4 Graceful Shutdown

Crosvm supports graceful shutdown via its control socket:

```bash
crosvm stop <crosvm_socket_path>
```

The pod `preStop` hook should:

1. Send `crosvm stop` to gracefully power off the Android guest
2. Wait for companion processes to drain (logcat, tombstone)
3. Allow Kubernetes to proceed with SIGTERM

```yaml
lifecycle:
  preStop:
    exec:
      command:
        - /bin/sh
        - -c
        - "crosvm stop /path/to/crosvm.sock && sleep 5"
terminationGracePeriodSeconds: 30
```

---

## 9. Decision Matrix

### 9.1 Networking Model

| Criterion           | Option A: CNI (per-pod netns)         | Option B: `hostNetwork: true`       |
| ------------------- | ------------------------------------- | ----------------------------------- |
| TAP isolation       | ✅ Automatic (separate netns)         | ❌ Requires unique instance numbers |
| iptables isolation  | ✅ Per-pod iptables                   | ❌ Shared host iptables             |
| External ADB access | ⚠️ Requires NodePort / port-forward   | ✅ Direct `host:6520+N`             |
| WebRTC UDP ingress  | ⚠️ Complex (NodePort UDP or hostPort) | ✅ Direct                           |
| Port conflict risk  | ✅ None (pod-scoped ports)            | ❌ High without coordination        |
| Security            | ✅ Network namespace isolation        | ❌ Full host network exposure       |
| Migration effort    | ⚠️ Needs ingress setup                | ✅ Minimal from bare-metal          |
| **Recommendation**  | **✅ Preferred for production**       | Acceptable for dev/test             |

### 9.2 Bridge Management

| Criterion               | Per-pod bridges (in pod netns)        | DaemonSet-managed bridges (host) |
| ----------------------- | ------------------------------------- | -------------------------------- |
| Isolation               | ✅ Complete                           | ❌ Shared across pods            |
| Lifecycle               | ✅ Auto-cleanup with pod              | ⚠️ Requires explicit cleanup     |
| Inter-CVD communication | ❌ Not possible (separate L2 domains) | ✅ Bridged CVDs can communicate  |
| Complexity              | ✅ Simple                             | ⚠️ DaemonSet + coordination      |
| **Recommendation**      | **✅ Default choice**                 | Only if inter-CVD L2 needed      |

### 9.3 TAP Creation

| Criterion          | Per-pod TAP creation           | DaemonSet-managed TAPs     |
| ------------------ | ------------------------------ | -------------------------- |
| When to use        | Option A (CNI networking)      | Option B (host networking) |
| Responsibility     | CVD startup scripts inside pod | DaemonSet pre-creates TAPs |
| Cleanup            | Automatic with netns teardown  | DaemonSet must reconcile   |
| **Recommendation** | **✅ Preferred with CNI**      | Only with hostNetwork      |

### 9.4 GPU Rendering

| Criterion            | `guest_swiftshader` (CPU)        | `gfxstream` / `drm_virgl` (GPU)            |
| -------------------- | -------------------------------- | ------------------------------------------ |
| Device plugin needed | ❌ None                          | ✅ `nvidia.com/gpu` or `intel.com/i915`    |
| Fractional sharing   | N/A                              | ⚠️ Requires MIG (NVIDIA) or SR-IOV (Intel) |
| Performance          | ⚠️ Slow rendering                | ✅ Near-native GPU performance             |
| Density impact       | ✅ No GPU contention             | ❌ Limited by GPU count/fractions          |
| Complexity           | ✅ None                          | ⚠️ Driver + device plugin setup            |
| **Recommendation**   | **✅ Start here; simplest path** | Add when rendering perf is critical        |

### 9.5 Storage Model

| Criterion          | `emptyDir` (ephemeral)             | `PVC` (persistent)                |
| ------------------ | ---------------------------------- | --------------------------------- |
| Data persistence   | ❌ Lost on pod deletion            | ✅ Survives pod restart           |
| Startup speed      | ✅ Fast (overlay created fresh)    | ⚠️ PVC attach latency             |
| Storage efficiency | ✅ Overlay only; shared base image | ⚠️ Full copy if not using overlay |
| Use case           | CI/CD, testing, stateless          | Long-running dev environments     |
| **Recommendation** | **✅ Default for most workloads**  | When user state must persist      |

---

## Appendix A: Minimal Pod Spec Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuttlefish-cvd
  labels:
    app: cuttlefish
spec:
  containers:
    - name: cvd
      image: cuttlefish-runner:latest
      env:
        - name: CUTTLEFISH_INSTANCE
          value: "1" # Or injected by Vsock CID Device Plugin
        - name: VSOCK_CID
          value: "3" # Allocated by Device Plugin
      resources:
        requests:
          cpu: "4"
          memory: "4Gi"
          devices.kubevirt.io/kvm: "1"
          # cuttlefish.dev/vsock-cid: "1"   # Custom device plugin
        limits:
          cpu: "4"
          memory: "6Gi"
          devices.kubevirt.io/kvm: "1"
          # cuttlefish.dev/vsock-cid: "1"
      securityContext:
        capabilities:
          add: ["NET_ADMIN", "NET_RAW"]
          drop: ["ALL"]
        seccompProfile:
          type: Unconfined
        runAsUser: 0
      volumeMounts:
        - name: base-images
          mountPath: /opt/android-images
          readOnly: true
        - name: instance-data
          mountPath: /root/cuttlefish
        - name: dev-kvm
          mountPath: /dev/kvm
        - name: dev-vhost-vsock
          mountPath: /dev/vhost-vsock
        - name: dev-vhost-net
          mountPath: /dev/vhost-net
        - name: dev-net-tun
          mountPath: /dev/net/tun
      ports:
        - containerPort: 6520
          name: adb
          protocol: TCP
        - containerPort: 8443
          name: webrtc-sig
          protocol: TCP
  volumes:
    - name: base-images
      hostPath:
        path: /opt/cuttlefish/images
        type: Directory
    - name: instance-data
      emptyDir:
        sizeLimit: 20Gi
    - name: dev-kvm
      hostPath:
        path: /dev/kvm
        type: CharDevice
    - name: dev-vhost-vsock
      hostPath:
        path: /dev/vhost-vsock
        type: CharDevice
    - name: dev-vhost-net
      hostPath:
        path: /dev/vhost-net
        type: CharDevice
    - name: dev-net-tun
      hostPath:
        path: /dev/net/tun
        type: CharDevice
  nodeSelector:
    cuttlefish.dev/kvm-enabled: "true"
  terminationGracePeriodSeconds: 30
```

## Appendix B: Vsock CID Device Plugin — Interface Sketch

```go
// pkg/vsock/plugin.go
type VsockCIDPlugin struct {
    cidPool    *bitmap.Bitmap    // tracks allocated CIDs [3..maxCID]
    maxCID     int               // default: 1024
    mu         sync.Mutex
}

func (p *VsockCIDPlugin) Allocate(req *pluginapi.AllocateRequest) (*pluginapi.AllocateResponse, error) {
    p.mu.Lock()
    defer p.mu.Unlock()

    cid, err := p.cidPool.AllocateNext()  // returns next free CID
    if err != nil {
        return nil, fmt.Errorf("no free vsock CIDs: %w", err)
    }

    return &pluginapi.AllocateResponse{
        ContainerResponses: []*pluginapi.ContainerAllocateResponse{{
            Envs: map[string]string{
                "VSOCK_CID":           strconv.Itoa(cid),
                "CUTTLEFISH_INSTANCE": strconv.Itoa(cid - 2),
            },
            Devices: []*pluginapi.DeviceSpec{{
                HostPath:      "/dev/vhost-vsock",
                ContainerPath: "/dev/vhost-vsock",
                Permissions:   "rw",
            }},
        }},
    }, nil
}
```

## Appendix C: Port Range Summary by Instance Number

For instance `N` (starting at 1), vsock CID `C = N + 2`:

| Port           | Formula        | N=1         | N=2  | N=3  |
| -------------- | -------------- | ----------- | ---- | ---- |
| ADB            | `6520 + N - 1` | 6520        | 6521 | 6522 |
| Tombstone      | `6600 + N - 1` | 6600        | 6601 | 6602 |
| Lights         | `6900 + N - 1` | 6900        | 6901 | 6902 |
| Pica UCI       | `7000 + N - 1` | 7000        | 7001 | 7002 |
| GNSS gRPC      | `7200 + N - 1` | 7200        | 7201 | 7202 |
| RootCanal HCI  | `7300 + N - 1` | 7300        | 7301 | 7302 |
| RootCanal Link | `7400 + N - 1` | 7400        | 7401 | 7402 |
| RootCanal Test | `7500 + N - 1` | 7500        | 7501 | 7502 |
| RootCanal BLE  | `7600 + N - 1` | 7600        | 7601 | 7602 |
| Casimir NCI    | `7800 + N - 1` | 7800        | 7801 | 7802 |
| Casimir RF     | `7900 + N - 1` | 7900        | 7901 | 7902 |
| Sig Server     | `8443 + N - 1` | 8443        | 8444 | 8445 |
| WebRTC TCP     | configurable   | 15550:15599 | —    | —    |
| WebRTC UDP     | configurable   | 15550:15599 | —    | —    |
