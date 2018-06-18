# SIGPIPE信号的产生与处理

`SIGPIPE`信号在网络编程中很常见，它的产生是因为远端已经关闭了连接，而你还调用了`send`想发数据，此时就会产生此信号，当然，你这次`send`肯定是失败的，errno会被置为`EPIPE`。


先简单看一下以下两个程序在干什么，再往后看数据包是怎样的，最后再思考一下可以怎么解决处理这个问题。

serv.c
```
#define LISTEN_PORT (7788)

int main()
{
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0)
    {
        printf("create sock failed\n");
        return 0;
    }

    struct sockaddr_in addr;
    bzero(&addr, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(LISTEN_PORT);
    addr.sin_addr.s_addr = htonl(INADDR_ANY);

    if (bind(sock, (struct sockaddr *)&addr, sizeof(addr)) != 0)
    {
        printf("bind failed\n");
        return 0;
    }

    if (listen(sock, 0) != 0)
    {
        printf("listen failed\n");
        return 0;
    }

    while (true)
    {
        int fd = accept(sock, NULL, NULL);
        if (fd < 0)
        {
            printf("accept failed\n");
            return 0;
        }

        char buf[1024] = {0};
        ssize_t recv_len = 0;

        do
        {
            recv_len = recv(fd, buf, sizeof(buf) - 1, 0);
        }
        while (recv_len <0 && errno == EINTR);

        if (recv_len < 0 && errno == EAGAIN)
        {
            printf("no data\n");
        }
        else if (recv_len < 0)
        {
            printf("other error occured\n");
        }
        else if (recv_len == 0)
        {
            printf("remote closed\n");
        }
        else
        {
            printf("recv data:%s\n", buf);
        }

        // 只要收到第一个包就立即close
        close(fd);
        printf("closed fd\n");
    }

    return 0;
}

```



client.c
```

void handler_pipe(int signo)
{
    printf("got sigpipe\n");
}

int main()
{
    if (signal(SIGPIPE, handler_pipe) == SIG_ERR)
    {
        printf("register handler failed\n");
        return 0;
    }

    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0)
    {
        printf("create sock failed\n");
        return 0;
    }

    struct sockaddr_in addr;
    bzero(&addr, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = inet_addr("127.0.0.1");
    addr.sin_port = htons(7788);

    int opt = 1;
    setsockopt(sock, IPPROTO_TCP, TCP_NODELAY, &opt, sizeof(opt));

    if (connect(sock, (struct sockaddr *)&addr, sizeof(addr)) < 0)
    {
        printf("connect failed\n");
        return 0;
    }

    // 首个包
    char buf[512] = {1};
    if (send(sock, buf, sizeof(buf), 0) != sizeof(buf)) {
        printf("first send failed\n");
        return 0;
    }

    // 保证第1个包已经发送，且收到对方的FIN，此时再发第2个包
    sleep(2);
    if (send(sock, buf, sizeof(buf), 0) != sizeof(buf)) {
        printf("second send failed\n");
        return 0;
    }

    // 第2个包是可以发送成功的，只是远端已经close了，所以相当于数据丢失了。
    // 此时会收到对方的RST，下面的send就会收到SIGPIPE了
    sleep(2);
    if (send(sock, buf, sizeof(buf), 0) != sizeof(buf)) {
        printf("third send failed\n");
        printf("%s\n", strerror(errno));
        return 0;
    }

    close(sock);
    return 0;
}
```

这是以上程序所产生的IP包

![package](https://raw.githubusercontent.com/xcw0754/tcp-trap/master/SIGPIPE/SIGPIPE_test.png)

第1~3个包，著名的TCP三次握手。
第4个包，client发送了512B数据
第5个包，serv发送的ACK，确认第4个包
第6个包，serv调用`close`发送的FIN，单向关闭连接。
第7个包，client回复ACK，确认第6个包
第8个包，client再次发送512B数据
第9个包，serv回复RST终止了连接

注：发第9个包是因为serv已经`close`句柄，无法再接收数据了，告知client别再发送数据了。client在收到第9个包之后，如果还调用send的话，就会产生SIGPIPE信号，因为这条连接已经不存在了。


接下来看看得怎么处理`SIGPIPE`。

在长时间运行的程序中，这种信号必须处理，否则默认就是终止进程，这不是我们期望的那样。以下提供几种常用方式可以避免进程由于`SIGPIPE`信号而被终止。

方法一:

用`sighandler_t signal(int signum, sighandler_t handler);`注册一个handler来处理信号。
```
void handler_pipe()
{
	// 这里可以啥也不干
}
int main()
{
    if (signal(SIGPIPE, handler_pipe) == SIG_ERR)
    {
        printf("register handler failed\n");
        return 0;
    }
    ...
    int ret = send(sock, buf, sizeof(buf), 0);
    if (ret != sizeof(buf) && errno == EPIPE)
    {
        close(sock);	// 这个close不会再发FIN了
        ...
    }
}
```

方法二：

在`send`最后一个参数加入`MSG_NOSIGNAL`，阻止内核发来`SIGPIPE`信号。不过errno仍会置为`EPIPE`。
```
int main()
{
    ...
    int ret = send(sock, buf, sizeof(buf), MSG_NOSIGNAL);
    if (ret != sizeof(buf) && errno == EPIPE)
    {
        close(sock);	// 这个close不会再发FIN了
        ...
    }
}
```

方法三：

直接忽略这个信号，进程不会再收到这个信号了。
```
int main()
{
	signal(SIGPIPE, SIG_IGN);
    ...
    int ret = send(sock, buf, sizeof(buf), MSG_NOSIGNAL);
    if (ret != sizeof(buf) && errno == EPIPE)
    {
        close(sock);	// 这个close不会再发FIN了
        ...
    }
}
```




好了，现在考虑另一个问题，把client.c的第3个`send`换成`recv`会收到`SIGPIPE`信号吗？

答案是否定的，`recv`会立即返回0，且errno为0，即成功。无论再`recv`几次，立即返回的结果还是一样的。所以，当tcp连接中调用`recv`返回值为0表示远端关闭了socket。

