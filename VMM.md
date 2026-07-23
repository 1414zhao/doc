VMM（虚拟内存管理，细化内存分配，主要分配不连续的物理内存），与cudamalloc（主要申请连续内存）不同，分离了物理内存分配和虚拟地址的映射，支持多进程之间共享内存

## main

​	测试VMM的export/import handle实现多进程之间的共享内存，使用父子进程作为例子

```c
//创建一对套接字，全双工，用于父子进程通信
int socks[2] = {-1, -1};
socketpair(AF_UNIX, SOCK_SEQPACKET, 0, socks)
pid = fork();
//子进程
if (pid == 0)  
    //关闭用不到的一端
    close(socks[0]);
	//子进程函数
    int rc = child_main(socks[1], child_dev);
    close(socks[1]);  
//父进程
//关闭用不到的一端    
close(socks[1]);
//父进程函数
int parent_rc = parent_main(socks[0], owner_dev, child_dev)
close(socks[0]);
```

## parent_main

​	父进程函数，负责通过VMM申请内存，申请共享handle exporthandle，通过socket将共享handle传给子进程

```c
//初始化cuda
cuInit(0)
cudaSetDevice(owner_dev)
//配置prop
CUmemAllocationProp prop = {}; 
//设置分配类型为固定（页锁定）内存
prop.type = CU_MEM_ALLOCATION_TYPE_PINNED;
//指定内存的物理位置，在gpu分配，gpu id为owner_dev
prop.location.type = CU_MEM_LOCATION_TYPE_DEVICE;
prop.location.id = owner_dev;
//多进程共享，句柄类型为CU_MEM_HANDLE_TYPE_POSIX_FILE_DESCRIPTOR
prop.requestedHandleTypes = CU_MEM_HANDLE_TYPE_POSIX_FILE_DESCRIPTOR;
//获取对齐大小
cuMemGetAllocationGranularity(&granularity, &prop, CU_MEM_ALLOC_GRANULARITY_MINI)
//将size向上对齐
alloc_size = ((requested_size + granularity - 1) / granularity) * granularity

//预留虚拟地址
cuMemAddressReserve(&parent_va, alloc_size, granularity, 0, 0)
//分配物理内存
cuMemCreate(&handle, alloc_size, &prop, 0)
//建立映射，将物理内存映射到虚拟地址
cuMemMap(parent_va, alloc_size, 0, handle, 0)  
//设置内存访问权限，可读可写
CUmemAccessDesc access = {};   
access.location.type = CU_MEM_LOCATION_TYPE_DEVICE;
access.location.id = device;
access.flags = CU_MEM_ACCESS_FLAGS_PROT_READWRITE; 
cuMemSetAccess(parent_va, alloc_size, &access, 1)
//创建共享的handle，实现多进程共享内存    
cuMemExportToShareableHandle(&exported_handle, handle, CU_MEM_HANDLE_TYPE_POSIX_FILE_DESCRIPTOR, 0)
shareable_fd = static_cast<int>(exported_handle);
//向子进程发送共享handle
send_fd(sock, shareable_fd);                        
close(shareable_fd);    
```

## send_fd

​	向接收进程发送文件描述符（内核对象的访问权），将这个fd所指向的内核对象（即那块物理显存）的引用，安装到接收方进程的文件描述符表中，并返回一个新的、对接收方有效的 fd 编号，接收进程获得后可以访问send进程fd对应的内存

```c
struct msghdr msg = {};
//设置1字节的普通数据，sendmsg 要求消息中必须包含至少1字节的普通数据，否则辅助数据（文件描述符）可能不会被发送
struct iovec iov[1];
char data = 'F';
iov[0].iov_base = &data;
iov[0].iov_len = 1;
msg.msg_iov = iov;
msg.msg_iovlen = 1;

//设置辅助数据
//CMSG_SPACE(sizeof(int)):容纳一个 int 类型辅助数据所需的总字节数
char control[CMSG_SPACE(sizeof(int))];
msg.msg_control = control;
msg.msg_controllen = sizeof(control);
//辅助数据的消息头
struct cmsghdr *cmsg = CMSG_FIRSTHDR(&msg);
//辅助数据的协议级别
cmsg->cmsg_level = SOL_SOCKET;
//告诉内核发送的文件描述符信息在cmsg的数据部分
cmsg->cmsg_type = SCM_RIGHTS;
//消息头的长度
cmsg->cmsg_len = CMSG_LEN(sizeof(int));
//将文件描述符写到cmsg的数据部分
*(int*)CMSG_DATA(cmsg) = fd_to_send;
//发送消息，传递内核对象的访问权
//在接收进程的文件描述符表中，新建一个条目，这个条目指向与 fd_to_send 完全相同的内核对象（比如那块 CUDA 显存），并给这个新条目分配一个接收进程可用的、新的fd。这个fd在接收方通过 recvmsg 接收时被获取
sendmsg(sock, &msg, 0)
```

