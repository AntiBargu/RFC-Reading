#### 3.1. 首部格式（Header Format）

```shell
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |          Source Port          |       Destination Port        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                        Sequence Number                        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Acknowledgment Number                      |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  Data |           |U|A|P|R|S|F|                               |
   | Offset| Reserved  |R|C|S|S|Y|I|            Window             |
   |       |           |G|K|H|T|N|N|                               |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |           Checksum            |         Urgent Pointer        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Options                    |    Padding    |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                             data                              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 
                            TCP Header Format
```

- **序列号（Sequence Number）**

  这个段的第一个字节的序列号（当 SYN 标志不出现时）。如果 SYN 标志出现，序列号则为初始序列号（ISN），而第一个字节的序列号则为 ISN+1。

- **首部偏移（Data Offset）**

  TCP 首部中的 32 位的数目。用于指示数据的起始位置。

- **控制位（Control Bits）**
  - URG:  紧急指针字段有效
  - ACK:  确认字段有效
  - PSH:  推送功能
  - RST:  重置连接
  - SYN:  同步序列号
  - FIN:  发送方没有更多数据发送

- **窗口（Window）**

  从 ACK 指示的字节开始，该段的发送方愿意接受的数据量。

- **校验和（Checksum）**

  首部和数据的校验和。

- **紧急指针（Urgent Pointer）**

  是一个正偏移量，跟序列号字段中的值相加表示紧急数据最后一个字节的序列号。仅在 URG 设置时有效。

- **选项（Options）：可变长度 **

  两种情况：

  - 一个字节，表示选项类型。
  - 一个字节，表示选项类型；一个字节，表示选项长度；一个字节选项数据。

  选项列表可能比首部偏移（Data Offset）字段所指示的长度更短。需要填充 0 对齐。

  TCP 必须实现所有选项。

  目前协议定义的选项：

  ```shell
        Kind     Length    Meaning
        ----     ------    -------
         0         -       End of option list.
         1         -       No-Operation.
         2         4       Maximum Segment Size.
  ```

  - **最大报文段长度（Maximum Segment Size）**

    如果此选项存在，它会传递发送该段的 TCP 的最大接收段大小。该字段只能在初始连接请求中发送（在带有 SYN 控制位的段中）。如果未使用此选项，则允许使用任何段大小。

- **填充字段（Padding）：可变长度 **

  补 0 对齐。



#### 3.2. 术语（Terminology）

维护 TCP 连接需要记住几个变量。这些变量存储在称为传输控制块 (TCB) 的连接记录中。存储在 TCB 中的变量包括本地和远程套接字编号、连接的安全性和优先级、指向用户进程发送和接收缓冲区的指针、指向重传队列和当前段的指针。此外，与发送和接收序列号相关的几个变量也存储在 TCB 中。

发送序列变量：

- SND.UNA 发送未确认
- SND.NXT 下一个发送
- SND.WND 发送窗口
- SND.UP 发送紧急指针
- SND.WL1 上一次窗口更新的段序列号
- SND.WL2 上一次窗口更新的段确认号
- ISS 初始发送序列号

接收序列变量：

- RCV.NXT 下一个接收
- RCV.WND 接收窗口
- RCV.UP 接收紧急指针
- IRS 初始接收序列号



发送序列号空间：

```shell

                   1         2          3          4      
              ----------|----------|----------|---------- 
                     SND.UNA    SND.NXT    SND.UNA        
                                          +SND.WND        

        1 - 已确认的序列号
        2 - 未确认的序列号           
        3 - 新数据发送可以使用的序列号
        4 - 当前不允许、未来会用的序列号 
```



接收序列号空间：

```shell

                       1          2          3      
                   ----------|----------|---------- 
                          RCV.NXT    RCV.NXT        
                                    +RCV.WND        

        1 - 已确认的序列号
        2 - 待确认的序列号       
        3 - 当前不允许、未来会用的序列号  
```



当前段变量：

- SEG.SEQ 段序列号
- SEG.ACK 段确认号
- SEG.LEN 数据段长度
- SEG.WND 段窗口大小
- SEG.UP 段紧急数据指针
- SEG.PRC 段优先级



一个连接的生命周期会有很多状态，状态含义如下：

- LISTEN

  等待来自任意远程 TCP 和端口的连接请求。

- SYN-SENT

  发送连接请求后，等待匹配的连接请求。

- SYN-RECEIVED

  在接收和发送连接请求后等待确认连接请求确认。

- ESTABLISHED

  连接已建立，收到的数据可以传送给用户进程。连接数据传输阶段的正常状态。

- FIN-WAIT-1

  等待远端 TCP 的连接终止请求，或者等待之前发送的连接终止请求的确认。

- FIN-WAIT-2

  等待远端 TCP 的连接终止请求。

- CLOSE-WAIT

  等待本地用户进程的连接终止请求。

- CLOSING

  等待远程 TCP 的连接终止请求确认。

- LAST-ACK

  等待先前发送给远程 TCP 的连接终止请求的确认（其中包括其连接终止请求的确认）。

- TIME-WAIT

  等待足够的时间以确保远程 TCP 收到其连接终止请求的确认。

TCP 连接响应事件从一个状态进展到另一个状态。

由事件驱动 TCP 状态迁移。事件包括：

- 用户进程调用、OPEN、SEND、RECEIVE、CLOSE、ABORT 和 STATUS
- 段到达，特别是包含 SYN、ACK、RST 和 FIN 标志的段
- 超时



