﻿# sfpatcher：针对应用商店的apk增量算法
**v1.0.15已正式上线**，当前最新版本 v1.0.16   
[**sfpatcher** 命令行工具下载](https://github.com/sisong/sfpatcher/releases)（支持Windows、Linux、MacOS），
[命令行使用说明](https://github.com/sisong/sfpatcher/blob/master/cmdline_doc.md)   
需要商业授权，请联系作者： <housisong@hotmail.com>   

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
 - **HDiffPatch** 是笔者开源的算法，创建的补丁一般比BsDiff略小一些，diff(-s模式)和patch场景下执行速度都很快，内存占用可控并很小。 diff时也支持-m模式，用更大的内存和时间代价(这时也比BsDiff快很多倍)来得到更小一些的补丁包。 HDiffPatch 现在已经被**VIVO**、**OPPO**、**腾讯**、**华为**、**字节跳动**、**米哈游**等所使用；从为8位CPU、4KB内存、2百KB储存的IoT设备创建ota增量补丁(4KB内存里一边解压一边patch!)，到为上百GB的游戏创建更新补丁，HDiffPatch 算法得到了越来越广泛的应用。
## 针对apk的diff&patch算法
现在提到的增量更新实现方案都只是把apk单纯的看作文件数据直接用算法进行diff&patch，而没有考虑apk包本身是一个zip压缩包的事实。这种简单使用方式可以概括为公式：
```c
  diffFile:= diff(oldApk,newApk)
  newApk  := patch(oldApk,diffFile)
```
大家应该都有这样的经验：一个大文件在1/4处简单进行修改，如果把修改前后的2个文件分别压缩成zip文件，用二进制工具打开2个zip文件对比会发现：前面一部分的编码相同（如果前后压缩率相当，那这部分约占1/4），但后面部分的编码数据会完全不同。   
压缩算法破坏了修改“现场”，diff算法的优势无法真正发挥出来(生成了巨大的补丁)。那么我们是否可以开发针对apk文件格式的diff&patch算法呢？ 它能够识别出上面的情况：其实只是修改了一个字符！(即生成极小的补丁)   
很容易想到的一个优化思路，先解压再diff不就好了！原理可以概括为公式：
```c
  diffFile:=diff(decompress(oldApk),decompress(newApk))
  newApk  :=recompress(patch(decompress(oldApk),diffFile))
```
公式里面的diff和patch函数很好办，选择上面提到的一种基于字节的diff&patch算法就行；decompress函数的实现也较简单，就是zip包的解压缩算法；recompress函数的实现比较麻烦一些，需要保证精确的原样还原出newApk（避免破坏签名和运行等）。   
而 [archive-patcher](https://github.com/google/archive-patcher) 和  [ApkDiffPatch](https://github.com/sisong/ApkDiffPatch) 正是这种思路的实现方案，也包括本文章要介绍的 [sfpatcher](https://github.com/sisong/sfpatcher) 方案都基于这个思路。   
 - **archive-patcher** 谷歌开源的一个针对apk文件的diff&patch实现，在谷歌play商店中使用。内部使用了BsDiff算法作为基础，主要用java语言开发，针对解压状态小于512MB的并且使用了zlib的deflate压缩算法创建的apk文件，执行优化的补丁算法。 该方案在patch时比较慢，特别是部分略大的apk文件使用了较高的压缩级别时，这时重新压缩出新版本apk会慢的让用户无法接受，所以该优化方案一般用在可以后台更新的场景下。
 - **ApkDiffPatch** 是笔者开源的算法，为给自己团队开发的apk更新来开发的方案，创建的补丁平均比 archive-patcher 小很多，patch速度中等；方案一般用在可以自己对需要升级的apk进行重新签名的场景，而不能部署于应用商店。内部使用了 HDiffPatch 算法作为基础，用C\C++语言开发，不支持zip64格式的apk包。为了patch时精确还原，和加快压缩还原时的速度，需要对发布的apk文件执行 ApkNormalized 预处理流程（使用了较快的压缩参数，处理过的apk需要重新签名）。 patch时可以支持并行压缩来加快apk还原速度，但这时内存资源占用比较严重，和多个线程正在分别压缩的多个文件源大小和其压缩后大小的总和相当。
## sfpatcher 是什么？
针对压缩档案文件的高性能增量更新方案。类似于 archive-patcher 方案可部署于应用商店的diff&patch算法，该领域的重要技术进展。   
内部使用了 HDiffPatch 算法作为基础，用C\C++语言开发，当前支持为deflate格式压缩的数据创建优化的补丁(用zlib压缩的支持最好)，支持在多种档案格式文件之间创建优化的补丁。     
### sfpatcher 之道
- 针对应用商店的场景专门设计，优化补丁大小，支持大型游戏，patch时精确快速还原任意apk文件，能够用于用户交互场景。
- 多级可选的补丁包大小，极致的patch速度：提供比谷歌**archive-patcher**方案(+brotli-9压缩)下载补丁小10%的情况下，手机上patch速度是其8倍(这时补丁比BsDiff方案小约50%并且速度快得多)！补丁比其略大1%的情况下，手机上速度是其21倍！补丁比其小21%的情况下，平均速度是其3.5倍！ （注1）
- patch时的内存等资源占用可控；diff时支持多种方案限制patch时的最大内存占用到合适的约定水平。即支持patch时O(1)平均内存占用的优化补丁包(并且patch不使用临时文件)！
- 优化的用户体验：支持下载数据的同时就可以开始patch，不用将补丁文件保存到内存或硬盘上，结合快速的patch，很多时候下载完成时，就可以得到了完整的新版本apk文件。（因为补丁变小，下载时间也会变短，从而可能缩短整体更新时间。）
- 利用精确还原算法，对于初次下载的apk文件也能进行解压后的重压缩；从而节省用户初次下载apk时的流量。最多情况下平均比直接下载apk文件减少22%的数据，部分文件能减少35%以上的数据！
- 支持从中断的位置继续patch的特性，节省程序被终止后再次执行程序时的打补丁时间。
- 高扩展性，框架支持多种压缩档案格式(apk、zip、zip64、jar、gz、tar等)和其嵌套(如apk中多个子apk); 档案格式本身和档案中数据的压缩算法都只是以插件的形式获得支持。patch端的执行，设计上不依赖于具体的档案格式。
- patch端支持对客户端的oldApk文件进行虚拟化；比如可以用一些简单的描述数据来移除oldApk(v1--v4签名)中添加的各种类型渠道号的影响，提高了补丁适应能力。
- diff端参数可选择性丰富，对各种使用场景可以定制性的设置合适的控制参数。
- patch结果提供丰富的错误号，以利于追踪patch失败的原因，提高升级成功率。

注1：（见性能测试对比数据）所有测试数据来源于收集的一些常用apk应用和游戏，共32个用例；在Kirin980上测试patch；archive-patcher 在patch上未包含解压缩补丁数据需要的时间。

## 方案主要特性对比
### **sfpatcher 和 archive-patcher**：
- sfpatcher 的patch端比 archive-patcher 快很多倍(相近大小的情况下快21x)，资源占用小(O(1))，能够满足各种使用场景的要求；而 archive-patcher 还原慢，资源占用大，一般用于后台更新场景。
- sfpatcher 生成的补丁大小在很多情况下也可以比 archive-patcher 的补丁（压缩后）更小。
- sfpatcher 创建压缩后的补丁，执行patch前不需要额外步骤解压缩补丁，边patch边随时解压缩补丁数据；而 archive-patcher 创建的补丁需要额外压缩和patch前解压。
- sfpatcher 在patch时不需要对oldApk提前进行解压，边patch边随时解压用到的oldApk中的数据，可以始终保持O(1)内存占用；而  archive-patcher 需要提前将oldApk中用到的数据全部解压缩到一个临时文件里。
- sfpatcher 支持大型apk文件，包括游戏等的更新升级，patch速度快；而 archive-patcher 有最大解压状态512M的大小限制，而且patch也很慢，某些情况下甚至patch需要几分钟。
- sfpatcher 基于C\C++，部署兼容性不受系统限制；结合JNI在安卓上使用，兼容安卓4.1到安卓13；而 archive-patcher 基于 java 并和系统环境耦合重，diff和patch时都可能会遇到环境兼容性问题。
### **sfpatcher 和 ApkDiffPatch**：
- sfpatcher 可用于广泛的场景，而ApkDiffPatch用于能够对apk文件进行重新签名的场景。
- sfpatcher 的patch端比 ApkDiffPatch 更快一些，多线程时内存占用也比较可控；而 ApkDiffPatch 在并行patch的时候内存占用较大。
- sfpatcher 生成的补丁平均比 ApkDiffPatch 稍大一些；当然对于同样ApkNormalized化过的apk，两个方案生成的补丁大小相近，即 sfpatcher 可以平滑的替代 ApkDiffPatch方案。
- sfpatcher 插件框架可支持多种档案格式，并且档案格式和patch端无关，兼容性和扩展性更好；而 ApkDiffPatch 的patch端和zip、apk格式、签名方式耦合过重，随着apk格式、签名方案和apk工具的变动，容易有兼容性问题(可能造成无法准确还原newApk或者无法优化补丁包大小)。

# 参与测试的diff&patch方案
- [BsDiff](http://www.daemonology.net/bsdiff)
- [xdelta3](https://github.com/jmacd/xdelta)
- [HDiffPatch](https://github.com/sisong/HDiffPatch)
- [archive-patcher](https://github.com/google/archive-patcher) 
- [ApkDiffPatch](https://github.com/sisong/ApkDiffPatch)
- [sfpatcher](https://github.com/sisong/sfpatcher)

# 测试目的
对比多种diff&patch方案在apk文件增量更新方面的运行数据；   
测试项主要包括：diff速度、diff内存占用、补丁大小(用压缩率代替，压缩后补丁大小/新版本apk大小)、patch速度、patch内存占用（后面这3项指标可能更重要一些）   

# 测试用例
收集了32组测试用例，这些用例来源于一些较长时间收集到的常见应用和游戏。（限于有限的用例和收集偏差，数据和实际情况可能略有差异）   
旧版本apk平均大小114.7MB，新版本apk平均大小117.1MB   

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
在一台笔记本PC上对比测试，CPU Ryzen 5800H，Windows11, SSD硬盘   
测试时关闭了HDiffPatch和sfpatcher在diff时的多线程。   
patch时标注tmpFile表示使用了临时文件来储存中间数据；mem表示在内存中执行不使用临时文件；limit mem表示使用限制内存占用的模式执行；而标注MT表示开启了多线程(8个)并行。   
**BsDiff** v4.3 还是保持着使用bzip2算法压缩补丁。   
**xdelta** v3.1.0 使用`-e -n -f -s`来创建补丁, 而用`-d -f -s`参数来执行的patch。   
**HDiffPatch** v4.2.4 支持2种diff模式，`-s-16`和`-m-1 -cache -block`模式分别测试，输出补丁时分别测试了用lzma2、zstd压缩和不压缩的测试。HDiffPatch支持输出兼容bsdiff的补丁(bzip2压缩)，补充了`-BSD -m-1 -cache -block`参数后的测试结果。   
**archive-patcher** v1.0 一般使用gzip或brotli算法压缩补丁，这里为了diff速度并更好的和其他方案对比补丁大小，diff时输出不压缩的补丁，然后再额外使用lzma2压缩补丁。 需要注意：这时收集到的diff数据不包含额外压缩时的时间和内存消耗，收集到的patch数据也**不包含**解压的时间和内存消耗等。   
**ApkDiffPatch** v1.3.6 使用了lzma来压缩输出的补丁。   
**sfpatcher** v1.0.16 支持4个级别的diff，-0,-1,-2和-3分别测试； sfpatcher支持不需要旧版本apk而直接重新压缩新版本apk的模式，标记为 -pre；sfpatcher支持多种可选压缩输出，这里测试了zstd-21和lzma2-9这2种。   
sfpatcher补充测试了用ApkNormalized(ApkDiffPatch方案)处理过的apk文件，分别进行增量测试和重压缩测试。   
   
另外在一部安卓手机(CPU:Kirin980)上对sfpatcher进行了一些patch时间测试，补充到了最后一列。   

# 测试汇总   

|diff方案|平均压缩率|平均内存|平均速度|patch|平均内存|最大内存|平均速度|Kirin980速度|
|:----|----:|----:|----:|----|----:|----:|----:|----:|
|**xdelta3 lzma**|**59.9 %**|**228 MB**|**2.9 MB/s**|mem|**100 MB**|**100 MB**|**159 MB/s**|
|**bsdiff bzip2**|**59.8 %**|**1035 MB**|**1.0 MB/s**|mem|**243 MB**|**751 MB**|**42 MB/s**|
|hdiffz -m-1 -BSD|59.5 %|523 MB|5.4 MB/s|mem|13 MB|14 MB|44 MB/s|
|hdiffz -m-1 no|59.9 %|523 MB|7.5 MB/s|mem|4 MB|5 MB|780 MB/s|
|**hdiffz -m-1 zstd**|**58.7 %**|**612 MB**|**5.0 MB/s**|mem|**13 MB**|**14 MB**|**680 MB/s**|
|hdiffz -m-1 lzma2|58.7 %|523 MB|3.7 MB/s|mem|12 MB|13 MB|285 MB/s|
|hdiffz -s-16 no|60.5 %|133 MB|31.8 MB/s|mem|3 MB|4 MB|806 MB/s|
|hdiffz -s-16 zstd|59.3 %|136 MB|9.7 MB/s|mem|12 MB|12 MB|763 MB/s|
|**archive-patcher**|28.5 %|**1740 MB**|**0.8 MB/s**|tmpFile|**64 MB**|**100 MB**|**15 MB/s**|
|ApkDiffPatch lzma|20.5 %|982 MB|2.0 MB/s|mem|138 MB|386 MB|21 MB/s|
|ApkDiffPatch lzma|20.5 %|982 MB|2.0 MB/s|mem MT|211 MB|461 MB|47 MB/s|
|ApkDiffPatch lzma|20.5 %|982 MB|2.0 MB/s|tmpFile|17 MB|22 MB|19 MB/s|
|ApkDiffPatch lzma|20.5 %|982 MB|2.0 MB/s|tmpFile MT|84 MB|207 MB|41 MB/s|
|sfpatcher -3 lzma2 Normalized|20.9 %|1032 MB|2.4 MB/s|limit mem|47 MB|57 MB|24 MB/s|
|**sfpatcher -3 lzma2 Normalized**|**20.9 %**|1032 MB|2.4 MB/s|limit mem MT|**52 MB**|**63 MB**|**78 MB/s**|
|sfpatcher -3 lzma2 Normalized|20.9 %|1032 MB|2.4 MB/s|mem MT|142 MB|397 MB|95 MB/s|
|sfpatcher -3 -pre lzma2 Normalized|73.6 %|601 MB|1.6 MB/s|mem|38 MB|41 MB|16 MB/s|
|**sfpatcher -3 -pre lzma2 Normalized**|**73.6 %**|601 MB|1.6 MB/s|mem MT|**43 MB**|**47 MB**|**55 MB/s**|
||
|sfpatcher -0 zstd|58.7 %|612 MB|4.9 MB/s|mem|13 MB|14 MB|716 MB/s|**265 MB/s**|
|sfpatcher -0 zstd|58.7 %|612 MB|4.9 MB/s|mem MT|14 MB|15 MB|890 MB/s|293 MB/s|
|sfpatcher -0 lzma2|58.7 %|523 MB|3.7 MB/s|mem|12 MB|13 MB|286 MB/s|160 MB/s|
|sfpatcher -0 lzma2|58.7 %|523 MB|3.7 MB/s|mem MT|13 MB|15 MB|342 MB/s|172 MB/s|
|sfpatcher -1 zstd|31.7 %|774 MB|2.8 MB/s|limit mem|16 MB|20 MB|227 MB/s|119 MB/s|
|**sfpatcher -1 zstd**|**31.7 %**|**774 MB**|**2.8 MB/s**|limit mem MT|**19 MB**|**22 MB**|**394 MB/s**|**218 MB/s**|
|sfpatcher -1 lzma2|30.8 %|725 MB|2.6 MB/s|limit mem|15 MB|19 MB|116 MB/s|65 MB/s|
|sfpatcher -1 lzma2|30.8 %|725 MB|2.6 MB/s|limit mem MT|18 MB|21 MB|170 MB/s|96 MB/s|
|sfpatcher -2 zstd|28.7 %|890 MB|2.6 MB/s|limit mem|17 MB|24 MB|48 MB/s|32 MB/s|
|**sfpatcher -2 zstd**|**28.7 %**|890 MB|2.6 MB/s|limit mem MT|**21 MB**|**30 MB**|**157 MB/s**|**85 MB/s**|
|sfpatcher -2 lzma2|27.5 %|859 MB|2.5 MB/s|limit mem|16 MB|24 MB|41 MB/s|26 MB/s|
|**sfpatcher -2 lzma2**|**27.5 %**|859 MB|2.5 MB/s|limit mem MT|**21 MB**|**29 MB**|**107 MB/s**|**59 MB/s**|
|sfpatcher -3 zstd|25.1 %|995 MB|2.3 MB/s|limit mem|19 MB|24 MB|21 MB/s|14 MB/s|
|sfpatcher -3 zstd|25.1 %|995 MB|2.3 MB/s|limit mem MT|24 MB|30 MB|80 MB/s|42 MB/s|
|sfpatcher -3 lzma2|23.7 %|976 MB|2.3 MB/s|limit mem|19 MB|24 MB|20 MB/s|13 MB/s|
|**sfpatcher -3 lzma2**|**23.7 %**|976 MB|2.3 MB/s|limit mem MT|**24 MB**|**29 MB**|**66 MB/s**|**36 MB/s**|
||
|sfpatcher -2 -pre zstd|87.6 %|517 MB|2.4 MB/s|mem|22 MB|26 MB|38 MB/s|25 MB/s|
|**sfpatcher -2 -pre zstd**|**87.6 %**|517 MB|2.4 MB/s|mem MT|**26 MB**|**33 MB**|**174 MB/s**|**80 MB/s**|
|sfpatcher -2 -pre lzma2|82.8 %|380 MB|1.8 MB/s|mem|22 MB|25 MB|24 MB/s|15 MB/s|
|**sfpatcher -2 -pre lzma2**|**82.8 %**|380 MB|1.8 MB/s|mem MT|**25 MB**|**31 MB**|**55 MB/s**|**30 MB/s**|
|sfpatcher -3 -pre zstd|83.2 %|545 MB|1.8 MB/s|mem|22 MB|26 MB|18 MB/s|12 MB/s|
|sfpatcher -3 -pre zstd|83.2 %|545 MB|1.8 MB/s|mem MT|28 MB|33 MB|80 MB/s|39 MB/s|
|sfpatcher -3 -pre lzma2|77.9 %|402 MB|1.6 MB/s|mem|22 MB|25 MB|14 MB/s|9 MB/s|
|**sfpatcher -3 -pre lzma2**|**77.9 %**|402 MB|1.6 MB/s|mem MT|**27 MB**|**31 MB**|**45 MB/s**|**24 MB/s**|


# sfpatcher的大规模测试
收集了Top500中多个app应用(不含游戏)和其多个历史版本，形成了4695个测试用例，进行了diff和patch多种参数测试并分别使用了lzma2压缩和zstd压缩输出补丁。   
测试条件：v1.0.15 用了-lp-2m加8m压缩字典 -pre时用的16m压缩字典   

| 方案|平均压缩率|
|:----|----:|
|sfpatcher -0 lzma2|50.8%|
|sfpatcher -1 lzma2|31.5%|
|sfpatcher -2 lzma2|29.3%|
|sfpatcher -3 lzma2|26.7%|
|sfpatcher -2 -pre lzma2|81.9%|
|sfpatcher -3 -pre lzma2|76.6%|
||
|sfpatcher -0 zstd|50.9%|
|sfpatcher -1 zstd|32.6%|
|sfpatcher -2 zstd|30.7%|
|sfpatcher -3 zstd|28.3%|
|sfpatcher -2 -pre zstd|86.3%|
|sfpatcher -3 -pre zstd|82.3%|

# 节省CDN带宽费用估算(仅供参考)
单个apk一次升级节省的流量估算：现在用户安卓手机经常使用的应用apk一般都越来越大，而经常使用的游戏平均应该更大，假设按平均100MB算。   
一般bsdiff或HDiffPatch创建的补丁平均为原apk大小的50%--60%（按51%计算）；sfpatcher按-o-1算，创建的补丁为平均原apk大小的33%。   
那sfpatcher相比bsdiff或HDiffPatch方案单次继续节省 100x(51%-33%) = 18 MB   
假设apk商店每天有2亿次apk升级，按峰值带宽计费，假设单价每天0.48元/Mbps; 而按经验，峰值带宽一般是平均带宽的2倍：   
每月节省费用：200000000x(18x1024x1024x8/1000/1000)/(3600x24)x2x0.48*30 = 1006.6 万元   
如果按流量计费，假设单价0.11元/GB：   
每月节省费用：200000000x(18/1024)x0.11x30 = 1160.2 万元   
   
---
需要商业授权，请联系作者： <housisong@hotmail.com>   


