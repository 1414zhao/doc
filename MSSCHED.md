## UMD

### createRingBuffer函数

​	在stream--createhalqueue中创建环形缓冲区，ringbuffer作为halqueue的参数

```c
createHalQueue
    ...
	createRingBuffer(processor, 0, 0, &hq->ringBuffer)
    ...
```

​	createRingBuffer函数，分配环形缓冲区内存节点，作为halqueue的ringbuffer参数，进入KMD

```c
createRingBuffer(rocessor, channelId, priority, buffer)
    p = processor->ringBuffer[priority * channelId];
	if(!p)
        allocMemory(...) //分配缓冲区内存节点到p->buffer
        //将环形缓冲区的readidx地址、writeidx地址、数据起始地址存在p->rbInst
        hrbQueueCreate(&p->rbInst, (void*)p->buffer->cpuLogical, size) 
	       queue->readIndexAddr = (uint32_t *)data;
            queue->writeIndexAddr = queue->readIndexAddr + 1;
            *queue->readIndexAddr = *queue->writeIndexAddr = 0;
            queue->baseAddr = (uint8_t *)data + 8
            atomic_init(&queue->pktWriteIndex, 0);
		   atomic_init(&queue->queueWriteIndex, 0);
        iface = { gcvHAL_RINGBUF_REGISTER }
	    iface.u.RegisterRingBuf.add = true;
	    iface.u.RegisterRingBuf.node = p->buffer->handle;
         ...
         //进入KMD
         KmdIoCtl(KmdHandle(processor, iface),
                IOCTL_GCHAL_INTERFACE,
                &iface, sizeof(iface),
                &iface, sizeof(iface));
     *buffer = p
```

### gckMSSCHED_RingBufRegister函数

​	创建当前进程的processnode结构体，将processnode插入attachprocesslist，将缓存区地址、size、readidx、writeidx等参数作为processnode的exchangeMemories参数和ringbuf参数

```c
gckMSSCHED_RingBufRegister(Device, Interface)
	gckOS_GetProcessID() //获得当前的进程id
    if (processNode == gcvNULL) {
        _CreateProcessNode(msSched, processID, &processNode);//创建processnode
        _CheckProcessWithRemovalSiganlAvail_WAIT(...) //等待一个空闲的processremove信号
        gckOS_AtomIncrement(...) //增加processnode引用次数
        gcsLIST_Add(&processNode->listEntry, &resources->attachedProcessList)//将processnode插入到attachedProcessList
         _AddExchangeMemory(...) 
            //将缓冲区内存节点赋给processnode的exchangememories参数，包括对应的逻辑地址、物理地址、size等
            ...
            ProcessNode->exchangeMemories[priority].node=        
		   ProcessNode->exchangeMemories[priority].videoMemNode=
            ProcessNode->exchangeMemories[priority].logical=
            ...
            _rbL1Init()
            	//将缓冲区的起始地址、readidx地址、write地址和数据地址作为processnode的ringbuf参数
            	rbL1->startLogical  = startLogical;
                rbL1->offsetStart   = offsetStart;
                rbL1->offsetRD      = offsetRD;
                rbL1->offsetWD      = offsetWD;
        		//将缓冲区数据地址、数据size、readidx、writeidx作为ringbuf->base参数
        		_rbInit(...)
                    rb->buffer      = buf;
                    rb->size        = elementNum;
                    rb->readIndex   = 0;
                    rb->writeIndex  = 0;
                    rb->elementSize = elementBytes;  
```

### commitCmdBuffer函数

​	在commitcmdbuffer函数中将fence信息、cmdtype、copyinfo、cmdbuffer等信息存入hrbPkt中，将hrbPkt存入halqueue的环形缓冲区ringbuffer中

```c
...
//将cmd中的类型等信息传进hrbPkt
hrbPacketInfoFill(pkt,cmd,hq)
	packetType = commandTypeToHrbPacketType(halCmd)//获得cmd的类型到packetType
    //配置pkt的pid和streamid
    pkt->processId =os_getCurrentProcessID
	pkt->id = hq->id
    hrbPacketFenceFill(...) //配置hrbPkt的fence参数 ？？？
    	//当lastpktType是DMA时，配置pkt的dmaSendFence参数（gpuLogical、expectedVal、mode）
    	//当lastpacketType和当前的packetType不同时，配置pkt的waitfence参数（gpuLogical、expectedVal、mode）
    switch (packetType)
    {
        case HRB_PACKET_TYPE_CM:  //kernel type
        case HRB_PACKET_TYPE_KERNEL:
        case HRB_PACKET_TYPE_KERNEL_BLT:
            pkt->type = packetType; //将packetType作为hrbPkt的type参数
            hq->lastHrbpktType = pkt->type; //pkt->type作为lastpacketType
            break;
         case HRB_PACKET_TYPE_DMA_COPY:  //DMA copy type
         	cpInfo = &pkt->copyInfo
             //将region、src\dst的地址、stride和origin存入hrbPkt的copyinfo参数中
              ...
              pkt->type = packetType;
			 hq->lastHrbpktType = pkt->type;
          case HRB_PACKET_TYPE_DMA_FILL:
              //将region、data、dst的地址、stride和origin存入hrbPkt的copyinfo参数中
      }         
//如果cmd类型为kernel，将每个die的1个4cluster mp中的cmdbuffer信息存入hrbPkt的cmdBufferInfo参数
if ((hrbPkt.type & HRB_PACKET_TYPE_KERNEL) || (hrbPkt.type & HRB_PACKET_TYPE_CM)
        || (hrbPkt.type & HRB_PACKET_TYPE_KERNEL_BLT))
     for (int d = 0; d < dieCount; d++)
         coreCount = getCoreCount(processor, coCore, d);
		idx = d * coreCount;	//每个die的1个4cluster mp
		p = &hrbPkt.cmdBufferInfo[d]
         p->videoMemNode = cmdPool->commandBufferPool->handle //对应的cmdpool内存节点
         p->reservedHead = reservedHead
         p->address = getDeviceAddr(cmdPool->commandBufferPool, processor) +  subCommit[idx].commandBuffer.startOffset  //对应mp cmdbuffer的设备地址    
         p->logical = getCPUAddr(cmdPool->commandBufferPool, processor) +  subCommit[idx].commandBuffer.startOffset //对应mp cmdbuffer的逻辑地址     
         p->size = cmdSize
//将hrbPkt中的信息存入环形缓冲区ringbuffer
hrbQueueSubmitPacket(queue->ringBuffer->rbInst, &hrbPkt) 
	hrbPacket = &((hrbPacket_PTR)hrbQueue->baseAddr)[pktWriteIndex & (hrbQueue->size - 1)];//根据writeidx找到要写入的环形缓冲区的数组索引
    *hrbPacket = *packet; //写入hrbPkt中的信息
	//writeidx+1，更新writeidx
//进入KMD进行commit_command和commit_event
KMD(...)
```

