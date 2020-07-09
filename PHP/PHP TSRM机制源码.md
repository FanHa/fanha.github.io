### startup 初始化
在sapi层,php会根据ZTS的值决定是否需要启动tsrm机制的初始化
```c
// sapi/fpm/fpm/fpm_main.c
#ifdef ZTS
	tsrm_startup(1, 1, 0, NULL);
	tsrm_ls = ts_resource(0);
#endif
```
```c
// sapi/cli/php_cli.c
#ifdef ZTS
	tsrm_startup(1, 1, 0, NULL);
	(void)ts_resource(0);
	ZEND_TSRMLS_CACHE_UPDATE();
#endif
```

```c
// TSRM/TSRM.c
TSRM_API int tsrm_startup(int expected_threads, int expected_resources, int debug_level, char *debug_filename)
{

	/* ensure singleton */
	in_main_thread = 1;

    // 这里expected_threads 和 expected_resources 只是歌初始化的指导值,在实际运行时会根据需要增减
	tsrm_tls_table_size = expected_threads;

    // 为tsrm_tls_table 指针分配内存, 这个空间最后是用来存资源容器真正的数据信息
	tsrm_tls_table = (tsrm_tls_entry **) calloc(tsrm_tls_table_size, sizeof(tsrm_tls_entry *));

    // 全局变量线程ID,用于区分不同线程
	id_count=0;
	resource_types_table_size = expected_resources;

    // 为 resource_types_table 指针分配内存,这个空间是用来存资源容器的“元信息”,比如ctor,dtor等
	resource_types_table = (tsrm_resource_type *) calloc(resource_types_table_size, sizeof(tsrm_resource_type));

    // 需要有一个锁来串行化不同线程调用tsrm机制分配内存时的竞态情况
	tsmm_mutex = tsrm_mutex_alloc();

	return 1;
}

```

### ts_allocate_id 申请资源ID
分配一个资源ID(在多线程环境下创建一个全局变量),多线程的情况下,通过这一层封装可以使实际创建的全局变量拷贝多份,每个线程可以通过这个封装得到一个属于自己的变量空间,实现了不同线程间对于同一个全局变量的隔离;
后续各线程可以通过这个资源id和线程id来操作全局变量中属于自己的那一份,而不用相互干涉

```c
// TSRM/TSRM.c
TSRM_API ts_rsrc_id ts_allocate_id(ts_rsrc_id *rsrc_id, size_t size, ts_allocate_ctor ctor, ts_allocate_dtor dtor)
{
	int i;
    // 申请资源容器这个操作需要串行,所以加锁
	tsrm_mutex_lock(tsmm_mutex);

	// 由全局变量id_count来获得一个唯一的资源容器ID
	*rsrc_id = TSRM_SHUFFLE_RSRC_ID(id_count++);

	if (resource_types_table_size < id_count) {
		tsrm_resource_type *_tmp;
        // 根据需要重新分配tsrm_resource_type 的内存(一般都是新增)
		_tmp = (tsrm_resource_type *) realloc(resource_types_table, sizeof(tsrm_resource_type)*id_count);
		if (!_tmp) {

            // 分配失败的话,return 前释放锁
			tsrm_mutex_unlock(tsmm_mutex);
			TSRM_ERROR((TSRM_ERROR_LEVEL_ERROR, "Unable to allocate storage for resource"));
			*rsrc_id = 0;
			return 0;
		}
		resource_types_table = _tmp;
		resource_types_table_size = id_count;
	}
    // 初始化当前资源容器的基本元信息
	resource_types_table[TSRM_UNSHUFFLE_RSRC_ID(*rsrc_id)].size = size;
	resource_types_table[TSRM_UNSHUFFLE_RSRC_ID(*rsrc_id)].ctor = ctor;
	resource_types_table[TSRM_UNSHUFFLE_RSRC_ID(*rsrc_id)].dtor = dtor;
	resource_types_table[TSRM_UNSHUFFLE_RSRC_ID(*rsrc_id)].done = 0;

    // 每一个 tsrm_tls_table[idx] 代表一个线程的“全局变量”存储区域
	for (i=0; i<tsrm_tls_table_size; i++) {
		tsrm_tls_entry *p = tsrm_tls_table[i];

        // 问? 为什么有个链表需要遍历?
        // 答: HASH冲突的解决方案..
		while (p) {
			if (p->count < id_count) {
				int j;
                // 新增一个变量需要先为以前的变量存储区域分配更多空间;
                // 在新空间上调用变量的初始化ctor方法,初始化变量
				p->storage = (void *) realloc(p->storage, sizeof(void *)*id_count);
				for (j=p->count; j<id_count; j++) {
					p->storage[j] = (void *) malloc(resource_types_table[j].size);
					if (resource_types_table[j].ctor) {
						resource_types_table[j].ctor(p->storage[j]);
					}
				}
				p->count = id_count;
			}
			p = p->next;
		}
	}
    // 释放锁
	tsrm_mutex_unlock(tsmm_mutex);
	return *rsrc_id;
}
```