**TCP 链接状态迁移图：**

```shell
                                
                              +---------+ ---------\      active OPEN  
                              |  CLOSED |            \    -----------  
                              +---------+<---------\   \   create TCB  
                                |     ^              \   \  snd SYN    
                   passive OPEN |     |   CLOSE        \   \           
                   ------------ |     | ----------       \   \         
                    create TCB  |     | delete TCB         \   \       
                                V     |                      \   \     
                              +---------+            CLOSE    |    \   
                              |  LISTEN |          ---------- |     |  
                              +---------+          delete TCB |     |  
                   rcv SYN      |     |     SEND              |     |  
                  -----------   |     |    -------            |     V  
 +---------+      snd SYN,ACK  /       \   snd SYN          +---------+
 |         |<-----------------           ------------------>|         |
 |   SYN   |                    rcv SYN                     |   SYN   |
 |   RCVD  |<-----------------------------------------------|   SENT  |
 |         |                    snd ACK                     |         |
 |         |------------------           -------------------|         |
 +---------+   rcv ACK of SYN  \       /  rcv SYN,ACK       +---------+
   |           --------------   |     |   -----------                  
   |                  x         |     |     snd ACK                    
   |                            V     V                                
   |  CLOSE                   +---------+                              
   | -------                  |  ESTAB  |                              
   | snd FIN                  +---------+                              
   |                   CLOSE    |     |    rcv FIN                     
   V                  -------   |     |    -------                     
 +---------+          snd FIN  /       \   snd ACK          +---------+
 |  FIN    |<-----------------           ------------------>|  CLOSE  |
 | WAIT-1  |------------------                              |   WAIT  |
 +---------+          rcv FIN  \                            +---------+
   | rcv ACK of FIN   -------   |                            CLOSE  |  
   | --------------   snd ACK   |                           ------- |  
   V        x                   V                           snd FIN V  
 +---------+                  +---------+                   +---------+
 |FINWAIT-2|                  | CLOSING |                   | LAST-ACK|
 +---------+                  +---------+                   +---------+
   |                rcv ACK of FIN |                 rcv ACK of FIN |  
   |  rcv FIN       -------------- |    Timeout=2MSL -------------- |  
   |  -------              x       V    ------------        x       V  
    \ snd ACK                 +---------+delete TCB         +---------+
     ------------------------>|TIME WAIT|------------------>| CLOSED  |
                              +---------+                   +---------+

                      TCP Connection State Diagram
```



#### 3.3. 序列号（Sequence Numbers）

实际的序列号空间是有限的。对序列号采用取模处理。在协议实现的时候需要格外注意。

如果重传队列中的段的序列号和长度之和小于或等于传入段中的确认值，则该段被完全确认。

请注意，当接收窗口为零时，除了 ACK 段之外，不应接受任何段。因此，TCP 可以在传输数据和接收 ACK 时保持零接收窗口。但是，即使接收窗口为零，TCP 也必须处理所有传入段的 RST 和 URG 字段。

我们还 **利用了编号方案来保护某些控制信息 **。通过在序列空间中隐式包含一些控制标志来实现的。这样可以 **避免在重新传输和确认的时候不会出现混淆（可以防止对一个控制信息进行多次处理）**。控制信息在物理上并不包含在段数据空间中。因此，我们必须采用规则来隐式分配序列号以进行控制。**SYN 和 FIN 是唯一需要这种保护的控制，这些控制仅在连接打开和关闭时使用 **。SYN 被认为发生在它所在的段的第一个实际数据字节之前，而 FIN 被认为发生在它所在的段的最后一个实际数据字节之后。



##### 初始序列号选择（Initial Sequence Number Selection）

协议不限制重复使用某个连接。由此产生的问题是——**“TCP 如何识别来自先前连接的重复段？” 如果连接被快速连续地打开和关闭，或者连接因内存丢失而中断然后重新建立，这个问题就会变得明显 **。

创建新连接时，将使用初始序列号 (ISN) 生成器来选择新的 32 位 ISN。生成器绑定到（可能是虚构的）32 位时钟，其低位大约每 4 微秒递增一次。因此，ISN 大约每 4.55 小时循环一次。由于我们假设段在网络中停留的时间不超过最大段寿命 (MSL)，并且 MSL 小于 4.55 小时，因此我们可以合理地假设 ISN 是唯一的。

对于每个连接，都有一个发送序列号和一个接收序列号。初始发送序列号 (ISS) 由数据发送 TCP 选择，初始接收序列号 (IRS) 在连接建立过程中获知。

要建立或初始化连接，两个 TCP 必须同步彼此的初始序列号。这是通过交换连接建立段来完成的，这些段携带控制位 “SYN”（同步）和初始序列号。简言之，携带 SYN 位的段也称为 “SYN”。因此，**该解决方案需要一种合适的机制来选择初始序列号，并进行稍微复杂的握手以交换 ISN**。

同步要求每一方都发送自己的初始序列号，并接收来自另一方的确认。

```shell
1) A --> B SYN my sequence number is X
2) A <-- B ACK your sequence number is X
3) A <-- B SYN my sequence number is Y
4) A --> B ACK your sequence number is Y
```

2) 和 3) 可以合并成一步，所以一般称之为三次握手。

三次握手是必要的，因为序列号与网络中的全局时钟无关，并且 TCP 可能具有不同的 ISN 选择机制。第一个 SYN 的接收方无法知道该段是否是旧的延迟段，除非它记得连接上使用的最后一个序列号（这并不总是可能的），因此它必须要求发送方验证此 SYN。



