---
layout:     post
title:      "ota_from_target_files 分析-整包部分"
date:       2021-11-25 11:30:00
author:     "hxg_"
comments:	true
header-img: "img/post-bg-01.jpg"
---

### 调用

```
build/make/core/Makefile
```

```
$(INTERNAL_OTA_PACKAGE_TARGET): $(OTA_TOOL_EXTENSION)/releasetools.py
  @echo "Package OTA: $@"
  $(hide) PATH=$(foreach p,$(INTERNAL_USERIMAGES_BINARY_PATHS),$(p):)$$PATH MKBOOTIMG=$(MKBOOTIMG) \
     build/make/tools/releasetools/ota_from_target_files -v -n \
     --block \
     --extracted_input_target_files $(patsubst %.zip,%,$(BUILT_TARGET_FILES_PACKAGE)) \
     -p $(HOST_OUT) \
     -k $(KEY_CERT_PAIR) \
     -s $(OTA_TOOL_EXTENSION)/releasetools \
     $(if $(OEM_OTA_CONFIG), -o $(OEM_OTA_CONFIG)) \
     $(BUILT_TARGET_FILES_PACKAGE) $@
```

### main()

main 里面只是对参数的解析，主要用于区分是否为 A/B 升级，整包和差分等。

#### 整包

```
# Generate a full OTA.
  if OPTIONS.incremental_source is None:
    with zipfile.ZipFile(args[0], 'r') as input_zip:
      WriteFullOTAPackage(
          input_zip,
          output_file=args[1])
```

##### WriteFullOTAPackage

1. 获取 build 信息

```
target_info = BuildInfo(OPTIONS.info_dict, OPTIONS.oem_dicts)
```

2. 获取 metadata

```
metadata = GetPackageMetadata(target_info)
```

3. 是否有 RecoveryPatch

```
assert HasRecoveryPatch(input_zip)

# Recovery is generated as a patch using both the boot image
# (which contains the same linux kernel as recovery) and the file
# /system/etc/recovery-resource.dat (which contains all the images
# used in the recovery UI) as sources.  This lets us minimize the
# size of the patch, which must be included in every OTA package.
#
# For older builds where recovery-resource.dat is not present, we
# use only the boot image as the source.

def HasRecoveryPatch(target_files_zip):
  namelist = [name for name in target_files_zip.namelist()]
  return ("SYSTEM/recovery-from-boot.p" in namelist or
          "SYSTEM/etc/recovery.img" in namelist)
```

4. 降级检查，根据 build.utc 检查

```
# Assertions (e.g. downgrade check, device properties check).
  if not OPTIONS.omit_prereq:
    ts = GetBuildProp("ro.build.date.utc", OPTIONS.info_dict)
    ts_text = GetBuildProp("ro.build.date", OPTIONS.info_dict)
    script.AssertOlderBuild(ts, ts_text)
```

5. system.img 处理

```
  # script中写入fingerprint
  script.Print("Target: {}".format(target_info.fingerprint))

  device_specific.FullOTA_InstallBegin()
  # 计算进度
  system_progress = 0.75

  if OPTIONS.wipe_user_data:
    system_progress -= 0.1
  if HasVendorPartition(input_zip):
    system_progress -= 0.1

  script.ShowProgress(system_progress, 0)

  # 获取system.img
  system_tgt = common.GetSparseImage("system", OPTIONS.input_tmp, input_zip,allow_shared_blocks)
  system_tgt.ResetFileMap()
  # 整包System的patch，实际上是与空文件做的BlockDifference.[见5.1]
  system_diff = common.BlockDifference("system", system_tgt, src=None)
  # OTA包中写入script.[见5.2]
  system_diff.WriteScript(script, output_zip)
```

   5.1 systemDiff 的计算与写入以下两部分，实际是在 common.py 中执行的。


   ```
   system_diff = common.BlockDifference("system", system_tgt, src=None)

   system_diff.WriteScript(script, output_zip)
   ```

   common的初始化，在整包的时候，传入的src=None

   ```
    def __init__(self, partition, tgt, src=None, check_first_block=False,
               version=None, disable_imgdiff=False):
    self.tgt = tgt
    self.src = src
    self.partition = partition
    self.check_first_block = check_first_block
    self.disable_imgdiff = disable_imgdiff

    # 在工作线程中，使用blockimgdiff.py中计算diff，此时的src=None [后续分析]
    b = blockimgdiff.BlockImageDiff(tgt, src, threads=OPTIONS.worker_threads,
                                    version=self.version,
                                    disable_imgdiff=self.disable_imgdiff)
    self.path = os.path.join(MakeTempDir(), partition)
    b.Compute(self.path)
    self._required_cache = b.max_stashed_size
    self.touched_src_ranges = b.touched_src_ranges
    self.touched_src_sha1 = b.touched_src_sha1

    if src is None:
      _, self.device = GetTypeAndDevice("/" + partition, OPTIONS.info_dict)
    else:
      _, self.device = GetTypeAndDevice("/" + partition,
                                        OPTIONS.source_info_dict)
   ```
   5.2 common.WriteScript
   ```
    def WriteScript(self, script, output_zip, progress=None):
    if not self.src:
      # write the output unconditionally.即为整包写入脚本
      script.Print("Patching %s image unconditionally..." % (self.partition,))
    else: # 即为差分写入脚本
      script.Print("Patching %s image after verification." % (self.partition,))

    if progress:
      script.ShowProgress(progress, 0)
    self._WriteUpdate(script, output_zip) [见5.3]
    if OPTIONS.verify:
      self._WritePostInstallVerifyScript(script)
   ```
   5.3 common._WritePostInstallVerifyScript
   ```
    # 先将system.transfer.list写入
    ZipWrite(output_zip,
             '{}.transfer.list'.format(self.path),
             '{}.transfer.list'.format(self.partition))
    # 整包逻辑
    if not self.src:
      # 使用brotli,quality=6的方式压缩new.dat文件
      # quality=9的方式压缩，耗时几乎是quality=6的3倍，但包的大小不会减小太多
      brotli_cmd = ['brotli', '--quality=6',
                    '--output={}.new.dat.br'.format(self.path),
                    '{}.new.dat'.format(self.path)]
      print("Compressing {}.new.dat with brotli".format(self.partition))
      p = Run(brotli_cmd, stdout=subprocess.PIPE)
      p.communicate()
      assert p.returncode == 0,\
          'compression of {}.new.dat failed'.format(self.partition)

      new_data_name = '{}.new.dat.br'.format(self.partition)
      # 将压缩后的new.dat.br文件写入包
      ZipWrite(output_zip,
               '{}.new.dat.br'.format(self.path),
               new_data_name,
               compress_type=zipfile.ZIP_STORED)
    # 差分逻辑，后面分析
    else:
   ```