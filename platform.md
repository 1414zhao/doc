# platform

## 平台设备（GPU）驱动

​	file_operations结构体，对应用层的ioctl\open\close进行实现

```c
static const struct file_operations driver_fops = {
    .owner = THIS_MODULE,
    .open = drv_open,
    .release = drv_release,
    .unlocked_ioctl = drv_ioctl,
};
```

​	platform_driver结构体，静态匹配后执行probe函数，通过name进行匹配

```c
static struct platform_driver viv_dev_driver = {
    .probe = viv_dev_probe,
#if LINUX_VERSION_CODE >= KERNEL_VERSION(3, 8, 0)
    .remove = viv_dev_remove,
#else
    .remove = __devexit_p(viv_dev_remove),
#endif

    .suspend = viv_dev_suspend,
    .resume = viv_dev_resume,
    .shutdown = viv_dev_shutdown,

    .driver = {
        .owner = THIS_MODULE,
        .name = DEVICE_NAME,
#if defined(CONFIG_PM) && LINUX_VERSION_CODE >= KERNEL_VERSION(2, 6, 30)
        .pm = &viv_dev_pm_ops,
#endif
    },
};
```

## 	viv_dev_probe函数

​	静态匹配后执行，将设备中的各种参数赋给platform->params参数，通过drv_init将参数赋给device结构体和kernel结构体，开启kernel和中断线程

```c
viv_dev_probe(platform_device *pdev)
	platform->device=pdev
    //将各module参数默认值赋给platform的params参数
	_initmoduleparam(&platform->params) 
    	p->irqs[i] = irqs[i];
        if (irqs[i] != -1) {
            p->registerBases[i] = registerBases[i];
            p->registerSizes[i] = registerSizes[i];
            ...
     //通过从PCIE结构体获得设备的各参数，赋给platform的params参数（覆盖掉之前的默认参数）
     //配置寄存器地址、中断号、内存地址、id等参数
     if (platform->ops->adjustParam)
        platform->ops->adjustParam(platform, &platform->params);
     //将platform中的params参数赋给module参数，为之后的drv_init使用
     _SyncModuleParam(&platform->params);
     //create device对象（将参数配置到device结构体中），start device开启kernel和中断线程
     drv_init()
```

### 	drv_init()函数

​	将palamform->params参数配置到device结构体，根据register的物理地址通过ioremap获得寄存器的逻辑地址，根据地址、size参数对contiguous、local内存进行内存配置，为之后分配内存做准备，为每个core分配kernel，start device开启kernel和中断线程

```c
drv_init()
	gckGALDEVICE_Construct(platform, &platform->params, &gal_device);
		...                   //配置参数和内存参数
    	gckDEVICE_AddCore();  //为每个core分配kernel
		...
	gckGALDEVICE_Start(gal_device)；
     galDevice = gal_device;
```

#### 	gckGALDEVICE_Construct函数

​	创建device、kernel、hardware、MMU、command、event等重要结构体，配置参数，进行硬件的初始化，最后返回一个device对象

```c
gckGALDEVICE_Construct(Platform,Args,Device)
	//创建device结构体，为device结构体配置platform中的param参数
	//Set up register memory region，根据获得的寄存器物理地址通过ioremap映射到逻辑地址
    //Setup contiguous video memory，根据param参数配置内存参数
    //Setup all the local memory，根据param参数配置内存参数
	for(i =0;i<8;i++)
        //为每个core分配kernel
        gckDEVICE_AddCore(...,&device->kernels[i])
	    	gckKERNEL_Construct(...)
        		//创建kernel结构体，给kernel分配device等参数
        		//创建kernel->db结构体
        		//Construct a video memory name database（包括db中pointer table和bitmap的创建）
        		gckKERNEL_CreateIntegerDatabase(kernel, 512, &kernel->db->nameDatabase)
        		//Construct a pointer name database（包括db中pointer table和bitmap的创建）
        		gckKERNEL_CreateIntegerDatabase(kernel, 512, &kernel->db->pointerDatabase)
        		//Initialize video memory node list
        		gcsLIST_Init(&kernel->db->videoMemList)
        		//Initialize video memory node rb-Tree
        		gcsRBTREE_Init(&kernel->db->videoMemRBTree)
        		//给kernel分配params中的recovery等参数
        		gckOS_QueryOption(Os, "recovery", &data);
				gckOS_QueryOption(Os, "sRAMLoopMode", &data);
				...
                  //创建hardware结构体，分配kernel等参数，进行设备硬件的一些初始化
                  gckHARDWARE_Construct(...)
                  //创建MMU结构体，配置MMU
                   gckMMU_Construct(...)
                   ...
                  //为FE创建command对象和event对象
				gckCOMMAND_Construct(...)
                  gckEVENT_Construct(...)
                  //Initialize the GPU
                  gckHARDWARE_InitializeHardware(...)
                  //Start the command queue
                  gckCOMMAND_Start
                  *Kernel = kernel
	//最后返回一个device结构体
```

