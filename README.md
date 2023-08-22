# sfpatcher：针对应用商店的apk增量算法
**v1.0.15 已正式上线**，为亿级手机终端用户提供更新服务，当前最新版本 v1.1.2   
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
 - **xdelta3** 在diff和patch场景下执行速度都很快，输出标准化的补丁格式(vcdiff格式,也有改成gdiff的)，apk补丁大小和BsDiff接近或略大，内存占用中等。该算法在diff处理较大文件的时候（比如几百MB以上），常会输出不正常的巨大补丁，除非使用和文件大小相当的参考内存来diff和patch，而这时patch时内存占用可控的优点就没有了。
 - **HDiffPatch** 是作者开源的算法，创建的补丁一般比BsDiff略小一些，diff(-s模式)和patch场景下执行速度都很快，内存占用可控并很小。 diff时也支持-m模式，用更大的内存和时间代价(这时也比BsDiff快很多倍)来得到更小一些的补丁包。 HDiffPatch 现在已经被**vivo**、**OPPO**、**小米**、**腾讯**、**华为**、**字节跳动**、**米哈游**等所使用；从为4KB RAM内存、8位CPU的MCU设备创建OTA增量补丁(4KB内存里一边解压一边patch!)，到为上百GB的游戏创建更新补丁，HDiffPatch 算法得到了越来越广泛的应用。
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
 - **ApkDiffPatch** 是作者开源的算法，为给自己团队开发的apk更新来开发的方案，创建的补丁平均比 archive-patcher 小很多，patch速度中等；方案一般用在可以自己对需要升级的apk进行重新签名的场景，而不能部署于应用商店。内部使用了 HDiffPatch 算法作为基础，用C\C++语言开发。为了patch时精确还原，和加快压缩还原时的速度，需要对发布的apk文件执行 ApkNormalized 预处理流程（使用了较快的压缩参数，处理过的apk需要重新签名）。 patch时可以支持并行压缩来加快apk还原速度，但这时内存资源占用比较严重，和多个线程正在分别压缩的多个文件源大小和其压缩后大小的总和相当。
## sfpatcher 是什么？
针对压缩档案文件的高性能增量更新方案。类似于 archive-patcher 方案可部署于应用商店的diff&patch算法，该领域的重要技术进展。   
内部使用了 HDiffPatch 算法作为基础，用C\C++语言开发，当前支持为deflate格式压缩的数据创建优化的补丁(用zlib压缩的支持最好)，支持在多种档案格式文件之间创建优化的补丁。     
### sfpatcher 之道
- 针对应用商店的场景专门设计，优化补丁大小，支持大型游戏，patch时精确快速还原任意apk文件，能够用于用户交互场景。
- 多级可选的补丁包大小，极致的patch速度：提供比谷歌**archive-patcher**方案(+lzma2压缩)下载补丁小24%的情况下，patch速度是其8倍(这时补丁比BsDiff方案小约55%并且速度快得多)！补丁比其小2%的情况下，速度是其30倍！ （注1）
- patch时的内存等资源占用可控；diff时支持多种方案限制patch时的最大内存占用到合适的约定水平。即支持patch时O(1)平均内存占用的优化补丁包(并且patch过程中不使用临时文件)！
- 优化的用户体验：可以支持下载数据的同时就开始patch，不用将补丁文件保存到内存或硬盘上，结合快速的patch，很多时候下载完成时，就可以得到了完整的新版本apk文件。 因为补丁变小，下载时间也会变短，从而可能缩短整体更新时间，也更省电省流量。
- 利用精确还原算法，对于初次下载的apk文件也能进行解压后的重压缩；从而节省用户初次下载apk时的流量。最多情况下平均比直接下载apk文件减少25%的数据，部分文件能减少35%以上的数据！
- 支持从中断的位置继续patch的特性，节省程序被终止后再次执行程序时的打补丁时间。
- 高扩展性，框架支持多种压缩档案格式(apk、zip、zip64、jar、gz、tar等)和其嵌套(如apk中多个子apk); 档案格式本身和档案中数据的压缩算法都只是以插件的形式获得支持。patch端的执行，设计上不依赖于具体的档案格式。
- patch端支持对客户端的oldApk文件进行虚拟化；比如可以用一些简单的描述数据来移除oldApk(v1--v4签名)中添加的各种类型渠道号的影响，提高了补丁适应能力。
- diff端参数可选择性丰富，对各种使用场景可以定制性的设置合适的控制参数。
- patch结果提供丰富的错误号，以利于追踪patch失败的原因，提高升级成功率。

