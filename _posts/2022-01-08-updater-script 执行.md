---
layout:     post
title:      "updater-script 执行"
date:       2022-01-08 15:40:00
author:     "hxg_"
comments:	true
header-img: "img/post-bg-01.jpg"
---

#### 以差分包中的script为例

##### 1. 校验设备的基本信息，是否为差分包的基础版本
```
getprop("ro.product.device") == "pufferx8" || abort("E3004: This package is for \"pufferx8\" devices; this is a \"" + getprop("ro.product.device") + "\".");
ui_print("Source: alps/pufferx8/pufferx8:8.1.0/O11019/1615868373:user/dev-keys");
ui_print("Target: alps/pufferx8/pufferx8:8.1.0/O11019/1617219089:user/dev-keys");
ui_print("Verifying current system...");
getprop("ro.build.fingerprint") == "alps/pufferx8/pufferx8:8.1.0/O11019/1615868373:user/dev-keys" ||
    getprop("ro.build.fingerprint") == "alps/pufferx8/pufferx8:8.1.0/O11019/1617219089:user/dev-keys" ||
    abort("E3001: Package expects build fingerprint of alps/pufferx8/pufferx8:8.1.0/O11019/1615868373:user/dev-keys or alps/pufferx8/pufferx8:8.1.0/O11019/1617219089:user/dev-keys; this device has " + getprop("ro.build.fingerprint") + ".");
```
##### 2. check boot分区，【见8】
```
apply_patch_check("EMMC:/dev/block/platform/bootdevice/by-name/boot:8528800:34bdb63751f0b2060a0230c0c4e788cf6fbd50ab:8528800:9319d1ab25bfa034dd6bf8912844d8d6b78c60e0") || abort("E3005: \"EMMC:/dev/block/platform/bootdevice/by-name/boot:8528800:34bdb63751f0b2060a0230c0c4e788cf6fbd50ab:8528800:9319d1ab25bfa034dd6bf8912844d8d6b78c60e0\" has unexpected contents.");
```
##### 3. 检查磁盘空间，【见9】
```
apply_patch_space(101793792) || abort("E3006: Not enough free space on /cache to apply patches.");
```
##### 4. 检查system分区,vendor分区逻辑同System，【见10】
```
if (range_sha1("/dev/block/platform/bootdevice/by-name/system", "80,1,...") == "555ee764756a67250de9c03805e8e737658b2830" || block_image_verify("/dev/block/platform/bootdevice/by-name/system", package_extract_file("system.transfer.list"), "system.new.dat", "system.patch.dat")) then
    ui_print("Verified system image...");
else
  check_first_block("/dev/block/platform/bootdevice/by-name/system");
ifelse (block_image_recover("/dev/block/platform/bootdevice/by-name/system", "80,1,...") && block_image_verify("/dev/block/platform/bootdevice/by-name/system", package_extract_file("system.transfer.list"), "system.new.dat", "system.patch.dat"), ui_print("system recovered successfully."), abort("E1004: system partition fails to recover"));
endif;
```
##### 5. 调用block_image_update去更新System、vendor分区，【见11】
```
ui_print("Patching system image after verification.");
show_progress(0.800000, 0);
block_image_update("/dev/block/platform/bootdevice/by-name/system", package_extract_file("system.transfer.list"), "system.new.dat", "system.patch.dat") ||
  abort("E1001: Failed to update system image.");
ui_print("Patching vendor image after verification.");
```
##### 6. 调用apply_patch去更新boot
```
apply_patch("EMMC:/dev/block/platform/bootdevice/by-name/boot:8528800:34b...",
            "-", 93..., 8528800,
            34b...,
            package_extract_file("patch/boot.img.p")) ||
    abort("E3008: Failed to apply patch to EMMC:/dev/block/platform/bootdevice/by-name/boot:8528800:3...);
```

##### 7.更新其他小分区
##### 7.1 MTK对小分区做了双备份的逻辑，同时在更新过程中，保存了更新状态的`cache/recovery/last_mtupdate_stage` 文件。根据stage的值，更新不同的分区。
```
less_than_int(get_mtupdate_stage("/cache/recovery/last_mtupdate_stage"), "1") ,
  (
    ui_print("start to update general image");
    package_extract_file("logo.bin", "/dev/block/platform/bootdevice/by-name/logo");
    package_extract_file("odmdtbo.img", "/dev/block/platform/bootdevice/by-name/odmdtbo");
    set_mtupdate_stage("/cache/recovery/last_mtupdate_stage", "1");
  ),
  ui_print("general images are already updated");
  );
```
##### 7.2 更新前，将影响启动的分区，切换为备用分区。
```
switch_active("tee1", "tee2");
switch_active("lk", "lk2");
```
##### 7.3 更新后，将已更新的分区切换为active
```
switch_active("tee2", "tee1");
switch_active("lk2", "lk");
```
##### 7.4此外，对小分区的更新，都采用整包覆盖的方式。主要是因为img很小，不需要做差分。
```
package_extract_file("lk.img", "/dev/block/platform/bootdevice/by-name/lk");
```

