bios和KMD都可以访问cls_mem 0x1D000000-0x1D200000（2M）

通过TX_RING_BUFFER和RX_RING_BUFFER进行信息的传递

TX_RING_BUFFER_SHARED_MEM 0x1D000000-0x1D000200（512B）

RX_RING_BUFFER_SHARED_MEM 0x1D000200-0X1D000400（512B）

## ipc_commu_init

### ipc_msg结构体

传给bios的消息结构体，包括消息传递的source和target，要执行的command（event_id）等配置参数

```c
typedef struct ipc_msg_hdr {
	u64 event_type : 4;
	u64 source : 4;    //消息源host
	u64 target : 4;    //消息目的AON
	u64 event_status : 2;
	u64 event_id : 7;  //要执行的command
	u64 event_pri : 2  //优先级
	u64 msg_seq : 8;
	u64 msg_sync : 1;  //是否同步传输
	u64 msg_type : 1;
	u64 need_reply : 1;    
	u64 payload_size : 6; //dword size
	u64 rsv : 24;
} ipc_msg_hdr_t;
```

### version结构体

上层需要的包含具体信息的结构体

```c
typedef struct ver_msg {
	u32 status;
	u32 rsv;
	u64 bios_version; //bios_version
	u64 bios_build_date;
} ver_msg_t;
```

### rb_hdr结构体

ringbuffer缓存区的消息头，包含读写指针，缓存区的size和缓存区中的数据基址（date基址，date中包括ipc_msg和ver_msg）             

​                        buffer_ptr                                date

|........rb_hdr_t.........|...........ipc_msg .............|..........................ver_msg.............................|                                                       

```c
typedef struct {
	rb_index_t read_index;  //读指针
	rb_index_t write_index; //写指针
	u32 reserved;
	u32 buffer_size;	   //缓存区size
	u64 buffer_ptr;        //缓存区中date的基址,不用指针确保兼容64和32位
} rb_hdr_t;
```

初始化ipc_commu，初始化锁、初始化等待队列、初始化tx和rx的rinfbuffer地址

ringbuffer存在一块bios和host共享的内存cls_mem中

```c
init_waitqueue_head(&g_ep->waitqueue) //初始化等待队列
g_ep->tx.real_queue =bar4+TX_RING_BUFFER_SHARED_MEM //tx ringbuffer地址
g_ep->rx.real_queue =bar4+RX_RING_BUFFER_SHARED_MEM //rx ringbuffer地址
```

## ipc_msg_submit

host向bios提交请求

配置ipc_msg参数，根据event_id调用commu_call_rpc函数通过tx向bios提交ipc_msg_data，host通过rx读取bios返回的数据

```c
ipc_msg_t *ipc_msg_data;
ipc_msg_t *out_msg_data;
ipc_msg_data=kzalloc(RING_BUFFER_TOTAL_SIZE,GEP_KERNEL) //512Byte
out_msg_data=kzalloc(RING_BUFFER_TOTAL_SIZE,GEP_KERNEL)
//配置ipc_msg参数，通过tx传给bios的配置参数
ipc_msg_data->hdr.event_type=event_type 
ipc_msg_data->hdr.source=IPC_HOST		//host作为source
ipc_msg_data->hdr.target=IPC_CCD1_AON    //AON作为target
...
ipc_msg_data->hdr.msg_seq=atomic_inc_return(&msg_id) //同步值，bios中会获取这个值传到rx中，host读取rx中seq与host的seq进行比较
switch(event_id)    //根据event_type向bios提交请求，返回数据到out_msg_data
    ...
    case EVENT_TYPE_GET_VERSION:
		//需要获得信息的个数（4字节数）
	    ipc_msg_data->hdr.payload_size = sizeof(ver_msg_t) / sizeof(u32);
	    //传进bios的参数size （bios配置参数size + 需要的信息结构体size）
        enqueue_size=sizeof(ipc_msg_hdr_t) + sizeof(ver_msg_t)：
        //调用commu_call_rpc函数将请求数据传给tx_rinfbuffer，从rx_ringbuffer读取bios处理后的信息
        commu_call_rpc(ep,ipc_msg_data,enqueue_size,out_msg_data,&out_size,timeout_jiffies)
```

## unconsumed_data_size

检查已写入，但缓冲区未消费的数据size

mirror_ringbuffer

mirror=0       mirror=1

/0/1/2/3/4/.../0/1/2/3/4/...

同一块物理内存映射到两块连续的虚拟地址上，ringbuffer回绕时自动回绕，不需要手动回绕（超过0自动回绕）

mirror作为回绕标志

1.读写指针的 index 相等 

