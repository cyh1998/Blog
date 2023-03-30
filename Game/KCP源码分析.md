#### 前 言
当游戏对实时性，打击感要求比较高，或者需要控制大量游戏单位时，往往会使用帧同步。帧同步则需要可靠的UDP协议，一般会选择[KCP](https://github.com/skywind3000/kcp)。有关KCP的优点和特性不再赘述了，网上包括官网有关于KCP的介绍。本文主要分析KCP的源码以及实现。

#### 源 码
**一、双向链表 IQUEUEHEAD**  
KCP中设计的链表是一个环状双向链表，结构体只有两个指针，源码以及操作接口如下：
```
struct IQUEUEHEAD {
	struct IQUEUEHEAD *next, *prev;
};

// 向后指针和向前指针均指向自己, 形成环状双向链表
#define IQUEUE_INIT(ptr) ( \
	(ptr)->next = (ptr), (ptr)->prev = (ptr))

// MEMBER在TYPE中的偏移量
#define IOFFSETOF(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)

// ptr所在type类型对象的首地址
#define ICONTAINEROF(ptr, type, member) ( \
		(type*)( ((char*)((type*)ptr)) - IOFFSETOF(type, member)) )

#define IQUEUE_ENTRY(ptr, type, member) ICONTAINEROF(ptr, type, member)

// node插入到head后面, head->node
#define IQUEUE_ADD(node, head) ( \
	(node)->prev = (head), (node)->next = (head)->next, \
	(head)->next->prev = (node), (head)->next = (node))

// node插入到head前面, node->head
#define IQUEUE_ADD_TAIL(node, head) ( \
	(node)->prev = (head)->prev, (node)->next = (head), \
	(head)->prev->next = (node), (head)->prev = (node))

// 删除p, n之间的节点
#define IQUEUE_DEL_BETWEEN(p, n) ((n)->prev = (p), (p)->next = (n))

// 删除entry, prev->entry->next  ====>>>>  prev->next
#define IQUEUE_DEL(entry) (\
	(entry)->next->prev = (entry)->prev, \
	(entry)->prev->next = (entry)->next, \
	(entry)->next = 0, (entry)->prev = 0)

#define IQUEUE_DEL_INIT(entry) do { \
	IQUEUE_DEL(entry); IQUEUE_INIT(entry); } while (0)

// 链表是否为空
#define IQUEUE_IS_EMPTY(entry) ((entry) == (entry)->next)
```
链表结构本身并没有定义任何value字段，而是让持有`IQUEUEHEAD`的结构体自己去定义。当持有了`IQUEUEHEAD`，该结构体就可以通过`IQUEUEHEAD`将自己构建成一个环状双向链表，如图：  
![持有IQUEUEHEAD的结构体](https://upload-images.jianshu.io/upload_images/22192996-693c7a5910cb2915.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那如何通过`IQUEUEHEAD`来获取到其所在的结构体呢？需要看下`IQUEUE_ENTRY`宏的展开
```
(type*)( ((char*)((type*)ptr)) - ((size_t) &((type *)0)->member)) )
```
意思就是将`IQUEUEHEAD`指针`ptr`减去`member`成员在类型`type`中的偏移量，这样就得到了所在类型`type`的首地址。

**二、数据段结构 IKCPSEG**  
`IKCPSEG`是发送包的数据结构，其包含了两部分信息：
- 发送信息：`conv`、`cmd`、`frg`、`wnd`、`ts`、`sn`、`una`、`len`、`data`，其中`cmd`包含四种类型  
  - `IKCP_CMD_PUSH`：发送数据包
  - `IKCP_CMD_ACK`：返回确认包
  - `IKCP_CMD_WASK`：探测对端可用窗口大小
  - `IKCP_CMD_WINS`：告知对端自身可用窗口大小
- 控制信息：其余的为控制信息，不会发送给对端。暂存于发送消息队列中，用来判断超时重发、快速重发。
```
/**
|<------------ 4 bytes ------------>|
+--------+--------+--------+--------+
|               conv                | conv: 会话序号
+--------+--------+--------+--------+ 
|  cmd   |  frg   |       wnd       | cmd:  指令标识; frg: 分段序号; wnd: 接收窗口大小
+--------+--------+--------+--------+ 
|                ts                 | ts:   发送的时间戳
+--------+--------+--------+--------+
|                sn                 | sn:   段序号
+--------+--------+--------+--------+
|                una                | una:  未收到的序号
+--------+--------+--------+--------+
|                len                | len:  data数据的长度
+--------+--------+--------+--------+
**/
//=====================================================================
// SEGMENT
// kcp的数据段结构
//=====================================================================
struct IKCPSEG
{
	struct IQUEUEHEAD node; // 双向链表定义的队列
    IUINT32 conv;           // conversation, 会话序号
    IUINT32 cmd;            // command, 指令类型
    IUINT32 frg;            // fragment, 分片序号
    IUINT32 wnd;            // window, 可用窗口大小
    IUINT32 ts;             // timestamp, 发送的时间戳
    IUINT32 sn;             // sequence number, 段序号
    IUINT32 una;            // unacknowledged, 当前未收到的序号
    IUINT32 len;            // length, data数据的长度
    IUINT32 resendts;       // 超时重发的时间戳
    IUINT32 rto;            // 超时重传的时间间隔
    IUINT32 fastack;        // ack跳过的次数, 用于快速重传
    IUINT32 xmit;           // 发送的次数
	char data[1];
};
```

**三、数据块 IKCPCB**  
`IKCPCB`用于存放KCP相关的配置、数据缓存、协议控制的数据结构，结构如下：
```
//---------------------------------------------------------------------
// IKCPCB
// kcp的控制块
//---------------------------------------------------------------------
struct IKCPCB
{
	// conv:会话ID, mtu:最大传输单元, mss:最大分片大小, state:连接状态
	IUINT32 conv, mtu, mss, state;
	// sun_una:第一个未确认的包, sen_nxt:待发送包的序号, rcv_nxt:待接收消息的序号
	IUINT32 snd_una, snd_nxt, rcv_nxt;
	// ts_recent(未使用), ts_lastack(未使用), ssthresh:拥塞窗口的阈值
	IUINT32 ts_recent, ts_lastack, ssthresh;
	// rx_rttval: rtt的变化量, rx_srtt: rtt的平滑值(smoothed), 
	// rx_rto: 由ack接收延迟计算出来的重传超时时间, rx_minrto: 最小重传超时时间
	IINT32 rx_rttval, rx_srtt, rx_rto, rx_minrto;
	// sen_wnd: 发送窗口大小, rcv_wnd: 接收窗口大小, rmt_wnd: 远端可用窗口大小, 
	// cwnd: 拥塞窗口大小, probe: 探查变量
	IUINT32 snd_wnd, rcv_wnd, rmt_wnd, cwnd, probe;
	//currunt: 当前时间戳, interval: 内部flush刷新间隔, ts_flush: 下次刷新时间戳, xmit: 重发消息的次数
	IUINT32 current, interval, ts_flush, xmit;
	// nrcv_buf: snd_buf的长度, nsnd_buf: rcv_buf的长度
	IUINT32 nrcv_buf, nsnd_buf;
	// nrcv_que: rcv_queue的长度, nsnd_que: snd_queue的长度
	IUINT32 nrcv_que, nsnd_que;
	// nodelay: 是否启动无延迟模式, update: 是否调用过update函数的标识
	IUINT32 nodelay, updated;
	//ts_probe: 下次探测窗口的时间戳, probe_wait: 探测窗口等待的时间
	IUINT32 ts_probe, probe_wait;
	//dead_link: 最大重传次数, incr: 可发送的最大数据量
	IUINT32 dead_link, incr;
	struct IQUEUEHEAD snd_queue;    //发送消息的队列
    struct IQUEUEHEAD rcv_queue;    //接收消息的队列
    struct IQUEUEHEAD snd_buf;      //发送消息的缓存
    struct IQUEUEHEAD rcv_buf;      //接收消息的缓存
    IUINT32 *acklist;               //待发送的ack的列表, 其内容是[sn1, ts1, sn2, ts2 …]的列表
    IUINT32 ackcount;               //当前ack数量
    IUINT32 ackblock;               //acklist的大小
    void *user;						//用户指针
    char *buffer;                   //储存消息字节流的内存
    int fastresend;                 //触发快速重传的重复ack个数
    int fastlimit;					//快速重传次数上限
    int nocwnd, stream;             //nocwnd: 取消拥塞控制, stream: 是否采用流传输模式
    int logmask;					//日志的类型
	int (*output)(const char *buf, int len, struct IKCPCB *kcp, void *user); //发送消息的回调函数
	void (*writelog)(const char *log, struct IKCPCB *kcp, void *user);		 //写日志的回调函数
};
```
**四、KCP主要接口**  
这里主要看下四个核心接口：
```
// 发送消息，将本端数据写入缓存
int ikcp_send(ikcpcb *kcp, const char *buffer, int len);

// 刷新数据，发送到对端
void ikcp_flush(ikcpcb *kcp);

// 输入数据，将对端数据写入缓存
int ikcp_input(ikcpcb *kcp, const char *data, long size);

// 接收消息
int ikcp_recv(ikcpcb *kcp, char *buffer, int len);
```
*4.1 ikcp_send*  
将发送数据根据mss进行分片，分片为KCP的数据格式，按序插入到发送队列中。分片默认构建新的KCP包并添加到队列，如果使用流模式分片，将判断发送队列的最后一个分片是否填满mss，若没有，则用新的数据填充分片后再分片。
```
int ikcp_send(ikcpcb *kcp, const char *buffer, int len)
{
	IKCPSEG *seg;
	int count, i;

	// mss = mtu - 24(IKCP_OVERHEAD KCP报文头部大小)
	assert(kcp->mss > 0);
	if (len < 0) return -1;

	// append to previous segment in streaming mode (if possible)
	// 使用流模式
	if (kcp->stream != 0) {
		if (!iqueue_is_empty(&kcp->snd_queue)) {
			// 判断发送队列的最后一个分片是否填满mss
			IKCPSEG *old = iqueue_entry(kcp->snd_queue.prev, IKCPSEG, node);
			if (old->len < kcp->mss) {
				// 用新的数据填充分片
				int capacity = kcp->mss - old->len; //分片剩余容量大小
				int extend = (len < capacity)? len : capacity; //填充大小
				seg = ikcp_segment_new(kcp, old->len + extend);
				assert(seg);
				if (seg == NULL) {
					return -2;
				}
				iqueue_add_tail(&seg->node, &kcp->snd_queue);
				memcpy(seg->data, old->data, old->len);
				if (buffer) {
					memcpy(seg->data + old->len, buffer, extend);
					buffer += extend;
				}
				seg->len = old->len + extend;
				seg->frg = 0;
				len -= extend;
				iqueue_del_init(&old->node);
				ikcp_segment_delete(kcp, old);
			}
		}
		// 没有剩余数据，返回成功
		if (len <= 0) {
			return 0;
		}
	}

	// 计算分片数量
	if (len <= (int)kcp->mss) count = 1;
	else count = (len + kcp->mss - 1) / kcp->mss;

	// 分片数量大于对端接收窗口大小
	if (count >= (int)IKCP_WND_RCV) return -2;

	if (count == 0) count = 1;

	// fragment
	// 将数据分片并添加到发送消息队列
	for (i = 0; i < count; i++) {
		int size = len > (int)kcp->mss ? (int)kcp->mss : len;
		seg = ikcp_segment_new(kcp, size);
		assert(seg);
		if (seg == NULL) {
			return -2;
		}
		if (buffer && len > 0) {
			memcpy(seg->data, buffer, size);
		}
		seg->len = size;
		// 设置分片序号，frg从大到小
		seg->frg = (kcp->stream == 0)? (count - i - 1) : 0;
		iqueue_init(&seg->node);
		iqueue_add_tail(&seg->node, &kcp->snd_queue);
		kcp->nsnd_que++;
		if (buffer) {
			buffer += size;
		}
		len -= size;
	}

	return 0;
}
```
*4.2 ikcp_flush*  
将数据包通过用户提供的发送函数发送到对端，所有的数据分片累计超过mtu后才会发送出去，同时根据这发送消息的情况，调整kcp配置。
```
void ikcp_flush(ikcpcb *kcp)
{
	IUINT32 current = kcp->current;
	char *buffer = kcp->buffer;
	char *ptr = buffer;
	int count, size, i;
	IUINT32 resent, cwnd;
	IUINT32 rtomin;
	struct IQUEUEHEAD *p;
	int change = 0;
	int lost = 0;
	// 此IKCPSEG被序列化多次，反复使用
	IKCPSEG seg;

	// 'ikcp_update' haven't been called. 
	if (kcp->updated == 0) return;

	// 序列化为发送确认包
	seg.conv = kcp->conv;
	seg.cmd = IKCP_CMD_ACK;
	seg.frg = 0;
	seg.wnd = ikcp_wnd_unused(kcp);
	seg.una = kcp->rcv_nxt;
	seg.len = 0;
	seg.sn = 0;
	seg.ts = 0;

	// flush acknowledges
	// 循环发送确认包
	count = kcp->ackcount;
	for (i = 0; i < count; i++) {
		size = (int)(ptr - buffer);
		// 超过mtu才发送，否则下文会不断序列化seg到buffer中
		if (size + (int)IKCP_OVERHEAD > (int)kcp->mtu) {
			ikcp_output(kcp, buffer, size);
			ptr = buffer;
		}
		// 根据待发送的确认包列表acklist，填充段序号和发送时间戳
		ikcp_ack_get(kcp, i, &seg.sn, &seg.ts);
		// 序列化seg到buffer中
		ptr = ikcp_encode_seg(ptr, &seg);
	}

	kcp->ackcount = 0;

	// probe window size (if remote window size equals zero)
	if (kcp->rmt_wnd == 0) {
		// 探测窗口等待时间为0
		if (kcp->probe_wait == 0) { 
			kcp->probe_wait = IKCP_PROBE_INIT; //重新赋值探测窗口等待时间，默认7秒
			kcp->ts_probe = kcp->current + kcp->probe_wait; //重新赋值下次探测窗口时间戳
		}	
		else {
			// 是否到达探测时间
			if (_itimediff(kcp->current, kcp->ts_probe) >= 0) {
				// 更新探测窗口等待时间和下次探测窗口时间戳
				// 设置探测变量为 IKCP_ASK_SEND，发送窗口探测
				if (kcp->probe_wait < IKCP_PROBE_INIT) 
					kcp->probe_wait = IKCP_PROBE_INIT;
				kcp->probe_wait += kcp->probe_wait / 2;
				if (kcp->probe_wait > IKCP_PROBE_LIMIT)
					kcp->probe_wait = IKCP_PROBE_LIMIT;
				kcp->ts_probe = kcp->current + kcp->probe_wait;
				kcp->probe |= IKCP_ASK_SEND;
			}
		}
	}	else {
		// 远端窗口正常，不需要探测，不发送窗口探测
		kcp->ts_probe = 0;
		kcp->probe_wait = 0;
	}

	// flush window probing commands
	// 是否需要探测对端窗口
	if (kcp->probe & IKCP_ASK_SEND) {
		// 序列化对端窗口探测包
		seg.cmd = IKCP_CMD_WASK;
		size = (int)(ptr - buffer);
		if (size + (int)IKCP_OVERHEAD > (int)kcp->mtu) {
			ikcp_output(kcp, buffer, size);
			ptr = buffer;
		}
		ptr = ikcp_encode_seg(ptr, &seg);
	}

	// flush window probing commands
	// 是否需要告知对端自身窗口大小
	if (kcp->probe & IKCP_ASK_TELL) {
		// 序列化告知对端自身窗口包
		seg.cmd = IKCP_CMD_WINS;
		size = (int)(ptr - buffer);
		if (size + (int)IKCP_OVERHEAD > (int)kcp->mtu) {
			ikcp_output(kcp, buffer, size);
			ptr = buffer;
		}
		ptr = ikcp_encode_seg(ptr, &seg);
	}

	kcp->probe = 0;

	// calculate window size
	// 计算最小发送窗口，取自身窗口大小与对端窗口大小的最小值
	cwnd = _imin_(kcp->snd_wnd, kcp->rmt_wnd);
	// 如果没有取消拥塞控制，取配置拥塞窗口、自身窗口和对端窗口三者最小值
	if (kcp->nocwnd == 0) cwnd = _imin_(kcp->cwnd, cwnd);

	// move data from snd_queue to snd_buf
	// 将满足窗口大小个数的数据分片，从kcp->snd_queue移动到kcp->snd_buf
	while (_itimediff(kcp->snd_nxt, kcp->snd_una + cwnd) < 0) {
		IKCPSEG *newseg;
		if (iqueue_is_empty(&kcp->snd_queue)) break;

		newseg = iqueue_entry(kcp->snd_queue.next, IKCPSEG, node);

		// 从snd_queue中删除自身
		iqueue_del(&newseg->node);
		// 添加到snd_buf中，注意这里是插入到了snd_buf的前面
		iqueue_add_tail(&newseg->node, &kcp->snd_buf);
		kcp->nsnd_que--;
		kcp->nsnd_buf++;

		// 设置数据分片属性
		newseg->conv = kcp->conv;
		newseg->cmd = IKCP_CMD_PUSH;
		newseg->wnd = seg.wnd;
		newseg->ts = current;
		newseg->sn = kcp->snd_nxt++;
		newseg->una = kcp->rcv_nxt;
		newseg->resendts = current;
		newseg->rto = kcp->rx_rto;
		newseg->fastack = 0;
		newseg->xmit = 0;
	}

	// calculate resent
	// 重新计算触发快速重传的重复ack个数
	resent = (kcp->fastresend > 0)? (IUINT32)kcp->fastresend : 0xffffffff;
	// 重新计算超时重传时间，rx_rto的1/8
	rtomin = (kcp->nodelay == 0)? (kcp->rx_rto >> 3) : 0;

	// flush data segments
	for (p = kcp->snd_buf.next; p != &kcp->snd_buf; p = p->next) {
		IKCPSEG *segment = iqueue_entry(p, IKCPSEG, node);
		int needsend = 0;
		// 第一次发送
		if (segment->xmit == 0) {
			needsend = 1;
			segment->xmit++;
			segment->rto = kcp->rx_rto;
			segment->resendts = current + segment->rto + rtomin;
		}
		// 超过了重发时间
		else if (_itimediff(current, segment->resendts) >= 0) {
			needsend = 1;
			segment->xmit++;
			kcp->xmit++;
			if (kcp->nodelay == 0) {
				segment->rto += _imax_(segment->rto, (IUINT32)kcp->rx_rto);
			}	else {
				IINT32 step = (kcp->nodelay < 2)? 
					((IINT32)(segment->rto)) : kcp->rx_rto;
				segment->rto += step / 2;
			}
			segment->resendts = current + segment->rto;
			// 发生超时重发
			lost = 1;
		}
		// 跳过应答的次数超过了配置
		else if (segment->fastack >= resent) {
			// 发送次数 <= 快速重传上限 || 未设置快速重发上限
			if ((int)segment->xmit <= kcp->fastlimit || 
				kcp->fastlimit <= 0) {
				needsend = 1;
				segment->xmit++;
				segment->fastack = 0;
				segment->resendts = current + segment->rto;
				// 发生快速重传
				change++;
			}
		}

		// 满足任意以上三个条件，则需要发送
		if (needsend) {
			int need;
			segment->ts = current;
			segment->wnd = seg.wnd;
			segment->una = kcp->rcv_nxt;

			size = (int)(ptr - buffer);
			need = IKCP_OVERHEAD + segment->len;

			if (size + need > (int)kcp->mtu) {
				ikcp_output(kcp, buffer, size);
				ptr = buffer;
			}

			ptr = ikcp_encode_seg(ptr, segment);

			// 把data数据写入到kcp->buffer
			if (segment->len > 0) {
				memcpy(ptr, segment->data, segment->len);
				ptr += segment->len;
			}

			if (segment->xmit >= kcp->dead_link) {
				kcp->state = (IUINT32)-1;
			}
		}
	}

	// flash remain segments
	// 将剩余的buffer发送出去
	size = (int)(ptr - buffer);
	if (size > 0) {
		ikcp_output(kcp, buffer, size);
	}

	// update ssthresh
	// 根据发送消息情况，调整kcp配置
	// 是否发生快速重传
	if (change) {
		IUINT32 inflight = kcp->snd_nxt - kcp->snd_una;
		kcp->ssthresh = inflight / 2;
		if (kcp->ssthresh < IKCP_THRESH_MIN)
			kcp->ssthresh = IKCP_THRESH_MIN;
		kcp->cwnd = kcp->ssthresh + resent;
		kcp->incr = kcp->cwnd * kcp->mss;
	}

	// 是否发生超时重发
	if (lost) {
		kcp->ssthresh = cwnd / 2;
		if (kcp->ssthresh < IKCP_THRESH_MIN)
			kcp->ssthresh = IKCP_THRESH_MIN;
		kcp->cwnd = 1;
		kcp->incr = kcp->mss;
	}

	if (kcp->cwnd < 1) {
		kcp->cwnd = 1;
		kcp->incr = kcp->mss;
	}
}
```
*4.3 ikcp_input*  
输入处理对端的消息数据，根据不同的操作类型进行不同的处理，写入缓存。计算并调整窗口。
```
int ikcp_input(ikcpcb *kcp, const char *data, long size)
{
	IUINT32 prev_una = kcp->snd_una;
	IUINT32 maxack = 0, latest_ts = 0;
	int flag = 0;

	if (ikcp_canlog(kcp, IKCP_LOG_INPUT)) {
		ikcp_log(kcp, IKCP_LOG_INPUT, "[RI] %d bytes", (int)size);
	}

	// 数据为空 || 消息大小小于KCP报文
	if (data == NULL || (int)size < (int)IKCP_OVERHEAD) return -1;

	while (1) {
		IUINT32 ts, sn, len, una, conv;
		IUINT16 wnd;
		IUINT8 cmd, frg;
		IKCPSEG *seg;

		if (size < (int)IKCP_OVERHEAD) break;

		// 获取KCP报文头部信息
		data = ikcp_decode32u(data, &conv);
		if (conv != kcp->conv) return -1;

		data = ikcp_decode8u(data, &cmd);
		data = ikcp_decode8u(data, &frg);
		data = ikcp_decode16u(data, &wnd);
		data = ikcp_decode32u(data, &ts);
		data = ikcp_decode32u(data, &sn);
		data = ikcp_decode32u(data, &una);
		data = ikcp_decode32u(data, &len);

		size -= IKCP_OVERHEAD;

		if ((long)size < (long)len || (int)len < 0) return -2;

		// 验证指令类型
		if (cmd != IKCP_CMD_PUSH && cmd != IKCP_CMD_ACK &&
			cmd != IKCP_CMD_WASK && cmd != IKCP_CMD_WINS) 
			return -3;

		// 更新对端接收窗口[1]
		kcp->rmt_wnd = wnd;
		// 确认(删除)una之前的数据包
		ikcp_parse_una(kcp, una);
		// 更新snd_una(未确认包序号)
		ikcp_shrink_buf(kcp);

		if (cmd == IKCP_CMD_ACK) {
			if (_itimediff(kcp->current, ts) >= 0) {
				// 更新rx_rttval rx_srtt rx_rto
				ikcp_update_ack(kcp, _itimediff(kcp->current, ts));
			}
			// 确认sn数据包
			ikcp_parse_ack(kcp, sn);
			// 更新snd_una
			ikcp_shrink_buf(kcp);
			// 记录最后一个包段序号和时间戳
			if (flag == 0) {
				flag = 1;
				maxack = sn;
				latest_ts = ts;
			}	else {
				if (_itimediff(sn, maxack) > 0) {
				#ifndef IKCP_FASTACK_CONSERVE
					maxack = sn;
					latest_ts = ts;
				#else
					if (_itimediff(ts, latest_ts) > 0) {
						maxack = sn;
						latest_ts = ts;
					}
				#endif
				}
			}
			if (ikcp_canlog(kcp, IKCP_LOG_IN_ACK)) {
				ikcp_log(kcp, IKCP_LOG_IN_ACK, 
					"input ack: sn=%lu rtt=%ld rto=%ld", (unsigned long)sn, 
					(long)_itimediff(kcp->current, ts),
					(long)kcp->rx_rto);
			}
		}
		else if (cmd == IKCP_CMD_PUSH) {
			if (ikcp_canlog(kcp, IKCP_LOG_IN_DATA)) {
				ikcp_log(kcp, IKCP_LOG_IN_DATA, 
					"input psh: sn=%lu ts=%lu", (unsigned long)sn, (unsigned long)ts);
			}
			if (_itimediff(sn, kcp->rcv_nxt + kcp->rcv_wnd) < 0) {
				// 添加sn确认包
				ikcp_ack_push(kcp, sn, ts);
				// 处理接收窗口范围内的数据包
				if (_itimediff(sn, kcp->rcv_nxt) >= 0) {
					seg = ikcp_segment_new(kcp, len);
					seg->conv = conv;
					seg->cmd = cmd;
					seg->frg = frg;
					seg->wnd = wnd;
					seg->ts = ts;
					seg->sn = sn;
					seg->una = una;
					seg->len = len;

					if (len > 0) {
						memcpy(seg->data, data, len);
					}

					// 处理消息数据
					ikcp_parse_data(kcp, seg);
				}
			}
		}
		else if (cmd == IKCP_CMD_WASK) {
			// ready to send back IKCP_CMD_WINS in ikcp_flush
			// tell remote my window size
			// 更新探测变量
			// 在update时告知对端自身接收窗口的大小
			kcp->probe |= IKCP_ASK_TELL;
			if (ikcp_canlog(kcp, IKCP_LOG_IN_PROBE)) {
				ikcp_log(kcp, IKCP_LOG_IN_PROBE, "input probe");
			}
		}
		else if (cmd == IKCP_CMD_WINS) {
			// do nothing
			// 在上文[1]已经更新对端的接收窗口大小
			if (ikcp_canlog(kcp, IKCP_LOG_IN_WINS)) {
				ikcp_log(kcp, IKCP_LOG_IN_WINS,
					"input wins: %lu", (unsigned long)(wnd));
			}
		}
		else {
			return -3;
		}

		data += len;
		size -= len;
	}

	// 是否有处理确认包
	if (flag != 0) {
		// 遍历发送缓冲，序号小于maxack消息的fastack++，以触发快速重传
		ikcp_parse_fastack(kcp, maxack, latest_ts);
	}

	if (_itimediff(kcp->snd_una, prev_una) > 0) {
		// 更新cwnd
		if (kcp->cwnd < kcp->rmt_wnd) {
			IUINT32 mss = kcp->mss;
			if (kcp->cwnd < kcp->ssthresh) {
				// 小于阈值，线性增加
				kcp->cwnd++;
				kcp->incr += mss;
			}	else {
				if (kcp->incr < mss) kcp->incr = mss;
				// incr += mss * (1/n + 1/16)
				kcp->incr += (mss * mss) / kcp->incr + (mss / 16);
				if ((kcp->cwnd + 1) * mss <= kcp->incr) {
				#if 1
					kcp->cwnd = (kcp->incr + mss - 1) / ((mss > 0)? mss : 1);
				#else
					kcp->cwnd++;
				#endif
				}
			}
			// 最大不超过rmt_wnd
			if (kcp->cwnd > kcp->rmt_wnd) {
				kcp->cwnd = kcp->rmt_wnd;
				kcp->incr = kcp->rmt_wnd * mss;
			}
		}
	}

	return 0;
}
```
*4.4 ikcp_recv*  
从rcv_queue中读取一帧完整的数据，再将rcv_buf中的连续分片转移到rcv_queue，最后判断是否需要告知自身窗口大小。
```
int ikcp_recv(ikcpcb *kcp, char *buffer, int len)
{
	struct IQUEUEHEAD *p;
	int ispeek = (len < 0)? 1 : 0;
	int peeksize;
	int recover = 0;
	IKCPSEG *seg;
	assert(kcp);

	if (iqueue_is_empty(&kcp->rcv_queue))
		return -1;

	if (len < 0) len = -len;

	// 获取一帧的数据大小
	peeksize = ikcp_peeksize(kcp);

	if (peeksize < 0) 
		return -2;

	if (peeksize > len) 
		return -3;

	// 标记可以开始窗口恢复
	if (kcp->nrcv_que >= kcp->rcv_wnd)
		recover = 1;

	// merge fragment
	for (len = 0, p = kcp->rcv_queue.next; p != &kcp->rcv_queue; ) {
		// 将一帧完整的数据从rcv_queue复制到data缓冲区
		int fragment;
		seg = iqueue_entry(p, IKCPSEG, node);
		p = p->next;

		if (buffer) {
			memcpy(buffer, seg->data, seg->len);
			buffer += seg->len;
		}

		len += seg->len;
		fragment = seg->frg;

		if (ikcp_canlog(kcp, IKCP_LOG_RECV)) {
			ikcp_log(kcp, IKCP_LOG_RECV, "recv sn=%lu", (unsigned long)seg->sn);
		}

		if (ispeek == 0) {
			iqueue_del(&seg->node);
			ikcp_segment_delete(kcp, seg);
			kcp->nrcv_que--;
		}

		if (fragment == 0) 
			break;
	}

	assert(len == peeksize);

	// move available data from rcv_buf -> rcv_queue
	// 将rcv_buf中连续段序号的数据包移动到rcv_queue
	while (! iqueue_is_empty(&kcp->rcv_buf)) {
		seg = iqueue_entry(kcp->rcv_buf.next, IKCPSEG, node);
		// 判断段序号是否连续 && 接收队列长度小于自身接收窗口
		if (seg->sn == kcp->rcv_nxt && kcp->nrcv_que < kcp->rcv_wnd) {
			iqueue_del(&seg->node);
			kcp->nrcv_buf--;
			// 添加到rcv_queue
			iqueue_add_tail(&seg->node, &kcp->rcv_queue);
			kcp->nrcv_que++;
			kcp->rcv_nxt++;
		}	else {
			// 待接收数据包的序号不连续，break，在下一次接收再提交
			break;
		}
	}

	// fast recover
	if (kcp->nrcv_que < kcp->rcv_wnd && recover) {
		// ready to send back IKCP_CMD_WINS in ikcp_flush
		// tell remote my window size
		// 下一次update时，发送窗口大小告知包
		kcp->probe |= IKCP_ASK_TELL;
	}

	return len;
}
```
参考：  
[KCP - A Fast and Reliable ARQ Protocol](https://github.com/skywind3000/kcp)  
[kcp源码分析-知乎](https://zhuanlan.zhihu.com/p/554252832)  