---
title: 'nix-pills-note-12'
tag: ["nix-pills-note"]
series: ["nix-pills-note"]

date: 2023-11-14T13:04:12+08:00
---



## 第12篇
## nix store paths

- 来看一下Nix的hash部分如何求值

- 先以单个文件为例
  `$ echo mycontent > myfile`
  
  ```
  nix-repl> derivation { system = "x86_64-linux"; builder = ./myfile; name = "foo"; }
  «derivation /nix/store/y4h73bmrc9ii5bxg6i7ck6hsf5gqv8ck-foo.drv»
  ```
  
  `$ nix derivation show /nix/store/y4h73bmrc9ii5bxg6i7ck6hsf5gqv8ck-foo.drv`
  ```
  {
    "/nix/store/y4h73bmrc9ii5bxg6i7ck6hsf5gqv8ck-foo.drv": {
      "args": [],
      "builder": "/nix/store/xv2iccirbrvklck36f1g7vldn5v58vck-myfile",
      "env": {
        "builder": "/nix/store/xv2iccirbrvklck36f1g7vldn5v58vck-myfile",
        "name": "foo",
        "out": "/nix/store/hs0yi5n5nw6micqhy8l1igkbhqdkzqa1-foo",
        "system": "x86_64-linux"
      },
      "inputDrvs": {},
      "inputSrcs": [
        "/nix/store/xv2iccirbrvklck36f1g7vldn5v58vck-myfile"
      ],
      "name": "foo",
      "outputs": {
        "out": {
          "path": "/nix/store/hs0yi5n5nw6micqhy8l1igkbhqdkzqa1-foo"
        }
      },
      "system": "x86_64-linux"
    }
  }
  ```
  (也可以用`nix-store --add myfile`来得到这个hash)

- `inputDrvs`部分文件的hash部分的求值
  其中`/nix/store/xv2iccirbrvklck36f1g7vldn5v58vck-myfile`的hash部分是怎么求出的呢？
  - 1. 计算出文件的hash
  `$ nix-hash --type sha256 myfile`
  `2bfef67de873c54551d884fdab3055d84d573e654efa79db3c0d7b98883f9ee3`
  (
  注：`nix-hash`做的事比普通的`sha256sum`要多一步，就是先把文件或目录打成`NAR`格式，以上面为例，其等于`nix-store --dump | sha256sum`
      而`nix-hash --type sha256 --flat myfile`和`sha256sum myfile`是等价的 
      对于Nix来说，只有两种内容格式，`flat for regular files, or recursive for NAR serializations which can be anything. `
  )

  - 2. 创建字符描述
    `$ echo -n "source:sha256:2bfef67de873c54551d884fdab3055d84d573e654efa79db3c0d7b98883f9ee3:/nix/store:myfile" > myfile.str`

  - 3. 计算出最后的hash
    `$ nix-hash --type sha256 --truncate --base32 --flat myfile.str`
    `xv2iccirbrvklck36f1g7vldn5v58vck`

- `out path`hash部分的求值
  之前我们说过，`out path`在derivation实际被构建之间就已知了，而且只与输入有关
  out path的hash求值与输入类似，除了类型变为了`output:out`(多个输出则类型分别为`output:<id>`)

  - 1. 获取`.drv`文件中除了`out path`部分的内容的hash
    ```
    cp -f /nix/store/y4h73bmrc9ii5bxg6i7ck6hsf5gqv8ck-foo.drv myout.drv
    sed -i 's,/nix/store/hs0yi5n5nw6micqhy8l1igkbhqdkzqa1-foo,,g' myout.drv
    ```
    (
    这么做有点hack，实际上out path的求值过程中应该是有个中间文件，其内容类似如下(全在一行，没有换行):
    ```
    Derive([("out","","","")],[],["/nix/store/xv2iccirbrvklck36f1g7vldn5v58vck-myfile"],"x86_64-linux","/nix/store/xv2iccirbrvklck36f1g7vldn5v58vck-myfile",[],[("builder","/nix/store/xv2iccirbrvklck36f1g7vldn5v58vck-myfile"),("name","foo"),("out",""),("system","x86_64-linux")])
    ```
    即`out`内容为空的`.drv`文件的内容
    )

    求hash
    `$ sha256sum myout.drv`
    `1bdc41b9649a0d59f270a92d69ce6b5af0bc82b46cb9d9441ebc6620665f40b5  myout.drv`

  - 2. 创建字符串描述
    `$ echo -n "output:out:sha256:1bdc41b9649a0d59f270a92d69ce6b5af0bc82b46cb9d9441ebc6620665f40b5:/nix/store:foo" > myout.str`

  - 3. 对字符串描述文件求hash，得到最后的hash
    `$ nix-hash --type sha256 --truncate --base32 --flat myout.str`
    `hs0yi5n5nw6micqhy8l1igkbhqdkzqa1`


