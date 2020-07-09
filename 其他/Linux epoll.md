## epoll
Linux 内核 epoll相关代码位置 fs/eventpoll.c
供用户层调用的三个最常用函数
+ epoll_create
+ epoll_ctl
+ epoll_wait

### epoll_create
```c
SYSCALL_DEFINE1(epoll_create, int, size)
{
	if (size <= 0)
		return -EINVAL;

	return do_epoll_create(0);
}

static int do_epoll_create(int flags)
{
	int error, fd;
	struct eventpoll *ep = NULL;
	struct file *file;

    // 初始化eventpoll数据结构ep
	error = ep_alloc(&ep);

    // Linux 把一切视为文件,为了统一这里创建了一种新的文件系统格式'eventpoll',这样对ep的操作可以抽象为对文件系统的操作;
    // eventpoll_fops是该文件系统的一系列回调,便于内核在发生了特定事件时使用
	file = anon_inode_getfile("[eventpoll]", &eventpoll_fops, ep,
				 O_RDWR | (flags & O_CLOEXEC));
	ep->file = file;
	fd_install(fd, file);

	return fd;
}

static const struct file_operations eventpoll_fops = {
#ifdef CONFIG_PROC_FS
	.show_fdinfo	= ep_show_fdinfo,
#endif
	.release	= ep_eventpoll_release,
	.poll		= ep_eventpoll_poll,
	.llseek		= noop_llseek,
};
```

###  epoll_ctl
```c
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
		struct epoll_event __user *, event)
{
	int error;
	int full_check = 0;
	struct fd f, tf;
	struct eventpoll *ep;
	struct epitem *epi;
	struct epoll_event epds;
	struct eventpoll *tep = NULL;

	// 将用户层要ctl的event内容复制到内核层
	if (ep_op_has_event(op) &&
	    copy_from_user(&epds, event, sizeof(struct epoll_event)))
		goto error_return;


	// 这里f.file->private_data就是前面create阶段建立的eventpoll数据结构
	ep = f.file->private_data;

    // 从eventpoll中找到搜寻fd所代表的epitem
	epi = ep_find(ep, tf.file, fd);
	error = -EINVAL;

    // 根据操作码 决定这个fd 是加入 还是 删除还是 改变属性
	switch (op) {

        // 当操作是 ADD 时,把fd ,epds等事件相关内容insert进 eventpoll 结构
	case EPOLL_CTL_ADD:
		if (!epi) {
			epds.events |= EPOLLERR | EPOLLHUP;
			error = ep_insert(ep, &epds, tf.file, fd, full_check);
		} else
			error = -EEXIST;
		if (full_check)
			clear_tfile_check_list();
		break;
	case EPOLL_CTL_DEL:
		if (epi)
			error = ep_remove(ep, epi);
		else
			error = -ENOENT;
		break;
	case EPOLL_CTL_MOD:
		if (epi) {
			if (!(epi->event.events & EPOLLEXCLUSIVE)) {
				epds.events |= EPOLLERR | EPOLLHUP;
				error = ep_modify(ep, epi, &epds);
			}
		} else
			error = -ENOENT;
		break;
	}

}

static int ep_insert(struct eventpoll *ep, const struct epoll_event *event,
		     struct file *tfile, int fd, int full_check)
{
    struct epitem *epi;
	
	epq.epi = epi;
	init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);

	// 这里使用了一个回调ep_ptable_queue_proc,ep_item_poll将这个回调注册到监听的文件中,这样当该文件触发事件时,会调用ep_ptable_queue_proc,这样就实现了事件信息从硬件到eventpoll结构的传送
	revents = ep_item_poll(epi, &epq.pt, 1);

	// 将已经准备好的eventpoll_item插入eventpoll结构的红黑树中便于下次的查询
	ep_rbtree_insert(ep, epi);
}

static __poll_t ep_item_poll(const struct epitem *epi, poll_table *pt,
				 int depth)
{
	struct eventpoll *ep;
	bool locked;

	pt->_key = epi->event.events;
	if (!is_file_epoll(epi->ffd.file))
		return vfs_poll(epi->ffd.file, pt) & epi->event.events;

	ep = epi->ffd.file->private_data;

	// 这里poll_wait的作用是 调用pt(即前面的ep_ptable_ququq_proc函数);
	poll_wait(epi->ffd.file, &ep->poll_wait, pt);
	locked = pt && (pt->_qproc == ep_ptable_queue_proc);

	return ep_scan_ready_list(epi->ffd.file->private_data,
				  ep_read_events_proc, &depth, depth,
				  locked) & epi->event.events;
}

// 这个函数的作用是向关心的文件注册一个回调(这里是ep_poll_callback),当关心的文件产生某事件时,系统会回调,把事件内容返回到一个能让eventpoll机制能访问到的地方
static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,
				 poll_table *pt)
{
	struct epitem *epi = ep_item_from_epqueue(pt);
	struct eppoll_entry *pwq;

    // 这里whead是系统底层的事件内容,就是在这里系统把事件内容复制给了eventpoll结构的队列里,后面的epoll_wait就可以根据eventpoll结构里的事件信息来操作
	if (epi->nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {

		// ep_poll_callback这个回调 在队列被唤醒,且file有事件时触发
		init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);
		pwq->whead = whead;
		pwq->base = epi;

		// 将包括epitem的封装结构 pwq 注册到队列whead,当whead队列被唤醒时,会触发ep_poll_callback
		if (epi->event.events & EPOLLEXCLUSIVE)
			add_wait_queue_exclusive(whead, &pwq->wait);
		else
			add_wait_queue(whead, &pwq->wait);
		list_add_tail(&pwq->llink, &epi->pwqlist);
		epi->nwait++;
	} else {

	}
}

static int ep_poll_callback(wait_queue_entry_t *wait, unsigned mode, int sync, void *key)
{

	// 就是在此时这里,epi产生的事件内容,被放入到eventpoll结构的rdllist里,后面的epoll_wait就是从这个rdllist取内容返回给用户层
	if (!ep_is_linked(epi)) {
		list_add_tail(&epi->rdllink, &ep->rdllist);
		ep_pm_stay_awake_rcu(epi);
	}

}
```

