# assemble_cvd Disk Construction Processes

When `launch_cvd` invokes `assemble_cvd`, a series of disk-image transformations
prepare the AOSP build artifacts (e.g. `system.img`, `vendor.img`) for execution
inside a Cuttlefish virtual device running on crosvm. This document explains each
of the five major steps: what it is, what it depends on, how it works, and why
it is necessary.

> **Source entry point:**
> [`create_dynamic_disk_files.cc → CreateDynamicDiskFiles()`](../base/cvd/cuttlefish/host/commands/assemble_cvd/create_dynamic_disk_files.cc)
> orchestrates all five processes in order.

---

## 1. OS Composite Disk Construction

### What it is

The OS composite disk (`os_composite.img`) is a **virtual GPT-partitioned disk**
that presents every Android physical partition (boot, vendor_boot, vbmeta,
super, userdata, misc, metadata, etc.) as a single block device to the guest VM.
It uses crosvm's **Composite Disk** format — a protobuf-based indirection layer
that references the individual image files on the host by path and offset
instead of copying their contents into a single monolithic file.

### What it depends on

| Dependency                                                                                                                                          | Purpose                                                                         |
| --------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| All physical partition images (`boot.img`, `init_boot.img`, `vendor_boot.img`, `vbmeta.img`, `super.img`, `userdata.img`, `misc`, `metadata`, etc.) | Raw content of each GPT partition                                               |
| The `super.img` (possibly rebuilt — see §2)                                                                                                         | Contains all dynamic/logical partitions                                         |
| Repacked boot/vendor_boot images (possibly modified — see §5)                                                                                       | Updated kernel & initramfs                                                      |
| `crosvm` binary                                                                                                                                     | Used when the VM manager is crosvm to handle the composite disk format natively |
| GPT header & footer templates                                                                                                                       | Written to `os_composite_gpt_header.img` and `os_composite_gpt_footer.img`      |

### How it works

1. **Partition list assembly** — `AndroidCompositeDiskConfig()` in
   [`android_composite_disk_config.cc`](../base/cvd/cuttlefish/host/commands/assemble_cvd/disk/android_composite_disk_config.cc)
   builds a `std::vector<ImagePartition>` containing every partition label
   and its backing image file path. A/B partitions (boot, init_boot,
   vendor_boot, vbmeta, vbmeta_system, vbmeta_system_dlkm,
   vbmeta_vendor_dlkm) are duplicated into `_a` and `_b` slot entries.
   Optional partitions (hibernation, vvmtruststore, custom) are included
   only when their backing files exist.

2. **DiskBuilder configuration** — `OsCompositeDiskBuilder()` in
   [`os_composite_disk.cc`](../base/cvd/cuttlefish/host/commands/assemble_cvd/disk/os_composite_disk.cc)
   creates a `DiskBuilder` with the partition list, GPT header/footer paths,
   the composite disk output path (`os_composite.img`), and flags for
   read-only mode and resume behaviour.

3. **Composite protobuf generation** — `BuildCompositeDiskIfNecessary()` calls
   `CreateOrUpdateCompositeDisk()` in
   [`image_aggregator.cc`](../base/cvd/cuttlefish/host/libs/image_aggregator/image_aggregator.cc).
   This function:
   - Converts any Android-Sparse images to raw images (`DeAndroidSparse`).
   - Builds a `CompositeDiskBuilder` that computes aligned partition offsets
     and a total virtual disk size.
   - Generates a GPT header (protective MBR + partition table) and GPT footer
     (backup partition table) as binary blobs, writing them to the header and
     footer files.
   - Serialises a `CompositeDisk` protobuf (version 2) that lists each
     component — header, each partition image file (with offset and
     read/write capability), and footer — and writes it to
     `os_composite.img` prefixed by a magic string.
   - If the existing composite disk's protobuf already matches, the file is
     **not** regenerated (preserving timestamps and avoiding unnecessary
     userdata wipes).

4. **Overlay creation** — After the composite disk is (re)built, a qcow2
   overlay (`overlay.img`) is created on top to capture guest writes when the
   composite disk is mounted read-only (`--use_overlay`).