## KMD

​	在gckGALDEVICE_Construct函数中创建MSSCHED

```c
gckGALDEVICE_Construct
	... //创建赋值device、kernel、hardware、command、event等结构体
	gckMSSCHED_Construct(device, &device->msSched, gal_device->args.hwDevCounts[device->platformIndex])
	*Device = gal_device
```

### gckMSSCHED_Construct函数

```c
gckMSSCHED_Construct(gckDEVICE Device, gckMSSCHED *MSSched, gctUINT32 TotalDevCount)
    //创建msSched结构体
    ...
    //将l1Schedule、l2Schedule、resSchedule、resources结构体作为msSched参数
    msSched->l1Schedule  = (gctPOINTER)&g_L1Schedules[platformId]
	msSched->l2Schedule  = (gctPOINTER)&g_L2Schedules[platformId]	
    msSched->resSchedule  = (gctPOINTER)&g_ResSchedules[platformId]
	msSched->resources    = (gctPOINTER)&g_Resourcess[platformId]
    /* Construct resource object. */
    _ConstructResources(msSched)
    /* Construct l1schedule object. */
    _ConstructL1Schedule(msSched)
    /* Construct l2schedule object. */
    _ConstructL2Schedule(msSched)
    /* Construct resschedule object. */
    _ConstructResSchedule(msSched)
    ...
```

#### _ConstructResources函数

​	配置resources结构体（创建位图、procremove等信号）

```c
_ConstructResources(MSSched)
    //创建ProcessList和ProcRemoveSignals锁
	gckOS_CreateMutex(MSSched->os, &resources->mutexProcessList))
	gckOS_CreateMutex(MSSched->os, &resources->mutexProcRemoveSignals))
    //为ProcRemove Signal创建位图,bitmap为1时表示signal被占用
    gckOS_CreateBitmap(...)
	gckOS_BitmapZero(...)
    //创建slot free signal
    gckOS_CreateSignal(MSSched->os, gcvFALSE, &resources->signalFreeProcRemoveSlotAvail)
    //创建ProcRemove signal
    for (i = 0; i < 8; i++)
        gckOS_CreateSignal(MSSched->os, gcvFALSE, &resources->signalsProcRemove[i])
    gcsLIST_Init(&resources->attachedProcessList); //初始化附加进程list双向链表
	gcsLIST_Init(&resources->pendingDettachProcessList); //初始化分离进程list双向链表
```

#### _ConstructL1Schedule函数

​	配置l1Schedule结构体，创建L1Schedule线程

```c
_ConstructL1Schedule
    l1Schedule = MSSched->l1Schedule
    //Create the L1 Schedule mutex
	gckOS_CreateMutex(MSSched->os, &l1Schedule->mutexSchedule)
    //创建L1Schedule线程
    gckOS_CreateThread(MSSched->os, _L1ScheduleThread, MSSched, &l1Schedule->thread)
```

##### L1Schedule线程

​	调用_L1ScheduleDoWork函数，根据L1 环形缓冲区中cmd的类型分配taskQueue，将L1 环形缓冲区中的数据存进L2 的环形缓冲区taskQueue->ringBuf

```c
_L1ScheduleThread
    while(1){
            gckOS_AcquireMutex(msSched->os, l1Schedule->mutexSchedule)
             _L1ScheduleDoWork(msSched)
            gckOS_ReleaseMutex(msSched->os, l1Schedule->mutexSchedule)
        }
_L1ScheduleDoWork(MSSched)
    //遍历attachedProcessList
     gcmkLIST_FOR_EACH_SAFE(processHead, processHeadTmp, attachedProcessLst)
    	//从attachedProcessList中取出processNode
    	processNode = gcmCONTAINEROF(processHead, gckMSSchedProcessNode, listEntry)
    	if (processNode)
            //ringbuffer的逻辑地址
            logical = processNode->exchangeMemories[l1Schedule->curPriority].logical
            if (logical != gcvNULL)
                rbL1 = &processNode->ringBuf[l1Schedule->curPriority]
                _rbL1LoadWriteIndex(rbL1) //从L1 ringbuffer的writeidx地址获得writeindex
                //将L1的ringbuffer中的数据存进L2的ringbuffer
                _L1CopyFromExchangeMem(MSSched, processNode, rbL1) 
                	//从L1ringbuffer中获得packType
                	rbPeekBytes(&rbL1->base, &packType, gcmSIZEOF(packType))
                	//Get taskQueue 每个packType有各自的taskQueue
                	taskQueue = l2Schedule->taskQueues[packType]
                	gckOS_BitmapFindFirstZeroBit(&freeSlot) //根据位图获得空闲的task slot
                	gckOS_BitmapSetBit() //更新位图
                	taskDesc = taskQueue->tasks[freeSlot] 
               	 	rbRead(&rbL1->base, (gctPOINTER)taskDesc, 1) //从L1环形缓冲区读取数据到taskDesc中，L1 readindex加1
                	taskDesc->processNode = ProcessNode //ProcessNode作为taskDesc的参数
                	//Update taskQueue submitCount
                	//Update process refCnt
                	rbWrite(&taskQueue->ringBuf, (gctPOINTER)&taskDesc, 1)//将taskDesc写入L2的环形缓冲区，L2 writeindex加1
                	_rbL1StoreReadIndex(rbL1)//将readindex写入L1 ringbuffer的readindex地址
                	//唤醒L2 schedule
                	gckOS_Signal(MSSched->os, taskQueue->signalSchedule, gcvTRUE)
                	//从attachedList中删除processNode，将processNode添加到dettachedList
                	_CheckMoveToPendingRemomalProcess_NOLOCK(...)
```

