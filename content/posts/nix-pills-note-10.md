+++
titile = 'nix-pills-note-10'
tag = ["nix-pills-note"]
series = ["nix-pills-note"]

date = 2023-11-13T11:32:09+08:00
+++



## override pattern

- 前面我们利用函数参数，通过改变调用时传入的参数，来获取一个包的不同变种

- 但是那仍然有些坏处，比如需要包的维护者手动调用来传参，如果包的可选项多了起来，一个包的不同变种就有十几种，每个特性都让维护者来调用形成一个新derivation
  不太现实。 相反，如果能让用户拥有改变参数的能力，那么他可以很自然地根据自己的需求来传参获取derivation，而包维护者只需要为那些选项设一个默认值即可
  比如
  与其：
  ```nix
  graphviz = callPackage ./graphviz.nix { };
  graphviz = callPackage ./graphviz { gdSupport = false; };
  ```
  不如：
  ```nix
  graphvizCore = pkgs.graphviz.override { gdSupport = false; };
  ```
- 函数式让我们很容易能够做到这一点，这是其它只用一系列用串行shell函数描述打包的包管理比较难做到的

- 基本的思路是为每个derivation的返回值(一个attr set)添加一个`override`的key，它的value是一个函数，该函数接收一个set类型的参数，返回一个新的derivation
  而derivation最后都是调用打包函数(`graphviz.nix`等)形成的，那么这个override应该应该是 用已有的参数，和提供的参数(`override`的参数)，做个 `//`，然后把这个合集传给打包函数，得到一个新的derivation。 另，还需要返回的derivation仍然具有override的能力(属性)

  那么可以写出类似如下的函数
  ```nix
  {
    makeOverridable = f: origArgs:
      let 
        origRes = f origArgs;
      in 
        origRes // { override = newArgs: f (origArgs // newArgs); };
  }
  ```
  其中`origRes`是一个`derivation`，`origRes // { override = newArgs: f (origArgs // newArgs); }` 也是一个`derivation`

  为了让返回的derivation也具有override的能力，可以这么改：
  ```nix
  # 为了让makeOverridable能访问自身，rec是必需的
  rec {
    makeOverridable = f: origArgs:
      let 
        origRes = f origArgs;
      in 
        origRes // { override = newArgs: makeOverridable f (origArgs // newArgs); };
  }
  ```
  唔，精巧！

  最后只需要让我们的`callPackage`集成这个功能就大功告成了

  `default.nix`
  ```nix
  let
    nixpkgs = import <nixpkgs> { };
    # pkgs defined bellow
    allPkgs = nixpkgs // pkgs;
    makeOverridable = f: origArgs:
      let 
        origRes = f origArgs;
      in 
        origRes // { override = newArgs: makeOverridable f (origArgs // newArgs); };
 
    callPackage = path: overrides:
      let f = import path;
      in makeOverridable f ((builtins.intersectAttrs (builtins.functionArgs f) allPkgs) // overrides);
    pkgs = with nixpkgs; {
      mkDerivation = import ./autotoolsv2.nix nixpkgs;
      graphviz = callPackage ./graphviz.nix { };
      graphvizCore = callPackage ./graphviz.nix { gdSupport = false; };
      hello = callPackage ./hellov4.nix { };
    };
  in pkgs
  ```