```
总结下整体流程：
1、校验设备的基本信息，是否为差分包的基础版本；
2、check boot分区；
3、检查system、vendor分区；
4、调用block_image_update去更新System、vendor分区；
5、调用apply_patch去更新boot；
6、更新其他小分区（MTK对小分区做了双备份的逻辑，同时在更新过程中，保存了更新状态的`cache/recovery/last_mtupdate_stage` 文件。根据stage的值，更新不同的分区）；
```

#### 8. apply_patch_check方法，最终映射后调用`ApplyPatchCheckFn`
```
/bootable/recovery/updater/install.cpp

RegisterFunction("apply_patch_check", ApplyPatchCheckFn);
```

```
/bootable/recovery/updater/applypatch/applypatch.cpp

Value* ApplyPatchCheckFn(const char* name, State* state, const std::vector<std::unique_ptr<Expr>>& argv) {
  ...
  int result = applypatch_check(filename.c_str(), sha1s);

  return StringValue(result == 0 ? "t" : "");
}
```

```
int applypatch_check(const char* filename, const std::vector<std::string>& patch_sha1_str) {
  FileContents file;

  // 先校验原始文件
  if (LoadFileContents(filename, &file) != 0 ||
      (!patch_sha1_str.empty() && FindMatchingPatch(file.sha1, patch_sha1_str) < 0)) {
    printf("file \"%s\" doesn't have any of expected sha1 sums; checking cache\n", filename);

    // 校验原始文件失败，去校验备份的文件 /cache/saved.file
    if (LoadFileContents(CACHE_TEMP_SOURCE, &file) != 0) {
      printf("failed to load cache file\n");
      return 1;
    }

    // 查找match的patch文件，filename encodes the sha1s
    if (FindMatchingPatch(file.sha1, patch_sha1_str) < 0) {
      printf("cache bits don't match any sha1 for \"%s\"\n", filename);
      return 1;
    }
  }
  return 0;
}
```

```
ApplyPatchCheckFn用于检查原始文件或缓存文件的内容sha1值费否与command中的相符。
1、先检查原始文件；
2、检查备份文件，即/cache/saved.file；
3、以文件名查找match的patch文件；
以上有一项检查通过即通过。
```

##### 9. apply_patch_space
```
RegisterFunction("apply_patch_space", ApplyPatchSpaceFn);
```

```
Value* ApplyPatchSpaceFn(const char* name, State* state, const std::vector<std::unique_ptr<Expr>>& argv) {
  ...

  size_t bytes;
  if (!android::base::ParseUint(bytes_str.c_str(), &bytes)) {
    return ErrorAbort(state, kArgsParsingFailure, "%s(): can't parse \"%s\" as byte count\n\n",
                      name, bytes_str.c_str());
  }

  // 增加的patch：如果掉电，cache分区可能存在stash文件，需要把此部分算在可用空间内
  size_t existing = 0;
  std::string dirname = "/cache/recovery/1fb420b778bead14ec9b2cf264fcaa8e147e3e39/";
  EnumerateStash(dirname, [&existing](const std::string& fn) {
    if (fn.empty()) return;
    struct stat sb;
    if (stat(fn.c_str(), &sb) == -1) {
        printf("error stat %s \n", fn.c_str());        
      return;
    }
    existing += static_cast<size_t>(sb.st_size);
  });
  
  printf("ApplyPatchSpaceFn %zu; existing %zu bytes \n", bytes, existing);
  if (bytes > existing) {
    size_t needed = bytes - existing;
    return StringValue(CacheSizeCheck(needed) ? "" : "t");
  } else {
    return StringValue("t");
  }
}
```

##### 10. System、Vendor分区的检查
```
1、先检查指定范围的sha1；
2、上述失败，则调用block_image_verify校验；【见10.1】
3、上述均校验失败，则先调用block_image_recover尝试恢复，恢复成功后再进行则调用block_image_verify校验；
4、上述均失败则终止；
```

##### 10.1 block_image_verify

```
映射为了BlockImageVerifyFn方法
RegisterFunction("block_image_verify", BlockImageVerifyFn);
```

