---
layout:     post
title:      "blockimgdiff.py 分析"
date:       2021-12-06 19:30:00
author:     "hxg_"
comments:	true
header-img: "img/post-bg-01.jpg"
---

1 在前面，分析了整包和差分的打包，遗留了diff的分析

```python
system_diff = common.BlockDifference("system", system_tgt, src=None)
```

2 其中，common中构造函数的调用如下：
```python
def __init__(self, partition, tgt, src=None, check_first_block=False,
               version=None, disable_imgdiff=False):
    self.tgt = tgt
    self.src = src
    self.partition = partition
    # 当system分区为ext4格式的时候，需要检查first block
    # system_src_partition = source_info["fstab"]["/system"]
    # check_first_block = system_src_partition.fs_type == "ext4"
    self.check_first_block = check_first_block
    self.disable_imgdiff = disable_imgdiff

    # 调用blockimgdiff.py.BlockImageDiff
    b = blockimgdiff.BlockImageDiff(tgt, src, threads=OPTIONS.worker_threads,
                                    version=self.version,
                                    disable_imgdiff=self.disable_imgdiff)
    self.path = os.path.join(MakeTempDir(), partition)
    
    # 调用blockimgdiff.Compute()
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

3 blockimgdiff.Compute如下
```python
def Compute(self, prefix):
    # 见【3.1】
    self.AbbreviateSourceNames()
    # 见【3.2】
    self.FindTransfers()

    self.FindVertexSequence()
    
    self.ReverseBackwardEdges()
    self.ImproveVertexSequence()

    # Ensure the runtime stash size is under the limit.
    if common.OPTIONS.cache_size is not None:
      self.ReviseStashSize()

    # Double-check our work.
    self.AssertSequenceGood()
    self.AssertSha1Good()

    # 见【4】
    self.ComputePatches(prefix)
    self.WriteTransfers(prefix)

    # Report the imgdiff stats.
    if common.OPTIONS.verbose and not self.disable_imgdiff:
      self.imgdiff_stats.Report()
```

    3.1 AbbreviateSourceNames，只是做了文件名和绝对路径的映射关系，放在了src_basenames里，其中basename为key，绝对路径为value。
    在后续会以target的basename和src的basename比较，如果存在相同的即做diff
    
```python
    def AbbreviateSourceNames(self):
    for k in self.src.file_map.keys():
      b = os.path.basename(k)
      self.src_basenames[b] = k
      b = re.sub("[0-9]+", "#", b)
      self.src_numpatterns[b] = k
```
    
    3.2 FindTransfers 方法
```python
    print("Finding transfers...")

    large_apks = []
    split_large_apks = []
    cache_size = common.OPTIONS.cache_size
    split_threshold = 0.125
    # 根据cache分区的大小来计算max_blocks
    # 如果cache分区放入了其他持久化的文件，需要将其排除，或直接指定大小
    max_blocks_per_transfer = int(cache_size * split_threshold /
                                  self.tgt.blocksize)
    # 
    empty = RangeSet()
    for tgt_fn, tgt_ranges in sorted(self.tgt.file_map.items()):
      if tgt_fn == "__ZERO":
        # the special "__ZERO" domain is all the blocks not contained
        # in any file and that are filled with zeros.  We have a
        # special transfer style for zero blocks.
        src_ranges = self.src.file_map.get("__ZERO", empty)
        AddTransfer(tgt_fn, "__ZERO", tgt_ranges, src_ranges,
                    "zero", self.transfers)
        continue

      elif tgt_fn == "__COPY":
        # "__COPY" domain includes all the blocks not contained in any
        # file and that need to be copied unconditionally to the target.
        AddTransfer(tgt_fn, None, tgt_ranges, empty, "new", self.transfers)
        continue
      
      # 当tgt文件名存在于src中
      # 大部分均会进入该逻辑
      elif tgt_fn in self.src.file_map:
        # Look for an exact pathname match in the source.
        AddTransfer(tgt_fn, tgt_fn, tgt_ranges, self.src.file_map[tgt_fn],
                    "diff", self.transfers, True)
        continue

      b = os.path.basename(tgt_fn)
      if b in self.src_basenames:
        # Look for an exact basename match in the source.
        src_fn = self.src_basenames[b]
        AddTransfer(tgt_fn, src_fn, tgt_ranges, self.src.file_map[src_fn],
                    "diff", self.transfers, True)
        continue

      b = re.sub("[0-9]+", "#", b)
      if b in self.src_numpatterns:
        # 需要把数字替换为#去比较，为了的一下so等文件名包含了版本号
        # 如 /system/etc/vison/01_ts_210112_pdl.mlm
        src_fn = self.src_numpatterns[b]
        AddTransfer(tgt_fn, src_fn, tgt_ranges, self.src.file_map[src_fn],
                    "diff", self.transfers, True)
        continue

      # 新增文件，如APK中新增了so等
      AddTransfer(tgt_fn, None, tgt_ranges, empty, "new", self.transfers)
