# cudaMalloc-rtMalloc

​	先runtimebegin 
​	将(void *) ptr转为(CUdeviceptr) ptr作为参数传入drvMemAlloc_v2 

```
(MemAlloc_v2)((CUdeviceptr *)ptr,size)
```

​	进入drvMemAlloc_v2，获得当前context (primary context \ 获得栈stack，栈顶为当前的context)，获得当前stream（传入stream为NULL，获得默认stream—legacystream）

## allocMemory 

​	进入allocDeviceMemory的allocMemory 函数，设置iface的type、size、对齐等信息，第一个KmdIoctl负责分配内存，第二个KmdIoctl负责对分配的内存进行映射，最后将地址等参数配置到pNode结构体

```c
allocMemory(device->processor,memNode,info)
    iface = { gcvHAL_ALLOCATE_LINEAR_VIDEO_MEMORY }; //设置type\size\对齐等信息
    iface.u.AllocateLinearVideoMemory.alignment = ...
	iface.u.AllocateLinearVideoMemory.bytes = ...
	iface.u.AllocateLinearVideoMemory.pool = ...
    ...
    KmdIoCtl(...) //第一个KmdIoCtl负责分配内存
    pNode->handle = iface.u.AllocateLinearVideoMemory.node;
	pNode->pool = (int32_t)iface.u.AllocateLinearVideoMemory.pool;
	...
    iface.command = gcvHAL_LOCK_VIDEO_MEMORY;
    KmdIoCtl(...) //第二个KmdIoCtl负责对分配的内存进行映射
    pNode->cpuLogical = iface.u.LockVideoMemory.memory;//配置内存地址参数
	pNode->gpuLogical = iface.u.LockVideoMemory.address;
	pNode->physical = iface.u.LockVideoMemory.physicalAddress 
    *node = pNode;
```

### _AllocateLinearMemory

​	根据interface的ALLOCATE_LINEAR_VIDEO_MEMORY进行device的dispatch，因为没有符合条件进行kernel的dispatch，进入_Allocatelinearmemory函数

```c
_AllocateLinearMemory(Kernel,ProcessID,*Interface)
	//分配显存内存节点
	gckKERNEL_AllocateVideoMemory(...)
    //为内存节点分配句柄
	gckVIDMEM_HANDLE_Allocate(...)
	gckVIDMEM_NODE_IsContiguous(Kernel, nodeObject, &isContiguous))
    //记录信息到database中
	gckKERNEL_AddProcessDB(Kernel, ProcessID, dbType,gcmINT2PTR(handle), gcvNULL, bytes))
```

#### gckKERNEL_AllocateVideoMemory

​	先设置相关参数传入 gckkernel_Allocvideomemory函数，从内存池中分配（false)，进入gckVIDMEM_AllocateVirtual函数分配virtual内存

reserved memory：为 GPU 预留一段连续的内存

​	Contiguous Pool(Host DDR）：在host DDR中预留连续内存区域给GPU使用（driver被insmod时，操作系统进行预留) 

​	External Pool：PCIE local memory (CPU can access directly)

​	Exclusive Pool：PCIE local memory (CPU cannot access directly)

virtual memory：连续内存耗尽时，从kernel管理的内存分配

​	GFP：使用 alloc_page`/`alloc_pages分配连续或非连续的内存

​	USER：将用户态分配的内存导入 GPU 虚拟地址空间，若 GPU 需要读取这段内存，就可以通过该分配器找到其物理地址，并通过 GPU MMU 映射为 GPU 虚拟地址，从而让 GPU 能够访问这段由 `malloc` 分配的内存

​	DMA：使用 dma_alloc_coherent`/`dma_alloc_wc 分配连续内存

