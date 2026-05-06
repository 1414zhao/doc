## culaunchkernel-drvlaunchkernel

​	将用户层的各种参数（griddim\\blockdim\\stream\\核函数\\核函数参数等）传给drvlaunchkernelex_commcon函数

```
drvlaunchkernelex_commcon(&config,f,kernelparams,extra,...)
```

​	获得当前context，获得当前stream（传入的stream），获得stream对应的halqueue，通过下面函数判断griddim\\blockdim等参数是否有效，是否在有效范围内

```
launchkernelvalidate(config,f,kernelparams,extra,...)
```

​	从f->kernelinfo中获得核函数的参数数量count，分配count大小的**argdata**

```
argcount=f->kernelinfo->argcount;
argdata=os_calloc(sizeof(kernelargdata_t)*argcount);
```

​	通过下面函数为argdata赋值，从f->kernelinfo中获得各参数的信息arg（包括name\\type\\size等），根据arg的type（参数分配内存的方式）判断（获得各参数的地址，根据地址获得映射结点，根据memnode的cpu逻辑地址\&参数地址是否在参数地址和cpu逻辑地址+size范围内判断参数地址是否为gpu逻辑地址，不是的话将参数地址转换为gpu逻辑地址，设置argdata值（包括各参数size\\映射节点\\offset（参数地址-memnode的gpu逻辑地址）））

```
launchkernelprocesskernelargs 
	for(int i=0;i<argcount;i++)
		data=argdata+i;
		arg=f->kernelinfo->args+i;
		data->arg=arg;
		...
		data->memallocinfo.size=mapmemnode->memnode->size
		...
		data->memallocinfo.node=mapmemnode->memnode
		data->memallocinfo.offset=
```

​	通过下面函数配置command.kernelinfo参数，包括griddim\\blockdim\\argdata\\f->kernel\\f->kernelinfo\\各方向的线程数global等，根据global设置**workdim**参数（z方向线程数>1 workdim为3，y方向线程数>1 workdim为2，否则workdim为1）

