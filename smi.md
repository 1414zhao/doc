## ioctl实现

### 初始化

​	GPU驱动加载时会创建smi_ctrl节点

​	卸载驱动时会注销节点

​	gcsSMI_DRIVER_ARGS结构体负责用户层和驱动层之间的信息传递，device_id表示GPU卡号，InputBuffer和InputBufferSize分别表示用户层的信息结构体地址和信息结构体size，负责将用户层的信息传递到驱动层，OutputBuffer和OutputBufferSize分别表示驱动层的信息结构体地址和信息结构体size，负责将驱动层的信息传递到用户层。

```C
typedef struct _gcsSMI_DRIVER_ARGS {
    gctINT              device_id;
    gctUINT64           InputBuffer;
    gctUINT64           InputBufferSize;
    gctUINT64           OutputBuffer;
    gctUINT64           OutputBufferSize;
} gcsSMI_DRIVER_ARGS;
```

​	KMD中device用MP表示，一个完整设备包括2个MP2和8个MP1

​	用户层的卡号device_id对应驱动中的platformindex，从gal_device获得insmod参数（后面应该会自适应），先获得设备0的MP数量platformDevCounts ，根据platformindex对当前设备起始MP的id startDevIndex和platformDevCounts 进行更新，最后从MP数组devices中将devices[startDevIndex]作为Device 

```C
copy_from_user(&drvArgs, (void *)arg, sizeof(gcsSMI_DRIVER_ARGS));
platformindex= drvArgs.device_id;
//number of MPs on card 0
platformDevCounts = gal_device->args.hwDevCounts[0];
//more than card0 
for (i = 0; i < platformindex; i++) {
    startDevIndex += gal_device->args.hwDevCounts[i];
    platformDevCounts += gal_device->args.hwDevCounts[i + 1];
}
Device = gal_device->devices[startDevIndex];
```

​	对于需要从BIOS获得信息的command，会初始化IPC，返回初始化后的ep结构体，ep结构体中包含tx\rx中的各种参数，Device、gcsSMI_DRIVER_ARGS结构体drvArgs和ep作为接口的参数

```C
    if (ioctlCode >= gcvSMI_GET_DEVICE_UUID) {
        ep = ipc_commu_init(Device);
    }
    ...
    gckBIOS_GetBiosVersion(Device, drvArgs, ep);
```

​	为了实现core利用率的周期采样，会在节点注册时初始化一个延迟工作队列

​	大多数信息结构体与用户层保持一致

### gckKMD_GetDriverVersion

​	以获取driver_version为例，大多数接口都是先初始化一个特定的信息结构体，填充信息后通过copy_to_user将结构体信息传递到用户层

```C
//初始化一个version结构体
gcsSMI_VERSION version_info = {0};
//填充信息
version_info.major = ...
version_info.minor = ...
version_info.patch = ...
version_info.build = ...
//通过copy_to_user将结构体信息传递到用户层
copy_to_user(gcmUINT64_TO_PTR(drvArgs.OutputBuffer),
                                &version_info,
                            sizeof(version_info));
```

### gckKMD_GetDeviceInfo

​	从Device结构体获得部分device信息，deviceCount、device_id、device_name、device_path

```C
typedef struct _gcsSMI_DEVICE_INFO {
    gctUINT32           len;
    gctUINT3            devIndex;
    gctUINT32           deviceCount;
    gctUINT32           device_name[64];
    gctCHAR             device_path[64];
    gctUINT32           uuid;
} gcsSMI_DEVICE_INFO;
```

### gckKMD_GetCoreCount

​	获取cluster总数，包含MP和MPLite中的cluster数，每个MP有8个cluser，每个MPLite有1个cluster，其中KMD中的device以MP2和MP1为单元，一个MP2包含2个MP，MP1就是MPLite，轮询当前设备的MP，对每个device获得cluster数，其中gckKMD_GetCoreCount函数通过从device每个MP中的相关结构体中获得alive的cluster数，所有device的cluster数累加就是最后的corecount  （目前实际上是从insmod中获取usercluster数量）

