# bio layer
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
