# Cuttlefish Configuration JSON Schema Reference

This document provides a comprehensive reference for every field in the Cuttlefish
configuration JSON file (`cuttlefish_config.json`). The configuration is persisted
to disk by `assemble_cvd` and consumed by `run_cvd` and other host tools.

The configuration is backed by a `Json::Value` dictionary and managed by the
`CuttlefishConfig` class defined in
[`cuttlefish_config.h`](../base/cvd/cuttlefish/host/libs/config/cuttlefish_config.h)
and implemented across:

- [`cuttlefish_config.cpp`](../base/cvd/cuttlefish/host/libs/config/cuttlefish_config.cpp) — top-level fields
- [`cuttlefish_config_instance.cpp`](../base/cvd/cuttlefish/host/libs/config/cuttlefish_config_instance.cpp) — per-instance fields
- [`cuttlefish_config_environment.cpp`](../base/cvd/cuttlefish/host/libs/config/cuttlefish_config_environment.cpp) — per-environment fields

CLI flags are defined in
[`assemble_cvd_flags.cpp`](../base/cvd/cuttlefish/host/commands/assemble_cvd/assemble_cvd_flags.cpp)
with defaults from
[`flags_defaults.h`](../base/cvd/cuttlefish/host/commands/assemble_cvd/flags_defaults.h).

---

## Table of Contents

