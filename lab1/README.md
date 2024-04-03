# Lab 1 - Reliable Data Transport Protocol

### 1. 网络包设计

- 对于sender，包的结构如下：

  ```
  |<-  2 bytes  ->|<-  4 bytes  ->|<-  1 byte  ->|<-            the rest        ->|
  |   checksum    |    sequence   | payload size |              payload           |
  ```

- 对于receiver，需要发送ack包，包的结构如下：

  ```
  |<-  2 bytes  ->|<-  4 bytes  ->|<-                   the rest              ->|
  |   checksum    |   ack number  |                     payload                 |
  ```

- 此外，对于sender的第一个包，我们额外用4个byte表示message的大小，以便该message所有packet都收到后，可以进行合并，一起发送。

### 2. sender 收包 & 发包机制

采用**Go-Back-N的机制**，并且添加buffer进行了一定的优化。

- sender收上层的包时，会将message全部放入buffer中，并且将message处理生成packet。同时维护一个滑动窗口，当滑动窗口的数量没有超过上限时，则将buffer中的数据取出，放入滑动窗口内。

- sender收下层发来的ack包时，会将ack与当前的seq进行对比。若ack<seq，代表之前有包没有发送成功，因此进行resend操作；若ack=seq，我们将当前的包发出，并移动滑动窗口。

- sender发包时，会将message进行切割操作，每次取滑动窗口大小的包发送。若计时器超时，则会将ack与滑动窗口中的seq进行比较，重新发包。

  ```C++
  void message_convert_to_packet() // 将消息 message 转换为数据包 packet 以便于发送
  {
      int idx = current_num % BUFFER_SIZE;
      message msg = buffer[idx];
      packet pkt;
  
      while (now_size < WINDOW_SIZE && current_num < next_num)
      {
          if (msg.size - cursor > RDT_PKTSIZE - header_size)
          {
              memcpy(pkt.data + 2, &mx_seq, 4);
              pkt.data[header_size - 1] = RDT_PKTSIZE - header_size;
              memcpy(pkt.data + header_size, msg.data + cursor, RDT_PKTSIZE - header_size);
  
              short checksum = calc_checksum(&pkt);
              memcpy(pkt.data, &checksum, 2);
              memcpy(window[mx_seq % WINDOW_SIZE].data, &pkt, sizeof(packet));
              mx_seq++;
              now_size++;
              cursor += RDT_PKTSIZE - header_size;
          }
          else if (cursor < msg.size)
          {
              memcpy(pkt.data + 2, &mx_seq, 4);
              pkt.data[header_size - 1] = msg.size - cursor;
              memcpy(pkt.data + header_size, msg.data + cursor, msg.size - cursor);
  
              short checksum = calc_checksum(&pkt);
              memcpy(pkt.data, &checksum, 2);
              memcpy(window[mx_seq % WINDOW_SIZE].data, &pkt, sizeof(packet));
              current_num++;
              cursor = 0;
              num_message--;
              mx_seq++;
              now_size++;
              if (current_num < next_num)
                  msg = buffer[current_num % BUFFER_SIZE];
          }
      }
  }
  ```

### 3.receiver 收包 & 发包机制

- 当收到的包seq>ack时，我们会将该数据包存到buffer中。以便当后续收到seq=ack的包时，可以迅速移动滑动窗口，减少发包数量。

- 当seq=ack时，代表这是我们想要的包，滑动窗口右移。此时我们需要查看已经存下的buffer是否有满足条件的后续的包，若有则一起发送给上层。最后一起发送ack包。

  ```C++
  void Receiver_FromLowerLayer(struct packet *pkt)
  {
      // 检查checksum,若checksum不正确则丢包
      short checksum;
      memcpy(&checksum, pkt->data, 2);
      if (checksum != calc_checksum(pkt))
      {
          return;
      }
  
      int seq;
      memcpy(&seq, pkt->data + 2, 4);
      if (ack < seq && seq < ack + WINDOW_SIZE)
      { // 滑动窗口范围内
          int idx = seq % WINDOW_SIZE;
          if (!valid[idx])
          {
              memcpy(packet_buffer[idx].data, pkt->data, RDT_PKTSIZE);
              valid[idx] = true;
          }
          send_ack(ack - 1);// 这一函数中加锁
          return;
      }
      else if (seq != ack)
      {
          send_ack(ack - 1);
      }
  
      if (seq == ack)
      { // seq=ack, 考虑buffer中的包
          int size;
          while (true)
          {
              ack++;
              size = (int)pkt->data[header_size - 1];
  
              if (cursor == 0)
              { // 第一个包有message size, +4
                  size -= 4;
  
                  memcpy(&msg->size, pkt->data + header_size, 4);
                  msg->data = (char *)malloc(msg->size);
  
                  memcpy(msg->data + cursor, pkt->data + header_size + 4, size);
                  cursor += size;
              }
              else
              {
                  memcpy(msg->data + cursor, pkt->data + header_size, size);
                  cursor += size;
              }
  
              if (cursor == msg->size)
              { // 整个message组装好发送
                  Receiver_ToUpperLayer(msg);
                  cursor = 0;
              }
  
              int idx = ack % WINDOW_SIZE;
              if (valid[idx] == true)
              {
                  pkt = &packet_buffer[idx];
                  memcpy(&seq, pkt->data + 2, 4);
                  valid[idx] = false;
              }
              else
              {
                  send_ack(seq);
                  break;
              }
          }
      }
  ```

  

