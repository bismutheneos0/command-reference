# Building BismutheneOS for the Galaxy A21s — Command Reference

Full command reference for building BismutheneOS (GrapheneOS/LineageOS-based, Android 16) for the Samsung Galaxy A21s (Exynos 850, kernel 4.19).

**Conventions**
- `$TOP` = root of the Android source tree.
- ✅ = step succeeded. ❌ = step failed; keep it anyway where the failure is instructive. ⚠️ = known footgun.
- Commands are grouped by phase in build order. Run everything from `$TOP` unless stated otherwise.

---

## 1. Source checkout and environment

```bash
repo init -u https://codeberg.org/bismutheneos/platform_manifest.git -b 16-qpr2 --depth=1 --partial-clone
repo sync -c --no-clone-bundle     # ❌ first attempt: HTTP 429 rate limit on one repo (tools/dexter)
repo sync -c --no-clone-bundle     # ✅ retry succeeds; safe to re-run

source build/envsetup.sh
croot

# Android 16 lunch format is <product>-<release>-<variant>
lunch lineage_a21s-cur-userdebug   # ✅
```

Prerequisite: the proprietary vendor tree (`vendor/samsung/a21s`) must be present, or the product makefile fails with `a21s-vendor.mk does not exist`.

---

## 2. Soong bootstrap fixes

The tree mixes GrapheneOS `build/` with LineageOS `vendor/lineage` modules, so the first build fails in Soong's analysis phase.

```bash
m recoveryimage
# ❌ vendor/lineage/build/soong: undefined: android.PathForSourceRelaxed
#    (LineageOS-only API, not present in GrapheneOS soong)
sed -i 's/android.PathForSourceRelaxed/android.PathForSource/g' \
  vendor/lineage/build/soong/generator/generator.go

m recoveryimage
# ❌ unrecognized module type "prebuilt_addon_d" (LineageOS-only module type)
sed -i '/^prebuilt_addon_d {/,/^}/s/^/\/\//' vendor/lineage/prebuilt/common/Android.bp

m recoveryimage
# ❌ Samsung/Pixel-only packages depend on modules that don't exist here
sed -i '/SamsungDoze/d' device/samsung/a21s-common/common.mk
mv hardware/samsung/packages/AdvancedDisplay/Android.bp{,.disabled}
mv hardware/samsung/packages/Doze/Android.bp{,.disabled}
# Note: renaming to .disabled hides the module from Soong, which scans every Android.bp.

# Disable Pixel-only modules that pull in libnos/pixelatoms
for f in $(grep -rl "libnos\|pixelatoms" hardware/google/pixel/*/Android.bp 2>/dev/null); do
  mv "$f" "${f}.disabled"
done
mv hardware/google/pixel/pixelstats/test/PixelAtomsTest/Android.bp{,.disabled}
for f in hardware/samsung/aidl/livedisplay/Android.bp \
         hardware/samsung/aidl/powershare/Android.bp \
         hardware/samsung/aidl/touch/Android.bp \
         hardware/samsung/aidl/usb/gadget/Android.bp; do
  mv "$f" "${f}.disabled"
done
mv hardware/google/pixel/pixelstats/test/mm/Android.bp{,.disabled}
mv hardware/google/pixel/touch/Android.bp{,.disabled}

# libsec-ril (prebuilt) references a versioned protobuf name that doesn't exist in this tree
grep -r "name:.*libprotobuf-cpp-full" external/protobuf/
sed -i 's/libprotobuf-cpp-full-21.7/libprotobuf-cpp-full/g' vendor/samsung/a21s-common/Android.bp

# Final approach for the remaining gaps: allow missing deps globally, restore everything
echo 'ALLOW_MISSING_DEPENDENCIES := true' >> device/samsung/a21s-common/BoardConfigCommon.mk
for f in $(find hardware/google/pixel hardware/samsung -name "Android.bp.disabled"); do
  mv "$f" "${f%.disabled}"
done

# GrapheneOS build/soong hides soong-shared from vendor modules; make it public
python3 scripts/fix_soong_visibility.py   # see Appendix A.4

m recoveryimage
# ❌ lineage_generator cmd: unknown variable '$(KERNEL_BUILD_OUT_PREFIX)' → next section
```

---

## 3. lineageVarsPlugin registration and zstd removal

`vendor/lineage/build/soong` expects a Soong config namespace that LineageOS's own `build/` fork registers. With GrapheneOS `build/` it must be declared in the device tree:

```bash
cat >> device/samsung/a21s-common/BoardConfigCommon.mk << 'EOF'
SOONG_CONFIG_NAMESPACES += lineageVarsPlugin
SOONG_CONFIG_lineageVarsPlugin := \
    KERNEL_BUILD_OUT_PREFIX TARGET_KERNEL_PLATFORM_TARGET KERNEL_PATH KERNEL_ARCH \
    TARGET_KERNEL_SOURCE KERNEL_MAKE_CMD KERNEL_MAKE_FLAGS KERNEL_CROSS_COMPILE \
    PATH_OVERRIDE_SOONG TARGET_PREBUILT_KERNEL_HEADERS
SOONG_CONFIG_lineageVarsPlugin_KERNEL_BUILD_OUT_PREFIX :=
SOONG_CONFIG_lineageVarsPlugin_TARGET_KERNEL_PLATFORM_TARGET :=
SOONG_CONFIG_lineageVarsPlugin_KERNEL_PATH :=
SOONG_CONFIG_lineageVarsPlugin_KERNEL_ARCH := arm64
SOONG_CONFIG_lineageVarsPlugin_TARGET_KERNEL_SOURCE := $(TARGET_KERNEL_SOURCE)
SOONG_CONFIG_lineageVarsPlugin_KERNEL_MAKE_CMD := make
SOONG_CONFIG_lineageVarsPlugin_KERNEL_MAKE_FLAGS :=
SOONG_CONFIG_lineageVarsPlugin_KERNEL_CROSS_COMPILE :=
SOONG_CONFIG_lineageVarsPlugin_PATH_OVERRIDE_SOONG :=
SOONG_CONFIG_lineageVarsPlugin_TARGET_PREBUILT_KERNEL_HEADERS :=
EOF

m recoveryimage
# ❌ Host modules should be in PRODUCT_HOST_PACKAGES, not PRODUCT_PACKAGES (zstd)
grep -rn "zstd" --include="*.mk" vendor/lineage/ | grep -i packages
sed -n '25,40p' vendor/lineage/config/common_mobile_full.mk
sed -i '/zstd/d' vendor/lineage/config/common_mobile_full.mk
echo 'PRODUCT_PACKAGES := $(filter-out zstd,$(PRODUCT_PACKAGES))' >> device/samsung/a21s-common/common.mk
```

---

## 4. Kernel: manual build + prebuilt injection

Integrating `vendor/lineage/build/tasks/kernel.mk` into this hybrid tree proved unreliable (see Appendix C.1), so the kernel is built manually with the tree's own clang and injected as a prebuilt.

### 4.1 Standard toolchain invocation

Used verbatim throughout; only the final make target changes.

```bash
cd $TOP && TOP=$(pwd)
cd kernel/samsung/exynos850
make O=out ARCH=arm64 \
  CC="$TOP/prebuilts/clang/host/linux-x86/clang-r574158/bin/clang" \
  LD="$TOP/prebuilts/clang/host/linux-x86/clang-r574158/bin/ld.lld" \
  AR="$TOP/prebuilts/clang/host/linux-x86/clang-r574158/bin/llvm-ar" \
  NM="$TOP/prebuilts/clang/host/linux-x86/clang-r574158/bin/llvm-nm" \
  STRIP="$TOP/prebuilts/clang/host/linux-x86/clang-r574158/bin/llvm-strip" \
  OBJCOPY="$TOP/prebuilts/clang/host/linux-x86/clang-r574158/bin/llvm-objcopy" \
  OBJDUMP="$TOP/prebuilts/clang/host/linux-x86/clang-r574158/bin/llvm-objdump" \
  CLANG_TRIPLE=aarch64-linux-gnu- CROSS_COMPILE=aarch64-linux-gnu- \
  -j$(nproc) <target>
```

### 4.2 Build

```bash
make O=out ARCH=arm64 ... exynos850-a21snsxx_defconfig   # generates out/.config
make -j$(nproc) O=out ARCH=arm64 ... Image
# ❌ link error: relocation R_AARCH64_ABS32 against '__crc_*' (CONFIG_MODVERSIONS)
sed -i 's/CONFIG_MODVERSIONS=y/# CONFIG_MODVERSIONS is not set/' out/.config
make -j$(nproc) O=out ARCH=arm64 ... Image               # ✅
make -j$(nproc) O=out ARCH=arm64 ... dtbs                # ✅
```

⚠️ Open issue: the framework compatibility matrix at level 6 requires `CONFIG_MODVERSIONS=y`, but enabling it triggers the `__crc_*` link error above on this Samsung 4.19 + clang-r574158 combination. The kernel that currently boots has MODVERSIONS disabled; a proper fix for the CRC relocation is still needed.

### 4.3 Package dtb.img / dtbo.img