```c
gckKERNEL_AllocateVideoMemory(...)
    ...
    pool = gcvPOOL_LOCAL_INTERNAL;
	loopCount = (gctINT)gcvPOOL_NUMBER_OF_POOLS; //pool的数量
	while (loopCount-- > 0) //遍历pool
        if (pool == gcvPOOL_VIRTUAL) //virtual pool分配内存
            //分配连续内存
            gckVIDMEM_NODE_AllocateVirtual(...,gcvALLOC_FLAG_CONTIGUOUS)
            	//分配内存node
            	gckVIDMEM_AllocateVirtual(Kernel, Flag, Type, bytes, &node)
            	//分配NodeObject
            	gckVIDMEM_NODE_Construct(Kernel, node, Type,Pool, Flag, &nodeObject)
            if (gcmIS_SUCCESS(status))//分配成功break
                 break;
			//分配非连续内存
			gckVIDMEM_NODE_AllocateVirtual(...,gcvALLOC_FLAG_NON_CONTIGUOUS)
             if (gcmIS_SUCCESS(status))//分配成功break
                 break;
		else						//reserved pool
             //获得reserved pool的memory node
         	 status = gckKERNEL_GetVideoMemoryPool(Kernel, pool, &videoMemory);
			if (gcmIS_SUCCESS(status))
                 //分配预留的连续内存
				status = gckVIDMEM_NODE_AllocateLinear(...)
                  	 //分配内存node
                 	 gckVIDMEM_AllocateLinear(...)
                 	 //分配NodeObject
                	 gckVIDMEM_NODE_Construct(...)
             	  if (gcmIS_SUCCESS(status))
    			 	break;
	...
	*Pool = pool;
    *Bytes = bytes;
    *NodeObject = nodeObject;
```

#### gckVIDMEM_AllocateVirtual

​	分配内存节点

​	配置节点参数，分配id作为node参数，进程id作为node参数，调用gckOS_AllocatePagedMemory函数分配物理页内存，最后返回内存节点

```c
gckVIDMEM_AllocateVirtual(Kernel,Flag,Type,Bytes,*Node)
	node = pointer;	//配置node参数
	node->VidMem.kernel = kernel
	node->VidMem.contiguous = (Flag & gcvALLOC_FLAG_CONTIGUOUS)
	node->VidMem.fromUser = (Flag & gcvALLOC_FLAG_FROM_USER)
	... 
	node->VidMem.pool = gcvPOOL_VIRTUAL
	gckOS_NodeIdAssign(Kernel->os, node)//分配id作为node参数
	gckOS_GetProcessID(&node->VidMem.processID)//进程id作为node参数
	//Allocate memory from the paged pool
	gckOS_AllocatePagedMemory(os, Kernel, Flag, Type,...，&node->VidMem.physical)
	*Node = node;                       
```

##### gckOS_AllocatePagedMemory

​	分配物理页内存

​	创建mdl结构体，配置mdl参数（初始化互斥锁、链表头、设备指针），遍历allocator链表的allocator（gfp、user、dma、reserved），调用allocator->ops的Alloc函数指针分配物理页内存（实际调用gfp的函数指针），内存信息作为mdl的priv参数，最后返回mdl地址

```c
gckOS_AllocatePagedMemory(Os, Kernel, Flag, Type,*Bytes, *Gid, *Physical)
	bytes = gcmALIGN(*Bytes, PAGE_SIZE) //对齐size
	numPages = GetPageCount(bytes, 0);  //计算物理页数
	mdl = _CreateMdl(Os, Kernel);	//创建mdl结构体，配置mdl参数
		mdl = kzalloc(sizeof(*mdl),...)
         if (mdl)
             mdl->os = Os;
             atomic_set(&mdl->refs, 1);
             mutex_init(&mdl->mapsMutex);//初始化互斥锁
             INIT_LIST_HEAD(&mdl->mapsHead)//初始化链表头
             INIT_LIST_HEAD(&mdl->rmaHead);
			mdl->device = Kernel->device->dev; //绑定设备
	mdl->fromUser = (Flag & gcvALLOC_FLAG_FROM_USER)
	list_for_each_entry(allocator, &Os->allocatorList, link)//遍历allocator链表的allocator
        //调用allocator->ops的Alloc函数指针分配内存
        status = allocator->ops->Alloc(allocator, mdl, numPages, Flag);
		if (gcmIS_SUCCESS(status)) 
            mdl->allocator = allocator;
            break;
	mdl->dmaHandle = 0; //将dmaHandle参数设置为空
 	mdl->addr = 0;
 	mdl->bytes = bytes;
 	mdl->numPages = numPages
	*Physical = (gctPHYS_ADDR)mdl   //返回mdl地址
```

##### allocatorlist构建

​	在gckOS_Construct函数中调用gckOS_ImportAllocators函数来构建allocator链表

​	遍历allocatorArray结构体数组，构建gfp、user、dma、reserved的allocator，将allocator插入到allocatorlist链表