##### 知道什么时候保持静默（Knowing When to Keep Quiet）

为确保 TCP 不会创建一个携带可能被网络中残留的旧段重复的序列号的段。TCP 必须在启动或从丢失正在使用的序列号的崩溃中恢复期间，分配任何序列号之前保持静默，最长为一个最大段生存期（MSL）。在本规范中，MSL 被设定为 2 分钟。这是一个工程选择，如果经验表明有必要，可则以更改。



##### TCP 静默时间概念（The TCP Quiet Time Concept）

如果主机 “崩溃” 且无法保留每个活跃（即未关闭）连接上传输的最后序列号，则应延迟发送任何 TCP 段，延迟时间至少为主机所属的互联网系统中商定的最大段生存期 (MSL)。

TCP 实现可以违反 “安静时间” 限制，但这样做的风险是，互联网系统中的某些接收方会将某些旧数据视为新数据或将新数据视为重复的旧数据而拒绝接收。

问题在于恢复主机可能不知道它崩溃了多长时间，也不知道系统中是否还存在早期连接中的旧重复项。

解决此问题的一种方法是故意在从崩溃中恢复后延迟一个 MSL 发送段 - 这就是 “静默时间” 规范。主机如果希望避免等待，愿意冒着在给定目的地可能混淆旧数据包和新数据包的风险，可以选择不等待“静默时间”。实施者可以为 TCP 用户进程提供在连接基础上选择是否在崩溃后等待的能力，或者可以为所有连接非正式地实现“安静时间”。

总结：每个段占用一个或多个序列号。段占用的序列号处于 “忙” 或“正在使用”状态直到 MSL 过去。崩溃时，最后发出的段会占用序列号空间。如果新连接启动得太快，会使用上一个连接最后一个段的序列号空间，存在潜在的序列号重叠区域，导致接收方感到混乱。



##### 建立连接（Establishing a connection）

“三次握手”是用于建立连接的过程。通常该过程由一个 TCP 发起，另一个 TCP 响应。如果两个 TCP 同时发起该过程，该过程也能正常工作。当同时尝试发生时，每个 TCP 在发送 “SYN” 之后都会收到一个不带确认的 “SYN” 段。

旧的重复 “SYN” 段的到达可能让接收方误以为正在进行同时的连接建立。正确使用“重置”（reset）段可以消除这些情况的歧义。

###### 经典的 “三次握手” 同步

```shell
TCP A                                                TCP B

  1.  CLOSED                                               LISTEN

  2.  SYN-SENT    --> <SEQ=100><CTL=SYN>               --> SYN-RECEIVED

  3.  ESTABLISHED <-- <SEQ=300><ACK=101><CTL=SYN,ACK>  <-- SYN-RECEIVED

  4.  ESTABLISHED --> <SEQ=101><ACK=301><CTL=ACK>       --> ESTABLISHED

  5.  ESTABLISHED --> <SEQ=101><ACK=301><CTL=ACK><DATA> --> ESTABLISHED

          Basic 3-Way Handshake for Connection Synchronization

```

ACK 不占用序列号空间（如果占用，我们最终将对 ACK 进行确认！）。



###### 同时建立连接

```shell

      TCP A                                            TCP B

  1.  CLOSED                                           CLOSED

  2.  SYN-SENT     --> <SEQ=100><CTL=SYN>              ...

  3.  SYN-RECEIVED <-- <SEQ=300><CTL=SYN>              <-- SYN-SENT

  4.               ... <SEQ=100><CTL=SYN>              --> SYN-RECEIVED

  5.  SYN-RECEIVED --> <SEQ=100><ACK=301><CTL=SYN,ACK> ...

  6.  ESTABLISHED  <-- <SEQ=300><ACK=101><CTL=SYN,ACK> <-- SYN-RECEIVED

  7.               ... <SEQ=101><ACK=301><CTL=ACK>     --> ESTABLISHED

                Simultaneous Connection Synchronization
```

**三次握手的主要原因是防止旧的重复连接请求引起混乱 **。为了解决这个问题，设计了一种特殊的控制消息，称为 “重置”（reset）。

如果接收的 TCP 处于非同步状态（例如 SYN-SENT、SYN-RECEIVED），在接收到一个可接受的重置后，它会返回到 LISTEN 状态。

如果 TCP 处于同步状态之一（例如 ESTABLISHED、FIN-WAIT-1、FIN-WAIT-2、CLOSE-WAIT、CLOSING、LAST-ACK、TIME-WAIT），它将中止连接并通知其用户进程。

下面讨论这种半开放连接的情况。



###### 从旧的重复 SYN 中恢复

```shell

      TCP A                                                TCP B

  1.  CLOSED                                               LISTEN

  2.  SYN-SENT    --> <SEQ=100><CTL=SYN>               ...

  3.  (duplicate) ... <SEQ=90><CTL=SYN>               --> SYN-RECEIVED

  4.  SYN-SENT    <-- <SEQ=300><ACK=91><CTL=SYN,ACK>  <-- SYN-RECEIVED

  5.  SYN-SENT    --> <SEQ=91><CTL=RST>               --> LISTEN
  

  6.              ... <SEQ=100><CTL=SYN>               --> SYN-RECEIVED

  7.  SYN-SENT    <-- <SEQ=400><ACK=101><CTL=SYN,ACK>  <-- SYN-RECEIVED

  8.  ESTABLISHED --> <SEQ=101><ACK=401><CTL=ACK>      --> ESTABLISHED

                    Recovery from Old Duplicate SYN
```



