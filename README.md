# sfpatcher：针对应用商店的apk增量算法
**v1.0.15 已正式上线**，为亿级手机终端用户提供优化的更新服务，当前最新版本 v1.1.1   
[**sfpatcher** 命令行工具下载](https://github.com/sisong/sfpatcher/releases)（支持Windows、Linux、MacOS），
[命令行使用说明](https://github.com/sisong/sfpatcher/blob/master/cmdline_doc.md)   
需要商业授权(含源代码&培训)，请联系作者： <housisong@hotmail.com>   

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
 - **xdelta3** 在diff和patch场景下执行速度都很快，输出标准化的补丁格式(vcdiff格式,也有改成gdiff的)，apk补丁大小和BsDiff接近或略大，内存占用中等。该算法在diff处理较大文件的时候（比如几百MB以上，而现在很多游戏apk都接近2GB了），常会输出不正常的巨大补丁，除非使用和文件大小相当的参考内存来diff和patch，而这时patch时内存占用可控的优点就没有了。
 - **HDiffPatch** 是笔者开源的算法，创建的补丁一般比BsDiff略小一些，diff(-s模式)和patch场景下执行速度都很快，内存占用可控并很小。 diff时也支持-m模式，用更大的内存和时间代价(这时也比BsDiff快很多倍)来得到更小一些的补丁包。 HDiffPatch 现在已经被**vivo**、**OPPO**、**小米**、**腾讯**、**华为**、**字节跳动**、**米哈游**等所使用；从为4KB内存、8位CPU的IoT设备创建OTA增量补丁(4KB内存里一边解压一边patch!)，到为上百GB的游戏创建更新补丁，HDiffPatch 算法得到了越来越广泛的应用。
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
 - **archive-patcher** 谷歌开源的一个针对apk文件的diff&patch实现，在谷歌play商店中使用。内部使用了BsDiff算法作为基础，主要用java语言开发，针对解压状态总数据量小于512MB的并且使用了zlib的deflate压缩算法创建的apk文件，执行优化的补丁算法。 该方案在patch时比较慢，特别是部分略大的apk文件使用了较高的压缩级别时，这时重新压缩出新版本apk会慢的让用户无法接受，所以该优化方案一般用在可以后台更新的场景下。
 - **ApkDiffPatch** 是笔者开源的算法，为给自己团队开发的apk更新来开发的方案，创建的补丁平均比 archive-patcher 小很多，patch速度中等；方案一般用在可以自己对需要升级的apk进行重新签名的场景，而不能部署于应用商店。内部使用了 HDiffPatch 算法作为基础，用C\C++语言开发。为了patch时精确还原，和加快压缩还原时的速度，需要对发布的apk文件执行 ApkNormalized 预处理流程（使用了较快的压缩参数，处理过的apk需要重新签名）。 patch时可以支持并行压缩来加快apk还原速度，但这时内存资源占用比较严重，和多个线程正在分别压缩的多个文件源大小和其压缩后大小的总和相当。
## sfpatcher 是什么？
针对压缩档案文件的高性能增量更新方案。类似于 archive-patcher 方案可部署于应用商店的diff&patch算法，该领域的重要技术进展。   
内部使用了 HDiffPatch 算法作为基础，用C\C++语言开发，当前支持为deflate格式压缩的数据创建优化的补丁(用zlib压缩的支持最好)，支持在多种档案格式文件之间创建优化的补丁。     
### sfpatcher 之道
- 针对应用商店的场景专门设计，优化补丁大小，支持大型游戏，patch时精确快速还原任意apk文件，能够用于用户交互场景。
- 多级可选的补丁包大小，极致的patch速度：提供比谷歌**archive-patcher**方案(+brotli-9压缩)下载补丁小10%的情况下，手机上patch速度是其8倍(这时补丁比BsDiff方案小约50%并且速度快得多)！补丁比其略大1%的情况下，手机上速度是其21倍！补丁比其小21%的情况下，平均速度是其3.5倍！ （注1）
- patch时的内存等资源占用可控；diff时支持多种方案限制patch时的最大内存占用到合适的约定水平。即支持patch时O(1)平均内存占用的优化补丁包(并且patch不使用临时文件)！
- 优化的用户体验：支持下载数据的同时就可以开始patch，不用将补丁文件保存到内存或硬盘上，结合快速的patch，很多时候下载完成时，就可以得到了完整的新版本apk文件。因为补丁变小，下载时间也会变短，从而可能缩短整体更新时间，也更省电省流量。
- 利用精确还原算法，对于初次下载的apk文件也能进行解压后的重压缩；从而节省用户初次下载apk时的流量。最多情况下平均比直接下载apk文件减少22%的数据，部分文件能减少35%以上的数据！
- 支持从中断的位置继续patch的特性，节省程序被终止后再次执行程序时的打补丁时间。
- 高扩展性，框架支持多种压缩档案格式(apk、zip、zip64、jar、gz、tar等)和其嵌套(如apk中多个子apk); 档案格式本身和档案中数据的压缩算法都只是以插件的形式获得支持。patch端的执行，设计上不依赖于具体的档案格式。
- patch端支持对客户端的oldApk文件进行虚拟化；比如可以用一些简单的描述数据来移除oldApk(v1--v4签名)中添加的各种类型渠道号的影响，提高了补丁适应能力。
- diff端参数可选择性丰富，对各种使用场景可以定制性的设置合适的控制参数。
- patch结果提供丰富的错误号，以利于追踪patch失败的原因，提高升级成功率。

注1：（见性能测试对比数据）所有测试数据来源于收集的一些常用apk应用和游戏，共32个用例；在Kirin980上测试patch；archive-patcher 在patch上未包含解压缩补丁数据需要的时间。

## 方案主要特性对比
### **sfpatcher 和 archive-patcher**：
- sfpatcher 的patch端比 archive-patcher 快很多倍(相近大小的情况下快21倍)，资源占用小(O(1))，能够满足各种使用场景的要求；而 archive-patcher 还原慢，资源占用大，一般用于后台更新场景。
- sfpatcher 生成的补丁大小在很多情况下也可以比 archive-patcher 的补丁（压缩后）更小。
- sfpatcher 创建压缩后的补丁，执行patch前不需要额外步骤解压缩补丁，边patch边随时解压缩补丁数据；而 archive-patcher 创建的补丁需要额外压缩和patch前解压。
- sfpatcher 在patch时不需要对oldApk提前进行解压，边patch边随时解压用到的oldApk中的数据，可以始终保持O(1)内存占用；而  archive-patcher 需要提前将oldApk中用到的数据全部解压缩到一个临时文件里。
- sfpatcher 支持大型apk文件，包括游戏等的更新升级，patch速度快；而 archive-patcher 有最大解压总数据512M的大小限制，而且patch也很慢，某些情况下甚至patch需要几分钟。
- sfpatcher 基于C\C++，部署兼容性不受系统限制；结合JNI在安卓上使用，兼容安卓4.1到安卓13；而 archive-patcher 基于 java 并和系统环境耦合重，diff和patch时都很可能会遇到环境兼容性问题。
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
收集了32组测试用例，这些用例来源于一些较长时间收集到的常见应用和游戏。   
限于有限的用例和收集偏差，数据和实际情况可能略有差异；因为bsdiff内存占用过大和archive-patcher 512M大小的限制放弃了大游戏用例。   
旧版本apk平均大小114.7MB，新版本apk平均大小117.1MB   

| 编号|app|新apk <-- 旧apk|新apk大小|旧apk大小|
|----:|:---:|:----|----:|----:|
|1|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.MobileTicket.png" width="36">|12306_5.2.11.apk <-- 5.1.2 | 61120025|66209244|
|2|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.eg.android.AlipayGphone.png" width="36">|alipay10.1.99.apk <-- 10.1.95 |94178674|90951351|
|3|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.eg.android.AlipayGphone.png" width="36">|alipay10.2.0.apk <-- 10.1.99 |95803005|94178674|
|4|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.baidu.BaiduMap.png" width="36">|baidumaps10.25.0.apk <-- 10.24.12 |95539893|104527191|
|5|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.baidu.BaiduMap.png" width="36">|baidumaps10.25.5.apk <-- 10.25.0 |95526276|95539893|
|6|<img src="https://github.com/sisong/sfpatcher/raw/master/img/tv.danmaku.bili.png" width="36">|bilibili6.15.0.apk <-- 6.14.0 |74783182|72067209|
|7|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.android.chrome.png" width="36">|chrome-64-0-3282-137.apk <-- 64-0-3282-123 |43879588|43879588|
|8|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.android.chrome.png" width="36">|chrome-65-0-3325-109.apk <-- 64-0-3282-137 |43592997|43879588|
|9|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.sdu.didi.psnger.png" width="36">|didi6.0.2.apk <-- 6.0.0 |100866981|91462767|
|10|<img src="https://github.com/sisong/sfpatcher/raw/master/img/org.mozilla.firefox.png" width="36">|firefox68.10.0.apk <-- 68.9.0 |43543846|43531470|
|11|<img src="https://github.com/sisong/sfpatcher/raw/master/img/org.mozilla.firefox.png" width="36">|firefox68.10.1.apk <-- 68.10.0 |43542786|43543846|
|12|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.google.android.apps.maps.png" width="36">|google-maps-9-71-0.apk <-- 9-70-0 |50568872|51304768|
|13|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.google.android.apps.maps.png" width="36">|google-maps-9-72-0.apk <-- 9-71-0 |54342938|50568872|
|14|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.jingdong.app.mall.png" width="36">|jd9.0.0.apk <-- 8.5.12 |96891703|94233891|
|15|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.jingdong.app.mall.png" width="36">|jd9.0.8.apk <-- 9.0.0 |97329322|96891703|
|16|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.tencent.tmgp.jnbg2.png" width="36">|jinianbeigu2_1.12.4.apk <-- 1.12.3 |171611658|159691189|
|17|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.blizzard.wtcg.hearthstone.cn.png" width="36">|lushichuanshuo19.4.71003.apk <-- 19.2.69054 |93799693|93442621|
|18|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.sankuai.meituan.png" width="36">|meituan10.9.401.apk <-- 10.9.203 |88956726|89384406|
|19|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.netease.mc.png" width="36">|minecraft1.17.30.apk <-- 1.17.20 |373025314|370324338|
|20|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.netease.mc.png" width="36">|minecraft1.18.10.apk <-- 1.17.30 |401075178|373025314|
|21|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.popcap.pvz2cthd.png" width="36">|popcap.pvz2_2.4.84.1010.apk <-- 2.4.84.1009 |387572492|386842079|
|22|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.supercell.clashofclans.png" width="36">|supercell.clashofclans13.369.3.apk <-- 13.180.18 |152896934|149011539|
|23|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.outfit7.talkingtomgoldrun.png" width="36">|tangmumaopaoku4.8.0.971.apk <-- 4.6.0.913 |105486308|104732413|
|24|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.taobao.taobao.png" width="36">|taobao9.8.0.apk <-- 9.7.2 |178734456|176964070|
|25|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.taobao.taobao.png" width="36">|taobao9.9.1.apk <-- 9.8.0 |184437315|178734456|
|26|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.ss.android.ugc.aweme.png" width="36">|tiktok11.5.0.apk <-- 11.3.0 |88544106|87075000|
|27|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.google.android.apps.translate.png" width="36">|translate6.9.0.apk <-- 6.8.0 |28171978|28795243|
|28|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.google.android.apps.translate.png" width="36">|translate6.9.1.apk <-- 6.9.0 |31290990|28171978|
|29|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.tencent.mm.png" width="36">|weixin7.0.15.apk <-- 7.0.14 |148405483|147695111|
|30|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.tencent.mm.png" width="36">|weixin7.0.16.apk <-- 7.0.15 |158906413|148405483|
|31|<img src="https://github.com/sisong/sfpatcher/raw/master/img/cn.wps.moffice_eng.png" width="36">|wps12.5.2.apk <-- 12.5.1 |51293286|51136905|
|32|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.tanwan.yscqlyzf.png" width="36">|yuanshichuanqi1.3.608.apk <-- 1.3.607 |192578139|192577253|

# 测试条件
在一台笔记本PC上对比测试，CPU Ryzen 5800H，Windows11, SSD硬盘   
测试时关闭了HDiffPatch和sfpatcher在diff时的多线程。   
patch时标注tmpf表示使用了临时文件来储存中间数据；mem表示在内存中执行不使用临时文件；limit表示使用限制内存占用的模式执行；而标注MT表示开启了多线程(8个)并行。   
**BsDiff** v4.3 还是保持着使用bzip2算法压缩补丁。   
**xdelta** v3.1.0 使用`-e -n -f -s`来创建补丁, 而用`-d -f -s`参数来执行的patch。   
**HDiffPatch** v4.2.4 支持2种diff模式，`-s-16`和`-m-1 -cache -block`模式分别测试，输出补丁时分别测试了用lzma2、zstd压缩的测试。HDiffPatch支持输出兼容bsdiff的补丁(bzip2压缩)，补充了`-BSD -m-1 -cache -block`参数后的测试结果。   
**archive-patcher** v1.0 一般使用gzip或brotli算法压缩补丁，这里为了diff速度并更好的和其他方案对比补丁大小，diff时输出不压缩的补丁，然后再额外使用lzma2压缩补丁。 需要注意：这时收集到的diff数据不包含额外压缩时的时间和内存消耗，收集到的patch数据也**不包含**解压的时间和内存消耗等。   
**ApkDiffPatch** v1.3.6 使用了lzma来压缩输出的补丁。   
**sfpatcher** v1.0.16 支持4个级别的diff，-0,-1,-2和-3分别测试； sfpatcher支持不需要旧版本apk而直接重新压缩新版本apk的模式(-pre)；sfpatcher支持多种可选压缩输出，这里测试了-c-zstd-21和-c-lzma2-9这2种。   
sfpatcher补充测试了用ApkNormalized(ApkDiffPatch方案)处理过的apk文件，分别进行增量测试和重压缩测试(标记为Norm)。   
   
另外在一部安卓手机(CPU:Kirin980)上对sfpatcher进行了一些patch时间测试，补充到了最后一列。   

# 测试结果   
其中：平均压缩率=(补丁大小/新apk大小)的平均值； 单次测试的内存统计值为峰值内存；

|diff方案|平均压缩率|平均内存|平均速度|patch|平均内存|最大内存|平均速度|Kirin980速度|
|:----|----:|----:|----:|----|----:|----:|----:|----:|
|zstd --patch-from|58.81%|2248M|3.0MB/s|mem|234M|741M|670MB/s|
|**xdelta3 lzma**|**59.92%**|**228M**|**2.9MB/s**|mem|**100M**|**100M**|**159MB/s**|
|**bsdiff bzip2**|**59.76%**|**1035M**|**1.0MB/s**|mem|**243M**|**751M**|**42MB/s**|
|hdiffz-m-1 -BSD|59.50%|523M|5.4MB/s|mem|13M|14M|44MB/s|
|**hdiffz-m-1 zstd**|**58.74%**|**612M**|**5.0MB/s**|mem|**13M**|**14M**|**680MB/s**|**265MB/s**|
|hdiffz-m-1 lzma2|58.67%|523M|3.7MB/s|mem|12M|13M|285MB/s|
|hdiffz-s-16 zstd-17|59.27%|136M|9.7MB/s|mem|12M|12M|763MB/s|
|**archive-patcher**|28.53%|**1740M**|**0.8MB/s**|tmpf|**64M**|**100M**|**15MB/s**|
|ApkDiffPatch|20.53%|982M|2.0MB/s|mem|138M|386M|21MB/s|
|ApkDiffPatch|20.53%|982M|2.0MB/s|memMT|211M|461M|47MB/s|
|ApkDiffPatch|20.53%|982M|2.0MB/s|tmpf|17M|22M|19MB/s|
|ApkDiffPatch|20.53%|982M|2.0MB/s|tmpfMT|84M|207M|41MB/s|
|sfpatcher-3 lzma2 Norm|20.88%|1032M|2.4MB/s|limit|47M|57M|24MB/s|
|**sfpatcher-3 lzma2 Norm**|**20.88%**|1032M|2.4MB/s|limitMT|**52M**|**63M**|**78MB/s**|
|sfpatcher-3 lzma2 Norm|20.88%|1032M|2.4MB/s|memMT|142M|397M|95MB/s|
|sfpatcher-3pre lzma2 Norm|73.64%|601M|1.6MB/s|mem|38M|41M|16MB/s|
|**sfpatcher-3pre lzma2 Norm**|**73.64%**|601M|1.6MB/s|memMT|**43M**|**47M**|**55MB/s**|
||
|sfpatcher-0 zstd|58.74%|612M|4.9MB/s|mem|13M|14M|716MB/s|265MB/s|
|sfpatcher-0 zstd|58.74%|612M|4.9MB/s|memMT|14M|15M|890MB/s|293MB/s|
|sfpatcher-0 lzma2|58.67%|523M|3.7MB/s|mem|12M|13M|286MB/s|160MB/s|
|sfpatcher-0 lzma2|58.67%|523M|3.7MB/s|memMT|13M|15M|342MB/s|172MB/s|
|sfpatcher-1 zstd|31.70%|774M|2.8MB/s|limit|16M|20M|227MB/s|119MB/s|
|**sfpatcher-1 zstd**|**31.70%**|**774M**|**2.8MB/s**|limitMT|**19M**|**22M**|**394MB/s**|**218MB/s**|
|sfpatcher-1 lzma2|30.76%|725M|2.6MB/s|limit|15M|19M|116MB/s|65MB/s|
|sfpatcher-1 lzma2|30.76%|725M|2.6MB/s|limitMT|18M|21M|170MB/s|96MB/s|
|sfpatcher-2 zstd|28.75%|890M|2.6MB/s|limit|17M|24M|48MB/s|32MB/s|
|**sfpatcher-2 zstd**|**28.75%**|890M|2.6MB/s|limitMT|**21M**|**30M**|**157MB/s**|**85MB/s**|
|sfpatcher-2 lzma2|27.53%|859M|2.5MB/s|limit|16M|24M|41MB/s|26MB/s|
|**sfpatcher-2 lzma2**|**27.53%**|859M|2.5MB/s|limitMT|**21M**|**29M**|**107MB/s**|**59MB/s**|
|sfpatcher-3 zstd|25.13%|995M|2.3MB/s|limit|19M|24M|21MB/s|14MB/s|
|sfpatcher-3 zstd|25.13%|995M|2.3MB/s|limitMT|24M|30M|80MB/s|42MB/s|
|sfpatcher-3 lzma2|23.70%|976M|2.3MB/s|limit|19M|24M|20MB/s|13MB/s|
|**sfpatcher-3 lzma2**|**23.70%**|976M|2.3MB/s|limitMT|**24M**|**29M**|**66MB/s**|**36MB/s**|
||
|sfpatcher-2pre zstd|87.61%|517M|2.4MB/s|mem|22M|26M|38MB/s|25MB/s|
|**sfpatcher-2pre zstd**|**87.61%**|517M|2.4MB/s|memMT|**26M**|**33M**|**174MB/s**|**80MB/s**|
|sfpatcher-2pre lzma2|82.84%|380M|1.8MB/s|mem|22M|25M|24MB/s|15MB/s|
|**sfpatcher-2pre lzma2**|**82.84%**|380M|1.8MB/s|memMT|**25M**|**31M**|**55MB/s**|**30MB/s**|
|sfpatcher-3pre zstd|83.18%|545M|1.8MB/s|mem|22M|26M|18MB/s|12MB/s|
|sfpatcher-3pre zstd|83.18%|545M|1.8MB/s|memMT|28M|33M|80MB/s|39MB/s|
|sfpatcher-3pre lzma2|77.92%|402M|1.6MB/s|mem|22M|25M|14MB/s|9MB/s|
|**sfpatcher-3pre lzma2**|**77.92%**|402M|1.6MB/s|memMT|**27M**|**31M**|**45MB/s**|**24MB/s**|
   

# 游戏测试用例
最近收集了32组游戏测试用例，这些apk下载于小米应用商店、TapTap商店、谷歌Play商店，按照下载量大和最近进行过更新为标准进行收集。   
对比测试了 xdelta、HDiffPatch、sfpatcher； 因为有大游戏所以放弃了无法顺利完成测试的bsdiff和archive-patcher。   
**sfpatcher** v1.1.0 测试时，调整了部分参数，相比前面的测试增大了patch时的内存需求。   
旧版本apk平均大小1029.5MB，新版本apk平均大小1040.9MB   

| 编号|app|新apk <-- 旧apk|新apk大小|旧apk大小|
|----:|:---:|:----|----:|----:|
|1|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.blizzard.wtcg.hearthstone.png" width="36">|com.blizzard.wtcg.hearthstone_24.4.150659.apk <-- 24.2.148211 |119192491|118798993|
|2|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.hypergryph.arknights.png" width="36">|com.hypergryph.arknights_1.9.21.apk <-- 1.9.01|1577358049|1530216284|
|3|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.igg.android.lordsmobile.png" width="36">|com.igg.android.lordsmobile_2.88.apk <-- 2.86|69151111|68970887|
|4|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.imangi.templerun2.png" width="36">|com.imangi.templerun2_6.5.1.apk <-- 6.5.0|142692127|141981473|
|5|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.joym.legendhero.mi.png" width="36">|com.joym.legendhero.mi_6.0.0.apk <-- 5.0.0|857787066|801198664|
|6|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.kiloo.subwaysurf.png" width="36">|com.kiloo.subwaysurf_3.36.1.apk <-- 3.36.0|172657047|172348585|
|7|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.king.candycrushsaga.png" width="36">|com.king.candycrushsaga_1.237.0.3.apk <-- 1.236.0.3 |89619604|89410785|
|8|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.knight.union.mi.png" width="36">|com.knight.union.mi_4.3.1.apk <-- 4.3.0 |281337183|279734645|
|9|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.mfp.jelly.xiaomi.png" width="36">|com.mfp.jelly.xiaomi_8.20.0.6.apk <-- 8.19.5.2 |454515054|444262109|
|10|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.miHoYo.enterprise.NGHSoD.png" width="36">|com.miHoYo.enterprise.NGHSoD_6.1.0.apk <-- 6.0.0 |633463833|617956855|
|11|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.miHoYo.Yuanshen.png" width="36">|com.miHoYo.Yuanshen_3.1.0.apk <-- 3.0.0 |292959792|281028246|
|12|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.netease.dwrg.mi.png" width="36">|com.netease.dwrg.mi_1.5.69.apk <-- 1.5.67 |2087354104|2083304114|
|13|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.netease.g104.mi.png" width="36">|com.netease.g104.mi_3.18.1.apk <-- 3.17.5 |1900704615|1853407883|
|14|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.netease.g67.mi.png" width="36">|com.netease.g67.mi_1.6.0.apk <-- 1.5.4 |2021330101|2016955181|
|15|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.netease.harrypotter.png" width="36">|com.netease.harrypotter_1.20.212190.apk <-- 1.20.211260 |2146351076|1976476298|
|16|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.netease.l10.mi.png" width="36">|com.netease.l10.mi_1.11.5.apk <-- 1.11.4 |1897664156|1835964179|
|17|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.netease.mc.mi.png" width="36">|com.netease.mc.mi_2.3.15.apk <-- 2.3.5 |986369238|982878980|
|18|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.netease.onmyoji.png" width="36">|com.netease.onmyoji_1.7.48.apk <-- 1.7.47 |2037653577|2029429678|
|19|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.netease.sky.png" width="36">|com.netease.sky_0.10.0.apk <-- 0.9.9 |1320667004|1188304200|
|20|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.nianticlabs.pokemongo.png" width="36">|com.nianticlabs.pokemongo_0.251.0.apk <-- 0.249.2 |51919509|51538569|
|21|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.outfit7.mytalkingtom2.mi.png" width="36">|com.outfit7.mytalkingtom2.mi_3.5.0.514.apk <-- 3.4.0.496 |151118938|152738799|
|22|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.papegames.nn4.cn.png" width="36">|com.papegames.nn4.cn_2.1.1147531.apk <-- 2.1.1094661 |1804200058|1911303194|
|23|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.rovio.baba.dbzq.m.png" width="36">|com.rovio.baba.dbzq.m_3.6.0.apk <-- 3.2.1 |285160536|298220983|
|24|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.tencent.ig.png" width="36">|com.tencent.ig_2.2.0.apk <-- 2.1.0 |1123848712|1161724820|
|25|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.tencent.lolm.png" width="36">|com.tencent.lolm_3.5.0.6093.apk <-- 3.4.0.5935 |1743471226|1956378264|
|26|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.tencent.tmgp.cf.png" width="36">|com.tencent.tmgp.cf_1.0.280.apk <-- 1.0.260 |2089132074|1998168460|
|27|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.tencent.tmgp.cod.png" width="36">|com.tencent.tmgp.cod_1.9.35.apk <-- 1.9.34 |2102131414|2109943466|
|28|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.tencent.tmgp.sgame.png" width="36">|com.tencent.tmgp.sgame_3.81.1.8.apk <-- 3.74.1.6 |1960032646|1976668388|
|29|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.tencent.tmgp.speedmobile.png" width="36">|com.tencent.tmgp.speedmobile_1.33.0.apk <-- 1.32.0 |2122819058|2096029603|
|30|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.ustwo.monumentvalleyzz.png" width="36">|com.ustwo.monumentvalleyzz_2.5.2.apk <-- 2.5.1 |159780651|160263061|
|31|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.wooduan.ssjj.mi.png" width="36">|com.wooduan.ssjj.mi_6.9.2.apk <-- 6.8.1 |1588557473|1534719411|
|32|<img src="https://github.com/sisong/sfpatcher/raw/master/gimg/com.yodo1.tew.mi.png" width="36">|com.yodo1.tew.mi_2.10.6.297.apk <-- 2.9.5.286 |655222738|622939950|
   

# 游戏测试结果   
其中：单次测试的内存统计值改为峰值私有内存。   
 **xdelta3 -B**是指增大引用窗口到oldApk文件相同大小进行的测试，其结果仅供参考，因为patch时内存占用也会随之同样增大。   

|diff方案|平均压缩率|平均内存|平均速度|patch|平均内存|最大内存|平均速度|最差速度|
|:----|----:|----:|----:|----:|----|----:|----:|----:|
|**xdelta3 lzma**|**54.14%**|423MB|2.3MB/s|mem|**99MB**|**104MB**|**54MB/s**|**17MB/s**|
|xdelta3 lzma +hpatchz -m|54.14%|423MB|2.3MB/s|mem|79MB|83MB|**191MB/s**|**63MB/s**|
|xdelta3 lzma -B|35.89%|6424MB|4.1MB/s|mem|1299MB|2090MB|131MB/s|46MB/s|
|xdelta3 lzma -B +hpatchz -m|35.89%|6424MB|4.1MB/s|mem|1043MB|2028MB|197MB/s|123MB/s|
|**hdiffz -zstd**|**35.41%**|3835MB|5.3MB/s|mem|20MB|20MB|**476MB/s**|**203MB/s**|
|hdiffz -lzma2|35.26%|3812MB|4.4MB/s|mem|19MB|20MB|253MB/s|106MB/s|
|sf_diff -1 -zstd|22.23%|4501MB|4.2MB/s|limit|23MB|35MB|300MB/s|112MB/s|
|**sf_diff -1 -zstd**|**22.23%**|4501MB|4.2MB/s|limitMT|**26MB**|**38MB**|**465MB/s**|**215MB/s**|
|sf_diff -2 -lzma2|19.61%|5166MB|3.8MB/s|limit|26MB|47MB|76MB/s|10MB/s|
|sf_diff -2 -lzma2|19.61%|5166MB|3.8MB/s|limitMT|32MB|54MB|186MB/s|32MB/s|
|sf_diff -3 -lzma2|17.87%|5685MB|3.6MB/s|limit|32MB|47MB|37MB/s|5MB/s|
|sf_diff -3 -lzma2|17.87%|5685MB|3.6MB/s|limitMT|39MB|54MB|119MB/s|20MB/s|   

# sfpatcher的大规模测试
收集了Top500中多个app应用(不含游戏)和其多个历史版本，形成了4695个测试用例，进行了diff和patch多种参数测试并分别使用了lzma2压缩和zstd压缩输出补丁。   
测试条件：v1.0.15 用了-lp-2m加8m压缩字典 -pre时用的16m压缩字典   

| 方案|平均压缩率|
|:----|----:|
|sfpatcher-0 lzma2|50.8%|
|sfpatcher-1 lzma2|31.5%|
|sfpatcher-2 lzma2|29.3%|
|sfpatcher-3 lzma2|26.7%|
|sfpatcher-2pre lzma2|81.9%|
|sfpatcher-3pre lzma2|76.6%|
||
|sfpatcher-0 zstd|50.9%|
|sfpatcher-1 zstd|32.6%|
|sfpatcher-2 zstd|30.7%|
|sfpatcher-3 zstd|28.3%|
|sfpatcher-2pre zstd|86.3%|
|sfpatcher-3pre zstd|82.3%|

# 节省CDN带宽费用估算(仅供参考)
单个apk一次升级节省的流量估算：用户的安卓手机现在经常使用的应用apk一般都越来越大(游戏平均更大)，假设按平均100MB算。   
一般bsdiff或HDiffPatch创建的补丁平均为原apk大小的50%--60%（按51%计算）；sfpatcher按-o-1算，创建的补丁为平均原apk大小的33%。   
那sfpatcher相比bsdiff或HDiffPatch方案单次继续节省 100x(51%-33%) = 18MB  (游戏apk平均节省会成倍数增加)  
假设apk商店每天有1亿次apk升级，按峰值带宽计费，假设单价每天0.48元/Mbps; 而按经验，峰值带宽一般是平均带宽的2倍：   
每月节省费用：100000000x(18x1024x1024x8/1000/1000)/(3600x24)x2x0.48*30 = 503.3万元   
如果按流量计费，假设单价0.11元/GB：   
每月节省费用：100000000x(18/1024)x0.11x30 = 580.1万元   
   
---
需要商业授权，请联系作者： <housisong@hotmail.com>   