​	通过里面的prepareexecinfo函数配置f->kernel的参数（包括总线程数\\线程块中线程数\\线程块总数\各方向的线程数），command.kernelinfo的percluster线程数min(8192，总线程数），percluster工作组数min(128，总线程数/线程块中线程数)

```
launchkernelprepareexecinfo(&command.kernelinfo,f,config,argdata,cmkernel)
	...
	prepareexecinfo(info)
```

​	进入launchcommand函数

```
launchcommand(halqueue,&command)
```

​	先createcommandbuffer为cmdbuffer分配1024\*1024内存，根据cmd（COMMAND\_KERNEL）进行execute

```
virUniform = clfBeginVIRUniform(cmdBuffer, execInfo->inst, nullptr);
...
clfEndVIRUniform(virUniform);
```

​	通过clfinvokevirkernel函数分配线程（分配双die线程数和cmdbuffer），配置各种寄存器

```
clfinvokevirkernel(cmdbuffer,execinfo->inst,...)
```

​	根据workdim设置info的各维度（X、Y、Z）参数（包括线程块中各方向的线程数，各方向的线程块数，各方向的线程总数）

```
switch (WorkDim)
{
   case 3:
        Info->globalSizeZ = (uint32_t)GlobalWorkSize[2];
        Info->globalOffsetZ = (uint32)GlobalWorkOffset[2];
        Info->workGroupSizeZ = LocalWorkSize[2] ? (uint32_t)LocalWorkSize[2] : 1;
        Info->workGroupCountZ = Info->globalSizeZ / Info->workGroupSizeZ;
        workGroupSize *= Info->workGroupSizeZ;
   case 2:
   		Info->globalSizeY = 
   		Info->globalOffsetY = 
   		...
   case 1:
   		Info->globalSizeX = 
   		Info->globalOffsetX = 
   		...
```

​	通过getCombineMode函数看是否是combine模式，通过环境变量VIV_MDIE_COMBINATION = 2设置

```
mpCombine_t mpCombine = getCombineMode(processor, cmdBuffer->coCore);
```

​      如果为combine模式，对双die进行线程分配，主要通过spiltTaskForWindowMode函数进行分配

```c
if (IS_MD_COMBINE_WIN(mpCombine) && !(Info->memoryAccessFlag & gceMA_FLAG_ATOMIC))
{  
	...
	usedDieCount = spiltTaskForWindowMode(...)
     //对双die中线程块中各方向的线程数，各方向的线程块数，各方向的线程总数进行设置
	for (i = 0; i < usedDieCount; i++)
     {
     	eachGPUInfo[i]                 = *Info;
        eachGPUInfo[i].workGroupCountX = workGroupCountPerGpu[i * 3 + 0];
        eachGPUInfo[i].globalSizeX     = globalSizePerGpu[i * 3 + 0];
        eachGPUInfo[i].globalOffsetX   = globalOffsetPerGpu[i * 3 + 0];
        ...Y
        ...
        ...Z
        ...
     }
 }
```

   spiltTaskForWindowMode函数

   当为combine模式时，对于线程块数最大的维度，线程块数进行均分后平均分到两个die中，对于其他维度（线程块数不均分）两个die中的线程块数相同

```c
gctUINT spiltTaskForWindowMode(gctUINT gpuCount ...)
{
	for (i = 0; i < workDim; i++)
        workGroupCount[i] = globalSize[i] / workGroupSize[i];
    //找到线程块数最大的维度maxDimInd
    count = workGroupCount[0];
    for (i = 1; i < workDim; i++)
        if (count < workGroupCount[i])
            count = workGroupCount[i];
            maxDimInd = i;
    //对于每个die，给每个die的不同维度设置相同的线程块数和线程数
    for (i = 0; i < gpuCount; i++)
        gctUINT j = 0;
        for (j = 0; j < workDim; j++)
            workGroupCountPerGpu[i* MAX_DIM + j] = workGroupCount[j];
            globalSizePerGpu[i * MAX_DIM + j] = globalSize[j];
    //将线程块数最大的维度对应的线程块数根据die数量分为2份
	groupCount = workGroupCount[maxDimIndex] / gpuCount;
	restGroupCount = workGroupCount[maxDimIndex] % gpuCount;
    //分为2份后更新这个维度的线程块数和块中线程数
     for (i = 0; i < gpuCount; i++)
        workGroupCountPerGpu[i * MAX_DIM + maxDimIndex] = groupCount;
        globalSizePerGpu[i * MAX_DIM + maxDimIndex] = groupCount * workGroupSize[maxDimIndex];
    //对于有余数的情况，线程块数向上进1
     for (i = 0; i < restGroupCount; i++)
            workGroupCountPerGpu[i * MAX_DIM + maxDimIndex]++;
            globalSizePerGpu[i * MAX_DIM + maxDimIndex] = workGroupCountPerGpu[i * MAX_DIM + maxDimIndex] * workGroupSize[maxDimIndex];
    ...

}
```

​      将hwstates->pstatebuffer放入cmdbuffer中，cmdbuffer地址后移hwstates->statebuffersize位

```
os_memcpy(*states,hwstates->pstatebuffer,hwstates->statebuffersize);
*state += hwstates->statebuffersize/sizeof(u32)
```

​	获得f->kernel的地址（gpu逻辑地址）为physical，设置寄存器值放入cmdbuffer中，配置指令地址

```
physical=getdeviceaddr(kernelinst->instmem,...)
_clcmdloadsinglehwstate(states,gcregpsinstructionregaddrs,physical)
```

​	设置寄存器值放入cmdbuffer中，配置local内存的size和地址，配置private内存的size和地址，配置const内存的size和地址

​	配置和shader相关的寄存器（1个warp有64个thread（16个shader core），1个shader core中有4个thread（相当于计算单元））

​	threadalloc  一个warp64个线程，threadalloc=线程块中的总线程数/64（向上取整），最多32个warp组，groupnumberpercluster=64/线程块中的总线程数-1（64/线程块中的总线程数<1时groupnumberpercluster=0）

​	commandbufferswitchtosingledie函数，将die0的cmdbuffer复制到die1。

​        对于双die，通过die0的cmdbuffer和lastcmdbuffer差计算die0中cmdbuffer的cmd长度，将die0对应的cmd复制到die1的cmdbuffer（复制长度为count），将lastcmdbuffer仍然设置为die0的cmdbuffer

```
commandbufferswitchtosingledie(cmdbuffer)
```

​	对于2个die的cmdbuffer，分别设置寄存器值放入它们的cmdbuffer中，配置和thread walker相关的寄存器，配置shader thread walker相关的寄存器，将各方向的线程块数，各方向的线程总数等配置进寄存器

​	将cmdbuffer->currentDie设置为die0，切换为die0的cmdbuffer

```
commandBufferSwitchToAllDie(cmdBuffer)
```

​	配置信号（FE请求信号，PIXEL\_engine发出信号），暂停cmd解析直到信号发出

​	配置寄存器使各种指令cache无效 

通过addcmdtoqueue函数  将cmdbuffer作为一个结点，插到halqueue的cmdqueue中



**hal线程进行round\_robin轮转调度**

​	一直while(1)轮询，根据idx从queuelist链表中取出指定的结点，结点的data为halqueue，取出的cmd为halqueue的commandqueue中的cmdbufffer

```
iterator=os_list_iterator_by_idx(qlist,idx)
q=iterator->data
cmd=os_queue_head(q->commandqueue)
```

​	进入commitcmdbuffer进行命令提交

```
commitcmdbuffer(cmd,commitinfo)
```

​	对于双die，分别得到cmdbuffer中cmd的长度count，找到双die中的最大cmd长度maxCmdCount，将count+readidxcmdcount作为每个die的cmdcount，将maxCmdCount+readidxcmdcount+waitFenceCmdCount作为cmdcount，每个cmd占4字节，cmdcount\*4与64对齐后得到cmdsize，除4获得最后的cmdcount，cmdcount\*die数量为总共的cmd长度（最后的cmdcount取决于两个die中最大的cmdcount）

```c
for (int d = 0; d < dieCount; d++)
    count=getcommandcount(cmd,d)
    maxCmdCount = count > maxCmdCount ? count : maxCmdCount; //找到die0、die1中最大的cmd长度
    cmdcountPerDie[d]=count+readidxcmdcount //每个die的cmd长度
cmdCount = maxCmdCount + GET_READ_IDX_CMD_COUNT(cmd) + waitFenceCmdCount;
cmdsize=cmdcount*4+reservedtail+reservedhead
cmdsize=ALIGN_NP2(cmdsize,64)
cmdcount=cmdsize/4
totalcount=cmdcount*diecount
```

​	将cmdpool中的commitcmdid+总共的cmd长度作为cmdid，更新commitcmdid为cmdid

​	将cmdid和cmdpool->readcmdidxaddr分别作为data和地址进行**sendfence**

​	将cmdpool的CPU逻辑地址+offset（256）得到一个逻辑地址作为dst，将cmdbuffer中的数据赋给dst，更新offset为offset+cmdsize（一共循环diecount*corecount（包含4cluster的mp数量）次，将每个die中cmdbuffer的数据赋给每个die中4cluster的mp ，所有mp的cmdbuffer数据都分配在cmdpool中，通过offset分隔）

```c
int subIdx  = 0;
offset += (cmdIdx - totalCount) * 4;//cmd size
for (int die = 0; die < dieCount; die++)
	for (int i = 0; i < coreCount; i++)
         subCommit[subIdx].commandBuffer.startOffset = (uint32_t)offset;
		logical = getCPUAddr(cmdPool->commandBufferPool, processor) + offset;
		...
		dst = (uint32_t*)(logical + reservedHead);
		os_memcpy(dst + waitFenceCmdCount, cmd->cmd.die[die].cmdBase, cmdCountPerDie[die] * 4);
		...
		offset+= cmdSize;
		subIdx = i;
		...
```

​	设置subcommit参数，包括cmdsize\\cmdpool的gpu逻辑地址\\cmdpool的cpu逻辑地址\\cmdpool的内存结点等，将subCommit作为interface参数（循环corecount次，包含4cluster的mp都会分配一个subCommit）

```c
//corecount是双die中包含4cluster的mp数量
int loopCount = bSingleCoreMode(cmd) && !coCore ? 1 : coreCount;
//设置corecount个subcommit参数
for (int i = 0; i < loopCount; i++)
    subCommit[i].context = 
    ...
    subCommit[i].commandBuffer.logical = 
    ...
```

​	循环halfencecount，增加每个fence的count，将fence插入到profilingqueue中

# 	 Kmd

## _commit 函数

根据interface的gcvHAL_COMMIT进入commit函数，提交subcommit，提交mpcount次（一共有mpcount个subcommit）

```c
_Commit(Device,hwtype,engine,...,commit)
    do{
        gckos_getdevice(Device->os,subcommit->devid,&device) //获得device
        ...
        kernel=device->coreinfoarry[subcommit->coreid].kernel //获得kernel
        ...
        command=kernel->asynccommand  //获得kernel commandqueue和eventogj
        eventobj=kernel->eventogj	
        ...
        gckCOMMAND_Commit(...) //提交cmdbuffer
        gckEVENT_Commit(...) //提交event
        next = submitcommit->next //下一个submitcommit
        userPtr = gcmUINT64_TO_PTR(next);
    }while(userPtr)
```

### gckcommand_commit函数

```
gckcommand_commit（command,subcommit,...,&commit->commitstamp,&commit->contextswitched)
```

​	获得context=command->kernel->db->pointerdatabase->table\[subcommit->context-1] 

```
context=gckkernel_querypointerfromname(command->kernel,subcommit->context)
```

​	获得mmu=command->kernel->mmu 

```
gckkernel_getcurrentmmu(command->kernel,gcvTRUE,0,&mmu)
```

​	通过gckcommand_entercommit函数获得相关锁和信号

```
增加command->atomcommit（增加commit atom来让power管理知道一个commit正在进行中）
获得硬件状态command->kernel->hardware  打开gpu（通知系统gpu有一个commit） 
获得power管理信号command->powersemaphore
获得command锁
```

#### _commitwaitlinkonce函数

```
_commandwaitlinkonce(command,context,commandbuffer,...)
```

​	通过下面函数获取link命令的size=16(linkbytes)

```
gckwlfe_link(hardware,gcvnull,0,0,&linkbytes,...)
```

​	获得commandbuffer的cpu\gpu逻辑地址（存之前commandbuffer中的cmd数据），获得commandbuffer中的cmd size

```
（之前的dst地址) commandbufferlogical=commandbuffer->logical+commandbuffer->startoffset (256)
commandbufferaddress=commandbuffer->address+commandbuffer->startoffset (256)
commandbuffersize=commandbuffer->size
```

​	gckcommand_checkflushmmu(command,hardware)   根据情况刷新mmu

​	获取commandqueue的offset和剩下的bytes

```c
offset=command->offset      //kernel commandqueue的offset
bytes=command->pagesize-offset //剩下的字节数
```

​	通过下面函数获取waitlink命令的size=32(waitlinkbytes)

```
gckwlfe_waitlink(hardware,gcvnull,0,0,&waitlinkbytes,...)
```

​	获取kernel commandqueue逻辑地址

```
waitlinklogical=command->logical+offset;
waitlinkaddress=command->address+offset;
```

​	看是否需要switch pipes对于commandbuffer，不需要切换offset=32，切换offset=0，如果是相同的context（entry地址值是commandbuffer起始地址），不同context（entry地址值是context->buffer起始地址）

```c
//得到entry地址和size
entrylogical=commandbufferlogical+offset;
entryaddress=commandbufferaddress+offset;
entrysize=commandbuffersize-offset;
```

​	得到kernel commandqueue的结束地址，也是下一个kernel commandqueue的起始地址

```
exitaddress=waitlinkaddress
exitbytes=waitlinkbytes
```

​	进入gckwlfe_waitlink函数，在kernel commandqueue中插入wait\link序列（因为传进去的waitlinklogical是数值，所以waitlinklogical本身地址不变，但地址对应的内容发生变化）,link指向地址是waitlinkaddress即当前wait\link的起始位置

```c
gckwlfe_waitlink(hw,waitlinklogical,waitlinkaddress,offset,&waitlinkbytes,&waitoffset,&waitsize)
	...
	oplogical = logical;
	*oplogical++ = （配置wait寄存器操作，设置wait下个命令被获取的时钟周期数）;
	oplogical++;
	...
	*oplogical++ = （配置link寄存器操作）;
	*oplogical=address 
```

​	获得commandbuffer的末尾位置，即cmd的末尾

```
commandbuffertail=commandbufferlogical+commandbuffersize-commandbuffer->reservedtail
```

​	在commandbuffer的末尾插入fence命令（command->commitstamp 值作为fence data）

```c
gckhardware_fence(hw,gcvengin_render,commandbuffertail,command->fence->address,command->commitstamp,&fencebytes)
	_fencerender()
		gckwrite_memory(logical,load_state...,gcregfenceaddresshiregaddrs)
		gckwrite_memory(logical,fenceaddress >> 32)
		gckwrite_memory(logical,load_state...,gcregfenceaddressregaddrs)
		gckwrite_memory(logical,fenceaddress)
		gckwrite_memory(logical,load_state...,gcregfencedatahighregaddrs)
		gckwrite_memory(logical,datahigh)
		gckwrite_memory(logical,load_state...,gcregfencedataregaddrs)
		gckwrite_memory(logical,datalow)
```

​	配置寄存器，设置fence的data和地址（在commandbuffertail末尾写入load_state寄存器值和fence的data\address），获得fence命令的size（24字节\32字节），一个配置寄存器操作占4字节数据

​	更新commandbuffer的末尾位置

```
commandbuffertail += fencebytes
```

​	在commandbuffer末尾插入link序列，将link地址写入commandbuffer末尾，配置link相关寄存器写入commandbuffer，linkbytes=16字节（link地址是exitaddress=waitlinkaddress，也就是kernel commandqueue中当前wait\link序列的起始位置）

<img src="C:\Users\OT\AppData\Roaming\Typora\typora-user-images\image-20250912101547737.png" alt="image-20250912101547737" style="zoom:50%;" />

```
gckwlfe_link(hw,commandbuffertail,exitaddress,exitbytes,&linkbytes,&exitlinklow,&exitlinkhigh)
```

```
gckvidmem_node_cleancache() 清除cache
```

​	在command->waitpos.logical（kernel commandqueue中之前wait\link序列的起始位置）处插入link序列，将link地址写入command->waitpos.logical，配置link相关寄存器写入command->waitpos.logical，link地址是entryaddress，是commandbuffer或context->buffer的起始地址）

<img src="C:\Users\OT\AppData\Roaming\Typora\typora-user-images\image-20250912101753935.png" alt="image-20250912101753935" style="zoom:50%;" />

```c
//将之前wait\link处的wait用link覆盖，指向commandbuffer或context->buffer的起始地址
gckwlfe_link(hw,command->waitpos.logical,entryaddress,entrybytes,&command->waitpos.size,...)
```

```
gckvidmem_node_cleancache() 清除cache
```

​	 更新command->offset

```
command->offset += waitlinkbytes;           
```

​	更新kernel commandqueue中的wait\link序列地址

```
...
command->waitpost.logical=waitlinklogical+waitoffset
command->waitpos.address=waitlikaddress+waitoffset
command->waitpos.size=waitsize
```

返回command_commit函数

```c
do{
        ...
        gckcommand_exitcommit(command,false) //释放一些command的相关锁和信号
        ...
        next=cmdloc->next //获取下一个interface->submit.commandbuffer
        userptr=next;
  }while(userptr)
  *commitstamp=command->commitstamp;
  command->commitstamp++;    //更新commitstamp值（commandbuffer中插入的fence值）
```

### gckevent_commit函数

​	gckevent_commit函数负责对UMD的record通过event进行提交，一共有两处调用，第一处在提交cmdbuffer后，因为UMD中subcommit->queue = 0，所以commitcmd后进入gckevent_commit函数，queue中的record为空，但还是可以通过函数内的gckevent_submit提交event 

​	在销毁stream或context时会在UMD中配置record（每个mp配置一个record），然后调用gckevent_commit（调用mpcount次）将UMD中的record通过event提交

```c
gckevent_commit(eventobj,queue,...)
	...
    while(queue != null) 
        gckOS_CopyFromUserData(...) //将UMD中的record信息copy到record中
        ...
        gckEVENT_AddList(...) //将record插入到event queue中
        next = record->next
        queue = next
	eventattr.wait=true;
	eventattr.shared=shared;
	eventattr.frompower=false;
	eventattr.broadcast=true;
	gckevent_submit(event,&eventattr);
```

#### gckevent_submit函数

​	负责将event queue中的record提交给GPU

​	主要有3处调用event_submit，除了在gckevent_commit会调用，还会在gckHARDWARE_SetPowerState->gckCOMMAND_Stall和drv_open->gckKERNEL_AttachProcess两处调用

​	gckHARDWARE_SetPowerState在将设备电源状态从ON设置为OFF时，会先调用_PmStallCommand函数对GPU进行延时，通过gckCOMMAND_Stall函数等待cmd全部完成

```c
_PmStallCommand(Hardware, Command, Broadcast)
	if (Broadcast) 
        gcmkONERROR(gckHARDWARE_QueryIdle(Hardware, &idle));
    else 
        status = gckCOMMAND_Stall(Command, gcvTRUE); //等待cmd全部完成
        for (;;) 
            gcmkONERROR(gckHARDWARE_QueryIdle(Hardware, &idle));
     		if (idle)
                break;
```

​	gckCOMMAND_Stall函数负责等待cmdqueue中的cmd执行完，先配置record中的信号，将record添加进event，然后调用gckEVENT_Submit提交event，睡眠等待signal被唤醒，等到GPU读取到kernelcmdqueue中的event后，返回中断给CPU，在中断处理函数中对record中的信号进行处理唤醒睡眠的signal

```c
gckCOMMAND_Stall
    ...
	gckOS_CreateSignal(os, gcvTRUE, &signal) //create 1个信号，signal->done = 0
    gckEVENT_Signal(eventObject, signal, gcvKERNEL_PIXEL) //添加信号
    gckEVENT_Submit(eventObject, &eventAttr) //提交event
    do{
        gckOS_WaitSignal(os, signal, !FromPower, gcdGPU_ADVANCETIMER) //睡眠等待信号
        ...
    }while
```

​	gckos_waitsignal函数负责等待信号，等待期间会睡眠，当CPU收到event中断对record进行处理时会将signal->done设置为1，唤醒睡眠的signal

```c
gckos_waitsignal
    ...
    spin_lock(&signal->lock);
    done = signal->done;  //获得signal的done参数
    spin_unlock(&signal->lock);
	//done为0时睡眠
	if (done) {
        status = gcvSTATUS_OK;
        if (!signal->manualReset)
            signal->done = 0;
    } else {
        //设置超时时间
        long timeout = (Wait == gcvINFINITE) ?
                       MAX_SCHEDULE_TIMEOUT : msecs_to_jiffies(Wait);
        //对该signal进行睡眠，当signal->done为1时唤醒睡眠
        wait_event_timeout(signal->wait, signal->done, timeout);
        ...
```

​	gckKERNEL_AttachProcess函数负责附加或分离进程到kernel，当分离进程时会调用gckEVENT_Submit函数提交event，触发中断通知CPU进程被分离（一共分离mpcount个进程）

​	获取kernel commandqueue和event queue

​	event->queues一共28个（queues中每个元素是一个链表）

```
command=event->command
...
queue=event->queuehead
```

​	通过gckcommand_entercommit函数获得相关锁和信号

​	获得当前commitstamp

```
commitstamp=command->commitstamp;
commitstamp -= 1;      之前更新加过1
```

​	在eventqueue中轮询所有的queue

​	获得event ID，遍历28个id，找到queues中head为0的id作为当前分配的id，配置queues[id]信息(time stamp,source)

```c
queue=event->queuehead //获得当前event
gckevent_getevent(event,wait,&id,queue->source)
	...
	id=event->lastid  //获得之前的eventid
	for(int i=0;i<event->totalqueuecount;i++) 
		nextid=id+1
		if(event->queues[id].head == null) //如果queues中对应id的head为null
			eventid=id           //将这个id作为分配的id
			event->lastid=nextid 更新lastid
			
			event->queues[id].head= ~(0)
			event->queues[id].stamp=++(event->stamp) //配置time stamp
			event->queues[id].source=source //配置source
			--event->freequeuecount //减少free的event数量
		id=nextid //不符合条件继续寻找id
```

​	配置queues[id]的head和commitstamp，head为event中的record（queue->head = record）

```c
event->queues[id].head = queue->head
event->queues[id].commitstamp = commitstamp
```

​	更新event->queuehead=event->queuehead->next，如果已经是末尾，将event->queuehead和event->queuetail置0

​	将queue插入到event->freelist中

```
gckevent_freequeue(event,queue)
```

​	根据event->queues[id].head->info.command获取需要flash的内容 (0)

​	获取event 命令的size大小，将event->queues[id].source转为gckkernel_blt（BLT as DMA copy engine），一共32字节

```
gckwlfe_event(hw,null,id,event->queues[id].source,&bytes)
```

​	获取flush命令的size大小，因为flush为0，flushsize为0

​	总的executebytes=bytes+flushsize

​	在kernel commandqueue中预留空间，返回kernel commandqueue中预留空间的起始位置和commandqueue 剩余的size

```c
gckcommand_reserve(command,bytes,&buffer,&bytes)
	//将bytes与8对齐，获得requestedaligned=32
	//获取waitlink命令的size=32
	gckwlfe_waitlink(hw,null,...&requiredbytes,...)
	//计算需要的总size=64
	requiredbytes += requiredaligned
	...
	//获得kernel commandqueue中reserve空间的位置
	buffer=command->logical+command->offset
	//获得kernel commandqueue剩余的size
	*buffersize=command->pagesize-command->offset
```

​	在kernel commandqueue中添加flush（flush=0）

```
gckhardware_flush(hw,flush,buffer,&flushbytes)
```

​	移动commandqueue中的位置，buffer=buffer+flushbytes

​	在kernel commandqueue中添加event

```c
gckwlfe_event(hw,buffer,id,event->queues[id].source,&bytes)
	将source由gcvkernel_pixel转为gcvkernel_blt
	获得event命令的size=32
	...
	根据source使能EVENT发送，表示event是blt所发的
	配置寄存器的值并写入kernel commandqueue，进行blt控制和bltcluster控制，设置event的id
```

​	执行event，executebytes即为event的size，在event后插入wait\link序列（配置wait\link寄存器的值并写入到kernel commandqueue指定位置），将之前wait\link处的wait用link覆盖，指向event地址

<img src="C:\Users\OT\AppData\Roaming\Typora\typora-user-images\image-20250912103239261.png" alt="image-20250912103239261" style="zoom:50%;" />

```c
gckcommand_execute(command,executebytes)
    //command->offset+event的size
	waitlinkoffset=command->offset+32;
	//commandqueue剩余的size
	waitlinkbytes=command->pagesize-waitlinkoffset;  
	//wait\link序列在kernel commandqueue中的起始位置
	waitlinklogical=command->logical+waitlinkoffset;
	waitlinkaddress=command->address+waitlinkoffset;
	//在kernel commandqueue中添加wait\link序列，返回的waitlinkbytes为wait\link的size=32
	//link指向地址是waitlinkaddress即当前wait\link的起始位置
     gckwlfe_waitlink(hw,waitlinklogical,waitlinkaddress,waitlinkoffset,&waitlinkbytes,...)
     ...
     //计算event在commandqueue的起始位置
     execlogical=command->logical+command->offset
     execaddress=command->address+command->offset
     execbytes=32+waitlinkbytes
     ...
     //在command->waitpos.logical（kernel commandqueue中之前wait\link序列的起始位置）处插入link序列，将link地址写入command->waitpos.logical，配置link相关寄存器写入command->waitpos.logical，link地址是execaddress，是event的起始地址）
      gckwlfe_link(hw,command->waitpos.logical,execaddress,execbytes,..)
      ...
      //更新kernel commandqueue中的wait\link序列地址
      ...
      command->waitpost.logical=waitlinklogical+waitoffset
      command->waitpos.address=waitlikaddress+waitoffset
      command->waitpos.size=waitsize
      //更新command->offset
      command->offset += 32+waitlinkbytes
```

​	释放相关锁和信号

```
gckcommand_exitcommit(command,frompower)
```

​	返回_commit函数

​	next=subcommit->next

​	每次提交cmd都会在kernel commandqueue中创建wait\link序列，每个wait\link序列中的link都指向序列中的wait，然后之前创建的wait\link序列中的wait会变为link指向commandbuffer的起始地址（FE取走commandbuffer中的命令），在commandbuffer后插入fence和link，这个link指向新创建的wait，在没有命令执行期间，gpu会在wait\link之间空转

![image-20250828175134577](C:\Users\OT\AppData\Roaming\Typora\typora-user-images\image-20250828175134577.png)

​	当需要中断时，会在kernel commandqueue中插入event和wait\link，其中之前的wait\link序列中的wait会变为link指向event，gpu执行event会触发中断，cpu收到中断，根据eventid执行相应的中断处理函数

<img src="C:\Users\OT\AppData\Roaming\Typora\typora-user-images\image-20250912103239261.png" alt="image-20250912103239261" style="zoom:50%;" />

#### event的作用

​	当需要更新commandqueue时会插入event，gpu执行完commandbuffer中的cmd后执行event，GPU执行相应的event命令时，会将eventid写入相应寄存器并触发中断（每个mp各有中断源），cpu收到中断，先读取相关寄存器获取eventid，在中断处理函数中进行相应的中断处理，对于有record的event会对record中的signal进行处理，唤醒睡眠的signal

​	每次提交cmd后会插入event触发中断，通知CPU cmdqueue中的cmd提交完成，当销毁stream、context或者进程时也会提交event通知CPU销毁完成，每次提交的event中有些会插入record（中断函数中进行signal处理），有些不插入record（只负责通知CPU）

### interrupt线程 （cmodel会在initdriver阶段创建）

​	cmodel中一共6个线程，线程1是主线程，线程2、线程3在驱动初始化时创建（initdriver-inithal-initkmd-...-cmodelconstructor)，分别为interrupt线程和cmodel线程，线程4、5、6在设备初始化时创建（initdevice-createcontext-createhalcontext)，线程4、5分别负责用于处理fence和资源释放，线程6负责cmd命令的提交

