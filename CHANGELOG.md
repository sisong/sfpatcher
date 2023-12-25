# sfpatcher Change Log

## [v1.2.0](https://github.com/sisong/sfpatcher/releases/tag/v1.2.0) - 2023-12-23
### Added
* added auto checksum newData/diffData/oldData; sf_patch speed -1%;   
  sf_patch return success only when the newData chencksum is also correct, otherwise an error code will be returned.
* sf_diff support java-SDK;    
  previously, the server needed to invoke the sf_diff cmdline to create diffFile; now it is possible to call sf_diff functions directly with Java code (by jni).

## [v1.1.3](https://github.com/sisong/sfpatcher/releases/tag/v1.1.3) - 2023-08-28
### Added
* hpatcher.patch() in sf_patch Android SDK added support diffFile created by $bsdiff4,$xdelta3 -S,$xdelta3 -S lzma,$open-vcdiff;   
  NOTE: need open _IS_NEED_BSDIFF _IS_NEED_VCDIFF when NDK build Android .so
### Fixed
* Reopen the LZMA asm optimize for arm64 (after upgrading LZMA lib from v1.1.2);

## [v1.1.2](https://github.com/sisong/sfpatcher/releases/tag/v1.1.2) - 2023-08-21
### Added
* sf_patch Android SDK support pass IReadStream when call API (previously supported pass fileName);    
 NOTE: read oldApk's file data by java code, it will affect the patch speed slightly; recommended for such as debug scenarios.

## [v1.1.1](https://github.com/sisong/sfpatcher/releases/tag/v1.1.1) - 2023-06-09
### Fixed
* fix a crash when sf_diff resave diffFile;
### Changed
* optimize sf_patch Android .so file size -13%;

## [v1.1.0](https://github.com/sisong/sfpatcher/releases/tag/v1.1.0) - 2022-11-01
### Changed
* sf_diff support muti-thread parallel, speed double;
* optimize sf_patch speed +6%;

## [v1.0.16](https://github.com/sisong/sfpatcher/releases/tag/v1.0.16) - 2022-09-12
### Changed
* more detailed error messages & added new errorCode;
* can got system errno when call system file API fail;
* can got errorCode when decompressor fail;

## [v1.0.15](https://github.com/sisong/sfpatcher/releases/tag/v1.0.15) - 2021-12-06
### released and online version
* serving hundreds of millions of end users;
