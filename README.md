# sfpatcher：针对应用商店的apk增量算法

联系作者： <housisong@hotmail.com>

## 定义
**app**：本文档一般指智能手机和Pad设备上的软件，包括应用和游戏等。   
**应用商店**：这里指当前安卓智能设备上负责管理app的软件，支持app的下载、更新、推荐等功能；软件也可能分成**应用市场**和**游戏中心**两个独立应用。   
**apk**：安卓设备上的约定的软件包格式，它属于特殊约定格式的zip压缩档案格式的一种。为了防止篡改，发布的apk都带有签名。   
**diff**：这里指一种创建补丁的算法，它能计算出两个数据之间的差异，比如生成从旧版本数据到新版本之间的补丁数据。   
**patch**：打补丁，diff算法的逆算法，它能在旧版本数据基础上应用补丁来得到新版本的数据。   
## 背景
随着智能手机的普及和应用数量的爆发式增长，手机上大量app的下载和更新将产生巨大的数据流量。手机上的应用商店一般都会部署增量更新系统来节省大量的服务带宽成本和提升用户体验（节省用户流量和减少下载时间），而不用每次都下载完整版本。   

应用商店使用的增量更新算法一般选择使用的是基于字节的 diff 和 patch 算法，包括：[BsDiff ](http://www.daemonology.net/bsdiff)、[xdelta3](https://github.com/jmacd/xdelta)、[HDiffPatch](https://github.com/sisong/HDiffPatch)等。   
 - **BsDiff** 是当前选择使用较多的算法，创建的补丁小，代码量小，容易移植。但diff和patch场景下执行速度都很慢、内存占用巨大，应用于应用商店等场景时需要进行一些修改定制。
 - **xdelta3** 在diff和patch场景下执行速度都很快，输出标准化的补丁格式(gdiff 格式)，apk补丁大小和BsDiff接近或略大，内存占用中等；**三星**的应用商店就应用了 gdiff 补丁。该算法在diff处理较大文件的时候（比如几百MB以上，而现在很多游戏apk都接近2GB了），常会输出不正常的巨大补丁，除非使用和文件大小相当的参考内存来diff和patch(这是原理上决定的)，而这时内存占用的优势就没有了。
 - **HDiffPatch** 是笔者开源的算法，创建的补丁一般比BsDiff略小一些，diff(-s模式)和patch场景下执行速度都很快，内存占用都很小。 diff时也支持-m模式，用更大的内存和时间代价(这时也比BsDiff快得多)来得到更小一些的补丁包。 HDiffPatch 现在已经被**腾讯**、**OPPO**、**VIVO**等的应用商店所使用，也被**华为**开源的IoT操作系统用于ota升级；从为8位CPU、4KB内存、2百KB储存的IoT设备创建ota增量补丁(4KB内存里一边解压一边patch!)，到为上百GB的游戏创建更新补丁，HDiffPatch 算法得到了越来越广泛的应用。
## 针对apk的diff&patch算法
现在提到的增量更新实现方案都只是把apk单纯的看作文件数据直接用算法进行diff&patch，而没有考虑apk包本身是一个zip压缩包的事实。这种简单使用方式可以概括为公式：
```c
  diffFile:= diff(oldApk,newApk)
  newApk  := patch(oldApk,diffFile)
```
大家应该都有这样的经验：一个大文件在1/4处简单进行修改，如果把修改前后的2个文件分别压缩成zip文件，用二进制工具打开2个zip文件对比会发现：前面一部分的编码相同（如果前后压缩率相当，那这部分占1/4），但后面部分的编码数据完全不同。   
压缩算法破坏了修改“现场”，diff算法的优势无法真正发挥出来(生成了巨大的补丁)。那么我们是否可以开发针对apk文件格式的diff&patch算法呢？ 它能够识别出上面的情况：其实只是修改了一个字符！(即生成极小的补丁)   
很容易想到的一个优化思路，先解压再diff不就好了！原理可以概括为公式：
```c
  diffFile:=diff(decompress(oldApk),decompress(newApk))
  newApk  :=recompress(patch(decompress(oldApk),diffFile))
```
公式里面的diff和patch函数很好办，选择上面提到的一种基于字节的diff&patch算法就行；decompress函数的实现也很简单，就是zip包的解压缩算法；recompress函数的实现比较麻烦一些，需要保证精确的原样还原newApk（避免破坏签名和运行等）。   
而 [archive-patcher](https://github.com/google/archive-patcher) 和  [ApkDiffPatch](https://github.com/sisong/ApkDiffPatch) 正是这种思路的实现方案（也包括本文章要介绍的 sfpatcher 方案）。   
 - **archive-patcher** 谷歌开源的一个针对apk文件的diff&patch实现，在谷歌play商店中使用。内部使用了BsDiff算法作为基础，针对解压大小小于512MB的并且使用了zlib的deflate压缩算法创建的apk文件，执行优化的补丁算法。 该方案在patch时比较慢，特别是部分略大的apk文件使用了较高的压缩级别的，这时重新压缩出新版本apk时会慢的让用户无法接受，所以该优化方案一般用在可以后台更新的场景下。
 - **ApkDiffPatch** 是笔者开源的算法，为给自己团队开发的apk更新来开发的方案，创建的补丁平均比archive-patcher小很多，patch速度中等；方案一般用在可以自己对apk进行重新签名的场景，而不能用于应用商店。内部使用了HDiffPatch算法作为基础，不支持zip64格式的apk包。为了patch时精确还原，和优化压缩时的速度，需要对发布的apk文件执行ApkNormalized预处理流程（使用了较快的压缩参数，处理过的apk需要重新签名）。 patch时可以支持并行压缩来加快apk还原速度，但这时内存资源占用比较严重，和多个线程正在分别压缩的多个文件源大小和其压缩后大小的总和相当。
# sfpatcher 是什么？
针对压缩档案文件的高性能增量更新方案。类似于archive-patcher方案可用于应用商店的diff&patch算法，该领域的重要技术进展。   
内部使用了HDiffPatch算法作为基础，支持为deflate格式压缩的数据创建优化的补丁(用zlib压缩的支持最好)，支持创建多种档案格式 apk (zip、zip64、jar)、gz、tar等（和其嵌套）文件之间的优化补丁。   
补丁输出支持多种压缩算法zlib、bz2、lzma、lzma2、zstd、lzham、brotli等；源代码跨平台；……    
# sfpatcher 之道
- 针对应用商店的场景专门设计，patch时精确快速还原任意apk文件，能够用于用户交互场景。
- 多级可选的补丁包大小，极致的patch速度：提供比谷歌**archive-patcher**方案下载补丁略小4%的情况下，patch速度是其7倍(这时比BsDiff方案小50%并且速度更快)！补丁比其大7%的情况下，速度是其13倍！补丁比其小21%的情况下，平均速度是其2.8倍！  （注1）
- patch时的内存等资源占用可控；支持多种方案限制patch时的内存占用到合适的约定水平。支持patch时O(1)内存占用(并且不使用临时文件)的优化补丁包！
- 优化的用户体验：下载数据的同时就可以开始patch，不用将补丁文件保存到内存或硬盘上，结合快速的patch，很多时候下载完成时，就可以得到了完整的新版本apk文件。
- 利用精确还原算法，对于初次下载的apk文件也能进行解压后的重压缩；从而节省用户初次下载apk时的带宽。最多平均比直接下载apk文件减少20%的数据，部分文件能减少35%的数据！

注1： 所有测试数据来源于收集的一些常用apk应用和游戏，共32个测试用例（见性能测试对比数据）

## 方案主要特性对比
- **sfpatcher 和 archive-patcher**：sfpatcher的patch端比archive-patcher快很多倍，能够满足各种使用场景的要求，而archive-patcher一般用于后台更新场景；sfpatcher 生成的补丁大小在很多情况下也可以比archive-patcher更小。
- **sfpatcher 和 ApkDiffPatch**：sfpatcher的patch端比ApkDiffPatch更快一些，内存占用更加可控；sfpatcher 生成的补丁平均比ApkDiffPatch大一些；sfpatcher 可用于广泛的场景，而ApkDiffPatch用于能够对apk文件进行重新签名的场景。

# 参与测试的diff&patch方案
- [BsDiff ](http://www.daemonology.net/bsdiff)
- [xdelta3](https://github.com/jmacd/xdelta)
- [HDiffPatch](https://github.com/sisong/HDiffPatch)
- [archive-patcher](https://github.com/google/archive-patcher) 
- [ApkDiffPatch](https://github.com/sisong/ApkDiffPatch)
- [sfpatcher](https://github.com/sisong/sfpatcher)

# 测试目的
对比多种diff&patch方案在apk文件增量更新方面的运行数据；   
测试项主要包括：diff时间、diff内存占用、补丁大小(用压缩率代替，压缩后补丁大小/新版本apk大小)、patch时间、patch内存占用（后面这3项指标可能更重要一些）   

# 测试用例
收集了32组测试用例，这些用例来源于一些较长时间收集到的常见应用和游戏。（限于有限的用例和收集偏差，数据和实际情况可能略有差异）   

| 编号|新apk <-- 旧apk|新apk大小|旧apk大小|
|----:|:----|----:|----:|
|1|12306_5.2.11.apk <-- 12306_5.1.2.apk| 61120025|66209244|
|2|alipay10.1.99.apk <-- alipay10.1.95.apk|94178674|90951351|
|3|alipay10.2.0.apk <-- alipay10.1.99.apk|95803005|94178674|
|4|baidumaps10.25.0.apk <-- baidumaps10.24.12.apk|95539893|104527191|
|5|baidumaps10.25.5.apk <-- baidumaps10.25.0.apk|95526276|95539893|
|6|bilibili6.15.0.apk <-- bilibili6.14.0.apk|74783182|72067209|
|7|chrome-64-0-3282-137.apk <-- chrome-64-0-3282-123.apk|43879588|43879588|
|8|chrome-65-0-3325-109.apk <-- chrome-64-0-3282-137.apk|43592997|43879588|
|9|didi6.0.2.apk <-- didi6.0.0.apk|100866981|91462767|
|10|firefox68.10.0.apk <-- firefox68.9.0.apk|43543846|43531470|
|11|firefox68.10.1.apk <-- firefox68.10.0.apk|43542786|43543846|
|12|google-maps-9-71-0.apk <-- google-maps-9-70-0.apk|50568872|51304768|
|13|google-maps-9-72-0.apk <-- google-maps-9-71-0.apk|54342938|50568872|
|14|jd9.0.0.apk <-- jd8.5.12.apk|96891703|94233891|
|15|jd9.0.8.apk <-- jd9.0.0.apk|97329322|96891703|
|16|jinianbeigu2_1.12.4.apk <-- jinianbeigu2_1.12.3.apk|171611658|159691189|
|17|lushichuanshuo19.4.71003.apk <-- lushichuanshuo19.2.69054.apk|93799693|93442621|
|18|meituan10.9.401.apk <-- meituan10.9.203.apk|88956726|89384406|
|19|minecraft1.17.30.apk <-- minecraft1.17.20.apk|373025314|370324338|
|20|minecraft1.18.10.apk <-- minecraft1.17.30.apk|401075178|373025314|
|21|popcap.pvz2_2.4.84.1010.apk <-- popcap.pvz2_2.4.84.1009.apk|387572492|386842079|
|22|supercell.clashofclans13.369.3.apk <-- supercell.clashofclans13.180.18.apk|152896934|149011539|
|23|tangmumaopaoku4.8.0.971.apk <-- tangmumaopaoku4.6.0.913.apk|105486308|104732413|
|24|taobao9.8.0.apk <-- taobao9.7.2.apk|178734456|176964070|
|25|taobao9.9.1.apk <-- taobao9.8.0.apk|184437315|178734456|
|26|tiktok11.5.0.apk <-- tiktok11.3.0.apk|88544106|87075000|
|27|translate6.9.0.apk <-- translate6.8.0.apk|28171978|28795243|
|28|translate6.9.1.apk <-- translate6.9.0.apk|31290990|28171978|
|29|weixin7.0.15.apk <-- weixin7.0.14.apk|148405483|147695111|
|30|weixin7.0.16.apk <-- weixin7.0.15.apk|158906413|148405483|
|31|wps12.5.2.apk <-- wps12.5.1.apk|51293286|51136905|
|32|yuanshichuanqi1.3.608.apk <-- yuanshichuanqi1.3.607.apk|192578139|192577253|

# 测试条件
在一台笔记本PC上对比测试，CPU Ryzen 4800H，Windows10   
BsDiff还是保持着使用bzip2算法压缩补丁。   
xdelta3使用`-e -n -f -s`来创建补丁, 而用`-d -f -s`参数来执行的patch。   
HDiffPatch支持2种diff模式，-s和-m模式分别测试，输出补丁用的lzma2压缩。   
archive-patcher一般使用brotli算法压缩补丁，这里为了diff速度并更好的和其他方案对比补丁大小，改成了diff时输出不压缩的补丁，然后再额外使用lzma2压缩补丁。 需要注意：这时收集到的diff数据不包含额外压缩时的时间和内存消耗，收集到的patch数据也不包含解压的时间和内存消耗。   
ApkDiffPatch使用了lzma来压缩输出的补丁。   
patch时标注tmpFile表示使用了临时文件来储存中间数据；而标注MT表示开启了多线程并行。   
sfpatcher支持4个级别的diff，-0,-1,-2和-3分别测试； sfpatcher支持不需要旧版本apk而直接重新压缩新版本apk的模式，标记为 -pre；sfpatcher支持多种压缩输出，这里测试了zstd和lzma2这2种。   

另外在一部安卓手机(CPU:Kirin980)上对sfpatcher进行了一些patch时间测试，补充到了最后一列。   
# 测试汇总   

| 方案|平均压缩率|平均内存(MB)|平均时间(秒)|patch|平均内存(MB)|平均时间(秒)|Kirin980时间(秒)|
|:----|----:|----:|----:|----|----:|----:|----:|
|bsdiff|59.8%|1035|188.5|mem|243|3.4|
|xdelta3|59.9%|228|36.2|mem|99|0.8|
|hdiffz -s-16|59.0%|252|9.8|mem|19|0.8|
|hdiffz -m-1|58.7%|781|77.8|mem|19|0.8|
|archive-patcher|28.4%|1745|199.6|tmpFile|62|7.7|
|ApkDiffPatch|20.7%|980|84.1|mem|137|6.7|
|ApkDiffPatch|20.7%|980|84.1|mem MT|218|3.1|
||
|sfpatcher -0 lzma2|58.6%|1071|78.4|mem|48|0.8|1.3|
|sfpatcher -0 zstd|58.7%|1238|82.4|mem|42|0.3|0.5|
|sfpatcher -1 lzma2|30.7%|1214|76.0|mem|114|2.0|3.0|
|sfpatcher -1 lzma2|30.7%|1214|76.0|mem MT|117|1.3|2.2|
|sfpatcher -1 zstd|31.9%|1435|81.7|mem|113|1.0|1.6|
|sfpatcher -1 zstd|31.9%|1435|81.7|mem MT|112|0.4|0.9|
|sfpatcher -1 limit zstd|32.4%|1138|84.9|limit mem|15|0.8||
|sfpatcher -1 limit zstd|32.4%|1138|84.9|limit mem MT|18|0.4||
|sfpatcher -1 limit zstd|32.4%|1138|84.9|mem MT|72|0.3||
|sfpatcher -2 lzma2|27.2%|1357|80.6|mem|139|4.0|5.5|
|sfpatcher -2 lzma2|27.2%|1357|80.6|mem MT|144|1.5|2.7|
|sfpatcher -2 zstd|28.5%|1560|86.8|mem|139|3.1|4.3|
|sfpatcher -2 zstd|28.5%|1560|86.8|mem MT|142|0.8|1.5|
|sfpatcher -3 lzma2|23.7%|1476|86.5|mem|156|7.5|10.8|
|sfpatcher -3 lzma2|23.7%|1476|86.5|mem MT|162|2.1|4.0|
|sfpatcher -3 lzma2|23.5%|1487|85.7|tmpFile|48|8.4||
|sfpatcher -3 lzma2|23.5%|1487|85.7|tmpFile MT|54|2.8||
|sfpatcher -3 zstd|25.1%|1663|92.9|mem|156|6.7|9.7|
|sfpatcher -3 zstd|25.1%|1663|92.9|mem MT|162|1.5|3.0|
|sfpatcher -2 -pre lzma2|82.0%|2593|51.9|mem|114|7.0|10.2|
|sfpatcher -2 -pre lzma2|82.0%|2593|51.9|mem MT|119|3.6|5.8|
|sfpatcher -2 -pre zstd|87.1%|1754|82.5|mem|94|3.7|5.2|
|sfpatcher -2 -pre zstd|87.1%|1754|82.5|mem MT|99|0.8|1.7|
|sfpatcher -3 -pre lzma2|77.0%|2830|58.1|mem|119|10.8|15.9|
|sfpatcher -3 -pre lzma2|77.0%|2830|58.1|mem MT|125|4.0|7.0|
|sfpatcher -3 -pre zstd|82.6%|1953|107.0|mem|112|7.7|11.4|
|sfpatcher -3 -pre zstd|82.6%|1953|107.0|mem MT|122|1.6|3.5|


# sfpatcher的大规模测试
收集了Top500中多个app应用和其多个历史版本，形成了4280个测试用例，进行了diff和patch测试并使用了lzma2压缩； 后来重新收集了更多数据形成了6495个测试用例，并使用了zstd压缩输出补丁。   

| 方案|平均压缩率|
|:----|----:|
|sfpatcher -0 lzma2|49.7%|
|sfpatcher -1 lzma2|30.8%|
|sfpatcher -2 lzma2|28.3%|
|sfpatcher -3 lzma2|26.0%|
|sfpatcher -2 -pre lzma2|81.4%|
|sfpatcher -3 -pre lzma2|76.4%|
||
|sfpatcher -0 zstd|50.9%|
|sfpatcher -1 zstd|32.8%|
|sfpatcher -2 zstd|30.7%|
|sfpatcher -3 zstd|28.3%|
|sfpatcher -0 -pre zstd|89.6%|
|sfpatcher -2 -pre zstd|85.5%|
|sfpatcher -3 -pre zstd|81.2%|


联系作者：<housisong@hotmail.com>