### epoll_wait
```c
SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events,
		int, maxevents, int, timeout)
{
	return do_epoll_wait(epfd, events, maxevents, timeout);
}

static int do_epoll_wait(int epfd, struct epoll_event __user *events,
			 int maxevents, int timeout)
{
	int error;
	struct fd f;
	struct eventpoll *ep;

	// 取出epfd所代表的文件的内容
	f = fdget(epfd);

    // 取出eventpoll结构
	ep = f.file->private_data;

	// 这里取出所有准备就绪的事件,并把这些事件的信息拷贝到参数events中,这个events指向一块用户空间,完成调用后,用户层可以通过这块空间得到各个事件的信息,并做相应处理
	error = ep_poll(ep, events, maxevents, timeout);

}

static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
		   int maxevents, long timeout)
{
// ...
check_events:
    // ep_send_events()函数把内核层发生的事件信息传到events(用户层)指向的空间
	if (!res && eavail &&
	    !(res = ep_send_events(ep, events, maxevents)) && !timed_out)
		goto fetch_events;

	return res;
}

static int ep_send_events(struct eventpoll *ep,
			  struct epoll_event __user *events, int maxevents)
{
	struct ep_send_events_data esed;
	esed.maxevents = maxevents;
	esed.events = events;

    // 从函数名得知,从eventpoll结构扫ready_list,然后传给带有用户空间的esed;
	// 这里传入了一个回调 ep_send_events_proc;
	ep_scan_ready_list(ep, ep_send_events_proc, &esed, 0, false);
	return esed.res;
}

static __poll_t ep_scan_ready_list(struct eventpoll *ep,
			      __poll_t (*sproc)(struct eventpoll *,
					   struct list_head *, void *),
			      void *priv, int depth, bool ep_locked)
{
	__poll_t res;
	int pwake = 0;
	struct epitem *epi, *nepi;
	LIST_HEAD(txlist);

	// 将eventpoll结构中的rdllist(准备就绪的事件列表)转移到txlist里
	spin_lock_irq(&ep->wq.lock);
	list_splice_init(&ep->rdllist, &txlist);
	ep->ovflist = NULL;
	spin_unlock_irq(&ep->wq.lock);

	// 对txlist里的事件们回调前面传入的ep_send_events_proc
	res = (*sproc)(ep, &txlist, priv);

	spin_lock_irq(&ep->wq.lock);

	// 在执行前面的sproc回调时,eventpoll也可能会产生新的事件,需要在这里作处理,重新入队什么的
	for (nepi = ep->ovflist; (epi = nepi) != NULL;
	     nepi = epi->next, epi->next = EP_UNACTIVE_PTR) {
		if (!ep_is_linked(epi)) {
			list_add_tail(&epi->rdllink, &ep->rdllist);
			ep_pm_stay_awake(epi);
		}
	}
	ep->ovflist = EP_UNACTIVE_PTR;


	// 并不是read list里所有信息都会得到处理,有时调用方定义了一次循环处理的最大值等等,这时队列里的信息要重新接入rdllist,便于下次处理;
	list_splice(&txlist, &ep->rdllist);
	__pm_relax(ep->ws);

	// 这个list_empty函数.竟然是在list为空时返回false............
	if (!list_empty(&ep->rdllist)) {

		//如果eventpoll的ready list里的事件已经处理完了;
		if (waitqueue_active(&ep->wq))
			wake_up_locked(&ep->wq);
		if (waitqueue_active(&ep->poll_wait))
			pwake++;
	}
	if (pwake)

		// 唤醒eventpoll结构中的poll_wait队列
		ep_poll_safewake(&ep->poll_wait);

	return res;
}
```

### Linux 怎么把事件让eventpoll结构知晓
+ epoll_create初始化ep;
+ epoll_ctl 控制事件监听,并对要监听的文件符设置回调;
+ 监听的文件符在产生事件时通过回调把事件信息加到了eventpoll结构的rdllist队列里;
+ epoll_wait 取出eventpoll里的已经就绪的rdllist,复制到用户内存;
