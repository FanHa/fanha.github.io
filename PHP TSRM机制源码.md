### startup
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

### ts_allocate_id
初始化分配一个资源容器ID,狭义上每个线程可以通过这样的一个资源容器ID来存放自己线程里的公共资源,相当于一个命名域的概念;
其实一些其他功能模块也需要一个命名域来隔离自己的资源时也会用到这个ts_allocate_id

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

            // 分配失败的话,return 前解除锁
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

	/* enlarge the arrays for the already active threads */
	for (i=0; i<tsrm_tls_table_size; i++) {
		tsrm_tls_entry *p = tsrm_tls_table[i];

		while (p) {
			if (p->count < id_count) {
				int j;

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
	tsrm_mutex_unlock(tsmm_mutex);

	TSRM_ERROR((TSRM_ERROR_LEVEL_CORE, "Successfully allocated new resource id %d", *rsrc_id));
	return *rsrc_id;
}
```
