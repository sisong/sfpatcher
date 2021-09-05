联系作者： housisong@hotmail.com   
   
# 创建补丁的工具sf_diff
使用命令行工具sf_diff命令，来创建新旧版本apk文件之间的补丁，直接运行sf_diff命令会得到如下所示的帮助信息输出：
```
diff   usage: sf_diff [options] oldArchiveFile newArchiveFile outDiffFile
compress usage: [-c - ...] -pre "" newArchiveFile outDiffFile
test   usage: sf_diff -t oldArchiveFile newArchiveFile testDiffFile
resave usage: sf_diff [-c-...] diffFile outDiffFile
options:
    input oldArchiveFile & newArchiveFile can zip,apk,jar,tar,gz... file type;
    oldArchiveFile can empty, and input parameter ""
  -o-{0..3}
      select optimize direction for patch speed or outDiffFileSize, DEFAULT -o-1;
      -o-0      run fastest, not need decompress temp buffer!
                but outDiffFile is biger;
      -o-1      run fast, outDiffFile is small enough!
                need decompress temp buffer (see -lo);
      -o-2      outDiffFile is smaller when input file compatible
                zlib & compressLevel<=6 (if uncompatible, as -o-1);
                need decompress temp buffer (see -lo),
                run slowly when re-compress newArchiveFile (see -ln);
      -o-3      outDiffFile is minimal when input file compatible
                zlib (if uncompatible, as -o-1);
                need decompress temp buffer (see -lo),
                run very slowly when re-compress newArchiveFile (see -ln);
  -c-compressType[-compressLevel]
      set outDiffFile Compress type & level, DEFAULT uncompress;
      for resave diffFile,recompress diffFile to outDiffFile by new set;
      support compress type & level:
       (re. https://github.com/sisong/lzbench/blob/master/lzbench171_sorted.md )
        -c-zlib[-{1..9}]                    DEFAULT level 9
        -c-lzma[-{0..9}[-dictSize]]         DEFAULT level 7
            dictSize can like 4096 or 4k or 4m or 128m etc..., DEFAULT 16m
            support run by 2-thread parallel.
        -c-lzma2[-{0..9}[-dictSize]]        DEFAULT level 7
            dictSize can like 4096 or 4k or 4m or 128m etc..., DEFAULT 16m
            WARNING: code not compatible with it compressed by -c-lzma!
            support run by multi-thread parallel, fast!
        -c-zstd[-{0..22}[-dictBits]]        DEFAULT level 20
            dictBits can 10--31, DEFAULT 24.
            support run by multi-thread parallel, fast!
  -m-matchScore
      matchScore>=0, DEFAULT -m-2, recommended: 0--6 etc...
  -step-patchStepMemroySize
      set patch step memory size, DEFAULT -sm-2m, recommended: 64k,512k,8m etc...
  -pre
      set is always decode newArchiveFile (precompress), DEFAULT false;
      if set this option, outDiffFile maybe smaller, but maybe slower when patch;
      if oldArchiveFile empty, recommended add -pre & -o-2 -c-lzma2 ...
  -lp-limitPatchMemSize
      a limit max size of decompress memory cache when patch;
      limitPatchMemSize>=128k, DEFAULT equal limitOldDecodeSize;
      recommended: -lp-8m, or 1m,32m etc... if set -o-1, recommended: -lp-512k;
      these memory cache will be saved part of decompressed when patch.
      Auto limit OPENED, limit size <= sum size of 2 largest decompressed files.
  -lo-limitOldDecodeSize
      a limit max size of decompress temp buffer from oldArchiveFile;
      DEFAULT no limit, recommended: -lo-128m, or 64m,256m etc...
      these data will be decompress to memory or temp file when patch.
  -ln-limitNewDecodeSize
      a limit max size of decompress data from newArchiveFile;
      DEFAULT no limit, recommended: -lo-256m, or 64m,128m etc...
      these data will be re-compress when patch.
  -p-parallelThreadNumber
      DEFAULT -p-4!  if parallelThreadNumber>1 then
      open multi-thread Parallel mode when compress outDiffFile;
  -d  Diff only, do't run patch check, DEFAULT run patch check.
  -t  Test only, run patch check;
      check sf_patch(oldArchiveFile,testDiffFile)==newArchiveFile?
  -v  output Version info.
```

 * **创建补丁**：比如你有一个旧版本的old.apk文件，现在有了一个新版本new.apk，创建升级补丁文件diff.pat的命令为：
 `$ sf_diff "old.apk" "new.apk" "diff.pat"`
创建成功，返回0，其他任何值都是发现了错误。 如果用户设备上安装了old.apk，下载diff.pat就能用本方案提供的patch算法升级到new.apk。   
 * 补丁**另存**：将已有的补丁文件使用新的压缩参数来重新压缩；   