![image-20250829135804435](C:\Users\OT\AppData\Roaming\Typora\typora-user-images\image-20250829135804435.png)

```c
for(;;)
	while(wrapper->intereuptsuspended);
	//获取kernel
	kernel=wrapper->emulator->getkernel(wrapper->devid,wrapper->core)
	gckhardware_intereupt(kernel->hardware);
	gckkernel_notify(kernel,gcvnotify_interrupt);
```

#### gckhardware_interrupt(kernel->hardware)函数

​	从相应寄存器读取gpu发的eventid

```c
gckos_readregisterex(...,&data) //data是eventid
```

​	将eventid添加到pending（待处理的eventid）

```c
gckos_interrupt(hw->kernel->eventObj,data)
	gckos_atomsetmask(event->pending,data)//将eventid添加到pending
```

#### gckkernel_notify(kernel,gcknotify_intereupt)函数

​	根据gcknotify_interrupt，进行中断处理

```
case gcknotify_interrupt:
	gckhardware_notify(kernel->hardware)
```

##### 	进入gckhardware_notify函数

```c
gckhardware_notify(kernel->hardware)
	gckos_atomget(os,kernel->eventObj->pending,&pending); //获取pending
	gckos_atomsetmask(kernel->eventObj->pending,pending); //添加pending到event->pending
	gckevent_notify(kernel->eventObj,0,&fault)
```

