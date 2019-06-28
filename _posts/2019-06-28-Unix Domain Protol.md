---
layout: post
title: "Unix域协议"
---
## 简介

如果我们的目的仅是在同一台主机上的不同进程之间进行通信，那么除了TCP/UDP套接字以外我们还可以使用Unix域协议。Unix域协议是IPC（进程间通信）的方式之一，Unix域协议使用套接字API，支持同一台主机的不同进程之间进行通信。直观上来说Unix域协议有点类似使用本地回环接口（lo）的TCP/UDP。但是Unix域协议比起TCP/UDP套接字还有几个其他优势：1.比起TCP协议通常要更快；2.支持在同一台主机上的不同进程之间传递描述符；3.支持传递客户端凭证。

使用Unix域协议的套接字（以下简称uds[unix domain socket]）用到的API与TCP/UDP套接字API完全一致，即服务端需要进行bind、listen、accpet等操作才能读写，客户端需要先connect才能进行读写。与TCP/UDP套接字不同的一点是uds绑定的地址是一个文件系统的绝对路径，比如"/tmp/myuds"，而TCP/UDP套接字使用的地址则包含了地址和端口号。uds使用的路径并不是普通的文件，需要和uds关联才能对其进行读写。Unix域套接字有两种类型，字节流套接字（类似TCP）和数据报套接字（类似UDP）。

## 相关API

### unix域协议的地址结构

前面提到uds并不使用地址加端口号作为协议地址，而是用一个文件路径来作为地址，所以uds使用的地址结构也有一点不同：

```c
struct  sockaddr_un {
    sa_family_t     sun_family;     /* 协议族，通常为AF_UNIX或者AF_LOCAL */
    char            sun_path[104];  /* 地址路径 */
};
```

uds使用的地址结构叫做sockaddr_un，后面的un即unix，而TCP/UDP套接字使用的地址结构叫做sockaddr_in，in表示internet。
>> unix域协议虽然名字里有unix，但它是POSIX的一部分，并不与unix系统强绑定，POSIX将unix域协议重新命名为“本地IPC”，把AF_UNIX改为了AF_LOCAL，但更多的时候我们还是称其为unix域协议，我们常见的linux和macos都支持unix域协议。

### unix域协议配合套接字API

unix域协议的使用方式与TCP/UDP套接字的方式类似，只需要将协议族替换为AF_LOCAL（或者AF_UNIX），然后将地址替换为sockaddr_un即可。下面是一个使用uds进行bind，然后通过getsockname获取套接字名称并打印的例子：

```c
#include "unp.h"
int main(int argc, char **argv)
{
    int                 sockfd;
    socklen_t           len;
    struct sockaddr_un  addr1, addr2;

    if (argc != 2)
        err_quit("usage: unixbind <pathname>");

    //先调用socket创建套接字
    sockfd = Socket(AF_LOCAL, SOCK_STREAM, 0);
    //对已存在的路径进行bind会导致失败，所以预先调用unlink删除文件
    unlink(argv[1]);
    //调用bzero初始化地址结构体
    bzero(&addr1, sizeof(addr1));
    //设置协议族为AF_LOCAL，AF_UNIX也可以
    addr1.sun_family = AF_LOCAL;
    //设置地址的文件路径
    strncpy(addr1.sun_path, argv[1], sizeof(addr1.sun_path)-1);
    //调用bind，通过SUN_LEN计算bind所需的长度这个参数
    Bind(sockfd, (SA *) &addr1, SUN_LEN(&addr1));

    len = sizeof(addr2);
    //获得socket的名字
    Getsockname(sockfd, (SA *) &addr2, &len);
    printf("bound name = %s, returned len = %d\n", addr2.sun_path, len);
    exit(0);
}
```

执行上面的程序，我们就可以看到控制台会有类似这样的输出：

```
bound name = xxx, returned len = yy
```

程序会输出我们绑定的路径以及对应的socket长度，这时候也可以看到对应路径也自动创建了同名的文件。如果用`ls -lF`命令查看，可以看到对应的文件类型为socket。

### socketpair

socketpair函数可以创建两个连接起来的unix域套接字：

```c
#include <sys/socket.h>
int socketpair(int family, int type, int protocol, int sockfd[2]);
```

socketpair的参数中family必须为AF_LOCAL，protocol必须为0，type可以为SOCK_STREAM或者SOCK_DGRAM，新创建的两个套接字描述符将作为sockfd[0]和sockfd[1]返回。

### 套接字函数

使用uds时，套接字函数中存在一些差异和限制，具体列举如下：

1. 有bind创建的路径名默认的权限为0777（所有者、组用户和其他用户都可读、可写、可执行），并按照当前umask进行修正。

>> umask和chmod中的权限配合使用，是权限的“补码”。比如在我的电脑上umask的值是022，所以uds创建出来的路径权限为777-022=755，表示所有者可读可写可执行，组用户和其他用户可读可写。而通常新创建的目录默认的权限为0777，新创建的文件默认的权限为0666.

2. uds绑定的路径应使用绝对路径。避免使用相对路径的原因是相对路径的解析会依赖调用者的当前路径。

>> POSIX声称使用相对路径绑定到uds将导致不可预计的结果

3. 调用conenct时传入的路径必须是和一个已经打开的uds绑定的路径。并且两个套接字的type（数据报或者字节流）必须相同。出错的条件有几个：a)路径已存在，但不是一个uds;b)路径已存在且是一个uds，但是没有与之关联的打开的描述符;c)路径已存在并且是一个打开的uds，但是类型不同。

4. 调用connect连接一个uds涉及的权限检查等同于调用open以只读模式访问对应路径。
5. unix域字节流套接字与TCP套接字类似，它们都提供无记录边界的字节流接口。
6. 如果对一个uds进行connect时发现监听套接字的队列已满，调用会立即返回一个ECONNECTREFUSED错误；而TCP监听套接字在队列满时则会忽略新到达的SYNC，进而连接发起端发起端进行重试。
7. unix域数据报套接字与UDP套接字类似，它们都提供保留记录边界的不可靠的数据报服务。
8. 在未绑定的uds上发送数据不会自动为其绑定一个路径，这一点不同于UDP套接字：在一个未绑定的UDP套接字上发送数据会为其绑定一个临时端口。这意味着

## 优势
## java中的uds