---
layout: post
title: "PHP异步调用服务"
description: "PHP 异步 服务 system popen file_get_contents timeout "
category: php
tags: [php, 异步, non-block]
---
{% include JB/setup %}

## Why need?
我们在应用中往往会有这样的场景：用户需要在web界面中启动一个需要耗时很长的操作，比如上传了视频需要压缩，或者一个很长的名单需要处理。这些处理需要花费比较长的时间而不能及时的返回客户端，这样用户就会看到一个timeout的错误信息。在php设置了最大运行时间时，这些应用甚至根本不能执行完成。这会对用户造成困扰。所以我们需要对这些耗时的操作做一个处理，使其既能完成任务，又能合理的提示用户，不会困扰用户。

## 解决方法
既然是异步调用，那么调用的对象是必然存在的，调用的对象也是可以“异步”的，这里所谓的“异步”，是指我们在调用的时候能实时返回，但其仍能在后台运行完成任务。在PHP里面，我们大概有以下3种方式。

### 1.采用system, exec, popen等方式调用后台脚本。
例如：

    $line = system("/opt/app/compress $uploadFile >/dev/null 2>&1", $ret);

这种方式通常来讲适合调用本地的一些功能脚本，当然服务器支持或者具有远程调用的脚本仍然可以完成一切工作。例如:

    $line = system("ssh user@remote-machine-1 -c \"ls /data/logs; echo $?\"", $ret);
    $line = system("connnect-to-remote-machine-and-to-do.sh", $ret);

### 2.采用fopen, file_get_contents, fsockopen, curl 等方式调用远程服务。
可参见 [Supported Protocols and Wrappers](http://www.php.net/manual/en/wrappers.php) 获取php支持的协议。
Ex:

    $contents = file_get_contents("http://rdnotes.com/not-really-exist-service.php?args=arguments-list");

这里要求这些远程服务的返回也是需要实时返回的。为了保险起见，一般需要对这些服务设置一个过期时间。

    $options = array(
        'http' => array(
            'timeout' => 3,
            'method' => 'GET',
            'header' => "User-Agent: Undefined-Name; do you want know me?"
             )
        );
    $context = stream_context_create($options);
    $result = @file_get_contents("the-url-you-want-visit", false, $context);

### 3. 采取中间状态的形式保存任务。
当一些任务的实时性要求没那么高的时候，我们可以将任务保存下来，由后端的服务去取出来执行。可以把任务保存在文件里，或者Memcache, Database里面。


实际生产中，我的做法是分别编写web应用和脚本，由web应用生成任务状态，然后调用脚本异步执行，web应用实时返回告诉用户任务已经在执行了，浏览器使用ajax保持论询获取任务状态，当脚本执行完以后页面就更新告诉用户。
