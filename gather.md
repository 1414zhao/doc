## gather注册

​	gpcl/hal/os/linux/kernel/platform/zixia/soc/chip/msi_vector.c

​	MSIX模式每个die分别有一个中断向量代表gather(6\13)，分配中断号

```c
if(chip->gather_irq_info[i].msi_vector >= 0) {
    if(chip->isr_mode == ZIXIA_ISR_POLLING) {
        chip->gather_irq_info[i].irq = -1;
    } else if(chip->isr_mode == ZIXIA_ISR_LEGACY) {
        chip->gather_irq_info[i].irq = (to_pci_dev(chip->dev))->irq;
    } else if(chip->isr_mode == ZIXIA_ISR_MSI) {
        chip->gather_irq_info[i].irq = pci_irq_vector(to_pci_dev(chip->dev), 0);
    } else if(chip->isr_mode == ZIXIA_ISR_MSIX) {
        chip->gather_irq_info[i].irq = pci_irq_vector(to_pci_dev(chip->dev), chip->gather_irq_info[i].msi_vector);
    } else {
        chip->gather_irq_info[i].irq = -1;
    }
}
```

​	注册gather中断，先通过zixia_chip_gather_irq_init初始化所有通道gather_irq参数(设置mask)

​	通过zixia_chip_register_irq注册gather irq的中断处理函数，当gather中断触发时会调用处理函数，在处理函数中先根据L1_irq[index].stat获得当前gather中断的中断号，然后根据中断号调用对应的handle

```c
if(chip->gather_irq_info[i].irq != -1) {
    zixia_chip_gather_irq_init(chip, i);
    if(zixia_chip_register_irq(chip, chip->gather_irq_info[i].irq, chip->gather_irq_info[i].msi_vector, (i == 0) ? zixia_chip_die0_gather_irq_handler : zixia_chip_die1_gather_handler, chip)) {
        if(i != 0) {
            zixia_chip_free_irq(chip, chip->gather_irq_info[0].irq, chip->gather_irq_info[0].msi_vector, chip);
        }
        hal_zixia_log_err(chip->dev, "Die%d gather_irq register failed", i);
        return -1;
    }
}
```

## ZIXIA_TEST_GATHER_IRQ_REGISTER

​	unmask指定中断

```c
zixia_chip_register_gather_irq
    //指定的中断已经注册，return函数
    if((chip->gather_irq_arr[die_id][irq].irq == irq) && (chip->gather_irq_arr[die_id][irq].active == true))
        if((chip->gather_irq_arr[die_id][irq].handler == handler) && (chip->gather_irq_arr[die_id][irq].handle_args == handle_args))
            return -1;
	//配置gather_irq_arr参数
	chip->gather_irq_arr[die_id][irq].irq = irq;
    chip->gather_irq_arr[die_id][irq].handler = handler;
    chip->gather_irq_arr[die_id][irq].hand
    //指定irq unmask
    __gather_irq_unmask(chip, die_id, irq);
```

​	L1一共10组寄存器，每组寄存器表示32位中断

​	L2一组寄存器，每一位表示L1的组，32*32=1024

```c
__gather_irq_unmask
    //通过bar4映射PCIE_EXT寄存器的基址
	regs = chip->sys_reg_bar.logical + GRP_DIE0_TOP_CTRL_CFGS_PCIE_EXT_IRQ_REGS_BASE
    //中断号与32取商作为L1寄存器组的index
    index = irq / 32;
	//读取该index寄存器的mask
    mask = readl((volatile void __iomem *)&regs->l1_irq[index].mask);
	//更新mask(该中断号unmask)
    mask &= ~(1 << (irq % 32));
	//写入寄存器
    writel(mask, (volatile void __iomem *)&regs->l1_irq[index].mask);

	//读取L2寄存器的mask
	mask = readl((volatile void __iomem *)&regs->l2_irq.mask);
	//更新mask(该index unmask)
    mask &= ~(1 << index);
	//写入寄存器
    writel(mask, (volatile void __iomem *)&regs->l2_irq.mask);
```

## ZIXIA_TEST_GATHER_IRQ_UNREGISTER

​	mask指定中断