##### 	进入gckevent_notify函数

​      event->notifystate=0 //表示开始处理event

​	先处理特殊eventid（特殊中断），再遍历普通的eventid

```c
for(;;)
    ...
	gckos_atomget(os,event->pending,&pending); //获取pending(eventid)
	if(pending==0) //没有event
        break;
     //event 30-32特殊处理
	if(pending & (1<<29)) //eventid == 30
	if(pending & 0x80000000) //eventid == 32 AXI BUSERROR
        pending &= 0x7FFFFFFF //将pending置为0，退出event_notify
        //进行设备复位
	if(pending & 0x40000000) //eventid == 31
    for(i =0 ;i < event->totalqueuecount; i++) //通过比较stamp找到最早的event
	   if(queue == null || event->queues[i].stamp < queue->stamp)
            queue=&event->queues[i]
            mask = 1 << i
    //又遍历一遍，看是否有遗漏
    gckos_atomclearmask(event->pending,mask) //根据mask更新pending，最后更新为0 (3--2--0)
```

​	等待fence，gckfence_signal()  （没使用）fence->waitinglist中没有数据

​	取出record

``` c
record=queue->head
queue->head=NULL //取出后置为null
event->freequeuecount++ //增加free的event数量
```

​	while在上面的for循环中

​	如果record不为空，进行signal处理，在每个event的record链表中轮询

