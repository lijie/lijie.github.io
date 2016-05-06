---
layout: post
title:  "Linux System Programming"
date:   2016-05-07 01:10:57
categories: os
---

### File IO

#### GID

新建文件的gid的值, 在Linux下, 根据mount的参数不同, 会遵循sysv或者bsd标准.

sysv会将gid设置为创建者的进程一致, 而bsd则设置为跟父目录一致.

#### fsync, fdatasync

fdatasync不会同步被修改的metadata除非某个metadata会影响文件读写的正确性.

虽然说是这么说, 我其实蛮怀疑具体文件系统的实现上是否会做这么细致的区分.

特别是如果我至少要写一个page的话, 说不定metadata就一并都写了算了...