```C
for (device_id = startDevIndex; device_id < platformDevCounts; device_id++)
    Device = gal_device->devices[device_id];
    status = gckKMD_GetCoreCount(Device, drvArgs, &core_id, &alive_core);
    corecount += alive_core;
```

```C
typedef struct _gcsSMI_CORE {
    gctUINT32           corecount;
    gcsSMI_Utilization  utilization;
}gcsSMI_CORE;
```

### gckKMD_GetTotalMemory

​	获取总显存大小，根据device的platformindex获得对应的pci_info结构体，从pci_info获得当前设备对应的pdev，从显存bar中获取当前显存的总size，不同平台代表显存的bar_num不同，qemu为1，实际设备为2

```C
typedef struct _gcsSMI_MEMORY_INFO {
    gctUINT64           totalBytes;
    gctUINT64           usedBytes;
} gcsSMI_MEMORY_INFO;
```

### gckKMD_GetUsedMemory

​	获取剩余显存，总显存减去所有内存池空闲的内存作为剩余显存

### gckKMD_GetProcessInfo

​	最多保存32个进程信息，轮询process的database填充进程的pid、usedmemory、name等信息，当实际进程数量多于32时，多余的进程信息不会记录，最后会返回用户层实际的进程数量

```C
typedef struct _gcsSMI_PROCESS_INFO {
    gctUINT32           pid;
    gctCHAR             name[64];
    gctUINT64           usedGpuMemory;
} gcsSMI_PROCESS_INFO;
```

#### CreateProcessDB

​	创建进程的database

​	一共16个slot（16个hash值，16个链表），分别保存不同hash值代表的进程，每个slot下的链表保存的是当前hash值的各个进程信息，每个进程分配一个database，进程中的信息保存在database中

hash为0的进程   hash为1的进程  hash为2的进程  hash为3的进程

​	 db[0]		db[1]		db[2]		db[3] ...

​	     |                        |                      |                       |

​	database	database	database	database

​	database	database	database	database	

​	      ...		    ...		    ...		   ...

```c
slot = ProcessID % 16; //用ProcessID计算hash
```

​	在drv_open中，给每个细化的MP attach进程，创建进程的database，多进程下会多次打开drv_open，不同进程的database插入到对应的hash链表

```c
for (i = startDevIndex; i < platformDevCounts; i++) 
    //MP2或MP1
    device = galDevice->devices[i]; 
	//细化的MP，kernel对应细化的MP
	for (j = 0; j < gcvCORE_COUNT; j++)
        //创建进程的database
        gckKERNEL_CreateProcessDB(Kernel, PID)
```

​	初始化database结构体，将database插入到hash对应的链表头

```c
//当前进程对应的slot
slot = ProcessID % 16;
database->slot = slot;
database->processID = ProcessID;
database->vidMem.bytes = 0;
...
database->nonPaged.bytes = 0;
...
database->mapMemory.bytes = 0;
...
//从task->comm获得，最大16个字符，name过长会被截断
database->processName =
//给当前kernel的database分配一个vm空间
if (Kernel->processPageTable)
    gckVM_VMConstruct(Kernel, ProcessID, (gckVM *)&database->vm)
    gckVM_VASpaceAdd((gckVM)database->vm, Kernel, ProcessID, gcvNULL)
//初始化database的list   
for (i = 0; i < gcmCOUNTOF(database->list); i++)
    database->list[i] = null

for (i = 0; i < gcvVIDMEM_TYPE_COUNT; i++)
    database->vidMemType[i].bytes = 0
    ...
//内存池内存相关
for (i = 0; i < gcvPOOL_NUMBER_OF_POOLS; i++)
    database->vidMemPool[i].bytes = 0;
	...
//设备内存相关
for (i = 0; i < gcdDEVICE_COUNT; i++)   
    database->vidMemDevice[i].bytes = 0;
	...
//头插法，将当前的database插入到hash链表头
database->next = Kernel->db->db[slot];
Kernel->db->db[slot] = database;    
```
#### VASpaceAdd

