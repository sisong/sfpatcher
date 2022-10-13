# sfpatcher：针对应用商店的apk增量算法
**v1.0.15已正式上线**，为亿级用户提供服务；当前最新版本 v1.0.16   
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
收集了32组测试用例，这些用例来源于一些较长时间收集到的常见应用和游戏。   
限于有限的用例和收集偏差，数据和实际情况可能略有差异；因为bsdiff内存占用过大和archive-patcher 512M大小的限制放弃了大游戏用例。   
旧版本apk平均大小114.7MB，新版本apk平均大小117.1MB   

| 编号|app|新apk <-- 旧apk|新apk大小|旧apk大小|
|----:|:---:|:----|----:|----:|
|1|<img src="img/com.MobileTicket.png" width="32">|12306_5.2.11.apk <-- 12306_5.1.2.apk| 61120025|66209244|
|2|<img src="img/com.eg.android.AlipayGphone.png" width="32">|alipay10.1.99.apk <-- alipay10.1.95.apk|94178674|90951351|
|3|<img src="img/com.eg.android.AlipayGphone.png" width="32">|alipay10.2.0.apk <-- alipay10.1.99.apk|95803005|94178674|
|4|<img src="img/com.baidu.BaiduMap.png" width="32">|baidumaps10.25.0.apk <-- baidumaps10.24.12.apk|95539893|104527191|
|5|<img src="img/com.baidu.BaiduMap.png" width="32">|baidumaps10.25.5.apk <-- baidumaps10.25.0.apk|95526276|95539893|
|6|<img src="img/tv.danmaku.bili.png" width="32">|bilibili6.15.0.apk <-- bilibili6.14.0.apk|74783182|72067209|
|7|<img src="img/com.android.chrome.png" width="32">|chrome-64-0-3282-137.apk <-- chrome-64-0-3282-123.apk|43879588|43879588|
|8|<img src="img/com.android.chrome.png" width="32">|chrome-65-0-3325-109.apk <-- chrome-64-0-3282-137.apk|43592997|43879588|
|9|<img src="img/com.sdu.didi.psnger.png" width="32">|didi6.0.2.apk <-- didi6.0.0.apk|100866981|91462767|
|10|<img src="img/org.mozilla.firefox.png" width="32">|firefox68.10.0.apk <-- firefox68.9.0.apk|43543846|43531470|
|11|<img src="img/org.mozilla.firefox.png" width="32">|firefox68.10.1.apk <-- firefox68.10.0.apk|43542786|43543846|
|12|<img src="img/com.google.android.apps.maps.png" width="32">|google-maps-9-71-0.apk <-- google-maps-9-70-0.apk|50568872|51304768|
|13|<img src="img/com.google.android.apps.maps.png" width="32">|google-maps-9-72-0.apk <-- google-maps-9-71-0.apk|54342938|50568872|
|14|<img src="img/com.jingdong.app.mall.png" width="32">|jd9.0.0.apk <-- jd8.5.12.apk|96891703|94233891|
|15|<img src="img/com.jingdong.app.mall.png" width="32">|jd9.0.8.apk <-- jd9.0.0.apk|97329322|96891703|
|16|<img src="img/com.tencent.tmgp.jnbg2.png" width="32">|jinianbeigu2_1.12.4.apk <-- jinianbeigu2_1.12.3.apk|171611658|159691189|
|17|<img src="img/com.blizzard.wtcg.hearthstone.cn.png" width="32">|lushichuanshuo19.4.71003.apk <-- lushichuanshuo19.2.69054.apk|93799693|93442621|
|18|<img src="img/com.sankuai.meituan.png" width="32">|meituan10.9.401.apk <-- meituan10.9.203.apk|88956726|89384406|
|19|<img src="img/com.netease.mc.png" width="32">|minecraft1.17.30.apk <-- minecraft1.17.20.apk|373025314|370324338|
|20|<img src="img/com.netease.mc.png" width="32">|minecraft1.18.10.apk <-- minecraft1.17.30.apk|401075178|373025314|
|21|<img src="img/com.popcap.pvz2cthd.png" width="32">|popcap.pvz2_2.4.84.1010.apk <-- popcap.pvz2_2.4.84.1009.apk|387572492|386842079|
|22|<img src="img/com.supercell.clashofclans.png" width="32">|supercell.clashofclans13.369.3.apk <-- supercell.clashofclans13.180.18.apk|152896934|149011539|
|23|<img src="img/com.outfit7.talkingtomgoldrun.png" width="32">|tangmumaopaoku4.8.0.971.apk <-- tangmumaopaoku4.6.0.913.apk|105486308|104732413|
|24|<img src="img/com.taobao.taobao.png" width="32">|taobao9.8.0.apk <-- taobao9.7.2.apk|178734456|176964070|
|25|<img src="img/com.taobao.taobao.png" width="32">|taobao9.9.1.apk <-- taobao9.8.0.apk|184437315|178734456|
|26|<img src="img/com.ss.android.ugc.aweme.png" width="32">|tiktok11.5.0.apk <-- tiktok11.3.0.apk|88544106|87075000|
|27|<img src="img/com.google.android.apps.translate.png" width="32">|translate6.9.0.apk <-- translate6.8.0.apk|28171978|28795243|
|28|<img src="img/com.google.android.apps.translate.png" width="32">|translate6.9.1.apk <-- translate6.9.0.apk|31290990|28171978|
|29|<img src="img/com.tencent.mm.png" width="32">|weixin7.0.15.apk <-- weixin7.0.14.apk|148405483|147695111|
|30|<img src="img/com.tencent.mm.png" width="32">|weixin7.0.16.apk <-- weixin7.0.15.apk|158906413|148405483|
|31|<img src="img/cn.wps.moffice_eng.png" width="32">|wps12.5.2.apk <-- wps12.5.1.apk|51293286|51136905|
|32|<img src="img/com.tanwan.yscqlyzf.png" width="32">|yuanshichuanqi1.3.608.apk <-- yuanshichuanqi1.3.607.apk|192578139|192577253|

