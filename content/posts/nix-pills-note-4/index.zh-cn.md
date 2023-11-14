+++
titile = 'nix-pills-note-4'
tag = ["nix-pills-note"]
series = ["nix-pills-note"]

date = 2023-11-14T12:51:41+08:00
+++



## generic builder
## 第4篇

- 前面我们构建了两个简单的`derivation`。每个`derivation`含有两个部分：
  - 描述构建过程，作为`builder`的参数的`builder.sh`和`simple-builder.sh`部分
  - 调用构建脚本，真正进行求值和构建部分的`simple.nix`

- 这么做的坏处很明显，每打一个包就要写一个构建脚本(`simple-builder.sh`这种)，而且每个
  builder脚本都会被复制到`/nix/store`里，容易把`/nix/store`搞乱

- 解决方案是写一个相对更通用的`generic-builder.sh`，把变化的量(如包名，依赖等)都写在`packname.nix`里，通过环境变量传过去
  而在`generic-builder.sh`中，读取变量，并利用bash的灵活性进行构建

- 通用性构建的一个例子是使用`autotools`的项目的构建
  `generic-builder.sh`
  ```bash
  set -e
  unset PATH
  for p in $buildInputs; do
    export PATH=$p/bin${PATH:+:}$PATH
  done

  tar -xf $src

  for d in *; do
    if [ -d "$d" ]; then
      cd "$d"
      break
    fi
  done

  ./configure --prefix=$out
  make
  make install
  ```

- 用`GNU hello world`来测试这个`generic-builder.sh`
  源码： `https://ftp.gnu.org/gnu/hello/hello-2.12.1.tar.gz`
  `hello.nix`
  ```nix
  let 
    pkgs = import <nixpkgs> { };
  in
    derivation {
      name = "hello";
      builder = "${bash}/bin/bash";
      system = builtins.currentSystem;
      args = [ ./generic-builder.sh ];
      src = ./hello-2.12.1.tar.gz;
      buildInputs = with pkgs; [
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
    }
  ```

  这里的`buildInputs`提供了一些打包过程中常用的工具(不一定全会用到)
  (注：对于传进去的list型参数，转换为环境变量时会先将每个元素用对应的规则转换为string，然后用空格连接(方便bash做for循环))


- 虽然构建脚本是统一了，但是每次写一个新包，`packname.nix`中仍有不少重复，可以把这部分抽出来，把`derivation`包装一下

  `autotools.nix`
  ```nix
  pkgs: attrs:
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
      };
    in
      derivation (defaultAttrs // attrs)

  ```

  `baseInputs`作为共用的基础性的构建工具
  `buildInputs`是用户自己添加的构建依赖
  由于这里我们引入了一个`baseInputs`，需要在`generic-builder.sh`的for循环中把它加入PATH

  做好`autotools.nix`的抽象后，可以试着用它来构建一个包了
  `hellov2.nix`
  ```nix
  let
    pkgs = import <nixpkgs> { };
    mkDerivation = import ./autotools.nix pkgs;
  in
    mkDerivation {
      name = "hello";
      src = ./hello-2.12.1.tar.gz;
    }
  ```
  这样简单多了！每个包的`.nix`文件无需再重复写共用的依赖


