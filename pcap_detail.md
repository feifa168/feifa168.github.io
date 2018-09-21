## pcap抓包总结

**代码摘要**
```c++
// 实际申请buffer
// pcap_findalldevs->pcap_add_if_win32->add_or_find_if->
// pcap_open_live->pcap_activate->pcap_activate_win32->PacketAllocatePacket
LPPACKET PacketAllocatePacket(void){
    LPPACKET    lpPacket;    
	lpPacket=(LPPACKET)GlobalAllocPtr(GMEM_MOVEABLE | GMEM_ZEROINIT,sizeof(PACKET));
    if (lpPacket==NULL)  {
        TRACE_PRINT("PacketAllocatePacket: GlobalAlloc Failed");
    }    
	return lpPacket;
}
// 在关闭网卡时释放
// 依次调用顺序为 pcap_close->pcap_cleanup_win32->PacketFreePacket
VOID PacketFreePacket(LPPACKET lpPacket) {
    GlobalFreePtr(lpPacket);
}
// 抓抱前初始化，长度默认25600，可通过pcap_setbuff、pcap_setuserbuffer设置，先调用PacketAllocatePacket
// 依次调用顺序为 pcap_open_live->pcap_activate->pcap_activate_win32->PacketInitPacket
VOID PacketInitPacket(LPPACKET lpPacket,PVOID Buffer,UINT Length) {
	lpPacket->Buffer = Buffer;
	lpPacket->Length = Length;
	lpPacket->ulBytesReceived = 0;
	lpPacket->bIoComplete = FALSE;
}
// 接收数据包，尽可能多的读取数据包，
// 依次调用顺序为 pcap_loop->pcap_read_win32_npf->PacketReceivePacket
BOOLEAN PacketReceivePacket(LPADAPTER AdapterObject,LPPACKET lpPacket,BOOLEAN Sync) {	
	if (AdapterObject->Flags == INFO_FLAG_NDIS_ADAPTER)
	{
		if((int)AdapterObject->ReadTimeOut != -1)
			WaitForSingleObject(AdapterObject->ReadEvent, (AdapterObject->ReadTimeOut==0)?INFINITE:AdapterObject->ReadTimeOut);
	
		res = (BOOLEAN)ReadFile(AdapterObject->hFile, lpPacket->Buffer, lpPacket->Length, &lpPacket->ulBytesReceived,NULL); // @1
	}
}
```
>注解
* @1 ReadFile读取到真实的数据流长度ulBytesReceived，包含很多条数据包

```c++
static int pcap_read_win32_npf(pcap_t *p, int cnt, pcap_handler callback, u_char *user) {
	/* capture the packets */
	if(PacketReceivePacket(p->adapter,p->Packet,TRUE)==FALSE){		// @1
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "read error: PacketReceivePacket failed");
		return (-1);
	}
	int cc = p->Packet->ulBytesReceived;
	u_char *bp = p->Packet->Buffer;
	#define bhp ((struct bpf_hdr *)bp)
	u_char *ep = bp + cc;											// @2
	while (1) {														// @5
		int caplen = bhp->bh_caplen;
		int hdrlen = bhp->bh_hdrlen;

		(*callback)(user, (struct pcap_pkthdr*)bp, bp + hdrlen);	// @3
		bp += BPF_WORDALIGN(caplen + hdrlen);						// @4
		if (++n >= cnt && cnt > 0) {
			p->bp = bp;
			p->cc = ep - bp;
			return (n);
		}
	}
}
```
>注解
* @1 最大化读取数据流，包含多条数据包
* @2 cc为实际抓包长度ulBytesReceived，bp为buff开始位置，ep为buffer结束位置
* @3 调用用户回调函数，用户可以做处理数据包操作
* @4 移动当前buff位置，跳过处理的数据包长度
* @5 循环处理数据包，直到处理完buff或者处理的数据包个数达到设定的处理总数
* 总结 PacketInitPacket申请的空间不会释放，后续读取的数据流会覆盖上次的buffer，循环使用，pcap_loop->pcap_read_win32_npf->PacketReceivePacket


## pcap文件格式分析
>pcap文件的格式为：
  * 文件头    24字节
  * 数据报头 + 数据报  数据包头为16字节，后面紧跟数据报
  * 数据报头 + 数据报  ......

