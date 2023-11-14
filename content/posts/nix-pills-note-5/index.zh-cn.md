---
title: 'nix-pills-note-5'
tag: ["nix-pills-note"]
series: ["nix-pills-note"]

date: 2023-11-14T13:04:12+08:00
---



## 第5篇
## automatic runtime
- 分析构建时依赖和运行时依赖
  `build dependency`
  `nix-instantiate hellov2.nix`
      ```/nix/store/3g7ccvlxrpmb2s9m4vkasz8y6444fwsc-hello.drv```
  
  `nix-store -q --references /nix/store/3g7ccvlxrpmb2s9m4vkasz8y6444fwsc-hello.drv`
      ```
        /nix/store/3077wznnjrbd318k17j7fjsgxy9xgrcl-gnutar-1.35.drv
        /nix/store/3mdqlcjhflm8mdm2192bap27g8zv60sq-generic-builder.sh
        /nix/store/982f8csn028v499k0fkrl66x28cbj3b8-bash-5.2-p15.drv
        /nix/store/dxsdysn7gpyjajiqjvab03bilarjjn15-coreutils-9.3.drv
        /nix/store/f7glbcn7n59k22b911bx1vyy13g4bdxh-binutils-2.40.drv
        /nix/store/fimilhby9fyqbfwmw826id3hwfhya6qx-hello-2.12.1.tar.gz
        /nix/store/fis965lcaxlhxsibjqr7g99l074niraj-gnumake-4.4.1.drv
        /nix/store/jhkn3namnqqkl7hmxmcml0ffir0dv774-gnugrep-3.11.drv
        /nix/store/n20wzc7mdmdq91vq5b6gklsz1171k82r-gnused-4.9.drv
        /nix/store/w1nk39ahhnzkj9yqa5vpggxgff11056g-gawk-5.2.2.drv
        /nix/store/w3f8imf2csiwc7jfinmgf682rvhg7fna-gcc-wrapper-12.3.0.drv
        /nix/store/zq7wlgpchcp0857swj8jv5hdwk0pzlg7-gzip-1.12.drv
      ```


  `runtime dependency`
  **derivation的运行时依赖是由Nix自动识别的**
  1. 把derivation的out path序列化为NAR
  2. 对derivation的每个构建时依赖及其对应的out path，在NAR中寻找该out path的hash部分
  3. 如果找到了，说明这个out path是该derivation的运行时依赖

  通过`nix-store -r /nix/store/3g7ccvlxrpmb2s9m4vkasz8y6444fwsc-hello.drv`
  得到`out path` `/nix/store/x3skf9wy7hpigvp8dca7kg1xafwhy5gv-hello`
  用`nix-store -q --references /nix/store/x3skf9wy7hpigvp8dca7kg1xafwhy5gv-hello`
  得到运行时依赖
  ```
    /nix/store/vq3sdi8l15rzfl5zvmwpafrzis4sm6xf-glibc-2.37-8
    /nix/store/c50v7bf341jsza0n07784yvzp5fzjpn5-gcc-12.3.0-lib
    /nix/store/kmp7p16z94gx9nm80n03j8cp9rllmla2-glibc-2.37-8-dev
    /nix/store/nd4dhnbh1wagnxh2fzd3cqb8ifzl2r43-gcc-12.3.0
    /nix/store/x3skf9wy7hpigvp8dca7kg1xafwhy5gv-hello
  ```
  嗯..glibc，gcc.. !? gcc, 我编译好的hello程序为啥还依赖gcc啊？
  `strings result/bin/hello | grep gcc`
  ```
  /nix/store/vq3sdi8l15rzfl5zvmwpafrzis4sm6xf-glibc-2.37-8/lib:/nix/store/c50v7bf341jsza0n07784yvzp5fzjpn5-gcc-12.3.0-lib/lib
  /nix/store/nd4dhnbh1wagnxh2fzd3cqb8ifzl2r43-gcc-12.3.0/lib/gcc/x86_64-unknown-linux-gnu/12.3.0/include
  ```
  还真依赖gcc 
  第一个是`ldrpath`，是用来在运行时寻找库文件的地方(对nix来说这是必须的，因为它不存在全局，不硬编码的话程序不知道位置)
  但是我们并不需要这个这个运行时库。可以通过[patchelf](https://github.com/NixOS/patchelf)来修改二进制来除去`rpath`。同时也需要用`strip`来去除调试表(这也使得其依赖gcc)

  这些操作可以在`generic-builder.sh`中完成，加条`fixupPhase`(别忘了在`hellov2.nix`的`buildInputs`中加入`findutils` 和 `patchelf`作为构建依赖)
  ```bash
  # fixup
  find $out -type f -exec patchelf --shrink-rpath '{}' \; -exec strip '{}' \; 2>/dev/null
  ```

  通过`nix build --file hellov2.nix --print-out-paths`
  得到`out path`: `/nix/store/vsrbxvz5psf44gdj7bq3wh0mwzqpx34w-hello`

  再通过`nix-store -q --references /nix/store/vsrbxvz5psf44gdj7bq3wh0mwzqpx34w-hello`
  得到此时的运行时依赖：
  ```
  /nix/store/vq3sdi8l15rzfl5zvmwpafrzis4sm6xf-glibc-2.37-8
  /nix/store/vsrbxvz5psf44gdj7bq3wh0mwzqpx34w-hello
  ```
  哈哈，gcc没有啦！ great :)

  
- `package is self-contained`
  复制一个包的运行时闭包到另一系统上能够直接运行(因为该包不依赖复制到的系统上的gcc(通过`ldd result/bin/hello`能够清楚地看到这一点))，
  其只依赖在构建时的特定位置和版本(/nix/store)中的gcc
  总之，Nix打出的包是**自包含的**



- 题外话：`NAR`(Nix ARchive)
为什么不用已有的归档格式(tar等)？
  它们都具有`不确定性`(不排序，添加时间戳...)
  NAR符合Nix，是`reproducible`的
  可以通过`nix-store --dump` `nix-store --restore` `nix nar ...` `nix store dump-path`来管理NAR
