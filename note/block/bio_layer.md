# bio layer

## 递归避免
像md和dm这种虚拟block设备的使用过程中经常会出现一个块设备的栈，其中的每个块设备经常会将bio稍作修改后传递给下一个块设备。如果没有特殊处理会造成溢出的情况。

通过在submit_bio_noacct中通过current->bio_list将bio进行保存避免了递归。

### 递归避免可能导致的死锁问题
generic_make_request（submit_bio_noacct）允许等待之前的bio返回。如果等待的bio在current->bio_list上，就会产生死锁。

一种case中，这种死锁与之前split使用的mempool有关。

为何split需要使用mempool呢。假设split不使用mempool，在bio split的时候需要新在内存中分配一个bio出来。那么如果出现了内存不够的情况，就需要回收内存。如果进行了脏页的写回，本身又需要通过block层，就可能出现问题。因此通过mempool进行bio的分配就可以避免这个问题。

但是mempool在不足的情况下就会需要等待之前的bio返回，也就可能产生之前提到的死锁。

一种解决上述问题方案是使用bioset来进行bio的分配。当bioset发现内存内存池比较紧张时，会使用rescuer线程同一个bioset中current->bio_list上的bio。但是这种方案有一些缺点：

1. 创建了大量几乎不用的线程
2. 死锁并不总是与内存分配相关，只是其中的一种case

因此在4.11中引入了一个新的方案(comit-id 79bd99596b7305ab08109a8bf44a6a4511dbf1cd)，通过修改generic_make_request（submit_bio_noacct），这种方法更通用，负载更低，但是对驱动有一些要求。

大致改动如下：

1. 每次提交一个新的bio时(__submit_bio_noacct)，将正在使用的栈进行重排，将不同设备的bio(更低层设备的bio（dm/md场景）)放在前面，相同设备的bio放在后面处理
2. To avoid deadlocks, drivers must never risk waiting for a request after submitting one to generic_make_request.  This includes never allocing from a mempool twice in the one call to a make_request_fn.
3. 旧逻辑中，driver会在loop中进行split，split后会继续进行split。新逻辑要求split后提交第一个bio，并直接返回。等待从栈中拿出bio后，如果需要split，再继续处理。

## bio split
什么时候可能需要split: (blk_bio_segment_spit)
- bi_vec数目超过max_segments (scatter gather io允许的最大段数)
- 累计sectors超过bio层允许的单次最大发送大小(max_sector_kb)
- 某个bi_vec跨过了一个page

在此基础上，判断是否要split: (bvec_split_segs)
- 当前bi_vec已经超过剩余单次允许发送大小，则需要split
- 计算能否将剩余长度塞进剩余的segment中,且不跨页
  - 可以，则不split (在哪儿塞？)
  - 不可以，split

如何split: (bio_split)
- 根据bio clone一个bio(split)出来 (bio_alloc_clone)
- 调整split和bio的bi_size和bi_sector
- split->bi_private = bio (bio_chain)
- 将bio压入栈中，等待split处理完之后再出栈(submit_bio_noacct->submit_bio_noacct_nocheck)