```
        3.2.1 AddTransfer
```python
def AddTransfer(tgt_name, src_name, tgt_ranges, src_ranges, style, by_id,
                    split=False):

      # 只处理style为diff的，包含了bsdiff/imgdiff/move 指令
      if style != "diff" or not split:
        Transfer(tgt_name, src_name, tgt_ranges, src_ranges,
                 self.tgt.RangeSha1(tgt_ranges), self.src.RangeSha1(src_ranges),
                 style, by_id)
        return

      # 针对odex文件做的优化，省略
      if (tgt_name.split(".")[-1].lower() == 'odex' and
          tgt_ranges.size() == src_ranges.size()):

        
      # 见【3.2.2】
      AddSplitTransfers(
          tgt_name, src_name, tgt_ranges, src_ranges, style, by_id)
```
        3.2.2 AddSplitTransfers
```python
def AddSplitTransfers(tgt_name, src_name, tgt_ranges, src_ranges, style,
                          by_id):
      # 小文件不做处,直接Transfer.
      if (tgt_ranges.size() <= max_blocks_per_transfer and
          src_ranges.size() <= max_blocks_per_transfer):
        Transfer(tgt_name, src_name, tgt_ranges, src_ranges,
                 self.tgt.RangeSha1(tgt_ranges), self.src.RangeSha1(src_ranges),
                 style, by_id)
        return

      # APK文件使用imageDiff
      if (self.FileTypeSupportedByImgdiff(tgt_name) and
          self.tgt.RangeSha1(tgt_ranges) != self.src.RangeSha1(src_ranges)):
        if self.CanUseImgdiff(tgt_name, tgt_ranges, src_ranges, True):
          large_apks.append((tgt_name, src_name, tgt_ranges, src_ranges))
          return
      
      # 见【3.2.3】
      AddSplitTransfersWithFixedSizeChunks(tgt_name, src_name, tgt_ranges,
                                           src_ranges, style, by_id)
```
        3.2.3 AddSplitTransfersWithFixedSizeChunks
```python
def AddSplitTransfersWithFixedSizeChunks(tgt_name, src_name, tgt_ranges,
                                             src_ranges, style, by_id):
      # 如果文件占用了太多的块，将会split it into smaller pieces by getting multiple Transfer()s
      
      pieces = 0
      while (tgt_ranges.size() > max_blocks_per_transfer and
             src_ranges.size() > max_blocks_per_transfer):
        # 如 /system/app/a.apk-0
        tgt_split_name = "%s-%d" % (tgt_name, pieces)
        src_split_name = "%s-%d" % (src_name, pieces)
        tgt_first = tgt_ranges.first(max_blocks_per_transfer)
        src_first = src_ranges.first(max_blocks_per_transfer)

        Transfer(tgt_split_name, src_split_name, tgt_first, src_first,
                 self.tgt.RangeSha1(tgt_first), self.src.RangeSha1(src_first),
                 style, by_id)

        tgt_ranges = tgt_ranges.subtract(tgt_first)
        src_ranges = src_ranges.subtract(src_first)
        pieces += 1

      # 处理剩余的.如 /system/app/a.apk-1
      if tgt_ranges.size() or src_ranges.size():
        # Must be both non-empty.
        assert tgt_ranges.size() and src_ranges.size()
        tgt_split_name = "%s-%d" % (tgt_name, pieces)
        src_split_name = "%s-%d" % (src_name, pieces)
        # 见【4】
        Transfer(tgt_split_name, src_split_name, tgt_ranges, src_ranges,
                 self.tgt.RangeSha1(tgt_ranges), self.src.RangeSha1(src_ranges),
                 style, by_id)
