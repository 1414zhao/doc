

异步分配内存，从内存池中分配

## memPoolAlloc

​	先获得一个当前stream的内存池mempool

```c
memPoolAlloc
    //timeStamp负责fence同步
    timeStamp s = { std::set<CUstream>(), nullptr };
	//轮询freecollect中的内存块，找到合适的内存块
    mapMemNode = allocCollectFindMemory(...)
    //没有合适的空闲内存块
    if (mapMemNode == nullptr)
        //预留足够的堆虚拟地址，分配block，配置内存节点
        vmHeapAlloc(...，dptr，...)
        //根据逻辑地址从gpgpuPLS.globalPtrMap中找到对应的内存节点mapMemNode
        mapMemNode = findNodeWithAddress(nullptr, (uint64_t)*dptr);
        cmd.halQueue = halQueue;
        memPoolAllocParameter->memPool = pool;
        memPoolAllocParameter->dptr = *dptr;
	    //memPoolAlloc回调函数
        data->fn = memPoolAllocCallBack;
        data->data = memPoolAllocParameter;
        eCallback->callback = callBackHelper;
        eCallback->parameter = data;
	   //在round_robin调度中调用回调函数
	   launchCommand(halQueue, &cmd)
   else
       *dptr = mapMemNode->memNode->gpuLogical;
    //将当前stream插入timeStamp的streamset
	s.streamSet.insert(hStream);
    //将mapMemNode插入到mempool的busyCollect
    allocCollectAddMemory(pool->busyCollect, mapMemNode, s);
		//插入到busyCollect->memoryCollect中
		collect->memoryCollect->insert({ {memNode->requestSize, memNode}, stamp });
		//更新busyCollect->totalSize
		collect->totalSize += memNode->requestSize;
	pool->attr.attrUsedMemCurrent = pool->busyCollect->totalSize;
    pool->attr.attrUsedMemHigh = pool->busyCollect->maxTotalSize;
	memoryPoolRetain(pool);
```

### allocCollectFindMemory

​	freecollect->memoryCollect是一个map结构，map<pair<size， mapMemoryNode*>, timeStamp>

​	轮询freecollect中的内存块，找到合适的内存块，返回对应的内存节点

```c
allocCollectFindMemory(pool->freecollect,size,stream)
    //从freecollect中取出第一个大于size的内存块
    start = freecollect->memoryCollect->lower_bound({size, nullptr});
	//轮询freecollect中的内存块，找到合适的内存块
	for (it = start; it != freecollect->memoryCollect->end(); it++)
        //如果内存块的size大于1.25size，返回null，避免浪费
        if (it->first.first > (size / 8.0) * 10.0)
            return null;
		   //通过fence同步，看是否空闲，如果空闲返回内存块的node，否则取下一个内存块
		   if (isSafeReuse(it->second, stream, opportunistc))
               	 //返回内存块对应的memnode
				memory = it->first.second;
				//从freecollect减去size
			  	freecollect->totalSize -= size;
				//从freecollect中删除当前内存块
				it = freecollect->memoryCollect->erase(it);
				break;
			else
                 it++
	return it
```

#### isSafeReuse

​	通过fence进行同步，看fence值和期望值是否相等，不相等说明正在使用

```c
isSafeReuse
    //当前stream是否在streamset中
	if (stamp.streamSet.find(stream) != stamp.streamSet.end())
        return true;
	else
         //fence的cpu地址
		data = getFenceCPULogical(stamp.fence, processor);
		//fence的期望值
		val  = *stamp.fence->pExpectedVal;
		//将GPU中的fence值更新到cpu
	return *data == val;
```

### vmHeapAlloc

从堆中分配出block

获得堆对应的起始逻辑地址，根据size分配block（根据block分配对应chunk的物理内存），获得对应的block逻辑地址，配置node参数，最后将mapnode参数插入到gpgpuPLS.globalPtrMap中

```c
vmHeapAlloc(pool->heap,size)
    //如果堆的起始地址为0
    if (heap->vaBaseAddres == 0)
        //预留足够大的堆虚拟地址，映射出heap->vaSize大小的内存，返回一个对应的gpu逻辑地址vaBaseAddres
        reserveVirtualMemory(...)
        heap->vaBaseAddres = tempNode->gpuLogical;
    //分配对应的block
    block = vmHeapAllocBlock(...)
    if (block)
    {
        //block对应的逻辑地址
        *ptr = block->offset + heap->vaBaseAddres;
    }
    //配置node节点参数
    node->gpuLogical = *ptr;//block对应的逻辑地址
    node->cpuLogical = *ptr;
    node->processor = DEVICE(heap->deviceId)->processor;
    node->size = size;
    //node作为mapNode参数
    mapNode->memNode = node;
    mapNode->userData = block;
	...
    //将mapNode插入到gpgpuPLS.globalPtrMap中 all mapMemNode will be insert global map
	insertNodeWithAddress(...)
```