## child_main

​	子进程函数，通过socket接收父进程的共享handle，将共享handle对应的物理内存映射到虚拟地址，和父进程共享一块物理内存

```c
//初始化cuda
cuInit(0)
cudaSetDevice(owner_dev)
//接收父进程的文件描述符
received_fd = recv_fd(sock);
shareable_fd = reinterpret_cast<void *>(static_cast<uintptr_t>(received_fd))
//将接收的描述符转为handle
cuMemImportFromShareableHandle(&imported, shareable_fd,CU_MEM_HANDLE_TYPE_POSIX_FILE_DESCRIPTOR)
close(received_fd);  
//预留虚拟地址
cuMemAddressReserve(&child_va, alloc_size, 0, 0, 0)
//建立映射，将共享handle对应的物理内存映射到虚拟地址
cuMemMap(child_va, alloc_size, 0, imported, 0)
//设置内存访问权限，可读可写
...
cuMemSetAccess(child_va, alloc_size, &access, 1)
```

## recv_fd

​	接收发送进程的文件描述符，两个文件描述符指向同一个内核对象

```c
struct msghdr msg = {0};
//接收普通数据
char data = 0;
struct iovec iov = {};
iov.iov_base = &data;
iov.iov_len = sizeof(data);

//设置辅助数据
char control[CMSG_SPACE(sizeof(int))];
msg.msg_iov = iov;
msg.msg_iovlen = 1;
msg.msg_control = control;
msg.msg_controllen = sizeof(control);
//接收消息
recvmsg(sock, &msg, 0) 
//解析消息头
struct cmsghdr *cmsg = CMSG_FIRSTHDR(&msg);
//获得接收的文件描述符，与发送进程指向同一个内核对象
received_fd = *(int*)CMSG_DATA(cmsg);
```

## cuMemAddressReserve

​	预留size大小的虚拟地址空间，得到虚拟内存空间的起始地址va

```c
//分配出va地址信息存在memNode中
reserveVirtualMemory(PROCESSOR_FROM_DEVICE(0), &memNode, &info)
    memNode->gpuLogical = iface.address;
    memNode->cpuLogical = iface.address;
    memNode->size = iface.size;
mapMemNode->requestSize = size;
mapMemNode->memNode = memNode;
mapMemNode->vmmAlloc = true;
//将mapMemNode插入到global map中
insertNodeWithAddress(nullptr, mapMemNode, memNode->gpuLogical, size);
```

### reserveVirtualMemory

​	分配出va地址返回给用户

```c
//找到当前进程的database
gckKERNEL_FindDatabase(Kernel, ProcessID, gcvFALSE, &dataBase)
//取出database中的vm
vm = dataBase->vm;
//用户层配置的size和起始地址(start为0由系统自动分配)
args.size = Interface.size
args.start = Interface.start
//true会从cpu分配va，false会将对齐的start作为va
args.createRange = true
args.flags = gcvMAP_FLAG_MAP_POOL
//分配va
gckVM_VARangeAlloc(Kernel, vm, &args, &range)
//返回用户size和va起始地址
Interface.size = range->size
Interface.address = range->start
```

### gckVM_VARangeAlloc

​	对齐起始地址和size，通过vm_mmap分配出va，所有地址和内存信息存在range结构体中，将range作为节点插入到Vm->rangeRoot中（每个vm一个rangeRoot）

