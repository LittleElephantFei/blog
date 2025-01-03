---
title: blog_install.md
date: 2024-12-22 21:40:07
tags:
---


# TODOLIST



## 2024.10.22





rwnd的调整可以监控速率，不需要根据计数器的方式，如果入队速率连续多次超过出队速率，那么就应该调整rwnd了。





师兄给了意见。

这个实验可以，但是要更加详细的分析，说明确实是减少了零窗口事件以及等待恢复的时间，从而实现了更高的平均吞吐量。

当然最主要的还是算法要有理有据，即接收端什么时候知道cwnd已经增长到了即将要占满接收缓存的值，rwnd要减少多少才能保证不使吞吐量降低的情况下还能保证接收缓冲区队列维持在合适大小。否则可能导致你现在设置的参数在这个网络条件下有提升，换个网络条件反而由于rwnd不合适的减少而降低了吞吐量。 还有一种方案是让接收缓冲区维持在一个合适的区间，比如现在缓冲区还剩多少，那我就可以将rwnd设置为多少，以保证在这个ack返回到发送端这段时间内（通常为rtt的一半）进来的数据量不会超过缓冲区剩余空间。甚至再精细一点可以综合考虑缓冲区的入队列速率（接收到数据）和出队列速率（被上层读走数据）来设计如何调整rwnd以保证缓冲区不会溢出。





## 2024.10.09记录

如何调整rwnd呢，目前在tcp的调整rwnd的函数里加了一些逻辑，但是调整的频率太低太低了。发了60秒，只调整了一次。所以在这个位置不太行。

- 主动去调整rwnd，

  - 写在排队的函数里，但是需要验证一下这里的有效性。是不是能及时的调整rwnd.
- 通过ack反馈给发送端，利用tcp头部的预留字段



2024.10.10

论文图

- 接收队列中平均排队长度降低
- 多留并发的吞吐量增加
- 单流完成时间不变的图
- 长流、短流

## 实验

- 在发送端加入观察缓冲区堆积的情况，根据缓冲区调整rwnd的代码

- 写一个文件流发送的socket，尝试一下在接收端调整rwnd是不是不影响流的完成时间。

  

## 调研

- 调研ecn标志是怎么设置的，在发送端是怎么解析的，在接收端是怎么设置的？ecn除了0和1能不能设置为其他值。
- ecn在什么情况下设置为1======显示拥塞控制相关的论文
- 反向代理服务器能否修改要转发的tcp包，======读一下吕超的论文



# Idea



## 突发奇想

反向代理的基于队列的拥塞控制：

- 既然有一个基于发送端的残余队列，接收端是不是也有一个消耗队列呢？在消耗队列去下手，然后结合到ack中？作为显示拥塞信息？

- 接收端决定传输速率的拥塞控制，接收端的能力有限，对不同应用的速率要求是不同的。

- 修改接收端呢，不一定不能修改接收端呀。对于发送的发来显示拥塞信号给予更大的惩罚。

  哦哦哦，，反向代理，服务器端可以根据残余队列的长度，不一定非要返回一个长度啊，可以在服务器端自己反馈一个拥塞信号给发送端。分段处理ecn的标志啊，由布尔信号传递为一个程度信号！！！！

  这个一定能行！分段处理拥塞信号

- 拥塞控制优化：针对短流，激进一点发送呢？局域网内，广域网内？基于历史cwnd，历史的

  ​	针对长流+强化学习+监督学习

- 慢启动优化：基于机器学习的慢启动优化，慢启动的记忆点 + 驱动队列调整cwnd

  ​	FBE的方案：前两轮rtt的返回速率 + 驱动队列调整cwnd

  ​	我的方案：基于历史的cwnd + 驱动队列调整cwnd

- 创新点

  内核中位于一个到目的ip的cwnd,min_rtt,max_rtt的表，

  ​	新的ip:

  ​	旧的ip:

  针对短流、还是长流？



## 成熟的想法

### 1. 基于接收端的，多流并发，多流协同优化

接收端的rwnd有一个最小值，默认值，最大值。不应该对所有的应用是一视同仁的。rwnd是远小于MTU的

| 参数     | 层次          | 控制对象       | 主要作用                         |
| -------- | ------------- | -------------- | -------------------------------- |
| **MTU**  | 链路层        | 数据包大小     | 限制单个网络帧的最大传输大小     |
| **cwnd** | 传输层（TCP） | 发送方拥塞窗口 | 控制发送方可以传输的未确认数据量 |
| **rwnd** | 传输层（TCP） | 接收方窗口     | 限制发送方根据接收方缓冲区的大小 |

接收端去优化接收窗口，避免拥塞情况的发生。