注1：所有测试数据来源于收集的一些常用apk应用，共32个用例；并在ARM CPU Kirin980上测试了部分patch。（见性能测试对比数据）

## 方案主要特性对比
### **sfpatcher 和 archive-patcher**：
- sfpatcher 的patch端比 archive-patcher 快很多倍(相近补丁大小的情况下快29倍)，资源占用小(O(1))，能够满足各种使用场景的要求；而 archive-patcher 还原慢，资源占用大，一般用于后台更新场景。
- sfpatcher 生成的补丁大小在很多情况下也可以比 archive-patcher 的补丁（压缩后）更小。
- sfpatcher 创建压缩后的补丁，执行patch前不需要额外步骤解压缩补丁，边patch边随时解压缩补丁数据；而 archive-patcher 创建的补丁需要额外压缩和patch前解压。
- sfpatcher 在patch时不需要对oldApk提前进行解压，边patch边随时解压用到的oldApk中的数据，可以始终保持O(1)内存占用；而  archive-patcher 需要提前将oldApk中用到的数据全部解压缩到一个临时文件里。
- sfpatcher 支持大型apk文件，包括游戏等的更新升级，patch速度快；而 archive-patcher 有解压+未压缩总数据最大512M的限制，而且patch也很慢，某些情况下甚至patch需要几分钟。
- sfpatcher 基于C\C++，部署兼容性不受系统限制；结合JNI在安卓上使用，兼容安卓4.1到安卓13；而 archive-patcher 基于 java 并和运行时系统环境耦合重，diff和patch时都很可能会遇到环境兼容性问题。
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
收集了32组测试用例；这些apk下载于小米应用商店、谷歌Play商店，按照下载量大和最近进行过更新为标准进行收集。   
限于有限的用例和收集偏差，数据和实际情况可能略有差异。 旧版本apk平均大小103.2MB，新版本apk平均大小103.8MB。   