```c
//获得当前mmu（从database->vm->vaSpace[i]->mmu取出mmu）
gckKERNEL_GetCurrentMMU(Kernel, gcvTRUE, Vm->processID, &mmu)
//对齐起始地址，得到对齐size
start = gcmALIGN_BASE(Args->start, gcd4K_PAGE_SIZE);
end = gcmALIGN(Args->start + Args->size, gcd4K_PAGE_SIZE);
alignedSize = end - start;  
//vm的vaSpace
range->parent = Vm->vaSpace[Kernel->devID];    
range->start = start;    
if (Args->createRange)  
    //通过vm_mmap分配出va
    gckOS_VARangeAlloc(os, range, alignedSize, &address)
else
    address = start;    
range->start = address;
range->size = alignedSize; 
//将range作为节点插入到Vm->rangeRoot
gcsLIST_Add(&range->rbLink, &Vm->rangeRoot)
```

#### gckOS_VARangeAlloc

​	通过vm_mmap分配出va（logical）

```c
//file为database->vm->vaSpace[i]中的设备文件
filp = (struct file *)Range->parent->filp;
if (Range->flags & gcvMAP_FLAG_MAP_POOL)
    //flags为指定地址映射
    flags |= gcdMAP_POOL;
//通过vm_mmap映射出va，如果Range->start为0由系统分配va
logical = vm_mmap(filp, Range->start, Bytes, PROT_READ | PROT_WRITE, flags, 0);
//检查映射出的va是否有效，其vma虚拟内存区域是否为null，va是否在其vma内
vma = find_vma(mm, logical);
if (!vma)
    return NULL;   
if (logical < vma->vm_start)
    return NULL;
Range->vma = (gctPOINTER)vma;
```

## cuMemCreate



```c

```

## cuMemMap



```c

```

## cuMemSetAccess



```c

```

## cuMemExportToShareableHandle

​	根据物理内存的handle创建包含内存信息的DMA-BUF对象，将它转为可以在进程间共享的fd

```c
//根据handle获取对应的nodeObject
if (handle) 
   gckVIDMEM_HANDLE_Lookup(Kernel, ProcessID, handle, &nodeObject)
//从nodeobject->node中获取size
gckVIDMEM_NODE_GetSize(Kernel, nodeObject, &size)
//增加nodeobject计数
gckVIDMEM_NODE_Reference(Kernel, nodeObject)
//准备DMA的sgtable
_getHostAddrFromVideoMemory(Kernel, ProcessID, nodeObject, &sgtable)   
//创建包含内存信息的DMA-BUF对象，转为fd
zx_mem_buf_export(nodeObject, sgtable, Kernel, size, &fd)    
Interface->u.ExportDmaMemory.fd = fd;        
```

### gckVIDMEM_HANDLE_Lookup

​	从handle获取nodeobject

​	先获取当前进程的database，从database中取得handledatabase，根据handle计算pos从handledatabase中获取handleObject，从handleObject中获得内存节点对象nodeobject

```c
//获取handledatabase
gckKERNEL_FindDatabase(Kernel, ProcessID, gcvFALSE, &database)
handledatabase = database->handleDatabase   
pos =handle - 1
//pos除32作为位图的下标
n = pos >> 5;
//pos与32的余数作为位图的偏移
i = pos & 31;
//如果handle对应的位图状态为0（表示对应内存未占用）返回错误
if (pos >= handledatabase->capacity || (handledatabase->bitmap[n] & (1u << i)) == 0)
    return null
//根据handle获得handleObject
handleObject = handledatabase->table[pos]
//获得nodeobject
nodeobject = handleObject->node
```

### _getHostAddrFromVideoMemory

​	准备DMA的sgtable，获得node对应的cpu物理地址、size、内存块数，配置dma地址为cpu的物理地址

```c
node = nodeobject->node;
//获得node对应的cpu物理地址、size、内存块数
while (node)
	cpuPhisicaladdr[cnt] = node->VidMem.devicePool->poolBase + node->VidMem.offset
	cpuPhisicalsize[cnt] = node->VidMem.reservedBytes
	node = node->VidMem.nextReserved;
	cnt++;
//创建sgtable，内存块数为cnt
sg_alloc_table(table, cnt, GFP_KERNEL);
//遍历sgtable，配置dma地址为cpu的物理地址
for_each_sg(table->sgl, sg, table->nents, i)
	sg_dma_address(sg) = cpuPhisicaladdr[i];
	sg->length = gpuPhisicalsize[i];
```

### zx_mem_buf_export

​	创建DMA-BUF对象（包含handle内存信息的一块缓存区），将DMA-BUF 对象转为fd