```c
allocatorArray[] = {
	("gfp", _GFPAlloctorInit)
	("user", _UserMemoryAlloctorInit)
	("dma", _DmaAlloctorInit)
    ("reserved-mem", _ReservedMemoryAllocatorInit)
}
gckOS_ImportAllocators
    INIT_LIST_HEAD(&Os->allocatorList); //初始化allocator链表头
	for (i = 0; i < gcmCOUNTOF(allocatorArray); i++) //遍历allocatorArray结构体数组
        if (allocatorArray[i].construct)
            status = allocatorArray[i].construct(..., &allocator);//构建allocator，绑定allocator->ops函数指针
		    allocator->name = allocatorArray[i].name;
			list_add_tail(&allocator->link, &Os->allocatorList);//将allocator插入到链表
```

##### _GFPAlloc

​	gfp allocator 的Alloc函数指针

​	如果分配连续内存，分配连续的页数为NumPages的物理页内存，将物理页内存映射为DMA地址，物理页缓存模式改为写合并模式；如果分配非连续内存，分配NumPages个物理页内存，将这些物理页与SG表项进行关联，将SG表映射到DMA地址空间，设置物理页为写合并模式

​	最后将连续或非连续的物理页固定防止被换出

```c
_GFPAlloc(Allocator, Mdl, NumPages, Flags)
    contiguous = Flags & gcvALLOC_FLAG_CONTIGUOUS
	mdlPriv = kzalloc(sizeof(*mdlPriv),...) //分配mdlPriv结构体
    if (contiguous)//连续内存
        bytes = NumPages << PAGE_SHIFT
        addr = alloc_pages_exact(bytes,...) //分配连续的物理页内存，返回对应逻辑地址
        mdlPriv->contiguousPages = virt_to_page(addr)//获得对应的page结构体
        mdlPriv->dma_addr = dma_map_page(dev, mdlPriv->contiguousPages,0, ...);//将物理页内存映射为DMA地址
		//物理页缓存模式改为写合并模式
	    set_memory_wc(page_address(mdlPriv->contiguousPages), NumPages)
	else	//非连续内存
         _NonContiguousAlloc(mdlPriv, NumPages, gfp)//分配NumPages个物理页内存
        	pages = kmalloc(NumPages * sizeof(page*),...)
        	for (i = 0; i < NumPages; i++)//遍历NumPages个物理页
                p = alloc_page(Gfp) //分配物理页内存，返回page结构体
                pages[i] = p
             mdlPriv->nonContiguousPages = pages//将NumPages个page结构体作为mdlPriv参数
        //基于物理页数组分配NumPages个sg_table结构体，将物理页与SG表项进行关联
        sg_alloc_table_from_pages(&mdlPriv->sgt, mdlPriv->nonContiguousPages,
                                           NumPages, 0, NumPages << PAGE_SHIFT,...)+
         //将SG表映射到DMA地址空间
		dma_map_sg(dev, mdlPriv->sgt.sgl, mdlPriv->sgt.nents, ...);
		//设置物理页为写合并模式
		set_pages_array_wc(mdlPriv->nonContiguousPages, NumPages)
		for (i = 0; i < NumPages; i++)
            if (contiguous)
    			page = nth_page(mdlPriv->contiguousPages, i);//取出第i个物理页
			else
    			 page = mdlPriv->nonContiguousPages[i];//第i个物理页
			SetPageReserved(page);//将物理页固定防止被换出
		Mdl->priv = mdlPriv;
		Mdl->numPages = NumPages;
```

##### _DmaAlloc

​	dma allocator 的Alloc函数指针

​	分配连续物理内存，将内存设为写合并模式，将物理页内存映射为DMA地址

```c
_DmaAlloc(Allocator, Mdl, NumPages, Flags)
	gckOS_Allocate(os, sizeof(struct mdl_dma_priv), (gctPOINTER *)&mdlPriv)//分配mdlPriv结构体
     //分配连续物理内存，将内存设为写合并模式，将物理页内存映射为DMA地址，返回逻辑地址
     mdlPriv->kvaddr = dma_alloc_wc(dev, NumPages * PAGE_SIZE, &mdlPriv->dmaHandle, gfp)
     set_memory_wc((unsigned long)mdlPriv->kvaddr, NumPages)
     Mdl->priv = mdlPriv;
     Mdl->dmaHandle = mdlPriv->dmaHandle;//DMA地址
```