##### 半开连接和其他异常（Half-Open Connections and Other Anomalies）

如果 TCP 的一端在另一端不知情的情况下关闭或中止了连接，或者由于崩溃导致内存丢失，连接的两端变得不同步，则已建立的连接被称为 “半开”。如果尝试向任意方向发送数据，这种连接将会自动被重置。然而，半开放连接预期是罕见的，其恢复过程也相对简单。

如果 A 上的连接不再存在，那么 B 试图发送给 A 任何数据时，B 的 TCP 将会收到一个重置控制消息。该消息用于通知 BTCP 存在某种问题，并应中止连接。

假设当 A 和 B 两个用户进程正在相互通信时，A 发生崩溃并导致其 TCP 连接内存丢失。根据支持 A 的 TCP 的操作系统，可能存在某些错误恢复机制。当 TCP 重新启动时，A 可能会从头开始或从某个恢复点开始。可能会再次尝试打开连接，或者尝试在其认为已打开的连接上发送数据。在后一种情况下，它会从本地（A 的）TCP 收到 “连接未打开” 的错误消息。为了建立连接，A 的 TCP 将发送一个包含 SYN 的段。

###### 半链接发现与恢复

```shell

      TCP A                                           TCP B

  1.  (CRASH)                               (send 300,receive 100)

  2.  CLOSED                                           ESTABLISHED

  3.  SYN-SENT --> <SEQ=400><CTL=SYN>              --> (??)

  4.  (!!)     <-- <SEQ=300><ACK=100><CTL=ACK>     <-- ESTABLISHED

  5.  SYN-SENT --> <SEQ=100><CTL=RST>              --> (Abort!!)

  6.  SYN-SENT                                         CLOSED

  7.  SYN-SENT --> <SEQ=400><CTL=SYN>              -->

                     Half-Open Connection Discovery
```

7 行后重新三次握手。



###### 主动方发现半链接

```shell
        TCP A                                              TCP B

  1.  (CRASH)                                   (send 300,receive 100)

  2.  (??)    <-- <SEQ=300><ACK=100><DATA=10><CTL=ACK> <-- ESTABLISHED

  3.          --> <SEQ=100><CTL=RST>                   --> (ABORT!!)

           Active Side Causes Half-Open Connection Discovery
```



###### 旧的重复的 SYN 重置两个被动套接字

```shell

      TCP A                                         TCP B

  1.  LISTEN                                        LISTEN

  2.       ... <SEQ=Z><CTL=SYN>                -->  SYN-RECEIVED

  3.  (??) <-- <SEQ=X><ACK=Z+1><CTL=SYN,ACK>   <--  SYN-RECEIVED

  4.       --> <SEQ=Z+1><CTL=RST>              -->  (return to LISTEN!)

  5.  LISTEN                                        LISTEN

       Old Duplicate SYN Initiates a Reset on two Passive Sockets

```



##### 重置的产生（Reset Generation）

通常，每当一个明显不属于当前连接的段到达时，必须发送重置（RST）。如果情况不明确，则不应发送重置。

有三种状态：

1. 如果连接不存在（CLOSED），则会对任何传入的段（除重置段外）发送重置（RST）。如果传入段有 ACK 字段，重置会从该段的 ACK 字段中获取序列号；否则，重置的序列号为零，ACK 字段将设置为传入段的序列号与段长度之和。连接状态保持为 CLOSED。

2.  如果连接处于任何非同步状态（LISTEN、SYN-SENT、SYN-RECEIVED），并且传入的段确认了一些尚未发送的数据（即段携带了一个不可接受的 ACK）或者传入的段与连接请求不匹配，则会发送一个重置（RST）。

   如果传入段有 ACK 字段，重置会从该段的 ACK 字段中获取序列号，否则，重置的序列号为零，ACK 字段将设置为传入段的序列号与段长度之和。连接状态保持不变。

3. 如果连接处于同步状态（ESTABLISHED、FIN-WAIT-1、FIN-WAIT-2、CLOSE-WAIT、CLOSING、LAST-ACK、TIME-WAIT），任何不可接受的段（窗口外的序列号或不可接受的 ACK）都只会触发一个空的确认段，该确认段包含当前发送序列号和一个指示下一个期望接收的序列号的确认信息，且连接状态保持不变。

   如果传入的段具有与连接请求的安全级别、分区或优先级不完全匹配的属性，则会发送一个重置（RST），并且连接进入 CLOSED 状态。重置的序列号从传入段的 ACK 字段中获取。



##### RESET 处理（Reset Processing）

在除 SYN-SENT 以外的所有状态中，所有重置（RST）段都通过检查其 SEQ 字段进行验证。若其序列号在窗口内，则该重置是有效的。在 SYN-SENT 状态中（即 RST 是对初始 SYN 的响应时），如果 ACK 字段确认了 SYN，则该 RST 是可接受的。

RST 的接收方首先会验证该 RST，然后改变状态。如果接收方处于 LISTEN 状态，则忽略该 RST。如果接收方处于 SYN-RECEIVED 状态，并且之前曾处于 LISTEN 状态，则接收方返回到 LISTEN 状态，否则接收方会中止连接并进入 CLOSED 状态。如果接收方处于任何其他状态，则它中止连接，通知用户进程，并进入 CLOSED 状态。



#### 3.5. 关闭连接（Closing a Connection）

