.. Ceph调研 documentation master file, created by
   sphinx-quickstart on Mon Apr 22 11:47:04 2013.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

+++++++++++
Ceph Survey
+++++++++++

**Index**

.. toctree::
   :maxdepth: 2

   installation.rst
   arch.rst
   api.rst
   implement.rst
   performance.rst
   reference.rst

**本次调研完成的内容主要有：**

* 搭建Ceph环境，试用客户端：POSIX fuse、S3兼容接口
* 调研Ceph Architecture，对各组件有一定了解
* 试用Ceph提供的librados库、libcephfs库，可以基于这些库构建自定义存储系统
* 调研Ceph S3兼容接口GetBucket的Prefix&Delimiter实现方式

**后续工作主要有以下几点：**

* RADOS对象存储系统的深入调研

  + CRUSH原理和实现，为SDFS多对多实现打基础

    - CRUSH考虑了机架、负载等因素，虽然看上去不怎么靠谱
    - 到时候可以在吃透原理的基础上做一个实现demo

  + 从librados库的接口来看，RADOS在实现基本的KEY/VALUE对象存储的同时，还有以下功能

    - 基于Pool的存储规则隔离，类似SDFS的用户类型（User Type）
    - Pool内对象遍历功能，bitcask模型无法实现这样的功能
    - 实现了快照、从快照恢复、从快照读取的功能
    - 带offset的写，以及append功能
    - 给对象设置xattr kv对
    - AIO异步读写功能
    - Watch/Notify功能：当对象执行某类操作时调用回调函数
    - 这些功能是很多单纯KV系统所不具备的

  + RADOS有自己原生文件系统EBOFS，但为了避免重复造轮子，推荐使用btrfs，RADOS如何组织底层存储Layout值得关注

* MDS集群调研

  + MDS集群的功能是存储POSIX文件系统元数据，解决目录层次的问题，如果我们不使用POSIX，现阶段研究意义不大
  + 如何来组织海量树枝，包括树枝的切分，动态平衡，持久化等问题，都是比较经典的分布式存储问题

* Ceph秉承一份存储，多类接口的设计理念，在整体构架设计上，还有很多值得学习借鉴的地方


