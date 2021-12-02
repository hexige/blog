---
layout:     post
title:      "ota_from_target_files 分析-差分包部分"
date:       2021-12-02 19:30:00
author:     "hxg_"
comments:	true
header-img: "img/post-bg-01.jpg"
---

## 差分
### 前面的流程，与整包一致
```python
# Generate an incremental OTA.
  else:
    print("unzipping source target-files...")
    OPTIONS.source_tmp = common.UnzipTemp(
        OPTIONS.incremental_source, UNZIP_PATTERN)
    with zipfile.ZipFile(args[0], 'r') as input_zip, \
        zipfile.ZipFile(OPTIONS.incremental_source, 'r') as source_zip:
      WriteBlockIncrementalOTAPackage(
          input_zip,
          source_zip,
          output_file=args[1])

    // 生成diff日志
    if OPTIONS.log_diff:
      with open(OPTIONS.log_diff, 'w') as out_file:
        import target_files_diff
        target_files_diff.recursiveDiff(
            '', OPTIONS.source_tmp, OPTIONS.input_tmp, out_file)
```

### WriteBlockIncrementalOTAPackage流程较长，分几步来看

#### boot.img
```python
# 获取source的boot.img
source_boot = common.GetBootableImage(
      "/tmp/boot.img", "boot.img", OPTIONS.source_tmp, "BOOT", source_info)
# 获取target的boot.img
target_boot = common.GetBootableImage(
      "/tmp/boot.img", "boot.img", OPTIONS.target_tmp, "BOOT", target_info)
# 是否更新boot的判断条件
updating_boot = (not OPTIONS.two_step and
                   (source_boot.data != target_boot.data))
```

#### recovery.img

```python
target_recovery = common.GetBootableImage(
      "/tmp/recovery.img", "recovery.img", OPTIONS.target_tmp,"RECOVERY")
```

#### allow_shared_blocks，默认为false
```python
allow_shared_blocks = (source_info.get('ext4_share_dup_blocks') == "true" or target_info.get('ext4_share_dup_blocks') == "true")
```

#### system.img
```python
system_src = common.GetSparseImage("system", OPTIONS.source_tmp, source_zip,allow_shared_blocks)
system_tgt = common.GetSparseImage("system", OPTIONS.target_tmp, target_zip,allow_shared_blocks)
```

#### 生成system_diff
```python
  # 当system分区为ext4格式的时候，需要检查first block
  system_src_partition = source_info["fstab"]["/system"]
  check_first_block = system_src_partition.fs_type == "ext4"
  # 当system分区为squashfs格式的时候，禁用imagediff
  system_tgt_partition = target_info["fstab"]["/system"]
  disable_imgdiff = (system_src_partition.fs_type == "squashfs" or
                     system_tgt_partition.fs_type == "squashfs")
  # 生成system_diff,此时check_first_block=true,disable_imgdiff=false
  system_diff = common.BlockDifference("system", system_tgt, system_src,
                                       check_first_block,
                                       version=blockimgdiff_version,
                                       disable_imgdiff=disable_imgdiff)
```

#### vendor.img
```python
if HasVendorPartition(target_zip):
    # target中有vendor，但source没有，则失败
    if not HasVendorPartition(source_zip):
      raise RuntimeError("can't generate incremental that adds /vendor")
    vendor_src = common.GetSparseImage("vendor", OPTIONS.source_tmp, source_zip,
                                       allow_shared_blocks)
    vendor_tgt = common.GetSparseImage("vendor", OPTIONS.target_tmp, target_zip,
                                       allow_shared_blocks)

    # ext4逻辑同system分区
    vendor_partition = source_info["fstab"]["/vendor"]
    check_first_block = vendor_partition.fs_type == "ext4"
    disable_imgdiff = vendor_partition.fs_type == "squashfs"
    # check_first_block=true,disable_imgdiff=false
    vendor_diff = common.BlockDifference("vendor", vendor_tgt, vendor_src,
                                         check_first_block,
                                         version=blockimgdiff_version,
                                         disable_imgdiff=disable_imgdiff)
  else:
    vendor_diff = None
```

####  校验信息
```python
  #完整性校验，默认没有
  AddCompatibilityArchiveIfTrebleEnabled(
      target_zip, output_zip, target_info, source_info)

  # Assertions (e.g. device properties check).
  target_info.WriteDeviceAssertions(script, OPTIONS.oem_no_mount)
  device_specific.IncrementalOTA_Assertions()
```

#### script中print fingerprint
```python
script.Print("Source: {}".format(source_info.fingerprint))
script.Print("Target: {}".format(target_info.fingerprint))
```

会在script中有如下信息

```
ui_print("Source: alps/full_puffert10/puffert10:9/PPR1.180610.011/20210903163639712:user/dev-keys");
ui_print("Target: alps/full_puffert10/puffert10:9/PPR1.180610.011/20211012111717858:user/dev-keys");
```

#### script中加入fingerprint的Assert
```python
WriteFingerprintAssertion(script, target_info, source_info)
```
会在script中有如下信息
```
PPR1.180610.011/20210903163639712:user/dev-keys" ||
    getprop("ro.build.fingerprint") == "alps/full_puffert10/puffert10:9/PPR1.180610.011/20211012111717858:user/dev-keys" ||
    abort("E3001: Package expects build fingerprint of alps/full_puffert10/puffert10:9/PPR1.180610.011/20210903163639712:user/dev-keys or alps/full_puffert10/puffert10:9/PPR1.180610.011/20211012111717858:user/dev-keys; this device has " + getprop("ro.build.fingerprint") + ".");
```

