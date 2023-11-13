# 




## garbage collector

- Nix有着比dpkg，rpm等更细粒度，更精确的垃圾回收
- `gc root`是`/nix/store`下的文件或目录
- Nix用了类似编程语言自动GC的概念及技术，有总是需要保留的`GC roots`(`/nix/var/nix/gcroots/`及`/nix/var/nix/profiles/`及`/run/booted-system`... 目录中指向的`/nix/store/`中的文件或目录)，所有直接或间接引用`GC root`的文件或目录都是需要保留的。除此之外的所有其它`/nix/store/`中的内容都gc掉(gc指的总是`/nix/store`目录下内容的清理)
- 当有旧的`profile`引用该derivation时其也不会被gc(因为旧的profile也在gc root list里)
- Nix先将`/nix/store/`中不在gc root list中的文件/目录移到`/nix/store/trash`中，这个操作是原子性的。然后清空`/nix/store/trash`

- 使用`nix-collect-garbage`进行gc
- 使用`nix-store -q --roots [$(which p/nix/store/mdi7lvrn2mx7rfzv3fdq3v5yw8swiks6-hello-2.12.1ackname)|store drv path|store out path]`查询引用了该derivation的gc root
- 使用`nix-store --gc --print-roots`来查看所有的`gc roots`

- 用`nix-build`等构建的`result/bin/...`都被`/nix/var/nix/gcroots/auto/`里间接引用，所以不会被gc

- `nix-collect-garbage -d`用于删除除了当前profile的所有profile并gc


