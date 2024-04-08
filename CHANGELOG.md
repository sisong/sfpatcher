# sfpatcher Change Log

## [v1.3.0](https://github.com/sisong/sfpatcher/releases/tag/v1.3.0) - 2024-01-29
### 添加
* 为 sfpatcher 方案添加了定制的apk文件标准化流程，在保持还不错的patch合成速度(比-o-1慢32%)的情况下进一步减小补丁包的大小(比-o-1小30%);   
* 添加 sf_normalize 命令行用于标准化apk文件（注意：标准后的apk文件需要重新签名）;
* sf_diff 在 -o-1 及以上级别时 添加 -e-ldefA 选项，以打开支持经过 sf_normalize 标准化处理过的apk文件，降低补丁包大小;
* sf_diff 使用 -pre -e-ldefA 选项支持重新压缩 经过 sf_normalize 标准化处理过的apk文件，降低其完整安装包下载时的大小(可以比原始apk小25%)。
### 修复
* 修复 sf_diff 支持另存带校验信息的 diffFile 补丁文件并生成新的补丁文件校验信息。

## [v1.2.0](https://github.com/sisong/sfpatcher/releases/tag/v1.2.0) - 2023-12-23
### 添加
* 添加自动校验 newData、diffData、oldData; sf_patch 合成速度平均约 -2% 以内，校验强度接近MD5;   
  只有当 newData 校验也正确时，sf_patch 才会返回成功，否则必将返回某个错误码;
* sf_diff 支持 java-SDK;   
  以前，服务器需要调用 sf_diff 命令行来创建补丁文件; 现在可以直接使用 Java 代码（通过 jni）调用 sf_diff 函数。

## [v1.1.3](https://github.com/sisong/sfpatcher/releases/tag/v1.1.3) - 2023-08-28
### 添加
* sf_patch 安卓 SDK 中的 hpatcher.patch() 添加支持 $bsdiff4、$xdelta3 -S、$xdelta3 -S lzma、$open-vcdiff 创建的补丁文件;  
  注意：该功能移动端默认关闭，需要在用 NDK 编译安卓 .so 库时打开定义 _IS_NEED_BSDIFF 、_IS_NEED_VCDIFF;
### 修复
* 重新打开了 LZMA arm64 汇编优化（从 v1.1.2 升级 LZMA 第三方库后）。

## [v1.1.2](https://github.com/sisong/sfpatcher/releases/tag/v1.1.2) - 2023-08-21
### 添加
* sf_patch 安卓 SDK 支持在调用 API 时传递流接口 IReadStream（之前支持传递文件名）;   
 注意：通过java代码读取oldApk的文件数据，会略微影响patch合成速度; 建议用于调试等场景。

## [v1.1.1](https://github.com/sisong/sfpatcher/releases/tag/v1.1.1) - 2023-06-09
### 修复
* 修复 sf_diff 另存 diffFile 补丁文件时可能崩溃的问题;
### 改变
* 优化 sf_patch 安卓库 .so 文件大小 -13%。

## [v1.1.0](https://github.com/sisong/sfpatcher/releases/tag/v1.1.0) - 2022-11-01
### 改变
* sf_diff 支持多线程并行，速度加倍;
* 优化 sf_patch 速度约 +6%。

## [v1.0.16](https://github.com/sisong/sfpatcher/releases/tag/v1.0.16) - 2022-09-12
### 改变
* 更详细的错误信息，并添加了新的错误代码;
* 调用系统文件 API 失败时，可以得到系统返回的 errno 错误码;
* 解压缩器失败时可以得到错误码。

## [v1.0.15](https://github.com/sisong/sfpatcher/releases/tag/v1.0.15) - 2021-12-06
### 发布上线版本
* 为数以亿计的终端用户提供服务。