#### 计算需要的cache size
```python
  size = []
  if system_diff:
    size.append(system_diff.required_cache)
  if vendor_diff:
    size.append(vendor_diff.required_cache)

  # 是否需要更新boot分区
  if updating_boot:
    boot_type, boot_device = common.GetTypeAndDevice("/boot", source_info)
    d = common.Difference(target_boot, source_boot)
    _, _, d = d.ComputePatch()
    if d is None:
      # 不需要更新，则直接写入boot.img，以raw image的方式写入
      include_full_boot = True
      common.ZipWriteStr(output_zip, "boot.img", target_boot.data)
    else:
      # 以patch/boot.img.p作为source，以applyPatch方式升级.
      # 当boot.img较大时，存在掉电后无法启动的风险.
      include_full_boot = False

      print("boot      target: %d  source: %d  diff: %d" % (
          target_boot.size, source_boot.size, len(d)))

      common.ZipWriteStr(output_zip, "patch/boot.img.p", d)

      script.PatchCheck("%s:%s:%d:%s:%d:%s" %
                        (boot_type, boot_device,
                         source_boot.size, source_boot.sha1,
                         target_boot.size, target_boot.sha1),
                        source_boot.sha1, target_boot.sha1)
      size.append(target_boot.size)
```

#### script中增加cache size的校验
```python
script.CacheFreeSpaceCheck(max(size))
```

会在script中有如下信息

```
pply_patch_space(419930112) || abort("E3006: Not enough free space on /cache to apply patches.");
```

#### 校验当前设备上的system分区
```python
system_diff.WriteVerifyScript(script, touched_blocks_only=True)
```

会在script中有如下信息

```
if (range_sha1("/dev/block/platform/bootdevice/by-name/system", "80,1,418,679,6811,7677,17552,17553,19639,19857,19995,20079,22226,22248,31708,31712,32770,32948,32949,33447,65537,66035,98306,98484,98485,98983,121173,125709,125711,126714,131073,131571,163842,164020,164021,164519,196609,197107,229378,229556,229557,230055,262145,262643,290182,290218,294914,295092,295093,295591,327681,328179,354291,354315,360449,360947,383245,383263,393217,393715,425985,426483,458753,459251,463257,463469,484673,484695,491521,492019,497176,500871,524289,524787,532356,720896,720897,730665,742243,742399,742400") == "5c6963d84683d8d3d56b10eed225e8b95e0518de" || block_image_verify("/dev/block/platform/bootdevice/by-name/system", package_extract_file("system.transfer.list"), "system.new.dat", "system.patch.dat")) then
ui_print("Verified system image...");
else
check_first_block("/dev/block/platform/bootdevice/by-name/system");
ifelse (block_image_recover("/dev/block/platform/bootdevice/by-name/system", "80,1,418,679,6811,7677,17552,17553,19639,19857,19995,20079,22226,22248,31708,31712,32770,32948,32949,33447,65537,66035,98306,98484,98485,98983,121173,125709,125711,126714,131073,131571,163842,164020,164021,164519,196609,197107,229378,229556,229557,230055,262145,262643,290182,290218,294914,295092,295093,295591,327681,328179,354291,354315,360449,360947,383245,383263,393217,393715,425985,426483,458753,459251,463257,463469,484673,484695,491521,492019,497176,500871,524289,524787,532356,720896,720897,730665,742243,742399,742400") && block_image_verify("/dev/block/platform/bootdevice/by-name/system", package_extract_file("system.transfer.list"), "system.new.dat", "system.patch.dat"), ui_print("system recovered successfully."), abort("E1004: system partition fails to recover"));
endif;
```

#### 校验当前设备上的vendor分区
```python
vendor_diff.WriteVerifyScript(script, touched_blocks_only=True)
```

#### system_diff.WriteScript
```python
system_diff.WriteScript(script, output_zip,
                          progress=0.8 if vendor_diff else 0.9)
```

会在script中有如下信息

```
ui_print("Patching system image after verification.");
show_progress(0.800000, 0);
block_image_update("/dev/block/platform/bootdevice/by-name/system", package_extract_file("system.transfer.list"), "system.new.dat", "system.patch.dat") ||
  abort("E1001: Failed to update system image.");
```

#### vendor_diff.WriteScript同上
#### 更新boot分区
```python
if updating_boot:
  # 整包方式更新，即直接像分区写boot.img
  if include_full_boot:
    print("boot image changed; including full.")
    script.Print("Installing boot image...")
    script.WriteRawImage("/boot", "boot.img")
  else:
    # Produce the boot image by applying a patch to the current
    # contents of the boot partition, and write it back to the
    # partition.
    print("boot image changed; including patch.")
    script.Print("Patching boot image...")
    script.ShowProgress(0.1, 10)
    script.ApplyPatch("%s:%s:%d:%s:%d:%s"
                      % (boot_type, boot_device,
                         source_boot.size, source_boot.sha1,
                         target_boot.size, target_boot.sha1),
                      "-",
                      target_boot.size, target_boot.sha1,
                      source_boot.sha1, "patch/boot.img.p")
```

#### wipe userData
```python
if OPTIONS.wipe_user_data:
    script.Print("Erasing user data...")
    script.FormatPartition("/data")
```

#### 签名，写入metadata
```python
# Sign the generated zip package unless no_signing is specified.
needed_property_files = (
  NonAbOtaPropertyFiles(),
)
  
FinalizeMetadata(metadata, staging_file, output_file, needed_property_files)
```