mirror相等，ringbuffer未处理size为0（剩余size为满）

mirror不相等 ringbuffer回绕，未处理size为满（剩余size为空）

2.写指针 > 读指针

未绕回，未处理size为write_index - read_index（剩余size为buffer_size -（write_index - read_index））

3.写指针 < 读指针

绕回，未处理size为buffer_size - (read_index - write_index)，（剩余size为read_index - write_index）

```c
// 情况1：读写指针的 index 相等    
if (rb_header.read_index.index == rb_header.write_index.index) {        
 // mirror相等 ringbuffer未处理size为0（剩余size为满）
 // mirror不相等 ringbuffer回绕，未处理size为满（剩余size为空）
 return (rb_header.read_index.mirror == rb_header.write_index.mirror) ? 0 : rb_header.buffer_size;
 }
 
  // 情况2：写指针 > 读指针
  //未绕回，未处理size为write_index - read_index（剩余size为buffer_size -（write_index - read_index））
  return rb_header.write_index.index > rb_header.read_index.index ? 
  (rb_header.write_index.index - rb_header.read_index.index) :
  // 情况3：写指针 < 读指针（绕回，数据分为两段）（剩余size为read_index - write_index）
  //绕回，未处理size为buffer_size - (read_index - write_index)
  rb_header.buffer_size - (rb_header.read_index.index - rb_header.write_index.index);
```

## rb_write_data

向ringbuffer传入数据

检测是否需要绕回

不需要绕回，将数据传入ringbuffer+write_index处，ringbuffer的write_index累增

需要绕回，将两段数据传入ringbuffer+write_index和ringbuffer起始处，在起始处重新更新ringbuffer的write_index

​    left_space                                                           left_space

|-------------------|----------------------------------------|---raw_left_space---|

​                  read_index                           write_index

|---------data_len - raw_left_space------|-------------------------------------|

​						   write_index

```c
//将ringbuffer中信息传到header中
memcpy_fromio(header，queue->start_addr，sizeof(rb_hdr_t));
//到ringbuffer末尾的剩余size
raw_left_space = header.buffer_size - header.write_index.index
//raw_left_space大于data_len，未回绕，size小于data_len回绕
mirror = raw_left_space > data_len ? false:true
//缓存区剩余size
left_space = header.buffer_size - rb_unconsumed_data_size(rb_header_old);
if (left_space < data_len) {
	return 0;
}
if(!mirror){
    //数据传到ringbuffer
    memcpy_toio(queue->start_addr+sizeof(rb_hdr_t)+header.write_index.index,data,data_len)
    //更新write_index，write_index累加（write_index + data_len）
    header.write_index.index += data_len;
}else{
    //回绕，数据分为两段
    memcpy_toio(queue->start_addr+sizeof(rb_hdr_t)+header.write_index.index,data,raw_left_space)
    memcpy_toio(queue->start_addr+sizeof(rb_hdr_t),data+left_space,data_len-raw_left_space)
    //起始处重新更新write_index，（data_len - raw_left_space）
    header.write_index.index = data_len-raw_left_space;
    //更新write_mirror
    header.write_index.mirror = ~header.write_index.mirror
}
```

## rb_enqueue(enqueue)

先加自旋锁且禁用CPU中断，防止写入数据的时候有中断触发，接收到新的数据

检查ringbuffer剩余size是否符合条件，将数据传入ringbuffer中

```c
//加自旋锁且禁用CPU中断，防止写入数据的时候有中断触发，接收到新的数据
spin_lock_irqsave(&queue->lock, irqflags);
rb_hdr_t header;
//将ringbuffer中信息传到header中
memcpy_fromio(header，queue->start_addr，sizeof(rb_hdr_t));
//检查ringbuffer剩余size是否符合条件
if (header.buffer_size-unconsumed_data_size(header) < size){
    pr_err("not enough space");
}
//向ringbuffer传入数据
rb_write_data(queue,buf,size)
//解锁
spin_unlock_irqrestore(&queue->lock, irqflags);
```

## rb_read_seq_only(glance)

从rx中读取seq（这里的seq是tx传进的seq，bios会把这个seq传到rx）

```c
//将ringbuffer中信息传到header中
memcpy_fromio(header，queue->start_addr，sizeof(rb_hdr_t));
size = sizeof(ipc_msg_hdr_t);
//先通过buffer_size-write_index.index检测是否绕回
left_space = header.buffer_size - header.write_index.index
mirror = left_space > size ? false:true
//读取信息到msg_header
if(!mirror){
    memcpy_fromio(&msg_header,queue->start_addr+sizeof(rb_hdr_t)+header.read_index.index,size);
}else{
    //绕回，分为两段
    memcpy_fromio(&msg_header,queue>start_addr+sizeof(rb_hdr_t)+header.read_index.index,left_space);
    memcpy_fromio(&msg_header,queue->start_addr+sizeof(rb_hdr_t),size-left_space);
}
//最后返回msg_seq
return msg_header.msg_seq
```