应用层能消耗的数据量小于网络发送来的数据量情况？即应用层的吞吐量小于网络吞吐量，或在某些时刻，应用层对数据的要求小于网络的吞吐量，或者说限制应用层的吞吐量不会影响用户体验，那么此时网络吞吐量可以适当减小，从而减少拥塞情况的发生。

网络协议栈有那么多的缓冲区，从缓冲区长度去入手，如果在某条流的缓冲区超过了设定的阈值，这个现象说明应用层对当前网络传输过来的数据使用速率没有那么快。这时候网络虽然没有发生拥塞，甚至可以提高cwnd，但是再次提高cwnd是没有任何意义的，因为发送来的数据只会存放到接收端的缓冲区，并不会送到应用层供应用使用。所以，这时候适当减小cwnd反而是一个更明智的行为，为什么呢？适当减小cwnd会让网络发送来的速率和应用层使用数据的速率进行匹配，这样不会造成在接收端缓冲区数据的排队，而接收端可以把空出来可增加cwnd的“余量”分配给更加需要的流（因为网卡发送能力是有限的），从而更高效的利用当前网络的资源。（应用层吞吐量才是网络的终极目标？）

在很长的时间内，比如5个RTT，观察到缓冲区超过阈值，则可以作为一个减小rwnd的信号



提出一个新的拥塞控制算法：对bbr加入自定义的显示拥塞控制

- 利用自定义的tcp头部空间，接收端赋值，发送端解析。优点：扩大了原本简单的拥塞控制信息，由之前的布尔信息扩展为多元信息

- 发送端可以及时及时知道接收端对于数据的需求，可以更高校的利用发送端的发送能力。扩大多流并发的有效吞吐量，单流完成时间不变

  

#### Movtivation

- 在接收端打印队列堆积长度，发现随着rwnd的增加，堆积长度也是增加的



#### 实验结果预期：

- 接收端队列累积时长的优化，堆积时长更小，说明发送速率和应用消耗速率匹配更佳

  - 队列堆积时长与bbr,cubic进行对比

- 对于发送端，多流并发吞吐量之和，多留并发的吞吐量之和增加就说明有效果。

  - 多流吞吐量之和

  - rtt几乎不变

  - 流完成时间之和

  - 多留并发的数量：流的数量，2，5，8，10.流的上限

    

- 对于接收端，单流的流完成时间，如果相差无几的话，说明有效果并且可以更有效的利用带宽。

  - 流完成时间可以与bbr，cubic进行对比。短流，长流。
  - 

### 2. 基于发送端的，多流并发，优先级的优化

pfifo_fast优化，Qdisc层的优化

- 低优先级响应时间降低
- 低优先级排队时延的降低
- 多流公平性分析





### 3. 显示拥塞控制的改进？

https://zhuanlan.zhihu.com/p/395200230

修改发送端呢，不一定不能修改发送端呀。对于发送的发来显示拥塞信号给予更大的惩罚。

哦哦哦，，反向代理，服务器端可以根据残余队列的长度，不一定非要返回一个长度啊，可以在服务器端自己反馈一个拥塞信号给发送端。分段处理ecn的标志啊，由布尔信号传递为一个程度信号！！！！

这个一定能行！分段处理拥塞信号







# 实验笔记



### 实验一：自定义tcp头部



- 能否自定义tcp的预留字段，发送端，接收端能否成功解析。

  **修改发送逻辑**

  在 `net/ipv4/tcp_output.c` 中，TCP 报文在函数 `tcp_transmit_skb()` 中被构造。找到该函数，并在其中设置保留字段。例如：

  ```c
  struct tcphdr *th;
  th = (struct tcphdr *)skb_transport_header(skb);
  
  /* 设置保留字段，假设你想设置为 0b1010 */
  th->res1 = 0xA;  // 设置预留字段为二进制 1010
  
  ```

  **修改接收逻辑**

  接收 TCP 报文的逻辑主要在 `net/ipv4/tcp_input.c` 中处理。在函数 `tcp_rcv_established()` 中，你可以读取 `tcphdr` 结构体，并处理保留字段：

  ```c
  struct tcphdr *th;
  th = (struct tcphdr *)skb_transport_header(skb);
  
  /* 读取保留字段的值 */
  u8 reserved_bits = th->res1;
  
  /* 自定义处理逻辑 */
  if (reserved_bits == 0xA) {
      // 处理特定的保留字段值
  }
  ```



tcp头部有预留字段啊，这个可以用来自定义显示拥塞控制。