#### _ConstructL2Schedule函数

​	创建Kernelqueue、CMqueue、DMAqueue作为l2Schedule的taskQueues参数，为每个taskQueues创建L2Schedule线程

```c
_ConstructL2Schedule(MSSched)
    //Create the L2 Schedule mutex
	gckOS_CreateMutex(MSSched->os, &l2Schedule->mutexSchedule)
    for (i = 0; i< 3; i++)
        if (i == 0)
            //创建Kernelqueue
            gckMSSCHED_KernelQueueConstruct(MSSched, &l2Schedule->taskQueues[i])
        else if (i == 1)
            //创建CMqueue
            gckMSSCHED_CMQueueConstruct(MSSched, &l2Schedule->taskQueues[i])
        else
            //创建DMAqueue
            gckMSSCHED_DMAQueueConstruct(MSSched, &l2Schedule->taskQueues[i])
        //创建L2Schedule线程
        gckOS_CreateThread(MSSched->os, _L2ScheduleCommonThread, l2Schedule->taskQueues[i], &l2Schedule->taskQueues[i]->thread)
```

##### MSSCHED_TaskQueueConstruct函数

​	配置taskqueue参数，为不同类型的taskqueue设置对应的freetask slot位图，原子量submitCount，对应的锁，对应的信号，对应的环形缓冲区和commitstamp

```c
gckMSSCHED_KernelQueueConstruct(MSSched, *TaskQueue)
	MSSCHED_TaskQueueConstruct(MSSched,TaskQueue,...)
    	//创建freetask slot位图
    	gckOS_CreateBitmap(...)
		gckOS_BitmapZero(...)
    	//创建原子量submitCount
    	gckOS_AtomConstruct(MSSched->os, &TaskQueue->submitCount)
    	gckOS_AtomSet(MSSched->os, TaskQueue->submitCount, 0)
    	//创建对应的锁
    	gckOS_CreateMutex(MSSched->os, &TaskQueue->mutexSchedule)
    	//创建task queue的调度信号 
    	gckOS_CreateSignal(MSSched->os, gcvFALSE, &TaskQueue->signalSchedule)
    	//创建环形缓冲区TaskQueue->ringBuf，一共L2SCHEDULE_TASK_QUEUE_SIZE个元素
    	_rbInit(&TaskQueue->ringBuf,...)
    	//每个mp设置commitstamp
    	for (i = 0; i < gcvCORE_COUNT; i++)
            TaskQueue->commitStamp[i].dataPrinted = (gctUINT64)~0ULL;
		   TaskQueue->commitStamp[i].data = 1;
```

##### gckMSSCHED_KernelQueueConstruct

​	创建kernelQueue，先分配kernelQueue，通过MSSCHED_TaskQueueConstruct配置kernelQueue的参数，之后配置L2SCHEDULE_TASK_QUEUE_SIZE个kernelTaskDesc，配置ops函数指针，最后返回kernelQueue

```c
gckMSSCHED_KernelQueueConstruct
	kernelQueue = &g_KernelQueues[platformId]
    //配置kernelQueue
	MSSCHED_TaskQueueConstruct(...)
    //配置L2SCHEDULE_TASK_QUEUE_SIZE个kernelTaskDesc，kernelQueue作为参数
    for (i = 0; i < L2SCHEDULE_TASK_QUEUE_SIZE; i++)
        kernelTaskDesc = &g_KernelTaskDescs[platformId][i]
        kernelTaskDesc->base.id = i;
        kernelTaskDesc->base.taskQueue = kernelQueue;//kernelQueue作为参数
        kernelQueue->base.tasks[i] = kernelTaskDesc 
    kernelQueue->base.ops = &g_KernelQueueOps;
	*TaskQueue = (L2TaskQueue_PTR)kernelQueue;
```

##### gckMSSCHED_DMAQueueConstruct

​	创建DMAQueue，先分配DMAQueue，通过MSSCHED_TaskQueueConstruct配置DMAQueue的参数，配置信号和锁，之后配置L2SCHEDULE_TASK_QUEUE_SIZE个dmaTaskDesc，配置ops函数指针，最后返回dmaQueue

```c
gckMSSCHED_DMAQueueConstruct
	dmaQueue = &g_DMAQueues[pltformId]
	//配置dmaQueue
	MSSCHED_TaskQueueConstruct(...)
    //创建transfer done signal
    gckOS_CreateSignal(MSSched->os, gcvFALSE, &dmaQueue->signalTransferDone)
    //创建task done signal
    gckOS_CreateSignal(MSSched->os, gcvFALSE, &dmaQueue->signalTaskDone)
    //创建mutex to proctect transfer
    gckOS_CreateMutex(MSSched->os, &dmaQueue->mutexTransferList)
    for (i = 0; i < L2SCHEDULE_TASK_QUEUE_SIZE; i++)
        dmaTaskDesc = &g_DMATaskDescs[pltformId][i]
        dmaTaskDesc->base.id = i;
        dmaTaskDesc->base.taskQueue = dmaQueue;//dmaQueue作为参数
        dmaQueue->base.tasks[i] = dmaTaskDesc;
		//分配fillDataBuffer(64kb)
	    gckOS_Allocate(MSSched->os, DMA_FILL_HOST_BUFFER_SIZE,&dmaTaskDesc->fillDataBuffer)
        //创建原子操作completedCount
        gckOS_AtomConstruct(MSSched->os, &dmaTaskDesc->completedCount)
     dmaQueue->base.ops = &g_DMAQueueOps;
	 dmaQueue->private = platform  //将platform作为dmaQueue参数
	 gcsLIST_Init(&dmaQueue->transferList); //将transferList添加到双向链表
	*TaskQueue = (L2TaskQueue_PTR)dmaQueue
```