CLOSE 操作的意思是 “我没有更多数据要发送”。我们选择以单工方式处理 CLOSE。关闭的用户进程可以继续接收，直到他被告知另一方也已关闭。

基本上有三种情况：

1. 主动关闭

   构造一个 FIN 段并将其放入传出段队列中。TCP 将不再接受来自用户进程的任何发送请求，并进入 FIN-WAIT-1 状态。在该状态下，接收操作仍然被允许。所有在 FIN 之前的段（包括 FIN 段）都将被重传，直到得到确认。当另一方的 TCP 既确认了 FIN 又发送了它自己的 FIN 时，第一个 TCP 可以确认该 FIN。注意，接收到 FIN 的 TCP 会确认 FIN，但不会发送自己的 FIN，直到其用户进程也关闭了连接。

2. 被动关闭

   如果 FIN 从网络到达，接收方的 TCP 可以确认（ACK）它，并通知用户进程连接正在关闭。用户进程会响应一个 CLOSE，TCP 在发送任何剩余数据后，便可以向对方 TCP 发送 FIN。然后，TCP 等待它自己的 FIN 被确认，确认后它删除连接。如果未能收到 ACK，超时后，连接将被中止，并通知用户进程。

3. 同时关闭连接

   当连接两端的用户进程同时关闭连接时，会交换 FIN 段。当所有在 FIN 段之前的段都已被处理并得到确认后，每个 TCP 都可以确认（ACK）它收到的 FIN。双方在收到这些 ACK 后，都会删除连接。



##### 正常关闭（Normal Close Sequence）

```shell

      TCP A                                                TCP B

  1.  ESTABLISHED                                          ESTABLISHED

  2.  (Close)
      FIN-WAIT-1  --> <SEQ=100><ACK=300><CTL=FIN,ACK>  --> CLOSE-WAIT

  3.  FIN-WAIT-2  <-- <SEQ=300><ACK=101><CTL=ACK>      <-- CLOSE-WAIT

  4.                                                       (Close)
      TIME-WAIT   <-- <SEQ=300><ACK=101><CTL=FIN,ACK>  <-- LAST-ACK

  5.  TIME-WAIT   --> <SEQ=101><ACK=301><CTL=ACK>      --> CLOSED

  6.  (2 MSL)
      CLOSED                                                      

                         Normal Close Sequence
```



##### 同时关闭（ Simultaneous Close Sequence）

```shell

      TCP A                                                TCP B

  1.  ESTABLISHED                                          ESTABLISHED

  2.  (Close)                                              (Close)
      FIN-WAIT-1  --> <SEQ=100><ACK=300><CTL=FIN,ACK>  ... FIN-WAIT-1
                  <-- <SEQ=300><ACK=100><CTL=FIN,ACK>  <--
                  ... <SEQ=100><ACK=300><CTL=FIN,ACK>  -->

  3.  CLOSING     --> <SEQ=101><ACK=301><CTL=ACK>      ... CLOSING
                  <-- <SEQ=301><ACK=101><CTL=ACK>      <--
                  ... <SEQ=101><ACK=301><CTL=ACK>      -->

  4.  TIME-WAIT                                            TIME-WAIT
      (2 MSL)                                              (2 MSL)
      CLOSED                                               CLOSED

                      Simultaneous Close Sequence
```



#### 3.7. 数据交互（Data Communication）

一旦连接建立，数据通过段的交换进行通信。由于错误（如校验和测试失败）或网络拥塞，段可能会丢失，因此 TCP 使用超时重传机制来确保每个段的成功传输。由于网络或 TCP 的重传机制，可能会收到重复的段。正如在序列号章节中讨论的那样，TCP 会对段中的序列号和确认号执行某些检查，以验证它们的可接受性。

请注意，一旦进入 ESTABLISHED 状态，所有段必须携带当前的确认信息。



#####   窗口管理（Managing the Window）

每个段中发送的窗口指示了窗口的发送方（即数据接收方）当前准备接受的序列号范围。与此连接当前可用的数据缓冲区空间相关。

较大的窗口会鼓励传输。如果到达的数据超过了可以接受的范围，它将被丢弃。这会导致过多的重传，给网络和 TCP 增加不必要的负担。较小的窗口可能会限制数据的传输，以至于每个新段传输之间引入一个往返延迟。

提供的机制允许 TCP 通告一个较大的窗口，然后在未接受那么多数据的情况下通告一个更小的窗口。这种所谓的 “缩小窗口” 操作是强烈不推荐的。稳健性原则规定，TCP 自身不会缩小窗口，但会为其他 TCP 执行此类行为做好准备。

为了避免报告窗口信息冲突，根据携带最高确认号的段上的窗口信息进行操作（即确认号等于或大于先前接收到的最高确认号的段）。

确认信息不应被延迟，否则会导致不必要的重传。



#### 3.8. 接口（Interfaces）

##### 用户进程 / TCP 接口（User/TCP Interface）

不同的 TCP 实现可能具有不同的用户接口。但是，所有 TCP 都必须提供一组最低限度的服务，以保证所有 TCP 实现都可以支持相同的协议层次结构。



##### TCP 原语（TCP User Commands）

下述指令指定了 TCP 必须执行的基本功能，以支持进程间通信。



###### Open

```shell
Format: OPEN (local port, foreign socket, active/passive
[, timeout] [, precedence] [, security/compartment] [, options]) -> local connection name
```

如果 active/passive 标志设置为 passive，那么这是一个用于 LISTEN 连接的调用。