```c
zixia_chip_free_gather_irq
    //指定的中断已经注册
     if((chip->gather_irq_arr[die_id][irq].irq == irq) && (chip->gather_irq_arr[die_id][irq].active == true))
         //配置参数
         chip->gather_irq_arr[die_id][irq].irq = -1
         chip->gather_irq_arr[die_id][irq].handler = NULL
         ...
    //指定irq mask
	__gather_irq_mask(chip, die_id, irq);
```

​	更新L1指定index中的mask，如果该index的L1都mask，更新L2的mask

```c
__gather_irq_mask
    //通过bar4映射PCIE_EXT寄存器的基址
	regs = chip->sys_reg_bar.logical + GRP_DIE0_TOP_CTRL_CFGS_PCIE_EXT_IRQ_REGS_BASE
    //更新L1指定index中的mask
    index = irq / 32;
    mask = readl((volatile void __iomem *)&regs->l1_irq[index].mask);
    mask |= 1 << (irq % 32);
    writel(mask, (volatile void __iomem *)&regs->l1_irq[index].mask);
	//如果该index的L1都mask，更新L2的mask
    if(mask == 0xffffffff) 
        mask = readl((volatile void __iomem *)&regs->l2_irq.mask);
        mask |= 1 << index;
        writel(mask, (volatile void __iomem *)&regs->l2_irq.mask);
```

## ZIXIA_TEST_GATHER_IRQ_SET

​	触发指定中断

```c
__gather_irq_set
    //通过bar4映射PCIE_EXT寄存器的基址
	regs = chip->sys_reg_bar.logical + GRP_DIE0_TOP_CTRL_CFGS_PCIE_EXT_IRQ_REGS_BASE
    //中断号与32取商作为L1寄存器组的index
    index = irq / 32;
	//读取该index寄存器的set
    set = readl((volatile void __iomem *)&regs->l1_irq[index].set);
	//更新set
    set |= 1 << (irq % 32);
    writel(set, (volatile void __iomem *)&regs->l1_irq[index].set);
```

## ZIXIA_TEST_GATHER_IRQ_DUMP_REGS

```c
__dump_gather_irq_regs
    //输出参数regs->l1_irq[index].mask、regs->l1_irq[index].set
```

## ZIXIA_TEST_GATHER_IRQ_UNMASK

​	创建一个线程，在线程中unmask全部中断，随机触发中断

```c
test_gather_unmask_all
    //创建一个线程
    irq_task = kthread_create(unmask_all_and_irq, chip, "unmask_and_irq");
```

```c
unmask_all_and_irq
    //注册IRQ_MAX个中断
	for(irq = 0; irq < IRQ_MAX ; irq ++) {
        zixia_chip_register_gather_irq(chip, irq, zixia_test_gather_irq_hander, data);
    }
	for(i = 0; i < TEST_TIMES; i ++)
        get_random_bytes( &random, sizeof(random));
		//随机中断号
        irq = random % IRQ_MAX;
	    printk(KERN_ERR" set irq %d occurred \n", irq);
		//触发随机中断
        zixia_chip_gather_irq_set(data, irq);
        msleep(200);
```

## ZIXIA_TEST_GATHER_IRQ_MASK

​	创建一个线程，先在线程中mask全部中断，再随机触发中断

```c
test_gather_mask_all
	irq_task = kthread_create(mask_all_and_irq, chip, "mask_and_irq");
```

```c
mask_all_and_irq
    //配置gather_irq_arr参数
    zixia_chip_gather_irq_init(chip, die_id);
	//mask全部中断
	zixia_chip_gather_irq_mask_all(chip);
	for(i = 0; i < TEST_TIMES; i ++) 
        get_random_bytes( &random, sizeof(random));
        irq = random % IRQ_MAX;
        printk(KERN_ERR" set irq %d occurred \n", irq);
	   //触发随机中断
        zixia_chip_gather_irq_set(data, irq);
        msleep(100); 
	    //人为handle打印信息
     	zixia_test_gather_irq_hander(irq, data);
```

​	zixia_chip_gather_irq_mask_all调用__gather_irq_mask_all

```c
__gather_irq_mask_all
	mask = 0xffffffff;
	//遍历10个L1寄存器配置mask
    for(index = 0; index < 10; index ++) 
        writel(mask, (volatile void __iomem *)&regs->l1_irq[index].mask);
	//配置L2寄存器
	writel(mask, (volatile void __iomem *)&regs->l2_irq.mask);
```









