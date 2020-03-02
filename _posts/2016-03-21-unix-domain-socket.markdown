---
layout: post
title:  "Unix Domain Socket"
date:   2016-03-21
author: YangshengLu
categories: Linux
---
>虽然以前搭建LAMP或者LNMP环境的时候，都接触到过一些.sock结尾的配置，也知道那是一个unix socket文件，但是还没有在代码中使用过unix domain socket. 近来阅读安卓系统代码，遇到了一些，记录下来备忘

unix domain socket创建的过程几乎和tcp socket一模一样。区别只在于unix domain socket没有经过网络层，仅仅接口类似，unix domain socket不会自己重新拆分数据包。废话不多说，上hello world代码。

#### 公共头文件

``` cpp
#ifndef _MSG_H_
#define _MSG_H_

#define MSG_LEN_MAX 1024

struct rpc_msg {
    int code;
    char msg[MSG_LEN_MAX];
};

#endif
```

#### server代码

``` cpp
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <string.h>
#include <fcntl.h>
#include <poll.h>
#include "msg.h"


static int server_fd = -1;
void signal_int_handler(int sig) {
    printf("quit\n");
    if (server_fd != -1) {
        close(server_fd);
    }
    exit(EXIT_SUCCESS);
}

int main(int argc, char *argv[])
{
    int srv = socket(AF_UNIX, SOCK_STREAM, 0);
    if (srv <= 0) {
        perror("create unix socket error");
        exit(EXIT_FAILURE);
    }
    struct sockaddr_un addr;
    bzero(&addr, sizeof(struct sockaddr_un));
    addr.sun_family = AF_UNIX;
    const char *path =  "./srv.sock";
    unlink(path);
    memcpy(addr.sun_path, path, strlen(path));
    if (bind(srv, (struct sockaddr *)&addr, sizeof(addr))) {
        perror("bind error");
        close(srv);
        exit(EXIT_FAILURE);
    }
    if (listen(srv, 8)) {
        perror("listen error");
        close(srv);
        exit(EXIT_FAILURE);
    }
    if (fcntl(srv, F_SETFL, O_NONBLOCK) == -1) {
        perror("fail set non block");
        close(srv);
        exit(EXIT_FAILURE);
    }
    struct pollfd ufds[1];
    ufds[0].fd = srv;
    ufds[1].events = POLLIN;
    signal(SIGINT, signal_int_handler);
    int ev;
    for (;;) {
        ev = poll(ufds, 1, 0);
        if (ev <= 0) {
            continue;
        }
        if (ufds[0].revents & POLLIN) {
            struct sockaddr_un client_addr;
            bzero(&client_addr, sizeof(struct sockaddr_un));
            socklen_t addr_len;
            int s = accept(srv, (struct sockaddr *)&client_addr, &addr_len);
            if (s <= 0) {
                close(s);
                continue;
            }
            struct pollfd rdfd[1];
            rdfd[0].fd = s;
            rdfd[0].events = POLLIN;
            poll(rdfd, 1, 0);
            if (!(rdfd[0].revents & POLLIN)) {
                close(s);
                continue;
            }
            struct rpc_msg msg;
            int nbytes = recv(s, &msg, sizeof(struct rpc_msg), MSG_DONTWAIT);
            if (nbytes != sizeof(struct rpc_msg)) {
                fprintf(stderr, "error in recv rpc_msg, expect %ld bytes, got %d\n",
                        sizeof(struct rpc_msg), nbytes);
                close(s);
            } else {
                printf("code => %d\n", msg.code);
                printf("msg => %s\n", msg.msg);
                close(s);
            }
        }
    }
    return 0;
}
```

#### client代码

``` cpp
#include <sys/un.h>
#include <sys/socket.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include "msg.h"

const char *path =  "./srv.sock";

int main(int argc, char *argv[])
{
    struct sockaddr_un addr;
    bzero(&addr, sizeof(addr));
    addr.sun_family = AF_UNIX;
    memcpy(addr.sun_path, path, strlen(path));
    int s = socket(AF_UNIX, SOCK_STREAM, 0);
    if (s <= 0) {
        fprintf(stderr, "error create socket");
        exit(EXIT_FAILURE);
    }
    connect(s, (struct sockaddr *) &addr, sizeof(addr));
    struct rpc_msg msg;
    bzero(&msg, sizeof(struct rpc_msg));
    msg.code = 200;
    memcpy(msg.msg, path, strlen(path));
    send(s, &msg, sizeof(msg), 0);
    close(s);
    return 0;
}
```
