title: LwIP的TCP实现之二
author: 钱晓豪
date: 2019-11-07 15:31:08
tags:
---
本文描述输入输出函数`tcp_input`和`tcp_output`

# TCP处理过程
## 概览

![TCP处理过程](\images\posts\20191107LwIP_TCP\tcp_process.png "TCP处理过程")

基本的 TCP 处理过程被分割为六个功能函数来实现:
* tcp_input()
* tcp_process()
* tcp_receive
* tcp_write()
* tcp_enqueue()
* tcp_output()

## 数据发送
现在，让我们先看看数据发送的过程是如何进行的。如上图所示，整个过程的发起者是应用层。应用层调用`tcp_write()`函数， 接着`tcp_write()`函数再将控制权交给`tcp_enqueue()`函数，这个函数会在必要时将数据分割为适当大小的 TCP 段，然后再把这些 TCP 段放到所属连接的传输队列中。这时，`tcp_output()`函数会检查现在是不是能够发送数据，也就是判断接收器窗口是否拥有足够大的空间，阻塞窗口是否也足够大，如果条件满足，就使用`ip_route()`及`ip_output_if()`函数发送数据。

## 数据接收
接下来，我们再看看数据的接收过程。如图所示，过程的发起者是网络接口层，这里不做描述。网络接口层将数据包传递给`ip_input()`函数，该函数验证IP头后移交TCP段给`tcp_input()`函数。`tcp_input()`函数完成两项工作：
* 其一，初始完整性检查（也就是校验和验证与 TCP 选项解析）；
* 其二，判定这个 TCP 段属于哪个 TCP 连接。

接着，这个TCP段到达`tcp_process()`函数，这个函数实现了 TCP 状态机，任何必要的状态转换在这里实现。当该TCP所属的连接正处于接受网络数据的状态，`tcp_receive()`函数将被调用。最终，`tcp_receive()`函数将数据传给上层的应用程序，完成接收过程。如果这个TCP段由一个不被承认的ACK应答数据构成，数据将会从缓冲区移走，它所占用的存储区被收回。同样，如果收到一个ACK应答确认数据，接收器同意接收更多的数据，`tcp_output()`函数将会被调用。

# tcp_input()
## 函数功能
* 提取TCP头部
* 调用tcp_process()
* TCP状态机实现
* 在ip_input()中调用此函数

## 函数返回值
* @param p收到要处理的TCP段（p->payload指向IP头部）
* @param inp网络接口（代表哪块收到的数据）
  