​	gckMSSCHED_DMAQueueConstruct

```

```

##### _L2ScheduleCommonThread线程

​	调用taskqueue对应的selectTask和handleTask函数指针

```c
_L2ScheduleCommonThread
	while(1)
        if (l2Schedule->killThread)
            //获得原子量taskQueueSubmitCnt
            gckOS_AtomGet(msSched->os, taskQueue->submitCount, &taskQueueSubmitCnt)
            //taskQueueSubmitCnt为0
        	if (taskQueueSubmitCnt == 0)
                //通过commitStamp检查所有的task都完成
                for (i = 0; i < gcmCOUNTOF(taskQueue->commitStamp); i++)
                    currentMemValue = *taskQueue->commitStamp[i].logical;
				   if (currentMemValue < (taskQueue->commitStamp[i].data - 1)
                       canExit = gcvFALSE;//有task未完成
                       break;
                 if (canExit)//有task未完成不停止线程，否则停止线程
                       while()
                       	  gckOS_ThreadShouldStop()
                          gckOS_Delay()
                          break
           gckOS_WaitSignal(msSched->os, taskQueue->signalSchedule,...)//等待task queue的调度信号
           taskQueue->ops->selectTask(taskQueue, &taskDesc)//调用ops函数指针selectTask
           taskQueue->ops->handleTask(taskQueue, taskDesc);//调用ops函数指针handleTask
```

##### ops函数指针selectTask

​	L2ScheduleTaskSimple，将L2环形缓冲区中的信息传进TaskDesc，从UMD获得fence信息（当前的task和lasttask不同时，UMD中的hrbPacketFenceFill函数会根据情况进行fence配置），若有fence配置进行waitfence

```c
L2ScheduleTaskSimple
	rbRead(rbL2, TaskDesc, 1) //从L2环形缓冲区读取到TaskDesc
    _TaskCheckWaitFence(*TaskDesc, gcvTRUE) 
    	MemDesc = &TaskDesc->waitFenceMemDesc
    	if (!TaskDesc->halTask.waitFence.gpuLogical) //如果UMD未配置fence参数，结束函数
               return gcvSTATUS_TRUE
         MSSCHED_InitMemDescFence(...)//将UMD的fence参数配置到MemDesc(mode、expectedVal、gpuLogical、cpuLogical)
                MemDesc->mode        = Fence->mode
                MemDesc->expectedVal = Fence->expectedVal
                MemDesc->gpuLogical  = Fence->gpuLogical
                //根据fence的gpuLogical获得对应的cpuLogical
                MSSCHED_InitMemDescByGpuAddress(...)
          MSSCHED_WaitFence_Sync(...) //进行waitfence
                 do {
                        MSSCHED_WaitFence(MemDesc, Fence, &waitSuccess)
                            cpuLogical = MemDesc->cpuLogic
                            fenVal = *cpuLogical;//fence地址对应的值
                            expected = MemDesc->expectedVal;
                            switch(MemDesc->mode)
                                ...
                                //比较fence期望值和fence地址的值（sendfence???）
                                case gcvHAL_OFFLOAD_WAIT_MODE_GREATER_EQUAL:
                                   *WaitSucess = (expected <= fenVal)
                         if (!waitSuccess) {
                                gckOS_Delay(Os, 1); //条件不符进行waitfence
                          }
                    }while(!waitSuccess)
```

##### ops函数指针handleTask，_CommandCommit

​	从TaskDesc中取出cmdbuffer信息，加commandqueue锁后，与simple_routine线程以多线程的方式提交commandbuffer、event到GPU

```c
_CommandCommit(TaskQueue, TaskDesc)
	...
    for (i = 0; i < 2; i++) //每个die 1个cmdBufferInfo
        ...
        curCmd = &TaskDesc->halTask.cmdBufferInfo[i]; //从TaskDesc获得cmdbuffer信息
		for (j = 0; j < 32; j++) //每个die 2个core(core0 core1)
	    ...	//获得device，kernel
		command = kernel->command; //获得commandqueue,event对象
		eventObj = kernel->eventObj
         //将command的fence地址设为TaskQueue->commitStamp地址
         TaskQueue->commitStamp[coreId].logical = command->fence->logical
         //加CommitStamp锁，将command->commitStamp配置到TaskDesc->processNode->commitStamp
         gckOS_AcquireMutex(kernel->os, TaskQueue->mutexCommitStamp, gcvINFINITE)
         TaskDesc->processNode->commitStamp[TaskQueue->type][coreId] = command->commitStamp
         gckOS_ReleaseMutex(kernel->os, TaskQueue->mutexCommitStamp)
         _CommandCommit_WLFE(...)//加锁后提交cmdbuffer
            //从TaskDesc中取出cmdbuffer信息（包括逻辑地址、设备地址、size、reservedhead）
            ...
            gckKERNEL_GetCurrentMMU(...)//获取当前MMU     
            gckCOMMAND_EnterCommit(...)//加commandqueue锁，与simple_routine中的commandqueue同步 
            CommitWaitLinkOnceWrapper(...)//在commandqueue中插入wait/link来提交cmd
            gckCOMMAND_ExitCommit(Command, gcvFALSE)//解commandqueue锁
            Command->commitStamp++//commitStamp加1
         //将command->commitStamp配置到TaskQueue->commitStamp
         TaskQueue->commitStamp[coreId].data = command->commitStamp
         ......	//配置event
         gckEVENT_Submit(eventObj, &eventAttr) //加锁后提交event
         _CommandDone(...) //解除fence对应的cpu映射
```