比如使用lzma2算法开启8线程来重新压缩补丁：
`$ sf_diff -c-lzma2-9-64m -p-8 "diff0.pat" "diff1.pat"`   
   
 * **附加参数**都以“-”符号开头紧跟字母来进行不同的设置，可以放置在命令行的任意位置，如果该项设置还有参数的话后面继续用“-”间隔。[…] 框起来的部分表示可选项，如果不填，将使用预设置的默认值。{1..9}表示可以选择1到9中的一个值。   
   
 * **-o 选项**：diff时的优化级别和策略，分别对应不同的补丁大小和patch速度；一般来说数值越大补丁包越小，而patch可能越慢。
   * **-o-0**	patch最快，但补丁包较大，和hdiffz生成的补丁包几乎一样大；patch时资源占用少，不需要临时解压空间。
   * **-o-1**	patch很快，补丁包足够小，patch时需要使用临时解压空间(临时空间的总大小上限可以用-lo控制，而用-lp控制其最大内存占用)。 该级别为默认值。
   * **-o-2**	当输入的压缩数据兼容zlib库并且压缩级别<=6时，可以得到较小的补丁，不兼容的压缩数据就会退回到-o-1模式；patch时还原压缩较慢(需要还原的数据量的上限可以用-ln控制)，并且patch时需要使用临时解压空间(见-lo和-lp)。
   * **-o-3**	当输入的压缩数据兼容zlib库时，一般可以得到最小的补丁文件；不兼容的压缩数据就会退回到-o-1模式；patch时还原压缩很慢(见-ln)，并且patch时需要使用临时解压空间(见-lo和-lp)。

* **-c 选项**：设置补丁数据使用的压缩算法；diff默认输出的补丁文件是不压缩的，可以在创建补丁时用该选项指定一个支持的压缩插件：
   * **-c-zlib**[-{1..9}]			使用zlib算法压缩，默认压缩级别9。
   * **-c-lzma**[-{0..9}[-dictSize]]	使用lzma算法压缩，默认压缩级别7，支持设置解压时字典大小，默认16MB（该值越大一般压缩的越小）；该插件支持2路并行压缩；（推荐使用lzma2）。
   * **-c-lzma2**[-{0..9}[-dictSize]]	使用lzma2算法压缩，默认压缩级别7，支持设置解压时字典大小，默认16MB；该插件支持多线程并行压缩。
   * **-c-zstd**[-{0..22}[-dictBits]]	使用zstd算法压缩，默认压缩级别20，支持设置解压时字典比特数(10--31)，默认24（即对应字典大小2^24=16MB）。

比如使用lzma2，使用64MB的字典，该算法压缩率高，但解压较慢：
`$ sf_diff "old.apk" "new.apk" "diff.pat" -o-3 -c-lzma2-9-64m`
比如使用zstd，使用64MB的字典，该算法压缩率不错，解压非常快：
`$ sf_diff "old.apk" "new.apk" "diff.pat" -o-2 -c-zstd-22-26`

* **-m 选项**：设置匹配最小分数(>=0)，默认值2；少量影响补丁包输出大小；一般输入数据可压缩性越大，这个值就可以设得越大。

* **-step 选项**：设置补丁包patch时的解压缩区步长，默认2MB；少量影响补丁包输出大小；一般这个值越大，输出文件越小。

* **-pre 选项**：设置是否始终让新版本数据处于解压缩状态，默认不(即保持原压缩状态不解压)；一般在diff过程中新旧数据都在解压缩状态进行匹配，当解压状态的部分新数据没有搜寻到匹配数据时，默认就会还原成压缩状态，这样有利于优化patch时的速度，减少需要重新压缩的数据量。 打开的情况下(即始终解压)，一般输出文件会小一些（相当于解压后用更好的压缩算法重新压缩了数据）。 
推荐用于当旧版本文件为空，并和-o-2和-c-lzma2一起用于新版本的初次下载优化，比如：
`$ sf_diff "" "new.apk" "recompressed.pat" –o-2 -c-lzma2-9-64m -p-8`

* **-lp 选项**：设置解压出临时数据的最大内存占用；该值可以控制patch时的最大内存占用值。当设置-o-2或-o-3时默认值为最大2文件解压缩后数据大小的和；而使用-o-1时一般不用设置；如果patch时使用临时文件来存放临时解压出的old数据，那-lp的设置值无意义；设置过小，可能会使输出的补丁包变大。 