## rb_query_new(query_new)

缓存区中的未消费size不够容纳消息头时返回false，返回true说明ringbuffer中已经有数据

```c
//将ringbuffer中信息传到header中
memcpy_fromio(header，queue->start_addr，sizeof(rb_hdr_t));
//缓存区中未消费的size
unconsumed_data_len = rb_unconsumed_data_size(rb_header);
//缓存区中size不够容纳消息头
if (unconsumed_data_len < sizeof(ipc_msg_hdr_t))
    return false
return true
```

## rb_read_data

从ringbuffer读取数据

检测是否需要绕回

不需要绕回，从ringbuffer+read_index处读取数据，更新ringbuffer的read_index

需要绕回，从ringbuffer+read_index和ringbuffer起始处读取数据，在起始处重新更新ringbuffer的read_index

```c
//将ringbuffer中信息传到header中
memcpy_fromio(header，queue->start_addr，sizeof(rb_hdr_t));
//到ringbuffer末尾的剩余size
raw_left_space = header.buffer_size - header.read_index.index
//剩余size不足，需要回绕
mirror = raw_left_space > data_len ? false:true
if(!mirror)    
    //数据从ringbuffer传到buf
    memcpy(data, (u8 *)rb->start_addr + sizeof(rb_hdr_t) + rb_header.read_index.index, data_len);
	//更新read_index
    rb_header.read_index.index += data_len;
else
     //绕回，数据分为两段
    memcpy_toio(data, (u8 *)rb->start_addr + sizeof(rb_hdr_t) + rb_header.read_index.index, raw_left_space)
    memcpy_toio(data + raw_left_space, (u8 *)rb->start_addr + sizeof(rb_hdr_t), data_len - raw_left_space)
    //起始处重新更新read_index
    header.write_index.index = data_len-raw_left_space;
    //更新read_mirror
    header.write_index.mirror = ~header.write_index.mirror
```

## rb_dequeue(dequeue)

读取ringbuffer中的数据到buf

```c
//加锁
spin_lock_irqsave(&q->lock, irqflags);
//将ringbuffer中信息传到header中
memcpy_fromio(header，queue->start_addr，sizeof(rb_hdr_t));
//读取消息头
rb_read_data(rb, (u8 *)&msg, sizeof(ipc_msg_t));
//copy ipc_msg_t到buf中
memcpy(buf, &msg, sizeof(ipc_msg_t));
//读取ringbuffer中data
rb_read_data(rb, ipc_msg_data, msg.hdr.payload_size * sizeof(u32));
//copy ver_msg到buf中
memcpy(buf + sizeof(ipc_msg_t), ipc_msg_data, msg.hdr.payload_size * sizeof(u32));
*size = sizeof(ipc_msg_t) + msg.hdr.payload_size * sizeof(u32);
spin_unlock_irqrestore(&q->lock, irqflags);
```

## commu_call_rpc

设置tx的seq和位图，将请求信息传进tx_ringbuffer中，通过doorbell寄存器触发mailbox中断，通知bios进行处理

等待队列等待，条件：rx_ringbuffer中有足够size的数据且tx的seq和rx的seq相同（说明bios中已经将信息传给对应的rx），或者rx_ringbuffer中的信息已经包含消息头但tx和rx的seq不相同，rx对应的位图清0（说明当前的rx是上一次的，bios侧还未完成上一次的处理，位图清0说明已经上一次的处理已经超时） 

对于tx和rx的seq不相同，上一次处理超时的情况，将rx中的信息drop，重新等待

将对应rx中的信息传到out，唤醒等待队列

当bios中信息传输完成，bios会触发中断唤醒等待队列