## 函数源码
```
void
tcp_input(struct pbuf *p, struct netif *inp)
{
  struct tcp_pcb *pcb, *prev;//那个PCB接收到数据就放到链表首部，提高遍历效率
  struct tcp_pcb_listen *lpcb;
#if SO_REUSE
  struct tcp_pcb *lpcb_prev = NULL;
  struct tcp_pcb_listen *lpcb_any = NULL;
#endif /* SO_REUSE */
  u8_t hdrlen;
  err_t err;

  PERF_START;

  TCP_STATS_INC(tcp.recv);//32位计数接收到的数据包

  iphdr = (struct ip_hdr *)p->payload;//ip头
  tcphdr = (struct tcp_hdr *)((u8_t *)p->payload + IPH_HL(iphdr) * 4);//tcp头（不知道为什么这么算）

#if TCP_INPUT_DEBUG
  tcp_debug_print(tcphdr);
#endif

  /* 移除IP头部（调整p->payload指向） */
  if (pbuf_header(p, -((s16_t)(IPH_HL(iphdr) * 4))) || (p->tot_len < sizeof(struct tcp_hdr))) {
    /* drop short packets */
    LWIP_DEBUGF(TCP_INPUT_DEBUG, ("tcp_input: short packet (%"U16_F" bytes) discarded\n", p->tot_len));
    TCP_STATS_INC(tcp.lenerr);
    goto dropped;
  }

  /* 不处理广播多播数据包 */
  if (ip_addr_isbroadcast(&current_iphdr_dest, inp) ||
      ip_addr_ismulticast(&current_iphdr_dest)) {
    TCP_STATS_INC(tcp.proterr);
    goto dropped;
  }

#if CHECKSUM_CHECK_TCP
  /* 检查校验和字段，可能在IP层保存了IP头部字段 */
  if (inet_chksum_pseudo(p, ip_current_src_addr(), ip_current_dest_addr(),
      IP_PROTO_TCP, p->tot_len) != 0) {//校验需要IP地址字段
      LWIP_DEBUGF(TCP_INPUT_DEBUG, ("tcp_input: packet discarded due to failing checksum 0x%04"X16_F"\n",
        inet_chksum_pseudo(p, ip_current_src_addr(), ip_current_dest_addr(),
      IP_PROTO_TCP, p->tot_len)));
#if TCP_DEBUG
    tcp_debug_print(tcphdr);
#endif /* TCP_DEBUG */
    TCP_STATS_INC(tcp.chkerr);
    goto dropped;
  }
#endif

  /* payload pointer指向TCP数据而不是头部*/
  hdrlen = TCPH_HDRLEN(tcphdr);
  if(pbuf_header(p, -(hdrlen * 4))){
    /* drop short packets */
    LWIP_DEBUGF(TCP_INPUT_DEBUG, ("tcp_input: short packet\n"));
    TCP_STATS_INC(tcp.lenerr);
    goto dropped;
  }

  /* TCP头部网络序到主机序 */
  tcphdr->src = ntohs(tcphdr->src);
  tcphdr->dest = ntohs(tcphdr->dest);
  seqno = tcphdr->seqno = ntohl(tcphdr->seqno);
  ackno = tcphdr->ackno = ntohl(tcphdr->ackno);
  tcphdr->wnd = ntohs(tcphdr->wnd);

  flags = TCPH_FLAGS(tcphdr);
  tcplen = p->tot_len + ((flags & (TCP_FIN | TCP_SYN)) ? 1 : 0);//一些报文字段的长度默认为1（握手报文之类，其数据为空但长度定义为1）

  /* 检查是哪个PCB的数据 */
  prev = NULL;

  
  for(pcb = tcp_active_pcbs; pcb != NULL; pcb = pcb->next) {
    LWIP_ASSERT("tcp_input: active pcb->state != CLOSED", pcb->state != CLOSED);
    LWIP_ASSERT("tcp_input: active pcb->state != TIME-WAIT", pcb->state != TIME_WAIT);
    LWIP_ASSERT("tcp_input: active pcb->state != LISTEN", pcb->state != LISTEN);
    if (pcb->remote_port == tcphdr->src &&
       pcb->local_port == tcphdr->dest &&
       ip_addr_cmp(&(pcb->remote_ip), &current_iphdr_src) &&
       ip_addr_cmp(&(pcb->local_ip), &current_iphdr_dest)) {
       
      LWIP_ASSERT("tcp_input: pcb->next != pcb (before cache)", pcb->next != pcb);
      if (prev != NULL) {//第一个是第一时不需要换
        prev->next = pcb->next;
        pcb->next = tcp_active_pcbs;
        tcp_active_pcbs = pcb;
      }
      LWIP_ASSERT("tcp_input: pcb->next != pcb (after cache)", pcb->next != pcb);
      break;
    }
    prev = pcb;
  }
	
  if (pcb == NULL) {
    /* 在 TIME-WAIT 链表中遍历 找到调用对应函数处理报文 */
    for(pcb = tcp_tw_pcbs; pcb != NULL; pcb = pcb->next) {
      LWIP_ASSERT("tcp_input: TIME-WAIT pcb->state == TIME-WAIT", pcb->state == TIME_WAIT);
      if (pcb->remote_port == tcphdr->src &&
         pcb->local_port == tcphdr->dest &&
         ip_addr_cmp(&(pcb->remote_ip), &current_iphdr_src) &&
         ip_addr_cmp(&(pcb->local_ip), &current_iphdr_dest)) {
        LWIP_DEBUGF(TCP_INPUT_DEBUG, ("tcp_input: packed for TIME_WAITing connection.\n"));
        tcp_timewait_input(pcb);
        pbuf_free(p);
        return;
      }
    }
    /* 在 Listen链表中遍历 找到调用对应函数处理报文 */
    prev = NULL;
    for(lpcb = tcp_listen_pcbs.listen_pcbs; lpcb != NULL; lpcb = lpcb->next) {
      if (lpcb->local_port == tcphdr->dest) {
#if SO_REUSE
        if (ip_addr_cmp(&(lpcb->local_ip), &current_iphdr_dest)) 
          break;
        } else if(ip_addr_isany(&(lpcb->local_ip))) {
          lpcb_any = lpcb;
          lpcb_prev = prev;
        }
#else /* SO_REUSE */
        if (ip_addr_cmp(&(lpcb->local_ip), &current_iphdr_dest) ||
            ip_addr_isany(&(lpcb->local_ip))) {
          break;
        }
#endif /* SO_REUSE */
      }
      prev = (struct tcp_pcb *)lpcb;
    }
#if SO_REUSE
	  /* 找不到匹配的，使用通配的 */
    if (lpcb == NULL) {
      lpcb = lpcb_any;
      prev = lpcb_prev;
    }
#endif /* SO_REUSE */
    if (lpcb != NULL) {
      /* 移到首字节 */
      if (prev != NULL) {
        ((struct tcp_pcb_listen *)prev)->next = lpcb->next;
              /* our successor is the remainder of the listening list */
        lpcb->next = tcp_listen_pcbs.listen_pcbs;
              /* put this listening pcb at the head of the listening list */
        tcp_listen_pcbs.listen_pcbs = lpcb;
      }
    
      LWIP_DEBUGF(TCP_INPUT_DEBUG, ("tcp_input: packed for LISTENing connection.\n"));
      tcp_listen_input(lpcb);
      pbuf_free(p);
      return;
    }
  }

#if TCP_INPUT_DEBUG
  LWIP_DEBUGF(TCP_INPUT_DEBUG, ("+-+-+-+-+-+-+-+-+-+-+-+-+-+- tcp_input: flags "));
  tcp_debug_print_flags(TCPH_FLAGS(tcphdr));
  LWIP_DEBUGF(TCP_INPUT_DEBUG, ("-+-+-+-+-+-+-+-+-+-+-+-+-+-+\n"));
#endif /* TCP_INPUT_DEBUG */

	/* 报文属于一个已经建立的连接 */
  if (pcb != NULL) {
#if TCP_INPUT_DEBUG
#if TCP_DEBUG
    tcp_debug_print_state(pcb->state);
#endif /* TCP_DEBUG */
#endif /* TCP_INPUT_DEBUG */

    /* tcp_seg 结构成员列表是TCP报文段的内部表示方法 */
    inseg.next = NULL;
    inseg.len = p->tot_len;
    inseg.p = p;
    inseg.tcphdr = tcphdr;

    recv_data = NULL;
    recv_flags = 0;

    if (flags & TCP_PSH) {//TCP的flag字段
      p->flags |= PBUF_FLAG_PUSH;//立刻发送给应用层
    }

    /* 如果该PCB上层有拒绝过数据 */
    if (pcb->refused_data != NULL) {
      if ((tcp_process_refused_data(pcb) == ERR_ABRT)
        ((pcb->refused_data != NULL) && (tcplen > 0))) {
        /* pcb已中止或拒绝的数据仍被拒绝，新段包含数据 */
        TCP_STATS_INC(tcp.drop);
        snmp_inc_tcpinerrs();
        goto aborted;
      }
    }
    tcp_input_pcb = pcb;
    err = tcp_process(pcb);//报文段处理
    if (err != ERR_ABRT) {
      if (recv_flags & TF_RESET) {
        /* TF_RESET表示另一端已重置连接，调用错误回调以通知应用*/
        TCP_EVENT_ERR(pcb->errf, pcb->callback_arg, ERR_RST);
        tcp_pcb_remove(&tcp_active_pcbs, pcb);
        memp_free(MEMP_TCP_PCB, pcb);
      } else if (recv_flags & TF_CLOSED) {
        /* The connection has been closed and we will deallocate the
           PCB. */
        if (!(pcb->flags & TF_RXCLOSED)) {
          TCP_EVENT_ERR(pcb->errf, pcb->callback_arg, ERR_CLSD);
        }
        tcp_pcb_remove(&tcp_active_pcbs, pcb);
        memp_free(MEMP_TCP_PCB, pcb);
      } else {
        err = ERR_OK;
        if (pcb->acked > 0) {
          TCP_EVENT_SENT(pcb, pcb->acked, err);
          if (err == ERR_ABRT) {
            goto aborted;
          }
        }

        if (recv_data != NULL) {
          LWIP_ASSERT("pcb->refused_data == NULL", pcb->refused_data == NULL);
          if (pcb->flags & TF_RXCLOSED) {
            pbuf_free(recv_data);
            tcp_abort(pcb);
            goto aborted;
          } 
          TCP_EVENT_RECV(pcb, recv_data, ERR_OK, err);
          if (err == ERR_ABRT) {
            goto aborted;
          }
          if (err != ERR_OK) {
            pcb->refused_data = recv_data;
            LWIP_DEBUGF(TCP_INPUT_DEBUG, ("tcp_input: keep incoming packet, because pcb is \"full\"\n"));
          }
        }
        if (recv_flags & TF_GOT_FIN) {
          if (pcb->refused_data != NULL) {
            /* Delay this if we have refused data. */
            pcb->refused_data->flags |= PBUF_FLAG_TCP_FIN;
          } else {
            if (pcb->rcv_wnd != TCP_WND) {
              pcb->rcv_wnd++;
            }
            TCP_EVENT_CLOSED(pcb, err);
            if (err == ERR_ABRT) {
              goto aborted;
            }
          }
        }

        tcp_input_pcb = NULL;
        tcp_output(pcb);
#if TCP_INPUT_DEBUG
#if TCP_DEBUG
        tcp_debug_print_state(pcb->state);
#endif /* TCP_DEBUG */
#endif /* TCP_INPUT_DEBUG */
      }
    }
aborted:
    tcp_input_pcb = NULL;
    recv_data = NULL;

    /* give up our reference to inseg.p */
    if (inseg.p != NULL)
    {
      pbuf_free(inseg.p);
      inseg.p = NULL;
    }
  } else {

    /* If no matching PCB was found, send a TCP RST (reset) to the
       sender. */
    LWIP_DEBUGF(TCP_RST_DEBUG, ("tcp_input: no PCB match found, resetting.\n"));
    if (!(TCPH_FLAGS(tcphdr) & TCP_RST)) {
      TCP_STATS_INC(tcp.proterr);
      TCP_STATS_INC(tcp.drop);
      tcp_rst(ackno, seqno + tcplen,
        ip_current_dest_addr(), ip_current_src_addr(),
        tcphdr->dest, tcphdr->src);
    }
    pbuf_free(p);
  }

  LWIP_ASSERT("tcp_input: tcp_pcbs_sane()", tcp_pcbs_sane());
  PERF_STOP("tcp_input");
  return;
dropped:
  TCP_STATS_INC(tcp.drop);
  snmp_inc_tcpinerrs();
  pbuf_free(p);
}
```
**报文IP头结构体：**
```
struct ip_hdr {
#if BYTE_ORDER == LITTLE_ENDIAN
  u8_t tclass1:4, v:4;
  u8_t flow1:4, tclass2:4;  
#else
  u8_t v:4, tclass1:4;
  u8_t tclass2:8, flow1:4;
#endif
  u16_t flow2;
  u16_t len;                /* payload length */
  u8_t nexthdr;             /* next header */
  u8_t hoplim;              /* hop limit (TTL) */
  struct ip_addr src, dest;          /* source and destination IP addresses */
}；
```
## 总结

tcp_input 函数开始会对 IP 层递交进来的数据包进行一些基础操作，如移动数据包的payload 指针、丢弃广播或多播数据包、数据和校验、提取 TCP 头部各个字段的值等等。接下来，函数根据接收到的 TCP 包的<IP 地址、端口>对遍历 tcp_active_pcbs 链表，寻找匹配的 PCB 控制块，若找到，则调用 tcp_process 函数对该数据包进行处理。若找不到，则再分别到 tcp_tw_pcbs 链表和 tcp_listen_pcbs 中寻找，找到则调用各自的数据包处理函数
tcp_timewait_input 和 tcp_listen_input 对数据包进行处理，若到这里都还未找到匹配的 TCP控制块，则 tcp_input 函数会调用函数 tcp_rst 向源主机发送一个 TCP 复位数据包。