#### gckVIDMEM_NODE_Construct

​	分配NodeObject

​	内存节点作为NodeObject参数，创建node锁，创建node sync信号，将node插入videoMem链表，初始化mapInfoHead链表头，最后返回NodeObject

```c
gckVIDMEM_NODE_Construct
	node->node = VideoNode;//配置参数
	node->kernel = Kernel;
	node->pool = Pool
    ...
	gckOS_AtomConstruct(os, &node->reference)
	gckOS_CreateMutex(os, &node->mutex) //创建node锁
	for (i = 0; i < 2; i++) 
        gckOS_CreateSignal(os, gcvFALSE,&node->sync[i].signal)//创建node sync信号
	gckOS_AtomSet(os, node->reference, 1)
	gcsLIST_Add(&node->link, &Kernel->db->videoMemList);//将node插入videoMem链表
	gcsLIST_Init(&node->mapInfoHead)//初始化mapInfoHead链表头
	*NodeObject = node //返回NodeObject
```

#### gckVIDMEM_AllocateLinear

​	分配预留的连续内存

​	根据memtype从Memory->mapping[Type]获得当前bank，每个Memory->mapping[Type]有8个bank

​	遍历当前bank的空闲VidMem节点，找到有足够size的节点，如果未找到遍历低bank的VidMem节点或遍历高bank的VidMem节点，如果节点剩余size大于Memory阈值，从当前节点分离出子节点，将子节点插入当前节点Node后，将子节点插入当前空闲节点后，将当前节点从空闲的VidMem双向链表中删除，配置node参数

```c
gckVIDMEM_AllocateLinear
	Alignment = gcmALIGN(Alignment, gcd4K_PAGE_SIZE);
	Bytes = gcmALIGN(Bytes, gcd4K_PAGE_SIZE);
	if (Bytes > Memory->freeBytes)
		status = gcvSTATUS_OUT_OF_MEMORY
	bank = Memory->mapping[Type];//获得当前bank
	node = _FindNode(Kernel, Memory, bank, Bytes, Type, &alignment);//找足够size的节点
	if (node == gcvNULL)
        for (i = bank - 1; i >= 0; --i)//遍历低bank
            node = _FindNode(Kernel, Memory, i, Bytes, Type, &alignment)//找足够size的节点
            if (node != gcvNULL)
                break;
	if (node == gcvNULL)
        for (i = bank + 1; i < gcmCOUNTOF(Memory->sentinel); ++i)//遍历高bank，一共8个bank
            node = _FindNode(Kernel, Memory, i, Bytes, Type, &alignment)//找足够size的节点
            if (node != gcvNULL)
    	    	break;
	if (node->VidMem.bytes - Bytes > Memory->threshold)//节点剩余size大于Memory阈值
     	_Split(Memory->os, node, Bytes);	//从当前节点分离出子节点
	//将当前节点从空闲的VidMem双向链表中删除
	node->VidMem.prevFree->VidMem.nextFree = node->VidMem.nextFree;
    node->VidMem.nextFree->VidMem.prevFree = node->VidMem.prevFree;
    node->VidMem.nextFree = gcvNULL;
    node->VidMem.prevFree = gcvNULL;
	gckOS_GetProcessID(&node->VidMem.processID) //配置进程pid参数
	node->VidMem.contiguous =	//配置node参数
	node->VidMem.fromUser =
	node->VidMem.secure = 
	...
	gckOS_NodeIdAssign(Kernel->os, node)//配置node的id参数
    gckOS_RequestReservedMemoryArea(...,Memory->physical,...,&node->VidMem.physical)
	*Node = node
```

##### _FindNode

​	遍历当前bank的空闲VidMem节点，找到有足够size的节点

```c
_FindNode
    //遍历空闲的VidMem节点
    for (node = Memory->sentinel[Bank].VidMem.nextFree;node->VidMem.bytes != 0;node = node->VidMem.nextFree)
        if (node->VidMem.bytes < Bytes)
            continue;
        if (node->VidMem.bytes >= Bytes)//找到有足够size的节点
            return node;
```

##### _Split

​	当当前节点的剩余size大于阈值时，从当前节点分离出子节点，将子节点插入当前节点Node后，将子节点插入当前空闲节点后