##### ops函数指针handleTask，_DMACommit

​	将DmaTask->listNode添加到TransferList链表，根据TaskDesc中的type类型进行_CopyBuffer 或 _FillBuffer

```c
_DMACommit(TaskQueue, TaskDesc)
    _AddTaskToTransferList(...)//将DmaTask->listNode添加到TransferList链表
    	gckOS_AcquireMutex(...,mutexTransferList,...)
    	gcsLIST_Add(...)
    	gckOS_ReleaseMutex(...,mutexTransferList,...)
    //HRB_PACKET_TYPE_DMA_COPY类型，_CopyBuffer
    if (TaskDesc->halTask.type == HRB_PACKET_TYPE_DMA_COPY){
        gcmkONERROR(_CopyBuffer(msSched, kernel, dmaTask, &isDone));
    }
	//HRB_PACKET_TYPE_DMA_FILL类型，_FillBuffer
	if (TaskDesc->halTask.type == HRB_PACKET_TYPE_DMA_FILL){
        gcmkONERROR(_FillBuffer(msSched, kernel, dmaTask, &isDone));
    }
	if (isDone) 
        _DMADone(dmaTask);
```

##### _CopyBuffer函数

​	进行数据的维度转换，将3D转为2D，2D转为1D，计算出数据地址然后通过DMA提交数据

```c
_CopyBuffer
	_checkConvertLogic(...)//判断src是否是host端
	_checkConvertLogic(...)//判断dst是否是host端
    _dimensionMerge(srcOrigin，srcStride，dstOrigin，dstStride，region)//维度转换，3D-2D，2D-1D
    	//3D-2D
    	//stride->X 每个数据单元4字节,stride->Y 一行10个数据单元,stride->Z 一层的数据字节数4*10*region->X 
    	//origin->X X方向偏移，origin->Y Y方向偏移，origin->Z Z方向偏移
    	//region->X 单元行数,region->Y 单元列数,region->Z 单元层数 
    	dstStride->y = dstStride->z; //整合dststride\srcstride
	    dstStride->z *= region->z;
    	dstOrigin->x += region->x * dstOrigin->y; //整合dstorigin\srcorigin
        dstOrigin->y = dstOrigin->z;
        dstOrigin->z = 0;
	    region->x *= region->y; //整合region
        region->y = region->z;
        region->z = 1;
    	//2D-1D
		dstStride->y = dstStride->z;//整合dststride\srcstride
		dstOrigin->x += region->x * dstOrigin->y;//整合dstorigin\srcorigin
         dstOrigin->y = 0;
		region->x *= region->y;//整合region
         region->y = 1;
	//如果srcStride\dstStride y、z的步幅为0，将y、z的步幅设置为size大小
    _calculateStride(srcStride, region);
	_calculateStride(dstStride, region);
	//根据srchost或dsthost判断src或dst是CPU或GPU地址
	srcBaseLogical = isSrcHost ? srcBaseCPULogical : copyInfo->srcBaseGPULogical;
	dstBaseLogical = isDstHost ? dstBaseCPULogical : copyInfo->dstBaseGPULog
     //判断DMA传输类型
     dmaDir = gcdOT_OFFLOAD_DMA_DIR_H2H
     ...
     //通过X,Y,Z计算数据地址
     for (z = 0; z < region->z; z++)
         sliceAddrSrc = ...
		sliceAddrDst = ...
         for (y = 0; y < region->y; y++)
             ...
             _SubmitToDMA(...,addrDst, addrSrc, copySize, dmaDir,...) //通过DMA提交数据
             	...	//将src、dst、bytes、dir等配置到dmaXferDesc结构体
             	platform->ops->msschedDmaXferSubmit()//调用platform->ops的函数指针 
```

##### ops的函数指针msschedDmaXferSubmit

​	调用_msschedDmaXferSubmit函数将host地址映射为DMA地址，配置transfer_info结构体，通过hal_zixia_dma_transfer函数开启DMA传输

```c
_msschedDmaXferSubmit
	switch(Xfer->dir) 
    case gcdOT_OFFLOAD_DMA_DIR_H2D:
        _MapDeviceBuf(...，&devPhysAddr) //获得dst设备物理地址
        sgTbl=hal_zixia_dma_map(...) 	//映射src为DMA地址
        transfer_info.src = (void*)sgTbl; //配置transfer_info
        transfer_info.dst = (void*)devPhysAddr;
        transfer_info.src_sgt = true; 
        transfer_info.dst_sgt = false;
        transfer_info.dir = HAL_ZIXIA_DMA_DIR_H2D;
    case gcdOT_OFFLOAD_DMA_DIR_D2H：
    	_MapDeviceBuf(...，&devPhysAddr) //获得src设备物理地址
        sgTbl=hal_zixia_dma_map(...) 	//映射dst为DMA地址
        ... 	//配置transfer_info
    case gcdOT_OFFLOAD_DMA_DIR_D2D：
    	_MapDeviceBuf(...，&devPhysAddr) //获得src设备物理地址
         _MapDeviceBuf(...，&devPhysAddr) //获得dst设备物理地址
        ...		//配置transfer_info
    case gcdOT_OFFLOAD_DMA_DIR_H2H：
        _DmaH2HRoutine(...)
    ...
    hal_zixia_dma_transfer(pdev, &transfer_info);
```

##### hal_zixia_dma_map函数

​	根据host内存地址构建SG表，将SG表映射为DMA地址

