---
layout: post
title: socket网络通讯及select使用
tags: 网络编程C
---

#### 一、介绍

**服务端**
socket:创建套接字，返回文件描述符。
bind：套接字绑定信息。绑定ip，端口。
listen:用于被动监听客户链接。将套接字变为被动描述符。
accept:三次握手Ok,返回一个通信描述符。
	注：示例程序中，将新返回描述符加入的监听集合。继续监听通讯数据链接。
	
**客户端**
connect:向服务器发起握手请求。socket之后。

#### 二、链接参考

```
https://vigourtyy-zhg.blog.csdn.net/article/details/102535706?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-102535706-blog-117226481.235%5Ev43%5Epc_blog_bottom_relevance_base4&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-102535706-blog-117226481.235%5Ev43%5Epc_blog_bottom_relevance_base4&utm_relevant_index=2&ydreferer=aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80OTg0MDI4NS9hcnRpY2xlL2RldGFpbHMvMTE3MjI2NDgxP3V0bV9tZWRpdW09ZGlzdHJpYnV0ZS5wY19yZWxldmFudC5ub25lLXRhc2stYmxvZy0yfmRlZmF1bHR%2BYmFpZHVqc19iYWlkdWxhbmRpbmd3b3JkfmRlZmF1bHQtMS0xMTcyMjY0ODEtYmxvZy04OTQwMDIxNS4yMzVedjQzXnBjX2Jsb2dfYm90dG9tX3JlbGV2YW5jZV9iYXNlNCZzcG09MTAwMS4yMTAxLjMwMDEuNDI0Mi4yJnV0bV9yZWxldmFudF9pbmRleD00
```

```
https://blog.csdn.net/weixin_51086006/article/details/132253704
```

Linux中对文件描述符的操作（FD_ZERO、FD_SET、FD_CLR、FD_ISSET）：

```
https://blog.csdn.net/houxiaoni01/article/details/103316774?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-1-103316774-blog-51736431.235^v43^pc_blog_bottom_relevance_base4&spm=1001.2101.3001.4242.2&utm_relevant_index=4
```



#### 三、示例代码

##### server

```
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/select.h>
#include <sys/time.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
 
#define LISTEN_BACKLOG (5)
#define BUF_SIZE (1500)
 
#define ACK_STR "ack ok"
 
void usage(void) {
    printf("*********************************\n");
    printf("./server 本端ip 本端端口\n");
    printf("*********************************\n");
}
 
int main(int argc, char *argv[])
{
    struct sockaddr_in local;
    struct sockaddr_in peer;
    int sock_fd = 0, new_fd = 0;
    int ret = 0;
    socklen_t addrlen = 0;
    char send_buf[BUF_SIZE] = {0};
    char recv_buf[BUF_SIZE] = {0};
    
    if (argc != 3) {
        usage();
        return -1;
    }
    
    char *ip = argv[1];
    unsigned short port = atoi(argv[2]);
    printf("ip:port->%s:%u\n", argv[1], port);
    
    sock_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (sock_fd == -1) {
        perror("socket error");
        return -1;
    }
    
    memset(&local, 0, sizeof(struct sockaddr_in));
    local.sin_family = AF_INET;
    local.sin_addr.s_addr = inet_addr(ip);
    local.sin_port = htons(port);
    
    ret = bind(sock_fd, (struct sockaddr *)&local, sizeof(struct sockaddr));
    if (ret == -1) {
        close(sock_fd);
        perror("bind error");
        return -1;
    }
    
    ret = listen(sock_fd, LISTEN_BACKLOG);
    if (ret == -1) {
        close(sock_fd);
        perror("listen error");
        return -1;
    }
    
    fd_set rfds;
    fd_set rfds_storage;
    FD_ZERO(&rfds_storage);
    FD_SET(sock_fd, &rfds_storage);
    
    int max_fd = sock_fd;
    while (1) {
        rfds = rfds_storage;
        struct  timeval tv = {.tv_sec = 5, .tv_usec = 0};
        ret = select(max_fd + 1, &rfds, NULL, NULL, &tv);
        if (ret == -1) {
            perror("select error");
            break;
        } else if (ret == 0) {
            printf("select timeout\n");
            continue;
        } else {
            printf("select ok\n");
        }
        
        for (int fd = 0; fd < max_fd + 1; fd++) {
            if (FD_ISSET(fd, &rfds)) {
                if (fd == sock_fd) {
                    addrlen = sizeof(peer);
                    new_fd = accept(sock_fd, (struct sockaddr *)&peer, &addrlen);
                    if (new_fd == -1) {
                        perror("accept error");
                        continue;
                    }
                    FD_SET(new_fd, &rfds_storage);
                    max_fd = (new_fd <= max_fd) ? max_fd : new_fd;
                    printf("accept success new fd:%d. listen fd:%d\n", new_fd, fd);
                } else {
                    memset(recv_buf, 0, BUF_SIZE);
                    ret = recv(fd, recv_buf, BUF_SIZE, 0);
                    if (ret <= 0) {
                        close(fd);
                        FD_CLR(fd, &rfds_storage);
                    } else {
                        printf("recv len:%d, %s\n", ret, recv_buf);
                    }
                }
            }
        }
    }
    
    for (int fd = 0; fd < max_fd + 1; fd++) {
        if (FD_ISSET(fd, &rfds)) {
            printf("clsoe fd:%d\n", fd);
            close(fd);
        }
    }
    
    FD_ZERO(&rfds);
    FD_ZERO(&rfds_storage);
    
    return 0;
}
```

##### client

```
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
 
#define LISTEN_BACKLOG (5)
#define BUF_SIZE (1500)
 
#define REQUEST_STR "tcp pack"
 
void usage(void) {
    printf("*********************************\n");
    printf("./client 对端ip 1对端端口\n");
    printf("*********************************\n");
}
 
int main(int argc, char *argv[])
{
    struct sockaddr_in server;
    int sock_fd = 0;
    int ret = 0;
    socklen_t addrlen = 0;
    char send_buf[BUF_SIZE] = {0};
    char recv_buf[BUF_SIZE] = {0};
    
    if (argc != 3) {
         usage();
         return -1;
     }
        
    char *ip = argv[1];
    unsigned short port = atoi(argv[2]);
    printf("ip:port->%s:%u\n", argv[1], port);
    
    sock_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (sock_fd == -1) {
        perror("socket error");
        return -1;
     }
    memset(&server, 0, sizeof(struct sockaddr_in));
    server.sin_family = AF_INET;
    server.sin_addr.s_addr = inet_addr(ip);
    server.sin_port = htons(port);
    
    ret = connect(sock_fd, (struct sockaddr *)&server, 
                  sizeof(struct sockaddr));
    if (ret == -1) {
        close(sock_fd);
        perror("connect error");
        return -1;
    }
            
    int seq = 0;
    while(1) {
        memset(send_buf, 0, BUF_SIZE);
        sprintf(send_buf, "%s:%d", REQUEST_STR, seq++);
        send(sock_fd, send_buf, strlen(send_buf), 0);
        printf("send %s\n", send_buf);
        sleep(1);
    }
    
    close(sock_fd);
    return 0;
}
```

