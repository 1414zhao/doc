## DMA初始化

### 配置通道的LLI信息

​	hal_zixia_chip_pciedma_init函数对 pciedma 进行初始化，配置pciedma信息，vram_reserved预留内存负责存储LLI区域（每个区域包括多个LLI，负责保存传输信息），遍历写通道和读通道来配置LLI区域信息（包括地址、size），每个通道对应一个LLI区域，通过zixia_pciedma_probe来初始化DMA配置

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
		pciedma->ll_region_wr[i].vaddr.io = ... //第i个LLI区域逻辑地址
         pciedma->ll_region_wr[i].sz = ... //第i个LLI区域的size
         chip->vram_reserved.remian_base += ll_region_sz; //更新剩余预留内存的物理地址
         chip->vram_reserved.remian_logical //更新剩余预留内存的逻辑地址
		 chip->vram_reserved.remian_size -= //更新剩余预留内存的size
      //遍历读通道配置rd_ch_cnt个LLI
      for(i=0; i<pciedma->rd_ch_cnt; i++) 
          ...
      zixia_pciedma_probe(pciedma)
```

#### zixia_pciedma_probe

​	为每个通道分配chan结构体，初始化通道自旋锁，关闭DMA，初始化通道中断，绑定中断处理函数，初始化通道配置，包括chan状态、irq、任务队列，配置初始DMA参数，开启DMA

```c
zixia_pciedma_probe
	if (pciedma->is_hdma)
        hdma_core_register(pciedma);//将函数指针绑定到pciedma->core
	pciedma->chan = devm_kcalloc(dev, pciedma->wr_ch_cnt + pciedma->rd_ch_cnt,
                                 sizeof(*pciedma->chan), GFP_KERNEL);//分配chan结构体
	spin_lock_init(&pciedma->wr_chan_alloc_lock);//初始化通道自旋锁
    spin_lock_init(&pciedma->rd_chan_alloc_lock);
	zixia_pciedma_core_off(pciedma);//关闭DMA
	zixia_pciedma_irq_request(pciedma); //初始化通道中断，绑定中断处理函数
	zixia_pciedma_channel_setup(pciedma);//初始化通道配置，包括chan状态、irq、任务队列
	zixia_pciedma_core_debugfs_on(pciedma);//配置初始DMA参数，开启DMA