| 编号|app|新apk <-- 旧apk|新apk大小|旧apk大小|
|----:|:---:|:----|----:|----:|
|1|<img src="https://github.com/sisong/sfpatcher/raw/master/img/cn.wps.moffice_eng.png" width="36">|cn.wps.moffice_eng_13.30.0.apk <-- 13.29.0|95904918|94914262|
|2|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.achievo.vipshop.png" width="36">|com.achievo.vipshop_7.80.2.apk <-- 7.79.9|127395632|120237937|
|3|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.adobe.reader.png" width="36">|com.adobe.reader_22.9.0.24118.apk <-- 22.8.1.23587|27351437|27087718|
|4|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.alibaba.android.rimet.png" width="36">|com.alibaba.android.rimet_6.5.50.apk <-- 6.5.45|195314449|193489159|
|5|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.amazon.mShop.android.shopping.png" width="36">|com.amazon.mShop.android.shopping_24.18.2.apk <-- 24.18.0|76328858|76287423|
|6|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.baidu.BaiduMap.png" width="36">|com.baidu.BaiduMap_16.5.0.apk <-- 16.4.5|131382821|132308374|
|7|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.dragon.read.png" width="36">|com.dragon.read_5.5.3.33.apk <-- 5.5.1.32|45112658|43518658|
|8|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.ebay.mobile.png" width="36">|com.ebay.mobile_6.80.0.1.apk <-- 6.79.0.1|61202587|61123285|
|9|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.eg.android.AlipayGphone.png" width="36">|com.eg.android.AlipayGphone_10.3.0.apk <-- 10.2.96|122073135|119046208|
|10|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.google.android.apps.translate.png" width="36">|com.google.android.apps.translate_6.46.0.apk <-- 6.45.0|48892967|48843378|
|11|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.google.android.googlequicksearchbox.png" width="36">|com.google.android.googlequicksearchbox_13.38.11.apk <-- 13.37.10|190539272|189493966|
|12|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.jingdong.app.mall.png" width="36">|com.jingdong.app.mall_11.3.2.apk <-- 11.3.0|101098430|100750191|
|13|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.netease.cloudmusic.png" width="36">|com.netease.cloudmusic_8.8.45.apk <-- 8.8.40|181914846|181909451|
|14|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.reddit.frontpage.png" width="36">|com.reddit.frontpage_2022.36.0.apk <-- 2022.34.0|50205119|47854461|
|15|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.sankuai.meituan.takeoutnew.png" width="36">|com.sankuai.meituan.takeoutnew_7.94.3.apk <-- 7.92.2|74965893|74833926|
|16|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.sankuai.meituan.png" width="36">|com.sankuai.meituan_12.4.207.apk <-- 12.4.205|93613732|93605911|
|17|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.sina.weibo.png" width="36">|com.sina.weibo_12.10.0.apk <-- 12.9.5|156881776|156617913|
|18|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.smile.gifmaker.png" width="36">|com.smile.gifmaker_10.8.40.27845.apk <-- 10.8.30.27728|102403847|101520138|
|19|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.ss.android.article.news.png" width="36">|com.ss.android.article.news_9.0.7.apk <-- 9.0.6|54444003|53947221|
|20|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.ss.android.ugc.aweme.png" width="36">|com.ss.android.ugc.aweme_22.6.0.apk <-- 22.5.0|171683897|171353597|
|21|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.taobao.taobao.png" width="36">|com.taobao.taobao_10.18.10.apk <-- 10.17.0|117218670|117111874|
|22|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.tencent.mm.png" width="36">|com.tencent.mm_8.0.28.apk <-- 8.0.27|266691829|276603782|
|23|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.tencent.mobileqq.png" width="36">|com.tencent.mobileqq_8.9.15.apk <-- 8.9.13|311322716|310529631|
|24|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.tencent.mtt.png" width="36">|com.tencent.mtt_13.2.0.0103.apk <-- 13.2.0.0045|97342747|97296757|
|25|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.tripadvisor.tripadvisor.png" width="36">|com.tripadvisor.tripadvisor_49.5.apk <-- 49.3|28744498|28695346|
|26|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.twitter.android.png" width="36">|com.twitter.android_9.61.0.apk <-- 9.58.2|36141840|35575484|
|27|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.ubercab.png" width="36">|com.ubercab_4.442.10002.apk <-- 4.439.10002|69923232|64284150|
|28|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.ximalaya.ting.android.png" width="36">|com.ximalaya.ting.android_9.0.66.3.apk <-- 9.0.62.3|115804845|113564876|
|29|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.xunmeng.pinduoduo.png" width="36">|com.xunmeng.pinduoduo_6.30.0.apk <-- 6.29.1|30896833|30951567|
|30|<img src="https://github.com/sisong/sfpatcher/raw/master/img/com.youdao.dict.png" width="36">|com.youdao.dict_9.2.29.apk <-- 9.2.28|110624682|110628778|
|31|<img src="https://github.com/sisong/sfpatcher/raw/master/img/org.mozilla.firefox.png" width="36">|org.mozilla.firefox_105.2.0.apk <-- 105.1.0|83078464|83086656|
|32|<img src="https://github.com/sisong/sfpatcher/raw/master/img/tv.danmaku.bili.png" width="36">|tv.danmaku.bili_7.1.0.apk <-- 7.0.0|104774723|104727005|