```
4 ComputePatches,用于计算patch并最终生成.patch.dat文件

```python
def ComputePatches(self, prefix):
    print("Reticulating splines...")
    diff_queue = []
    patch_num = 0
    # 生成.new.dat
    with open(prefix + ".new.dat", "wb") as new_f:
      for index, xf in enumerate(self.transfers):
        if xf.style == "zero":
          tgt_size = xf.tgt_ranges.size() * self.tgt.blocksize
          print("%10d %10d (%6.2f%%) %7s %s %s" % (
              tgt_size, tgt_size, 100.0, xf.style, xf.tgt_name,
              str(xf.tgt_ranges)))

        elif xf.style == "new":
          self.tgt.WriteRangeDataToFd(xf.tgt_ranges, new_f)
          tgt_size = xf.tgt_ranges.size() * self.tgt.blocksize
          print("%10d %10d (%6.2f%%) %7s %s %s" % (
              tgt_size, tgt_size, 100.0, xf.style,
              xf.tgt_name, str(xf.tgt_ranges)))

        elif xf.style == "diff":
          # 比较block的sha1是否相同，如果相同则可以直接使用move指令
          if xf.src_sha1 == xf.tgt_sha1:
            xf.style = "move"
            xf.patch = None
            tgt_size = xf.tgt_ranges.size() * self.tgt.blocksize
            if xf.src_ranges != xf.tgt_ranges:
              print("%10d %10d (%6.2f%%) %7s %s %s (from %s)" % (
                  tgt_size, tgt_size, 100.0, xf.style,
                  xf.tgt_name if xf.tgt_name == xf.src_name else (
                      xf.tgt_name + " (from " + xf.src_name + ")"),
                  str(xf.tgt_ranges), str(xf.src_ranges)))
          else:
            if xf.patch:
              # imgdiff/bsdiff
              assert not self.disable_imgdiff
              imgdiff = True
              if (xf.src_ranges.extra.get('trimmed') or
                  xf.tgt_ranges.extra.get('trimmed')):
                imgdiff = False
                xf.patch = None
            else:
              imgdiff = self.CanUseImgdiff(
                  xf.tgt_name, xf.tgt_ranges, xf.src_ranges)
            xf.style = "imgdiff" if imgdiff else "bsdiff"
            diff_queue.append((index, imgdiff, patch_num))
            patch_num += 1

        else:
          assert False, "unknown style " + xf.style

    if diff_queue:
      # 省略，实际为在工作线程调用compute_patch 见【5】
      # 同时，将计算后的patch给数组赋值
      # patches[patch_index] = (xf_index, patch)

    # 生成.patch.dat
    offset = 0
    with open(prefix + ".patch.dat", "wb") as patch_fd:
      for index, patch in patches:
        xf = self.transfers[index]
        xf.patch_len = len(patch)
        xf.patch_start = offset
        offset += xf.patch_len
        patch_fd.write(patch)
```

5 compute_patch,只是调用了bsdiff/imgdiff指令
```python
def compute_patch(srcfile, tgtfile, imgdiff=False):
  patchfile = common.MakeTempFile(prefix='patch-')

  cmd = ['imgdiff', '-z'] if imgdiff else ['bsdiff']
  cmd.extend([srcfile, tgtfile, patchfile])

  # Don't dump the bsdiff/imgdiff commands, which are not useful for the case
  # here, since they contain temp filenames only.
  p = common.Run(cmd, verbose=False, stdout=subprocess.PIPE,
                 stderr=subprocess.STDOUT)
  output, _ = p.communicate()

  if p.returncode != 0:
    raise ValueError(output)

  with open(patchfile, 'rb') as f:
    return f.read()
```