```c
while(record != null)
	recordnext=record->next
	...
	switch(record->info.command)
		case gcvhal_signal:
			signal = record->info.u.signal.signal //获得信号
             if(record->info.u.signal.process == null)
                 gckos_signal(os,signal,ture)  //kernel signal
             else  							//usersignal
```

##### 	gckos_signal(os,signal,ture)函数

​	处理record中的信号，state为true时唤醒等待的signal

```c
gckos_signal(os,signal,state)	
	_querysignalid(os,(uint)signal,&signal) //根据获得的signal ID从os->signal_arr-	>signals[id]中获取signal
	spin_lock(&signal->lock)    //给signal加自旋锁
	if(state)
	{
   		signal->done = 1 //将done设置为1
    	wake_up(&signal->wait) //唤醒等待的signal
	}else{
    	signal->done = 0
	}
	spin_unlock(&signal->lock)   //解锁
```

​	将处理后的record插入event->freeeventlist链表

​	更新record，record=recordnext

​	结束event处理，event->notifystate = -1



## 	gckcommand_construct函数

​	创建2个commandqueue，初始化一些参数

```c
command_construct(kernel,fetype,*command)
	... 先分配command结构体，设置各种参数，创建各种锁等操作
    for(int i=0;i<2;i++)
        //分配commandqueue的内存结点
        gckkernel_allocatevideomemory(...,&videomem)
        command->queues[i].videomem=videomem
        command->queues[i].pool=pool
        获得对应的cpu\gpu逻辑地址  command->queues[i].logical\command->queues[i].address
        //create2个kernelcmdqueue的信号
       gckOS_CreateSignal(os, gcvFALSE, &command->queues[i].signal);
	   //唤醒信号，将signal->done设置为1
	   gckOS_Signal(os, command->queues[i].signal, gcvTRUE);
```