```c
//设置tx_ringbuffer的seq，host侧的seq（初始值为0）
seq = ep->tx.real_queue.seq++;
//seq对齐到COMMU_CALL_RPC_SEQ_INUSED_BITS范围内
seq_bit = seq & COMMU_CALL_RPC_SEQ_INUSED_BITS;
//将rpc_flag的第seq_bit位设置为1（设置位图rpc_flag）
set_bit(seq_bit, &ep->rpc_flag);

//将ipc_msg传入tx_ringbuffer中
ret = ep->tx.ops->enqueue(ep->tx.real_queue，ipc_msg_data，in_size + sizeof(ipc_msg_hdr_t));
//剩余size不足，清除位图退出
if (ret < 0) 
	goto clear; 
//写doorbell寄存器，触发mailbox中断（通知bios侧请求数据准备完成，bios开始处理）
commu_data_doorbell(ep);
remaining_time = timeout_jiffiesl;

retry:
    if(timeout_jiffies){
        //等待bios侧传输完成，返回剩余的jiffies(ret)
        //rx_ringbuffer中有足够size的数据 && tx的seq和rx的seq相同（tx和rx是对应的）
        //rx_ringbuffer中有足够size的数据 && tx的seq和rx的seq不相同且rx的seq对应的rpc_flag为0（tx和rx不对应，且rx的rpc_flag被清0）
        ret = wait_event_timeout(ep->waitqueue,ep->rx.ops->query_new(ep->rx.real_queue) && 
        	(seq == ep->rx.ops->glance(ep->rx.real_queue)||!test_bit((unsigned long)ep->rx.ops-				>glance(ep->rx.real_queue) & COMMU_CALL_RPC_SEQ_INUSED_BITS, (unsigned long *)&ep-				>rpc_flag),remaining_time);
        //超时timeout，清除位图退出
        if (ret == 0) 
             goto clear;   
                                 
        if (mutex_lock_killable(&ep->ep_mutex))
            goto clear;                     
                                 
        //读取rx_ringbuffer的seq
        glance_seq = ep->rx.ops->glance(ep->rx.real_queue)
        //tx的seq和rx的seq不相同且rx的seq对应的rpc_flag为0的情况
        if (seq != glance_seq)
            remaining_time = ret;
            //取出位图中第一个被设置的位
            first_bit = find_first_bit((unsigned long *)&ep->rpc_flag,...);
            //相同说明rx的rpc_flag位被清0，此rx超时
            if (seq_bit == first_bit)
            	//将这个rx_ringbuffer中的数据读取出到drop
                ep->rx.ops->dequeue_rpc(ep->rx.real_queue, drop, &drop_size);
            mutex_unlock(&ep->ep_mutex);
            //重新wait_event_timeout
		   goto retry;   
//将rx中信息传到out
ep->rx.ops->dequeue_rpc(ep->rx.real_queue, out, &got_size); 
mutex_unlock(&ep->ep_mutex);
//唤醒等待队列
if (ep->rx.ops->query_new(ep->rx.real_queue))
	wake_up(&ep->waitqueue);
                                 
clear:
     //清除位图
	clear_bit(seq_bit, (unsigned long *)&ep->rpc_flag);
	up(&ep->ep_sema);
	return ret;
```

## bios侧

根据tx_ringbuffer中的信息进行处理，处理后的数据放到rx_ringbuffer中

bios侧会先对ring_buffer进行初始化，初始化rb_hdr结构体（主要初始化buffer_size和buffer_ptr）

```c
buffer_size = 512 - sizeof(rb_hdr)
buffer_ptr = date   //ring_buffer中date首地址
...
while (ep->tx.ops->query_new(ep->tx.real_queue))
    //从tx_ringbuffer中取出数据到inbuf
	ep->tx.ops->dequeue_rpc(ep->tx.real_queue, inbuf, &in_size);
	if (in_size < sizeof(ipc_msg_hdr_t))
		continue;
	//根据tx中的event_id进行处理
	if (inbuf->hdr.event_id == IPC_EVENT_TYPE_GET_VERSION)
        memset(outbuf, 0, sizeof(outbuf));
	   outbuf->hdr = inbuf->hdr;
	   //data size
	   outbuf->hdr.payload_size = sizeof(ver_msg_t) / 4;
	   //填充data数据到outbuf
	   ver = (ver_msg_t *)outbuf->data;
       ver->status = 0;
       ver->bios_version = 0x01020304;
       ver->bios_build_date = in_msg->hdr.msg_seq;
       //最后的outbuf总size
	   out_size = sizeof(ipc_msg_hdr_t) + sizeof(ver_msg_t);
	else 
        //未知的event_id，直接返回inbuf中信息
        memcpy(outbuf, inbuf, in_size);
        out_size = in_size;
	//将outbuf中信息传进rx_ringbuffer
	ep->rx.ops->enqueue(ep->rx.real_queue, outbuf, out_size, seq);		
```

## ipc_commu_handler

bios处理完成后会触发中断通知host信息已经处理完成，wake_up等待队列

```c
irqreturn_t ipc_commu_handler(int irq, void *data)
	struct commu_endpoint *ep = *(struct commu_endpoint **)data;
	wake_up(&ep->waitqueue);
```