```c
hal_zixia_dma_map(...)
    //根据host内存地址来判断host的地址类型
	if (uaddr >= PAGE_OFFSET) //内核地址
        if (is_vmalloc_addr(addr)) //vmalloc地址
    		dmap->type = AT_VMALLOC;
		else
            dmap->type = AT_KMALLOC; //kmalloc地址
     else					//user地址
        dmap->type = AT_USER;	//malloc地址
	//计算地址所跨越的物理页数
	npages = (ALIGN_DOWN(uaddr + size - 1, PAGE_SIZE) -  ALIGN_DOWN(uaddr, PAGE_SIZE)) / PAGE_SIZE + 1;
	switch (dmap->type)
         case AT_KMALLOC:
			sg_alloc_table(&dmap->sgt, 1, GFP_KERNEL) //分配sg_table结构体
             page = virt_to_page(addr) //获取内核虚拟地址对应的物理页
             offset = offset_in_page(addr) //得到物理页的偏移量
             sg_set_page(dmap->sgt.sgl, page, size, offset); //将物理页与SG表进行关联
		case AT_VMALLOC:

		case AT_USER:
			dmap->npages = npages;
			kmalloc_array(dmap->npages, sizeof(struct page *), GFP_KERNEL);//分配数组内存
			find_get_pid(pid) //获取进程的pid结构
             pid_task(pid_st, PIDTYPE_PID) //获取进程的task结构
             get_task_mm(tsk)	//获取进程的内存描述符
             down_read(&mm->mmap_lock); //加锁
			pin_user_pages_remote(...) //将进程用户态对应的物理页固定在内存，防止被内核换出
             up_read(&mm->mmap_lock) //解锁
			...
             sg_alloc_table_from_pages(...) //基于固定的物理页构建SG表（SG表将物理上离散的物理页描述为连续的传输区间，便于DMA访问）
       dma_map_sg(dev, dmap->sgt.sgl，...) //将SG表映射到DMA地址空间
```

​	hal_zixia_dma_transfer函数，multi_chan == 1时启动单通道传输，multi_chan大于1时启动多通道传输

```c
hal_zixia_dma_transfer(*pdev, *transfer_info)
	...
	multi_chan = transfer_info->multi_chan
	if(multi_chan == 1) 
    	return zixia_dma_single_chan_transfer(pdev, transfer_info); //单通道传输
	... //多通道传输
```

##### DMA单通道传输

​	zixia_dma_single_chan_transfer函数（单通道传输），获取DMA通道，初始化等待队列、工作队列，将DMA通道与外设关联，准备基于SG缓冲区的传输描述符，设置传输完成后的回调函数，提交传输描述符到DMA引擎，启动DMA传输，让进程在等待队列上睡眠来等待DMA传输完成，DMA传输完成后会触发中断，调用回调函数（中断下半部），在回调函数中将工作队列添加到内核线程，执行与工作队列绑定的处理函数，在处理函数中唤醒睡眠中的等待队列，释放DMA通道，解除DMA映射

```c
zixia_dma_single_chan_transfer(*pdev, *transfer_info)
    ...		//根据host内存地址构建SG表，将SG表映射为DMA地址
    chan = zixia_dma_chan_alloc(...) //获取DMA通道数，找到传输size最小的通道作为DMA通道
            chip = pci_get_drvdata(pdev);
            pciedma = (struct zixia_pciedma *)chip->pciedma; //从pci结构体取出pciedma信息
            if(dir == HAL_ZIXIA_DMA_DIR_H2D) //H2D获取读通道数，读通道锁，读通道
                ch_cnt = pciedma->rd_ch_cnt;
                alloc_lock = &pciedma->rd_chan_alloc_lock;
                pchan = &pciedma->chan[pciedma->wr_ch_cnt];
            if(dir == HAL_ZIXIA_DMA_DIR_D2H) //D2H获取写通道数，写通道锁，写通道
                ch_cnt = pciedma->wr_ch_cnt;
                alloc_lock = &pciedma->wr_chan_alloc_lock;
                pchan = &pciedma->chan[0];
            spin_lock_irqsave(alloc_lock, lock_flags);//开自旋锁
            for (i = 0; i < ch_cnt; i++) //找到传输size最小的通道
                if(min_trans_size > pchan[i].transmisions.total_size) 
                    min_trans_size = pchan[i].transmisions.total_size;
                    ch = i
             pchan[ch].transmisions.total_descs += 1; //更新该通道的描述符
             pchan[ch].transmisions.total_size += size;//更新该通道的size
             chan = pchan[ch].transmisions.chan;          
             spin_unlock_irqrestore(alloc_lock, lock_flags);//解自旋锁
	//将src\dst地址、DMA通道、transfer_info等信息配置进dt的transfer_info结构体
	 dt->dev = &pdev->dev;
     dt->dmap_sgt = dmap_sgt;
     dt->host_sgt = host_sgt;
     dt->chan = chan;
     memcpy((void *)&dt->transfer_info, (void *)transfer_info, sizeof(*transfer_info))
     //初始化等待队列头
     dt->done = &done;
     init_waitqueue_head(&wait);
     dt->wait = &wait;
	 //初始化工作队列，dma_transfer_finish_work作为处理函数
	 INIT_WORK(&dt->work, dma_transfer_finish_work);
	 if(transfer_info->dir == HAL_ZIXIA_DMA_DIR_H2D)
    		dma_sync_sg_for_device(...);	//同步 CPU 缓存数据到物理内存
	dmaengine_slave_config(chan, &config) //将DMA通道与外设进行关联
     dmaengine_prep_slave_sg(...) //为Slave 模式 DMA 通道准备基于SG缓冲区的传输描述符，DMA_PREP_INTERRUPT | DMA_CTRL_ACK标志，传输完成后触发中断，描述符自动复用
     tx->callback_result = dma_transfer_callback; //设置传输完成的回调函数
	tx->callback_param = dt;	//回调函数参数
	dt->time_start = ktime_get()
     dmaengine_submit(tx) //提交传输描述符到DMA引擎
     dma_async_issue_pending(chan) //启动DMA传输
     wait_event_interruptible_timeout(wait, done, msecs_to_jiffies(timeout))//让进程在等待队列上睡眠，dnoe为1时或到达超时时间唤醒(等待DMA传输完成)
```