## 	gckcommand_start函数

​	启动commandqueue，先在commandqueue起始处插入waitlink序列

```c
gckcommand_start(command)
    command->fetype == gcvhw_fe_wait_link
	_startwaitlinkfe(command)
    	//根据硬件参数获得waitlink的size
    	gckwlfe_waitlink(hw,...,&waitlinkbytes,...)
    	//commandqueue的开头地址
    	logical =command->logical+command->offset
    	address=command->address+command->offset
    	//在commandqueue起始处插入waitlink，link指向wait(address)
    	gckwlfe_waitlink(hw,logical,address,0,&waitlinkbytes,&waitoffset,&command->waitpos.size)
    	//之后更新commandqueue位置
```

<img src="C:\Users\OT\AppData\Roaming\Typora\typora-user-images\image-20250910154505549.png" alt="image-20250910154505549" style="zoom:50%;" />

​	gckwlfe_execute(hw,address,waitlinkbytes)      配置相关寄存器（使能所有的中断 31个， 配置commandqueue的地址，使能commandqueue解析以及预取位置等）

##        	commandqueue切换

​	commandqueue空间不足时，切换commandqueue，交错进行，切换到1，event存的是0的信号（唤醒0）

​	主要通过_newqueue函数，包括gckos_waitsignal（阻塞线程，等待cmd执行完切换queue）和gckevent_signal（配置signal-record并挂到event上）两个函数