```c
_Split(Os, Node, Bytes)
    node->VidMem.offset = Node->VidMem.offset + Bytes; //子节点偏移量
    node->VidMem.bytes = Node->VidMem.bytes - Bytes; //子节点size
	...											//子节点参数
	node->VidMem.next = Node->VidMem.next;//将子节点插入当前节点Node后
    node->VidMem.prev = Node;
    Node->VidMem.next = node;
    node->VidMem.next->VidMem.prev = node;

	node->VidMem.nextFree = Node->VidMem.nextFree;//将子节点插入当前空闲节点后
    node->VidMem.prevFree = Node;
    Node->VidMem.nextFree = node;
    node->VidMem.nextFree->VidMem.prevFree = node;

	Node->VidMem.bytes = Bytes;					//将Bytes设为当前节点size
```

#### gckVIDMEM_HANDLE_Allocate

​	返回到_AllocateLinearMemory函数，进入gckVIDMEM_HANDLE_Allocate函数

​	获得当前processid对应的database，获得database中的handledatabase

​	根据pos遍历handledatabase中的位图找到未占用的位，设置占用状态，pos+1作为handle

```c
gckVIDMEM_HANDLE_Allocate(kernel,nodeobject,handle)
	handle->reference=1 //将handle的reference设为1
    gckOS_GetProcessID(&processID) //获得当前PID
	gckkernel_findhandledatabase(...)  //获得当前processid对应的database，获得database中的handledatabase
    //根据pos遍历handledatabase中的位图找到未占用的位，设置占用状态，pos+1作为handle
    gckkernel_allocateintegerid(handledatabase,handleobject,&handle)
    //返回handle
	handleObject->node = Node;
    handleObject->handle = handle;
    *Handle = handle;
```

##### gckKERNEL_FindHandleDatabase

​	计算当前进程PID的hash值，从当前hash对应的database开始遍历database链表，找到当前processid对应的database，获得database中的handledatabase

```c
gckKERNEL_FindHandleDatabase
    slot = ProcessID % gcmCOUNTOF(Kernel->db->db)//计算当前进程PID的hash值
    gckOS_AcquireMute(...,Kernel->db->dbMutex,...)//加锁
    //从当前hash对应的database开始遍历database链表
    for (previous = gcvNULL, database = Kernel->db->db[slot];database != gcvNULL;database 		  = database->next)
        if (database->processID == ProcessID)//找到与当前进程PID相同的database
                break;
		previous = database;
	if (previous != gcvNULL) //将database插入到当前hash链表头
         previous->next = database->next;
		database->next = Kernel->db->db[slot];
		Kernel->db->db[slot] = database;
	gckOS_ReleaseMutex(...,Kernel->db->dbMutex) //解锁
	*HandleDatabase = database->handleDatabase;
    *HandleDatabaseMutex = database->handleDatabaseMutex;     
```

##### gckKERNEL_AllocateIntegerId

​	从handleDatabase中取出pos，根据pos遍历handleDatabase中的位图找到未占用的位，设置占用状态，pos+1作为handle

```c
gckKERNEL_AllocateIntegerId(database, Pointer, *Id)
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
	   database->table[pos]=Pointer   //handobject作为database->table[pos]
       database->bitmap[n]|=(lu<<i)   //设置为占用
       *Id=pos+1					//pos+1作为handle
       database->next=(pos+1)%64   //更新handledatabase->nextid
       database->freecount--
```

​	设置interface

<img src="C:\Users\OT\AppData\Roaming\Typora\typora-user-images\image-20250908105748028.png" alt="image-20250908105748028" style="zoom:50%;" />

#### gckKERNEL_AddProcessDB

​	配置当前PID database的record参数，更新vidMem、vidMemType、vidMemPool的size、count

```c
gckKERNEL_AddProcessDB(kernel,processid,type,handle,null,size)	
	//根据type解码出vidmemtype\vidmempool等
    //type=gcvbd_video_memory
    gckKERNEL_FindDatabase(...) //获得当前processid对应的database
    gckKERNEL_NewRecord(...)       //分配一个record到database
    record->kernel=kernel	//设置record参数
    record->type=type
    record->data=handle
    record->physical=pthsical
    record->bytes=size

    count = &database->vidMem //更新vidMem的size、count
    count->totalBytes += Size;
    count->bytes += Size;
    count->allocCount++;
	count = &database->vidMemType[vidMemType]; //更新vidMemType的size、count
 	count->totalBytes += Size;
	count->bytes += Size
	count->allocCount++;
	count = &database->vidMemPool[vidMemPool]; //更新vidMemPool的size、count
	count->totalBytes += Size;
	count->bytes += Size;
     count->allocCount++;
```