```
/bootable/recovery/updater/blockimg.cpp

Value* BlockImageVerifyFn(const char* name, State* state,
                          const std::vector<std::unique_ptr<Expr>>& argv) {
    // transfer.list 中command对应执行
    const Command commands[] = {
        { "bsdiff",     PerformCommandDiff  },
        ...
    };

    // Perform a dry run without writing to test if an update can proceed
    return PerformBlockImageUpdate(name, state, argv, commands,
                sizeof(commands) / sizeof(commands[0]), true);
}
```

```
最终会调用PerformBlockImageUpdate方法，在update的时候也会调用此方法，二者的区别只是最后一个参数的区别
```

```
Value* BlockImageUpdateFn(const char* name, State* state,
                          const std::vector<std::unique_ptr<Expr>>& argv) {
    const Command commands[] = {
        { "bsdiff",     PerformCommandDiff  },
        ...
    };

    return PerformBlockImageUpdate(name, state, argv, commands,
                sizeof(commands) / sizeof(commands[0]), false);
}
```

##### 11. 校验通过后，调用`block_image_update`，更新system、vendor分区
```
最终会映射为BlockImageUpdateFn方法，即【10.1】中的方法，顺序调用每个command对应的方法
RegisterFunction("block_image_update", BlockImageUpdateFn);
```

##### 11.1 
##### 12. PerformCommandStash方法，对应script中的stash指令
```
static int PerformCommandStash(CommandParameters& params) {
  ...

  const std::string& id = params.tokens[params.cpos++];
  size_t blocks = 0;
  // 尝试loadStash
  // 存储文件已存在，并且具有预期的内容。不要再次从源中读取数据，因为在上一次尝试中该源可能已被覆盖。
  if (LoadStash(params, id, true, &blocks, params.buffer, false) == 0) {
    // 增加的patch，如果可以load成功，这个stashID的文件也不应该被删除
    stash_map2[id] = true;
    LOG(ERROR) << "PerformCommandStash id :"<< id ;
    return 0;
  }

  RangeSet src = RangeSet::Parse(params.tokens[params.cpos++]);

  allocate(src.blocks() * BLOCKSIZE, params.buffer);
  if (ReadBlocks(src, params.buffer, params.fd) == -1) {
    return -1;
  }
  blocks = src.blocks();
  stash_map[id] = src;

  // 虽然校验失败，但不中断。
  // 如果后面真的要使用这些数据，肯定是一个不能恢复的错误。但可能在前面使用这个数据的命令已经完成了。
  // 所以，在verify时可能失败。
  if (VerifyBlocks(id, params.buffer, blocks, true) != 0) {
    LOG(ERROR) << "failed to load source blocks for stash " << id;
    return 0;
  }

  // 在verify时，不需要写文件
  if (!params.canwrite) {
    return 0;
  }

  LOG(INFO) << "stashing " << blocks << " blocks to " << id;
  params.stashed += blocks;
  // 写入stash文件
  return WriteStash(params.stashbase, id, blocks, params.buffer, false, nullptr);
}
```
```
总结：
1、尝试loadStash，如果存储文件已存在，并且具有预期的内容。不要再次从源中读取数据，因为在上一次尝试中该源可能已被覆盖；
2、上述失败，尝试校验blocks。校验失败则返回继续。此处校验，应该只是为了保证下面写入stash文件使用的blocks是正确的；
3、将block写入stash文件；
```

##### 13. PerformCommandFree方法，对应script中的free指令
```
static int PerformCommandFree(CommandParameters& params) {
  ...

  // 清除不用的stash_id引用
  const std::string& id = params.tokens[params.cpos++];
  stash_map.erase(id);
  stash_map2.erase(id);

  // 删除stash_id对应的文件
  if (params.createdstash || params.canwrite) {
    return FreeStash(params.stashbase, id);
  }

  return 0;
}
```

##### 14. PerformCommandMove方法，对应script中move指令
```
static int PerformCommandMove(CommandParameters& params) {
  size_t blocks = 0;
  bool overlap = false;
  RangeSet tgt;
  // 【见15】
  int status = LoadSrcTgtVersion3(params, tgt, &blocks, true, &overlap);

  if (status == -1) {
    LOG(ERROR) << "failed to read blocks for move";
    return -1;
  }

  ...

  // move后删除stash文件
  if (!params.freestash.empty()) {
    FreeStash(params.stashbase, params.freestash);
    params.freestash.clear();
  }
  
  params.written += tgt.blocks();

  return 0;
}
```