```bash
cd $TOP
MKDTBOIMG=system/libufdt/utils/src/mkdtboimg.py
DTB_DIR=kernel/samsung/exynos850/out/arch/arm64/boot/dts
python3 $MKDTBOIMG cfg_create dtb.img  device/samsung/a21s-common/configs/kernel/exynos850.cfg -d $DTB_DIR
python3 $MKDTBOIMG cfg_create dtbo.img device/samsung/a21s-common/configs/kernel/a21s.cfg -d $DTB_DIR
python3 $MKDTBOIMG dump dtb.img      # verify table header/entries
python3 $MKDTBOIMG dump dtbo.img
cp kernel/samsung/exynos850/out/arch/arm64/boot/Image device/samsung/a21s/prebuilt/
cp dtb.img dtbo.img device/samsung/a21s/prebuilt/
```

⚠️ These are relative paths; they only work from `$TOP`.

### 4.4 BoardConfig (device/samsung/a21s/BoardConfig.mk)

```makefile
## Kernel (prebuilt — built out-of-tree and injected)
TARGET_PREBUILT_KERNEL := $(DEVICE_PATH)/prebuilt/Image
BOARD_PREBUILT_DTBOIMAGE := $(DEVICE_PATH)/prebuilt/dtbo.img
TARGET_KERNEL_SOURCE :=
TARGET_KERNEL_CONFIG :=
BOARD_DTB_CFG :=
BOARD_DTBO_CFG :=

## dtb.img copy rule (BOARD_CUSTOM_DTBIMG_MK only applies when
## FULL_KERNEL_BUILD=true, which is not the case with a prebuilt kernel)
$(PRODUCT_OUT)/dtb.img: $(DEVICE_PATH)/prebuilt/dtb.img
	$(hide) echo "Copying prebuilt dtb.img"
	$(hide) cp -f $< $@
```

---

## 5. sepolicy fixes → recovery image

GrapheneOS's `build/` does not auto-include the LineageOS common sepolicy, so it must be pulled in explicitly, then conflicts with AOSP's `system/sepolicy` resolved.

```makefile
# BoardConfigCommon.mk
$(eval include device/lineage/sepolicy/common/sepolicy.mk)
include device/lineage/sepolicy/exynos/sepolicy.mk
include device/samsung_slsi/sepolicy/sepolicy.mk
```