### _LockVideoMemory

​	进入allocMemory 函数的第二个KmdIoCtl，interface输入给KmdIoctl--Ioctl，根据interface的gcvHAL_LOCK_VIDEO_MEMORY进入_LockVideoMemory函数，对之前内存节点的物理页内存进行映射，映射到进程中的虚拟地址空间，得到内存对应的CPU逻辑地址，根据CPU的物理地址获得GPU的物理地址，通过mmu映射得到GPU的逻辑地址，配置当前PID database的record参数

​	GPU的逻辑地址作为返回值

```c
_LockVideoMemory
    //获得当前processid对应的database，获得database中的handledatabase,从handleobject获得对应的nodeObject
    gckVIDMEM_HANDLE_Lookup(Kernel, ProcessID, handle, &nodeObject)
    //增加node的reference
    gckVIDMEM_NODE_Reference(Kernel, nodeObject)
    //Lock for userspace CPU userspace
    gckVIDMEM_NODE_LockCPU(Kernel, nodeObject,...,gcvTRUE, &logical)
    //CPU逻辑地址作为Interface参数                                         
    Interface->u.LockVideoMemory.memory = gcmPTR_TO_UINT64(logical)
    //Lock for GPU address
    gckVIDMEM_NODE_Lock(Kernel, nodeObject, &address)
    //Get CPU physical address
    gckVIDMEM_NODE_GetCPUPhysical(Kernel, nodeObject, 0, &physical)
	gckVIDMEM_NODE_GetGid(Kernel, nodeObject, &gid)
    //GPU逻辑地址和CPU物理地址作为Interface参数
    Interface->u.LockVideoMemory.address = address;
    Interface->u.LockVideoMemory.physicalAddress = physical;
    Interface->u.LockVideoMemory.gid = gid;
	//配置当前PID database的record参数
	gckKERNEL_AddProcessDB(...)
	gckVIDMEM_HANDLE_Reference(Kernel, ProcessID, handle)               
```

#### gckVIDMEM_HANDLE_Lookup

​	获得当前processid对应的database，获得database中的handledatabase和handledatabasemutex，根据handle得到pos=handle-1，database->table[pos]作为handleObject，从handleobject获得对应的nodeObject

```c
gckVIDMEM_HANDLE_Lookup(Kernel, ProcessID,Handle, *Node)
    gckKERNEL_FindHandleDatabase(Kernel, ProcessID,&database, &mutex)
    gckKERNEL_QueryIntegerId(database, Handle,(gctPOINTER *)&handleObject)
    node = handleObject->node;
	*Node = node;
```

#### gckVIDMEM_NODE_LockCPU

​	对于virtual pool或reserved pool，在进程创建新的虚拟内存区域vma，将连续或非连续的物理页映射到vma，返回映射后的物理页CPU逻辑地址

```c
gckVIDMEM_NODE_LockCPU
    node = NodeObject->node
    memory = node->VidMem.parent
    if (memory && memory->object.type == gcvOBJ_VIDMEM) //virtual pool
        //对物理页内存进行映射 
        gckOS_LockPages(os, node->poolPhysical,node->poolSize,gcvFALSE, &logical)
        	mdl = (PLINUX_MDL)Physical //mdl结构体地址
        	allocator = mdl->allocator
        	//遍历mdl->mapshead链表找到与当前进程PID相同的mdlMap
        	mdlMap = FindMdlMap(mdl, _GetProcessID()) 
        	//调用MapUser函数指针对内存进行映射
        	allocator->ops->MapUser(allocator, mdl, mdlMap, Cacheable)
        	mdlMap->count++;
			//返回映射后的逻辑地址
			*Logical = mdlMap->vmaAddr 
	else											//reserved pool
        gckKERNEL_MapVideoMemory(Kernel, gcvTRUE, pool,physical,offset,bytes,&logical)
        	switch (Pool)
                case gcvPOOL_LOCAL_EXTERNAL:
					//获得对应池的mdl结构体
			   case gcvPOOL_LOCAL_INTERNAL:
			   		//获得对应池的mdl结构体
                ...
             //对物理页内存进行映射
             gckOS_LockPages(Kernel->os, physHandle, bytes, gcvFALSE, &logical)
             *Logical = logical
	node->VidMem.logical = logical
```