​	给database分配一个vm虚拟内存空间，每个细化的MP分配一个vaSpace，vaSpace作为database->vm的参数

```c
//配置vaSpace参数
space->deviceID = deviceId;  //deviceid
space->kernel = Kernel;      //kernel
gckMMU_ConstructProcessMMU(Kernel, ProcessID, &space->mmu) //mmu
space->vm = database->vm;       
space->mmu->space = space;
//每个kernel一个space
database->vm->vaSpace[deviceId] = space;
```
#### FindDatabase	

​	寻找当前进程对应的database，并把它移到hash链表头

```c
//遍历hash链表
for (previous = gcvNULL, database = Kernel->db->db[slot];database != gcvNULL;database = database->next)
    //找到当前进程对应的database
    if (database->processID == ProcessID) 
       break;
	previous = database;
	//如果当前database不是头结点，将database移到hash链表头
	if (previous != gcvNULL) 
        previous->next = database->next;
        database->next = Kernel->db->db[slot];
        Kernel->db->db[slot] = database;
//返回对应的database
*Database = database
```

#### AddProcessDB

​	分配内存等阶段调用，往database中添加内存等信息，向database的list插入record（保存数据地址等）

​	每个database有一个list链表（48个hash值，48个链表），每个hash链表保存一种数据，record记录数据地址等信息

​	 list[0]		list[1]		list[2]		list[3] ...

​	     |                        |                      |                       |

​	record		record	    record	      record

​	record	        record	    record	      record

​	    ...		      ...		    ...		     ...

```c
slot = handle % 48; //用data地址计算hash
```

```c
//先找到当前进程对应的database
gckKERNEL_FindDatabase(...)
//分配一个record，插入到database的list链表头
gckKERNEL_NewRecord(...)
    //用data的地址得到hash
    slot = handle % 48;
	//头插法，将record插入到list链表头
	record->next = Database->list[Slot];
    Database->list[Slot] = record;
//配置record参数，包括kernel，data地址等
record->kernel = Kernel;
record->type = Type;
record->data = Pointer
...
//根据type更新database的内存参数
if (Type == gcvDB_VIDEO_MEMORY)
	count = &database->vidMemPool[vidMemPool];
    count->totalBytes += Size;
    count->bytes += Size;
	...
```

### gckKMD_GetProcessCount

​	轮询process的database，获取运行中调用设备的进程数量process_count

```C
typedef struct _gcsSMI_PROCESS {
    gctUINT32             count;
    gctUINT32             length;
    gcsSMI_PROCESS_INFO   process_info[PROCESS_MAX_COUNT];
} gcsSMI_PROCESS;
```

### gckKMD_GetPciInfo

​	从pdev获取pcie相关信息，包含segment、bus、device、devfn、vendor 、subsystem_vendor 、subsystem_device、pciMaxWidth、pciMaxSpeed、pciMaxGen、pciCurSpeed、pciCurGen、pciCurWidth、slot_id

​	通过PCI_EXP_LNKCAP和PCI_EXP_LNKSTA获得width和speed等信息

```C
typedef struct _gcsSMI_PCI_INFO {
    gctUINT32           segment;
    gctUINT32           bus;
    gctUINT32           device;
    gctUINT32           devfn;
    gctUINT32           vendor;
    gctUINT32           subsystem_vendor;
    gctUINT32           subsystem_device;
    gctFLOAT            pciMaxSpeed;
    gctFLOAT            pciCurSpeed;
    gctUINT32           pciMaxWidth;
    gctUINT32           pciCurWidth;
    gctUINT32           pciMaxGen;
    gctUINT32           pciCurGen;
    gctCHAR             slot_name[DEVICE_SLOT_NAME_BUFFER_SIZE];
    gctUINT32           slot_id;
    gctINT32            rsvd[6];
} gcsSMI_PCI_INFO;
```

### gckKMD_GetGpuUtilization

​	获取GPU的利用率，从全局数组中取出得到的利用率，与实际cluster数做平均得到最后的利用率，返回给用户层