#### 	gckGALDEVICE_Start函数

​	创建kernel线程（一共mp个，12个）、中断线程（一共mp个，12个）或注册中断

```c
gckGALDEVICE_Start
for(0;<devidx;devidx++)
    //创建kernel线程 
	for (i=0;i<8;++i)
		_StartThread(device, i)
        	//创建threadRoutine线程
        	threadRoutine
                while(1)
                    //线程函数负责从pending读取eventid，处理event
                    gckHARDWARE_Notify(...) 
        		//线程优先级设置为最高
                set_user_nice 
     //创建isr routine 
     for (i=0;i<8;++i)
         _SetupIsr(device, i)
         	//polling模式
         	if (isrPolling) 
                //创建isrRoutine线程 
                isrRoutine
                    while(1)
                        //线程函数负责读取eventid，将id传进pending
                        gckHARDWARE_Interrupt(...) 
          	//MISX模式
            handler = isrRoutine
            //通过platform中的registerirq注册中断，isrRoutine作为处理函数，触发中断后最后会跳转到处理函数
            platform->ops->registerIrq(platform, Device->platformIndex, \
                Device->irqLines[Core], handler, (void *)Device->kernels[Core]
```

## 	入口函数viv_dev_init

​	装载驱动时调用，先添加平台设备，将pcie设备注册到pci子系统，初始化zixia_test，返回一个platform对象，将平台设备驱动（gpu驱动）挂载在平台设备

```c
viv_dev_init()
    //platform初始化，注册PCIE设备和zixia_test，返回一个platform对象
    gckPLATFORM_Init(&viv_dev_driver,&platform)
    //only_platform_driver 跳过注册平台设备（GPU）驱动
    if (only_platform_driver) {
        return 0;
    }
    //将平台设备驱动挂载在平台
	platform_driver_register(&viv_dev_driver)
    //将平台驱动地址作为platform结构体的一个参数
    platform->driver=&viv_dev_driver
```

# platform_zixia

## 	default_ops函数指针

​	通过default_ops定义一些自定义函数指针来实现功能，这些函数作为上层和pci之间的中转，本质上都是获得pci设备，通过pci结构体中的信息或函数指针实现功能，default_ops作为platform结构体的参数

```c
static struct _gcsPLATFORM_OPERATIONS default_ops = {
    .adjustParam    = _AdjustParam,
    .freeIrq        = _FreeIrq,
    .registerIrq    = _RegisterIrq,
    .getGPUPhysical = _GetGPUPhysical,
    .getCPUPhysical = _GetCPUPhysical,
    .queryPCIInfo   = _QueryPCIInfo,
};
```

​	_RegisterIrq函数对对应的中断号注册中断，从platform结构体获得pci设备，通过pci设备获得pci结构体，通过结构体中的信息获得中断号对应的中断向量，通过结构体中的函数指针调用zixia_chip_register_irq函数注册中断处理函数msi_isr_routine，触发中断时会在msi_isr_routine函数中调用传进来的处理函数handler

```c
_RegisterIrq(Platform, PlatformIndex, irq, handler,handle_args)
    //从platform结构体中获得pcie设备指针
	*pcie_platform = (struct _gcsPLATFORM_PCIE *)Platform;
	*pdev = pcie_platform->pcie_info[PlatformIndex].pdev；
     //通过pci_get_driverdata获得pci结构体，根据pci结构体中的信息找到中断号对应的中断向量
     msi_vector = _Find_msi_vector(pdev, irq, dev_id_start, kernel);
     //获得pci结构体，根据pci结构体中的函数指针调用zixia_chip_register_irq函数注册中断
	 hal_zixia_chip_register_irq(pdev, irq, msi_vector, handler, handle_args)
```

​	可以通过platform结构体中的.ops来调用default_ops中的函数指针，还可以通过platform结构体中的pcie_info来调用pcie设备指针和设备对应的probe函数

