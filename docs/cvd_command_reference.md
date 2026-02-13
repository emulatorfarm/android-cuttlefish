# CVD Command Reference

Comprehensive reference for the Cuttlefish Virtual Device (`cvd`) command-line interface.

> **Generated from source** — based on the handler implementations in
> `base/cvd/cuttlefish/host/commands/cvd/cli/commands/` and flag definitions in
> `base/cvd/cuttlefish/host/commands/assemble_cvd/`.

---

## Table of Contents

- [Synopsis](#synopsis)
- [Global Options](#global-options)
  - [Selector Options](#selector-options)
  - [Driver Options](#driver-options)
- [Commands](#commands)
  - [bugreport](#bugreport)
  - [cache](#cache)
  - [clear](#clear)
  - [create](#create)
  - [display](#display)
  - [env](#env)
  - [fetch](#fetch)
  - [fleet](#fleet)
  - [help](#help)
  - [lint](#lint)
  - [load](#load) _(deprecated)_
  - [login](#login)
  - [powerbtn](#powerbtn)
  - [powerwash](#powerwash)
  - [remove / rm](#remove)
  - [reset](#reset)
  - [restart](#restart)
  - [screen_recording](#screen_recording)
  - [snapshot (suspend / resume / snapshot_take)](#snapshot)
  - [start](#start)
  - [status](#status)
  - [stop](#stop)
  - [version](#version)
- [Deprecated / No-Op Commands](#deprecated--no-op-commands)
- [`cvd start` / `assemble_cvd` Flags](#cvd-start--assemble_cvd-flags)
  - [VM & Compute](#vm--compute)
  - [Graphics & Display](#graphics--display)
  - [Boot & Kernel](#boot--kernel)
  - [Disk & Images](#disk--images)
  - [Networking & Connectivity](#networking--connectivity)
  - [Bluetooth / NFC / UWB](#bluetooth--nfc--uwb)
  - [WebRTC & Streaming](#webrtc--streaming)
  - [Security](#security)
  - [Device Features](#device-features)
  - [Directories & Paths](#directories--paths)
  - [Debugging & Development](#debugging--development)
  - [Instance Management](#instance-management)
  - [Miscellaneous](#miscellaneous)
- [Alphabetical Flag Index](#alphabetical-flag-index)

---

## Synopsis

```
cvd [<selector options>] <command> [<args>...]
```

If invoked without a command (`cvd` or `cvd -h` / `cvd --help`), the `help` command is run automatically.

---

## Global Options

### Selector Options

These flags appear **before** the command name and select which instance group or instance to target.

| Flag                     | Type         | Description                                                              |
| ------------------------ | ------------ | ------------------------------------------------------------------------ |
| `-group_name <name>`     | string       | Specify the name of the instance group to create or select.              |
| `-instance_name <name>`  | string       | Select the device of the given name.                                     |
| `-instance_name <names>` | string (CSV) | Comma-separated list of device names to create within an instance group. |

### Driver Options

| Flag                 | Type   | Description                                                                                                                                |
| -------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `-help`              | bool   | Print top-level help and exit.                                                                                                             |
| `-verbosity=<LEVEL>` | string | Adjust log verbosity level. `LEVEL` is an Android log severity (e.g. `VERBOSE`, `DEBUG`, `INFO`, `WARNING`, `ERROR`). Requires cvd ≥ v1.3. |

---

## Commands

Commands are listed alphabetically. Aliases are noted in parentheses.

---

### bugreport

**Aliases:** `bugreport`, `host_bugreport`, `cvd_host_bugreport`

Collect host-side diagnostic information into a zip file.

```
cvd bugreport [<output_file>]
```

| Flag           | Type   | Default              | Description                            |
| -------------- | ------ | -------------------- | -------------------------------------- |
| _(positional)_ | string | `host_bugreport.zip` | Output filename for the bugreport zip. |

> Help is delegated to the underlying `cvd_internal_host_bugreport` binary; run `cvd bugreport --help` for full details.

---

### cache

Manage the local file cache used by `cvd fetch`.

```
cvd cache <action> [<flags>...]
```

**Subcommands:**

| Subcommand | Description                                                 |
| ---------- | ----------------------------------------------------------- |
| `empty`    | Wipe all files in the cache directory.                      |
| `info`     | Display the file path and approximate size of the cache.    |
| `prune`    | Cap the cache at a given size, removing oldest files first. |

**Flags:**

| Flag                | Type | Default              | Description                                           |
| ------------------- | ---- | -------------------- | ----------------------------------------------------- |
| `--allowed_size_gb` | int  | _(built-in default)_ | Allowed size of the cache during prune, in gigabytes. |
| `--json`            | bool | `false`              | Output `info` in JSON format.                         |

**Notes:**

- `info` and `prune` round the cache size up to the nearest gigabyte.
- `prune` uses last-modification time to remove the oldest files first.

---

### clear

Clears the instance database, stopping any running instances first.

```
cvd clear
```

No additional flags.

---

### create

Create a Cuttlefish virtual device group (and optionally start it).

```
cvd create [--product_path=PATH] [--host_path=PATH] [--[no]start] [START_ARGS]
cvd create --config_file=PATH [--[no]start]
```

| Flag                    | Type   | Default                                                         | Description                                                        |
| ----------------------- | ------ | --------------------------------------------------------------- | ------------------------------------------------------------------ |
| `--host_path`           | string | `$ANDROID_HOST_OUT`, `$ANDROID_SOONG_HOST_OUT`, `$HOME`, or cwd | Path to the directory containing Cuttlefish host artifacts.        |
| `--product_path`        | string | `$ANDROID_PRODUCT_OUT`, `$HOME`, or cwd                         | Path(s) to the directory containing guest images.                  |
| `--start` / `--nostart` | bool   | `true`                                                          | Whether to start the instance group after creation.                |
| `--config_file`         | string | _(none)_                                                        | Path to an environment config file to load.                        |
| `--acquire_file_lock`   | bool   | `true`                                                          | Whether the cvd server attempts to acquire the instance lock file. |

All other arguments are passed verbatim to `cvd start`. See [`cvd start` flags](#cvd-start--assemble_cvd-flags).

---

### display

Hotplug / unplug displays from running Cuttlefish virtual devices.

```
cvd display <subcommand> [<args>]
```

**Subcommands:**

| Subcommand       | Description                             |
| ---------------- | --------------------------------------- |
| `help <command>` | Print help for a subcommand.            |
| `add`            | Add a new display to a device.          |
| `list`           | Print the currently connected displays. |
| `remove`         | Remove a display from a device.         |
| `screenshot`     | Capture a screenshot of a display.      |

**Flags (screenshot):**

| Flag                | Type   | Default      | Description                       |
| ------------------- | ------ | ------------ | --------------------------------- |
| `--display`         | int    | _(required)_ | Display ID to capture.            |
| `--screenshot_path` | string | _(required)_ | File path to save the screenshot. |

---

### env

Enumerate and query gRPC service APIs exposed by a virtual device instance.

```
cvd env ls
cvd env ls <SERVICE_NAME>
cvd env ls <SERVICE_NAME> <METHOD_NAME>
cvd env type <SERVICE_NAME> <REQUEST_MESSAGE_TYPE>
```

| Subcommand                 | Description                                                         |
| -------------------------- | ------------------------------------------------------------------- |
| `ls`                       | List all available services per instance.                           |
| `ls <service>`             | List all methods for the given service.                             |
| `ls <service> <method>`    | Show input/output message types for a method.                       |
| `type <service> <message>` | Output the proto definition for the specified request message type. |

---

### fetch

Retrieve build artifacts from the Android Build API.

**Aliases:** `fetch`, `fetch_cvd`

```
cvd fetch [<flags>...]
```

#### General Flags

| Flag                         | Type   | Default   | Description                                                             |
| ---------------------------- | ------ | --------- | ----------------------------------------------------------------------- |
| `--target_directory`         | string | cwd       | Target directory to fetch files into.                                   |
| `--keep_downloaded_archives` | bool   | `false`   | Keep downloaded zip/tar archives after extraction.                      |
| `--host_package_build`       | string | _(none)_  | Build source for the host cvd tools.                                    |
| `--host_substitutions`       | string | _(empty)_ | Comma-separated list of executables to override with packaged versions. |

#### Build API Flags

| Flag                  | Type   | Default      | Description                                                                   |
| --------------------- | ------ | ------------ | ----------------------------------------------------------------------------- |
| `--api_key`           | string | `""`         | API key for the Android Build API.                                            |
| `--credential_source` | string | `""`         | Build API credential source. **(deprecated — use specific credential flags)** |
| `--project_id`        | string | `""`         | Project ID used to access the Build API.                                      |
| `--wait_retry_period` | int    | `20`         | Retry period (seconds) for pending builds. `0` = don't wait.                  |
| `--api_base_url`      | string | _(built-in)_ | Base URL for API requests.                                                    |
| `--enable_caching`    | bool   | `true`       | Enable local fetch file caching.                                              |
| `--max_cache_size_gb` | int    | _(built-in)_ | Max cache size in GB; cache is pruned after fetches complete.                 |

#### Credential Flags

| Flag                         | Type   | Default | Description                                 |
| ---------------------------- | ------ | ------- | ------------------------------------------- |
| `--use_gce_metadata`         | bool   | `false` | Use GCE metadata credentials.               |
| `--credential_filepath`      | string | `""`    | Path to a credentials file.                 |
| `--service_account_filepath` | string | `""`    | Path to a service-account credentials file. |

#### CAS Downloader Flags

| Flag                               | Type   | Default                                      | Description                                             |
| ---------------------------------- | ------ | -------------------------------------------- | ------------------------------------------------------- |
| `--cas_config_filepath`            | string | `/etc/casdownloader/config.json` (if exists) | CAS downloader config file path.                        |
| `--cas_downloader_binary`          | string | `/usr/bin/casdownloader` (if exists)         | Path to the CAS downloader binary.                      |
| `--cas_prefer_uncompressed`        | bool   | `false`                                      | Download uncompressed artifacts if available.           |
| `--cas_cache_dir`                  | string | `""`                                         | Cache directory for CAS downloads.                      |
| `--cas_invocation_id`              | string | `""`                                         | Optional invocation identifier for CAS downloader runs. |
| `--cas_cache_max_size`             | int64  | `8589934592` (8 GiB)                         | Max CAS cache size in bytes.                            |
| `--cas_cache_lock`                 | bool   | `false`                                      | Enable CAS cache lock.                                  |
| `--cas_use_hardlink`               | bool   | `true`                                       | Use hardlinks in CAS cache.                             |
| `--cas_concurrency`                | int    | `500`                                        | Max concurrent download operations.                     |
| `--cas_memory_limit`               | int    | `0`                                          | Memory limit in MiB (`0` = no limit).                   |
| `--cas_rpc_timeout`                | int    | `120`                                        | Default RPC timeout (seconds).                          |
| `--cas_get_capabilities_timeout`   | int    | `5`                                          | GetCapabilities RPC timeout (seconds).                  |
| `--cas_get_tree_timeout`           | int    | `5`                                          | GetTree RPC timeout (seconds).                          |
| `--cas_batch_read_blobs_timeout`   | int    | `180`                                        | BatchReadBlobs RPC timeout (seconds).                   |
| `--cas_batch_update_blobs_timeout` | int    | `60`                                         | BatchUpdateBlobs RPC timeout (seconds).                 |

#### Build Source Flags (repeatable per-build)

| Flag                          | Type   | Default   | Description                                                                 |
| ----------------------------- | ------ | --------- | --------------------------------------------------------------------------- |
| `--target_subdirectory`       | string | _(empty)_ | Subdirectory for organizing multi-fetch downloads.                          |
| `--default_build`             | string | _(empty)_ | Source for the cuttlefish build (vendor.img + host).                        |
| `--system_build`              | string | _(empty)_ | Source for `system.img` and `product.img`.                                  |
| `--kernel_build`              | string | _(empty)_ | Source for the kernel or GKI target.                                        |
| `--boot_build`                | string | _(empty)_ | Source for the boot or GKI target.                                          |
| `--bootloader_build`          | string | _(empty)_ | Source for the bootloader target.                                           |
| `--android_efi_loader_build`  | string | _(empty)_ | Source for the UEFI app target.                                             |
| `--otatools_build`            | string | _(empty)_ | Source for the host OTA tools.                                              |
| `--test_suites_build`         | string | _(empty)_ | Source for test suites.                                                     |
| `--chrome_os_build`           | string | _(empty)_ | Source for a ChromeOS build (numeric ID or `<project>/<bucket>/<builder>`). |
| `--boot_artifact`             | string | _(empty)_ | Name of the boot image in `boot_build`. **(deprecated)**                    |
| `--download_img_zip`          | bool   | `true`    | Fetch the `-img-*.zip` file.                                                |
| `--download_target_files_zip` | bool   | `false`   | Fetch the `-target_files-*.zip` file.                                       |
| `--dynamic_super_image`       | bool   | `false`   | Fetch super image members as independent files.                             |

---

### fleet

List active devices with relevant information.

```
cvd fleet
```

No additional flags.

---

### help

Display help information for `cvd` and its commands.

```
cvd help
cvd help <command>
```

- `cvd help` — display summary help for all available commands.
- `cvd help <command>` — display detailed help for a specific command.

---

### lint

Error-check a virtual device JSON config file.

```
cvd lint /path/to/input.json
```

No flags — the config file path is a positional argument.

---

### load

> **⚠️ Deprecated** — use `cvd create --config_file` instead.

Load a JSON configuration file and launch devices based on its options.

```
cvd load <config_filepath> [--override=<key>:<value>]
```

| Flag                  | Type                | Default  | Description                                                              |
| --------------------- | ------------------- | -------- | ------------------------------------------------------------------------ |
| `--override`          | string (repeatable) | _(none)_ | Override config values using `key:value` syntax with dot-notation paths. |
| `--credential_source` | string              | `""`     | Credential source for fetching artifacts.                                |
| `--project_id`        | string              | `""`     | GCP project ID for fetching.                                             |
| `--base_directory`    | string              | `$HOME`  | Parent directory for artifacts and runtime files.                        |

---

### login

Acquire credentials for the Android Build API.

```
cvd login --client_id=CLIENT_ID --client_secret=SECRET --scopes=SCOPES [--ssh]
```

| Flag              | Type   | Default       | Description                                                                                                 |
| ----------------- | ------ | ------------- | ----------------------------------------------------------------------------------------------------------- |
| `--client_id`     | string | `""`          | OAuth2 client ID.                                                                                           |
| `--client_secret` | string | `""`          | OAuth2 client secret.                                                                                       |
| `--scopes`        | string | `""`          | OAuth2 scopes.                                                                                              |
| `--ssh`           | bool   | auto-detected | Use SSH-compatible login flow. Defaults to `true` if `SSH_CLIENT` or `SSH_TTY` environment variable is set. |

---

### powerbtn

Trigger a power-button event on the device.

```
cvd powerbtn
```

No additional flags (uses selector options to pick the target instance).

---

### powerwash

Reset device state to first boot — functionally equivalent to removing and recreating the device, but more efficient.

```
cvd powerwash [--boot_timeout=SECONDS] [--wait_for_launcher=SECONDS]
```

| Flag                  | Type | Default | Description                                                           |
| --------------------- | ---- | ------- | --------------------------------------------------------------------- |
| `--boot_timeout`      | int  | `500`   | Seconds to wait for the device to reboot.                             |
| `--wait_for_launcher` | int  | `30`    | Seconds to wait for the launcher to respond. `0` = wait indefinitely. |

---

### remove

**Aliases:** `remove`, `rm`

Remove devices and artifacts from the system.

```
cvd remove
cvd rm
```

Running devices are stopped first. Deletes build and runtime artifacts, including log files and images (only if downloaded by `cvd` itself).

No additional flags (uses selector options to pick the target group).

---

### reset

Stop devices, optionally clean up instance files, and shut down the deprecated cvd server process.

> ⚠️ **Experimental** — when you are in panic, `cvd reset` is the last resort.

```
cvd reset [<flags>...]
```

| Flag                   | Type | Default | Description                                                                                                                |
| ---------------------- | ---- | ------- | -------------------------------------------------------------------------------------------------------------------------- |
| `--device-by-cvd-only` | bool | `false` | Only terminate devices started by a cvd server (excludes those launched directly by `launch_cvd` or `cvd_internal_start`). |
| `--clean-runtime-dir`  | bool | `true`  | Clean up the runtime directory for stopped devices.                                                                        |
| `--yes` / `-y`         | bool | `false` | Reset without asking for user confirmation.                                                                                |

**Steps performed:**

1. Gracefully stop all reachable devices.
2. Forcefully stop all `run_cvd` processes and their subprocesses.
3. Kill the cvd server itself if unresponsive.
4. Reset the states of involved instance lock files.
5. Optionally clean up runtime files.

---

### restart

Reboot the virtual device.

```
cvd restart [--boot_timeout=SECONDS] [--wait_for_launcher=SECONDS]
```

| Flag                  | Type | Default | Description                                                           |
| --------------------- | ---- | ------- | --------------------------------------------------------------------- |
| `--boot_timeout`      | int  | `500`   | Seconds to wait for the device to reboot.                             |
| `--wait_for_launcher` | int  | `30`    | Seconds to wait for the launcher to respond. `0` = wait indefinitely. |

---

### screen_recording

Record screen contents of a virtual device.

```
cvd screen_recording <subcommand> [--timeout SECONDS]
```

| Subcommand | Description                              |
| ---------- | ---------------------------------------- |
| `list`     | Print paths to existing recording files. |
| `start`    | Start a recording.                       |
| `stop`     | Stop a recording.                        |

| Flag        | Type | Default | Description                                  |
| ----------- | ---- | ------- | -------------------------------------------- |
| `--timeout` | int  | `5`     | Seconds to wait for the instance to respond. |

---

### snapshot

Suspend, resume, or snapshot a Cuttlefish device.

**Command names:** `suspend`, `resume`, `snapshot_take`

```
cvd [<selector flags>] suspend [--snapshot_path=PATH]
cvd [<selector flags>] resume  [--snapshot_path=PATH]
cvd [<selector flags>] snapshot_take [--snapshot_path=PATH]
```

| Flag                | Type   | Default  | Description                                                                          |
| ------------------- | ------ | -------- | ------------------------------------------------------------------------------------ |
| `--snapshot_path`   | string | _(none)_ | Directory that contains (or will contain) saved snapshot files.                      |
| `--snapshot_compat` | bool   | `false`  | **(Crosvm only)** Tell the device to be snapshot-compatible; verifies compatibility. |

---

### start

Start a Cuttlefish virtual device or environment.

**Aliases:** `start`, `launch_cvd`

```
cvd start [<flags>...]
```

> **Note:** `cvd start` rejects `--config_file` — use `cvd create --config_file` instead.
> `--daemon=false` / `--nodaemon` are also rejected; the device always starts in daemon mode.

The full set of flags accepted by `cvd start` is documented in the
[`cvd start` / `assemble_cvd` Flags](#cvd-start--assemble_cvd-flags) section below.

**Start-specific selector flags** (in addition to global selectors):

| Flag                  | Type         | Default  | Description                                                                                                                                              |
| --------------------- | ------------ | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--num_instances`     | int          | `1`      | Number of instances to create.                                                                                                                           |
| `--instance_nums`     | string (CSV) | _(none)_ | Explicit comma-separated instance IDs. Mutually exclusive with `--base_instance_num`.                                                                    |
| `--base_instance_num` | int          | _(none)_ | Base instance number; subsequent instances get consecutive IDs. Mutually exclusive with `--instance_nums`. Falls back to `$CUTTLEFISH_INSTANCE` env var. |
| `--use_cvdalloc`      | bool         | `false`  | Use `cvdalloc` for static resource allocation.                                                                                                           |

---

### status

Query the status of a single instance group.

**Aliases:** `status`, `cvd_status`

```
cvd status [<flags>...]
```

| Flag                  | Type   | Default | Description                                                           |
| --------------------- | ------ | ------- | --------------------------------------------------------------------- |
| `--wait_for_launcher` | int    | `5`     | Seconds to wait for the launcher to respond. `0` = wait indefinitely. |
| `--instance_name`     | string | `""`    | **(Deprecated)** Use selector options instead.                        |
| `--print`             | bool   | `false` | Print status and instance config to stdout. Requires Android > 12.    |

> Use `cvd fleet` to see all devices.

---

### stop

Stop all instances in an instance group.

**Aliases:** `stop`, `stop_cvd`

```
cvd stop [--wait_for_launcher=SECONDS] [--clear_instance_dirs]
```

| Flag                    | Type | Default | Description                                                           |
| ----------------------- | ---- | ------- | --------------------------------------------------------------------- |
| `--wait_for_launcher`   | int  | `5`     | Seconds to wait for the launcher to respond. `0` = wait indefinitely. |
| `--clear_instance_dirs` | bool | `false` | Delete instance directories after stopping.                           |

---

### version

Print version information for the cvd client and server.

```
cvd version [--json]
```

| Flag     | Type | Default | Description                                |
| -------- | ---- | ------- | ------------------------------------------ |
| `--json` | bool | `false` | Output version information in JSON format. |

---

## Deprecated / No-Op Commands

The following commands are retained for backward compatibility and perform no operation:

| Command          | Notes                                                |
| ---------------- | ---------------------------------------------------- |
| `server-kill`    | No-op.                                               |
| `kill-server`    | No-op.                                               |
| `restart-server` | No-op.                                               |
| `load`           | Deprecated — use `cvd create --config_file` instead. |

---

## `cvd start` / `assemble_cvd` Flags

These flags are accepted by `cvd start`, `cvd create --start` (passed through), and the legacy `launch_cvd` / `assemble_cvd` binaries. Approximately 155 flags organized by category.

Many flags accept comma-separated lists for multi-instance configurations (one value per instance).

---

### VM & Compute

| Flag                            | Type   | Default      | Description                                                                                                              |
| ------------------------------- | ------ | ------------ | ------------------------------------------------------------------------------------------------------------------------ |
| `--vm_manager`                  | string | _(auto)_     | Virtual machine manager: `crosvm`, `qemu_cli`.                                                                           |
| `--cpus`                        | string | `"4"`        | Virtual CPU count.                                                                                                       |
| `--memory_mb`                   | string | `"0"` (auto) | Total guest memory in MB.                                                                                                |
| `--enable_sandbox`              | string | `"false"`    | Enable crosvm sandbox. Disabled automatically when in a container or when GPU is enabled. `--noenable-sandbox` disables. |
| `--enable_virtiofs`             | string | `"false"`    | Enable shared folder using virtiofs.                                                                                     |
| `--seccomp_policy_dir`          | string | _(auto)_     | Override security comp policy directory for sandboxed crosvm.                                                            |
| `--crosvm_binary`               | string | `crosvm`     | Path to the crosvm binary.                                                                                               |
| `--qemu_binary_dir`             | string | _(auto)_     | Path to the directory containing the QEMU binary.                                                                        |
| `--gem5_binary_dir`             | string | `gem5`       | Path to the gem5 build tree root.                                                                                        |
| `--gem5_checkpoint_dir`         | string | `""`         | Path to the gem5 restore checkpoint directory.                                                                           |
| `--gem5_debug_file`             | string | `""`         | File for gem5 debug prints and logs.                                                                                     |
| `--gem5_debug_flags`            | string | `""`         | Debug flags for gem5.                                                                                                    |
| `--enable_smt`                  | string | `"false"`    | Enable simultaneous multithreading (SMT/HT).                                                                             |
| `--vsock_guest_cid`             | string | _(auto)_     | Guest vsock CID; also determines vsock server ports.                                                                     |
| `--vsock_guest_group`           | string | `""`         | Vsock isolation group name for inter-VM communication.                                                                   |
| `--vm_node_file`                | string | `""`         | Device node file for creating VMs.                                                                                       |
| `--vhost_vsock_node_file`       | string | `""`         | Device node file for vhost-vsock. Ignored for QEMU.                                                                      |
| `--crosvm_use_balloon`          | string | `"true"`     | Enable crosvm balloon device.                                                                                            |
| `--crosvm_use_rng`              | string | `"true"`     | Enable crosvm RNG device.                                                                                                |
| `--crosvm_use_pmem`             | string | `"true"`     | Enable crosvm pmem.                                                                                                      |
| `--crosvm_simple_media_device`  | string | `"false"`    | Use crosvm simple media device.                                                                                          |
| `--crosvm_v4l2_proxy`           | string | `""`         | Crosvm v4l2 proxy path. When set, overrides `crosvm_simple_media_device`.                                                |
| `--crosvm_use_vhost_user_block` | string | `"false"`    | **(Experimental)** Use crosvm vhost-user block device.                                                                   |
| `--snapshot_compatible`         | bool   | `false`      | Ensure device is snapshot-compatible.                                                                                    |
| `--protected_vm`                | string | `"false"`    | Boot in Protected VM (pVM) mode.                                                                                         |
| `--mte`                         | string | `"false"`    | Enable Memory Tagging Extension (MTE).                                                                                   |

---

### Graphics & Display

| Flag                        | Type   | Default                                              | Description                                                                                                                                                                                           |
| --------------------------- | ------ | ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--gpu_mode`                | string | `"auto"`                                             | GPU configuration: `auto`, `custom`, `drm_virgl`, `gfxstream`, `gfxstream_guest_angle`, `gfxstream_guest_angle_host_lavapipe`, `gfxstream_guest_angle_host_swiftshader`, `guest_swiftshader`, `none`. |
| `--gpu_vhost_user_mode`     | string | `"auto"`                                             | Run Virtio GPU worker in a separate process: `auto`, `on`, `off`.                                                                                                                                     |
| `--hwcomposer`              | string | `"auto"`                                             | Hardware composer: `auto`, `drm`, `ranchu`.                                                                                                                                                           |
| `--gpu_capture_binary`      | string | `""`                                                 | Path to GPU trace capture binary (ngfx, renderdoc, etc.).                                                                                                                                             |
| `--enable_gpu_udmabuf`      | string | `"false"`                                            | Use udmabuf driver for zero-copy virtio-gpu.                                                                                                                                                          |
| `--gpu_renderer_features`   | string | `""`                                                 | Renderer-specific features. For Gfxstream: semicolon-separated `"<name>:[enabled\|disabled]"`.                                                                                                        |
| `--gpu_context_types`       | string | `"gfxstream-vulkan:cross-domain:gfxstream-composer"` | Colon-separated virtio-gpu context types. Only valid with `--gpu_mode=custom`.                                                                                                                        |
| `--guest_hwui_renderer`     | string | `""`                                                 | Default HWUI renderer: `skiagl` or `skiavk`.                                                                                                                                                          |
| `--guest_vulkan_driver`     | string | `"ranchu"`                                           | Vulkan driver for guest. Only valid with `--gpu_mode=custom`.                                                                                                                                         |
| `--zygote_preload_disabled` | string | `"auto"`                                             | Disable Zygote renderer preload: `auto`, `enabled`, `disabled`.                                                                                                                                       |
| `--frames_socket_path`      | string | `""`                                                 | Frame socket path for VM (e.g., Wayland socket).                                                                                                                                                      |
| `--x_res`                   | string | `"0"`                                                | Screen width in pixels.                                                                                                                                                                               |
| `--y_res`                   | string | `"0"`                                                | Screen height in pixels.                                                                                                                                                                              |
| `--dpi`                     | string | `"0"`                                                | Screen density in DPI.                                                                                                                                                                                |
| `--refresh_rate_hz`         | string | `"60"`                                               | Screen refresh rate in Hertz.                                                                                                                                                                         |
| `--display0`                | string | `""`                                                 | Display 0 configuration string.                                                                                                                                                                       |
| `--display1`                | string | `""`                                                 | Display 1 configuration string.                                                                                                                                                                       |
| `--display2`                | string | `""`                                                 | Display 2 configuration string.                                                                                                                                                                       |
| `--display3`                | string | `""`                                                 | Display 3 configuration string.                                                                                                                                                                       |
| `--displays_textproto`      | string | `""`                                                 | Text proto input for multi-VD multi-displays.                                                                                                                                                         |
| `--displays_binproto`       | string | `""`                                                 | Binary proto input for multi-VD multi-displays.                                                                                                                                                       |
| `--displays_overlay`        | string | `""`                                                 | List of display overlays. Format: `"vm_index:display_index ..."`.                                                                                                                                     |
| `--record_screen`           | string | `"false"`                                            | Enable screen recording. Requires WebRTC.                                                                                                                                                             |
| `--enable_boot_animation`   | string | `"true"`                                             | Whether to enable the boot animation.                                                                                                                                                                 |
| `--min_mode`                | string | `"false"`                                            | Only enable minimum features for boot and minimal UI. Handheld/phone targets only.                                                                                                                    |

---

### Boot & Kernel

| Flag                             | Type   | Default   | Description                                                                                                                                                              |
| -------------------------------- | ------ | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `--kernel_path`                  | string | `""`      | Path to the kernel. Overrides the one from the boot image.                                                                                                               |
| `--initramfs_path`               | string | `""`      | Path to the initramfs. Overrides the one from the boot image.                                                                                                            |
| `--extra_kernel_cmdline`         | string | `""`      | Additional flags for the kernel command line.                                                                                                                            |
| `--extra_bootconfig_args`        | string | `""`      | Space-separated extra bootconfig args. Use `:=` to overwrite existing args.                                                                                              |
| `--extra_bootconfig_args_base64` | string | `""`      | Base64-encoded version of `extra_bootconfig_args`. For multi-device clusters.                                                                                            |
| `--boot_slot`                    | string | `""`      | Force booting into a given slot (e.g., `a`, `b`). Empty = auto-detect.                                                                                                   |
| `--bootloader`                   | string | `""`      | Bootloader binary path.                                                                                                                                                  |
| `--boot_image`                   | string | `""`      | Location of the boot image. Defaults to `boot.img` in `system_image_dir`.                                                                                                |
| `--vendor_boot_image`            | string | `""`      | Location of the vendor boot image. Defaults to `vendor_boot.img` in `system_image_dir`.                                                                                  |
| `--android_efi_loader`           | string | `""`      | Location of the Android EFI loader.                                                                                                                                      |
| `--pause_in_bootloader`          | string | `"false"` | Stop the bootflow in U-Boot. Continue by typing `boot` in the console.                                                                                                   |
| `--fail_fast`                    | string | `"true"`  | Exit when a heuristic predicts the boot will not complete.                                                                                                               |
| `--resume`                       | bool   | `true`    | Resume using the disk from the last session. `--noresume` resets disk state. Ignored if partition images have been updated. Always `true` when starting from a snapshot. |

---

### Disk & Images

| Flag                         | Type   | Default          | Description                                                                                                                                   |
| ---------------------------- | ------ | ---------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `--system_image_dir`         | string | _(auto)_         | Directory where `.img` files are loaded from.                                                                                                 |
| `--super_image`              | string | `""`             | Location of `super.img`. Defaults to `system_image_dir`.                                                                                      |
| `--data_policy`              | string | `"use_existing"` | Userdata partition handling: `use_existing`, `resize_up_to`, or `always_create`.                                                              |
| `--blank_data_image_mb`      | string | `"8192"`         | Size of the blank data image in MB.                                                                                                           |
| `--blank_sdcard_image_mb`    | string | `"2048"`         | Size of the blank SD card image in MB.                                                                                                        |
| `--use_sdcard`               | string | `"true"`         | Create blank SD card image and expose to guest.                                                                                               |
| `--userdata_format`          | string | `"f2fs"`         | Userdata filesystem format.                                                                                                                   |
| `--use_overlay`              | bool   | `true`           | Capture disk writes in an overlay. Required for powerwash and multi-instance.                                                                 |
| `--vbmeta_image`             | string | `""`             | Location of `vbmeta.img`.                                                                                                                     |
| `--vbmeta_system_image`      | string | `""`             | Location of `vbmeta_system.img`.                                                                                                              |
| `--vbmeta_vendor_dlkm_image` | string | `""`             | Location of `vbmeta_vendor_dlkm.img`.                                                                                                         |
| `--vbmeta_system_dlkm_image` | string | `""`             | Location of `vbmeta_system_dlkm.img`.                                                                                                         |
| `--vvmtruststore_path_name`  | string | `""`             | Default file name of the vvmtruststore image.                                                                                                 |
| `--vvmtruststore_path`       | string | `""`             | Location of the vvmtruststore image.                                                                                                          |
| `--default_target_zip`       | string | `""`             | Location of default target zip file.                                                                                                          |
| `--system_target_zip`        | string | `""`             | Location of system target zip file.                                                                                                           |
| `--custom_partition_path`    | string | `""`             | Path to custom partition image(s). Multiple images separated by `;`. Accessible via `/dev/block/by-name/custom`, `custom_1`, `custom_2`, etc. |
| `--build_super_image`        | string | `"false"`        | Build the super image at runtime. Probably not what you want.                                                                                 |

---

### Networking & Connectivity

| Flag                          | Type   | Default     | Description                                                                     |
| ----------------------------- | ------ | ----------- | ------------------------------------------------------------------------------- |
| `--ril_dns`                   | string | `"8.8.8.8"` | DNS address for mobile network (RIL).                                           |
| `--vhost_net`                 | string | `"false"`   | Enable vhost acceleration of networking.                                        |
| `--vhost_user_vsock`          | string | `"auto"`    | Enable vhost-user-vsock.                                                        |
| `--wifi_tap_interfaces`       | string | `"true"`    | Use TAP devices for network connectivity.                                       |
| `--connectivity_mode`         | string | `"tap"`     | Mechanism to connect to the public internet.                                    |
| `--guest_wifi`                | string | `"true"`    | Enable guest WiFi. Mainly for Minidroid.                                        |
| `--vhost_user_mac80211_hwsim` | string | `""`        | Unix socket path for vhost-user mac80211_hwsim (served by wmediumd).            |
| `--wmediumd_config`           | string | `""`        | Path to wmediumd config file. Defaults to a config for up to 16 instances + AP. |
| `--ap_rootfs_image`           | string | `""`        | Root filesystem image for AP instance.                                          |
| `--ap_kernel_image`           | string | `""`        | Kernel image for AP instance.                                                   |

---

### Bluetooth / NFC / UWB

| Flag                                | Type   | Default   | Description                                                 |
| ----------------------------------- | ------ | --------- | ----------------------------------------------------------- |
| `--rootcanal_default_commands_file` | string | `""`      | _(deprecated)_ Bluetooth emulator default commands file.    |
| `--enable_host_bluetooth`           | string | `"true"`  | Enable rootcanal (Bluetooth emulator) on the host.          |
| `--rootcanal_instance_num`          | int    | `0`       | Share an existing rootcanal instance. `0` = launch new.     |
| `--rootcanal_args`                  | string | `""`      | Space-separated rootcanal args.                             |
| `--enable_host_nfc`                 | bool   | `true`    | Enable the NFC emulator on the host.                        |
| `--casimir_instance_num`            | int    | `0`       | Share an existing Casimir (NFC) instance. `0` = launch new. |
| `--casimir_args`                    | string | `""`      | Space-separated Casimir args.                               |
| `--enable_host_uwb`                 | bool   | `true`    | Enable the UWB host and connector.                          |
| `--pica_instance_num`               | int    | `0`       | Share an existing Pica (UWB) instance. `0` = launch new.    |
| `--netsim`                          | string | `"false"` | **(Experimental)** Connect all radios to netsim.            |
| `--netsim_bt`                       | string | `"true"`  | Connect Bluetooth radio to netsim.                          |
| `--netsim_uwb`                      | string | `"true"`  | **(Experimental)** Connect UWB radio to netsim.             |
| `--netsim_args`                     | string | `""`      | Space-separated netsim args.                                |

---

### WebRTC & Streaming

| Flag                            | Type   | Default                     | Description                                                                       |
| ------------------------------- | ------ | --------------------------- | --------------------------------------------------------------------------------- |
| `--start_webrtc`                | string | `"false"`                   | **(Deprecated)** WebRTC is enabled depending on the VMM.                          |
| `--webrtc_assets_dir`           | string | `"usr/share/webrtc/assets"` | Path to WebRTC webpage assets.                                                    |
| `--sig_server_address`          | string | `/run/cuttlefish/operator`  | Address of the WebRTC signaling server.                                           |
| `--tcp_port_range`              | string | `"15550:15599"`             | TCP port range for ICE candidates (`min:max`). `0:0` = any port.                  |
| `--udp_port_range`              | string | `"15550:15599"`             | UDP port range for ICE candidates (`min:max`). `0:0` = any port.                  |
| `--webrtc_device_id`            | string | `"cvd-{num}"`               | Device ID for the signaling server. `{num}` is replaced with the instance number. |
| `--uuid`                        | string | _(auto)_                    | UUID for the device. Random if not specified.                                     |
| `--webrtc_enable_adb_websocket` | string | `"DISABLED"`                | ADB over WebSocket: `DISABLED`, `OPTIONAL`, `REQUIRED`.                           |

---

### Security

| Flag                       | Type   | Default             | Description                                                      |
| -------------------------- | ------ | ------------------- | ---------------------------------------------------------------- |
| `--serial_number`          | string | `"CUTTLEFISHCVD01"` | Serial number for the device.                                    |
| `--use_random_serial`      | string | `"false"`           | Use a random serial number.                                      |
| `--guest_enforce_security` | string | `"true"`            | Run in enforcing (non-permissive) SELinux mode.                  |
| `--secure_hals`            | string | `""`                | Host security-enabled HALs. Supports `keymint` and `gatekeeper`. |

---

### Device Features

| Flag                             | Type   | Default   | Description                                           |
| -------------------------------- | ------ | --------- | ----------------------------------------------------- |
| `--enable_audio`                 | string | `"true"`  | Enable audio playback/capture.                        |
| `--enable_usb`                   | string | `"false"` | Enable USB passthrough.                               |
| `--enable_host_jcard`            | string | `"false"` | Enable host jcard simulator.                          |
| `--camera_server_port`           | string | `"0"`     | Camera vsock port.                                    |
| `--enable_modem_simulator`       | string | `"true"`  | Enable the modem simulator for RILD AT commands.      |
| `--modem_simulator_sim_type`     | string | `"1"`     | SIM type: `1` = normal, `2` = CtsCarrierApiTestCases. |
| `--modem_simulator_count`        | string | `"1"`     | Modem simulator count (max SIM number).               |
| `--start_gnss_proxy`             | string | `"true"`  | Start the GNSS proxy.                                 |
| `--gnss_file_path`               | string | `""`      | Local GNSS raw measurement file path.                 |
| `--fixed_location_file_path`     | string | `""`      | Local fixed-location file path for GNSS proxy.        |
| `--enable_host_automotive_proxy` | bool   | `false`   | Enable the automotive proxy service on the host.      |
| `--enable_vhal_proxy`            | string | `"false"` | Enable the VHAL proxy service on the host.            |

---

### Directories & Paths

| Flag                         | Type   | Default                     | Description                                                                  |
| ---------------------------- | ------ | --------------------------- | ---------------------------------------------------------------------------- |
| `--assembly_dir`             | string | `$HOME/cuttlefish_assembly` | Directory for generated files common between instances.                      |
| `--instance_dir`             | string | `$HOME/cuttlefish`          | Directory for all cuttlefish generated files (instance-specific and common). |
| `--snapshot_path`            | string | `""`                        | Path to snapshot for restore.                                                |
| `--tmpdir_for_early_startup` | string | _(auto)_                    | Parent directory for temporary files in early startup.                       |

---

### Debugging & Development

| Flag                     | Type   | Default   | Description                                                                                              |
| ------------------------ | ------ | --------- | -------------------------------------------------------------------------------------------------------- |
| `--gdb_port`             | string | `"0"`     | Port number for kernel GDB (`0` = disabled). Kernel must be built with `CONFIG_RANDOMIZE_BASE` disabled. |
| `--kgdb`                 | string | `"false"` | Debug the kernel with kgdb/kdb. Kernel must have kgdb support; serial console must be enabled.           |
| `--enable_kernel_log`    | string | `"true"`  | Enable kernel console/dmesg logging.                                                                     |
| `--console`              | string | `"false"` | Enable the serial console.                                                                               |
| `--strace_cmd`           | string | `""`      | Comma-separated executable names to run under strace.                                                    |
| `--track_host_tools_crc` | bool   | `false`   | Track changes to host executables.                                                                       |

---

### Instance Management

| Flag                             | Type   | Default   | Description                                                                              |
| -------------------------------- | ------ | --------- | ---------------------------------------------------------------------------------------- |
| `--num_instances`                | int    | `1`       | Number of Android guests to launch.                                                      |
| `--instance_nums`                | string | `""`      | Comma-separated list of instance numbers. Mutually exclusive with `--base_instance_num`. |
| `--report_anonymous_usage_stats` | string | `""`      | Report anonymous usage statistics (`y` or `n`).                                          |
| `--daemon`                       | string | `"false"` | Run in background; launcher exits on boot completed/failed.                              |
| `--restart_subprocesses`         | string | `"false"` | Restart any crashed host process.                                                        |
| `--use_cvdalloc`                 | string | `"unset"` | Acquire static resources with cvdalloc.                                                  |

---

### Miscellaneous

| Flag                            | Type   | Default | Description                              |
| ------------------------------- | ------ | ------- | ---------------------------------------- |
| `--mcu_config_path`             | string | `""`    | Configuration file for the MCU emulator. |
| `--virtual_cpufreq_config_path` | string | `""`    | Configuration file for Virtual Cpufreq.  |

#### Other-OS / ChromeOS / Fuchsia Images

| Flag                           | Type   | Default | Description                                      |
| ------------------------------ | ------ | ------- | ------------------------------------------------ |
| `--otheros_linux_kernel`       | string | `""`    | Linux kernel for cuttlefish other-OS flow.       |
| `--otheros_linux_initramfs`    | string | `""`    | Linux initramfs for other-OS flow.               |
| `--otheros_linux_rootfs`       | string | `""`    | Linux root filesystem image for other-OS flow.   |
| `--chromeos_disk`              | string | `""`    | Complete ChromeOS GPT disk.                      |
| `--chromeos_kernel`            | string | `""`    | ChromeOS kernel.                                 |
| `--chromeos_root_image`        | string | `""`    | ChromeOS root filesystem image.                  |
| `--fuchsia_zedboot_path`       | string | `""`    | Fuchsia zedboot path for other-OS flow.          |
| `--fuchsia_multiboot_bin_path` | string | `""`    | Fuchsia multiboot binary for other-OS flow.      |
| `--fuchsia_root_image`         | string | `""`    | Fuchsia root filesystem image for other-OS flow. |

---

## Alphabetical Flag Index

All `cvd start` / `assemble_cvd` flags listed alphabetically with their category cross-reference.

| Flag                                | Category                                           |
| ----------------------------------- | -------------------------------------------------- |
| `--android_efi_loader`              | [Boot & Kernel](#boot--kernel)                     |
| `--ap_kernel_image`                 | [Networking](#networking--connectivity)            |
| `--ap_rootfs_image`                 | [Networking](#networking--connectivity)            |
| `--assembly_dir`                    | [Directories & Paths](#directories--paths)         |
| `--blank_data_image_mb`             | [Disk & Images](#disk--images)                     |
| `--blank_sdcard_image_mb`           | [Disk & Images](#disk--images)                     |
| `--boot_image`                      | [Boot & Kernel](#boot--kernel)                     |
| `--boot_slot`                       | [Boot & Kernel](#boot--kernel)                     |
| `--bootloader`                      | [Boot & Kernel](#boot--kernel)                     |
| `--build_super_image`               | [Disk & Images](#disk--images)                     |
| `--camera_server_port`              | [Device Features](#device-features)                |
| `--casimir_args`                    | [Bluetooth / NFC / UWB](#bluetooth--nfc--uwb)      |
| `--casimir_instance_num`            | [Bluetooth / NFC / UWB](#bluetooth--nfc--uwb)      |
| `--chromeos_disk`                   | [Miscellaneous](#miscellaneous)                    |
| `--chromeos_kernel`                 | [Miscellaneous](#miscellaneous)                    |
| `--chromeos_root_image`             | [Miscellaneous](#miscellaneous)                    |
| `--connectivity_mode`               | [Networking](#networking--connectivity)            |
| `--console`                         | [Debugging & Development](#debugging--development) |
| `--cpus`                            | [VM & Compute](#vm--compute)                       |
| `--crosvm_binary`                   | [VM & Compute](#vm--compute)                       |
| `--crosvm_simple_media_device`      | [VM & Compute](#vm--compute)                       |
| `--crosvm_use_balloon`              | [VM & Compute](#vm--compute)                       |
| `--crosvm_use_pmem`                 | [VM & Compute](#vm--compute)                       |
| `--crosvm_use_rng`                  | [VM & Compute](#vm--compute)                       |
| `--crosvm_use_vhost_user_block`     | [VM & Compute](#vm--compute)                       |
| `--crosvm_v4l2_proxy`               | [VM & Compute](#vm--compute)                       |
| `--custom_partition_path`           | [Disk & Images](#disk--images)                     |
| `--daemon`                          | [Instance Management](#instance-management)        |
| `--data_policy`                     | [Disk & Images](#disk--images)                     |
| `--default_target_zip`              | [Disk & Images](#disk--images)                     |
| `--display0` – `--display3`         | [Graphics & Display](#graphics--display)           |
| `--displays_binproto`               | [Graphics & Display](#graphics--display)           |
| `--displays_overlay`                | [Graphics & Display](#graphics--display)           |
| `--displays_textproto`              | [Graphics & Display](#graphics--display)           |
| `--dpi`                             | [Graphics & Display](#graphics--display)           |
| `--enable_audio`                    | [Device Features](#device-features)                |
| `--enable_boot_animation`           | [Graphics & Display](#graphics--display)           |
| `--enable_gpu_udmabuf`              | [Graphics & Display](#graphics--display)           |
| `--enable_host_automotive_proxy`    | [Device Features](#device-features)                |
| `--enable_host_bluetooth`           | [Bluetooth / NFC / UWB](#bluetooth--nfc--uwb)      |
| `--enable_host_jcard`               | [Device Features](#device-features)                |
| `--enable_host_nfc`                 | [Bluetooth / NFC / UWB](#bluetooth--nfc--uwb)      |
| `--enable_host_uwb`                 | [Bluetooth / NFC / UWB](#bluetooth--nfc--uwb)      |
| `--enable_kernel_log`               | [Debugging & Development](#debugging--development) |
| `--enable_modem_simulator`          | [Device Features](#device-features)                |
| `--enable_sandbox`                  | [VM & Compute](#vm--compute)                       |
| `--enable_smt`                      | [VM & Compute](#vm--compute)                       |
| `--enable_usb`                      | [Device Features](#device-features)                |
| `--enable_vhal_proxy`               | [Device Features](#device-features)                |
| `--enable_virtiofs`                 | [VM & Compute](#vm--compute)                       |
| `--extra_bootconfig_args`           | [Boot & Kernel](#boot--kernel)                     |
| `--extra_bootconfig_args_base64`    | [Boot & Kernel](#boot--kernel)                     |
| `--extra_kernel_cmdline`            | [Boot & Kernel](#boot--kernel)                     |
| `--fail_fast`                       | [Boot & Kernel](#boot--kernel)                     |
| `--fixed_location_file_path`        | [Device Features](#device-features)                |
| `--frames_socket_path`              | [Graphics & Display](#graphics--display)           |
| `--fuchsia_multiboot_bin_path`      | [Miscellaneous](#miscellaneous)                    |
| `--fuchsia_root_image`              | [Miscellaneous](#miscellaneous)                    |
| `--fuchsia_zedboot_path`            | [Miscellaneous](#miscellaneous)                    |
| `--gdb_port`                        | [Debugging & Development](#debugging--development) |
| `--gem5_binary_dir`                 | [VM & Compute](#vm--compute)                       |
| `--gem5_checkpoint_dir`             | [VM & Compute](#vm--compute)                       |
| `--gem5_debug_file`                 | [VM & Compute](#vm--compute)                       |
| `--gem5_debug_flags`                | [VM & Compute](#vm--compute)                       |
| `--gnss_file_path`                  | [Device Features](#device-features)                |
| `--gpu_capture_binary`              | [Graphics & Display](#graphics--display)           |
| `--gpu_context_types`               | [Graphics & Display](#graphics--display)           |
| `--gpu_mode`                        | [Graphics & Display](#graphics--display)           |
| `--gpu_renderer_features`           | [Graphics & Display](#graphics--display)           |
| `--gpu_vhost_user_mode`             | [Graphics & Display](#graphics--display)           |
| `--guest_enforce_security`          | [Security](#security)                              |
| `--guest_hwui_renderer`             | [Graphics & Display](#graphics--display)           |
| `--guest_vulkan_driver`             | [Graphics & Display](#graphics--display)           |
| `--guest_wifi`                      | [Networking](#networking--connectivity)            |
| `--hwcomposer`                      | [Graphics & Display](#graphics--display)           |
| `--initramfs_path`                  | [Boot & Kernel](#boot--kernel)                     |
| `--instance_dir`                    | [Directories & Paths](#directories--paths)         |
| `--instance_nums`                   | [Instance Management](#instance-management)        |
| `--kernel_path`                     | [Boot & Kernel](#boot--kernel)                     |
| `--kgdb`                            | [Debugging & Development](#debugging--development) |
| `--mcu_config_path`                 | [Miscellaneous](#miscellaneous)                    |
| `--memory_mb`                       | [VM & Compute](#vm--compute)                       |
| `--min_mode`                        | [Graphics & Display](#graphics--display)           |
| `--modem_simulator_count`           | [Device Features](#device-features)                |
| `--modem_simulator_sim_type`        | [Device Features](#device-features)                |
| `--mte`                             | [VM & Compute](#vm--compute)                       |
| `--netsim`                          | [Bluetooth / NFC / UWB](#bluetooth--nfc--uwb)      |
| `--netsim_args`                     | [Bluetooth / NFC / UWB](#bluetooth--nfc--uwb)      |
| `--netsim_bt`                       | [Bluetooth / NFC / UWB](#bluetooth--nfc--uwb)      |
| `--netsim_uwb`                      | [Bluetooth / NFC / UWB](#bluetooth--nfc--uwb)      |
| `--num_instances`                   | [Instance Management](#instance-management)        |
| `--otheros_linux_initramfs`         | [Miscellaneous](#miscellaneous)                    |
| `--otheros_linux_kernel`            | [Miscellaneous](#miscellaneous)                    |
| `--otheros_linux_rootfs`            | [Miscellaneous](#miscellaneous)                    |
| `--pause_in_bootloader`             | [Boot & Kernel](#boot--kernel)                     |
| `--pica_instance_num`               | [Bluetooth / NFC / UWB](#bluetooth--nfc--uwb)      |
| `--protected_vm`                    | [VM & Compute](#vm--compute)                       |
| `--qemu_binary_dir`                 | [VM & Compute](#vm--compute)                       |
| `--record_screen`                   | [Graphics & Display](#graphics--display)           |
| `--refresh_rate_hz`                 | [Graphics & Display](#graphics--display)           |
| `--report_anonymous_usage_stats`    | [Instance Management](#instance-management)        |
| `--restart_subprocesses`            | [Instance Management](#instance-management)        |
| `--resume`                          | [Boot & Kernel](#boot--kernel)                     |
| `--ril_dns`                         | [Networking](#networking--connectivity)            |
| `--rootcanal_args`                  | [Bluetooth / NFC / UWB](#bluetooth--nfc--uwb)      |
| `--rootcanal_default_commands_file` | [Bluetooth / NFC / UWB](#bluetooth--nfc--uwb)      |
| `--rootcanal_instance_num`          | [Bluetooth / NFC / UWB](#bluetooth--nfc--uwb)      |
| `--seccomp_policy_dir`              | [VM & Compute](#vm--compute)                       |
| `--secure_hals`                     | [Security](#security)                              |
| `--serial_number`                   | [Security](#security)                              |
| `--sig_server_address`              | [WebRTC & Streaming](#webrtc--streaming)           |
| `--snapshot_compatible`             | [VM & Compute](#vm--compute)                       |
| `--snapshot_path`                   | [Directories & Paths](#directories--paths)         |
| `--start_gnss_proxy`                | [Device Features](#device-features)                |
| `--start_webrtc`                    | [WebRTC & Streaming](#webrtc--streaming)           |
| `--strace_cmd`                      | [Debugging & Development](#debugging--development) |
| `--super_image`                     | [Disk & Images](#disk--images)                     |
| `--system_image_dir`                | [Disk & Images](#disk--images)                     |
| `--system_target_zip`               | [Disk & Images](#disk--images)                     |
| `--tcp_port_range`                  | [WebRTC & Streaming](#webrtc--streaming)           |
| `--tmpdir_for_early_startup`        | [Directories & Paths](#directories--paths)         |
| `--track_host_tools_crc`            | [Debugging & Development](#debugging--development) |
| `--udp_port_range`                  | [WebRTC & Streaming](#webrtc--streaming)           |
| `--use_cvdalloc`                    | [Instance Management](#instance-management)        |
| `--use_overlay`                     | [Disk & Images](#disk--images)                     |
| `--use_random_serial`               | [Security](#security)                              |
| `--use_sdcard`                      | [Disk & Images](#disk--images)                     |
| `--userdata_format`                 | [Disk & Images](#disk--images)                     |
| `--uuid`                            | [WebRTC & Streaming](#webrtc--streaming)           |
| `--vbmeta_image`                    | [Disk & Images](#disk--images)                     |
| `--vbmeta_system_dlkm_image`        | [Disk & Images](#disk--images)                     |
| `--vbmeta_system_image`             | [Disk & Images](#disk--images)                     |
| `--vbmeta_vendor_dlkm_image`        | [Disk & Images](#disk--images)                     |
| `--vhost_net`                       | [Networking](#networking--connectivity)            |
| `--vhost_user_mac80211_hwsim`       | [Networking](#networking--connectivity)            |
| `--vhost_user_vsock`                | [VM & Compute](#vm--compute)                       |
| `--vhost_vsock_node_file`           | [VM & Compute](#vm--compute)                       |
| `--virtual_cpufreq_config_path`     | [Miscellaneous](#miscellaneous)                    |
| `--vm_manager`                      | [VM & Compute](#vm--compute)                       |
| `--vm_node_file`                    | [VM & Compute](#vm--compute)                       |
| `--vsock_guest_cid`                 | [VM & Compute](#vm--compute)                       |
| `--vsock_guest_group`               | [VM & Compute](#vm--compute)                       |
| `--vvmtruststore_path`              | [Disk & Images](#disk--images)                     |
| `--vvmtruststore_path_name`         | [Disk & Images](#disk--images)                     |
| `--webrtc_assets_dir`               | [WebRTC & Streaming](#webrtc--streaming)           |
| `--webrtc_device_id`                | [WebRTC & Streaming](#webrtc--streaming)           |
| `--webrtc_enable_adb_websocket`     | [WebRTC & Streaming](#webrtc--streaming)           |
| `--wifi_tap_interfaces`             | [Networking](#networking--connectivity)            |
| `--wmediumd_config`                 | [Networking](#networking--connectivity)            |
| `--x_res`                           | [Graphics & Display](#graphics--display)           |
| `--y_res`                           | [Graphics & Display](#graphics--display)           |
| `--zygote_preload_disabled`         | [Graphics & Display](#graphics--display)           |