```C
typedef struct _gcsSMI_Utilization {
    gctUINT32          enable;
    gctUINT32          core_utilization[CLUSTER_MAX_COUNT];
    gctUINT32          gpu_utilization;
}gcsSMI_Utilization;
```

### gckKMD_CoreUtilization

​	初始化ctrl节点时会初始化一个延时工作队列（在安装驱动时只初始化一次），绑定周期采样函数

```c
//绑定周期采样函数gckKMD_CoreUtilization
util_work_ptr = kzalloc(sizeof(gcsUtilization_Work), GFP_KERNEL);
INIT_DELAYED_WORK(&util_work_ptr->work, gckKMD_CoreUtilization);
```

​	获取cluster利用率，最后返回的是每个alive cluster的利用率，每个cluster各有一个cc和tc core，cc和tc有各自的寄存器和时钟，所以单个cluster的利用率为（cc利用率+tc利用率）/2

​	当用户层调用gcvSMI_SET_COREUTILIZATION_TOGGLE指定的接口，设置为1时，会先将一个全局变量enable原子性变为1（当这个enable不为0时会跳过初始化和调度，防止多进程下重复初始化和调度），之后进行core利用率采样的初始化，配置每个device对应的寄存器开启debug状态，清除每个cc/tc时钟的cyclecount，为了多进程安全对初始化流程进行加锁保护，之后开始调度执行gckKMD_GetCoreUtilization函数

```c
//从用户层获得enable
copy_from_user(&utilization,...)
if (utilization.enable) {
    //采样初始化，开始采样
    gckStart_Sample(Device);
} else {
    //结束采样
    gckStop_Sample(Device);
}    	
```

​	gckKMD_GetCoreUtilization函数主要使用AHB计数来统计cc/tc的totalcycle和idlecycle，开启debug后每个cycle相关的totalcycle寄存器值会加1，当相关的模块都idle时idlecycle寄存器会加1

​	每次采样结束后会保存上一次的totalcycle和idlecycle作为last_totalcycle和last_idlecycle，通过他们的差值来计算利用率，因为实际core的最大频率约为1.25GHZ，totalcycle寄存器为32位，将采样周期设置为1s，当寄存器满时会对cycle计数进行清除，每1s函数执行一次，得到1次采样结果，采样结果保存在全局数组中

​	当用户想结束采样设置为0时，enable原子性变为0，函数停止周期执行，通过寄存器关闭debug状态

#### gckStart_Sample

​	当enable为0时原子操作将enable设为1，enable为其他值时跳过，保证多进程下只会进行一次初始化，loop设备的MP（每个MP有各自的寄存器地址空间，间隔1M），开启debug寄存器，清除totalcycle寄存器中的值，开始周期采样

```c
//当enable为0时原子操作将enable设为1，enable为其他值时跳过，保证多进程下只会进入一次
if (atomic_cmpxchg(&util_work_ptr->enable, 0, 1) == 0)
    mutex_lock(&utilization_lock);
	//loop设备的MP，配置MP的采样初始化
     for (i = startDevIndex; i < platformDevCounts; i++) {
            Device = gal_device->devices[i];
         	for (j = 0; j < Device->coreNum; j++) {
                if (Device->kernels[j])
                    //当前MP的hardware
                    hardware = Device->kernels[j]->hardware;
                    if (hardware) 
                        //开启debug寄存器
 					  gckHARDWARE_EnableClusterUtilization(hardware);
                	   //清除totalcycle寄存器中的值
                        gckHARDWARE_ResetCycleCount(hardware);
    //清除gcsCycleCount中的记录值
    memset(&util_work_ptr->cycle_count, 0, sizeof(gcsCycleCount));
    mutex_unlock(&utilization_lock);
    //开始执行周期采样函数gckKMD_CoreUtilization
    schedule_delayed_work(&util_work_ptr->work, 0);
```

#### gckStop_Sample

​	将enable原子性设为0，loop设备的MP，关闭debug寄存器，停止周期函数采样

