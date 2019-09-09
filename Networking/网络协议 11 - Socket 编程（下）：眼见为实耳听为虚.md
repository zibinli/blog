---
title: 网络协议 11 - Socket 编程（下）：眼见为实耳听为虚
date: 2018-12-10 10:08:00
tags: 
  - TCP协议
  - socket编程
categories: 
  - 网络协议
---

之前我们基本了解了网络通信里的大部分协议，一直都是在“听”的过程。很多人都会觉得，好像看懂了，但关了页面回忆起来，好像又什么都没懂。这次咱们就“真枪实弹”的码起来，再用一个“神器”-网络分析系统详细跟踪下数据包的生命历程，让我们的理论真实的呈现出来，对网络通信感兴趣的博友，还可以自己拿着系统分析一遍，你一定会大有所获。

不多说，直接上代码。有兴趣的博友可以按各编程语言进行相关改写，然后拿着我们的分析系统真实的看看网络通信过程。

### 本机请求转发到网关
代码中的 192.168.1.10 是内网另一台服务器，楼主的 IP 是 192.168.1.73。在本机跑服务器的时候，要做一个路由配置，否则分析系统无法抓取相关的包。window 下可按下面步骤配置：
1. 管理员身份打开 DOS 窗口；
2. route add 本机ip mask 255.255.255.255 网关ip（路由转发，还记得吗？忘记了？[点我点我点我](https://www.cnblogs.com/BeiGuo-FengGuang/p/9990693.html)）；

什么？不知道怎么查 IP 和网关？[点我告诉你](https://www.cnblogs.com/BeiGuo-FengGuang/p/9877801.html)
操作完成后记得**删除转发规则**，否则，你会发现本机的请求，速度会变得很慢、、、
实例：
```
// 添加路由转发规则
route add 192.168.1.73 mask 255.255.255.255 192.168.1.1 

// 删除转发规则
route delete 192.168.1.73
```

### 基于 TCP 的 Socket
服务端：

```
<?php
/**
 * 1. socket_create: 新建 socket
 * 2. socket_bind:   绑定 IP 和 port
 * 3. socket_listen: 监听
 * 4. socket_accept: 接收客户端连接，返回连接 socket
 * 5. socket_read:   读取客户端发送数据
 * 6. socket_write:  返回数据
 * 7. socket_close:  关闭 socket
 */

$ip = '192.168.1.10';
$port = 23333;
// $port = 80;
$sk = socket_create(AF_INET, SOCK_STREAM, SOL_TCP);
!$sk && outInfo('socket_create error');

// 绑定 IP
!socket_bind($sk, $ip, $port) && outInfo('socket_bind error');

// 监听
!socket_listen($sk) && outInfo('sever listen error');

outInfo("Success Listen: $ip:$port", 'INFO');
while (true) {
    $accept_res = socket_accept($sk);
    !$accept_res && outInfo('sever accept error');
    $reqStr = socket_read($accept_res, 1024);

    if (!$reqStr) outInfo('sever read error');
    outInfo("Server receive client msg: $reqStr", 'INFO');
    $response = 'Hello A, I am B. you msg is : ' . $reqStr . PHP_EOL;
    if (socket_write($accept_res, $response, strlen($response)) === false) {
        outInfo('response error');
    }

    socket_close($accept_res);
}

socket_close($sk);

function outInfo($errMsg, $level = 'ERROR')
{
    if ($level === 'ERROR') {
        $errMsg = "$errMsg, msg: " . socket_strerror(socket_last_error());
    }
    echo $errMsg . PHP_EOL;
    $level === 'ERROR' && die;
}

```

客户端：

```
<?php
/**
 * 1. socket_create:  新建 socket
 * 2. socket_connect: 连接服务端
 * 3. socket_write:   给服务端发数据
 * 4. socket_read:    读取服务端返回的数据
 * 5. socket_close:   关闭 socket
 */

$ip = '192.168.1.10';
$port = 23333;
$sk = socket_create(AF_INET, SOCK_STREAM, SOL_TCP);
!$sk && outInfo('socket_create error');

!socket_connect($sk, $ip, $port) && outInfo('connect fail');
$msg = 'hello, I am A';

if (socket_write($sk, $msg, strlen($msg)) === false) {
    outInfo('socket_write fail');
}

while ($res = socket_read($sk, 1024)) {
    echo 'server return message is:'. PHP_EOL. $res;
}

socket_close($sk);//工作完毕，关闭套接流

function outInfo($errMsg, $level = 'ERROR')
{
    if ($level === 'ERROR') {
        $errMsg = "$errMsg, msg: " . socket_strerror(socket_last_error());
    }
    echo $errMsg . PHP_EOL;
    $level === 'ERROR' && die;
}

```

上面的代码是基于 PHP 原生 Socket 写的，其它语言也有对应 Socket 操作函数，进行相关的改写即可。主要是下面的分析过程。

![](https://img2018.cnblogs.com/blog/861679/201812/861679-20181208172638786-201596475.png)

如上图，这是我们的分析系统捕捉的所有数据传输过程，你可以真实的看到每一步都发生了什么，以及对应的状态的改变（图片较大，建议右键在新标签页打开看）。

在图中上半部分，我们可以看到分析系统将整个 TCP 的生命历程分为了三个阶段：建立连接、交易、关闭连接。这和我们之前了解的理论知识完全相符。
左下角的交易时序图，则详细记录了客户端和服务端每次通信的详细信息，而右下角部分，则展示了每次通信，数据包的状态等信息。

### 基于 UDP 的Socket
```
<?php
/**
 * 1. socket_create:   新建 socket
 * 2. socket_bind:     绑定 IP 和 port
 * 3. socket_recvfrom: 读取客户端发送数据
 * 4. socket_sendto:   返回数据
 * 5. socket_close:    关闭 socket
 */

$ip = '192.168.1.10';
$port = 23333;
$sk = socket_create(AF_INET, SOCK_DGRAM, SOL_UDP);
!$sk && outInfo('socket_create error');

// 绑定 IP
!socket_bind($sk, $ip, $port) && outInfo('socket_bind error');

outInfo("Success Listen: $ip:$port", 'INFO');
while (true) {
    $from = '';
    $reqPort = 0;
    if (!socket_recvfrom($sk, $buf, 1024, 0, $from, $reqPort)) {
        outInfo('sever socket_recvfrom error');
    }

    outInfo("Received msg $buf from remote address $from:$port", 'INFO');
    $response = "Hello $from:$port, I am Server. your msg : " . $buf . PHP_EOL;
    if (!socket_sendto($sk, $response, strlen($response), 0, $from, $reqPort)) {
        outInfo('socket_sendto error');
    }
}

socket_close($sk);

function outInfo($errMsg, $level = 'ERROR')
{
    if ($level === 'ERROR') {
        $errMsg = "$errMsg, msg: " . socket_strerror(socket_last_error());
    }
    echo $errMsg . PHP_EOL;
    $level === 'ERROR' && die;
}
```

客户端：
```
<?php
/**
 * 1. socket_create:  新建 socket
 * 2. socket_write:   给服务端发数据
 * 3. socket_read:    读取服务端返回的数据
 * 4. socket_close:   关闭 socket
 */

$ip = '192.168.1.10';
$port = 23333;
$sk = socket_create(AF_INET, SOCK_DGRAM, SOL_UDP);
!$sk && outInfo('socket_create error');

$msg = 'hello, I am A';

if (!socket_sendto($sk, $msg, strlen($msg), 0, $ip, $port)) {
    outInfo('socket_sendto fail');
}

$from = '';
$reqPort = 0;
if (!socket_recvfrom($sk, $buf, 1024, 0, $from, $reqPort)) {
    outInfo('server socket_recvfrom error');
}

outInfo("Received $buf from server address $from:$port", 'INFO');

socket_close($sk);

function outInfo($errMsg, $level = 'ERROR')
{
    if ($level === 'ERROR') {
        $errMsg = "$errMsg, msg: " . socket_strerror(socket_last_error());
    }
    echo $errMsg . PHP_EOL;
    $level === 'ERROR' && die;
}
```

UDP 数据包分析图：
![](https://img2018.cnblogs.com/blog/861679/201812/861679-20181208174449420-2050339711.png)

如上图，UDP 数据包分析图，明显比 TCP 要简单很多，人家单纯嘛，就不多说了。不过要注意的，写代码的时候，**UDP 的服务端，在循环里千万不要关闭 Socket**。

### 分析系统介绍
上面用到的分析系统叫：科来网络分析系统，[点我下载](http://www.colasoft.com.cn/download/capsa.php)。这个分析系统很良心，提供了一个免费的技术交流版。有兴趣的小伙伴可以下载下来玩玩，很强大。