```c
struct _gcsPLATFORM_PCIE {
    struct _gcsPLATFORM base;
    struct _gcsPCIeInfo pcie_info[gcdPLATFORM_COUNT];
    unsigned int device_number;
#if gcdOT_VERIFY_MEM
    gcsINTLV_MEM    intlv_mems[gcdPLATFORM_COUNT];
#endif
};

struct _gcsPLATFORM_PCIE default_platform = {
    .base = {
        .name = __FILE__,
        .ops = &default_ops,
    },
};
```

## 	gckPLATFORM_Init函数

​	在kernel_platform_zixia中负责添加平台设备，将pci驱动注册到pcie子系统，进行zixia_test的初始化，最后返回一个platform对象（default_platform)

```c
gckPLATFORM_Init
	    //初始化并分配一个指定名字的平台设备
        *default_dev=platform_device_alloc(&viv_dev_driver->driver.name,-1)
        //添加设备
        platform_device_add(default_dev)
	    do {
        	pdev = pci_get_device(vivpci_ids[idx].vendor, vivpci_ids[idx].device, pdev);
            ...
        }while (idx < info_count); //轮询获得可用的PCI设备
		init_completion(&pcie_info->probed); //将pcie_probe添加到等待队列
	    pci_register_driver(&gpu_pci_subdriver); //注册PCI驱动
         wait_for_completion_timeout(&pcie_info->probed, timeout);//设置超时时间等待pcie_probe完成
		*platform = (gcsPLATFORM *)&default_platform;
		 //进行zixia_test初始化
          zixia_test_init(vivpci_ids, sizeof(vivpci_ids) / sizeof(vivpci_ids[0]));
```

### 	zixia_test_init函数

​	负责注册zixia_test设备，包含一些自定义的ioctl实现和uio设备的注册

```c
zixia_test_init
	misc_register(&zixia_test_miscdev.misc_dev)； //注册misc设备，类似功能简单的字符设备
    //轮询可用的pci设备
    do{
        pdev = pci_get_device(pci_ids[id_idx].vendor, pci_ids[id_idx].device, pdev); //获得pci设备
        if(pdev)
            zixia_test_uio_dev_register(&zixia_test_miscdev, pdev);//注册uio设备，用于上层的地址映射
       }while
```

​	出口函数viv_dev_exit，卸载驱动时调用

```c
viv_dev_exit()
    //注销驱动
	platform_driver_unregister(&gpu_pci_subdriver)
    //注销设备
    platform_device_unregister(platform->device)
    //注销zixia_test
    zixia_test_exit();
```

​	module_init(viv_dev_init) 声明入口函数

​	module_exit(viv_dev_exit) 声明出口函数

​	每次加载驱动，会先调用入口函数，向内核注册驱动的相关信息，包括设备类、注册中断等，之后通过platform_driver中的name匹配到设备后开始调用probe函数，进行初始化、注册设备（将file_operations与设备相关联）等操作

​	应用层的open\close\read\ioctl等函数会通过驱动中的file_operations结构体中的函数实现，read\write函数通过copy_to_user和copy_from_user实现内核信息和用户信息的交互，其中ioctl通过定义的cmd进行相关函数的匹配，实现相关功能

### gpu_sub_probe函数

​	用pci_register_driver函数将pci驱动注册到pcie子系统，驱动定义一个struct pci_driver结构体（包括厂商id\设备id等），内核在识别出pcie设备后，会遍历所有已注册的pcie驱动的设备id表，将设备的id与驱动支持的设备id进行对比，匹配后调用probe函数，先对pcie设备进行使能，设置发送数据的master，请求资源，对相关寄存器进行映射等操作，将映射得到的数据保存到pcie设备结构体中

​	PCI驱动主要部分是gc_hal_kernel_platform_zixia中的gpu_sub_probe函数，在匹配设备后进行调用（负责配置PCI设备，设置相关参数，配置中断等）

```c
static struct pci_driver gpu_pci_subdriver = {
    .name      = DEVICE_NAME,
    .id_table  = vivpci_ids,
    .probe     = gpu_sub_probe,
    .remove    = gpu_sub_remove
};
```

#### 配置pci设备	

​	pci_enable_device 使能PCI设备

​	pci_set_dma_mask配置 PCI 设备 DMA 地址的访问范围，设置当前 PCI 设备能够支持的最大物理地址宽度（即设备可访问的内存地址范围），确保 DMA 传输时使用的地址在设备的硬件能力范围内

