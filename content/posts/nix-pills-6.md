+++
titile = 'nix-pills-6'
tag = ["nix-pills"]
series = ["nix-pills"]

date = 2023-10-09T12:20:10+08:00
+++



## developing(debug) with nix-shell

- 当我们运行`nix-shell hellov2.nix`时，与`nix-build`的区别是其只会进入bash环境，并把`derivation` 
  中的参数作为环境变量传进去，但不会将`args`作为参数运行
  即`nix-shell`环境下只有变量，不自动运行`generic-builder.sh`脚本
  (当然想要运行构建脚本我们可以手动`source generic-builder.sh`)
  但是手动source有许多坏处：
    - 脚本在当前目录运行，没在特殊的隔离的环境中，原shell的环境可能会影响构建(impure)
    - tar包被解压到当前目录，污染用户环境
    - `generic-builder.sh`没有被复制到`/nix/store/`中

- 但是`nix-shell`可以方便分步手动构建，调试。前提是构建是分步的，而且`nix-shell`有构建所需的环境(make,gcc,等)



- 为了同时满足`nix-build` 和 `nix-shell`的需求(即nix-build时运行args中的构建脚本，nix-shell不运行args构建脚本，只传入环境变量，可以利用环境变量手动调用构建步骤)
- 那么可以把构建步骤写成各个独立的函数，写在一个`setup.sh`里，作为环境变量传进shell(`setup = ./setup.sh`)。然后作为args参数的`generic-builder.sh`可以`source $setup`，
  再调用`setup.sh`中的构建函数进行构建

- 结构如下：
  `setup.sh`
  ```bash
  # 不应该在这set -e，在generic-builder.sh中做
  # 毕竟手动调用buildPhase等步骤时可不想因为某些错误就直接退出bash了
  # set -e
  
  unset PATH
  for p in $baseInputs $buildInputs; do
    export PATH=$p/bin${PATH:+:}$PATH
  done
  
  function unpackPhase() {
    tar -xf $src
    for d in *; do
      if [ -d "$d" ]; then
        cd "$d"
        break
      fi
    done
  }
  
  
  function configurePhase() {
    ./configure --prefix=$out
  }

  function buildPhase() {
    make
  }

  function installPhase() {
    make install
  }

  function fixupPhase() {
    find $out -type f -exec patchelf --shrink-rpath '{}' \; -exec strip '{}' \; 2>/dev/null
  }

  # generic build phase calling convenience for generic-builder.sh
  function genericBuild() {
    unpackPhase
    configurePhase
    buildPhase
    installPhase
    fixupPhase
  }
  ```

  `autotoolsv2.nix`可以这么写
  ```nix
  let 
    defaultAttrs = {
      builder = "${pkgs.bash}/bin/bash";
      args = [ ./generic-builder.sh ];
      baseInputs = with pkgs; [
        gnutar
        gzip
        gnumake
        gcc
        coreutils
        gawk
        gnused
        gnugrep
        binutils.bintools
      ];
      buildInputs = [ ];
      system = builtins.currentSystem;
      setup = ./setup.sh;
    };
  in
    derivation (defaultAttrs // attrs)

  ```

  `generic-builder.sh`
  ```bash
  set -e
  source $setup
  genericBuild
  ```

  这时候我们的`hellov3.nix`可以这么写
  ```nix
  let 
    pkgs = import <nixpkgs> { };
    mkDerivation = import ./autotoolsv2.nix pkgs;
  in 
    mkDerivation {
      name = "hellov3";
      src = ./hello-2.12.1.tar.gz;

      buildInputs = with pkgs; [
        findutils
        patchelf
      ];
    }
  ```

  `nix-shell hellov3.nix`进shell后，既可以手动`source $setup`来获得构建环境
  也可以`nix build --file hellov3.nix`来直接构建了