#### vmHeapAllocBlock

分配block  walker->size =1G  heap->blockSize=512  

将size与blocksize对齐，遍历block，找到最接近alignSize的block bestFit，对于和size相等的block（将这个块从freelist中移除，将这个块添加到busylist，从freeSize减去alignSize，对当前block对应的chunk分配内存，最后返回当前block），对于最接近alignSize的bestFit（将alignSize大小的block作为newBlock，将newBlock添加到busylist，对当前block对应的chunk分配内存，最后返回当前block）

```c
vmHeapAllocBlock(heap,size)
    walker = heap->freeList
    bestFit = nullptr;
    //将size与blocksize（512字节）对齐
    alignSize = drvAlignUp(size, heap->blockSize);
    //遍历block
	while (walker)
        //块size大于对齐后的size && （最佳块未找到 || 块size小于最佳块size），更新bestFit
        //bestFit越接近alignSize越好，避免浪费
        if (walker->size > alignSize && (bestFit == nullptr || walker->size < bestFit->size))
            bestFit = walker;
		else if (walker->size == alignSize) //块size正好和alignSize相同
            //将这个块从freelist中移除
            detachBlockFromList()
            //将这个块添加到busylist
            insertBlockIntoList()
            //从freeSize减去alignSize
            heap->freeSize -= alignSize;
            //对当前block对应的chunk分配内存
		   if (mapImmediately && chunkMapPaMem(heap, walker->offset, walker->size))
               return null
            //返回block
            return walker;
        walker = walker->next;
	//bestFit是大于alignSize的block
	if (bestFit != nullptr)
        //alignSize大小的block作为newBlock
        newBlock->vaOwner = bestFit->vaOwner;
    	newBlock->size = alignSize;
    	newBlock->offset = bestFit->offset;
		//更新bestFit的offset和size（heap->freeList）
	    bestFit->offset += alignSize;
    	bestFit->size -= alignSize;
        //将newBlock添加到busylist
        insertBlockIntoList()
        //对当前block对应的chunk分配内存
        if (mapImmediately && chunkMapPaMem(heap, newblock->offset, newblock->size))
               return null
        //返回block
        return newBlock;
```

##### chunkMapPaMem

分配chunk  chunkSize=32M

对当前block跨越的chunk（start_chunk和end_chunk）进行遍历，至少分配一个chunk（对于小于32M的block，分配一个chunk）

对于未分配内存的chunk，分配chunk->size大小的物理内存，对内存进行映射，对于其他设备（将节点物理地址map到其他设备，给其他设备分配逻辑地址）

多个chunk的逻辑地址是连续的，每个node对应一个chunk地址

```c
chunkMapPaMem(heap,block->offset,block->size)
	//得到当前块对应的startchunk和endchunk
	startChunk = offset / heap->chunkSize
	endChunk = drvAlignUp(offset + size, heap->chunkSize) / heap->chunkSize
    //遍历startchunk和endchunk之间的chunk
    for (int i = startChunk; i < endChunk; i++)
        //当前chunk未分配物理地址
        if(heap->mapPaStatus->at(i).first == false)
            //分配chunk->size大小的物理内存，返回内存节点memHandle
            allocPhysicalMemory(...)
            node->handle = memHandle;
		   //更新chunk的gpu地址（vaBaseAddres堆的基地址）
            node->gpuLogical = heap->vaBaseAddres + i * heap->chunkSize;
		   //配置map参数
		   info.u.mapInfo.address = node->gpuLogical;
            info.u.mapInfo.size = heap->chunkSize;
		   ...
            //map出节点的cpuLogical
            relockVirtualMemory(...)
            //将节点地址map到其他设备，给其他设备分配逻辑地址
            for (auto& it : *heap->memoryPool->accessMap)
                 relockVirtualMemory(...)
            //更新heap的mapsize、mapstatus
            heap->mappedSize += heap->chunkSize;
            heap->mapPaStatus->at(i).first = true;
		   heap->mapPaStatus->at(i).second = node;
```

### memPoolAllocCallBack

mempoolalloc回调函数

mapImmediately为true时，会直接在vmHeapAlloc中分配block内存，不在回调函数中分配

```c
//根据逻辑地址从gpgpuPLS.globalPtrMap中找到对应的内存节点
mapMemNode = (mapMemNode_t*)findNodeWithAddress(nullptr, (uint64_t)dptr);
//内存节点中的block
block = (vmBlock*)mapMemNode->userData;
//分配block的chunk内存
if (chunkMapPaMem(pool->heap, block->offset, block->size))
    return null
//配置pool参数
pool->mappedSize = pool->heap->mappedSize;
pool->maxTotalSize = drvMAX(pool->maxTotalSize, pool->mappedSize);
```