​	pci_set_master启用 PCI 设备 “总线主控模式，设置发送数据的master，允许 PCI 设备主动发起总线传输

​	pci_request_regions请求 PCI 设备资源（I/O 端口或内存区域）独占访问权，向内核注册 “当前驱动需要使用设备的特定资源”，防止其他驱动或系统组件占用这些资源，确保设备能安全、独占地使用其硬件资源（如寄存器、缓冲区等）

```
gpu_sub_probe(struct pci_dev *pdev, const struct pci_device_id *ent)
{
	pci_enable_device(pdev)
	pci_set_dma_mask(pdev, DMA_BIT_MASK(64))
	pci_set_master(pdev);
	pci_request_regions(pdev, "galcore")
}
```

#### 资源初始化

​	hal_zixia_chip_resource_init-->zixia_chip_resource_init资源初始化，所有的信息保存在对应的结构体中，通过pci_get_drvdata(pdev)调用保存信息的结构体

```c
if (hal_zixia_chip_resource_init(pdev)) {
    hal_zixia_log_err(&pdev->dev, "hal_zixia_chip_resource_init failed.\n");
    goto err_chip_res;
}
```

​	通过 pci_resource_start获取 PCI 设备特定资源（如内存区域或 I/O 端口）的起始物理地址

（PCIe 会枚举所有 PCIe 设备，读取设备的 BAR 寄存器（基地址寄存器），确定从设备的地址范围和类型等记录在struct pci_dev结构体中）

​	调用 ioremap函数，将物理地址映射到虚拟地址，返回可用于访问该物理区域的虚拟地址

​	通过 readl从映射的虚拟地址读取设备的 die 数量和类型

```c
edu_info_base = pci_resource_start(to_pci_dev(chip->dev), chip->pcie_reg_bar.bar_num) + EDU_CHIP;
edu_info_ptr = ioremap(edu_info_base, sizeof(edu_info));
edu_info.die_count = readl(edu_info_ptr);
edu_info.die_type = readl(edu_info_ptr + 4);
```

配置 ddr 内存的容量分配和预留空间

两个 die，每个 die256MB

四个 mp，每个 mp128MB

总的 ddr 容量 die0+die1=512MB

显存预留空间配置 32MB

配置 outbound 地址空间

配置 outbound 地址空间的起始地址为 0

配置 outbound 地址空间的总容量为 128TB（决定cpu能访问的gpu资源范围）

根据 die 的类型进行不同属性的配置（配置 mp_count 和 mp_lite_count）

```c
switch (edu_info.die_type) {
case CC9100MP1X1: {
    chip->gpu_info.die_num = edu_info.die_count;
    for (i = 0; i < chip->gpu_info.die_num; i++) {
        chip->gpu_info.die[i].mp_count = 0;
        chip->gpu_info.die[i].mp_lite_count = 1;
    }
    break;
}
case CC9800MP2X1: {
    chip->gpu_info.die_num = edu_info.die_count;
    for (i = 0; i < chip->gpu_info.die_num; i++) {
        chip->gpu_info.die[i].mp_count = 2;
        chip->gpu_info.die[i].mp_lite_count = 0;
    }
    break;
}
```

遍历每个 die，对每个 die 中的 mp 和 mp_lite 配置寄存器偏移地址、cluster 掩码、中断向量和中断号信息

reg_off：寄存器偏移地址，每次循环增加 0x100000（即 1MB），表示每个 MP 的寄存器区域在地址空间中连续且间隔 1MB

cluster_mask = 0xFF：cluster 掩码，0xFF（二进制 11111111）表示该 MP 管理 8 个集群（每一位对应一个 cluster 的使能）

msi_vector = vector：MSI 中断向量，每次循环 vector 加 1，为每个 MP 分配独立的中断向量（用于消息信号中断）

irq = -1：中断号，初始设为 -1 表示暂未分配，后续由系统动态分配

初始化PCI和DMA的相关配置

初始化中断（IRQ）系统配置

```c
chip->isr_mode = zixia_isr_mode;// 配置中断服务（ISR）的工作模式，POLLING/MSI/MSIX
chip->irq_num = 12 + 2 + （DMA读写通道数）;// 中断向量数
chip->gather_irq_info[0].msi_vector = 6;// 初始化die0 gather_irq的中断向量为6
chip->gather_irq_info[0].irq = -1;//初始化中断号为-1
chip->gather_irq_info[1].msi_vector = 13;;// 初始化die1 gather_irq的中断向量为13
chip->gather_irq_info[1].irq = -1;//初始化中断号为-1
chip->pciedma_irq_vector_start_index = 14;//PCIe DMA 中断向量的起始索引为 14
```