##### 15. LoadSrcTgtVersion3，这个方法diff move都会运行
```
这个函数里面的onehas参数为true时，source和target的hash值是一样的，为false时，不一样。如果是move的情况,onehash值为true,diff情况为false
函数返回值为-1，表示无法载入必须的block或者内容和hash值不符，命令无法进行。
函数返回值为1，表示经验证，该命令要升级的block已经完成了升级。也就是这次升级是retry，而当前block在上次升级的时候已经完成了升级。
函数返回值为0，表示block的hash值验证通过，可以升级。
```

```
static int LoadSrcTgtVersion3(CommandParameters& params, RangeSet& tgt, size_t* src_blocks,
                              bool onehash, bool* overlap) {
  ...

  std::string srchash = params.tokens[params.cpos++];
  std::string tgthash;

  // //如果是move的情况,onehash值为true,diff情况为false
  if (onehash) { // tgt和src的hash是一样的
    tgthash = srchash;
  } else { // 找到tgt的hash
    if (params.cpos >= params.tokens.size()) {
      LOG(ERROR) << "missing target hash";
      return -1;
    }
    tgthash = params.tokens[params.cpos++];
  }

  ...

  // 解析出tgt_range
  tgt = RangeSet::Parse(params.tokens[params.cpos++]);

  // BLOCKSIZE是4096。如果读不到target的指定block的内容，返回-1，命令无法进行。
  std::vector<uint8_t> tgtbuffer(tgt.blocks() * BLOCKSIZE);
  if (ReadBlocks(tgt, tgtbuffer, params.fd) == -1) {
    LOG(ERROR) << "ReadBlocks error";
    return -1;
  }
  
  // 如果target的block的hash值和指定的hash值相同，说明该block在上次升级的时候已经完成了升级，返回1。
  if (VerifyBlocks(tgthash, tgtbuffer, tgt.blocks(), false) == 0) {
    LOG(INFO) << "VerifyBlocks return " ;
    return 1;
  }

  // 加载src block，即把source和stash的内容组合后，存储到params.buffer中
  if (LoadSourceBlocks(params, tgt, src_blocks, overlap) == -1) {
    return -1;
  }

  // 校验源数据的hash是否匹配,如果校验通过,verify直接返回0
  if (VerifyBlocks(srchash, params.buffer, *src_blocks, true) == 0) {
    // 如果source和target有重叠，就把source的内容先写到/cache分区
    if (*overlap && params.canwrite) {
      bool stash_exists = false;
      // 存储重叠的块
      if (WriteStash(params.stashbase, srchash, *src_blocks, params.buffer, true,
                     &stash_exists) != 0) {
        LOG(ERROR) << "failed to stash overlapping source blocks";
        return -1;
      }

      params.stashed += *src_blocks;
      // Can be deleted when the write has completed.
      if (!stash_exists) {
        params.freestash = srchash;
      } else if (stash_map.find(srchash) == stash_map.end()) {
        // 针对PerformCommandStash方法中加的patch，此时需要将之前找到是stash文件标记为可删除
        if (stash_map2.find(srchash) == stash_map2.end()) {
            params.freestash = srchash;  
            LOG(INFO) << "LoadSrcTgtVersion3 set freestash1 +++: "<< srchash ;
        }
      }
    }

    // Source blocks have expected content, command can proceed.
    return 0;
  }
  
  // 如果source和target有重叠，而且，前面source校验失败，那么就尝试从cache分区里面恢复。
  // 因为有可能上次更新的时候，将source复制了一份到cache。
  if (*overlap && LoadStash(params, srchash, true, nullptr, params.buffer, true) == 0) {
    if (params.canwrite) {
        if (stash_map.find(srchash) == stash_map.end()) {
            LOG(INFO) << "LoadSrcTgtVersion3 set freestash2: "<< srchash ;
            // 同样针对PerformCommandStash方法中加的patch，此时需要将之前找到是stash文件标记为可删除
            if (stash_map2.find(srchash) == stash_map2.end()) {
                params.freestash = srchash;
            }

        }
    }
    LOG(INFO) << "LoadStash return " ;
    return 0;
  }

  // Valid source data not available, update cannot be resumed.
  LOG(ERROR) << "partition has unexpected contents";
  PrintHashForCorruptedSourceBlocks(params, params.buffer);

  params.isunresumable = true;

  return -1;
}
```

```
总结：
1.获取参数信息 diff为双hash move为单hash，通过onahash参数区分；
2.确认目标区间能不能读取到数据,即能不能写入；
3.优先校验目标区间的数据已经已经写入过，即之前是否成功升级过；
4.如果没有写入过,加载源版本src_block,并通过区间对比赋值overlap,是否存在交集；
5.如果校验srchash通过,说明源数据没有问题,校验阶段在此结束；
6.如果校验srchash通过,写入阶段首先存储重叠的块；
```
