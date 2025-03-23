联系作者： housisong@hotmail.com   
   
[**sfpatcher** 命令行工具下载](https://github.com/sisong/sfpatcher/releases)（支持Windows、Linux、MacOS）

# 创建补丁的工具 sf_diff
使用命令行工具 sf_diff 命令，来创建新旧版本apk文件之间的补丁，直接运行sf_diff命令会得到如下所示的帮助信息输出：
```
diff   usage: sf_diff [options] oldArchiveFile newArchiveFile outDiffFile
compress usage: [-c - ...] -pre "" newArchiveFile outDiffFile
test   usage: sf_diff -t oldArchiveFile newArchiveFile testDiffFile
resave usage: sf_diff [-c-...] diffFile outDiffFile
options:
    input oldArchiveFile & newArchiveFile can zip,apk,jar,tar,gz... file type;
    oldArchiveFile can empty, and input parameter ""
  -o-{0..3}
      select optimize level for patch speed or outDiffFileSize, DEFAULT -o-1;
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
  -e-ldefA
      open Enable ldefA encoder&decoder for normalized newArchiveFile, the
        apk file created by $sf_normalize -cl-A; out outDiffFile file size
        will smaller, and recompress speed is much faster when patching!
      NOTE: sf_diff & sf_patch both needed support ldefA & version>=v1.3.0!
  -m-matchScore
      matchScore>=0, DEFAULT -m-2, recommended: 0--6 etc...
  -c-compressType[-compressLevel]
      set outDiffFile Compress type & level, DEFAULT uncompress;
      for resave diffFile,recompress diffFile to outDiffFile by new set;
      support compress type & level:
        -c-zlib[-{1..9}]                    DEFAULT level 9
            support run by multi-thread parallel, fast!
        -c-lzma[-{0..9}[-dictSize]]         DEFAULT level 7
            dictSize can like 4096 or 4k or 4m or 128m etc..., DEFAULT 16m
            support run by 2-thread parallel.
        -c-lzma2[-{0..9}[-dictSize]]        DEFAULT level 7
            dictSize can like 4096 or 4k or 4m or 128m etc..., DEFAULT 16m
            NOTE: code not compatible with it compressed by -c-lzma!
            support run by multi-thread parallel, fast!
        -c-zstd[-{0..22}[-dictBits]]        DEFAULT level 20
            dictBits can 10--31, DEFAULT 24.
            support run by multi-thread parallel, fast!
  -block-fastMatchBlockSize
      set block match befor slow byte-by-byte match, DEFAULT -block-4k;
      if set -block-0, means don't use block match;
      fastMatchBlockSize>=4, recommended 256,1k,64k,1m etc...
      if newData similar to oldData then diff speed++ & diff memory--,
      but small possibility outDiffFile's size+
  -cache
      set is use a big cache for slow match, DEFAULT false;
      if newData not similar to oldData then diff speed++,
      big cache max used O(oldFileSize) memory, and build slow(diff speed--)
  -step-patchStepMemroySize
      set patch step memory size, DEFAULT -step-2m, recommended: 64k,512k,8m etc...
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
      if -o level>1 then level 1 decompress data is not limited by limitNewDecodeSize.
  -p-parallelThreadNumber
      DEFAULT -p-4!  if parallelThreadNumber>1 then
      open multi-thread Parallel mode when compress outDiffFile;
  -neq
      open check: if newArchiveFile & oldArchiveFile's datas are equal, then return error;
      DEFAULT not check equal.
  -d  Diff only, do't run patch check, DEFAULT run patch check.
  -t  Test only, run several times patch check; (or -T , do more tests)
      check sf_patch(oldArchiveFile,testDiffFile)==newArchiveFile?
  -v  output Version info.
```

 * **创建补丁**：比如你有一个旧版本的old.apk文件，现在有了一个新版本new.apk，创建升级补丁文件diff.pat的命令为：   
 `$ sf_diff "old.apk" "new.apk" "diff.pat"`   
创建成功，返回0，其他任何值都是发现了错误。 如果用户设备上安装了old.apk，下载diff.pat就能用本方案提供的patch算法升级到new.apk。   
 * 补丁**另存**：将已有的补丁文件使用新的压缩参数来重新压缩；   
比如使用lzma2算法开启8线程来重新压缩补丁：   
`$ sf_diff -c-lzma2-9-16m -p-8 "diff0.pat" "diff1.pat"`   
   
 * **附加参数** 都以“-”符号开头紧跟字母来进行不同的设置，可以放置在命令行的任意位置，如果该项设置还有参数的话后面继续用“-”间隔。   
 […] 框起来的部分表示可选项，如果不填，将使用预设置的默认值。   
 {1..9}表示可以选择1到9中的一个值。   
   
 * **-o 选项**：diff时的优化级别和策略，分别对应不同的补丁大小和patch速度；一般来说数值越大补丁包越小，而patch可能越慢。
   * **-o-0**	patch最快，但补丁包较大，和hdiffz生成的补丁包几乎一样大；patch时资源占用少，不需要临时解压空间。
   * **-o-1**	patch很快，补丁包足够小，patch时需要使用临时解压空间(临时空间的总大小上限可以用-lo控制，而用-lp控制其最大内存占用)。 该级别为默认值。
   * **-o-2**	当输入的压缩数据兼容zlib库并且压缩级别<=6时，可以得到较小的补丁，不兼容的压缩数据就会退回到-o-1模式；patch时还原压缩较慢(需要还原的数据量的上限可以用-ln控制)，并且patch时需要使用临时解压空间(见-lo和-lp)。
   * **-o-3**	当输入的压缩数据兼容zlib库时，一般可以得到最小的补丁文件；不兼容的压缩数据就会退回到-o-1模式；patch时还原压缩很慢(见-ln)，并且patch时需要使用临时解压空间(见-lo和-lp)。

 * **-e-ldefA 选项**：
   允许使用 ldefA 编解码器来处理标准化过的 new.apk 文件，该文件被 `$sf_normalize -cl-A` 处理过；这样可以得到更小的补丁包，并且patch合成时还原压缩速度还比较快！   
    注意：sf_diff 和 sf_patch 需要 v1.3.0 及以上版本才能支持该新的 ldefA 编解码器。(即使new.apk没有标准化过，diff时打开该新编解码器后，patch端也可能需要升级才能支持该新补丁)

* **-c 选项**：设置补丁数据使用的压缩算法；diff默认输出的补丁文件是不压缩的，可以在创建补丁时用该选项指定一个支持的压缩插件：
   * **-c-zlib**[-{1..9}]		使用zlib算法压缩，默认压缩级别9。
   * **-c-lzma**[-{0..9}[-dictSize]]	使用lzma算法压缩，默认压缩级别7，支持设置解压时字典大小，默认16MB（该值越大一般压缩的越小）；该插件支持2路并行压缩（推荐使用lzma2）。
   * **-c-lzma2**[-{0..9}[-dictSize]]	使用lzma2算法压缩，默认压缩级别7，支持设置解压时字典大小，默认16MB；该插件支持多线程并行压缩。
   * **-c-zstd**[-{0..22}[-dictBits]]	使用zstd算法压缩，默认压缩级别20，支持设置解压时字典比特数{10..31}，默认24（即对应字典大小2^24=16MB）。   
比如使用lzma2，使用32MB的字典，该算法压缩率高，但解压较慢：   
`$ sf_diff "old.apk" "new.apk" "diff.pat" -o-3 -c-lzma2-9-32m`   
比如使用zstd，使用16MB的字典，该算法压缩率不错，解压非常快：   
`$ sf_diff "old.apk" "new.apk" "diff.pat" -o-2 -c-zstd-21-24`   

* **-m 选项**：设置匹配最小分数(>=0)，默认值2；少量影响补丁包输出大小；一般输入数据可压缩性越大，这个值就可以设得越大。

* **-block 选项**：设置块匹配，在较慢的逐字节匹配之前使用基于块的快速匹配，默认-block-4k；如果设置为-block-0，意思是关闭基于块的提前匹配；块匹配大小fastMatchBlockSize推荐256,1k,64k,1m等; 如果新版本和旧版本相同数据比较多,那diff速度就会比较快,并且减少内存占用，但有很小的可能补丁包会变大。

* **-cache 选项**：设置是否在匹配的时候使用一个大缓存，从而优化匹配的速度，默认不；一般old原始数据和new不相同数据越多速度就越快，该缓存需要占用oldSize相当的内存，并且建立缓存较慢需要花费额外时间。

* **-step 选项**：设置补丁包patch时的解压缩区步长，默认2MB；少量影响补丁包输出大小；一般这个值越大，输出文件越小。

* **-pre 选项**：设置是否始终让新版本数据处于解压缩状态，默认不(即保持原压缩状态不解压)；一般在diff过程中新旧数据都在解压缩状态进行匹配，当解压状态的部分新数据没有搜寻到匹配数据时，默认就会还原成压缩状态，这样有利于优化patch时的速度，减少需要重新压缩的数据量。 打开的情况下(即始终解压)，一般输出文件会小一些（相当于解压后用更好的压缩算法重新压缩了数据）。    
推荐用于当旧版本文件为空，并和-o-2和-c-lzma2一起用于新版本的初次下载优化，比如：   
`$ sf_diff "" "new.apk" "recompressed.pat" -pre -o-2 -c-lzma2-9-32m -p-8`   

* **-lp 选项**：设置解压出临时数据的最大内存占用；该值可以控制patch时的最大内存占用值。当设置-o-2或-o-3时默认值为最大2文件解压缩后数据大小的和；而使用-o-1时一般不用设置(推荐-lp-512k)；如果patch时使用临时文件来存放临时解压出的old数据，那-lp的设置值无意义；设置过小，可能会使输出的补丁包变大。    
为了控制patch时的最大内存，推荐始终设置该值，比如：   
`$ sf_diff "old.apk" "new.apk" "diff.pat" -lp-16m -o-2 -c-zstd-21-24`   

* **-lo 选项**：设置旧版本数据的最大解压临时空间大小，默认不限制；这部分临时数据在patch的时候可以选择放置到临时文件中或者放置在内存中（而-lp的设置值可以控制其使用内存时的最大内存占用）；设置过小，可能会使输出的补丁包变大。 

* **-ln 选项**：设置新版本数据的最大解压大小limitNewDecodeSize，默认不限制；这部分数据在patch的时候需要重新压缩还原；设置过小，可能会使输出的补丁包变大，设置过大可能对patch速度有影响。如果-o级别>1时，级别1的解压数据不受limitNewDecodeSize的限制.

* **-p 选项**：设置输出补丁包文件数据时允许的并行压缩线程数。需要注意：多线程并行diff时创建的补丁包每次都可能数据不相同。

* **-neq 选项**：拒绝为相同数据创建补丁；该选项默认处于关闭状态，即不执行检查;   
  设置该参数后在sf_diff执行时会检查newArchiveFile和oldArchiveFile的文件数据是否完全相同，如果相同则直接返回失败。

* **-d 选项**：设置输出是否只执行diff，不执行patch校验检查。

* **-t 选项**：只执行基本的patch校验检查，不会执行diff，不会修改任何文件；
   在旧版本数据上应用已经存在的补丁数据，看得到的数据是否和新版本数据完全相同。而使用 **-T** 可以执行更多次的打补丁测试。

* **-v 选项**： 输出当前程序的版本等信息。
   

# PC上打补丁的工具 sf_patch
sf_patch 端一般在安卓手机上运行，提供了NDK编译出的.so库文件和其对应java代码（即SDK，使用需要获得商业授权）。   
*安卓库 libsfpatch.so (v1.2.0) 文件大小参考：   
 arm64-v8a 静态库 275KB (zip压缩后 134KB), 动态库 207KB (zip压缩后 110KB);   
 armeabi-v7a 静态库 198KB (zip压缩后 123KB), 动态库 170KB (zip压缩后 110KB).*   
   
直接在PC上运行 sf_patch 命令会得到如下所示的帮助信息输出： 
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
  -ct
      set is run patch with continue mode, DEFAULT false.
  -p-parallelThreadNumber
      DEFAULT -p-8!  if parallelThreadNumber>1 then
      open multi-thread Parallel patch mode!
  -t  test other patcher, diffFile created by
      $hdiffz $hdiffz -SD $hdiffz -BSD $bsdiff4 $hdiffz -VCD $xdelta3 -S $xdelta3 -S lzma $open-vcdiff
  -v  output Version info.
```

* **打补丁**：可以使用命令行工具sf_patch命令在PC上测试补丁包，将旧版本apk文件应用相应的补丁文件升级到新版本的apk文件的命令如下：  
`$ sf_patch "old.apk" "diff.pat" "out_new.apk"`   
如果没有旧版本，下载的只是新版本解压后的重新压缩包，那命令为：   
`$ sf_patch "" "recompressed.pat" "out_new.apk"`   
 生成成功，返回0，其他任何值都是发现了错误。   

* 临时**解压空间**：设置maxUncompressMemory和tempUncompressFileName参数可以用来控制patch过程中需要解压的临时数据的存储方式；   
如果maxUncompressMemory大于等于临时解压空间大小，那所有解压数据完整存放在内存中，并且多线程时patch速度最快；   
如果maxUncompressMemory小于临时解压空间大小，但大于等于解压数据的最大内存占用限制值，则临时解压数据在patch时依次解压到内存中使用，patch速度稍慢；   
如果maxUncompressMemory小于解压数据的最大内存占用限制值，则临时解压数据会全部存放到临时文件中，这时patch速度会慢一些。   
提示：sf_diff时 -lo选项可以控制需要解压的数据大小的上限，-lp选项可以限制解压数据的最大内存占用   
比如，启用8个线程，并控制最大解压内存占用不超过32MB，否则使用临时文件来暂存这些数据(程序执行完成后sfpatch.tmp会被删除)：   
`$ sf_patch "old.apk" "diff.pat" "out_new.apk" 32m "sfpatch.tmp" -p-8`   
推荐做法：diff时始终设置合适的-lp限制值，从而可以控制patch时的最大内存占用，避免使用临时文件。

* **-lp 选项**： 自动设置maxUncompressMemory=limitPatchMemSize，从而使用限制内存占用的模式执行patch；limitPatchMemSize是diff时指定的。   
**推荐用法**：diff时设置好-lp的值，patch时自动限制内存占用    
`$ sf_patch "old.apk" "diff.pat" "out_new.apk" -lp -p-8`   

* **-ct 选项**： 设置是否使用继续patch模式，默认不；该模式支持从上次程序意外结束时的outNewArchiveFile最后写入位置继续patch,从而得到更好的patch体验。   
继续patch时不会检查已有的outNewArchiveFile数据是否正确；继续时能够跳过已经输出的outNewArchiveFile数据的压缩过程，并跳过部分oldNewArchiveFile数据的解压过程，但不能跳过对diffFile文件的解压和解析等过程。

* **-p 选项**：设置patch时使用的最大线程数，一般来说线程数越多速度越快。很多时候该参数对-o-0生成的补丁加速作用较小，对-o-1生成的补丁有不错的并行加速效果（可以不用给太多的线程数）。   
而对-o-2和-o-3生成的补丁并行加速效果会非常好。并行调度代码能很好的支持big.LITTLE大小核架构的CPU。

* **-t 选项**： 测试其他兼容格式的补丁合成，支持**hdiffz**、**bsdiff**、**xdelta3 -S**、**xdelta3 -S lzma**、**open-vcdiff**创建的补丁包文件。   
对于bsdiff的补丁，和**bspatch**不同，本程序使用一个较小的固定内存大小O(1)的模式执行patch过程。

* **-v 选项**： 输出当前程序的版本等信息。 
   

# PC上的标准化工具 sf_normalize
为了在保持一定的patch合成速度的前提下，得到更小的补丁包，定义了一个定制的apk文件标准化流程：   
对要发布的apk文件，先使用 sf_normalize 命令进行处理，输出的新apk文件需要重新签名(如果apk只含v1版签名时不用重签)。   
注意：如果是第三方的apk文件，自己无法进行重新签名，那该流程需要第三方apk拥有方的配合才能使用。   
直接运行 sf_normalize 命令会得到如下所示的帮助信息输出：
```
usage: sf_normalize srcApk normalizedApk [options]
options:
  input srcApk file can *.zip *.jar *.apk file type;
    sf_normalize normalized zip file:
      recompress all compressed files's data by libdeflate or zlib,
      align file data offset in zip file (compatible with AndroidSDK#zipalign),
      remove all data descriptor, normalized Extra field & reserve Comment,
      compatible with jar sign(apk v1 sign), etc...
    if apk file only used apk v1 sign, don't re-sign normalizedApk file!
    if apk file used apk v2 sign or later, must re-sign normalizedApk file after normalized;
      release signedApk:=AndroidSDK#apksigner(normalizedApk)
  -cl-{4|A} , DEFAULT -cl-A
    if set -cl-4 , then used zlib compressor; compatible with all versions of
      sf_diff -o-3 & sf_patch; but patch speed is not as fast as -cl-A.
    if set -cl-A , then used libdeflate compressor; this is recommended,
      recompress speed is much faster when patching!
      NOTE: sf_diff & sf_patch both needed support ldefA & version>=v1.3.0!
  -as-alignSize
    set align size for uncompressed file in zip for optimize app run speed,
    1 <= alignSize <= PageSize, recommended 4,8, DEFAULT -as-8.
    NOTE: if not -ap-0, must PageSize%alignSize==0;
  -ap-pageAlignSoFile
    if found uncompressed .so file in the zip, need align it to page size?
      -ap-0      not page-align uncompressed .so files;
      -ap-4k     page-align uncompressed .so files to 4KB;
      -ap-16k    (or -ap-1) DEFAULT, page-align uncompressed .so files to 16KB;
      -ap-64k    page-align uncompressed .so files to 64KB.
  -q  quiet mode, don't print fileName
  -v  output Version info.
```

* **标准化**：可以使用命令行工具 sf_normalize 在PC上对apk文件进行标准化处理，命令如下：  
`$ sf_normalize "srcApk" "normalizedApk"`   
 生成成功，返回0，其他任何值都是发现了错误。   
 sf_normalize 支持 *.zip *.jar *.apk 文件格式，使用 libdeflate 或 zlib 重新压缩其中的文件数据, 并对齐文件数据偏移位置(和 AndroidSDK#zipalign 兼容), 移除所有数据描述符,并标准化扩展字段，保留注释, 兼容 jar签名(即apk v1版签名), 等...   
 如果 srcApk 只含v1版签名时，那输出的 normalizedApk 文件不用重新签名! 否则标准后的normalizedApk必须重新签名：   
 可以使用安卓NDK中的apksigner程序来重新签名, signedApk:=AndroidSDK#apksigner(normalizedApk)

* **-cl-{4|A} 选项**：默认为 -cl-A
 如果设置 -cl-4 , 则使用 zlib 压缩器; 兼容所有旧版本的 sf_diff -o-3 和 sf_patch; 但是patch合成速度没有 -cl-A 那么快。   
 如果设置 -cl-A , 则使用 libdeflate 压缩器; 这是推荐的用法, patch合成时重压缩速度快得多！   
  注意: sf_diff 和 sf_patch 需要 v1.3.0 及以上版本才能支持该新的 ldefA 编解码器。

* **-as-alignSize 选项**：
 设置未压缩文件的对齐大小，优化apk执行时的速度，1 <= alignSize <= 4k, 推荐 4,8, 默认 -as-8。   
  注意: 如果没有设置 -ap-0, 那alignSize必须满足PageSize整除要求，即 PageSize%alignSize==0

* **-ap-pageAlignSoFile 选项**：
 如果在apk文件中发现了未压缩的 .so 文件, 是否需要按内存页面大小对齐?   
  -ap-0     未压缩的 .so 文件不需要按页对齐;   
  -ap-4k    未压缩的 .so 文件需要按4KB页面对齐；
  -ap-16k   (或 -ap-1) 默认, 未压缩的 .so 文件需要按16KB页面对齐；
  -ap-64k   未压缩的 .so 文件需要按64KB页面对齐。

* **-q 选项**： 安静模式, 不要输出apk包中的文件名称。（只显示重要信息，并且程序执行速度更快）

* **-v 选项**： 输出当前程序的版本等信息。 
   

# 推荐做法：
* 获得较小的优化补丁，并有优化的patch速度：   
`$ sf_diff "old.apk" "new.apk" "diff.pat" -o-1 -c-zstd-21-24 -step-3m -lp-512k -cache`   
（patch时内存占用估算：16m+3m+0.5m，最大约30MB左右。 diff时的-p并行线程数量根据机器情况设置。）   
patch端建议参数：`$ sf_patch "old.apk" "diff.pat" "out_new.apk" -lp -p-6`   

* 如果想要更小的补丁，并且保持最差patch速度勉强可接受：   
`$ sf_diff "old.apk" "new.apk" "diff.pat" -o-2 -c-zstd-21-22 -step-2m -lp-8m -cache`   
（patch时内存占用估算：4m+2m+8m，最大约30MB左右。 压缩算法也可以考虑lzma2。）   
patch端建议参数：`$ sf_patch "old.apk" "diff.pat" "out_new.apk" -lp -p-8`   

* 如果想要最小的补丁，并且不太在意最差patch速度（比如后台空闲自动更新的场景）：   
`$ sf_diff "old.apk" "new.apk" "diff.pat" -o-3 -c-lzma2-9-4m -step-2m -lp-8m -cache`   
（patch时内存占用估算：4m+2m+8m，最大约30MB左右。）   
patch端建议参数：`$ sf_patch "old.apk" "diff.pat" "out_new.apk" -lp -p-10`   

* 对新版本的初次下载大小优化，并且保持最差patch速度勉强可接受：   
`$ sf_diff "" "new.apk" "recompressed.pat" -pre -o-2 -c-lzma2-9-16m`   
（patch时内存占用估算：0m+0m+16m，最大约30MB左右。）   
patch端建议参数：`$ sf_patch "" "recompressed.pat" "out_new.apk" -p-10`   

* SDK高级用法：patch线程数可以根据回调时得到的任务量和当前设备的CPU核心数和空闲情况动态调整。 当客户端patch回调时得知完整临时**解压空间**需要的内存远小于当前设备的空闲内存时，不限制patch内存占用；这样patch速度会更快一点。 如果知道客户端内存资源比较大，可以增大-lp和-c压缩字典的大小，得到更小一些的补丁包。   

* 标准化流程推荐做法：
  * 请确保sf_patch客户端已经升级到v1.3.0, 支持 ldefA 编解码器
  * 标准化apk文件：`$ sf_normalize new.apk normalized_new.apk -cl-A -q`
  * apk拥有者重新签名： `snnew.apk = AndroidSDK#apksigner(normalized_new.apk);`
  * 创建补丁包：`$ sf_diff "old.apk" "snnew.apk" "diff.pat" -o-1 -e-ldefA -c-zstd-21-22 -step-2m -lp-8m -cache`
  * patch端打补丁：`$ sf_patch "old.apk" "diff.pat" "out_new.apk" -lp -p-10`
  * 如果是首次下载，可以创建完整压缩包：`$ sf_diff "" "snnew.apk" "recompressed.pat" -pre -o-1 -e-ldefA -c-zstd-21-24`
  * 完整包在patch端解压：`$ sf_patch "" "recompressed.pat" "out_new.apk" -p-10`
  * 如果需要兼容所有旧版本sf_patch, 那推荐使用  `sf_normalize -cl-4` 命令进行标准化处理，并且 sf_diff 启用 -o-2 并且不使用-e-ldefA，可以输出几乎一样大小的优化包，只是patch的时候会慢些(慢45%)。
