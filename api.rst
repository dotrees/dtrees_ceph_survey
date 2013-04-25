+++
API
+++

.. image:: _static/ditaa-767aa6c670f2dbf104e76207678627effaf25695.png

librados
========

RADOS提供可靠分布式对象存储服务，其服务接口通过\ `librados库 <http://ceph.com/docs/master/rados/api/librados/>`_\统一给出，前文我们提到Ceph给用户提供了：\ **对象存储接口**\、\ **块设备接口**\、\ **标准POSIX文件系统接口**\，这些接口的实现都是基于librados库。我们甚至可以使用librados库开发自定义Client，本节给出使用librados库编程的示例。\ **（试用librados并不需要MDS参与，RADOS服务是不包括MDS的）**

使用\ **rados_create()**\实例化\ **rados_t**\变量：

.. code-block:: c
    :linenos:

    int err;
    rados_t cluster;

    err = rados_create(&cluster, NULL);
    if (err < 0) {
        fprintf(stderr, "%s: cannot create a cluster handle: %s\n", argv[0], strerror(-err));
        exit(1);
    }

使用\ **rados_conf_read_file()**\函数读取配置文件

.. code-block:: c
    :linenos:

    err = rados_conf_read_file(cluster, "/path/to/myceph.conf");
    if (err < 0) {
        fprintf(stderr, "%s: cannot read config file: %s\n", argv[0], strerror(-err));
        exit(1);
    }

配置文件读取完成后，即可连接RADOS集群\ **rados_connect()**

.. code-block:: c
    :linenos:

    err = rados_connect(cluster);
    if (err < 0) {
        fprintf(stderr, "%s: cannot connect to cluster: %s\n", argv[0], strerror(-err));
        exit(1);
    }

进行IO操作前，使用\ **rados_ioctx_create()**\创建一个IO Context，注：poolname必须是一个存在的pool，可以使用Ceph命令行工具创建。

.. code-block:: c
    :linenos:

    rados_ioctx_t io;
    char *poolname = "mypool";
    
    err = rados_ioctx_create(cluster, poolname, &io);
    if (err < 0) {
        fprintf(stderr, "%s: cannot open rados pool %s: %s\n", argv[0], poolname, strerror(-err));
        rados_shutdown(cluster);
        exit(1);
    }

至此，IO Context创建完成，用户可以使用它进行相关数据操作，例如：使用\ **rados_write_full()**\ 函数写入对象。

.. code-block:: c
    :linenos:

    err = rados_write_full(io, "key", "i am value", 5);
    if (err < 0) {
        fprintf(stderr, "%s: cannot write pool %s: %s\n", argv[0], poolname, strerror(-err));
        rados_ioctx_destroy(io);
        rados_shutdown(cluster);
        exit(1);
    }

程序退出前，务必进行清理

.. code-block:: c
    :linenos:

    rados_ioctx_destroy(io);
    rados_shutdown(cluster);

编译巨简单

.. code-block:: c
    :linenos:

    gcc -o test rados_test.c -lrados

libcephfs
=========

libcephfs基于RADOS和MDS构建，使用他能够方便得实现标准POSIX文件系统接口，Ceph上层内核文件系统和基于fuse的文件系统，都是调用了libceph。与librados相比，libcephfs凭借MDS提供的文件系统元数据服务，实现了包括文件目录，文件权限控制，文件xattr属性等POSIX友好的接口。下面我们将通过代码示例的方式展示libcephfs库的强大功能。

首先调用\ **ceph_create()**\创建挂载点实例\ **ceph_mount_info**

.. code-block:: c
    :linenos:

    struct ceph_mount_info *cmount;
    err = ceph_create(&cmount, NULL);
    if (err < 0) {
        fprintf(stderr, "%s: ceph create fail: %s\n", argv[0], strerror(-err));
        exit(1);
    }

和librados一样，也要读取配置文件，读取配置文件的函数为\ **ceph_conf_read_file()**

.. code-block:: c
    :linenos:

    err = ceph_conf_read_file(cmount, NULL);
    if (err < 0) {
        fprintf(stderr, "%s: ceph conf read fail: %s\n", argv[0], strerror(-err));
        exit(1);
    }

配置读取完成后，进行文件系统挂载

.. code-block:: c
    :linenos:

    err = ceph_mount(cmount, NULL);
    if (err < 0) {
        fprintf(stderr, "%s: ceph mount fail: %s\n", argv[0], strerror(-err));
        exit(1);
    }

挂载成功后就能对文件系统进行操作了，例如：打开文件(ceph_open)、写文件(ceph_write)、关闭文件(ceph_close)

.. code-block:: c
    :linenos:

    // open file
    int flag = O_CREAT | O_RDWR;
    int mode = S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP | S_IROTH | S_IWOTH;
    int fd = ceph_open(cmount, "cephfs.txt", flag, mode);
    if (fd < 0) {
        fprintf(stderr, "%s: ceph open fail: %s\n", argv[0], strerror(-err));
        exit(1);
    }

    // write file
    int count = ceph_write(cmount, fd, "hello", 5, 0);
    if (count != 5) {
        fprintf(stderr, "%s: ceph write fail: %s\n", argv[0], strerror(-err));
        exit(1);
    }

    // close file
    ceph_close(cmount, fd);

写完后，还能调用dir系列操作，查看文件是否写入成功

.. code-block:: c
    :linenos:

    // open dir
    struct ceph_dir_result *dirp = NULL;
    err = ceph_opendir(cmount, "/", &dirp);
    if (err < 0) {
        fprintf(stderr, "%s: ceph opendir fail: %s\n", argv[0], strerror(-err));
        exit(1);
    }

    // read dir
    while (1) {
        struct dirent de;
        err = ceph_readdir_r(cmount, dirp, &de);
        if (err < 0) {
            fprintf(stderr, "%s: ceph readdir_r fail: %s\n", argv[0], strerror(-err));
            exit(0);
        }
        if (err == 0) {
            break;
        }

        fprintf(stdout, "dname: %s\n", de.d_name);
    }

    // close dir
    ceph_closedir(cmount, dirp);

最后umount挂载的cephfs实例，\ **ceph_shutdown**

.. code-block:: c
    :linenos:

    ceph_shutdown(cmount);

编译同样很方便

.. code-block:: c
    :linenos:

    gcc -o test cephfs_test.c -D_FILE_OFFSET_BITS=64 -lcephfs

运行输出如下结果，说明写入操作成功

.. code-block:: c
    :linenos:

    dfs@inspur1:~/ldm/ceph$ ./test 
    cwd: /
    dname: .
    dname: ..
    dname: cephfs.txt