##### _GFPMapUser

​	gfp_allocator的_GFPMapUser函数指针，在进程创建新的虚拟内存区域vma，将连续或非连续的物理页映射到vma，返回映射后的逻辑地址(vma的地址)

```c
_GFPMapUser(Allocator, Mdl, MdlMap)
    userLogical = vm_mmap(...,Mdl->numPages * PAGE_SIZE,...)//在进程创建新的虚拟内存区域vma
    down_write(&current_mm_mmap_sem) //对进程的内存描述符加锁
    vma = find_vma(current->mm, (unsigned long)userLogical)//返回userLogical所在的vma
    _GFPMmap(Allocator, Mdl, Cacheable, 0, 0, Mdl->numPages, vma)//将物理页映射到vma
    up_write(&current_mm_mmap_sem) //解锁
    MdlMap->vmaAddr = userLogical
```

###### _GFPMmap

​	将连续或非连续的物理页映射到vma

```c
_GFPMmap
	*mdlPriv = Mdl->priv
    if (mdlPriv->contiguous)//连续物理页
        start = vma->vm_start//虚拟内存区域首地址
        pfn = page_to_pfn(mdlPriv->contiguousPages)//物理页框数
        //将物理内存映射到虚拟内存区域
        remap_pfn_range(vma, start,pfn,numPages << PAGE_SHIFT,vma->vm_page_prot)
	else				  //不连续物理页
        start = vma->vm_start
        for (i = 0; i < numPages; ++i)//遍历numPages个物理页
            pfn = page_to_pfn(mdlPriv->nonContiguousPages[i + skipPages])//物理页框数
            //将物理内存映射到虚拟内存区域
            remap_pfn_range(vma, start, pfn, PAGE_SIZE, vma->vm_page_prot)
            start += PAGE_SIZE
```

#### gckVIDMEM_NODE_Lock

​	通过mdl获得物理页的起始物理地址作为CPU物理地址，通过CPU物理地址判断地址范围，得到GPU的物理地址，最后通过mmu得到GPU的逻辑地址

```c
gckVIDMEM_NODE_Lock(Kernel,NodeObject,*Address))
    node = NodeObject->node
    gckVIDMEM_LockVirtual(Kernel, Node, *Address)
        //获得当前mmu
        gckKERNEL_GetCurrentMMU(...) 
        //获得mdl地址
        gckVIDMEM_GetMemoryHandle(Kernel, Node, &memoryHandle)
        gckVIDMEM_GetOffset(Kernel, Node, &offset)
        //获得物理页的起始物理地址（CPU物理地址）
        gckOS_GetPhysicalFromHandle(os, memoryHandle, offset, &physicalAddress)
    		if (mdlPriv->contiguous)
                *Physical = page_to_phys(nth_page(mdlPriv->contiguousPages, index))
             else
                 *Physical = page_to_phys(mdlPriv->nonContiguousPages[index])
        //通过CPU的物理地址得到GPU的物理地址，通过mmu得到GPU逻辑地址
        _ConvertPhysical(Kernel, mmu, ..., physicalAddress,&Node->VidMem.addresses[index])
        ...
        *Address = Node->VidMem.addresses[index]
```

##### _ConvertPhysical

​	通过CPU物理地址判断地址范围，得到GPU的物理地址，最后通过mmu得到GPU的逻辑地址

```c
_ConvertPhysical(Kernel，Mmu，Node，VidMemBlock，PhysicalAddress，*Address))
	//通过CPU的物理地址得到GPU的物理地址
	gckOS_CPUPhysicalToGPUPhysical(...)
    	platform->ops->getGPUPhysical(platform, PlatformIndex, CPUPhysical, GPUPhysical)
    //得到GPU逻辑地址
    gckMMU_IsFlatMapped(Mmu, physical,bytes, &flatMapped, &address)
    *Address = address
```

###### _GetGPUPhysical

​	getGPUPhysical函数指针，获得pci的chip结构体，调用chip的函数指针