```c
//将enable原子性设为0
atomic_set(&util_work_ptr->enable, 0);
mutex_lock(&utilization_lock);
//loop设备的MP
for (i = startDevIndex; i < platformDevCounts; i++) 
    Device = gal_device->devices[i];
    for (j = 0; j < Device->coreNum; j++) {
        if (Device->kernels[j])
            //当前MP的hardware
            hardware = Device->kernels[j]->hardware;
            if (hardware) 
                //关闭debug寄存器
              gckHARDWARE_EnableClusterUtilization(hardware);
mutex_unlock(&utilization_lock);
//关闭延时调度
cancel_delayed_work_sync(&util_work_ptr->work);
```

#### gckKMD_CoreUtilization

​	周期采样函数，每1s获得一次所有cluster的利用率

​	loop设备的MP，选中当前MP的寄存器空间，loop当前MP的所有的cluster，每个cluster包含1个cc和1个tc，通过配置clock寄存器选中指定cluster的cc/tc core，读取指定的totalcycle和idlecycle寄存器值，每次读取后保存当前的cycle值作为lastcycle，通过差值计算出利用率

```c
//当前totalcycle差值
delta_total = totalcycle - last_totalcycle
//当前idlecycle差值
delta_idle = idlecycle - last_idlecycle
//利用率（busy率）
load = (delta_total-delta_idle)/delta_total
```

```c
mutex_lock(&utilization_lock);
//loop设备的MP
for (i = startDevIndex; i < platformDevCounts; i++) 
    Device = gal_device->devices[i];
    for (j = 0; j < Device->coreNum; j++) {
        if (Device->kernels[j])
            //当前MP的hardware
            hardware = Device->kernels[j]->hardware;
            if (hardware) 
		   	   //Device为MP2或者MP1
               	cluster_num = (Device->coreNum == 2 ? 8 : 1);
        		//loop 当前MP的所有的cluster
			   for (j = 0; j < cluster_num; j++)
                    //通过配置clock寄存器选中指定cluster的cc/tc core，读取指定的totalcycle和idlecycle寄存					器值，计算出core的利用率
				   gckHARDWARE_QueryClusterUtilization(hardware, load, j, cycle_count)
                     //cc/tc的利用率和均值作为当前cluster的利用率，保存在全局数组中
                     core_utilization[cluster_id] = (load[0] + load[1])/2;
                     cluster_id++;
mutex_unlock(&utilization_lock);
//max freq:1.25GHZ, register overflows in about 3.43s, set sampling period to 1s
//当enable为1时会周期调度，enable为0停止函数周期运行，退出函数
if (atomic_read(&utilization_work->enable))
    schedule_delayed_work(&utilization_work->work, msecs_to_jiffies(1000))
```

## gckKMD_GetCoreUtilization

​	从全局数组中取出得到的利用率，返回给用户层

### gckBIOS_GetBiosVersion

​	获取BIOS的version，通过一块共享内存使用ringbuffer将BIOS侧获得的version信息传递到KMD侧

```C
typedef struct _gcsSMI_BIOS_MSG {
    SMI_COMMAND_CODES_t bios_command;
    gctINT32            event_pri;
    gctBOOL             need_reply;
    gctINT8             data[RING_BUFFER_SIZE];
} gcsSMI_BIOS_MSG;
```

其他BIOS接口暂未实现

## 用户层

​	以gcvSMI_GET_DRIVER_VERSION为例，对于不需要配置参数到KMD的接口gcsSMI_DRIVER_ARGS 的InputBuffer可以不设置

```C
gcsSMI_VERSION version = {0};     //version结构体
gcsSMI_DRIVER_ARGS version_args = {0}; //gcsSMI_DRIVER_ARGS结构体，负责用户层和驱动层的信息传递
version_args.device_id = 0;            //设备卡号
version_args.InputBuffer  = (uintptr_t)&version;//version结构体地址，驱动由此获取信息（信息传进驱动）
version_args.OutputBuffer = (uintptr_t)&version;//version结构体地址，用户由此获取信息（信息传出驱动）
ioctl(fd, gcvSMI_GET_DRIVER_VERSION, &version_args) //调用ioctl获取指定信息
```















