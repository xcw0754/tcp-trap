# 纳格算法与延迟确认(额外的0.5s)

纳格算法就是[Nagle’s Algorithm](https://zh.wikipedia.org/wiki/%E7%B4%8D%E6%A0%BC%E7%AE%97%E6%B3%95)，它与TCP的一个选项TCP_NODELAY有关。延迟确认就是[TCP delayed acknowledgment](https://en.wikipedia.org/wiki/TCP_delayed_acknowledgment)，它是提升网络效率的一种技术。下面依次了解一下这两者，接着了解它们会产生的火花。



TCP_NODELAY常见的用法是这样的:
```
int opt = 1;
setsockopt(serverfd, IPPROTO_TCP, TCP_NODELAY, &opt, sizeof(opt));
```

现在来了解一下设与不设`TCO_NODELAY`会有什么样的好处与后果。

TCP协议有个著名算法`Nagle’s Algorithm`，此算法一般默认是开启的，它的目的是降低网络消耗。试想你写了个for循环，里面调用了`send`发送一个字符，难道循环多少次就要发送多少个IP包吗？这消耗是很大的，报文头部总长比要发送的数据(1字节)要长得多。所以，`Nagle’s Algorithm`的规则就是:
```
if 有新资料要传送
    if 远端窗口大小 >= MSS and 可传送的资料 >= MSS
        立刻传送完整MSS大小的segment
    else
        if 管道中有尚未确认的资料
            在最新确认（ACK）封包收到前，將资料排进缓冲区队列
        else
            立即传送资料
```

看到这里，其实是没有问题的，积累了更多数据再发送，这个行为很正常。现在来了解一下`TCP delayed acknowledgment`。

`TCP delayed acknowledgment`其实就是延迟确认，远端在收到多个数据包之后才会发送一个ack包对最后一个包进行确认，以说明所有包都收到了。这个机制也是默认开启的，因为这也可以大大地降低网络消耗。举个例子，TCP著名的四次挥手，正常情况下是会产生4个包的，但由于上面提到的delay ack，所以可能出现只有3个包的情况。第2个包是ACK，第3个包是FIN，它们都是同一端发出的，故可能一块发送。由于两个包合并为1个包发送了，所以发第2、3个包的一端没有`CLOSE-WAIT`状态，接收端也没有`FIN-WAIT-2`的状态。所以在只抓到3个包的时候不要惊讶，这是正常的。


问题来了。如果本地开启了`Nagle’s Algorithm`，而远端开启了`TCP delayed acknowledgment`，那么，本地`管道中有尚未确认的资料`这种情况是很常见的，因为远端很可能想要积攒多几个TCP包再给你回ACK，而本地却想积攒多一点数据再发送。双方僵持不下，在等待了最多0.5s([RFC1122 4.2.3.2](https://tools.ietf.org/html/rfc1122))之后，远端(TCP delayed acknowledgment)终于妥协发送了ACK，这样通信得以继续下去。

虽然RFC定义的是0.5s为上限，但是各个平台的实现可能低于这个值。在网络世界里，0.5s已经很严重了，这不是说一条TCP链接从建立到销毁只是额外花费了0.5s，而是说这条链接可能会多次花费这样的0.5s，注意是额外的！这在没有数据要发送的时候其实无关紧要，不会有什么影响的，但是在着急发送数据的时候，这0.5s可能就会有困扰了，试想每次发送数据都要等0.5秒，很恐怖。

不过，这种情况是可以避免的，`TCP delayed acknowledgment`是对端的行为，我们改变不了，所以得从`Nagle’s Algorithm`入手，TCP提供了`TCP_NODELAY`这个选项，可以关闭`Nagle’s Algorithm`，这样每次调用send就会发送数据包了，不会产生这额外的0.5s，但是带来的开销也是要考虑的。

其实，只有在特定情况下调用send才需要等待0.5s(这是上限，未必恰好是这个值)，仔细分析上面提到的资料就可以了。这里不打算从三次握手讲起，直接讲要点。假设现在A端和B端都是TCP的两端，两端处于初始状态，即都没有发送过数据，现在A端发送了500B数据(不含头部)，根据`Nagle’s Algorithm`，这个包应该是立即发送出去。B端收到包后执行延迟确认。接着，A端又发送了500B数据(不含头部)，根据`Nagle’s Algorithm`，得等上个包的ACK或者有更多的数据要发送才会发出这500B数据。A端程序紧接着来了个recv等着收数据。根据以上知识，A端起码要等待的时间计算如下:
```
0.5s B端延迟确认的消耗
1*rtt B端的ACK包
1*rtt A端发送第2个包(A端调用了recv说明了B端收到此包后会回包)
1*rtt B端发来数据顺便ACK掉第2个包
````
理想情况下要等待的时间计算如下:
```
1*rtt A端发送第2个包
1*rtt B端发来数据顺便ACK掉第2个包
```
这样算起来，额外花了`0.5s + 1 * rtt`，这就是`send-send-recv`会产生的问题，如果换成了`send-recv`交替就没问题了，所需耗时分析如下:
```
1*rtt A端发送首包(接着调用recv说明B端会回包啦)
1*rtt B端发来的数据顺便ACK掉第1个包
1*rtt A端发送第2个包
1*rtt B端发来数据顺便ACK掉第2个包
... 类推
```

关于`send-send-send-recv`、`send-send-send-send-recv`、...等等，其实是看你第2个send到首个recv之间发了多少数据，如果待发送数据量没有达到MSS，那还是得等。说那么多，究竟该不该关掉`Nagle’s Algorithm`呢？看着办吧，nginx默认是开启的，而shadowsocks是关闭的。
