+++
titile = 'nix-pills-note-2'
tag = ["nix-pills-note"]
series = ["nix-pills-note"]

date = 2023-11-14T12:51:41+08:00
+++



## 第2篇
## derivation
- 用`builtins.derivation`来创建`derivation`
- 以`Nix language`的角度来看，一个`derivation`只是一个含有`type = derivation` 的 `attribute set`
- `derivation`有必要的三个元素: `name` `system` `builder` 
- 对 `derivation { name = "foo"; system = "fake-system"; builder = "fake-builder"; }` 进行求值，会创建一个`/nix/store/???.drv`文件，但不会直接进行构建
- `.drv`文件描述了如何构建一个`derivation`(attribute set) 
- build `.drv`文件产生output(可能有多个输出)
- `.drv` 和 `out paths`都在`/nix/store/`中

- 可以通过`nix derivation show /nix/store/hash-name.drv`来以`json`的形式查看该`.drv`所表示的详细信息
- 在真正构建之前`out path`就已经确定了(所以后面写构建脚本时可以直接用`$out`，其正是已经求值出的`out path`路径(/nix/store/hash-name)。
  只不过该路径(文件或者目录)还不存在，需要在脚本中自己创建)
- `derivation`的`out path` 只由当前版本下的`Nix`的`inputs`决定，所以在构建之前就已知了，并不是由构建的结果(即`out`)的内容来决定的
- 但是也有根据内容决定的`out path`(例如`tarball`的`out path`) (`[content-addressable](https://en.wikipedia.org/wiki/Content-addressable_storage)`)

- `attribute set` 中的`outPath`类似`java`中的`toString`，当求值`builtins.toString { foo = ???; outPath = "some"; }`时，结果是`"some"`

- 总的来说，有两个阶段
  - (eval/instantiate) stage 对`derivation {...}`进行求值，得到`.drv`文件，并获得其返回值。`.drv`文件的中的依赖(`inputDrvs`)是递归求值的
  - (realize/build) stage 对`.drv`进行构建，创建`out path`，也是递归地进行构建的(先构建出`inputDrvs`中的`out out`)