```bash
# LiveDisplay HAL domain definition (device-specific, kept; attribute decls come from common/dynamic)
cat > device/lineage/sepolicy/exynos/vendor/hal_lineage_livedisplay_exynos.te << 'EOF'
type hal_lineage_livedisplay_exynos, domain;
hal_server_domain(hal_lineage_livedisplay_exynos, hal_lineage_livedisplay)
type hal_lineage_livedisplay_exynos_exec, exec_type, vendor_file_type, file_type;
init_daemon_domain(hal_lineage_livedisplay_exynos)
binder_call(hal_lineage_livedisplay_exynos, servicemanager)
vndbinder_use(hal_lineage_livedisplay_exynos)
allow hal_lineage_livedisplay_exynos display_vendor_data_file:dir rw_dir_perms;
allow hal_lineage_livedisplay_exynos display_vendor_data_file:file create_file_perms;
EOF

# Duplicate type declarations between LineageOS common/private and AOSP private:
grep -l "^type " device/lineage/sepolicy/common/private/*.te
grep -r "^type mkfs,\|^type mediashell_app\|^type updater_app" system/sepolicy/
rm device/lineage/sepolicy/common/private/updater_app.te   # AOSP already declares the type

# neverallow violations: drop the offending rules rather than weakening AOSP neverallows
cat device/lineage/sepolicy/common/private/{fsck_untrusted,recovery,vold}.te
rm device/lineage/sepolicy/common/private/fsck_untrusted.te
cat > device/lineage/sepolicy/common/private/recovery.te << 'EOF'
recovery_only(`
userdebug_or_eng(`
permissive recovery;
')
get_prop(recovery, custom_version_prop)
allow recovery node:tcp_socket node_bind;
allow recovery port:tcp_socket name_bind;
allow recovery self:tcp_socket { create_socket_perms_no_ioctl listen accept };
# vsock_socket removed — violates AOSP neverallow (domain.te:2350)
get_prop(recovery, adbd_config_prop)
r_dir_file(recovery, sdcard_type)
allow recovery block_device:dir create_dir_perms;
allow recovery block_device:blk_file { create unlink rw_file_perms };
allow recovery self:capability fsetid;
# mknod removed — violates AOSP neverallow (domain.te:665)
allow recovery proc_filesystems:file r_file_perms;
allow recovery self:netlink_kobject_uevent_socket create_socket_perms_no_ioctl;
allow recovery sysfs:file w_file_perms;
allow recovery tmpfs:file link;
allow recovery rootfs:dir w_dir_perms;
# rootfs:file write removed — violates AOSP neverallow (domain.te:800)
allow recovery media_rw_data_file:dir r_dir_perms;
allow recovery sysfs_perdev_minors:file r_file_perms;
allowxperm recovery block_device:blk_file ioctl { HDIO_GETGEO BLKGETSIZE };
r_dir_file(recovery, unencrypted_data_file)
')
EOF
cat > device/lineage/sepolicy/common/private/vold.te << 'EOF'
allow vold self:capability { setgid setuid };
allow vold mkfs_exec:file rx_file_perms;
allow vold mnt_media_rw_stub_file:dir r_dir_perms;
allow vold storage_stub_file:dir rw_dir_perms;
# relabelto removed — violates AOSP neverallow (domain.te:804)
allow vold sdcard_posix:filesystem relabelfrom;
allow vold labeledfs:filesystem relabelfrom;
EOF

m recoveryimage                    # ✅

# Verify the result with the header parser (Appendix A.2)
python3 scripts/bootimg_info.py out/target/product/a21s/recovery.img
# Expected: header v2, kernel ≈ 31.4 MB (matches prebuilt Image),
# dtbo ≈ 730 KB, dtb ≈ 138 KB.
```

---

## 6. Full system build (`m`)

```bash
# Settings overlay references a string GrapheneOS Settings doesn't ship
grep -rn "backup_transport_setting_label" vendor/lineage/overlay/ packages/apps/Settings/ 2>/dev/null
cat > vendor/lineage/overlay/no-rro/packages/apps/Settings/res/values/config.xml << 'EOF'
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="config_backup_settings_intent" translatable="false">settings://com.android.settings.backup.transport</string>
    <string name="backup_transport_setting_label" translatable="false">Backup</string>
    <string name="config_backup_settings_label" translatable="true">@string/backup_transport_setting_label</string>
</resources>
EOF

m
# ❌ gen-bootanimation.zip missing dependencies: mogrify
sudo pacman -S imagemagick
# Soong caches host-tool detection; clear the cache after installing
rm -f out/soong/soong.environment.available
rm -f out/soong/soong.environment.used.lineage_a21s.build
rm -f out/soong/soong.lineage_a21s.variables
# The genrule requires mogrify as a build-tools path tool → symlink it there
ln -s /usr/bin/mogrify prebuilts/build-tools/path/linux-x86/mogrify
rm -rf out/soong/.intermediates/vendor/lineage/bootanimation/

# Remove the bootanimation packages from LineageOS common.mk (one multiline match,
# not line-by-line deletion — see Appendix B)
sed -i '/^PRODUCT_PACKAGES += \\$/{N;/bootanimation\.zip/{N;/bootanimation-dark\.zip/d}}' vendor/lineage/config/common.mk
grep "bootanimation" vendor/lineage/config/common.mk   # expect no output

m
# ❌ libwvhidl missing libprotobuf-cpp-lite-3.9.1-vendorcompat
#    (pruned from Soong by a negative PRODUCT_SOURCE_ROOT_DIRS entry)
grep -rn "PRODUCT_SOURCE_ROOT_DIRS" vendor/lineage/config/common.mk
sed -i '/PRODUCT_SOURCE_ROOT_DIRS += -prebuilts\/misc\/protobuf_vendorcompat/d' vendor/lineage/config/common.mk

# mke2fs fails: too many inodes (2^64-1) — build_image passes "-1" and it isn't filtered.
# Patch the Python script (Appendix A.5) and shim the prebuilt ELF with a wrapper:
python3 scripts/patch_mkuserimg_mke2fs.py
mv out/host/linux-x86/bin/mkuserimg_mke2fs out/host/linux-x86/bin/mkuserimg_mke2fs.bak
cat > out/host/linux-x86/bin/mkuserimg_mke2fs << 'EOF'
#!/bin/bash
exec python3 system/extras/ext4_utils/mkuserimg_mke2fs.py "$@"
EOF
chmod +x out/host/linux-x86/bin/mkuserimg_mke2fs
out/host/linux-x86/bin/mkuserimg_mke2fs --help 2>&1 | head -3   # sanity check

# libsec-ril prebuilt has an undeclared DT_NEEDED (old protobuf soname)
grep -rn "libsec-ril" vendor/samsung/a21s-common/Android.bp
sed -i '/name: "libsec-ril",/a\    check_elf_files: false,' vendor/samsung/a21s-common/Android.bp

m   # ✅ produces system/vendor/product/system_ext/odm images, boot.img, super image
```

Post-build sanity checks:

```bash
ls out/host/linux-x86/bin/lpmake
ls -la out/target/product/a21s/{system,vendor,product,system_ext,odm}.img
grep "BOARD_SUPER_PARTITION\|BOARD_SAMSUNG_DYNAMIC_PARTITIONS" device/samsung/a21s-common/BoardConfigCommon.mk
python3 scripts/bootimg_info.py out/target/product/a21s/boot.img
```

---

## 7. Boot failure debugging (abox audio deadlock)

First flash boots for ~2 s, then panics: `abox_register_extra_sound_card` blocks on a mutex owned by `kworker/0:0`, and the hung-task watchdog panics. The kernel itself (incl. hardened_malloc on the 39-bit VAS) is fine — confirmed by checking for mmap/allocator failure signatures and finding none.

### 7.1 Hybrid recovery image

The LineageOS recovery ramdisk is needed (GrapheneOS recovery lacks `libvolume_manager` for Samsung dynamic partitions). The declarative override (`INSTALLED_RECOVERY_RAMDISK_TARGET`) was silently ignored by the build system, so the image is assembled manually with the script in Appendix A.3:

```bash
python3 scripts/extract_ramdisk.py lineage_recovery.img \
  device/samsung/a21s/prebuilt/recovery_ramdisk.cpio.gz   # one-time extraction

python3 scripts/make_hybrid_recovery.py \
  out/target/product/a21s/boot.img lineage_recovery.img recovery_hybrid.img
# Output: 45.1 MB — our kernel/dtb/dtbo + LineageOS ramdisk (13.6 MB)
```

### 7.2 Keep the device alive after the audio deadlock

```bash
cd kernel/samsung/exynos850
# Hung-task watchdog: log instead of panic
sed -i 's/CONFIG_BOOTPARAM_HUNG_TASK_PANIC=y/# CONFIG_BOOTPARAM_HUNG_TASK_PANIC is not set/' out/.config
sed -i 's/CONFIG_BOOTPARAM_HUNG_TASK_PANIC_VALUE=1/CONFIG_BOOTPARAM_HUNG_TASK_PANIC_VALUE=0/' out/.config
# Samsung sec_debug: don't BUG() on workqueue lockup / init exit
sed -i 's/CONFIG_SEC_DEBUG_WORKQUEUE_LOCKUP_PANIC=y/# CONFIG_SEC_DEBUG_WORKQUEUE_LOCKUP_PANIC is not set/' out/.config
sed -i 's/CONFIG_SEC_DEBUG_INIT_EXIT_PANIC=y/# CONFIG_SEC_DEBUG_INIT_EXIT_PANIC is not set/' out/.config
make O=out ARCH=arm64 olddefconfig ...
grep "HUNG_TASK\|SEC_DEBUG_WORKQUEUE\|SEC_DEBUG_INIT_EXIT" out/.config
make O=out ARCH=arm64 -j$(nproc) Image ...
cp out/arch/arm64/boot/Image $TOP/device/samsung/a21s/prebuilt/Image
cd $TOP && m bootimage
```

### 7.3 Preserve pstore across the panic

Samsung's debug-snapshot handler and the CPU-stop path dump large amounts of state on panic, which can evict the ramoops log. Two small source patches quiet them (marked with `BismutheneOS:` comments):

```bash
cd kernel/samsung/exynos850
grep -n "dbg_snapshot_dump_task_info" lib/debug-snapshot-helper.c
sed -i 's/^\t*dbg_snapshot_dump_task_info();/\t\/\/ dbg_snapshot_dump_task_info(); \/\/ BismutheneOS: skip task dump to preserve pstore/' lib/debug-snapshot-helper.c

grep -n "dump_stack\|IPI_CPU_STOP" arch/arm64/kernel/smp.c
# ipi_cpu_stop() calls dump_stack() during panic stop — comment it out
sed -i '880s/dump_stack();/\/\/ dump_stack(); \/\/ BismutheneOS: suppress CPU stop trace to preserve pstore/' arch/arm64/kernel/smp.c
sed -n '875,885p' arch/arm64/kernel/smp.c   # verify the edit landed on the right line

make O=out ARCH=arm64 -j$(nproc) Image ...
cp out/arch/arm64/boot/Image $TOP/device/samsung/a21s/prebuilt/Image
cd $TOP && m bootimage_debug
```

### 7.4 Verify hardened_malloc is actually shipped

```bash
grep -A4 "CONFIG_CLASS_REGION_SIZE" external/hardened_malloc/Android.bp
#   -DCONFIG_CLASS_REGION_SIZE=4294967296   (4 GiB region for 39-bit VAS)
#   -DN_ARENA=1  -DCONFIG_STATS=true  -DCONFIG_SELF_INIT=false

# libc lives in the runtime APEX — extract and check for allocator markers
mkdir -p /tmp/apex_inspect && cd /tmp/apex_inspect
cp $TOP/out/target/product/a21s/system/apex/com.android.runtime.apex .
unzip -o com.android.runtime.apex -d apex_contents/
debugfs apex_contents/apex_payload.img -R "ls lib64/bionic"
debugfs apex_contents/apex_payload.img -R "dump lib64/bionic/libc.so /tmp/libc_from_apex.so"
strings /tmp/libc_from_apex.so | grep -i "slab\|hardened"
# ✅ contains "failed to allocate slab region", "hardened_malloc: fatal allocator error: %s", ...
cd $TOP
```

---

## Appendix A — Python utilities

### A.1 `extract_ramdisk.py`

Extracts the ramdisk from a boot-image-format file (used to pull the LineageOS recovery ramdisk).

```python
#!/usr/bin/env python3
import struct, sys

src, dst = sys.argv[1], sys.argv[2]
with open(src, 'rb') as f:
    f.seek(8);  ks = struct.unpack('<I', f.read(4))[0]   # kernel size
    f.seek(16); rs = struct.unpack('<I', f.read(4))[0]   # ramdisk size
    f.seek(36); ps = struct.unpack('<I', f.read(4))[0]   # page size
    kernel_pages = (ks + ps - 1) // ps
    off = (1 + kernel_pages) * ps
    f.seek(off)
    with open(dst, 'wb') as o:
        o.write(f.read(rs))
    print(f'Ramdisk: {rs} bytes ({rs/1024/1024:.1f} MB)')
```

### A.2 `bootimg_info.py`

Parses a boot image header (v2 layout including recovery_dtbo and dtb fields).

```python
#!/usr/bin/env python3
import struct, sys

with open(sys.argv[1], 'rb') as f:
    magic = f.read(8)
    kernel_size = struct.unpack('<I', f.read(4))[0]      # offset 8
    kernel_addr = struct.unpack('<I', f.read(4))[0]
    ramdisk_size = struct.unpack('<I', f.read(4))[0]     # offset 16
    ramdisk_addr = struct.unpack('<I', f.read(4))[0]
    second_size = struct.unpack('<I', f.read(4))[0]
    second_addr = struct.unpack('<I', f.read(4))[0]
    tags_addr = struct.unpack('<I', f.read(4))[0]
    f.seek(36);   page_size = struct.unpack('<I', f.read(4))[0]
    f.seek(40);   header_version = struct.unpack('<I', f.read(4))[0]
    f.seek(64);   cmdline = f.read(512).rstrip(b'\x00').decode('ascii', 'replace')
    f.seek(1632); recovery_dtbo_size = struct.unpack('<I', f.read(4))[0]
    f.seek(1648); dtb_size = struct.unpack('<I', f.read(4))[0]

print(f'Magic:          {magic}')
print(f'Kernel:         {kernel_size:>10} bytes ({kernel_size/1024/1024:.1f} MB) @ 0x{kernel_addr:08X}')
print(f'Ramdisk:        {ramdisk_size:>10} bytes ({ramdisk_size/1024/1024:.1f} MB) @ 0x{ramdisk_addr:08X}')
print(f'Second:         {second_size:>10} bytes')
print(f'Page size:      {page_size}')
print(f'Header version: {header_version}')
print(f'Cmdline:        {cmdline}')
print(f'Recovery DTBO:  {recovery_dtbo_size:>10} bytes ({recovery_dtbo_size/1024:.1f} KB)')
print(f'DTB:            {dtb_size:>10} bytes ({dtb_size/1024:.1f} KB)')
```

### A.3 `make_hybrid_recovery.py`

Reassembles a recovery image from our boot.img (kernel, dtb, dtbo, header) plus a foreign ramdisk (e.g. LineageOS recovery). Preserves page alignment and updates header sizes.

```python
#!/usr/bin/env python3
import struct, sys

def read_boot_img(path):
    with open(path, 'rb') as f:
        data = f.read()
    page_size = struct.unpack('<I', data[36:40])[0]
    kernel_size = struct.unpack('<I', data[8:12])[0]
    ramdisk_size = struct.unpack('<I', data[16:20])[0]
    second_size = struct.unpack('<I', data[24:28])[0]
    recovery_dtbo_size = struct.unpack('<I', data[1632:1636])[0]
    dtb_size = struct.unpack('<I', data[1648:1652])[0]

    kernel_pages = (kernel_size + page_size - 1) // page_size
    ramdisk_pages = (ramdisk_size + page_size - 1) // page_size
    second_pages = (second_size + page_size - 1) // page_size

    kernel_offset = page_size
    ramdisk_offset = (1 + kernel_pages) * page_size
    second_offset = (1 + kernel_pages + ramdisk_pages) * page_size
    recovery_dtbo_offset = (1 + kernel_pages + ramdisk_pages + second_pages) * page_size
    dtb_offset = recovery_dtbo_offset + \
        ((recovery_dtbo_size + page_size - 1) // page_size) * page_size

    return {
        'header': data[0:page_size],
        'kernel': data[kernel_offset:kernel_offset + kernel_size],
        'ramdisk': data[ramdisk_offset:ramdisk_offset + ramdisk_size],
        'second': data[second_offset:second_offset + second_size],
        'recovery_dtbo': data[recovery_dtbo_offset:recovery_dtbo_offset + recovery_dtbo_size],
        'dtb': data[dtb_offset:dtb_offset + dtb_size],
        'page_size': page_size,
    }

def write_boot_img(path, parts):
    page_size = parts['page_size']
    kernel_size = len(parts['kernel'])
    ramdisk_size = len(parts['ramdisk'])
    second_size = len(parts['second'])
    recovery_dtbo_size = len(parts['recovery_dtbo'])
    dtb_size = len(parts['dtb'])

    header = bytearray(parts['header'])
    struct.pack_into('<I', header, 8, kernel_size)
    struct.pack_into('<I', header, 16, ramdisk_size)

    kernel_pages = (kernel_size + page_size - 1) // page_size
    ramdisk_pages = (ramdisk_size + page_size - 1) // page_size
    second_pages = (second_size + page_size - 1) // page_size
    recovery_dtbo_pages = (recovery_dtbo_size + page_size - 1) // page_size
    dtb_pages = (dtb_size + page_size - 1) // page_size

    total = (1 + kernel_pages + ramdisk_pages + second_pages +
             recovery_dtbo_pages + dtb_pages) * page_size

    with open(path, 'wb') as f:
        f.write(header)
        f.write(b'\x00' * (page_size - len(header)))
        f.write(parts['kernel'])
        f.write(b'\x00' * (kernel_pages * page_size - kernel_size))
        f.write(parts['ramdisk'])
        f.write(b'\x00' * (ramdisk_pages * page_size - ramdisk_size))
        if second_size > 0:
            f.write(parts['second'])
            f.write(b'\x00' * (second_pages * page_size - second_size))
        if recovery_dtbo_size > 0:
            f.write(parts['recovery_dtbo'])
            f.write(b'\x00' * (recovery_dtbo_pages * page_size - recovery_dtbo_size))
        if dtb_size > 0:
            f.write(parts['dtb'])
            f.write(b'\x00' * (dtb_pages * page_size - dtb_size))

    print(f'Wrote {path}: {total/1024/1024:.1f} MB '
          f'(kernel {kernel_size/1024/1024:.1f} MB, ramdisk {ramdisk_size/1024/1024:.1f} MB)')

if __name__ == '__main__':
    base_img, donor_img, out_img = sys.argv[1], sys.argv[2], sys.argv[3]
    mine, donor = read_boot_img(base_img), read_boot_img(donor_img)
    write_boot_img(out_img, {
        'header': mine['header'],
        'kernel': mine['kernel'],
        'ramdisk': donor['ramdisk'],     # foreign ramdisk (e.g. LineageOS recovery)
        'second': mine['second'],
        'recovery_dtbo': mine['recovery_dtbo'],
        'dtb': mine['dtb'],
        'page_size': mine['page_size'],
    })
```

### A.4 `fix_soong_visibility.py`

GrapheneOS's `build/soong/shared` is not visible to vendor modules; add public visibility.

```python
#!/usr/bin/env python3
path = 'build/soong/shared/Android.bp'
with open(path) as f:
    content = f.read()
content = content.replace(
    '    name: "soong-shared",',
    '    name: "soong-shared",\n    visibility: ["//visibility:public"],'
)
with open(path, 'w') as f:
    f.write(content)
```

### A.5 `patch_mkuserimg_mke2fs.py`

`build_image` passes `-1` for inode count; argparse keeps it as the string `"-1"`, and an int comparison never matches, so mke2fs receives `-N -1` (= 2⁶⁴−1 inodes). Skip the flag for `-1`.

```python
#!/usr/bin/env python3
path = 'system/extras/ext4_utils/mkuserimg_mke2fs.py'
with open(path) as f:
    content = f.read()
content = content.replace(
    '    if args.inodes != -1:',
    '    if str(args.inodes) != "-1":'
)
with open(path, 'w') as f:
    f.write(content)
```

Result:

```python
if args.inodes:
    if str(args.inodes) != "-1":
        mke2fs_opts += ["-N", args.inodes]
```

Note: `out/host/linux-x86/bin/mkuserimg_mke2fs` is a prebuilt ELF, not this script; patching the source alone does not change the binary's behavior. The build uses the wrapper shown in section 6.

### A.6 `patch_lineage_kernel_mk.py` (historical — superseded)

Part of the abandoned in-tree kernel integration (Appendix C.1). Kept for reference.

```python
#!/usr/bin/env python3
path = 'vendor/lineage/build/tasks/kernel.mk'
with open(path) as f:
    content = f.read()

# (1) Ninja can't have a directory as a target: $(KERNEL_OUT) -> stamp file
content = content.replace(
    "$(KERNEL_OUT):\n\tmkdir -p $(KERNEL_OUT)",
    "$(KERNEL_OUT)/.dir:\n\tmkdir -p $(KERNEL_OUT)\n\ttouch $(KERNEL_OUT)/.dir")
content = content.replace(
    "$(KERNEL_OUT) $(ALL_KERNEL_DEFCONFIG_SRCS)",
    "$(KERNEL_OUT)/.dir $(ALL_KERNEL_DEFCONFIG_SRCS)")

# (2) PATH_OVERRIDE used relative paths that broke under sandboxing;
#     merge into one absolute PATH=
content = content.replace(
    'PATH_OVERRIDE := PATH=$(KERNEL_BUILD_OUT_PREFIX)$(HOST_OUT_EXECUTABLES):$$PATH',
    'PATH_OVERRIDE := PATH=$(BUILD_TOP)/$(KERNEL_BUILD_OUT_PREFIX)$(HOST_OUT_EXECUTABLES):$$PATH')
content = content.replace(
    'PATH_OVERRIDE += PATH=$(TARGET_KERNEL_CLANG_PATH)/bin:$$PATH',
    'PATH_OVERRIDE := PATH=$(BUILD_TOP)/$(KERNEL_BUILD_OUT_PREFIX)$(HOST_OUT_EXECUTABLES):'
    '$(BUILD_TOP)/$(TARGET_KERNEL_CLANG_PATH)/bin:$$PATH')
content = content.replace(
    'PATH_OVERRIDE += LD_LIBRARY_PATH=$(TARGET_KERNEL_CLANG_PATH)/lib64:$$LD_LIBRARY_PATH',
    'PATH_OVERRIDE += LD_LIBRARY_PATH=$(BUILD_TOP)/$(TARGET_KERNEL_CLANG_PATH)/lib64:$$LD_LIBRARY_PATH')

with open(path, 'w') as f:
    f.write(content)
```

---

## Appendix B — Pitfalls

1. **Working directory matters.** `m` and every relative-path command in this document must run from `$TOP`. Several failures trace back to running from `kernel/…` or `/tmp` instead.
2. **Soong caches host-tool detection.** After installing a host tool, delete `out/soong/soong.environment.*` or the cached "missing" result persists.
3. **Soong scans every `Android.bp`.** Removing a package from `PRODUCT_PACKAGES` does not stop analysis of its module — rename the file to `.disabled` (or use `ALLOW_MISSING_DEPENDENCIES`).
4. **`sed -i '<line>d'` on makefiles is risky.** A line number that was right five minutes ago may now delete part of an `if/endif` or `PRODUCT_PACKAGES` block. Match content, or match multiple lines with `{N;...}`. `git checkout -- <file>` is the undo button.
5. **Prebuilt host tools aren't source scripts.** Patching `system/extras/ext4_utils/mkuserimg_mke2fs.py` does nothing while the build calls a prebuilt ELF of the same name — check with `file` first.
6. **Negative `PRODUCT_SOURCE_ROOT_DIRS` prunes whole trees from Soong**, producing misleading "missing dependencies" errors for modules that clearly exist on disk.
7. **Don't fix neverallow violations by weakening the neverallow.** Remove the offending rule in the vendor policy instead.

## Appendix C — Abandoned approaches

| Approach | Why it was abandoned | Superseded by |
|---|---|---|
| In-tree kernel build via LineageOS `kernel.mk` (incl. the A.6 patch) | GrapheneOS `build/` + sandbox PATH rules + relative paths made it unreliable (`make` disallowed PATH tool; clang not found) | Out-of-tree manual build + prebuilt injection (§4) |
| `INSTALLED_RECOVERY_RAMDISK_TARGET` override in BoardConfig | Silently ignored; recovery ramdisk stayed the source-built 12.7 MB one | Hybrid assembly with A.3 (§7.1) |
| Regenerating defconfig to add `WQ_WATCHDOG` debug options | Broke the build (`secdbg_show_sched_info` implicit declaration); deps live in excluded drivers | Reverted; edited `.config` directly instead |
| Hard-patching `hung_task.c` sysctl defaults to 0 | Source-level hack with no lasting advantage over config/cmdline | Reverted via `git checkout` |
| Enlarging `log_pstore` 32 KB → 512 KB in `exynos3830.dts` | Accidentally reverted in a later `git checkout` | Not yet reapplied — redo deliberately if needed |
| Declaring `sdcard_posix_contextmount_type` in public `attributes.te` | Wrong layer; root cause was LineageOS `common/private` conflicting with AOSP | Skip `common/private` duplicates instead (§5) |
| Disabling every broken module via `.disabled` | Whack-a-mole; dependencies kept surfacing one by one | `ALLOW_MISSING_DEPENDENCIES := true` + restore all (§2) |

## Appendix D — The debugging loop

Every fix in this document came out of one loop:

```
Read the error → grep for who defines/depends on the symbol → patch with sed/script → rebuild to verify
```
