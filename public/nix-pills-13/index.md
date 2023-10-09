# 




## stdenv

- stdenv 是我们打包中最常用到的工具，`nixpgks`中绝大多数包都是用`stdenv.mkDerivation`打的

- stdenv 只是一个普通的`derivation`，其为`stdenv.mkDerivation`提供out path。可以用`nix build 'nixpgks#stdenv'`来看看其构建的东西
```bash
$ ls -R result
result:
nix-support  setup

result/nix-support:
```

重要的是`setup`文件，打开看看的话可以发现和我们之间写的setup.sh很类似
可以通过`nix derivation show 'nixpgks#stdenv'`看看它的`.drv`的详细信息(
能看到args也是有个`builder.sh`
还有`defaultBuildInputs` `initialPath`等，类似我们之前写过的`baseInputs`，包含(提供)了许多基本工具(gcc, gawk...)
)

- stdenv.mkDerivation更类似我们写过的`autotoolsv2.nix`，它是一个函数，
  接收一个set，返回一个derivation。其args用了`stdenv`这个derivation的out path
  ```
  {
    ...
    builder = attrs.realBuilder or shell;
    # default-builder类似我们写的generic-builder.sh。其只是source $stdenv/setup，然后调用generibBuild
    args = attrs.args or ["-e" (attrs.builder or ./default-builder.sh)];
    # 这里result就是stdenv derivation。在nix-build或nix-shell时会转为out path并作为环境变量
    # 所以`nix-shell -E 'with import <nixpkgs> {}; stdenv.mkDerivation { name = "foo"; }'`后`echo $stdenv`能看见stdenv的out path
    stdenv = result;
    ...
  }
  ```