传输控制块（TCB）会被创建，并使用 OPEN 的参数填充 TCB 部分字段的值。

在 active OPEN 命令中，TCP 会立即开始同步（即建立）连接。

TCP 将向用户返回本地连接名。本地连接名随后可以用作对由 <本地套接字，远程套接字> 对定义的连接的简称。



###### Send

```shell
Format:  SEND (local connection name, buffer address, byte
        count, PUSH flag, URGENT flag [,timeout])
```

指定缓冲区中包含的数据在指定连接上发送。如果连接尚未打开，则发送操作将被视为错误。某些实现可能允许用户先发送数据；在这种情况下，将自动执行 OPEN 操作。如果调用进程未被授权使用此连接，则会返回错误。

多个 SEND 操作按先来先服务的顺序进行处理，因此 TCP 会对无法立即服务的 SEND 操作进行排队。



###### Receive

```shell
Format: RECEIVE (local connection name, buffer address, byte count) -> byte count, urgent flag, push flag
```

此命令分配一个与指定连接关联的接收缓冲区。如果在此命令之前没有 OPEN 操作，或者调用进程没有权限使用此连接，将返回一个错误。



###### Close

```shell
 Format:  CLOSE (local connection name)
```

此命令导致指定的连接被关闭。如果连接没有打开，或者调用进程未被授权使用该连接，则会返回错误。

关闭连接应该是一种优雅的操作，意味着尚未发送的发送（SEND）将被传输（并重新传输），直到所有请求都被服务。

关闭意味着 “我没有更多要发送的数据”，但并不意味着 “我将不再接收任何数据”。

关闭连接需要与外部 TCP 进行通信，连接可能会在关闭状态下保持一段时间。在 TCP 回复关闭命令之前尝试重新打开连接会导致错误响应。



###### Status

```shell
Format: STATUS (local connection name) -> status data
```

这是一个依赖于实现的用户命令。通常返回与连接相关联的 TCB 的信息。



###### Abort

```shell
Format:  ABORT (local connection name)
```

这个命令会导致所有待处理的发送和接收操作被中止，TCP 控制块被移除，并向连接另一侧的 TCP 发送复位（RESET）消息。



#### 3.9. 事件处理（Event Processing）

事件可以分为三类：

- 用户进程调用
  - OPEN
  - SEND
  - RECEIVE
  - CLOSE
  - ABORT
  - STATUS
- 段到达
- 超时
  - 用户超时
  - 重传超时
  - TIME-WAIT 超时



##### 用户进程调用

###### OPEN 调用

- CLOSED STATE

  创建一个新的传输控制块（TCB）来保存连接状态信息。

- LISTEN STATE

  连接从被动状态改为主动状态，选择一个初始发送序列号（ISS）。发送 SYN 段。

- SYN-SENT STATE
  SYN-RECEIVED STATE
  ESTABLISHED STATE
  FIN-WAIT-1 STATE
  FIN-WAIT-2 STATE
  CLOSE-WAIT STATE
  CLOSING STATE
  LAST-ACK STATE
  TIME-WAIT STATE

  "error:  connection already exists".



###### SEND 调用

- CLOSED STATE

  - 无权访问

    "error: connection illegal for this process".

  - 其他情形

    "error: connection does not exist".

- LISTEN STATE

  连接从被动状态改为主动状态，选择一个初始发送序列号（ISS）。发送 SYN 段。

- SYN-SENT STATE
  SYN-RECEIVED STATE

  将数据排队以在进入建立状态后传输。

  如果没有足够的空间来排队，"error: insufficient resources".

- ESTABLISHED STATE
  CLOSE-WAIT STATE

  将缓冲区分段并与带有 ACK（ACK 值 = RCV.NXT）的数据一起发送。

  如果没有足够的空间来排队，"error: insufficient resources".

- FIN-WAIT-1 STATE
  FIN-WAIT-2 STATE
  CLOSING STATE
  LAST-ACK STATE
  TIME-WAIT STATE

  "error:  connection closing" 



###### RECEIVE 调用

- CLOSED STATE

  - 无权访问

    "error: connection illegal for this process".

  - 其他情形

    "error: connection does not exist".

- LISTEN STATE
  SYN-SENT STATE
  SYN-RECEIVED STATE

  将数据排队以在进入建立状态后接收。

  如果没有足够的空间来排队，"error: insufficient resources".

- ESTABLISHED STATE
  FIN-WAIT-1 STATE
  FIN-WAIT-2 STATE

  将数据排队以在进入建立状态后接收。

  如果没有足够的空间来排队，"error: insufficient resources".

- CLOSE-WAIT STATE

  由于发送端已经发送了 FIN，RECEIVE 处理 FIN 之前已经接收的数据段。

  如果没有这样的数据段，"error: connection closing".

- CLOSING STATE
  LAST-ACK STATE
  TIME-WAIT STATE

  "error: connection closing".



###### CLOSE 调用

- CLOSED STATE

  - 无权访问

    "error: connection illegal for this process".

  - 其他情形

    "error: connection does not exist".

- LISTEN STATE

  任何未完成的 RECEIVE 都将返回 "error:  closing"。

  删除 TCB（传输控制块），进入 CLOSED 状态，然后返回。

- SYN-SENT STATE

  删除 TCB（传输控制块），并向任何排队的 SEND 或 RECEIVE 返回 "error:  closing"。