初始化一些函数指针，调用这些函数实现功能

```c
 //chip ops
 chip->ops.hardware_init = zixia_chip_hardware_init;
 chip->ops.hardware_deinit = zixia_chip_hardware_deinit;
 chip->ops.pcie_pmu_init = zixia_chip_pcie_pmu_init;
 ...
```

#### 配置中断

​	hal_zixia_chip_irq_vector_alloc 函数根据不同的中断模式（irq_mode）配置信息

​	POLLING中断模式，中断向量数irq_num为0，不依赖硬件信号，通过驱动轮询读取寄存器判断中断

​	LEGACY中断模式，中断向量数irq_num为1（中断向量指不同中断源对应的中断），所有中断源共享一个中断号，触发中断后通过轮询判断中断源

​	MSI/MSI-X 中断模式，调用 pci_alloc_irq_vectors函数分配 chip->irq_num个中断向量，然后通过msi_vector_init调用zixia_chip_msi_vector_init函数设置中断号（遍历每个 die 的 mp、mplite ，polling 模式中断号都为 - 1，MSI 模式中断号都为中断向量索引为 0 的中断号，MSIX 模式中断号为中断向量索引对应的中断号）

​	中断汇聚gather_irq，避免某些中断频繁触发，给每个die的gather_irq分配中断号

​	每个die分别有一个中断源代表gather（die0 vector6，die1 vector13），当gather中断触发时会调用msi_isr处理函数，之后调用gather_irq处理函数，在gather_irq处理函数中先根据当前的gather中断源分配中断号，然后根据中断号调用对应的gather处理函数（先通过zixia_chip_register_gather_irq对gather分配中断号和处理函数，一共有289个gather）

```c
// Gather_irq register
if (chip->gather_irq_info[i].irq != -1) {
    zixia_chip_gather_irq_init(chip, i);
    if (zixia_chip_register_irq(chip, chip->gather_irq_info[i].irq, chip->gather_irq_info[i].msi_vector, (i == 0) ? \
        zixia_chip_die0_gather_irq_handler : zixia_chip_die1_gather_irq_handler, chip)) {
        if (i != 0) {
            zixia_chip_free_irq(chip, chip->gather_irq_info[0].irq, chip->gather_irq_info[0].msi_vector, chip);
        }
        hal_zixia_log_err(chip->dev, "Die%d gather_irq register failed", i);
        return -1;
    }
}
```

​	先通过zixia_chip_gather_irq_init初始化所有通道gather_irq参数

```c
zixia_chip_gather_irq_init	
	__gather_irq_mask_all(chip, die_id); //设置掩码屏蔽中断
	//初始化每个gather通道的gather_irq参数（289个）
    for(irq = 0; irq < IRQ_VEC__PCIE_EXT_IRQ_GATHERED__CNT; irq++)
        chip->gather_irq_arr[die_id][irq].irq = -1;
        chip->gather_irq_arr[die_id][irq].handler = NULL;
        chip->gather_irq_arr[die_id][irq].handle_args = NULL;
        chip->gather_irq_arr[die_id][irq].active = false;
```

​	通过zixia_chip_register_irq注册对应中断的中断处理函数msi_isr_routine

```c
zixia_chip_register_irq
    //获取中断号对应的中断向量索引
     index = irq -  pci_irq_vector(to_pci_dev(chip->dev), 0);
	 ...
      if(chip->msi_vector[index].active == false)
      	sprintf(chip->msi_vector[index].name, "zixia_chip_msix_%d", irq);
		//将gather_irq的中断号与处理函数绑定
		//注册gather_irq的中断处理函数zixia_chip_msi_isr_routine
      	request_irq(irq, zixia_chip_msi_isr_routine, 0, chip- >msi_vector[index].name, chip); 
	  	//配置gather_irq对应的中断向量、中断号、函数指针
	  	chip->msi_vector[index].msi_vector = msi_vector;
       	 chip->msi_vector[index].irq = irq;
       	 chip->msi_vector[index].handler = handler;
       	 chip->msi_vector[index].handle_args = handle_args;
       	 chip->msi_vector[index].active = true;
	 __msi_vector_unmask(chip, msi_vector) //配置msi_vector对应的中断mask，启用该中断
```