![1](https://pic1.zhimg.com/80/v2-efab0aee7a556a8815aebe2e6da518f4_1440w.webp)



实验虚拟机：ubuntu01发包，ubuntu02收包

ubuntu01发包，

```C
th->res1 = 0xA;  // 设置预留字段为二进制 1010
```

ubuntu02收包

```C
/* 读取保留字段的值 */
u8 reserved_bits = th->res1;
printk(KERN_DEBUG "the TCP reserved_bits  is %u\n",reserved_bits);
printk(KERN_DEBUG "add successful!!==============================================");
/* 自定义处理逻辑 */
if (reserved_bits == 0xA) {
    printk(KERN_DEBUG "add successful!!==============================================");
}
```





实验结果：发送端自定义，接收端可以解析

### 实验二：固定接收端的rwnd

```bash
sysctl net.ipv4.tcp_rmem
# 单位都是字节，命令时单次生效，重启后就失效了
sudo sysctl -w net.ipv4.tcp_rmem="4096 131072 6291456"
sudo sysctl -w net.ipv4.tcp_rmem="131072 131072 131072"
sudo sysctl -w net.ipv4.tcp_rmem="4096 4096 4096
# 永久生效
echo "net.ipv4.tcp_rmem = 4096 65536 65536" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

iperf -s  -p 2025
iperf -c 192.168.83.129 -p 2025  -u -i 1

iperf -c 192.168.38.255 -p 2025  -i 1
```

实验结果：可以调整带宽



rwnd是能影响吞吐量的，发送速率由接收端来调整，是不是会更及时的发现拥塞呢？

稳定性优先，而速率次之？

在协议栈中有那么多缓冲区，如果每一层都有数据的排队，那一定是造成了带宽的浪费，那么，尽可能是这些缓冲区数据包的排队长度较小，既有可能提高带宽的利用率，也有可能减少拥塞。

rwnd = cwnd = Max(接收端需求，接收端能力能力，发送端能力)

如果在各层之间有数据包的排队，那么说明应用层对数据的要求小于当前网络传输速率，可以适当给降低发送端的速率。 



### 实验三:在接收端各层打印队列长度

tcp_data_queue()	tcp_input.c

```c
// 检查是否是端口 2025 上的 TCP 流
    if (inet->inet_dport == htons(2025)) {
        // 打印 TCP 接收队列长度
        printk(KERN_DEBUG "TCP: Port 2025, sk_rmem_alloc=%u, sk_receive_queue length=%u\n",
               sk->sk_rmem_alloc.counter,
               sk->sk_receive_queue.qlen);
    }

```



### 实验四 ：限制虚拟机的网卡发送速率

```bash
tc qdisc show dev ens33
tc qdisc del dev ens33 root


qdisc fq_codel 0: root refcnt 2 limit 10240p flows 1024 quantum 1514 target 5.0ms interval 100.0ms memory_limit 32Mb ecn
tc qdisc add dev ens33 root tbf rate 80Mbit latency 50ms burst 500kb

tc qdisc add dev eth0 root tbf rate 30*1024Kbit latency 50ms burst 15kb
#将eth0网卡限速到500Kbit/s，15bk的buffer，TBF最多产生50ms的延迟
#tbf是Token Bucket Filter的简写，适合于把流速降低到某个值


tc qdisc add dev ens33 root fq_codel limit 10240 flows 1024 quantum 1514 target 5ms interval 100ms memory_limit 32Mb ecn

```







# 学习参考





## 一些概念

MSS = 链路层最大传输单元（MTU） - IP 头部长度 - TCP 头部长度



## 工具

### Mac安装Ubuntu

下载安装VMware

https://support.broadcom.com/group/ecx/productfiles?subFamily=VMware%20Fusion&displayGroup=VMware%20Fusion%2013%20Pro%20for%20Personal%20Use&release=13.5.2&os=&servicePk=520445&language=EN

下载ubuntu

https://releases.ubuntu.com

安装ubuntu

https://zhuanlan.zhihu.com/p/656654128

dptcp分析

https://zhuanlan.zhihu.com/p/430215470



## 参考链接

Linux2.6内核解析

https://gitcode.com/open-source-toolkit/aa76a/overview?utm_source=highlight_word_gitcode&word=Linux&isLogin=1

一在ip头部“选项”字段如何添加自定义的信息，作为自定义的匹配字段用

https://www.cnblogs.com/ssyfj/p/12179675.html#

显示拥塞控制

https://zhuanlan.zhihu.com/p/395200230

linux收包流程

https://mp.weixin.qq.com/s?__biz=MjM5Njg5NDgwNA==&mid=2247484058&idx=1&sn=a2621bc27c74b313528eefbc81ee8c0f&scene=21#wechat_redirect

# Done

- **优先级调度：pfifo_fast**





# END