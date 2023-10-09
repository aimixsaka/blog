+++
titile = 'nix-pills-9'
tag = ["nix-pills"]
series = ["nix-pills"]

date = 2023-10-09T12:20:45+08:00
+++



## callPackage pattern

- 在前面的一节中，我们为了用户灵活度和与仓库文件解耦合，用了`input design`(将包构建写作函数)
  并为了方便包的调用，集中管理，我们把包都集中在`default.nix`
  但是还是有一点不好的地方，在`defaut.nix`中我们还是需要手动传参给函数，所以参数在`graphviz.nix`和`default.nix`中重复了两次

- 我们可以写一个函数(`callPackage`)来自动地填充参数，再和自定义的参数set做个并集
  它应该这么用
  ```nix
  {
    lib1 = callPackage package1.nix { };
    program2 = callPackage package2.nix { someoverride = overriddenDerivation; };
  }
  ```
  所以`callPackage`应该是一个接受一个参数并返回一个函数的函数(或者说接收两个参数)

- 在写`callPackage`的实现前我们先看看Nix中内置的几个有用的函数
  - `builtins.functionArgs {fun}` 返回函数的参数集合，元素是`argname = true|false`的形式，有默认参数值的为true，否则为false。而我们只关心参数的名字
  - `builtins.intersectAttrs {attr set1} {attr set2}` 对两个集合做交集，如果key重复，用后一个覆盖

- 那么`callPackage`可以有如下实现
  ```nix
  callPackage = path: overrides:
    let f = import path;
    in f ((builtins.intersectAttrs (builtins.functionArgs f) allPkgs) // overrides);
  ```

- 具体`default.nix`中`callPackage`可以这么实现
  `default.nix`
  ```nix
  let
    nixpkgs = import <nixpkgs> { };
    # pkgs defined bellow
    allPkgs = nixpkgs // pkgs;
    callPackage = path: overrides:
      let f = import path;
      in f ((builtins.intersectAttrs (builtins.functionArgs f) allPkgs) // overrides);
    pkgs = with nixpkgs; {
      mkDerivation = import ./autotoolsv2.nix nixpkgs;
      graphviz = callPackage ./graphviz.nix { };
      hello = callPackage ./hellov4.nix { };
    };
  in pkgs
  ```
  唔，简洁，精简的代码... ！？等会儿，callPackage用了allPkgs，但是allPkgs需要pkgs，而pkgs里又调用了callPackage，这难道不会死循环吗？
  哈哈，这是函数式`惰性求值`的魔法了。`builtins.intersectAttrs` 不需要知道 `allPkgs`的全部键值对，只有当前一个参数中有`nixpkgs`中不含有的包时才会对`allPkgs`
  剩余部分逐个求值，有则返回
