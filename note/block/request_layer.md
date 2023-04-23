## request的merge

## request的完成
对于mq来说，blk_mq_complete_request 是中断处理函数。完成request的方式主要有以下几种：
1. 直接在中断上下文中通过调用驱动的.complete来完成
	a. REQ_HIPRI类型的request(polled io)
	b. 需要在本cpu完成的request，但是controller支持多hw_queue
2. 发送ipi让其他cpu(rq->mq_ctx->cpu)完成
	a. rq->q->queue_flags满足如下条件：
		- 首先支持同cpu group的cpu完成request
		- 其次目标cpu与当前cpu不同
		- 且强制要求必须在同一个cpu完成request, 或者目标cpu与本cpu不在一个cache domain
	c. 目标cpu处于online状态
3. 给本cpu发送软中断来完成
	a. 本应发ipi出去，但是目标cpu offline
	b. 需要在本cpu完成的request，且controller只支持单hw_queue

对于绝大部分仅支持单queue的controller，仅有一个 irq vector用于处理IO completion，通常这个irq affinity会设置为所有cpu，但是在绝大多数arch中，会指定一个cpu处理。

在这种情况下，单queue controller的io completion不要在中断上下文中完成，否则会导致较长的irq关闭的时间，从而降低io性能。

在complete函数中调用到blk_update_request会调用req_bio_endio