# 测试条件
在一台笔记本PC上对比测试，CPU Ryzen 5800H，Windows11, SSD硬盘   
测试时关闭了HDiffPatch和sfpatcher在diff时的多线程。   
patch时标注tmpf表示使用了临时文件来储存中间数据；mem表示在内存中执行不使用临时文件；limit表示使用限制内存占用的模式执行；而标注MT表示开启了多线程(8个)并行。   
**BsDiff** v4.3 还是保持着使用bzip2算法压缩补丁。   
**xdelta** v3.1.0 使用`-e -n -f -s`来创建补丁, 而用`-d -f -s`参数来执行的patch。   
**HDiffPatch** v4.2.4 支持2种diff模式，`-s-16`和`-m-1 -cache -block`模式分别测试，输出补丁时分别测试了用lzma2、zstd压缩和不压缩的测试。HDiffPatch支持输出兼容bsdiff的补丁(bzip2压缩)，补充了`-BSD -m-1 -cache -block`参数后的测试结果。   
**archive-patcher** v1.0 一般使用gzip或brotli算法压缩补丁，这里为了diff速度并更好的和其他方案对比补丁大小，diff时输出不压缩的补丁，然后再额外使用lzma2压缩补丁。 需要注意：这时收集到的diff数据不包含额外压缩时的时间和内存消耗，收集到的patch数据也**不包含**解压的时间和内存消耗等。   
**ApkDiffPatch** v1.3.6 使用了lzma来压缩输出的补丁。   
**sfpatcher** v1.0.16 支持4个级别的diff，-0,-1,-2和-3分别测试； sfpatcher支持不需要旧版本apk而直接重新压缩新版本apk的模式(-pre)；sfpatcher支持多种可选压缩输出，这里测试了-c-zstd-21和-c-lzma2-9这2种。   
sfpatcher补充测试了用ApkNormalized(ApkDiffPatch方案)处理过的apk文件，分别进行增量测试和重压缩测试(标记为Norm)。   
   
另外在一部安卓手机(CPU:Kirin980)上对sfpatcher进行了一些patch时间测试，补充到了最后一列。   

# 测试结果   
其中：平均压缩率=(补丁大小/新apk大小)的平均值； 单次测试的内存统计值为峰值内存；