# 测试条件
在一台笔记本PC上对比测试：Windows11, CPU R9-7945HX, SSD PCIe4.0x4 4T, DDR5 5200MHz 32Gx2   
测试时关闭了HDiffPatch和sfpatcher在diff时的多线程，而开启多线程时一般可以成倍的提高diff速度。   
patch时标注tmpf表示使用了临时文件来储存中间数据；mem表示在内存中执行不使用临时文件；limit表示使用限制内存占用的模式执行；而标注MT表示开启了多线程(8个)并行。   
**BsDiff** v4.3 还是保持着使用bzip2算法压缩补丁。   
**xdelta** v3.1.0 使用`-e -n -f -s`来创建补丁, 而用`-d -f -s`参数来执行的patch。   
**xdelta3 -B** 是指diff时添加`-B oldSize`增大引用窗口到oldApk文件相同大小进行的测试，其结果仅供参考，因为patch时内存占用也会随之同样增大。   
**HDiffPatch** v4.6.3 支持2种diff模式，`-s-16`和`-m-1 -cache -block`模式分别测试，输出补丁时分别测试了用lzma2、zstd压缩的测试。HDiffPatch支持输出兼容bsdiff的补丁(bzip2压缩)，补充了`-BSD -m-1 -cache -block`参数后的测试结果。   
**archive-patcher** v1.0 一般使用gzip或brotli算法压缩补丁，这里为了diff速度并更好的和其他方案对比补丁大小，diff时输出不压缩的补丁，然后再额外使用lzma2压缩补丁。 需要注意：这时收集到的diff数据不包含额外压缩时的时间和内存消耗，收集到的patch数据也**不包含**解压的时间和内存消耗等。   
**ApkDiffPatch** v1.6.0 使用了lzma2来压缩输出的补丁。   
**sfpatcher** v1.1.1 支持4个级别的diff，-0,-1,-2和-3分别测试； sfpatcher支持不需要旧版本apk而直接重新压缩新版本apk的模式(-pre)；sfpatcher支持多种可选压缩输出，这里测试了-c-zstd-21和-c-lzma2-9这2种。   
sfpatcher补充测试了用ApkNormalized(ApkDiffPatch方案)处理过的apk文件，分别进行增量测试和lzma2重压缩测试(标记为Norm)。   
   
另外在一部安卓手机(CPU:Kirin980 2×A76 2.6G + 2×A76 1.92G + 4×A55 1.8G)上对hpatchz&hsynz&sfpatcher进行了一些patch时间测试，补充到了最后一列。   

# 测试结果   
其中：平均压缩率=(补丁大小/新apk大小)的平均值； 单次测试的内存占用统计值为峰值私有内存；