​	触发gather_irq后，调用中断处理函数zixia_chip_msi_isr_routine

```c
zixia_chip_msi_isr_routine	
    //获取中断号对应的中断向量索引
	index = irq-pci_irq_vector(to_pci_dev(chip->dev), 0);
	...
     //中断号符合且经过配置后
     if((chip->msi_vector[index].irq == irq) && (chip->msi_vector[index].active == true))
          //调用对应的函数指针
          if(chip->msi_vector[index].handler) {
                 chip->msi_vector[index].handler(irq, chip->msi_vector[index].handle_args);
          }
		  ... //清除寄存器相关状态
```

​	在中断处理函数中会调用函数指针zixia_chip_die0_gather_irq_hander，并分配gather的中断号，调用gather_irq对应的函数指针

```c
zixia_chip_die0_gather_irq_hander
	zixia_chip_gather_irq_hander
		irq = __gather_irq_stat_query(chip, die_id) //根据当前gather中断源范围分配中断号
    	 irq = irq + die_id * IRQ_VEC__PCIE_EXT_IRQ_GATHERED__CNT;
	     //如果该中断号对应的gather_irq参数符合条件
		 if((chip->gather_irq_arr[die_id][irq].active) && (chip->gather_irq_arr[die_id][irq].handler))
             //调用gather_irq参数对应的函数指针
             chip->gather_irq_arr[die_id][irq].handler(irq, chip->gather_irq_arr[die_id][irq].handle_args)
```

​	gather_irq参数对应的函数指针handler会在zixia_chip_register_gather_irq中进行配置，将中断号和处理函数分配给gather_irq_arr

```c
zixia_chip_register_gather_irq(chip, irq,handler,handle_args)
	chip->gather_irq_arr[die_id][irq].irq = irq; //分配中断号
    chip->gather_irq_arr[die_id][irq].handler = handler; //分配处理函数
    chip->gather_irq_arr[die_id][irq].handle_args = handle_args;//分配参数
    chip->gather_irq_arr[die_id][irq].active = true;
	__gather_irq_unmask() //解除对应中断源的mask
```

#### 硬件初始化

​	hal_zixia_chip_hardware_init函数进行 hardware 硬件初始化，对每个 die 的 mp 写入编号值

#### pciedma初始化

​	hal_zixia_chip_pciedma_init函数对 pciedma 进行初始化，配置pciedma信息，vram_reserved预留内存负责存储LLI，遍历写通道和读通道来配置LLI信息，通过zixia_pciedma_probe来

```c
hal_zixia_chip_pciedma_init(struct pci_dev *pdev)
    //配置pciedma信息，pciedma寄存器地址、lli_recycle、prefetch_len、通道数等
    pciedma->dev = &pdev
    pciedma->reg_base = chip->pcie_reg_bar.logical + chip->pciedma_reg_off
    pciedma->is_hdma = chip->is_hdma;
    pciedma->lli_recycle = chip->pciedma_lli_recycle;
	pciedma->prefetch_len = ...
    ...
    //vram_reserved预留内存32M负责存储LLI（描述符），每个LLI的size = 32M/通道数 = 1M
    ll_region_sz = chip->vram_reserved.size / (pciedma->wr_ch_cnt + pciedma->rd_ch_cnt);
	//遍历写通道配置wr_ch_cnt个LLI
	for(i=0; i<pciedma->wr_ch_cnt; i++)
         pciedma->ll_region_wr[i].paddr = chip->vram_reserved.remian_base; //第i个LLI物理地址
		pciedma->ll_region_wr[i].vaddr.io = ... //第i个LLI逻辑地址
         pciedma->ll_region_wr[i].sz = ... //第i个LLI的size
         chip->vram_reserved.remian_base += ll_region_sz; //更新剩余预留内存的物理地址
         chip->vram_reserved.remian_logical //更新剩余预留内存的逻辑地址
		chip->vram_reserved.remian_size -= //更新剩余预留内存的size
      //遍历读通道配置rd_ch_cnt个LLI
      for(i=0; i<pciedma->rd_ch_cnt; i++) 
          ...
      zixia_pciedma_probe(pciedma)
```

##### zixia_pciedma_probe

```c
zixia_pciedma_probe
	if (pciedma->is_hdma)
        hdma_core_register(pciedma);
```

