### 4. 参数设置

timeout时间设为0.3 s，滑动窗口大小设为5。

### 5. checksum计算方法

Checksum采用了tcp checksum的计算方法。首先将checksum位置0，将整个包划分成一个个16进制数，然后将这些数逐个相加，最后将得到的结果取反。由于我们的包长度为偶数，因此不需要考虑补0的问题。

对于一个packet，先将其他部分填入，最后再计算checksum。对于checksum不准确的包，我们直接丢弃即可。

代码如下所示：

```c++
static short calc_checksum(struct packet *pkt) {
    long sum = 0;
    for (int i = 2; i < RDT_PKTSIZE; i += 2) {
        sum += *(unsigned short *) (&(pkt->data[i]));
    }
    while (sum >> 16) {
        sum = (sum & 0xffff) + (sum >> 16);
    }
    return ~sum;
}
```

### 6. 测试结果

测试结果如下所示。**除了`rdt_receiver.cc`和`rdt_sender.cc`，没有对其他文件进行改动。**对于文档中给的所有样例，均进行了20次以上的测试，没有发现错误。此外，采用了`rdt_sim 1000 0.1 100 0.9 0.9 0.9 0`，来测试错误概率较高的极端情况，也没有发现错误。

对于文档中给出参考吞吐量的两个样例，我们进行了如下测试，吞吐量在合理范围内。

- rdt_sim 1000 0.1 100 0.15 0.15 0.15 0

  ```shell
  ## Reliable data transfer simulation with:
  	simulation time is 1000.000 seconds
  	average message arrival interval is 0.100 seconds
  	average message size is 100 bytes
  	average out-of-order delivery rate is 15.00%
  	average loss rate is 15.00%
  	average corrupt rate is 15.00%
  	tracing level is 0
  Please review these inputs and press <enter> to proceed.
  
  At 0.00s: sender initializing ...
  At 0.00s: receiver initializing ...
  At 1695.50s: sender finalizing ...
  At 1695.50s: receiver finalizing ...
  
  ## Simulation completed at time 1695.50s with
  	1009578 characters sent
  	1009578 characters delivered
  	43548 packets passed between the sender and the receiver
  ## Congratulations! This session is error-free, loss-free, and in order.
  ```

- rdt_sim 1000 0.1 100 0.3 0.3 0.3 0

  ```shell
  ## Reliable data transfer simulation with:
  	simulation time is 1000.000 seconds
  	average message arrival interval is 0.100 seconds
  	average message size is 100 bytes
  	average out-of-order delivery rate is 30.00%
  	average loss rate is 30.00%
  	average corrupt rate is 30.00%
  	tracing level is 0
  Please review these inputs and press <enter> to proceed.
  
  At 0.00s: sender initializing ...
  At 0.00s: receiver initializing ...
  At 3125.88s: sender finalizing ...
  At 3125.88s: receiver finalizing ...
  
  ## Simulation completed at time 3125.88s with
  	998547 characters sent
  	998547 characters delivered
  	54493 packets passed between the sender and the receiver
  ## Congratulations! This session is error-free, loss-free, and in order.
  ```

- rdt_sim 1000 0.1 100 0.9 0.9 0.9 0

  ```shell
  ## Reliable data transfer simulation with:
  	simulation time is 1000.000 seconds
  	average message arrival interval is 0.100 seconds
  	average message size is 100 bytes
  	average out-of-order delivery rate is 90.00%
  	average loss rate is 90.00%
  	average corrupt rate is 90.00%
  	tracing level is 0
  Please review these inputs and press <enter> to proceed.
  
  At 0.00s: sender initializing ...
  At 0.00s: receiver initializing ...
  At 1869462.48s: sender finalizing ...
  At 1869462.48s: receiver finalizing ...
  
  ## Simulation completed at time 1869462.48s with
  	1002692 characters sent
  	1002692 characters delivered
  	2982611 packets passed between the sender and the receiver
  ## Congratulations! This session is error-free, loss-free, and in order.
  ```

### Reference

https://www.baeldung.com/cs/networking-go-back-n-protocol

https://blog.csdn.net/breakout_alex/article/details/102515371