### ts_resource_ex 获取“全局资源”
```c
TSRM_API void *ts_resource_ex(ts_rsrc_id id, THREAD_T *th_id)
{
	THREAD_T thread_id;
	int hash_value;
	tsrm_tls_entry *thread_resources;

    if (!th_id) {
		// 通过快捷通道来寻找当前线程的全局变量区域,可以不用每次都计算HASH
		thread_resources = tsrm_tls_get();

		if (thread_resources) {
			TSRM_ERROR((TSRM_ERROR_LEVEL_INFO, "Fetching resource id %d for current thread %d", id, (long) thread_resources->thread_id));
			// 通过快捷通道返回要查找的变量
			TSRM_SAFE_RETURN_RSRC(thread_resources->storage, id, thread_resources->count);
		}
		thread_id = tsrm_thread_id();
	} else {
		thread_id = *th_id;
	}
	// ...
	tsrm_mutex_lock(tsmm_mutex);

    // 找到当前在tsrm_tls_table的变量存储区域
	hash_value = THREAD_HASH_OF(thread_id, tsrm_tls_table_size);
	thread_resources = tsrm_tls_table[hash_value];

	if (!thread_resources) {
        // 懒加载,在线程真正第一次用到全局变量时才正式分配这个线程的全局变量区域
		allocate_new_resource(&tsrm_tls_table[hash_value], thread_id);
		return ts_resource_ex(id, &thread_id);
	} else {
		 do {
             // 懒加载,全局变量中有很多变量,线程也只在第一次用到某个全局变量时才初始化这个线程的全局变量
			if (thread_resources->thread_id == thread_id) {
				break;
			}
            // HASH冲突时的链表遍历,直到找到想要的
			if (thread_resources->next) {
				thread_resources = thread_resources->next;
			} else {
				allocate_new_resource(&thread_resources->next, thread_id);
				return ts_resource_ex(id, &thread_id);
			}
		 } while (thread_resources);
	}
	tsrm_mutex_unlock(tsmm_mutex);

     // 返回线程需要的全局变量
	TSRM_SAFE_RETURN_RSRC(thread_resources->storage, id, thread_resources->count);
}
```

#### allocate_new_resource
因为懒加载,线程第一次用到全局变量时才会去初始化全局变量在当前线程的复制
```c
static void allocate_new_resource(tsrm_tls_entry **thread_resources_ptr, THREAD_T thread_id)
{
	int i;

	TSRM_ERROR((TSRM_ERROR_LEVEL_CORE, "Creating data structures for thread %x", thread_id));
    // 初始化线程的全局变量存储区域
	(*thread_resources_ptr) = (tsrm_tls_entry *) malloc(sizeof(tsrm_tls_entry));
	(*thread_resources_ptr)->storage = NULL;
	if (id_count > 0) {
		(*thread_resources_ptr)->storage = (void **) malloc(sizeof(void *)*id_count);
	}
	(*thread_resources_ptr)->count = id_count;
	(*thread_resources_ptr)->thread_id = thread_id;
	(*thread_resources_ptr)->next = NULL;

	// 优化,设置快捷通道便于线程迅速找到自己的变量存储区域,而不用每次都计算HASH来映射
	tsrm_tls_set(*thread_resources_ptr);

	
    // 线程就是通过全局变量的元信息(ctor,dtor等)来为自己重新创建一遍全局变量
	for (i=0; i<id_count; i++) {
		if (resource_types_table[i].done) {
			(*thread_resources_ptr)->storage[i] = NULL;
		} else
		{
			(*thread_resources_ptr)->storage[i] = (void *) malloc(resource_types_table[i].size);
			if (resource_types_table[i].ctor) {
				resource_types_table[i].ctor((*thread_resources_ptr)->storage[i]);
			}
		}
	}
	tsrm_mutex_unlock(tsmm_mutex);
}
```