- SYN-RECEIVED STATE

  如果没有发出任何 SEND，并且没有待发送的数据， 则形成一个 FIN 段并发送它，然后进入 FIN-WAIT-1 状态； 否则，进入 ESTABLISHED 状态后排队等待处理。

- ESTABLISHED STATE

  排队，直到所有之前的 SEND 操作都已分段，然后形成一个 FIN 段并发送它。进入 FIN-WAIT-1 状态。

- FIN-WAIT-1 STATE
  FIN-WAIT-2 STATE

  严格来说，这是一个错误，应该收到 "error: connection closing"

  如果第二个 FIN 没有发送（第一个 FIN 可能会被重新发送），则 “ok” 响应也是可以接受的。

- CLOSE-WAIT STATE

  将此请求排队，直到所有之前的 SEND 操作都已分段；然后发送一个 FIN 段，并进入 CLOSING 状态。

  > 应该是进入 LAST-ACK 状态。RFC 似乎有错误。

- CLOSING STATE
  LAST-ACK STATE
  TIME-WAIT STATE

  "error: connection closing".



###### ABORT 调用

- CLOSED STATE

  - 无权访问

    "error: connection illegal for this process".

  - 其他情形

    "error: connection does not exist".

- LISTEN STATE

  任何未完成的 RECEIVE 都将返回 "error:  closing"。

  删除 TCB（传输控制块），进入 CLOSED 状态，然后返回。

- SYN-SENT STATE

  删除 TCB（传输控制块），所有排队的 SEND 或 RECEIVE 收到 "connection reset" 通知。

- SYN-RECEIVED STATE
  ESTABLISHED STATE
  FIN-WAIT-1 STATE
  FIN-WAIT-2 STATE
  CLOSE-WAIT STATE

  发送 RST 段：

  ```shell
  <SEQ=SND.NXT><CTL=RST>
  ```

  所有排队的 SEND 或 RECEIVE 收到 "connection reset" 通知。将所有排队等待传输（除了上面的 RST）或重传的段都刷新，删除 TCB（传输控制块），进入 CLOSED 状态，然后返回。

- CLOSING STATE
  LAST-ACK STATE
  TIME-WAIT STATE

  删除 TCB（传输控制块），进入 CLOSED 状态，然后返回。



##### 段到达

- CLOSED STATE

  所有传入段中的数据都被丢弃。包含 RST 的传入段也被丢弃。不包含 RST 的传入段将导致发送 RST 作为响应。

  如果 ACK 没有设置，使用序列号 0：

  ```shell
  <SEQ=0><ACK=SEG.SEQ+SEG.LEN><CTL=RST,ACK>
  ```

  如果设置了 ACK：

  ```shell
  <SEQ=SEG.ACK><CTL=RST>
  ```

- LISTEN STATE

  1. 检查 RST

     RST 忽略。

  2. 检查 ACK

     任何到达处于 LISTEN 状态的连接的 ACK 都是不好的。对于任何带有 ACK 的到达段应该形成一个可接受的复位段。

     ```shell
     <SEQ=SEG.ACK><CTL=RST>
     ```

  3. 检查 SYN

     如果 SYN 位被设置，检查安全性。

  4. 检查其他控制信息

     任何其他的控制或携带文本的段（不包含 SYN）必须带有 ACK，因此会被 ACK 处理丢弃。传入的 RST 段可能无效，因为它不可能是对该连接实例发送的任何内容的响应。所以你不太可能到达这里，但如果确实到达了，请丢弃该段并返回。

- SYN-SENT STATE

  1. 检查 ACK

     如果 SEG.ACK =<ISS 或者 SEG.ACK> SND.NXT，发送 RST：

     ```shell
     <SEQ=SEG.ACK><CTL=RST>
     ```

     如果 SND.UNA =< SEG.ACK =< SND.NXT，ACK 可被接受。

  2. 检查 RST

     如果 ACK 是可接受的，那么向用户返回 "error: connection reset"，丢弃该段，进入 CLOSED 状态，删除 TCB，并返回。否则（没有 ACK），丢弃该段并返回。

  3. 检查安全性 / 优先级

  4. 检查 SYN

     RCV.NXT 被设置为 SEG.SEQ+1，IRS 被设置为 SEG.SEQ。如果有 ACK，则 SND.UNA 应该前进到等于 SEG.ACK，并且因此被确认的任何重传队列中的段应该被移除。

     如果 SND.UNA > ISS（我们的 SYN 已被 ACK），则将连接状态更改为 ESTABLISHED，形成一个 ACK 段并发送它：

     ```shell
     <SEQ=SND.NXT><ACK=RCV.NXT><CTL=ACK>
     ```

     如果段中有其他控制或文本，则继续处理下面检查 URG 位。否则进入 SYN-RECEIVED 状态，形成一个 SYN,ACK 段并发送它。

     ```shell
     <SEQ=ISS><ACK=RCV.NXT><CTL=SYN,ACK>
     ```

     如果段中有其他控制或文本，则将它们排队等待在达到 ESTABLISHED 状态后处理，然后返回。

  5. 如果既未设置 SYN 位也未设置 RST 位，则丢弃该段并返回。

