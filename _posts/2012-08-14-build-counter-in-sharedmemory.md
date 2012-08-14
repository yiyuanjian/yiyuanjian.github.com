---
layout: post
title: "PHP使用共享内存作为计数器"
description: "php 共享内存 sharememory shm 计数器 fclock ftok sysv 高性能"
category: php 
tags: [php, share memory, count, high performance]
---
{% include JB/setup %}

## 何为共享内存？
`共享内存`是指多进程/线程环境中供进程间通信的内存，在Linux环境中，共享内存位于/dev/shm下, 默认设置大概是物理内存的一半。/dev/shm目录可当作普通目录使用。数据保存在内存中，所以操作起来效率比操作磁盘上的文件高，当我们的应用需要高性能的时候就可以将一些临时文件，缓存等文件放在这个目录以提高读写性能。不足的地方是机器重启后数据就丢失了。当物理内存不够时，共享内存的部分可能位于交换区中，这时候性能就会下降。
通过`df -h`命令查看共享内存空间

    $ du -h
    Filesystem            Size  Used Avail Use% Mounted on
    /dev/mapper/vg00-lv00
                           40G  3.4G   35G   9% /
    tmpfs                 934M  248K  933M   1% /dev/shm
    /dev/sda1             2.0G  112M  1.8G   6% /boot
    /dev/mapper/vg00-lv02
                           78G  194M   74G   1% /data
    /dev/mapper/vg00-lv01
                       40G  3.1G   35G   9% /home
    
查看/dev/shm下的文件：

    $ ls -l /dev/shm
    total 248
    -r--------. 1 yiyuanjian users 67108904 Aug  6 11:08 pulse-shm-1339680607
    -r--------. 1 yiyuanjian users 67108904 Jul 30 09:20 pulse-shm-1363895807
    -r--------. 1 yiyuanjian users 67108904 Jul 30 16:33 pulse-shm-1715517018
    -r--------. 1 yiyuanjian users 67108904 Aug 14 12:09 pulse-shm-2034719936
    -r--------. 1 yiyuanjian users 67108904 Jul 30 09:20 pulse-shm-2081537163
    -r--------. 1 yiyuanjian users 67108904 Aug 13 15:33 pulse-shm-2128019069
    -r--------. 1 yiyuanjian users 67108904 Jul 31 18:05 pulse-shm-3456407072
    -r--------. 1 gdm        gdm   67108904 Jul 30 09:20 pulse-shm-3521973166
    -r--------. 1 yiyuanjian users 67108904 Aug 14 14:13 pulse-shm-871492857

查看现在系统中存在的共享内存

    $ ipcs -m
    ------ Shared Memory Segments --------
    key        shmid      owner      perms      bytes      nattch     status
    0x00000000 5373980    yiyuanjian 600        393216     2          dest                
    0x00000000 37584927   yiyuanjian 600        784        2          dest
    ......         
    0x61020075 16187426   yiyuanjian 600        65536      0


## 为什么要使用共享内存？

共享内存通常位于物理内存中，IO性能较磁盘文件快上许多。

## 在PHP中使用共享内存

在php中使用共享内存有2个函数组, shm 和 shmop, 两者的区别就是前者是sys v标准的，只存在于Linux/Unix中，而后者则不依赖于系统，在windows里面也可以使用。

### shm_ 组
使用此函数组需要编译时指定 --enable-sysvshm, 通常情况下，消息队列--enable-sysvmsg和信号量--enable-sysvsem也是我们需要的，一般可以一并打开。这组函数包括：

    shm_attach
    shm_detach
    shm_get_var
    shm_has_var
    shm_put_var
    shm_remove
    shm_remove_var

### shmop_ 组
使用此函数组需要编译时指定 --enable-shmop。这组函数包括：

    shmop_open
    shmop_close
    shmop_delete
    shmop_read
    shmop_size
    shmop_write

为保证兼容，我们使用shmop_函数组

    // Open
    $shmId = shmop_open(ftok(__FILE__, 'k'), 'c', 0600, 100000);
    //Write。FIXME： 需检查数据长度，不能超过申请大小
    shmop_write($shmId, serialize($data), 0);
    //read
    $data = shmop_read($shmId, 0, 100000);
    //close
    shmop_close($shmId);

Notes: ftok为sys v中的函数，需要判断一下

    if(!function_exists("ftok")) {
        $s = stat(__FILE__);
        return sprintf("%u", (($s['ino'] & 0xffff) | (($s['dev'] & 0xff) << 16)
                        | (($proj & 0xff) << 24)));
    }



## 如何加锁保证排它操作
PHP中有两种方式加排它锁，一种是使用flock函数，可以在内存中创建一个文件来进行加解锁操作，另外一个是上文提到的信号量


### flock
使用文件方式加解锁

    //lock
    $fp = fopen("/dev/shm/".$lockFile, "wb");
    //TODO: check the $fp
    flock($fp, LOCK_EX);
    //do operation here
    //unlock
    flock($fp, LOCK_UN);
    fclose($fp);

### 信号量

    $sem = sem_get($key, 1, 0600); //一个锁，仅限于自己使用
    //lock
    sem_acquire($sem);
    //do operation here
    //unlock
    sen_release($sem);

Note: 千万不能重复2次sem_acquire, 否则会导致死锁！

这样，我们就可以把一些需要快速操作的数据保存在共享内存里，再定时把数据同步到其他地方，尤其适合动作计数器。