|diff方案|平均压缩率|平均内存|平均速度|patch|平均内存|最大内存|平均速度|Kirin980速度|
|:----|----:|----:|----:|----|----:|----:|----:|----:|
|**xdelta3 lzma**|**59.9%**|**228MB**|**2.9MB/s**|mem|**100MB**|**100MB**|**159MB/s**|
|**bsdiff bzip2**|**59.8%**|**1035MB**|**1.0MB/s**|mem|**243MB**|**751MB**|**42MB/s**|
|hdiffz-m-1 -BSD|59.5%|523MB|5.4MB/s|mem|13MB|14MB|44MB/s|
|hdiffz-m-1|59.9%|523MB|7.5MB/s|mem|4MB|5MB|780MB/s|268MB/s|
|**hdiffz-m-1 zstd**|**58.7%**|**612MB**|**5.0MB/s**|mem|**13MB**|**14MB**|**680MB/s**|**265MB/s**|
|hdiffz-m-1 lzma2|58.7%|523MB|3.7MB/s|mem|12MB|13MB|285MB/s|
|hdiffz-s-16|60.5%|133MB|31.8MB/s|mem|3MB|4MB|806MB/s|
|hdiffz-s-16 zstd|59.3%|136MB|9.7MB/s|mem|12MB|12MB|763MB/s|
|**archive-patcher**|28.5%|**1740MB**|**0.8MB/s**|tmpf|**64MB**|**100MB**|**15MB/s**|
|ApkDiffPatch|20.5%|982MB|2.0MB/s|mem|138MB|386MB|21MB/s|
|ApkDiffPatch|20.5%|982MB|2.0MB/s|memMT|211MB|461MB|47MB/s|
|ApkDiffPatch|20.5%|982MB|2.0MB/s|tmpf|17MB|22MB|19MB/s|
|ApkDiffPatch|20.5%|982MB|2.0MB/s|tmpfMT|84MB|207MB|41MB/s|
|sfpatcher-3 lzma2 Norm|20.9%|1032MB|2.4MB/s|limit|47MB|57MB|24MB/s|
|**sfpatcher-3 lzma2 Norm**|**20.9%**|1032MB|2.4MB/s|limitMT|**52MB**|**63MB**|**78MB/s**|
|sfpatcher-3 lzma2 Norm|20.9%|1032MB|2.4MB/s|memMT|142MB|397MB|95MB/s|
|sfpatcher-3pre lzma2 Norm|73.6%|601MB|1.6MB/s|mem|38MB|41MB|16MB/s|
|**sfpatcher-3pre lzma2 Norm**|**73.6%**|601MB|1.6MB/s|memMT|**43MB**|**47MB**|**55MB/s**|
||
|sfpatcher-0 zstd|58.7%|612MB|4.9MB/s|mem|13MB|14MB|716MB/s|265MB/s|
|sfpatcher-0 zstd|58.7%|612MB|4.9MB/s|memMT|14MB|15MB|890MB/s|293MB/s|
|sfpatcher-0 lzma2|58.7%|523MB|3.7MB/s|mem|12MB|13MB|286MB/s|160MB/s|
|sfpatcher-0 lzma2|58.7%|523MB|3.7MB/s|memMT|13MB|15MB|342MB/s|172MB/s|
|sfpatcher-1 zstd|31.7%|774MB|2.8MB/s|limit|16MB|20MB|227MB/s|119MB/s|
|**sfpatcher-1 zstd**|**31.7%**|**774MB**|**2.8MB/s**|limitMT|**19MB**|**22MB**|**394MB/s**|**218MB/s**|
|sfpatcher-1 lzma2|30.8%|725MB|2.6MB/s|limit|15MB|19MB|116MB/s|65MB/s|
|sfpatcher-1 lzma2|30.8%|725MB|2.6MB/s|limitMT|18MB|21MB|170MB/s|96MB/s|
|sfpatcher-2 zstd|28.7%|890MB|2.6MB/s|limit|17MB|24MB|48MB/s|32MB/s|
|**sfpatcher-2 zstd**|**28.7%**|890MB|2.6MB/s|limitMT|**21MB**|**30MB**|**157MB/s**|**85MB/s**|
|sfpatcher-2 lzma2|27.5%|859MB|2.5MB/s|limit|16MB|24MB|41MB/s|26MB/s|
|**sfpatcher-2 lzma2**|**27.5%**|859MB|2.5MB/s|limitMT|**21MB**|**29MB**|**107MB/s**|**59MB/s**|
|sfpatcher-3 zstd|25.1%|995MB|2.3MB/s|limit|19MB|24MB|21MB/s|14MB/s|
|sfpatcher-3 zstd|25.1%|995MB|2.3MB/s|limitMT|24MB|30MB|80MB/s|42MB/s|
|sfpatcher-3 lzma2|23.7%|976MB|2.3MB/s|limit|19MB|24MB|20MB/s|13MB/s|
|**sfpatcher-3 lzma2**|**23.7%**|976MB|2.3MB/s|limitMT|**24MB**|**29MB**|**66MB/s**|**36MB/s**|
||
|sfpatcher-2pre zstd|87.6%|517MB|2.4MB/s|mem|22MB|26MB|38MB/s|25MB/s|
|**sfpatcher-2pre zstd**|**87.6%**|517MB|2.4MB/s|memMT|**26MB**|**33MB**|**174MB/s**|**80MB/s**|
|sfpatcher-2pre lzma2|82.8%|380MB|1.8MB/s|mem|22MB|25MB|24MB/s|15MB/s|
|**sfpatcher-2pre lzma2**|**82.8%**|380MB|1.8MB/s|memMT|**25MB**|**31MB**|**55MB/s**|**30MB/s**|
|sfpatcher-3pre zstd|83.2%|545MB|1.8MB/s|mem|22MB|26MB|18MB/s|12MB/s|
|sfpatcher-3pre zstd|83.2%|545MB|1.8MB/s|memMT|28MB|33MB|80MB/s|39MB/s|
|sfpatcher-3pre lzma2|77.9%|402MB|1.6MB/s|mem|22MB|25MB|14MB/s|9MB/s|
|**sfpatcher-3pre lzma2**|**77.9%**|402MB|1.6MB/s|memMT|**27MB**|**31MB**|**45MB/s**|**24MB/s**|
   