- 其他状态

  1. 检查序列号

     - SYN-RECEIVED STATE
       ESTABLISHED STATE
       FIN-WAIT-1 STATE
       FIN-WAIT-2 STATE
       CLOSE-WAIT STATE
       CLOSING STATE
       LAST-ACK STATE
       TIME-WAIT STATE

     分段按顺序进行处理。到达时的初始测试用于丢弃旧的重复项。进一步处理按照 SEG.SEQ 的顺序进行。

     可接受性测试有 4 种情况：

     ```shell
     Segment Receive  Test
     Length  Window
     ------- -------  -------------------------------------------
     
        0       0     SEG.SEQ = RCV.NXT
     
        0      >0     RCV.NXT =< SEG.SEQ < RCV.NXT+RCV.WND
     
       >0       0     not acceptable
     
       >0      >0     RCV.NXT =< SEG.SEQ < RCV.NXT+RCV.WND
                   or RCV.NXT =< SEG.SEQ+SEG.LEN-1 < RCV.NXT+RCV.WND
     ```

     如果 RCV.WND 为零，则不接受任何段，有效的 ACK、URG 和 RST 除外。

     如果传入的段不可接受，则应该回复一个 ACK（除非设置了 RST 位，如果是，则丢弃该段并返回）。发送确认后，丢弃不可接受的段并返回。

     ```shell
      <SEQ=SND.NXT><ACK=RCV.NXT><CTL=ACK>
     ```

  2. 检查 RST

     - SYN-RECEIVED STATE

       如果此连接是通过被动打开（即来自 LISTEN 状态）发起的，则将此连接返回到 LISTEN 状态并返回。不需要通知用户。如果此连接是通过主动打开（即来自 SYN-SENT 状态）发起的，则连接被拒绝，并向用户发送 "connection refused"。在任一情况下，应删除重传队列中的所有段。在主动打开的情况下，进入 CLOSED 状态并删除 TCB，然后返回。

     - ESTABLISHED
       FIN-WAIT-1
       FIN-WAIT-2
       CLOSE-WAIT

       如果 RST 位被设置，则任何未完成的接收和发送应接收到 “复位” 响应。所有段队列应被清空。用户还应接收到一条未经请求的一般 "connection reset" 信号。进入 CLOSED 状态，删除 TCB，然后返回。

     - CLOSING STATE
       LAST-ACK STATE
       TIME-WAIT

       如果 RST 位被设置，则进入 CLOSED 状态，删除 TCB，然后返回。

  3. 检查优先级

  4. 检查 SYN

     - SYN-RECEIVED
       ESTABLISHED STATE
       FIN-WAIT STATE-1
       FIN-WAIT STATE-2
       CLOSE-WAIT STATE
       CLOSING STATE
       LAST-ACK STATE
       TIME-WAIT STATE

       如果 SYN 在窗口内，则这是一个错误，发送复位。任何未完成的接收和发送应接收到 “复位” 响应，所有段队列应被清空，用户还应接收到一条未经请求的一般 "connection reset" 信号。进入 CLOSED 状态，删除 TCB，然后返回。

  5. 检查 ACK

     如果没有设置 ACK，丢弃该段。

     如果设置 ACK：

     - SYN-RECEIVED STATE

       如果 SND.UNA =< SEG.ACK =< SND.NXT，进入 ESTABLISHED 状态，继续处理。

       如果 ACK 不可接受，发送 RST 段：

       ```shell
        <SEQ=SEG.ACK><CTL=RST>
       ```

     - ESTABLISHED STATE

       检查合法性。

     - FIN-WAIT-1 STATE

       如果我们的 FIN 现在得到确认，则进入 FIN-WAIT-2 状态。

     - FIN-WAIT-2 STATE

       如果重传队列为空，则可以确认用户的 CLOSE，但不要删除 TCB。

     - CLOSE-WAIT STATE

       执行与 ESTABLISHED 状态相同的处理。

     - CLOSING STATE

       如果 ACK 确认我们的 FIN，则进入 TIME-WAIT 状态；否则，忽略该段。

     -  LAST-ACK STATE

       如果我们的 FIN 现在得到确认，则删除 TCB，进入 CLOSED 状态，然后返回。

     - TIME-WAIT STATE

       在此状态下能收到的唯一内容是远端 FIN 的重传。确认它，并重新启动 2MSL 超时计时器。

  6. 检查 URG

  7. 处理用户数据

  8. 检查 FIN

     如果状态为 CLOSED、LISTEN 或 SYN-SENT，则不要处理 FIN，因为无法验证 SEG.SEQ；丢弃该段并返回。

     - SYN-RECEIVED STATE
       ESTABLISHED STATE

       进入 CLOSE-WAIT 状态。

     - FIN-WAIT-1 STATE

       如果我们的 FIN 已经被确认（也许在这个段中），那么进入 TIME-WAIT 状态，启动 2MSL 计时器，关闭其他计时器；否则进入 CLOSING 状态。

     - FIN-WAIT-2 STATE

       进入 TIME-WAIT 状态。启动 2MSL 计时器，关闭其他计时器。

     - CLOSE-WAIT STATE

       保持 CLOSE-WAIT 状态。

     - CLOSING STATE

       保持 CLOSING 状态。

     - LAST-ACK STATE

       保持 LAST-ACK 状态。

     - TIME-WAIT STATE

       保持 TIME-WAIT 状态。重置 2MSL 计时器。

##### 超时

- 用户调用超时

  清空所有队列，向用户发出 “error:  connection aborted due to user timeout” 的信号，并对所有未完成的调用进行处理，删除 TCB，进入 CLOSED 状态，然后返回。

- 重传超时

  重新发送重传队列前面的段，重新初始化重传计时器，然后返回。

- TIME-WAIT 超时

  删除 TCB，进入 CLOSED 状态，然后返回。