+++
titile = 'nix-pills-8'
tag = ["nix-pills"]
series = ["nix-pills"]

date = 2023-10-09T12:49:31+08:00
+++



## input design pattern

- 我们已经写了一个`setup.sh`来方便`nix-shell`处于一个和`nix-build`类似的隔离环境，并有分步构建，手动调试的能力，并且用其来构建了一个包(`GNU hello`)。但是当包多了起来，集中管理就变得重要。除此之外，我们写的包过于耦合，不便于用户定制化(比如用户可能想要一个包的不同版本，或者为某个包添加特性等)，应该添加某些“选项”来让用户输入，而函数的参数正是用来做这个的

- 以构建[graphviz-2.49.3.tar.gz](https://gitlab.com/api/v4/projects/4207231/packages/generic/graphviz-releases/2.49.3/graphviz-2.49.3.tar.gz)来说
  如果需要让其支持png格式，那么需要额外的库: `gd`，并且需要用`pkg-config`来提供编译选项。为了让Nix能在隔离的环境中让`pkg-config`找到对应的`.pc`文件，需要在`setup.sh`中export环境变量`PKG_CONFIG_PATH`

- (注：我们把`findutils` 和 `patchelf`加到了`autotoolsv2.nix`的`baseInputs`中)

  `setup.sh`
  ```bash
  for p in $baseInputs $buildInputs; do
    # other code
    ...
    if [ -d $p/lib/pkgconfig ]; then
    # libname/lib/pkgconfig 是包含.pc文件的目录
      export PKG_CONFIG_PATH="$p/lib/pkgconfig${PKG_CONFIG_PATH:+:}$PKG_CONFIG_PATH"
    fi
  done
  ```

  然后在`graphviz.nix`中添加`gd`的包就行
  `graphviz.nix`
  ```nix
  {
    mkDerivation,
    lib,
    gdSupport ? true,
    gd,
    pkg-config
  }:

  mkDerivation {
    name = "graphviz";
    src = ./graphviz-2.49.3.tar.gz;
    buildInputs =
      if gdSupport
        then [
          pkg-config
          (lib.getLib gd)
          (lib.getDev gd)
        ]
      else [];
  }
  ```

  `hellov4.nix`
  ```nix
  {
    mkDerivation,
  }:

  mkDerivation {
    name = "hellov3";
    src = ./hello-2.12.1.tar.gz;
  }
  ```

  把两个包集合在`default.nix`里
  `default.nix`
  ```nix
  let
    pkgs = import <nixpkgs> { };
    mkDerivation = import ./autotoolsv2.nix pkgs;
  in with pkgs; {
      hello = import ./hellov4.nix { inherit mkDerivation findutils patchelf; };
      graphviz = import ./graphviz.nix { inherit mkDerivation lib gd pkg-config; };
      graphvizCore = import ./graphviz.nix {
        inherit mkDerivation lib gd pkg-config;
        gdSupport = false;
      };
    }
  ```

  当我们想要构建其中的`graphviz`包时，可以`nix-build -A graphviz`
  