* **-lo 选项**：设置旧版本数据的最大解压临时空间大小，默认不限制；这部分临时数据在patch的时候可以选择放置到临时文件中或者放置在内存中（而-lp的设置值可以控制其使用内存时的最大内存占用）；设置过小，可能会使输出的补丁包变大。 

* **-ln 选项**：设置新版本数据的最大解压大小，默认不限制；这部分数据在patch的时候需要重新压缩还原；设置过小，可能会使输出的补丁包变大，设置过大可能对patch速度有影响。

* **-p 选项**：设置输出补丁包文件数据时允许的并行压缩线程数。

* **-d 选项**：设置输出是否只执行diff，不执行patch校验检查。

* **-t 选项**：只执行patch校验检查，在旧版本数据上应用已经存在的补丁数据，看得到的数据是否和新版本数据完全相同。

* **-v 选项**： 输出当前程序的版本等信息。

推荐做法：如果为了较小的补丁,并且最快的打补丁速度，那可以使用 -o-1 -c-zstd-22-25类似的参数；如果想要更小的补丁，并且有较快的速度，建议使用 -o-2，并设置-lp-32m等来控制patch时的最大内存占用。 当patch已经很快时可以使用-c-lzma2来得到更小一些的补丁包（牺牲部分patch速度）。

# PC上打补丁的工具sf_patch
patch端一般在安卓手机上运行（库提供了.so库文件和java代码），这里提供了一个在PC上测试执行的命令行工具；直接在PC上运行sf_patch命令会得到如下所示的帮助信息输出： 
```
usage: sf_patch oldArchiveFile diffFile outNewArchiveFile [maxUncompressMemory tempUncompressFileName] [-lp] [-v] [-p-parallelThreadNumber]
  if oldArchiveFile is empty input parameter ""

options:
  if DEFAULT or decompressDataSize<=maxUncompressMemory,
      decompress data all save in memory, run fast;
  else if maxUncompressMemory>=limitPatchMemSize,
      decompress data part save in limit memory step by step;
      limitPatchMemSize is set by -lp when run sf_diff;
  else if maxUncompressMemory<limitPatchMemSize,
      save decompress data into file tempUncompressFileName;
    maxUncompressMemory can like 8388608,8m,40m,120m,1g etc...
  -lp
      set maxUncompressMemory=limitPatchMemSize, patch run with limit memory.
  -v  output Version info.
  -p-parallelThreadNumber
    open multi-thread Parallel patch model, DEFAULT closed;
    if parallelThreadNumber>=2 then model opened!
```

* **打补丁**：可以使用命令行工具sf_patch命令在PC上测试补丁包，将旧版本apk文件应用相应的补丁文件升级到新版本的apk文件的命令如下：
`$ sf_patch "old.apk" "diff.pat" "out_new.apk"`
如果没有旧版本，下载的只是新版本解压后的重新压缩包，那命令为：
`$ sf_patch "" "recompressed.pat" "out_new.apk"`
	生成成功，返回0，其他任何值都是发现了错误。

* 临时**解压空间**：设置maxUncompressMemory和tempUncompressFileName参数可以用来控制patch过程中需要解压的临时数据的存储方式； 如果maxUncompressMemory大于等于临时解压空间大小，那所有解压数据完整存放在内存中，并且patch速度最快；如果maxUncompressMemory小于临时解压空间大小，但大于等于解压数据的最大内存占用限制值，则临时解压数据在patch时依次解压到内存中使用，patch速度稍慢；如果maxUncompressMemory小于解压数据的最大内存占用限制值，则临时解压数据会全部存放到临时文件中，这是patch速度会慢一些。（参见diff时的-lo选项，可以控制需要解压的数据大小的上限，-lp可以限制解压数据的最大内存占用）
	比如，启用8个线程，并控制最大解压内存占用不超过32MB，否则使用临时文件来暂存这些数据(程序执行完成后sfpatch.tmp会被删除)：
`$ sf_patch "old.apk" "diff.pat" "out_new.apk" 32m "sfpatch.tmp" -p-8`   
	推荐做法：diff时始终设置合适的-lp限制值，从而控制patch时的最大内存占用，并且不使用临时文件。

* **-lp 选项**： 设置maxUncompressMemory=limitPatchMemSize，从而使用限制内存占用的模式执行patch；limitPatchMemSize是diff时指定的。   

* **-p 选项**：设置patch时使用的最大线程数，一般来说线程数越多速度越快。很多时候该参数对-o-0生成的补丁加速作用较小，对-o-1生成的补丁有不错的并行加速效果（可以不用给太多的线程数）；而对-o-2和-o-3生成的补丁并行加速效果会非常好。并行调度代码能很好的支持big.LITTLE大小核架构的CPU。

* **-v 选项**： 输出当前程序的版本等信息。 