```c
if(bytes < waitlinkbytes)
	_newqueue(command,false)
    	currentindex=command->index
    	newindex=(currentindex+1)%2
    	//阻塞，等待中断处理函数唤醒
    	gckos_waitsignal(command->os,command->queues[newindex].signal,false,kernel->timeout)
    	...
    	//将record插入eventqueue(将record与eventqueue关联)
    	gckevent_signal(command->kernel->eventobj,command-queues[curidx].signal,gckkernel_command)	
```

​	当第一次进入_newqueue函数进行commandqueue切换时，进入gckos_waitsignal函数，因为之前gckcommand_construct函数中会将各commandqueue对应的signal的done参数设置为1，manualreset参数设置为0

​	所以第一次进入会跳过阻塞，并将signal的done参数设置为0

<img src="C:\Users\OT\AppData\Roaming\Typora\typora-user-images\image-20250917100246205.png" alt="image-20250917100246205" style="zoom:67%;" />

​	之后会进入gckevent_signal函数配置signal参数，将record（signal）插入eventqueue

```c
gckevent_signal(event,signal,fromwhere)
	//配置相关参数，信号类型等
    iface.command=gcvhal_signal 
    iface.u.Signal.signal=ptr_to_uint64(Signal)
    ...
    gckevent_addlist(event,&iface,fromwhere,false,true)
    	gckevent_allocaterecord(event,allocateallowed,&record) //record分配空间
    	gckos_memcpy(&record->info,iface,...) //给record传递参数
    	...
    	//将record插入eventqueue
    	if(event->queuetail == null || event->queuetail->source < fromwhere)
            gckevent_allocatequeue(event,&queue)
            queue->source=fromwhere
            queue->head=null
            queue->next=null
            if(event->queuetail == null)
                event->queuehead=queue
                event->queuetail=queue
    	//将record插入queue
         queue->head=record
         queue->tail=record
```