- `tar`包采用一种特别的求值方式，其只与文件的内容有关，而与文件名无关(`fixed-output paths`)，但步骤类似
  - derivation有三个特别的和hash相关的参数: `outputHashMode` `outputHash` `outputHashAlgo`，分别表示该derivation的hash的模式("flat" or "recursive")，hash值和hash的算法
    (refer: [outputHashMode](https://nixos.org/manual/nix/stable/language/advanced-attributes.html#adv-attr-outputHashMode))
  - 当指定了`outputHash`时，derivation函数会确保该derivation的`out path`的hash部分的值为`outputHash`的值

  - 以之前的tar文件`hello-2.12.1.tar.gz`为例
  `$ sha256sum hello-2.12.1.tar.gz`的到hash`8d99142afd92576f30b0cd7cb42a8dc6809998bc5d607d88761f512e26c7db20`
  `$ nix-instantiate --expr 'derivation { system = "x86_64-linux"; name = "helloTar"; builder = "none"; outputHash = "8d99142afd92576f30b0cd7cb42a8dc6809998bc5d607d88761f512e26c7db20"; outputHashMode = "flat"; outputHashAlgo = "sha256"; }'`得到derivation `/nix/store/gszqyzlnns85sjy1rj9jg04kil5fl39w-helloTar.drv`

  通过`nix derivation show /nix/store/gszqyzlnns85sjy1rj9jg04kil5fl39w-helloTar.drv` 查看derivation的详细信息
  ```
  {
    "/nix/store/gszqyzlnns85sjy1rj9jg04kil5fl39w-helloTar.drv": {
      "args": [],
      "builder": "none",
      "env": {
        "builder": "none",
        "name": "helloTar",
        "out": "/nix/store/qwj2km5i1p31616kmxgkm9iinfxs7iqr-helloTar",
        "outputHash": "8d99142afd92576f30b0cd7cb42a8dc6809998bc5d607d88761f512e26c7db20",
        "outputHashAlgo": "sha256",
        "outputHashMode": "flat",
        "system": "x86_64-linux"
      },
      "inputDrvs": {},
      "inputSrcs": [],
      "name": "helloTar",
      "outputs": {
        "out": {
          "hash": "8d99142afd92576f30b0cd7cb42a8dc6809998bc5d607d88761f512e26c7db20",
          "hashAlgo": "sha256",
          "path": "/nix/store/qwj2km5i1p31616kmxgkm9iinfxs7iqr-helloTar"
        }
      },
      "system": "x86_64-linux"
    }
  }
  ```
  
  然后像以前一样，把描述性的字符放入文件，然后对其求hash
  `$ echo -n 'fixed:out:sha256:8d99142afd92576f30b0cd7cb42a8dc6809998bc5d607d88761f512e26c7db20:' > helloTar.str`
  `sha256sum helloTar.str`
    `2dd22467c73f65de429fd32c70e68444aeb55f502082f23f8d509185e0341c22  helloTar.str`

  比inputSrc和out path多一步，再用结果放入文件求一次hash
  `$ echo -n "output:out:sha256:2dd22467c73f65de429fd32c70e68444aeb55f502082f23f8d509185e0341c22:/nix/store:helloTar" > helloTar.res`
  `$ nix-hash --type sha256 --truncate --base32 --flat helloTar.res`
    `qwj2km5i1p31616kmxgkm9iinfxs7iqr`
  哈哈，这正是我们的`out path`的hash部分

  

- 总结
  hash的计算总共有两次hash 
  对于输入文件(inputSrcs)来讲，其依赖于文件内容和文件名
    先把用NAR的格式归档的文件本身hash 
    将hash即一些元数据以字符串的形式存在文件中
    对该文件再次hash
  对于输出文件(out path)来说，其hash只依赖输入(`.drv`文件)，而与输出的内容无关
    先将`.drv文件(残缺"out" 部分的版本)`求NAR hash 
    然后把hash与"output:<id>"等源信息以字符串形式写入文件
    最后对该文件求一次`nix-hash --type sha256 --truncate --base32 --flat`得到最后的hash
  tar文件与out path类似，只不过其不依赖`.drv`文件，而只依赖tar文件内容和derivation name等
    