```c
//初始化exp_info
DEFINE_DMA_BUF_EXPORT_INFO(exp_info);
//配置私有数据buffer的参数
buffer->kernel = Kernel;
buffer->sg_table = sgtable;
buffer->size = Size;
buffer->nodeObject = NodeObject;
mutex_init(&buffer->lock);
//配置exp_info
exp_info.ops = &zx_dma_buf_ops; //缓存区操作函数(回调函数)
exp_info.size = buffer->size;	//缓存区size
exp_info.flags = O_RDWR;
exp_info.priv = buffer;			//私有数据(handle的一些信息，作为回调函数的参数)
//将一个内核管理的内存区域导出为 DMA-BUF 对象
dmabuf = dma_buf_export(&exp_info);
//将DMA-BUF 对象转为fd
fd = dma_buf_fd(dmabuf, O_CLOEXEC);
```

## cuMemImportFromShareableHandle

​	从共享fd获得DMA-BUF，从DMA-BUF的私有数据获得内存节点nodeobject，分配一个包含nodeobject的handle给用户

```c
//从fd获得DMA-BUF，从DMA-BUF的私有数据获得内存节点nodeObject
zx_mem_buf_import(fd, &nodeObject);
//增加nodeObject的引用计数
gckVIDMEM_NODE_Reference(Kernel, nodeObject)
//分配一个handle，nodeObject作为它的参数
gckVIDMEM_HANDLE_Allocate(Kernel, nodeObject, &handle)
//往database中添加内存handle等信息，向database的list插入record
gckKERNEL_AddProcessDB(Kernel, ProcessID, type, handle, gcvNULL, 0)  
Interface.handle = handle;
```

### zx_mem_buf_import

​	从fd获得DMA-BUF，从DMA-BUF的私有数据获得内存节点nodeObject

```c
//从fd获得DMA-BUF
dmabuf = dma_buf_get(fd);
//从DMA-BUF的私有数据获得内存节点nodeObject
buffer = dmabuf->priv;
nodeObject = buffer->nodeObject;
//减少引用计数，释放
dma_buf_put(dmabuf);
```

## cuIpcGetMemHandle

​	主要适用于cumalloc分配的内存，生成一个共享的handle（name），这个handle包含了当前内存的nodeObject

```c
//根据gpu逻辑地址dptr获取对应的mapMemNode
mapMemNode = findNodeWithAddress(ctx->devPtrMap, dptr)
//生成一个共享的handle（name)
exportMemory(ctx, mapMemNode->memNode->handle, &fd)
//转为CUipcMemHandle类型
os_memcpy(pHandle, &fd, sizeof(fd));
```

### exportMemory

​	根据handle获取对应的nodeObject，分配一个新的handle，nodeObject作为database->table[pos]

```c
//根据handle获得对应的nodeObject
gckVIDMEM_HANDLE_Lookup(Kernel, ProcessID, Interface.handle, &nodeObject)
//增加nodeObject引用计数
gckVIDMEM_NODE_Reference(Kernel, nodeObject)    
//分配一个handle（name),nodeObject作为database->table[pos]
gckKERNEL_AllocateIntegerId(database, nodeObject, &name)
```

### gckKERNEL_AllocateIntegerId

​	分配一个handle，从nameDatabase中取出当前的pos，根据pos遍历nameDatabase中的位图找到未占用的位，设置占用状态，pos+1作为handle(name)

```c
pos=database->nextid  	
n=pos>>5   //pos/32 bitmap中的索引
i=pos&31   //pos%32 bitmap[n]中的第几位
//用32位二进制标记32个id的状态，1表示占用，遍历位图找到未占用的位
while(database->bitmap[n] & (lu<<i))
    pos++
    if (pos == database->capacity) 
        pos = 0;
    n=pos>>5 //bitmap中的索引
    i=pos&31//bitmap[n]中的第几位
database->table[pos]=nodeObject   //nodeObject作为database->table[pos]
database->bitmap[n]|=(lu<<i)      //设置为占用
*Id=pos+1					    //pos+1作为handle
database->nextid=(pos+1)%64       //更新nameDatabase的nextid
database->freecount--
```

## cuIpcOpenMemHandle

​	获取共享handle，从共享handle获得内存信息，返回一个gpu逻辑地址给用户

```c
//从共享handle获得对应的nodeobject，封装为memNode
importMemory(ctx, &memNode, fd)
//lock和映射对应的内存
lockMemoryNode(ctx, memNode)
//返回用户内存的逻辑地址
*pdptr = memNode->gpuLogical;
```