##### dma_transfer_callback回调函数

​	dma_transfer_callback回调函数（属于中断下半部），传输完成时调用回调函数，计算传输时间，将工作队列添加到内核线程，调用dma_transfer_finish_work处理函数

```c
dma_transfer_callback(*data, *result)
    struct dma_transfer *dt = data
    dt->time_val = ktime_to_us(ktime_sub(ktime_get(), dt->time_start)) //计算传输时间
    schedule_work(&dt->work) //提交工作队列，调用处理函数
```

##### dma_transfer_finish_work处理函数

​	与工作队列绑定，同步DMA数据到CPU，唤醒等待队列，释放DMA通道，解除DMA映射

```c
dma_transfer_finish_work
	if(dt->transfer_info.dir == HAL_ZIXIA_DMA_DIR_D2H) 
    	dma_sync_sg_for_cpu(...); //解决 CPU 缓存与 DMA 物理内存的一致性，同步DMA数据到CPU
	if(dt->done && dt->wait) 
        *dt->done = true;
        wake_up(dt->wait);//将done设置为1，唤醒等待队列
	zixia_dma_chan_free(...) //释放DMA通道
     ...	//解除DMA映射
```

##### dma_async_issue_pending函数

​	调用device_issue_pending函数指针->zixia_pciedma_device_issue_pending函数，调用zixia_pciedma_start_transfer函数开始DMA传输，从描述符的数据块链表取出第一个数据块，调用zixia_pciedma_core_start函数开始传输 

```c
zixia_pciedma_device_issue_pending(struct dma_chan *dchan)
    spin_lock_irqsave(&chan->vc.lock, flags);
	//检查当前 DMA 通道的待执行描述符队列是否有未处理的任务，通道request为NONE，通道状态为IDLE
	if (vchan_issue_pending(&chan->vc) && chan->request == ZIXIA_PCIEDMA_REQ_NONE &&
        chan->status == ZIXIA_PCIEDMA_ST_IDLE) {
        chan->status = ZIXIA_PCIEDMA_ST_BUSY;
        zixia_pciedma_start_transfer(chan); //开始传输
    }
	spin_unlock_irqrestore(&chan->vc.lock, flags);

zixia_pciedma_start_transfer(struct zixia_pciedma_chan *chan)
    vd = vchan_next_desc(&chan->vc); //获取下一个描述符
	desc = vd2zixia_pciedma_desc(vd);//描述符转换
	child = list_first_entry_or_null(&desc->chunk->list,
                                     struct zixia_pciedma_chunk, list);//从链表取第一个数据块
	zixia_pciedma_core_start(pciedma, child, true);//数据块信息传进LLI，开始DMA传输
	zixia_pciedma_free_burst(child); //遍历数据块的子数据块，从链表删除子数据块，释放子数据块内存
    list_del(&child->list); //从链表删除数据块
    kfree(child); //释放数据块内存
```

##### zixia_pciedma_core_start函数

​	LLI是存储DMA传输参数的一块内存，DMA传输可以拆分成多个 LLI，用链表串联，硬件按顺序读取 LLI 自动完成批量传输

​	调用hdma_core_start函数指针，通过hdma_core_write_chunk_lli_recycle或hdma_core_write_chunk将数据块中信息写入LLI内存，配置寄存器开启engine、中断等，对LLI内存进行伪读操作来保证PCIE的顺序执行，配置doorbell寄存器，启动DMA传输（PCIE可能不按顺序传输，先写doorbell，后写LLI内存，导致DMA传输旧数据）

```c
hdma_core_start()
    if(pciedma->lli_recycle) {
        hdma_core_write_chunk_lli_recycle(chunk, first);
    } else {
        hdma_core_write_chunk(chunk);
    }
	//配置PCIE寄存器开启watermask中断、engine，配置Prefetch和Channel control等
	SET_CH_32(pciedma, chan->dir, chan->id, ch_en, BIT(0));
	...、
     //对LLI内存进行一次读操作，PCIE会在写LLI都完成后，才返回读LLI的结果，保证PCIE顺序执行
     hdma_sync_ll_data(chunk);
	//配置doorbell寄存器，通知硬件LLI准备好，启动DMA传输
	SET_CH_32(pciedma, chan->dir, chan->id, doorbell, HDMA_DOORBELL_START);
```

##### hdma_core_write_chunk_lli_recycle

​	lli_recycle开启后DMA的LLI可以自动回收复用，传输完成后的LLI被回收到空闲链表，避免重复申请

​	watermask_index是空闲链表的阈值，小于阈值时自动申请LLI到空闲链表

​	prefetch_len预取长度，硬件一次取prefetch_len个LLI到缓存

​	从数据块链表遍历子数据块child，当占用的LLI数量达到watermask，配置control的HDMA_LWIE 和HDMA_RWIE（通过中断补充LLI到空闲链表），将子数据块的传输信息写入第i个LLI，占用的LLI数量递增，配置prefetch_len，配置control的HDMA_LLP（硬件自动遍历LLI），将第i个 LLI 的链表指针指向 LLI 区域的起始物理地址