1. pcap文件头（24B）结构
![pcap文件头](https://images2015.cnblogs.com/blog/1190302/201707/1190302-20170713193759900-747577404.png)
pcap文件头处于pcap文件的最开始处，大小固定为24字节，文件头中各自段说明如下：
* Magic：4B：0×1A 2B 3C 4D:用来识别文件自己和字节顺序。0xa1b2c3d4用来表示按照原来的顺序读取，0xd4c3b2a1表示下面的字节都要交换顺序读取。一般，我们使用0xa1b2c3d4
* Major：2B，0×02 00:当前文件主要的版本号
* Minor：2B，0×04 00当前文件次要的版本号
* ThisZone：4B 时区。GMT和本地时间的相差，用秒来表示。如果本地的时区是GMT，那么这个值就设置为0.这个值一般也设置为0 SigFigs：4B时间戳的精度；全零
* SnapLen：4B最大的存储长度（该值设置所抓获的数据包的最大长度，如果所有数据包都要抓获，将该值设置为65535； 例如：想获取数据包的前64字节，可将该值设置为64）
* LinkType：4B链路类型


pcapet文件头结构体


```c++
//文件头结构体
sturct pcap_file_header
{
     magic;
     version_major;
     version_minor;
     thiszone;
     sigfigs;
     snaplen;
     linktype;
}
```


**说明**


* 标识位：32位的，这个标识位的值是16进制的 0xa1b2c3d4。
  * a 32-bit        magic number ,The magic number has the value hex a1b2c3d4.
* 主版本号：16位， 默认值为0x2。
  * a 16-bit    major version number,The major version number should have the value 2.
* 副版本号：16位，默认值为0x04。
  * a 16-bit    minor version number,The minor version number should have the value 4.
* 区域时间：32位，实际上该值并未使用，因此可以将该位设置为0。
  * a 32-bit    time zone offset field that actually not used, so you can (and probably should) just make it 0;
* 精确时间戳：32位，实际上该值并未使用，因此可以将该值设置为0。
  * a 32-bit    time stamp accuracy field tha not actually used,so you can (and probably should) just make it 0;
* 数据包最大长度：32位，该值设置所抓获的数据包的最大长度，如果所有数据包都要抓获，将该值设置为65535；例如：想获取数据包的前64字节，可将该值设置为64。
  * a 32-bit    snapshot length" field;The snapshot length field should be the maximum number of bytes perpacket that will be captured. If the entire packet is captured, make it 65535; if you only capture, for example, the first 64 bytes of the packet, make it 64.
* 链路层类型：32位， 数据包的链路层包头决定了链路层的类型。
  * a 32-bit link layer type field.The link-layer type depends on the type of link-layer header that the packets in the capture file have:
 
> 以下是数据值与链路层类型的对应表
* 0      BSD       loopback devices, except for later OpenBSD
* 1      Ethernet, and Linux loopback devices   以太网类型，大多数的数据包为这种类型。
* 6      802.5 Token Ring
* 7      ARCnet
* 8      SLIP
* 9      PPP
* 10     FDDI
* 100    LLC/SNAP-encapsulated ATM
* 101    raw IP, with no link
* 102    BSD/OS SLIP
* 103    BSD/OS PPP
* 104    Cisco HDLC
* 105    802.11
* 108    later OpenBSD loopback devices (with the AF_value in network byte order)
* 113    special Linux cooked capture
* 114    LocalTalk

2. pcapet包头(16B)和packet数据组成
![packet包头和数据](https://images2015.cnblogs.com/blog/1190302/201707/1190302-20170713194506509-1696240703.png)
* Timestamp：时间戳高位，精确到seconds（值是自从January 1, 1970 00:00:00 GMT以来的秒数来记）
* Timestamp：时间戳低位，精确到microseconds （数据包被捕获时候的微秒（microseconds）数，是自ts-sec的偏移量）
* Caplen：当前数据区的长度，即抓取到的数据帧长度，由此可以得到下一个数据帧的位置。
* Len：离线数据长度：网络中实际数据帧的长度，一般不大于caplen，多数情况下和Caplen数值相等。
（例如，实际上有一个包长度是1500 bytes（Len=1500），但是因为在Global Header的snaplen=1300有限制，所以只能抓取这个包的前1300个字节，这个时候，Caplen = 1300 ）
* Packet 数据：即 Packet（通常就是链路层的数据帧）具体内容，长度就是Caplen，这个长度的后面，就是当前PCAP文件中存放的下一个Packet数据包，也就 是说：PCAP文件里面并没有规定捕获的Packet数据包之间有什么间隔字符串，下一组数据在文件中的起始位置。我们需要靠第一个Packet包确定。 最后，Packet数据部分的格式其实就是标准的网路协议格式了可以任何网络教材上找得到。
</br>
**packet数据包头**</br>

```c++
// packet数据包头 
struct tim
{
    GMTtime;
    microTime
}
struct pcap_pkthdr
{
    struct tim  ts;
    caplen;
    len;
}
```
</br>

![pcap格式](https://images2015.cnblogs.com/blog/1190302/201707/1190302-20170713194825009-648390994.png)

***参考资料***
* [pcap文件格式分析](https://www.cnblogs.com/2017Crown/p/7162303.html)
* [pcap文件格式及文件解析](https://blog.csdn.net/jackyzhousales/article/details/78032054)