1. [Top-Level Fields](#1-top-level-fields)
   - [Paths & Directories](#11-paths--directories)
   - [VM Manager](#12-vm-manager)
   - [Security](#13-security)
   - [Bluetooth (Rootcanal)](#14-bluetooth-rootcanal)
   - [NFC (Casimir)](#15-nfc-casimir)
   - [UWB (Pica)](#16-uwb-pica)
   - [Network Simulation (Netsim)](#17-network-simulation-netsim)
   - [Metrics](#18-metrics)
   - [Wi-Fi](#19-wi-fi)
   - [Access Point (AP)](#110-access-point-ap)
   - [WebRTC Signaling](#111-webrtc-signaling)
   - [Kernel](#112-kernel)
   - [Host Tools](#113-host-tools)
   - [Debugging](#114-debugging)
   - [Other](#115-other)
2. [Per-Instance Fields](#2-per-instance-fields)
   - [Identity](#21-identity)
   - [System Image Files](#22-system-image-files)
   - [Other OS Images](#23-other-os-images)
   - [Target Zips](#24-target-zips)
   - [Boot & Kernel](#25-boot--kernel)
   - [Hardware (CPU / Memory)](#26-hardware-cpu--memory)
   - [Display](#27-display)
   - [Touchpad](#28-touchpad)
   - [GPU & Graphics](#29-gpu--graphics)
   - [Disk & Data](#210-disk--data)
   - [Networking](#211-networking)
   - [Mobile Network (RIL)](#212-mobile-network-ril)
   - [Bluetooth & UWB](#213-bluetooth--uwb)
   - [Vsock & Session](#214-vsock--session)
   - [Ports](#215-ports)
   - [ADB](#216-adb)
   - [WebRTC](#217-webrtc)
   - [Console & Debugging](#218-console--debugging)
   - [Modem Simulator](#219-modem-simulator)
   - [Feature Toggles](#220-feature-toggles)
   - [Crosvm Options](#221-crosvm-options)
   - [Emulator Binaries](#222-emulator-binaries)
   - [Service Start Flags](#223-service-start-flags)
   - [GNSS](#224-gnss)
   - [Audio](#225-audio)
   - [Automotive / VHAL](#226-automotive--vhal)
   - [Input Devices](#227-input-devices)
   - [Miscellaneous](#228-miscellaneous)
3. [Per-Environment Fields](#3-per-environment-fields)
4. [Enum Reference](#4-enum-reference)
5. [Computed (Non-JSON) Paths](#5-computed-non-json-paths)

---

## 1. Top-Level Fields

These fields are stored at the JSON root and accessed via `CuttlefishConfig` methods.

### 1.1 Paths & Directories

| JSON Key               | Type   | Default            | CLI Flag             | Description                                             |
| ---------------------- | ------ | ------------------ | -------------------- | ------------------------------------------------------- |
| `root_dir`             | string | `$HOME/cuttlefish` | `--instance_dir`     | Root directory for all instance and assembly artifacts. |
| `instances_uds_dir`    | string | _(derived)_        | —                    | Directory for per-instance Unix domain sockets.         |
| `environments_uds_dir` | string | _(derived)_        | —                    | Directory for per-environment Unix domain sockets.      |
| `snapshot_path`        | string | `""`               | `--snapshot_path`    | Path to a snapshot directory for save/restore.          |
| `kvm_path`             | string | `""` _(auto)_      | `--kvm_path`         | Device node for `/dev/kvm`. Uses default if empty.      |
| `vhost_vsock_path`     | string | `""` _(auto)_      | `--vhost_vsock_path` | Device node for vhost-vsock. Ignored for QEMU.          |

> **Note:** `instances_dir`, `assembly_dir`, `environments_dir` are **computed** from `root_dir` and have no dedicated JSON key.

### 1.2 VM Manager

| JSON Key           | Type          | Default                    | CLI Flag             | Description                                       |
| ------------------ | ------------- | -------------------------- | -------------------- | ------------------------------------------------- |
| `vm_manager`       | string (enum) | _(auto)_                   | `--vm_manager`       | Virtual machine manager. See [VmmMode](#vmmmode). |
| `ap_vm_manager`    | string        | `""`                       | —                    | VM manager for the Access Point VM.               |
| `crosvm_binary`    | string        | `HostBinaryPath("crosvm")` | `--crosvm_binary`    | Path to the crosvm binary.                        |
| `gem5_debug_flags` | string        | `""`                       | `--gem5_debug_flags` | Debug flags for gem5.                             |

### 1.3 Security

| JSON Key      | Type     | Default | CLI Flag        | Description                                                   |
| ------------- | -------- | ------- | --------------- | ------------------------------------------------------------- |
| `secure_hals` | string[] | `""`    | `--secure_hals` | List of HALs with host security. See [SecureHal](#securehal). |

### 1.4 Bluetooth (Rootcanal)

| JSON Key                  | Type     | Default | CLI Flag           | Description                         |
| ------------------------- | -------- | ------- | ------------------ | ----------------------------------- |
| `rootcanal_args`          | string[] | `""`    | `--rootcanal_args` | Space-separated args for rootcanal. |
| `rootcanal_hci_port`      | int      | 0       | —                  | HCI port for rootcanal.             |
| `rootcanal_link_port`     | int      | 0       | —                  | Link-layer port for rootcanal.      |
| `rootcanal_link_ble_port` | int      | 0       | —                  | BLE link-layer port for rootcanal.  |
| `rootcanal_test_port`     | int      | 0       | —                  | Test port for rootcanal.            |

### 1.5 NFC (Casimir)

| JSON Key                    | Type     | Default | CLI Flag                 | Description                        |
| --------------------------- | -------- | ------- | ------------------------ | ---------------------------------- |
| `enable_host_nfc`           | bool     | `true`  | `--enable_host_nfc`      | Enable NFC emulator on host.       |
| `enable_host_nfc_connector` | bool     | `false` | —                        | Enable NFC connector.              |
| `casimir_instance_num`      | int      | `0`     | `--casimir_instance_num` | Casimir instance number (0 = new). |
| `casimir_args`              | string[] | `""`    | `--casimir_args`         | Space-separated args for casimir.  |
| `casimir_nci_port`          | int      | 0       | —                        | NCI port for casimir.              |
| `casimir_rf_port`           | int      | 0       | —                        | RF port for casimir.               |

### 1.6 UWB (Pica)

| JSON Key          | Type | Default | CLI Flag            | Description                    |
| ----------------- | ---- | ------- | ------------------- | ------------------------------ |
| `enable_host_uwb` | bool | `true`  | `--enable_host_uwb` | Enable UWB host and connector. |
| `pica_uci_port`   | int  | 0       | —                   | UCI port for pica UWB.         |

### 1.7 Network Simulation (Netsim)

| JSON Key                        | Type          | Default | CLI Flag                                  | Description                                                                             |
| ------------------------------- | ------------- | ------- | ----------------------------------------- | --------------------------------------------------------------------------------------- |
| `netsim_radios`                 | int (bitmask) | `0`     | `--netsim`, `--netsim_bt`, `--netsim_uwb` | Bitmask of radios connected to netsim. Bits: `0x01`=Bluetooth, `0x02`=Wifi, `0x04`=UWB. |
| `netsim_instance_num`           | int           | 0       | —                                         | Netsim instance number.                                                                 |
| `netsim_connector_instance_num` | int           | 0       | —                                         | Netsim connector instance number.                                                       |
| `netsim_args`                   | string[]      | `""`    | `--netsim_args`                           | Space-separated args for netsim.                                                        |

### 1.8 Metrics

| JSON Key         | Type       | Default       | CLI Flag                         | Description                                      |
| ---------------- | ---------- | ------------- | -------------------------------- | ------------------------------------------------ |
| `enable_metrics` | int (enum) | `0` (Unknown) | `--report_anonymous_usage_stats` | Metrics reporting. `0`=Unknown, `1`=Yes, `2`=No. |
| `metrics_binary` | string     | `""`          | —                                | Path to metrics collection binary.               |

### 1.9 Wi-Fi

| JSON Key                | Type | Default | CLI Flag | Description                   |
| ----------------------- | ---- | ------- | -------- | ----------------------------- |
| `virtio_mac80211_hwsim` | bool | `false` | —        | Enable virtio mac80211_hwsim. |

### 1.10 Access Point (AP)

| JSON Key          | Type   | Default | CLI Flag            | Description                          |
| ----------------- | ------ | ------- | ------------------- | ------------------------------------ |
| `ap_rootfs_image` | string | `""`    | `--ap_rootfs_image` | Root filesystem image for the AP VM. |
| `ap_kernel_image` | string | `""`    | `--ap_kernel_image` | Kernel image for the AP VM.          |

### 1.11 WebRTC Signaling

| JSON Key                 | Type   | Default                      | CLI Flag                   | Description                         |
| ------------------------ | ------ | ---------------------------- | -------------------------- | ----------------------------------- |
| `webrtc_sig_server_port` | int    | 0                            | —                          | WebRTC signaling server proxy port. |
| `webrtc_sig_server_addr` | string | `"/run/cuttlefish/operator"` | `--webrtc_sig_server_addr` | WebRTC signaling server address.    |

### 1.12 Kernel

| JSON Key               | Type     | Default | CLI Flag | Description                          |
| ---------------------- | -------- | ------- | -------- | ------------------------------------ |
| `extra_kernel_cmdline` | string[] | `""`    | —        | Extra kernel command-line arguments. |

### 1.13 Host Tools

| JSON Key                   | Type     | Default | CLI Flag                     | Description                                         |
| -------------------------- | -------- | ------- | ---------------------------- | --------------------------------------------------- |
| `host_tools_version`       | object   | `{}`    | —                            | Map of tool name → CRC version for change tracking. |
| `straced_host_executables` | string[] | `""`    | `--straced_host_executables` | Host executables to run under strace.               |

### 1.14 Debugging

| JSON Key                  | Type | Default | CLI Flag                    | Description                      |
| ------------------------- | ---- | ------- | --------------------------- | -------------------------------- |
| `enable_automotive_proxy` | bool | `false` | `--enable_automotive_proxy` | Enable automotive proxy service. |

### 1.15 Other

| JSON Key         | Type     | Default     | CLI Flag | Description                                       |
| ---------------- | -------- | ----------- | -------- | ------------------------------------------------- |
| `instance_names` | string[] | _(derived)_ | —        | Stable list of instance names (e.g. `["cvd-1"]`). |
| `fragments`      | object   | `{}`        | —        | Extension point for `ConfigFragment` plug-ins.    |

---

## 2. Per-Instance Fields

Each instance is stored under `instances.<id>` in the JSON, where `<id>` is the
instance number as a string (e.g. `"1"`). These are accessed via
`CuttlefishConfig::InstanceSpecific`.

### 2.1 Identity

| JSON Key           | Type   | Default             | CLI Flag          | Description                                       |
| ------------------ | ------ | ------------------- | ----------------- | ------------------------------------------------- |
| `instance_dir`     | string | _(computed)_        | —                 | Legacy directory path for acloud compatibility.   |
| `serial_number`    | string | `"CUTTLEFISHCVD01"` | `--serial_number` | Device serial number.                             |
| `uuid`             | string | _(generated)_       | `--uuid`          | Device UUID.                                      |
| `environment_name` | string | _(derived)_         | —                 | Name of the environment this instance belongs to. |

### 2.2 System Image Files

| JSON Key                       | Type   | Default     | CLI Flag                     | Description                           |
| ------------------------------ | ------ | ----------- | ---------------------------- | ------------------------------------- |
| `images_dir`                   | string | _(derived)_ | —                            | Directory containing Android images.  |
| `boot_image`                   | string | _(auto)_    | —                            | Path to `boot.img`.                   |
| `new_boot_image`               | string | _(auto)_    | —                            | Path to the rewritten `boot.img`.     |
| `init_boot_image`              | string | _(auto)_    | —                            | Path to `init_boot.img`.              |
| `data_image`                   | string | _(auto)_    | —                            | Path to `userdata.img`.               |
| `new_data_image`               | string | _(auto)_    | —                            | Path to the new data image.           |
| `super_image`                  | string | _(auto)_    | —                            | Path to `super.img`.                  |
| `new_super_image`              | string | _(auto)_    | —                            | Path to the rewritten `super.img`.    |
| `vendor_boot_image`            | string | _(auto)_    | —                            | Path to `vendor_boot.img`.            |
| `new_vendor_boot_image`        | string | _(auto)_    | —                            | Path to rewritten `vendor_boot.img`.  |
| `vbmeta_image`                 | string | _(auto)_    | `--vbmeta_image`             | Path to `vbmeta.img`.                 |
| `new_vbmeta_image`             | string | _(auto)_    | —                            | Path to rewritten `vbmeta.img`.       |
| `vbmeta_system_image`          | string | _(auto)_    | `--vbmeta_system_image`      | Path to `vbmeta_system.img`.          |
| `vbmeta_vendor_dlkm_image`     | string | _(auto)_    | `--vbmeta_vendor_dlkm_image` | Path to `vbmeta_vendor_dlkm.img`.     |
| `new_vbmeta_vendor_dlkm_image` | string | _(auto)_    | —                            | Path to rewritten vendor DLKM vbmeta. |
| `vbmeta_system_dlkm_image`     | string | _(auto)_    | `--vbmeta_system_dlkm_image` | Path to `vbmeta_system_dlkm.img`.     |
| `new_vbmeta_system_dlkm_image` | string | _(auto)_    | —                            | Path to rewritten system DLKM vbmeta. |
| `vvmtruststore_path`           | string | `""`        | `--vvmtruststore_path`       | Path to VVM trust store image.        |

### 2.3 Other OS Images

| JSON Key                | Type   | Default | CLI Flag                       | Description                                     |
| ----------------------- | ------ | ------- | ------------------------------ | ----------------------------------------------- |
| `otheros_esp_image`     | string | `""`    | —                              | ESP image for OtherOS boot flow.                |
| `android_efi_loader`    | string | `""`    | —                              | Android EFI loader for EFI boot flow.           |
| `chromeos_disk`         | string | `""`    | `--chromeos_disk`              | Complete ChromeOS GPT disk image.               |
| `chromeos_kernel_path`  | string | `""`    | `--chromeos_kernel_path`       | ChromeOS kernel image.                          |
| `chromeos_root_image`   | string | `""`    | `--chromeos_root_image`        | ChromeOS root filesystem image.                 |
| `linux_kernel_path`     | string | `""`    | `--linux_kernel_path`          | Linux kernel for OtherOS flow.                  |
| `linux_initramfs_path`  | string | `""`    | `--linux_initramfs_path`       | Linux initramfs for OtherOS flow.               |
| `linux_root_image`      | string | `""`    | `--linux_root_image`           | Linux root filesystem image.                    |
| `fuchsia_zedboot_path`  | string | `""`    | `--fuchsia_zedboot_path`       | Fuchsia zedboot image.                          |
| `multiboot_bin_path`    | string | `""`    | `--fuchsia_multiboot_bin_path` | Fuchsia multiboot binary.                       |
| `fuchsia_root_image`    | string | `""`    | `--fuchsia_root_image`         | Fuchsia root filesystem image.                  |
| `custom_partition_path` | string | `""`    | `--custom_partition_path`      | Custom partition image(s), semicolon-separated. |

### 2.4 Target Zips

| JSON Key             | Type   | Default | CLI Flag               | Description                       |
| -------------------- | ------ | ------- | ---------------------- | --------------------------------- |
| `default_target_zip` | string | `""`    | `--default_target_zip` | Default target zip file location. |
| `system_target_zip`  | string | `""`    | `--system_target_zip`  | System target zip file location.  |

### 2.5 Boot & Kernel

| JSON Key                   | Type   | Default  | CLI Flag                  | Description                                    |
| -------------------------- | ------ | -------- | ------------------------- | ---------------------------------------------- |
| `kernel_path`              | string | _(auto)_ | —                         | Path to the kernel image.                      |
| `initramfs_path`           | string | _(auto)_ | —                         | Path to the initramfs image.                   |
| `bootloader`               | string | _(auto)_ | —                         | Path to the bootloader binary.                 |
| `boot_slot`                | string | `""`     | `--boot_slot`             | Force boot slot (`a` or `b`). Empty = auto.    |
| `bootconfig_supported`     | bool   | _(auto)_ | —                         | Whether bootconfig is supported by the kernel. |
| `extra_bootconfig_args`    | string | `""`     | `--extra_bootconfig_args` | Space-separated extra bootconfig arguments.    |
| `guest_android_version`    | string | _(auto)_ | —                         | Android version string of the guest.           |
| `filename_encryption_mode` | string | _(auto)_ | —                         | Filesystem filename encryption mode.           |

### 2.6 Hardware (CPU / Memory)

| JSON Key           | Type       | Default  | CLI Flag             | Description                                          |
| ------------------ | ---------- | -------- | -------------------- | ---------------------------------------------------- |
| `cpus`             | int        | `4`      | —                    | Number of virtual CPUs.                              |
| `memory_mb`        | int        | _(auto)_ | —                    | Total guest RAM in megabytes.                        |
| `ddr_mem_mb`       | int        | 0        | —                    | DDR memory in MB (for gem5).                         |
| `target_arch`      | int (enum) | _(auto)_ | —                    | Target architecture. See [Arch](#arch).              |
| `device_type`      | int (enum) | _(auto)_ | —                    | Device type. See [DeviceType](#devicetype).          |
| `vcpu_config_path` | string     | `""`     | `--vcpu_config_path` | Configuration file for virtual CPU frequency.        |
| `smt`              | bool       | `false`  | `--smt`              | Enable simultaneous multithreading (HyperThreading). |
| `mte`              | bool       | `false`  | `--mte`              | Enable Memory Tagging Extension (ARM).               |

### 2.7 Display

| JSON Key          | Type     | Default                      | CLI Flag                                                                        | Description                      |
| ----------------- | -------- | ---------------------------- | ------------------------------------------------------------------------------- | -------------------------------- |
| `display_configs` | object[] | `[{720, 1280, 320, 60, ""}]` | `--display0` … `--display3`, `--x_res`, `--y_res`, `--dpi`, `--refresh_rate_hz` | Array of display configurations. |

Each element of `display_configs` has the following sub-fields:

| Sub-Key           | Type   | Default | Description                         |
| ----------------- | ------ | ------- | ----------------------------------- |
| `x_res`           | int    | `720`   | Display width in pixels.            |
| `y_res`           | int    | `1280`  | Display height in pixels.           |
| `dpi`             | int    | `320`   | Pixels per inch.                    |
| `refresh_rate_hz` | int    | `60`    | Refresh rate in Hertz.              |
| `overlays`        | string | `""`    | Display overlay composition config. |

> **CLI format:** `--display0=width=720,height=1280,dpi=320,refresh_rate_hz=60`

### 2.8 Touchpad

| JSON Key           | Type     | Default | CLI Flag | Description                       |
| ------------------ | -------- | ------- | -------- | --------------------------------- |
| `touchpad_configs` | object[] | `[]`    | —        | Array of touchpad configurations. |

Each element:

| Sub-Key | Type | Description      |
| ------- | ---- | ---------------- |
| `x_res` | int  | Touchpad width.  |
| `y_res` | int  | Touchpad height. |

### 2.9 GPU & Graphics

| JSON Key                               | Type          | Default               | CLI Flag                   | Description                                                                      |
| -------------------------------------- | ------------- | --------------------- | -------------------------- | -------------------------------------------------------------------------------- |
| `gpu_mode`                             | string (enum) | `"guest_swiftshader"` | —                          | GPU rendering mode. See [GpuMode](#gpumode).                                     |
| `gpu_angle_feature_overrides_enabled`  | string        | `""`                  | —                          | Enabled ANGLE feature overrides.                                                 |
| `gpu_angle_feature_overrides_disabled` | string        | `""`                  | —                          | Disabled ANGLE feature overrides.                                                |
| `gpu_capture_binary`                   | string        | `""`                  | `--gpu_capture_binary`     | Path to GPU capture tool (renderdoc, etc.).                                      |
| `gpu_gfxstream_transport`              | string        | `""`                  | —                          | Gfxstream transport mode.                                                        |
| `gpu_renderer_features`                | string        | `""`                  | `--gpu_renderer_features`  | Renderer-specific feature flags (semicolon-separated).                           |
| `gpu_context_types`                    | string        | `""`                  | `--gpu_context_types`      | Colon-separated list of virtio-gpu context types.                                |
| `guest_hwui_renderer`                  | string (enum) | `"auto"`              | `--guest_hwui_renderer`    | Guest HWUI renderer. See [GuestHwuiRenderer](#guesthwuirenderer).                |
| `guest_renderer_preload`               | string (enum) | `"auto"`              | `--guest_renderer_preload` | Zygote renderer preload mode. See [GuestRendererPreload](#guestrendererpreload). |
| `guest_vulkan_driver`                  | string        | `""`                  | `--guest_vulkan_driver`    | Guest Vulkan driver (e.g. `"ranchu"`).                                           |
| `guest_uses_bgra_framebuffers`         | bool          | `false`               | —                          | Whether the guest uses BGRA framebuffers.                                        |
| `enable_gpu_udmabuf`                   | bool          | `false`               | `--enable_gpu_udmabuf`     | Use udmabuf driver for zero-copy virtio-gpu.                                     |
| `enable_gpu_vhost_user`                | bool          | `false`               | —                          | Run GPU worker in a separate vhost-user process.                                 |
| `enable_gpu_external_blob`             | bool          | `false`               | —                          | Enable GPU external blob support.                                                |
| `enable_gpu_system_blob`               | bool          | `false`               | —                          | Enable GPU system blob support.                                                  |
| `hwcomposer`                           | string        | `"auto"`              | `--hwcomposer`             | HW composer implementation. One of `auto`, `drm`, `ranchu`.                      |
| `frame_sock_path`                      | string        | `""`                  | `--frames_socket_path`     | Frame socket path (e.g. Wayland socket).                                         |

### 2.10 Disk & Data

| JSON Key                | Type          | Default          | CLI Flag                  | Description                                                              |
| ----------------------- | ------------- | ---------------- | ------------------------- | ------------------------------------------------------------------------ |
| `data_policy`           | string (enum) | `"use_existing"` | —                         | Data partition handling policy. See [DataImagePolicy](#dataimagepolicy). |
| `blank_data_image_mb`   | int           | `"8192"`         | —                         | Size of blank data image in MB.                                          |
| `blank_sdcard_image_mb` | int           | `"2048"`         | `--blank_sdcard_image_mb` | Size of blank SD card image in MB.                                       |
| `userdata_format`       | string        | `"f2fs"`         | `--userdata_format`       | Userdata filesystem format (`f2fs` or `ext4`).                           |
| `use_sdcard`            | bool          | `false`          | `--use_sdcard`            | Whether to create and expose an SD card image.                           |
| `virtual_disk_paths`    | string[]      | _(auto)_         | —                         | Ordered list of virtual disk image paths for the VM.                     |

### 2.11 Networking

| JSON Key                | Type          | Default  | CLI Flag                    | Description                                                                          |
| ----------------------- | ------------- | -------- | --------------------------- | ------------------------------------------------------------------------------------ |
| `mobile_bridge_name`    | string        | _(auto)_ | —                           | Bridge device name for mobile network.                                               |
| `mobile_tap_name`       | string        | _(auto)_ | —                           | TAP device name for mobile network.                                                  |
| `mobile_mac`            | string        | _(auto)_ | —                           | MAC address for mobile interface.                                                    |
| `wifi_tap_name`         | string        | _(auto)_ | —                           | TAP device name for Wi-Fi.                                                           |
| `wifi_bridge_name`      | string        | _(auto)_ | —                           | Bridge device name for Wi-Fi.                                                        |
| `wifi_mac`              | string        | _(auto)_ | —                           | MAC address for Wi-Fi interface.                                                     |
| `has_wifi_card`         | bool          | `true`   | —                           | Whether the device has a Wi-Fi card.                                                 |
| `use_bridged_wifi_tap`  | bool          | `false`  | —                           | Whether to use a bridged Wi-Fi TAP.                                                  |
| `ethernet_tap_name`     | string        | _(auto)_ | —                           | TAP device name for Ethernet.                                                        |
| `ethernet_bridge_name`  | string        | _(auto)_ | —                           | Bridge device name for Ethernet.                                                     |
| `ethernet_mac`          | string        | _(auto)_ | —                           | MAC address for Ethernet interface.                                                  |
| `ethernet_ipv6`         | string        | _(auto)_ | —                           | IPv6 address for Ethernet.                                                           |
| `external_network_mode` | string (enum) | `"tap"`  | `--device_external_network` | How to connect to external network. See [ExternalNetworkMode](#externalnetworkmode). |
| `wifi_mac_prefix`       | int           | 0        | —                           | Prefix for guest Wi-Fi MAC address.                                                  |
| `enable_tap_devices`    | bool          | `true`   | `--enable_tap_devices`      | Whether to use TAP devices for networking.                                           |

### 2.12 Mobile Network (RIL)

| JSON Key        | Type   | Default     | CLI Flag    | Description                    |
| --------------- | ------ | ----------- | ----------- | ------------------------------ |
| `ril_dns`       | string | `"8.8.8.8"` | `--ril_dns` | DNS server for mobile network. |
| `ril_ipaddr`    | string | _(auto)_    | —           | IP address for RIL.            |
| `ril_gateway`   | string | _(auto)_    | —           | Gateway for RIL.               |
| `ril_broadcast` | string | _(auto)_    | —           | Broadcast address for RIL.     |
| `ril_prefixlen` | uint8  | _(auto)_    | —           | Subnet prefix length for RIL.  |

### 2.13 Bluetooth & UWB

| JSON Key                          | Type | Default  | CLI Flag                  | Description                               |
| --------------------------------- | ---- | -------- | ------------------------- | ----------------------------------------- |
| `has_bluetooth`                   | bool | `true`   | —                         | Whether the device has Bluetooth.         |
| `enable_host_bluetooth_connector` | bool | _(auto)_ | `--enable_host_bluetooth` | Enable Bluetooth connector and rootcanal. |
| `enable_host_uwb_connector`       | bool | _(auto)_ | —                         | Enable UWB connector.                     |

### 2.14 Vsock & Session

| JSON Key            | Type   | Default              | CLI Flag              | Description                                      |
| ------------------- | ------ | -------------------- | --------------------- | ------------------------------------------------ |
| `session_id`        | uint32 | _(auto)_             | —                     | Session identifier.                              |
| `use_cvdalloc`      | bool   | `false`              | —                     | Whether to use cvdalloc for resource allocation. |
| `vsock_guest_cid`   | int    | _(auto: 2+instance)_ | `--vsock_guest_cid`   | Guest vsock Context ID.                          |
| `vsock_guest_group` | string | `""`                 | `--vsock_guest_group` | Vsock isolation group name.                      |

### 2.15 Ports

| JSON Key                      | Type | Default  | CLI Flag               | Description                               |
| ----------------------------- | ---- | -------- | ---------------------- | ----------------------------------------- |
| `adb_host_port`               | int  | _(auto)_ | —                      | Host port for ADB connections.            |
| `fastboot_host_port`          | int  | _(auto)_ | —                      | Host port for fastboot connections.       |
| `qemu_vnc_server_port`        | int  | 0        | —                      | VNC server port (QEMU only).              |
| `tombstone_receiver_port`     | int  | _(auto)_ | —                      | Port for tombstone receiver.              |
| `config_server_port`          | int  | _(auto)_ | —                      | Port for config server.                   |
| `audiocontrol_server_port`    | int  | _(auto)_ | —                      | Port for audio control server.            |
| `lights_server_port`          | int  | 0        | —                      | Port for lights HAL server.               |
| `camera_server_port`          | int  | `0`      | `--camera_server_port` | Port for camera HAL server.               |
| `modem_simulator_host_id`     | int  | _(auto)_ | —                      | 4-digit modem simulator ID.               |
| `gnss_grpc_proxy_server_port` | int  | _(auto)_ | —                      | Port for GNSS gRPC proxy.                 |
| `gdb_port`                    | int  | `0`      | `--gdb_port`           | Port for kernel GDB stub. `0` = disabled. |

### 2.16 ADB

| JSON Key          | Type   | Default  | CLI Flag | Description                                      |
| ----------------- | ------ | -------- | -------- | ------------------------------------------------ |
| `adb_ip_and_port` | string | _(auto)_ | —        | ADB connection string (e.g. `"127.0.0.1:6520"`). |

### 2.17 WebRTC

| JSON Key                | Type       | Default                     | CLI Flag              | Description                                                               |
| ----------------------- | ---------- | --------------------------- | --------------------- | ------------------------------------------------------------------------- |
| `webrtc_device_id`      | string     | `"cvd-{num}"`               | `--webrtc_device_id`  | Device ID for WebRTC signaling. `{num}` is replaced with instance number. |
| `webrtc_assets_dir`     | string     | `"usr/share/webrtc/assets"` | `--webrtc_assets_dir` | Path to WebRTC webpage assets.                                            |
| `webrtc_tcp_port_range` | [int, int] | `[15550, 15599]`            | `--tcp_port_range`    | Min/max TCP ports for WebRTC ICE candidates.                              |
| `webrtc_udp_port_range` | [int, int] | `[15550, 15599]`            | `--udp_port_range`    | Min/max UDP ports for WebRTC ICE candidates.                              |

### 2.18 Console & Debugging

| JSON Key            | Type   | Default  | CLI Flag              | Description                          |
| ------------------- | ------ | -------- | --------------------- | ------------------------------------ |
| `console`           | bool   | `false`  | `--console`           | Enable serial console.               |
| `kgdb`              | bool   | `false`  | `--kgdb`              | Enable kernel debugger (kgdb/kdb).   |
| `enable_kernel_log` | bool   | `true`   | `--enable_kernel_log` | Enable kernel console/dmesg logging. |
| `enable_sandbox`    | bool   | _(auto)_ | `--enable_sandbox`    | Enable crosvm sandbox (seccomp).     |
| `enable_virtiofs`   | bool   | `false`  | `--enable_virtiofs`   | Enable shared folder via virtiofs.   |
| `record_screen`     | bool   | `false`  | `--record_screen`     | Enable screen recording.             |
| `fail_fast`         | bool   | `false`  | `--fail_fast`         | Exit if boot is predicted to fail.   |
| `gem5_debug_file`   | string | `""`     | `--gem5_debug_file`   | Gem5 debug output file.              |

### 2.19 Modem Simulator

| JSON Key                          | Type   | Default  | CLI Flag                     | Description                                  |
| --------------------------------- | ------ | -------- | ---------------------------- | -------------------------------------------- |
| `enable_modem_simulator`          | bool   | `true`   | `--enable_modem_simulator`   | Enable modem simulator for RILD AT commands. |
| `modem_simulator_instance_number` | int    | `1`      | —                            | Number of modem simulator instances.         |
| `modem_simulator_sim_type`        | int    | `1`      | `--modem_simulator_sim_type` | SIM type: `1` = normal, `2` = CTS carrier.   |
| `modem_simulator_ports`           | string | _(auto)_ | —                            | Comma-separated modem simulator port list.   |

### 2.20 Feature Toggles

| JSON Key                 | Type | Default  | CLI Flag                   | Description                             |
| ------------------------ | ---- | -------- | -------------------------- | --------------------------------------- |
| `enable_audio`           | bool | `true`   | `--enable_audio`           | Enable audio playback/capture.          |
| `enable_mouse`           | bool | _(auto)_ | —                          | Enable mouse input device.              |
| `enable_gamepad`         | bool | _(auto)_ | —                          | Enable gamepad input device.            |
| `enable_usb`             | bool | `false`  | `--enable_usb`             | Enable USB passthrough.                 |
| `enable_gnss_grpc_proxy` | bool | `true`   | `--start_gnss_proxy`       | Enable GNSS gRPC proxy.                 |
| `enable_bootanimation`   | bool | `true`   | `--enable_bootanimation`   | Enable boot animation.                  |
| `enable_minimal_mode`    | bool | `false`  | `--enable_minimal_mode`    | Minimal device mode (reduced features). |
| `enable_jcard_simulator` | bool | `false`  | `--enable_jcard_simulator` | Enable Java Card simulator.             |
| `guest_enforce_security` | bool | `true`   | —                          | Enforce SELinux in guest.               |
| `restart_subprocesses`   | bool | _(auto)_ | —                          | Restart crashed subprocesses.           |
| `pause_in_bootloader`    | bool | `false`  | `--pause_in_bootloader`    | Pause boot in U-Boot.                   |
| `run_as_daemon`          | bool | _(auto)_ | —                          | Run as a daemon process.                |

### 2.21 Crosvm Options

| JSON Key                     | Type   | Default  | CLI Flag                       | Description                                          |
| ---------------------------- | ------ | -------- | ------------------------------ | ---------------------------------------------------- |
| `crosvm_binary`              | string | _(auto)_ | —                              | Per-instance override for crosvm binary path.        |
| `crosvm_use_balloon`         | bool   | `true`   | `--crosvm_use_balloon`         | Enable crosvm balloon device.                        |
| `crosvm_use_rng`             | bool   | `true`   | `--crosvm_use_rng`             | Enable crosvm RNG device.                            |
| `crosvm_simple_media_device` | bool   | `false`  | `--crosvm_simple_media_device` | Use simple media device in crosvm.                   |
| `crosvm_v4l2_proxy`          | string | `""`     | `--crosvm_v4l2_proxy`          | Path for V4L2 proxy (overrides simple media device). |
| `use_pmem`                   | bool   | `true`   | `--use_pmem`                   | Enable persistent memory (pmem) in crosvm.           |
| `vhost_net`                  | bool   | `false`  | `--vhost_net`                  | Enable vhost-net acceleration.                       |
| `vhost_user_vsock`           | bool   | `false`  | `--vhost_user_vsock`           | Enable vhost-user-vsock.                             |
| `vhost_user_block`           | bool   | `false`  | `--vhost_user_block`           | Enable vhost-user block device.                      |

### 2.22 Emulator Binaries

| JSON Key              | Type   | Default  | CLI Flag                | Description                                  |
| --------------------- | ------ | -------- | ----------------------- | -------------------------------------------- |
| `seccomp_policy_dir`  | string | _(auto)_ | `--seccomp_policy_dir`  | Seccomp policy directory for crosvm sandbox. |
| `qemu_binary_dir`     | string | _(auto)_ | `--qemu_binary_dir`     | Path to QEMU binaries directory.             |
| `gem5_binary_dir`     | string | _(auto)_ | `--gem5_binary_dir`     | Path to gem5 build tree root.                |
| `gem5_checkpoint_dir` | string | `""`     | `--gem5_checkpoint_dir` | Path to gem5 checkpoint directory.           |

### 2.23 Service Start Flags

| JSON Key                  | Type | Default  | CLI Flag | Description                                 |
| ------------------------- | ---- | -------- | -------- | ------------------------------------------- |
| `start_rootcanal`         | bool | _(auto)_ | —        | Whether this instance starts rootcanal.     |
| `start_casimir`           | bool | _(auto)_ | —        | Whether this instance starts casimir (NFC). |
| `start_pica`              | bool | _(auto)_ | —        | Whether this instance starts pica (UWB).    |
| `start_netsim`            | bool | _(auto)_ | —        | Whether this instance starts netsim.        |
| `start_wmediumd_instance` | bool | _(auto)_ | —        | Whether this instance starts wmediumd.      |

### 2.24 GNSS

| JSON Key                   | Type   | Default | CLI Flag                     | Description                         |
| -------------------------- | ------ | ------- | ---------------------------- | ----------------------------------- |
| `gnss_file_path`           | string | `""`    | `--gnss_file_path`           | Local GNSS raw measurement file.    |
| `fixed_location_file_path` | string | `""`    | `--fixed_location_file_path` | Fixed location file for GNSS proxy. |

### 2.25 Audio

| JSON Key                     | Type   | Default | CLI Flag | Description                               |
| ---------------------------- | ------ | ------- | -------- | ----------------------------------------- |
| `audio_output_streams_count` | int    | 0       | —        | Number of audio output streams.           |
| `audio_settings_textproto`   | string | `""`    | —        | Audio settings as a text-format protobuf. |

### 2.26 Automotive / VHAL

| JSON Key                   | Type | Default | CLI Flag                     | Description               |
| -------------------------- | ---- | ------- | ---------------------------- | ------------------------- |
| `enable_vhal_proxy_server` | bool | `false` | `--enable_vhal_proxy_server` | Enable VHAL proxy server. |
| `vhal_proxy_server_port`   | int  | 0       | —                            | VHAL proxy server port.   |

### 2.27 Input Devices

| JSON Key                 | Type              | Default | CLI Flag | Description                                  |
| ------------------------ | ----------------- | ------- | -------- | -------------------------------------------- |
| `custom_keyboard_config` | string (nullable) | `null`  | —        | Path to custom keyboard layout JSON.         |
| `domkey_mapping_config`  | object            | `{}`    | —        | DOM key mapping configuration (inline JSON). |

### 2.28 Miscellaneous

| JSON Key              | Type       | Default      | CLI Flag                | Description                                               |
| --------------------- | ---------- | ------------ | ----------------------- | --------------------------------------------------------- |
| `setupwizard_mode`    | string     | `"DISABLED"` | `--setupwizard_mode`    | Setup wizard mode: `DISABLED`, `OPTIONAL`, or `REQUIRED`. |
| `ap_boot_flow`        | int (enum) | _(auto)_     | —                       | AP boot flow mode. See [APBootFlow](#apbootflow).         |
| `mcu`                 | object     | `{}`         | —                       | MCU (Microcontroller Unit) configuration.                 |
| `ti50`                | string     | `""`         | —                       | Ti50 security chip emulator path.                         |
| `openthread_node_id`  | int        | 0            | —                       | OpenThread node ID.                                       |
| `grpc_config`         | string     | _(auto)_     | —                       | gRPC socket path.                                         |
| `gpu_vhost_user_mode` | string     | `"auto"`     | `--gpu_vhost_user_mode` | Vhost-user GPU mode: `auto`, `on`, `off`.                 |

---

## 3. Per-Environment Fields

Stored under a separate `environments` section, accessed via `EnvironmentSpecific`.

| JSON Key                     | Type   | Default  | CLI Flag                      | Description                                           |
| ---------------------------- | ------ | -------- | ----------------------------- | ----------------------------------------------------- |
| `enable_wifi`                | bool   | `true`   | `--enable_wifi`               | Enable guest Wi-Fi.                                   |
| `start_wmediumd`             | bool   | _(auto)_ | —                             | Whether to start wmediumd for this environment.       |
| `vhost_user_mac80211_hwsim`  | string | `""`     | `--vhost_user_mac80211_hwsim` | Unix socket for vhost-user mac80211_hwsim (wmediumd). |
| `wmediumd_api_server_socket` | string | _(auto)_ | —                             | Wmediumd API server Unix socket path.                 |
| `wmediumd_config`            | string | _(auto)_ | `--wmediumd_config`           | Path to wmediumd configuration file.                  |
| `wmediumd_mac_prefix`        | int    | 0        | —                             | MAC address prefix for wmediumd.                      |
| `group_uuid`                 | int    | 0        | —                             | Group UUID for environment isolation.                 |

---

## 4. Enum Reference

### VmmMode

Virtual Machine Manager selection.

| Value      | Description                                  |
| ---------- | -------------------------------------------- |
| `crosvm`   | Chrome OS Virtual Machine Monitor (default). |
| `qemu_cli` | QEMU command-line interface.                 |
| `gem5`     | gem5 full-system simulator.                  |
| `unknown`  | Unknown / unrecognized.                      |

### GpuMode

GPU rendering backend.

| Value               | Description                                                |
| ------------------- | ---------------------------------------------------------- |
| `auto`              | Automatic selection based on host capabilities.            |
| `drm_virgl`         | Virgl-based GPU acceleration via DRM.                      |
| `gfxstream`         | Gfxstream GPU passthrough.                                 |
| `guest_swiftshader` | Software rendering via SwiftShader in guest.               |
| `custom`            | Custom GPU context types (use with `--gpu_context_types`). |
| `none`              | No GPU.                                                    |

### BootFlow

Determined automatically from which OS image paths are set.

| Value              | Trigger                                                                                   |
| ------------------ | ----------------------------------------------------------------------------------------- |
| `Android`          | Default when no other OS images are specified.                                            |
| `AndroidEfiLoader` | `android_efi_loader` is non-empty.                                                        |
| `ChromeOs`         | `chromeos_kernel_path` or `chromeos_root_image` is set.                                   |
| `ChromeOsDisk`     | `chromeos_disk` is set.                                                                   |
| `Linux`            | Any of `linux_kernel_path`, `linux_initramfs_path`, `linux_root_image` is set.            |
| `Fuchsia`          | Any of `fuchsia_zedboot_path`, `fuchsia_root_image`, `fuchsia_multiboot_bin_path` is set. |

### APBootFlow

Access Point VM boot mode.

| Value          | Description                                      |
| -------------- | ------------------------------------------------ |
| `None`         | AP not started (e.g. not the first instance).    |
| `Grub`         | Generate ESP and use U-Boot/GRUB to boot the AP. |
| `LegacyDirect` | Legacy direct boot without ESP.                  |

### DataImagePolicy

How the userdata partition is handled.

| Value               | Description                                        |
| ------------------- | -------------------------------------------------- |
| `use_existing`      | Use existing data image as-is.                     |
| `always_create`     | Always create a fresh data image.                  |
| `create_if_missing` | Create only if the image does not exist.           |
| `resize_up_to`      | Resize existing image up to `blank_data_image_mb`. |

### ExternalNetworkMode

Mechanism for connecting the guest to the external network.

| Value     | Description                     |
| --------- | ------------------------------- |
| `tap`     | Use TAP devices (default).      |
| `slirp`   | Use SLIRP user-mode networking. |
| `unknown` | Unknown / unrecognized.         |

### SecureHal

HALs that can be backed by host security features.

| Value              | Description                         |
| ------------------ | ----------------------------------- |
| `keymint`          | KeyMint key management.             |
| `gatekeeper`       | Gatekeeper credential verification. |
| `oemlock`          | OEM lock service.                   |
| `confirmationui`   | Trusted UI confirmation.            |
| `guest_keymint`    | Guest-side KeyMint.                 |
| `guest_gatekeeper` | Guest-side Gatekeeper.              |
| `guest_oemlock`    | Guest-side OEM lock.                |

### GuestHwuiRenderer

HWUI rendering backend inside the guest.

| Value    | Description           |
| -------- | --------------------- |
| `auto`   | Automatic selection.  |
| `skiagl` | Skia OpenGL renderer. |
| `skiavk` | Skia Vulkan renderer. |

### GuestRendererPreload

Zygote renderer preload policy.

| Value      | Description                                 |
| ---------- | ------------------------------------------- |
| `auto`     | Choose based on GPU mode and HWUI renderer. |
| `enabled`  | Enable renderer preload.                    |
| `disabled` | Disable renderer preload.                   |

### DeviceType

Type of emulated device.

| Value | Description        |
| ----- | ------------------ |
| `0`   | Unknown.           |
| `1`   | Phone.             |
| `2`   | Tablet.            |
| `3`   | TV.                |
| `4`   | Wearable.          |
| `5`   | Auto (automotive). |
| `6`   | Slim.              |
| `7`   | Go.                |

### Arch

Target CPU architecture.

| Value     | Description           |
| --------- | --------------------- |
| `Arm`     | 32-bit ARM.           |
| `Arm64`   | 64-bit ARM (AArch64). |
| `RiscV64` | 64-bit RISC-V.        |
| `X86`     | 32-bit x86.           |
| `X86_64`  | 64-bit x86.           |

---

## 5. Computed (Non-JSON) Paths

The following paths are derived at runtime from instance directory paths and have
**no backing JSON key**. They are accessed via `InstanceSpecific` helper methods.

| Method                                   | Computed Path                                        |
| ---------------------------------------- | ---------------------------------------------------- |
| `instance_dir()`                         | `<root_dir>/instances/cvd-<id>`                      |
| `instance_internal_dir()`                | `<instance_dir>/internal`                            |
| `instance_uds_dir()`                     | `<instances_uds_dir>/cvd-<id>`                       |
| `CrosvmSocketPath()`                     | `<instance_uds_dir>/internal/crosvm_control.sock`    |
| `OpenwrtCrosvmSocketPath()`              | `<instance_uds_dir>/internal/ap_control.sock`        |
| `hwcomposer_pmem_path()`                 | `<instance_dir>/hwcomposer-pmem`                     |
| `pstore_path()`                          | `<instance_dir>/pstore`                              |
| `pflash_path()`                          | `<instance_dir>/pflash.img`                          |
| `console_path()`                         | `<instance_dir>/console`                             |
| `logcat_path()`                          | `<instance_dir>/logs/logcat`                         |
| `launcher_log_path()`                    | `<instance_dir>/logs/launcher.log`                   |
| `launcher_monitor_socket_path()`         | `<instance_uds_dir>/launcher_monitor.sock`           |
| `sdcard_path()`                          | `<instance_dir>/sdcard.img`                          |
| `sdcard_overlay_path()`                  | `<instance_dir>/sdcard_overlay.img`                  |
| `persistent_composite_disk_path()`       | `<instance_dir>/persistent_composite.img`            |
| `persistent_composite_overlay_path()`    | `<instance_dir>/persistent_composite_overlay.img`    |
| `persistent_ap_composite_disk_path()`    | `<instance_dir>/ap_persistent_composite.img`         |
| `persistent_ap_composite_overlay_path()` | `<instance_dir>/ap_persistent_composite_overlay.img` |
| `os_composite_disk_path()`               | `<instance_dir>/os_composite.img`                    |
| `ap_composite_disk_path()`               | `<instance_dir>/ap_composite.img`                    |
| `uboot_env_image_path()`                 | `<instance_dir>/uboot_env.img`                       |
| `ap_uboot_env_image_path()`              | `<instance_dir>/ap_uboot_env.img`                    |
| `esp_image_path()`                       | `<instance_dir>/esp.img`                             |
| `ap_esp_image_path()`                    | `<instance_dir>/ap_esp.img`                          |
| `otheros_esp_grub_config()`              | `<instance_dir>/grub.cfg`                            |
| `ap_esp_grub_config()`                   | `<instance_dir>/ap_grub.cfg`                         |
| `audio_server_path()`                    | `<instance_uds_dir>/internal/audio_server.sock`      |
| `touch_socket_path(N)`                   | `<instance_uds_dir>/internal/touch_N.sock`           |
| `mouse_socket_path()`                    | `<instance_uds_dir>/internal/mouse.sock`             |
| `gamepad_socket_path()`                  | `<instance_uds_dir>/internal/gamepad.sock`           |
| `rotary_socket_path()`                   | `<instance_uds_dir>/internal/rotary.sock`            |
| `keyboard_socket_path()`                 | `<instance_uds_dir>/internal/keyboard.sock`          |
| `switches_socket_path()`                 | `<instance_uds_dir>/internal/switches.sock`          |