```c
_GetGPUPhysical
	pdev = pcie_platform->pcie_info[PlatformIndex].pdev //pci设备
	*chip = pci_get_drvdata(pdev)	//获得pci的chip结构体
	dev_paddr = chip->ops.cpu_paddr_to_dev_paddr(chip, cpu_paddr)//调用chip的函数指针得到GPU地址
```

###### zixia_chip_cpu_paddr_to_dev_paddr

​	chip的cpu_paddr_to_dev_paddr函数指针，根据CPU的物理地址范围是否在outbound内存地址、reserved的CPU地址或interleave的CPU地址范围内，得到对应的GPU物理地址（GPU基址+CPU地址相对CPU基址的偏移量）

```c
zixia_chip_cpu_paddr_to_dev_paddr
	if((cpu_paddr < chip->ddr_bar.base) || (cpu_paddr >= (chip->ddr_bar.base + chip->ddr_bar.len))) //判断CPU地址范围
        //GPU地址=outbound内存地址(IO die扩展)
        dev_paddr = cpu_paddr_to_dev_outbound_addr(chip, cpu_paddr); 
	else if((cpu_paddr>=chip->vram_reserved.base) && (cpu_paddr<(chip->vram_reserved.base + chip->vram_reserved.size)))//CPU地址在reserved地址范围内
        //GPU地址=reserved的GPU基址+CPU地址相对reserved基址的偏移
        dev_paddr = chip->vram_reserved.gpu_base + (cpu_paddr-chip->vram_reserved.base);
	else				//CPU地址在interleave地址范围内
        for (i = 1; i < NOC_INTERLEAVE_VIEW_TYPE_NUM; i++)//遍历interleave
            //当前interleave的CPU基址
            view_start_cpu_addr =chip->noc_interleave_view_cpu_base[i];
		   //当前interleave的CPU末地址
            view_end_cpu_addr=view_start_cpu_addr+chip->noc_interleave_view_ddr_sizes[i]
            //判断CPU地址是否在当前interleave地址范围内
		    if (cpu_paddr>=view_start_cpu_addr && cpu_paddr<(view_end_cpu_addr))
                //GPU地址=当前interleave的GPU基址+CPU地址相对interleave基址的偏移
                dev_paddr=chip->noc_interleave_view_bases[i]+(cpu_paddr-view_start_cpu_addr)
                break;
```

###### gckMMU_IsFlatMapped

​	循环多个GPU的物理内存区域，看GPU物理地址是否在当前mmu的GPU物理内存区域内，最后的GPU地址是当前mmu的GPU物理内存映射区域的基址+（物理地址相对当前mmu的GPU物理内存映射区域的偏移），最后得到GPU的逻辑地址

```c
gckMMU_IsFlatMapped
	for (i = 0; i < Mmu->gpuPhysicalRangeCount; i++) 
    	if (Physical >= Mmu->gpuPhysicalRanges[i].start &&(Physical + Bytes - 1 <= Mmu->gpuPhysicalRanges[i].end)) 
        	inFlatmapping = gcvTRUE;
 
        	if (Address) 
            	*Address = Mmu->gpuPhysicalRanges[i].vStart +(gctADDRESS)(Physical - Mmu->gpuPhysicalRanges[i].start); 
```

gckVIDMEM_NODE_GetCPUPhysical

gckos_mappagesex(...)函数

通过将GPU逻辑地址页表映射得到物理地址，将GPU的物理地址赋给node->vidmem.pagetables[index]

​	将得到的CPU\GPU逻辑信息更新到interface
​	将interface中信息更新到node中，返回最后的node(包含各种内存地址等信息)
​	insertnodewithaddress(将node插入到globalmap全局映射表，键值对(GPU逻辑地址，size）=node)
​	最后返回GPU的逻辑地址

<img src="C:\Users\OT\AppData\Roaming\Typora\typora-user-images\image-20250905180952443.png" alt="image-20250905180952443" style="zoom: 50%;" />

## interleave

​	DIE0 interleave中将8*256=2K分给一个MP，如果有4个MP，分别将4个2K数据分给MP0-MP3，每个MP的2K分为8个target，每个target占256字节

​	将DIE0 interleave内存数据分为多份，分给多个MP，加大内存带宽

<img src="C:\Users\OT\AppData\Roaming\Typora\typora-user-images\image-20260414110220282.png" alt="image-20260414110220282" style="zoom:50%;" />

​	





