# 游戏测试用例
最近收集了32组游戏测试用例，这些apk下载于小米应用商店、TapTap商店、谷歌Play商店，按照下载量大和最近进行过更新为标准进行收集。   
对比测试了xdelta、sfpatcher； 因为有大游戏所以放弃了无法顺利完成测试的bsdiff和archive-patcher。   
旧版本apk平均大小1010.3MB，新版本apk平均大小1017.5MB   

| 编号|app|新apk <-- 旧apk|新apk大小|旧apk大小|
|----:|:---:|:----|----:|----:|
|1|<img src="gimg/com.blizzard.wtcg.hearthstone.png" width="32">|com.blizzard.wtcg.hearthstone_24.4.150659.apk <-- 24.2.148211.apk|119192491|118798993|
|2|<img src="gimg/com.ea.simcitymobile.mi.png" width="32">|com.ea.simcitymobile.mi_0.68.21338.22253.apk <-- 0.67.21338.22186.apk|262485392|256687957|
|3|<img src="gimg/com.happyelements.AndroidAnimal.png" width="32">|com.happyelements.AndroidAnimal_1.114.apk <-- 1.113.apk|246916191|276046501|
|4|<img src="gimg/com.hermes.h1game.mi.png" width="32">|com.hermes.h1game.mi_1.10.1.apk <-- 1.9.1.apk|1764369655|1742356334|
|5|<img src="gimg/com.imangi.templerun2.png" width="32">|com.imangi.templerun2_6.5.1.apk <-- 6.5.0.apk|142692127|141981473|
|6|<img src="gimg/com.kiloo.subwaysurf.png" width="32">|com.kiloo.subwaysurf_3.36.1.apk <-- 3.36.0.apk|172657047|172348585|
|7|<img src="gimg/com.king.candycrushsaga.png" width="32">|com.king.candycrushsaga_1.237.0.3.apk <-- 1.236.0.3.apk|89619604|89410785|
|8|<img src="gimg/com.knight.union.mi.png" width="32">|com.knight.union.mi_4.3.1.apk <-- 4.3.0.apk|281337183|279734645|
|9|<img src="gimg/com.kurogame.haru.hero.png" width="32">|com.kurogame.haru.hero_1.32.0.apk <-- 1.31.2.apk|2076982449|2071240294|
|10|<img src="gimg/com.miHoYo.enterprise.NGHSoD.png" width="32">|com.miHoYo.enterprise.NGHSoD_6.1.0.apk <-- 6.0.0.apk|633463833|617956855|
|11|<img src="gimg/com.miHoYo.Yuanshen.png" width="32">|com.miHoYo.Yuanshen_3.1.0.apk <-- 3.0.0.apk|292959792|281028246|
|12|<img src="gimg/com.miniclip.eightballpool.png" width="32">|com.miniclip.eightballpool_5.10.2.apk <-- 5.10.0.apk|68794783|69208479|
|13|<img src="gimg/com.minitech.miniworld.TMobile.mi.png" width="32">|com.minitech.miniworld.TMobile.mi_1.19.0.apk <-- 1.18.1.apk|615403134|606383668|
|14|<img src="gimg/com.netease.dwrg.mi.png" width="32">|com.netease.dwrg.mi_1.5.69.apk <-- 1.5.67.apk|2087354104|2083304114|
|15|<img src="gimg/com.netease.g104.mi.png" width="32">|com.netease.g104.mi_3.18.1.apk <-- 3.17.5.apk|1900704615|1853407883|
|16|<img src="gimg/com.netease.g67.mi.png" width="32">|com.netease.g67.mi_1.6.0.apk <-- 1.5.4.apk|2021330101|2016955181|
|17|<img src="gimg/com.netease.hyxd.mi.png" width="32">|com.netease.hyxd.mi_1.283.479407.apk <-- 1.282.479407.apk|2099155328|2107908315|
|18|<img src="gimg/com.netease.mc.mi.png" width="32">|com.netease.mc.mi_2.3.15.apk <-- 2.3.5.apk|986369238|982878980|
|19|<img src="gimg/com.netease.stzb.mi.png" width="32">|com.netease.stzb.mi_5.1.1.apk <-- 4.4.8.apk|2035866423|2114161454|
|20|<img src="gimg/com.nianticlabs.pokemongo.png" width="32">|com.nianticlabs.pokemongo_0.251.0.apk <-- 0.249.2.apk|51919509|51538569|
|21|<img src="gimg/com.pandadastudio.ninjamustdie3.mi.png" width="32">|com.pandadastudio.ninjamustdie3.mi_2.0.15.apk <-- 2.0.5.apk|1494616373|1486154585|
|22|<img src="gimg/com.popcap.pvz2cthdxm.png" width="32">|com.popcap.pvz2cthdxm_2.9.6.apk <-- 2.9.4.apk|942247868|916279479|
|23|<img src="gimg/com.sy.dldlhsdj.mi.png" width="32">|com.sy.dldlhsdj.mi_2.8.4.apk <-- 2.8.1.apk|1748742033|1658132570|
|24|<img src="gimg/com.tencent.ig.png" width="32">|com.tencent.ig_2.2.0.apk <-- 2.1.0.apk|1123848712|1161724820|
|25|<img src="gimg/com.tencent.lolm.png" width="32">|com.tencent.lolm_3.4.0.apk <-- 3.3.0.apk|1956378264|1808791425|
|26|<img src="gimg/com.tencent.mf.uam.png" width="32">|com.tencent.mf.uam_1.0.128.apk <-- 1.0.118.apk|1934202996|2050880984|
|27|<img src="gimg/com.tencent.tmgp.cf.png" width="32">|com.tencent.tmgp.cf_1.0.280.apk <-- 1.0.260.apk|2089132074|1998168460|
|28|<img src="gimg/com.tencent.tmgp.sgame.png" width="32">|com.tencent.tmgp.sgame_3.81.1.8.apk <-- 3.74.1.6.apk|1960032646|1976668388|
|29|<img src="gimg/com.tencent.tmgp.speedmobile.png" width="32">|com.tencent.tmgp.speedmobile_1.33.0.apk <-- 1.32.0.apk|2122819058|2096029603|
|30|<img src="gimg/com.ustwo.monumentvalleyzz.png" width="32">|com.ustwo.monumentvalleyzz_2.5.2.apk <-- 2.5.1.apk|159780651|160263061|
|31|<img src="gimg/com.wepie.snake.new.mi.png" width="32">|com.wepie.snake.new.mi_5.3.2.apk <-- 5.3.0.2.apk|388251536|382250707|
|32|<img src="gimg/com.youzu.bs.png" width="32">|com.youzu.bs_44.270.apk <-- 44.265.apk|272298484|270552420|
   