```

##### zixia_pciedma_irq_request

​	所有通道共用一个中断，从中断向量获得对应的中断号irq，irq<0时运行内核线程等待中断，绑定中断处理函数interrupt_common（包括interrupt_write、interrupt_read）

​	每个通道对应一个中断，遍历通道，从中断向量获得对应的中断号irq，根据通道绑定中断处理函数interrupt_write或interrupt_read

```c
zixia_pciedma_irq_request
    if (pciedma->nr_irqs == 1) //单中断，所有通道一个中断
        irq = hal_zixia_chip_get_pciedma_irq_vector(...)//获得中断向量对应的中断号
        if (irq < 0)
            //irq<0时运行内核线程zixia_pciedma_isr_polling，等待中断
            pciedma->isr_thread = kthread_run(zixia_pciedma_isr_polling, ...);
		//绑定中断处理函数interrupt_common(包括interrupt_write、interrupt_read)
        request_irq(irq, zixia_pciedma_interrupt_common,
                  IRQF_SHARED, pciedma->irq[0].name, &pciedma->irq[0]);
	else					//多中断，每个通道对应一个中断
        for (i = 0; i < (pciedma->nr_irqs); i++) //遍历通道
            irq = hal_zixia_chip_get_pciedma_irq_vector(...)//获得中断向量对应的中断号
            if (irq < 0)
                goto err_irq_free;
            //根据通道绑定中断处理函数interrupt_write或interrupt_read
            request_irq(irq,(i < pciedma->wr_ch_cnt) ?zixia_pciedma_interrupt_write :
            			zixia_pciedma_interrupt_read,...
```

##### zixia_pciedma_interrupt_write

​	调用函数指针handle_int，调用hdma_core_handle_int函数，遍历通道的中断mask，从对应通道取出状态信息，从状态信息判断watermask中断，清除中断标志，调用watermask函数指针；判断done中断，清除中断标志，调用done函数指针；判断abort中断，清除中断标志，调用abort函数指针

```c
hdma_core_handle_int(pciedma_irq, dir,done, watermask, abort)
	if (dir == ZIXIA_PCIEDMA_DIR_WRITE)
        total = pciedma->wr_ch_cnt;
        off = 0;
        mask = pciedma_irq->wr_mask; //写通道中断mask
	else
        total = pciedma->rd_ch_cnt;
        off = pciedma->wr_ch_cnt;
        mask = pciedma_irq->rd_mask; //读通道中断mask
	//遍历通道的中断mask
	for_each_set_bit(pos, &mask, total)
        chan = &pciedma->chan[pos + off];//获取通道结构体
		val = hdma_core_status_int(chan);//从对应通道取出状态信息
		if (FIELD_GET(HDMA_WATERMASK_INT_MASK, val))//判断watermask中断
            hdma_core_clear_watermask_int(chan);	//清除中断标志
		   watermask(chan);						  //调用watermask函数指针
		if (FIELD_GET(HDMA_STOP_INT_MASK, val)) //判断done中断
            hdma_core_clear_done_int(chan);		//清除中断标志
		   done(chan);						 //调用done函数指针
		if (FIELD_GET(HDMA_ABORT_INT_MASK, val)) //判断abort中断
            hdma_core_clear_abort_int(chan);	 //清除中断标志
		   abort(chan);						  //调用abort函数指针
```

###### watermask中断

​	当占用的LLI数量达到watermask后，触发watermask中断，取得下一个描述符，如果仍有chunk，从chunk取出数据块，将数据块信息传进第i个LLI（重新从i=0的LLI开始），继续DMA传输

```c
zixia_pciedma_watermask_interrupt
    spin_lock_irqsave(&chan->vc.lock, flags); //chan->vc自旋锁
	vd = vchan_next_desc(&chan->vc);		//取得下一个描述符
	if (vd) 
        desc = vd2zixia_pciedma_desc(vd);	//描述符转换
        if (desc->chunks_alloc) 			//仍有chunk
            zixia_pciedma_start_transfer(chan);//从chunk取出数据块，将数据块信息传进LLI，继续DMA传输
 	spin_unlock_irqrestore(&chan->vc.lock, flags);
```

###### done中断

​	传输完成后调用done中断，取得下一个描述符

​	chan的request为ZIXIA_PCIEDMA_REQ_NONE时，传输结束时，删除描述符节点，使用vchan_cookie_complete函数标记DMA Cookie完成，通过tasklet_schedule调用tasklet的处理函数，更新chan状态

​	chan的request为ZIXIA_PCIEDMA_REQ_STOP时，删除描述符节点，通过vchan_cookie_complete函数标记DMA Cookie完成，调用tasklet的处理函数，更新chan状态

​	chan的request为ZIXIA_PCIEDMA_REQ_PAUSE时，更新chan状态

```c
zixia_pciedma_done_interrupt
    spin_lock_irqsave(&chan->vc.lock, flags);//chan->vc自旋锁
    vd = vchan_next_desc(&chan->vc);		//取得下一个描述符
	if (vd)
        switch (chan->request)
            case ZIXIA_PCIEDMA_REQ_NONE:
			    desc = vd2zixia_pciedma_desc(vd); //描述符转换
				if (!desc->chunks_alloc)	//没有chunk，传输结束
                  	 list_del(&vd->node);	 //删除描述符节点
					vchan_cookie_complete(vd); //标记DMA Cookie完成，通过tasklet_schedule调用任务队列的处理函数
				chan->status = zixia_pciedma_start_transfer(chan) ?ZIXIA_PCIEDMA_ST_BUSY:ZIXIA_PCIEDMA_ST_IDLE; //根据zixia_pciedma_start_transfer更新chan状态
		   case ZIXIA_PCIEDMA_REQ_STOP:
			    list_del(&vd->node);		//删除描述符节点
                 vchan_cookie_complete(vd);	  //标记DMA Cookie完成，tasklet_schedule
                 chan->request = ZIXIA_PCIEDMA_REQ_NONE //更新chan状态 
                 chan->status = ZIXIA_PCIEDMA_ST_IDLE;
		   case ZIXIA_PCIEDMA_REQ_PAUSE:
			    chan->request = ZIXIA_PCIEDMA_REQ_NONE; //更新chan状态
			    chan->status = ZIXIA_PCIEDMA_ST_PAUSE;
```

###### abort中断

```c
zixia_pciedma_abort_interrupt
    spin_lock_irqsave(&chan->vc.lock, flags);//chan->vc自旋锁
	vd = vchan_next_desc(&chan->vc); //取出下一个描述符
    if (vd) 
        list_del(&vd->node); //删除描述符节点
        vchan_cookie_complete(vd);//标记DMA Cookie完成，tasklet_schedule调用任务队列处理函数
	spin_unlock_irqrestore(&chan->vc.lock, flags);
	chan->request = ZIXIA_PCIEDMA_REQ_NON	//更新chan状态
	chan->status = ZIXIA_PCIEDMA_ST_IDLE;
	zixia_pciedma_isr_thread_stop(chan->pciedma);
```

##### zixia_pciedma_channel_setup

​	遍历所有通道，初始化chan状态，配置通道的irq，配置通道的中断mask，通过tasklet_setup初始化tasklet

​	设置DMA通道的回调函数，最后注册DMA设备

```c
zixia_pciedma_channel_setup
    //遍历所有通道
    for (i = 0; i < ch_cnt; i++)
        chan->request = ZIXIA_PCIEDMA_REQ_NONE;//初始化chan状态
	    chan->status = ZIXIA_PCIEDMA_ST_IDLE;
		if (chan->dir == ZIXIA_PCIEDMA_DIR_WRITE) //配置最大的LLI数量
			chan->ll_max = (pciedma->ll_region_wr[chan->id].sz / ZIXIA_PCIEDMA_LL_SZ);
		else
              ...
          if (pciedma->nr_irqs == 1)  //配置通道的irq
    			irq = &pciedma->irq[0];
		  else 
              	 irq = &pciedma->irq[i];
		  if (chan->dir == ZIXIA_PCIEDMA_DIR_WRITE) //配置通道的中断mask
              	  irq->wr_mask |= BIT(chan->id);
		  else
                  ...
           vchan_init(&chan->vc, dma); //初始化vc
				spin_lock_init(&vc->lock);	//初始化vc自旋锁
		  		dma_cookie_init(&vc->chan)	//初始化DMA Cookie
				...						  //初始化各种链表头
             	 tasklet_setup(&vc->task, vchan_complete); //初始化tasklet
	dma->device_caps = zixia_pciedma_device_caps;//设置DMA通道的回调函数
	.......
	dma_async_device_register(dma); 	//注册DMA设备
```

###### vchan_complete

​	tasklet处理函数，将tx中的回调函数指针和参数赋给cb结构体，之后传进回调函数参数，执行回调函数

```c
vchan_complete(struct tasklet_struct *t)
    spin_lock_irq(&vc->lock);	//开自旋锁
	dmaengine_desc_get_callback(&vd->tx, &cb);//将tx中的回调函数指针和参数赋给cb结构体
	spin_unlock_irq(&vc->lock);
	dmaengine_desc_callback_invoke(&cb, &vd->tx_result);//传进回调函数参数，调用回调函数
```

## _CopyBuffer函数

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

### ops的函数指针msschedDmaXferSubmit

​	调用_msschedDmaXferSubmit函数

​	通过GPU的逻辑地址找到所属的内存节点，通过对应的CPU物理地址获得GPU物理地址（设备物理地址），通过hal_zixia_dma_map将host地址映射为DMA地址，配置transfer_info结构体，通过hal_zixia_dma_transfer函数开启DMA传输

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

### hal_zixia_dma_map函数

​	SG表将物理上离散的物理页描述为连续的传输区间，便于DMA访问（每个sg表项对应一个离散的物理内存，需要DMA传输的内存由多个离散的物理内存组成）

​	根据host内存地址构建SG表，地址为KMALLOC时，分配bulk_cnt个sg_table结构体，遍历bulk_cnt个sg表项，基于每个子数据块bulk_size分配物理页，得到物理页的偏移量，将物理页与SG表进行关联

​	地址为USER时，分配数组内存，获取进程的pid结构，从而获取进程的task结构，获取进程的内存描述符，将进程用户态对应的物理页固定在内存，防止被内核换出，基于固定的物理页构建SG表，分配npages个sg表项

​	最后将SG表映射为DMA地址

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
			bulk_cnt =  (size + (bulk_size - 1)) / bulk_size;
			sg_alloc_table(sgt, bulk_cnt, GFP_KERNEL); //分配bulk_cnt个sg_table结构体
			remain_size = size;
			for_each_sg(sgt->sgl, sg, sgt->orig_nents, i)//遍历bulk_cnt个sg表项
				page = alloc_pages(GFP_KERNEL, get_order(bulk_size));//分配连续物理页，返回page结构体
				addr = page_address(page);//物理页的逻辑地址
				addr += args.offset;	//地址加偏移量进行对齐？
  				page = virt_to_page(addr) //获得对应的page结构体
                 offset = offset_in_page(addr) //得到物理页的偏移量
                 sg_set_page(dmap->sgt.sgl, page, bulk_size, offset); //将物理页与SG表进行关联
		case AT_VMALLOC:
			...
		case AT_USER:
			dmap->npages = npages;
			kmalloc_array(dmap->npages, sizeof(struct page *), GFP_KERNEL);//分配数组内存
			find_get_pid(pid) //获取进程的pid结构
             pid_task(pid_st, PIDTYPE_PID) //获取进程的task结构
             get_task_mm(tsk)	//获取进程的内存描述符
             down_read(&mm->mmap_lock); //加读锁（可以多个线程获得锁）
			pin_user_pages_remote(...) //将进程用户态对应的物理页固定在内存，防止被内核换出
             up_read(&mm->mmap_lock) //解锁
			...
             sg_alloc_table_from_pages(...) //基于固定的物理页构建SG表，基于物理页分配npages个sg表项
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

### DMA单通道传输

​	zixia_dma_single_chan_transfer函数（单通道传输），获取DMA通道，初始化等待队列、工作队列，将DMA通道与外设关联，准备基于SG缓冲区的传输描述符，设置传输完成后的回调函数（tasklet中的处理函数会调用），提交传输描述符到DMA引擎，启动DMA传输，让进程在等待队列上睡眠来等待DMA传输完成，DMA传输完成后会触发中断，通过调用tasklet的处理函数来触发回调函数（中断下半部），在回调函数中将工作队列添加到内核线程，执行与工作队列绑定的处理函数，在处理函数中唤醒睡眠中的等待队列，释放DMA通道，解除DMA映射

```c
zixia_dma_single_chan_transfer(*pdev, *transfer_info)
    ...		//根据host内存地址构建SG表，将SG表映射为DMA地址
    chan = zixia_dma_chan_alloc(...) //获取DMA通道，找到传输size最小的通道作为DMA通道
            chip = pci_get_drvdata(pdev);
            pciedma = (struct zixia_pciedma *)chip->pciedma; //从pci结构体取出pciedma信息
            if(dir == HAL_ZIXIA_DMA_DIR_H2D) //H2D获取读通道数，读通道锁，读通道地址
                ch_cnt = pciedma->rd_ch_cnt;
                alloc_lock = &pciedma->rd_chan_alloc_lock;
                pchan = &pciedma->chan[pciedma->wr_ch_cnt];
            if(dir == HAL_ZIXIA_DMA_DIR_D2H) //D2H获取写通道数，写通道锁，写通道地址
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
	dmaengine_slave_config(chan, &config) //将config中的信息（外设地址）赋值到DMA通道参数
     dmaengine_prep_slave_sg(...) //为Slave 模式 DMA 通道准备基于SG缓冲区的传输描述符，DMA_PREP_INTERRUPT | DMA_CTRL_ACK标志，传输完成后触发中断，描述符自动复用
     tx->callback_result = dma_transfer_callback; //设置传输完成的tasklet回调函数
	tx->callback_param = dt;	//回调函数参数
	dt->time_start = ktime_get()
     dmaengine_submit(tx) //提交传输描述符到DMA引擎
     dma_async_issue_pending(chan) //启动DMA传输
     wait_event_interruptible_timeout(wait, done, msecs_to_jiffies(timeout))//让进程在等待队列上睡眠，dnoe为1时或到达超时时间唤醒(等待DMA传输完成)
```

#### dma_transfer_callback回调函数

​	tasklet的回调函数（属于中断下半部），传输完成时调用回调函数，计算传输时间，将工作队列添加到内核线程，调用dma_transfer_finish_work处理函数

```c
dma_transfer_callback(*data, *result)
    struct dma_transfer *dt = data
    dt->time_val = ktime_to_us(ktime_sub(ktime_get(), dt->time_start)) //计算传输时间
    schedule_work(&dt->work) //提交工作队列，调用处理函数
```

#### dma_transfer_finish_work处理函数

​	与工作队列绑定，同步DMA数据到CPU，当transfer_info设置回调函数时（多通道），运行回调函数，唤醒等待队列，释放DMA通道，解除DMA映射

```c
dma_transfer_finish_work
	if(dt->transfer_info.dir == HAL_ZIXIA_DMA_DIR_D2H) 
    	dma_sync_sg_for_cpu(...); //解决 CPU 缓存与 DMA 物理内存的一致性，同步DMA数据到CPU
	if(!dt->transfer_info.block) 
         if(dt->transfer_info.fn) 
             dt->transfer_info.fn(dt->transfer_info.fn_args, &dt->result);
     else if(dt->done && dt->wait) 
        *dt->done = true;
        wake_up(dt->wait);//将done设置为1，唤醒等待队列
	zixia_dma_chan_free(...) //释放DMA通道
     ...	//解除DMA映射
```

#### dmaengine_prep_slave_sg

​	为Slave 模式 DMA 通道准备基于SG缓冲区的传输描述符，调用zixia_pciedma_device_prep_slave_sg函数指针，调用zixia_pciedma_device_transfer函数，最后返回一个传输描述符

```c
zixia_pciedma_device_prep_slave_sg(dchan,sgl,len,direction,flags, void *context)
    //将传进的参数配置到xfer参数
	struct zixia_pciedma_transfer xfer;
    xfer.dchan = dchan;
    xfer.direction = direction;
    xfer.xfer.sg.sgl = sgl;
    xfer.xfer.sg.len = len;
    xfer.flags = flags;
    xfer.type = ZIXIA_PCIEDMA_XFER_SCATTER_GATHER;
	//调用zixia_pciedma_device_transfer函数
	return zixia_pciedma_device_transfer(&xfer);
```

##### zixia_pciedma_device_transfer

​	分配chunk节点，将节点插入chunk链表，遍历sg表项，分配burst节点，将burst节点插入chunk->burst->list链表，每个burst节点对应一个sg表项（DMA传输内存中的离散内存块），配置burst参数（size、外设地址、DMA地址），初始化一个传输描述符

```c
zixia_pciedma_device_transfer
	desc = zixia_pciedma_alloc_desc(chan);//分配desc，初始化chunk链表头
	chunk = zixia_pciedma_alloc_chunk(desc);//分配chunk节点
	//获得config中的外设GPU地址
	src_addr = chan->config.src_addr;
	dst_addr = chan->config.dst_addr;
	if (xfer->type == ZIXIA_PCIEDMA_XFER_SCATTER_GATHER) 
        cnt = xfer->xfer.sg.len; //sg表项数
        sg = xfer->xfer.sg.sgl; //sg表
	//遍历sg表项，每块内存对应一个sg表项
	for (i = 0; i < cnt; i++)
        //分配burst节点，将burst节点插入chunk->burst->list链表
        burst = zixia_pciedma_alloc_burst(chunk);
	    burst->sz = sg_dma_len(sg); //sg表项对应的size
	    chunk->ll_region.sz += burst->sz; //更新LLI区域size
		desc->alloc_sz += burst->sz; //更新alloc_sz
		if (dir == DMA_DEV_TO_MEM)
            burst->sar = src_addr; //外设地址
		   src_addr += sg_dma_len(sg); //更新外设地址
	        burst->dar = sg_dma_address(sg); //sg表项对应的DMA地址
		if (dir == DMA_MEM_TO_DEV)
            ...
         sg = sg_next(sg); //取出下一个sg表项
	memcpy((void *)&desc->last_burst, burst, sizeof(*burst))
	vchan_tx_prep(&chan->vc, &desc->vd, xfer->flags); //初始化一个传输描述符
```

###### zixia_pciedma_alloc_chunk

​	分配chunk节点，配置chunk的LLI参数（包括对应LLI区域的物理地址和逻辑地址），将节点插入chunk链表，每个chunk节点对应一个LLI区域，每个chunk对应一个DMA传输的内存

```c
zixia_pciedma_alloc_chunk
    chunk = kzalloc(sizeof(*chunk), GFP_NOWAIT);//分配chunk节点
	INIT_LIST_HEAD(&chunk->list); //初始化链表头
	//配置chunk参数
	chunk->chan = 
     chunk->prefetch_len = 
     chunk->ll_max_align_prefetch =
    if (chan->dir == ZIXIA_PCIEDMA_DIR_WRITE)
        //配置chunk的LLI参数（包括对应LLI区域的物理地址和逻辑地址）
        chunk->ll_region.dev_paddr = pciedma->ll_region_wr[chan->id].dev_paddr;
        chunk->ll_region.paddr = pciedma->ll_region_wr[chan->id].paddr;
		chunk->ll_region.vaddr = pciedma->ll_region_wr[chan->id].vaddr;
	else
        ...
    if (desc->chunk)
        desc->chunks_alloc++; //chunks_alloc参数递增
        list_add_tail(&chunk->list, &desc->chunk->list);//节点插入chunk->list链表
	else
        /* List head */
        chunk->burst = NULL;
        desc->chunks_alloc = 0;
        desc->chunk = chunk;
```

#### dma_async_issue_pending函数

​	调用device_issue_pending函数指针，调用zixia_pciedma_start_transfer函数开始DMA传输，从描述符的数据块链表取出第一个数据块，调用zixia_pciedma_core_start函数开始传输 

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
                                     struct zixia_pciedma_chunk, list);//从链表取第一个数据块chunk
	zixia_pciedma_core_start(pciedma, child, true);//数据块信息传进LLI，开始DMA传输
	zixia_pciedma_free_burst(child); //遍历数据块的子数据块burst，从链表删除子数据块，释放子数据块内存
    list_del(&child->list); //从链表删除数据块chunk
    kfree(child); //释放数据块内存
```

##### zixia_pciedma_core_start函数

​	LLI区域是存储DMA传输参数的一块内存，DMA传输可以拆分成多个 LLI，用链表串联，硬件按顺序读取 LLI 自动完成批量传输（一个数据块chunk对应一个LLI区域，一个LLI区域包含多个LLI，每个LLI对应一个burst的传输参数）

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

###### hdma_core_write_chunk_lli_recycle

​	lli_recycle开启后DMA的LLI可以自动回收复用，传输完成后的LLI被回收到空闲链表，避免重复申请

​	watermask_index是空闲链表的阈值，小于阈值时自动申请LLI到空闲链表

​	prefetch_len预取长度，硬件一次取prefetch_len个LLI到缓存

​	从数据块链表遍历子数据块burst，当占用的LLI数量达到watermask，配置control的HDMA_LWIE 和HDMA_RWIE（通过中断补充LLI到空闲链表），将子数据块的传输信息写入chunk对应LLI区域的第i个LLI，占用的LLI数量递增，配置prefetch_len，配置control的HDMA_LLP（硬件自动遍历LLI），将第i个 LLI 的链表指针指向 LLI 区域的起始物理地址

```c
hdma_core_write_chunk_lli_recycle(struct zixia_pciedma_chunk *chunk)	
    upper = chunk->bursts_index ? false : true;
	//计算watermask_index（ll_max_align_prefetch最大预取描述符的1/2，与prefetch_len对齐）
    water_mask_index =prefetch_len*（chunk->ll_max_align_prefetch / 2 /prefetch_len）; 
	//从数据块遍历子数据块burst
    list_for_each_entry(child, &chunk->burst->list, list)
        //占用的LLI数量达到水位线或接近最大预取长度
        if(i == water_mask_index || i == (chunk->ll_max_align_prefetch - 2))
            control |= HDMA_LWIE | HDMA_RWIE; //开启中断，补充LLI到空闲链表
		else
            control &= ~(HDMA_LWIE | HDMA_RWIE);//不开启中断
		//将子数据块的control、size、src地址、dst地址写入LLI内存的第i个区域
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

###### hdma_core_write_chunk

​	未开启lli_recycle，从数据块链表遍历子数据块burst，将子数据块的传输信息写入chunk对应LLI区域的第i个LLI，配置prefetch_len，配置control的HDMA_LLP（硬件自动遍历LLI），将第i个 LLI 的链表指针指向 LLI 区域的起始物理地址

```c
hdma_core_write_chunk(struct zixia_pciedma_chunk *chunk)
    //从数据块遍历子数据块child
    list_for_each_entry(child, &chunk->burst->list, list)
    	//将子数据块的control、size、src地址、dst地址写入LLI内存，便于硬件传输
		hdma_write_ll_data(chunk, i++, control, child->sz,child->sar, child->dar);
	//将prefetch_len写入硬件
	control = ((chunk->prefetch_len > 0) ? (chunk->prefetch_len - 1) : 0) << 16;
	control |= HDMA_LLP;	//硬件自动遍历LLI
	//将第i个 LLI 的链表指针指向 LLI 区域的起始物理地址
	hdma_write_ll_link(chunk, i, control, chunk->ll_region.dev_paddr);
```

### DMA多通道传输

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
				tmp_transfer_info[i].fn = dma_multi_chan_callback;//回调函数作为transfer_info参数
				tmp_transfer_info[i].fn_args = dmct;
				...
          for(i = 0; i < multi_chan; i++)//进行multi_chan个单通道传输
              	zixia_dma_single_chan_transfer(pdev, &tmp_transfer_info[i]);
		  if (transfer_info->block) //阻塞等待
            	//加入等待队列进行睡眠等待所有通道的DMA传输完成
    			wait_event_interruptible_timeout(wait, done, msecs_to_jiffies(timeout));
```

#### dma_multi_chan_callback

​	多通道传输回调函数dma_multi_chan_callback，每个通道传输完成后通过tasklet的处理函数调用回调函数，回调函数调用工作队列处理函数，在工作队列处理函数中调用dma_multi_chan_callback函数指针，dmct->complete参数加1，当dmct->complete与multi_chan相同时，计算传输时间，唤醒等待队列

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