```c
hdma_core_write_chunk_lli_recycle(struct zixia_pciedma_chunk *chunk)	
    upper = chunk->bursts_index ? false : true;
	//计算watermask_index（ll_max_align_prefetch最大预取描述符的1/2）
    water_mask_index =chunk->ll_max_align_prefetch / 2 ; 
	//从数据块遍历子数据块child
    list_for_each_entry(child, &chunk->burst->list, list)
        //占用的LLI数量达到水位线或接近最大预取长度
        if(i == water_mask_index || i == (chunk->ll_max_align_prefetch - 2))
            control |= HDMA_LWIE | HDMA_RWIE; //开启中断，补充LLI到空闲链表
		else
            control &= ~(HDMA_LWIE | HDMA_RWIE);//不开启中断
		//将子数据块的control、size、src地址、dst地址写入第i个LLI，便于硬件传输
		hdma_write_ll_data(chunk, i, control, child->sz,child->sar, child->dar);
		//当达到水位线后，退出函数
		if(!first && upper && (i == water_mask_index)) {
    		chunk->bursts_index = ++i;
    		return;
		}
		//index递增
		chunk->bursts_index = ++i;
	//将prefetch_len写入硬件
	control = ((chunk->prefetch_len > 0) ? (chunk->prefetch_len - 1) : 0) << 16;
	control |= HDMA_LLP; 	//硬件自动遍历LLI
	if(chunk->bursts_index == (chunk->ll_max_align_prefetch - 1))
    	control |= HDMA_TCB; 	//通知硬件是最后一个LLI
	//将第i个 LLI 的链表指针指向 LLI 区域的起始物理地址
	hdma_write_ll_link(chunk, i, control, chunk->ll_region.dev_paddr);
```

##### hdma_core_write_chunk

​	未开启lli_recycle，从数据块链表遍历子数据块child，将子数据块的传输信息写入第i个LLI，配置prefetch_len，配置control的HDMA_LLP（硬件自动遍历LLI），将第i个 LLI 的链表指针指向 LLI 区域的起始物理地址

```c
hdma_core_write_chunk(struct zixia_pciedma_chunk *chunk)
    //从数据块遍历子数据块child
    list_for_each_entry(child, &chunk->burst->list, list)
    	//将子数据块的control、size、src地址、dst地址写入第i个LLI，便于硬件传输
		hdma_write_ll_data(chunk, i++, control, child->sz,child->sar, child->dar);
	//将prefetch_len写入硬件
	control = ((chunk->prefetch_len > 0) ? (chunk->prefetch_len - 1) : 0) << 16;
	control |= HDMA_LLP;	//硬件自动遍历LLI
	//将第i个 LLI 的链表指针指向 LLI 区域的起始物理地址
	hdma_write_ll_link(chunk, i, control, chunk->ll_region.dev_paddr);
```

##### DMA多通道传输

​	多通道传输，将dmct->complete通过原子操作设为0，初始化等待队列，将host的SG表分为multi_chan个子表，传输multi_chan次，配置multi_chan个传输参数（每次传输size/multi_chan大小，配置每次传输的子表和回调函数），通过调用multi_chan次zixia_dma_single_chan_transfer函数来进行multi_chan次单通道传输，加入等待队列进行睡眠等待DMA传输完成

```c
hal_zixia_dma_transfer(*pdev, *transfer_info)
	if(multi_chan == 1) //单通道传输
		zixia_dma_single_chan_transfer(pdev, transfer_info);
	else			   //多通道传输
        dmct->dev = &pdev->dev;
	    dmct->multi_chan = multi_chan;
	    //将transfer_info memcpy到dmct结构体
        memcpy((void *)&dmct->transfer_info, (void *)transfer_info,...); 
	    atomic_set(&dmct->complete, 0);	//dmct->complete设为0
		dmct->done = &done;
		init_waitqueue_head(&wait); //初始化等待队列
		dmct->wait = &wait;
		if(transfer_info->src_sgt) //将host的SG表分为子表
            	//将SG表分为multi_chan个子SG表，分配multi_chan个sg_table结构体
            	src_sgt_split = zixia_sgt_split_by_multi_chan(...)	
            	dmct->src_sgt_split = src_sgt_split;
		if(transfer_info->dst_sgt)
            	//将SG表分为multi_chan个子SG表，分配multi_chan个sg_table结构体
            	dst_sgt_split = zixia_sgt_split_by_multi_chan(...)
            	dmct->dst_sgt_split = dst_sgt_split;
		for(i = 0; i < multi_chan; i++) //分为multi_chan个进行传输
            	mp_transfer_info[i].pid = transfer_info->pid
            	tmp_transfer_info[i].size = transfer_info->size / multi_chan
            	if(transfer_info->src_sgt)//分配子SG表
                    tmp_transfer_info[i].src = &src_sgt_split->tmp_sgts[i];
				   tmp_transfer_info[i].src_sgt = true;
            	if(transfer_info->dst_sgt)//分配子SG表
                     tmp_transfer_info[i].dst = &dst_sgt_split->tmp_sgts[i];
				    tmp_transfer_info[i].dst_sgt = true;
				...
				tmp_transfer_info[i].fn = dma_multi_chan_callback;//回调函数
				tmp_transfer_info[i].fn_args = dmct;
				...
          for(i = 0; i < multi_chan; i++)//进行multi_chan个单通道传输
              	zixia_dma_single_chan_transfer(pdev, &tmp_transfer_info[i]);
		  if (transfer_info->block) 
            	//加入等待队列进行睡眠等待DMA传输完成
    			wait_event_interruptible_timeout(wait, done, msecs_to_jiffies(timeout));
```

​	多通道传输回调函数dma_multi_chan_callback，每个通道传输完成后调用回调函数，dmct->complete参数加1，当dmct->complete与multi_chan相同时，计算传输时间，唤醒等待队列

```c
dma_multi_chan_callback
	atomic_inc(&dmct->complete) //将dmct->complete参数加1
    if(atomic_read(&dmct->complete) == dmct->multi_chan) //读取dmct->complete参数，当dmct->complete参数与multi_chan相等时
        dmct->time_val = ktime_to_us(...)
        if(dmct->done && dmct->wait) {
    	*dmct->done = true;
    	wake_up(dmct->wait); //唤醒等待队列
         ...	//释放SG结构体
```























