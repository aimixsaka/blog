# 




## working derivation
- 为了让`derivation`两个阶段成功运行，除了必要的三个元素，还需要`out path`存在
- 为了构建`out path`，使用bash脚本来进行构建，通过向`derivation`函数传递参数`args = [ ./builder.sh ]`来让`builder(bash)`运行`builder.sh`构建脚本


- 第一个可运行的`derivation`
  `builder.sh`
  ```bash
  declare -xp
  echo foo > $out
  ```

  `nix repl -f '<nixpkgs>'`
  ```
  nix-repl> d = derivation { name = "foo"; builder = "${bash}/bin/bash"; args = [ ./builder.sh ]; system = builtins.currentSystem; }
  nix-repl> :b d
  ```

  通过`nix log /nix/store/gvwysj9dkjjan9hvxfnwq2f97chnz437-foo.drv`来查看构建过程中的日志(包含打印出的环境变量)
  使用`nix derivation show /nix/store/gvwysj9dkjjan9hvxfnwq2f97chnz437-foo.drv`查看`out path`等

- 第二个`derivation`
  由前一个`derivation`的日志可知，`PATH="/path-not-set"`，所以为了构建一个简单的c程序，我们需要将gcc的bin路径加入PATH中
  对`derivation`函数来说，所有传给它的键值对(除了部分保留的元素，如args)都会以一定的规则转换成`builder`的环境变量
  如`derivation { other = ./other.file; some = "some"; }`，其中`other`(转换成复制到/nix/store/中的地址)和`some`会以环境变量的形式传给`builder`(多为bash)
  
  # 源码，作为src环境变量传入
  `simple.c`
  ```c 
  void main() {
          puts("Simple!");
  }
  ```

  # 作为args参数传入，是builder(bash)的参数 (bash simple-builder.sh)
  `simple-builder.sh`
  ```bash
  export PATH="$coreutils/bin/:$gcc/bin"
  mkdir $out
  gcc -o $out/simple $src
  ```

  # 调用derivation求值， 并构建
  ```nix
  nix-repl> d = derivation { name = "simple"; builder = "{bash}/bin/bash"; system = builtins.currentSystem; args = [ ./builder.sh ]; src = ./simple.c; inhert (pkgs) gcc coreutils; }
  nix-repl> :b d

  运行 /nix/store/5hykxkxc8rj9c0q5p19fmagk5ms2rczx-simple/simple
  ```

- 用`.nix`文件而不是`nix repl`确保`reproducible`
  `simple.nix`
  ```nix
  let 
    pkgs = import <nixpkgs> { };
  in
    derivation {
      name = "simple";
      builder = "${pkgs.bash}/bin/bash";
      system = builtins.currentSystem;
      args = [ ./simple-builder.sh ];
      inherit (pkgs) gcc coreutils;
      src = ./simple.c;
    }
  ```
  使用`nix build --file ./simple.nix` 或者 `nix-build ./simple.nix`来求值并构建该`derivation`
  (注：`nix-build`实际上做了两件事， `nix-instantiate`和`nix-sotre -r`)