# 游戏测试结果   
其中：总和压缩率=补丁总大小/新apk总大小； 单次测试的内存统计值改为峰值私有内存；   
 **xdelta3 -B**是指增大引用窗口到oldApk文件相同大小进行的测试，其结果仅供参考，因为patch时内存占用也会随之增大。   

|diff方案|总和压缩率|平均压缩率|平均内存|平均速度|patch|平均内存|最大内存|平均速度|最差速度|
|:----|----:|----:|----:|----:|----|----:|----:|----:|----:|
|**xdelta3 lzma**|**54.60%**|**47.64%**|423MB|2.0MB/s|mem|**99MB**|**103MB**|**37MB/s**|**11MB/s**|
|xdelta3 djw|55.67%|48.34%|413MB|3.3MB/s|mem|98MB|102MB|38MB/s|15MB/s|
|xdelta3 lzma -B |27.35%|31.85%|6184MB|3.5MB/s|mem|1251MB|2090MB|76MB/s|14MB/s|
|xdelta3 djw -B |27.73%|32.16%|6174MB|5.1MB/s|mem|1249MB|2089MB|77MB/s|20MB/s|
|sf_diff -0|28.02%|32.57%|3996MB|5.6MB/s|mem|2MB|3MB|383MB/s|141MB/s|
|sf_diff -0 -lzma2|23.99%|29.81%|3997MB|3.7MB/s|mem|10MB|11MB|186MB/s|62MB/s|
|sf_diff -0 -zstd|24.28%|30.02%|4033MB|4.3MB/s|mem|11MB|11MB|340MB/s|128MB/s|
|sf_diff -1 -zstd|21.38%|22.76%|4422MB|3.7MB/s|limit|14MB|19MB|287MB/s|116MB/s|
|**sf_diff -1 -zstd**|**21.38%**|**22.76%**|4422MB|3.7MB/s|limitMT|**17MB**|**22MB**|**410MB/s**|**144MB/s**|
|sf_diff -2 -lzma2|20.55%|20.69%|4731MB|3.2MB/s|limit|15MB|20MB|112MB/s|11MB/s|
|sf_diff -2 -lzma2|20.55%|20.69%|4731MB|3.2MB/s|limitMT|20MB|26MB|208MB/s|37MB/s|
|sf_diff -3 -lzma2|19.95%|19.71%|5216MB|2.7MB/s|limit|17MB|25MB|49MB/s|6MB/s|
|sf_diff -3 -lzma2|19.95%|19.71%|5216MB|2.7MB/s|limitMT|23MB|32MB|132MB/s|31MB/s|
   

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
单个apk一次升级节省的流量估算：现在用户安卓手机经常使用的应用apk一般都越来越大，而经常使用的游戏平均应该更大，假设按平均100MB算。   
一般bsdiff或HDiffPatch创建的补丁平均为原apk大小的50%--60%（按51%计算）；sfpatcher按-o-1算，创建的补丁为平均原apk大小的33%。   
那sfpatcher相比bsdiff或HDiffPatch方案单次继续节省 100x(51%-33%) = 18MB  (游戏apk平均节省会成倍数增多)  
假设apk商店每天有1亿次apk升级，按峰值带宽计费，假设单价每天0.48元/Mbps; 而按经验，峰值带宽一般是平均带宽的2倍：   
每月节省费用：100000000x(18x1024x1024x8/1000/1000)/(3600x24)x2x0.48*30 = 503.3万元   
如果按流量计费，假设单价0.11元/GB：   
每月节省费用：100000000x(18/1024)x0.11x30 = 580.1万元   
   
---
需要商业授权，请联系作者： <housisong@hotmail.com>   