|diff方案|平均压缩率|平均内存|平均速度|patch|平均内存|最大内存|平均速度|arm Kirin980|
|:----|----:|----:|----:|----|----:|----:|----:|----:|
|zstd --patch-from|53.18%|2199M|3.6MB/s|mem|209M|596M|609MB/s|
|**xdelta3**|**54.51%**|422MB|3.8MB/s|mem|**98MB**|**99MB**|**170MB/s**|
|xdelta3 -B|53.53%|848M|4.3MB/s|mem|183M|548M|171MB/s|
|**bsdiff**|**53.84%**|931MB|1.2MB/s|mem|**218MB**|**605MB**|**54MB/s**|
|hdiffz|54.40%|509M|8.8MB/s|mem|5M|6M|682MB/s|443MB/s|
|hdiffz lzma2|52.93%|525M|4.1MB/s|mem|21M|22M|260MB/s|131MB/s|
|**hdiffz zstd**|**53.04%**|537MB|5.4MB/s|mem|**21MB**|**22MB**|**598MB/s**|371MB/s|
|hdiffz -s zstd|53.44%|221M|10.1MB/s|mem|20M|22M|620MB/s|
|**archive-patcher**|**31.65%**|1448MB|0.9MB/s|temf|**558MB**|**587MB**|**14MB/s**|
|ApkDiffPatch|18.44%|1003M|2.2MB/s|mem|164M|453M|20MB/s|
|ApkDiffPatch|18.44%|1003M|2.2MB/s|memMT|257M|628M|64MB/s|
|ApkDiffPatch|18.44%|1003M|2.2MB/s|tmpf|21M|25M|17MB/s|
|ApkDiffPatch|18.44%|1003M|2.2MB/s|tmpfMT|101M|211M|46MB/s|
|sfpatcher-3 Norm|18.35%|1147M|2.2MB/s|limit|43M|59M|21MB/s|
|**sfpatcher-3 Norm**|**18.35%**|1147MB|2.2MB/s|limitMT|**49MB**|**65MB**|**77MB/s**|
|sfpatcher-3 Norm|18.35%|1147M|2.2MB/s|memMT|170M|452M|99MB/s|
|sfpatcher-3pre Norm|64.91%|610M|1.7MB/s|mem|37M|41M|14MB/s|
|**sfpatcher-3pre Norm**|**64.91%**|610MB|1.7MB/s|memMT|**42MB**|**46MB**|**64MB/s**|
||
|sfpatcher-0 zstd|53.04%|537M|5.4MB/s|mem|19M|20M|564MB/s|371MB/s|
|sfpatcher-0 zstd|53.05%|1251M|11.0MB/s|memMT|20M|22M|739MB/s|489MB/s|
|sfpatcher-0 lzma2|52.92%|525M|4.1MB/s|mem|18M|20M|253MB/s|131MB/s|
|sfpatcher-0 lzma2|52.94%|557M|18.8MB/s|memMT|19M|22M|346MB/s|157MB/s|
|sfpatcher-1 zstd|31.08%|818M|2.3MB/s|limit|15M|19M|201MB/s|92MB/s|
|**sfpatcher-1 zstd**|**31.07%**|1025MB|4.6MB/s|limitMT|**18MB**|**25MB**|**424MB/s**|189MB/s|
|sfpatcher-1 lzma2|29.75%|819M|2.3MB/s|limit|14M|19M|104MB/s|50MB/s|
|sfpatcher-1 lzma2|29.75%|809M|5.3MB/s|limitMT|17M|24M|167MB/s|74MB/s|
|sfpatcher-2 zstd|26.27%|975M|2.1MB/s|limit|15M|20M|43MB/s|
|sfpatcher-2 zstd|26.29%|1002M|4.7MB/s|limitMT|20M|27M|155MB/s|
|sfpatcher-2 lzma2|24.11%|976M|2.1MB/s|limit|15M|20M|37MB/s|19MB/s|
|**sfpatcher-2 lzma2**|**24.15%**|968MB|5.0MB/s|limitMT|**20MB**|**26MB**|**108MB/s**|45MB/s|
|sfpatcher-3 lzma2|23.53%|997M|2.0MB/s|limit|16M|20M|31MB/s|17MB/s|
|sfpatcher-3 lzma2|23.56%|987M|4.5MB/s|limitMT|21M|26M|98MB/s|40MB/s|
|sfpatcher-2pre zstd|81.36%|376M|2.8MB/s|mem|19M|23M|37MB/s|
|sfpatcher-2pre zstd|81.36%|1646M|7.1MB/s|memMT|24M|30M|193MB/s|
|sfpatcher-3pre zstd|79.20%|387M|2.4MB/s|mem|20M|23M|29MB/s|
|sfpatcher-3pre zstd|79.20%|1698M|6.5MB/s|memMT|25M|30M|144MB/s|
|sfpatcher-2pre lzma2|75.23%|378M|1.9MB/s|mem|20M|23M|24MB/s|12MB/s|
|**sfpatcher-2pre lzma2**|**75.42%**|1091MB|8.3MB/s|memMT|**24MB**|**28MB**|**63MB/s**|26MB/s|
|sfpatcher-3pre lzma2|73.34%|386M|1.7MB/s|mem|20M|23M|21MB/s|11MB/s|
|sfpatcher-3pre lzma2|73.53%|1126M|8.1MB/s|memMT|25M|29M|60MB/s|24MB/s|
   