​	当done为0时，即不是第一次进入_newqueue时，会阻塞线程，等待中断处理函数唤醒

```c
gckos_waitsignal(command->os,command->queues[newindex].signal,false,kernel->timeout)
    		//根据signal ID从os->signal_arr->signals[id]中获取signal
    		_querysignalid(os,(uint)signal,&signal) 
    		//设置超时时间  timeout = 20000ms
    		gettimeofday(&tv,null) 
    		//tv--ts
    		ts.tv_sec += timeout/1000
    		ts.tv_nsec += (timeout%1000)*1000
    		pthread_cond_timewait(&signal->cond,&signal->mutex,&ts) //阻塞，等待唤醒 
```

​	最后更新新commandqueue的参数

<img src="C:\Users\OT\AppData\Roaming\Typora\typora-user-images\image-20250910140633569.png" alt="image-20250910140633569" style="zoom:50%;" />

​	返回_commitwaitlinkonce函数，更新commandqueue的offset和剩余的bytes

```c
offset=command->offset
bytes=command->pagesize-offset
```

​	在第一次调用_newqueue之前，不会对event中的queue进行分配，所以event的queuehead为0

​	当调用 _newqueue时会对event的queue进行分配，event的queuehead不为空，所以执行完 gckcommand_commit后会进入 gckevent_commit函数对event进行提交

​	所以之后的每次cmd提交后都会提交一个event（每次运行完 gckcommand_commit函数后都会运行 gckevent_commit函数），这样当切换commandqueue时，gpu可以及时触发中断来唤醒阻塞线程

​	而第一次切换commandqueue时会直接切换到另一个commandqueue1，不需要阻塞等待（刚开始时commandqueue为空，不需要清理），在新commandqueue1中插入wait\link和event(record中的signal是旧commandqueue0的signal)，这样之后需要切换回旧的commandqueue0时，会阻塞线程，并等待signal0来唤醒（通过commandqueue1中event的record触发中断来唤醒）

<img src="C:\Users\OT\AppData\Roaming\Typora\typora-user-images\image-20250917133626082.png" alt="image-20250917133626082" style="zoom:67%;" />

## kernel_recovery

​	当触发AXI_Bus_Error中断，在退出gckevent_notify函数后会通过kernel_recovery进行recovery

<img src="C:\Users\OT\AppData\Roaming\Typora\typora-user-images\image-20250925173257326.png" alt="image-20250925173257326" style="zoom: 50%;" />

### gckhardware_reset

​	先进行设备复位

​	先设置设备的power状态

![image-20250925151737729](C:\Users\OT\AppData\Roaming\Typora\typora-user-images\image-20250925151737729.png)

​	从寄存器中读取contextid到hardware->contextid

​	通过配置相关的寄存器复位GPU设备，并将GPU设备初始化

<img src="C:\Users\OT\AppData\Roaming\Typora\typora-user-images\image-20250925152110554.png" alt="image-20250925152110554" style="zoom: 67%;" />

​	将当前的command->context置为null，使在提交commandqueue时触发context切换（可能因为GPU配置发生变化，在下一个context执行commandqueue）

```c
command->currcontext = null //触发context切换
gckcommand_start(command)
```



<img width="757" height="309" alt="image" src="https://github.com/user-attachments/assets/9c7156c8-3661-49d8-9163-1e4bae43c69c" />





​	退回到kernel_notify函数，读取event，设置pending，再通过gckevent_notify函数处理中断