### Why it is necessary for crosvm

crosvm does not accept a directory of loose `.img` files. It expects either a
single raw block device or its native **Composite Disk** format. The composite
disk format lets crosvm present many host-side files as one contiguous virtual
disk with a real GPT, without ever copying data — each component is mapped
at a fixed byte offset. This is efficient (no large copy), supports both
read-only and read-write partitions in the same disk, and gives the guest
kernel a standard GPT-partitioned block device identical to what it would see
on a physical phone.

The separate `InstanceCompositeDisk` ("persistent composite disk") holds
mutable per-boot metadata: the U-Boot environment, persistent vbmeta, FRP
(Factory Reset Protection), and bootconfig. It is also assembled via the
same `DiskBuilder` / `CreateOrUpdateCompositeDisk` mechanism.

---

## 2. Super Image Rebuild

### What it is

The **super image** (`super.img`) is Android's dynamic-partition container.
It uses Linux's device-mapper and `liblp` metadata to multiplex several
logical partitions — `system`, `system_ext`, `product`, `system_dlkm`
(system group, shown in green in the codebase's disk.dot diagram) and
`vendor`, `odm`, `vendor_dlkm`, `odm_dlkm` (vendor group, shown in blue)
— onto a single physical partition. The super image rebuild merges a
**system build** and a **vendor build** from potentially different AOSP
branches into a single, coherent `super.img`.

### What it depends on

| Dependency                                                                 | Purpose                                                                |
| -------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| Vendor target-files zip (`*-target_files-*.zip` from the default build)    | Provides vendor-side partition images and build metadata               |
| System target-files zip (from the system build)                            | Provides system-side partition images and build metadata               |
| `build_super_image` host binary                                            | Assembles the final `super.img` from combined target files             |
| `avbtool`                                                                  | Regenerates `vbmeta.img` with updated hash descriptors after the merge |
| `META/misc_info.txt` and `META/dynamic_partitions_info.txt` from both zips | Partition sizes, group membership, and super-device geometry           |

### How it works

The logic lives in
[`super_image_mixer.cc`](../base/cvd/cuttlefish/host/commands/assemble_cvd/super_image_mixer.cc):

1. **Trigger check** — `SuperImageNeedsRebuilding()` returns `true` only when
   both a default-build and a system-build target-files zip are present
   (either passed via flags or discovered in the fetcher config). If the user
   is running a single, un-mixed build, this step is skipped entirely.

2. **Target-files extraction** — `ExtractTargetFiles()` opens both zips and
   selectively extracts images:
   - _Vendor images_ (boot, dtbo, vendor, vendor_dlkm, odm, recovery, etc.)
     come from the vendor/default zip.
   - _System images_ (system, system_ext, product, etc.) come from the
     system zip.
   - Build props are extracted from the corresponding zip as well.

3. **Metadata merging** — `CombineMiscInfo()` and
   `CombineDynamicPartitionsInfo()` parse `misc_info.txt` and
   `dynamic_partitions_info.txt` from both zips and merge them, producing
   a combined metadata directory that `build_super_image` can consume.

4. **Super image build** — The `build_super_image` host tool reads the
   combined target-files directory and produces the final `super.img` at
   the instance's `new_super_image()` path.

5. **Vbmeta regeneration** — `RegenerateVbmeta()` calls `avbtool` to
   re-sign `vbmeta.img`, incorporating hash descriptors from the newly
   built super partitions and any chained vbmeta partitions.

A second path exists for per-partition repacking (see §4): when a custom
initramfs supplies kernel modules, `RepackSuperWithPartition()` uses the
`lpadd --replace` tool to swap the `vendor_dlkm` and `system_dlkm` logical
partitions inside an existing `super.img` without rebuilding the entire
image.

### Why it is necessary for crosvm

On a real device, the bootloader reads `super` metadata and creates
device-mapper devices for each logical partition before handing control to
the kernel. Cuttlefish emulates this by providing the guest with a `super.img`
that has valid `liblp` metadata and correctly sized logical partitions. If
the system and vendor images come from different builds (the common
"mixed-build" scenario for testing a new platform release against an
older vendor), their partition sizes and group allocations differ; without
rebuilding super, `init` would fail to mount the logical partitions.

---

## 3. Pflash Initialization

### What it is

**Pflash** (programmable flash) is a writable NOR-flash device that stores
UEFI firmware variables. In the context of Cuttlefish, `pflash.img` is a
small (4 MB) raw image file that is concatenated with the bootloader
(U-Boot / UEFI) binary and presented to crosvm (or QEMU) as a second
flash bank.

### What it depends on

| Dependency                                         | Purpose                                                      |
| -------------------------------------------------- | ------------------------------------------------------------ |
| `instance.bootloader()` — the UEFI / U-Boot binary | Determines how much of the 4 MB flash is already occupied    |
| `CreateBlankImage()` utility                       | Creates a zero-filled raw image of the required padding size |

### How it works

The implementation is in
[`pflash.cc`](../base/cvd/cuttlefish/host/commands/assemble_cvd/disk/pflash.cc):

1. If `pflash.img` already exists (e.g. from a previous boot with `--resume`),
   the function returns immediately, preserving any UEFI variables written
   by the guest.

2. Otherwise, the bootloader file size is measured and rounded down to
   megabytes. A blank image of `(4 - bootloader_size_mb)` MB is created at
   `pflash.img` so that **bootloader + pflash = 4 MB** total.

crosvm receives the pflash file via `--pflash=<path>` (x86_64 only), while
the bootloader itself is passed via `--bios=<path>`. QEMU receives both
as `if=pflash` drives (one read-only for the firmware code, one read-write
for variables).

### Why it is necessary for crosvm

UEFI firmware expects two flash regions: a **read-only code region** (the
bootloader/BIOS binary) and a **read-write variable store** (pflash). The
variable store persists settings like the boot order, Secure Boot keys, and
the `BootNext` variable that the Android bootloader control HAL uses. Without
pflash, UEFI firmware running inside crosvm would have nowhere to save
runtime variables, and the boot flow would fail or lose state across reboots.
This is relevant only for x86_64 Cuttlefish instances (ARM64 typically uses
a different firmware interface).

---

## 4. Vendor DLKM Image Construction

### What it is

The **vendor_dlkm** (Dynamic Loadable Kernel Module) partition holds kernel
modules that are loaded during Android's second-stage init — after the
initial ramdisk has brought up the basic block-device and dm-verity stack.
When a custom kernel or custom initramfs is supplied, `assemble_cvd` must
rebuild `vendor_dlkm.img` (and `system_dlkm.img`) to match the modules
actually present, and then replace the corresponding logical partition
inside `super.img`.

### What it depends on

| Dependency                                                             | Purpose                                                |
| ---------------------------------------------------------------------- | ------------------------------------------------------ |
| Custom `initramfs_path` (from `--initramfs_path` flag or kernel build) | Source of all kernel modules before splitting          |
| `modules.load` and `modules.dep` inside the initramfs                  | Module load order and dependency graph                 |
| `mkuserimg_mke2fs` host binary                                         | Builds an ext4 filesystem image from a directory tree  |
| `sefcontext_compile` host binary                                       | Compiles SELinux file-context rules into binary format |
| `avbtool`                                                              | Adds an AVB hashtree footer for dm-verity verification |
| `lpadd` host binary                                                    | Replaces a logical partition inside `super.img`        |

### How it works

The logic spans
[`vendor_dlkm_utils.cc`](../base/cvd/cuttlefish/host/commands/assemble_cvd/vendor_dlkm_utils.cc)
and
[`kernel_ramdisk_repacker.cpp`](../base/cvd/cuttlefish/host/commands/assemble_cvd/disk/kernel_ramdisk_repacker.cpp):

1. **Module splitting** — `SplitRamdiskModules()` unpacks the initramfs (see
   §5), reads `modules.load` and `modules.dep`, and partitions every `.ko`
   file into three buckets:
   - **Ramdisk modules** — a hardcoded set of virtio drivers and networking
     modules needed during first-stage init (e.g. `virtio_blk.ko`,
     `virtio_pci.ko`, `virtio_net.ko`). These stay in the initramfs.
   - **Vendor DLKM modules** — unsigned, non-ramdisk modules. Moved to
     `<superimg_build_dir>/vendor_dlkm/lib/modules/`.
   - **System DLKM modules** — signed (GKI) non-ramdisk modules. Moved to
     `<superimg_build_dir>/system_dlkm/lib/modules/`.

   Updated `modules.dep` and `modules.load` files are written for each
   bucket, and the initramfs is re-packed with only the ramdisk modules.

2. **Image building** — `BuildDlkmImage()` creates a new ext4 image:
   - Generates `filesystem_config.txt` (standard Linux permissions: 0755 for
     dirs, 0644 for files).
   - Generates and compiles SELinux `file_contexts` with the appropriate
     label (`vendor_file` for vendor_dlkm, `system_dlkm_file` for
     system_dlkm).
   - Invokes `mkuserimg_mke2fs` with deterministic UUID, hash seed, and
     timestamp to produce a reproducible ext4 image.
   - Calls `avbtool add_hashtree_footer` to append an AVB hashtree footer,
     enabling dm-verity verification in the guest.

3. **Vbmeta regeneration** — `BuildVbmetaImage()` creates a standalone
   `vbmeta_vendor_dlkm.img` (and `vbmeta_system_dlkm.img`) that includes
   the hash descriptor for the newly built image.

4. **Super image repacking** — `RepackSuperWithPartition()` calls
   `lpadd --replace` to swap the `vendor_dlkm_a` (and `system_dlkm_a`)
   logical partition inside `super.img` with the freshly built image.

### Why it is necessary for crosvm

Cuttlefish often runs with a kernel that differs from the one the AOSP
build produced (e.g. a kernel engineer testing a GKI pre-release). Kernel
modules must exactly match the running kernel's `vermagic` and ABI. If
modules from the old build were left in `vendor_dlkm`, they would fail to
load with version-mismatch errors. By extracting modules from the supplied
initramfs, splitting them into the correct partitions, and rebuilding the
ext4 images with proper AVB footers, `assemble_cvd` guarantees that the
guest's module loading (`modprobe` during second-stage init) succeeds even
when the kernel is swapped.

---

## 5. Initramfs Packing and Unpacking

### What it is

The **initramfs** (initial RAM filesystem) is a cpio archive, typically
LZ4-compressed, embedded in or alongside the kernel. It contains the
first-stage `init`, essential kernel modules (virtio drivers), and the
configuration needed to mount the real root filesystem. During
`assemble_cvd`, the initramfs is **unpacked**, **modified** (kernel
modules are split out; see §4), and **re-packed** into the vendor_boot
image.

### What it depends on

| Dependency                                                              | Purpose                                                  |
| ----------------------------------------------------------------------- | -------------------------------------------------------- |
| `initramfs_path` flag (or the ramdisk extracted from `vendor_boot.img`) | The raw initramfs to modify                              |
| `lz4` host binary                                                       | LZ4 decompression and re-compression                     |
| `cpio` host binary                                                      | Extracting and archiving the cpio filesystem             |
| `mkbootfs` host binary                                                  | Creating a new cpio archive from a directory             |
| `mkbootimg` / `unpack_bootimg` host binaries                            | Unpacking and repacking `boot.img` and `vendor_boot.img` |
| `avbtool`                                                               | Re-signing the boot image after kernel replacement       |

### How it works

The implementation is in
[`boot_image_utils.cc`](../base/cvd/cuttlefish/host/commands/assemble_cvd/boot_image_utils.cc):

#### Unpacking (`UnpackRamdisk`)

1. Detect whether the input file is already a raw cpio (magic `070701`) or
   LZ4-compressed. If compressed, decompress with `lz4 -c -d -l` to produce
   a `.cpio` file.
2. Create a staging directory and pipe the `.cpio` through `cpio -idu` to
   extract the filesystem tree. The loop runs `cpio` repeatedly until it
   returns non-zero (handling concatenated cpio archives, a common pattern
   in Android where the vendor ramdisk is appended to the generic ramdisk).

#### Modification (during `RepackKernelRamdisk`)

- If a custom **kernel** is supplied (`--kernel_path`), `RepackBootImage()`
  replaces the kernel inside `boot.img` (unpacks, swaps the kernel binary,
  re-packs, re-signs with AVB).
- If a custom **initramfs** is supplied (`--initramfs_path`):
  1. The initramfs is copied to a working location (`ramdisk_repacked`).
  2. `SplitRamdiskModules()` (see §4) unpacks it, moves non-essential
     modules to vendor_dlkm and system_dlkm build directories, rewrites
     `modules.dep` and `modules.load`, and re-packs the trimmed ramdisk.
  3. The super image is repacked with the new DLKM images.
  4. `RepackVendorBootImage()` replaces the ramdisk inside
     `vendor_boot.img` with the trimmed, re-packed ramdisk.

#### Packing (`PackRamdisk`)

1. `mkbootfs <staging_dir>` creates a new cpio archive (`.cpio` file).
2. `lz4 -c -l -12 --favor-decSpeed` compresses the cpio archive at
   compression level 12, favouring decompression speed (important for fast
   boot times).

#### Vendor ramdisk repacking (`RepackVendorRamdisk`)

When the new ramdisk contains kernel modules (the `initramfs_path` case),
the original vendor_boot ramdisk is unpacked, its `lib/modules` directory
is removed, the stripped ramdisk is re-packed, and the custom initramfs is
**concatenated** after it. Android's boot protocol expects multiple cpio
archives to be concatenated — the kernel processes them in order, each one
overlaying files on top of the previous.

### Why it is necessary for crosvm

Unlike a physical device where the bootloader loads ramdisks from flash,
crosvm receives boot images as files. The ramdisk embedded in
`vendor_boot.img` (or `boot.img` on older formats) **must** contain the
correct set of virtio drivers for the virtual hardware that crosvm presents:
`virtio_blk` (block devices), `virtio_net` (networking), `virtio_console`
(serial), `virtio_pci`/`virtio_mmio` (transport), `virtio-gpu` (graphics),
and `vsock` (host–guest communication). Without these modules loaded in
first-stage init, the guest cannot mount its root filesystem or communicate
with the host.

When a developer supplies a custom kernel, the corresponding modules are
shipped in a flat initramfs. Assemble_cvd must:

1. Keep only the essential first-stage modules in the ramdisk (so it stays
   small and boots fast).
2. Move everything else into `vendor_dlkm` / `system_dlkm` (so they are
   loaded later via `modprobe` under dm-verity).
3. Update dependency metadata so `modprobe` resolves cross-partition
   dependencies (vendor_dlkm modules may depend on system_dlkm modules via
   `/system/lib/modules/` path prefixes).
4. Re-pack the trimmed ramdisk and embed it in the vendor_boot image so
   crosvm can pass it to the kernel at boot.

---

## Summary of Execution Order

The following shows the order in which `CreateDynamicDiskFiles()` invokes these processes:

```
CreateDynamicDiskFiles()
 ├─ 5. RepackKernelRamdisk()          ← initramfs unpack/repack + module split
 │     └─ 4. RepackSuperAndVbmeta()   ← vendor DLKM image build + super repack
 ├─ 2. RebuildSuperImageIfNecessary() ← super image rebuild (mixed builds)
 ├─ 3. InitializePflash()             ← pflash creation
 ├─ 1. OsCompositeDiskBuilder() +
 │     BuildCompositeDiskIfNecessary() ← OS composite disk construction
 └─    BuildOverlayIfNecessary()       ← qcow2 overlay for read-only composite
```

Note that initramfs repacking (5) runs **before** the super image rebuild (2)
because the module split may produce a new `vendor_dlkm.img` that needs to be
inserted into super. The OS composite disk (1) is assembled last because it
references the final versions of all partition images.