# 游戏测试用例
最近收集了32组游戏测试用例，这些apk下载于小米应用商店、TapTap商店、谷歌Play商店，按照下载量大和最近进行过更新为标准进行收集。   
对比测试了 xdelta、bsdiff、HDiffPatch、sfpatcher； 因为有大游戏所以放弃了无法完成测试的archive-patcher。   
**sfpatcher** v1.1.1 测试时，调整了部分参数，相比前面的测试增大了patch时的内存需求。   
旧版本apk平均大小1029.5MB，新版本apk平均大小1040.9MB。   

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

|diff方案|平均压缩率|平均内存|平均速度|patch|平均内存|最大内存|平均速度|最差速度|
|:----|----:|----:|----:|----|----:|----:|----:|----:|
|zstd|36.35%|5297MB|3.2MB/s|mem|2088MB|4033MB|852MB/s|576MB/s|
|**xdelta3 lzma**|**54.14%**|423MB|3.5MB/s|mem|**99MB**|**104MB**|**94MB/s**|**30MB/s**|
|xdelta3 lzma +hpatchz -m|54.14%|423MB|3.5MB/s|mem|79MB|83MB|258MB/s|84MB/s|
|xdelta3 lzma -B|35.89%|6424MB|6.6MB/s|mem|1299MB|2090MB|233MB/s|76MB/s|
|xdelta3 lzma -B +hpatchz -m|35.89%|6424MB|6.6MB/s|mem|1043MB|2028MB|291MB/s|180MB/s|
|**bsdiff**|**36.15%**|9284MB|1.1MB/s|mem|**2085MB**|**4042MB**|**89MB/s**|**32MB/s**|
|bsdiff +hpatchz -m|36.15%|9284MB|1.1MB/s|mem|1045MB|2029MB|95MB/s|33MB/s|
|bsdiff +hpatchz -s|36.15%|9284MB|1.1MB/s|mem|14MB|14MB|87MB/s|32MB/s|
|hdiffz p1 zstd|35.41%|3812MB|6.3MB/s|mem|23MB|23MB|712MB/s|291MB/s|
|**hdiffz p8 zstd**|**35.42%**|4249MB|25.3MB/s|mem|**22MB**|**23MB**|**703MB/s**|**284MB/s**|
|hdiffz p1 lzma2|35.26%|3813MB|5.3MB/s|mem|22MB|23MB|344MB/s|136MB/s|
|hdiffz p8 lzma2|35.28%|3842MB|29.9MB/s|mem|22MB|23MB|344MB/s|135MB/s|
|sfpatcher-1 zstd|22.23%|4491MB|5.4MB/s|limit|23MB|35MB|412MB/s|152MB/s|
|**sfpatcher-1 zstd**|**22.20%**|4684MB|15.5MB/s|limitMT|**26MB**|**38MB**|**634MB/s**|**308MB/s**|
|sfpatcher-2 lzma2|19.61%|5162MB|4.9MB/s|limit|26MB|47MB|101MB/s|13MB/s|
|**sfpatcher-2 lzma2**|**19.62%**|5121MB|16.8MB/s|limitMT|**32MB**|**54MB**|**249MB/s**|**44MB/s**|
|sfpatcher-3 lzma2|17.87%|5682MB|4.6MB/s|limit|32MB|47MB|49MB/s|6MB/s|
|sfpatcher-3 lzma2|17.91%|5631MB|15.6MB/s|limitMT|39MB|54MB|157MB/s|26MB/s|
   

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


