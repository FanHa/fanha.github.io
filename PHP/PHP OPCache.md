### PHP执行逻辑
由多种SAPI 初始化了MINIT,RINIT等Hook后,最终都会调用php_execute_script(&file_handle)

```c
// main/main.c
PHPAPI int php_execute_script(zend_file_handle *primary_file)
{
    //...
    //做一些检测,并对要执行的文件 primary_file 的信息标准化;
    retval = (zend_execute_scripts(ZEND_REQUIRE, NULL, 2, primary_file, append_file_p) == SUCCESS);
    //...
}
```

```c
// Zend/zend.c
ZEND_API int zend_execute_scripts(int type, zval *retval, int file_count, ...) 
{
    // ...
    // 这里调用zend_compile_file将文件解析成zend引擎可以执行的OPCODE(s);
    // 正常情况需要经过词法解析,语法解析等等耗时耗资源的操作才能得出结果;
    // Opcache扩展通过改写这个函数,将解析结果缓存,免去对同一文件的重复解析
    op_array = zend_compile_file(file_handle, type);
    //...
    if (op_array) {
		zend_execute(op_array, retval);
		// ...
	} 
    // ...
}
```

### OPCache 实现

OPCache扩展再MINIT 阶段注册了accel_startup函数
```c
// ext/opcache/ZendAccelerator.c
ZEND_EXT_API zend_extension zend_extension_entry = {
	ACCELERATOR_PRODUCT_NAME,               /* name */
	PHP_VERSION,							/* version */
	"Zend Technologies",					/* author */
	"http://www.zend.com/",					/* URL */
	"Copyright (c) 1999-2018",				/* copyright */
	accel_startup,					   		/* startup */
	NULL,									/* shutdown */
	accel_activate,							/* per-script activation */
	accel_deactivate,						/* per-script deactivation */
	NULL,									/* message handler */
	NULL,									/* op_array handler */
	NULL,									/* extended statement handler */
	NULL,									/* extended fcall begin handler */
	NULL,									/* extended fcall end handler */
	NULL,									/* op_array ctor */
	NULL,									/* op_array dtor */
	STANDARD_ZEND_EXTENSION_PROPERTIES
};
```
---
在`accel_startup`里将`zend_compile_file`替换成了带缓存查找逻辑的`persistent_compile_file`
```c
// ext/opcache/ZendAccelerator.c
static int accel_startup(zend_extension *extension)
{
    //...
    /* Override compiler */
	accelerator_orig_compile_file = zend_compile_file;
	zend_compile_file = persistent_compile_file;
    //...
}
```

#### persistent_compile_file
```c
// ext/opcache/ZendAccelerator.c
zend_op_array *persistent_compile_file(zend_file_handle *file_handle, int type)
{
    // 根据要执行的文件生成一个key,并从缓存池中查找是否由满足的persistent_script
    key = accel_make_persistent_key(file_handle->filename, strlen(file_handle->filename), &key_length);
	persistent_script = zend_accel_hash_str_find(&ZCSG(hash), key, key_length);

    //...
    // 判断缓存时间有没有过期
    if (persistent_script && ZCG(accel_directives).validate_timestamps) {
		if (validate_timestamp_and_record(persistent_script, file_handle) == FAILURE) {
			//...
			persistent_script = NULL;
		}
	}

    // 判断文件是否发生了变化
	if (persistent_script && ZCG(accel_directives).consistency_checks
		&& persistent_script->dynamic_members.hits % ZCG(accel_directives).consistency_checks == 0) {
		unsigned int checksum = zend_accel_script_checksum(persistent_script);
		if (checksum != persistent_script->dynamic_members.checksum ) {
            //...
			persistent_script = NULL;
		}
	}

    // 如果未能找到有效的persistent_script,则需要按正常流程解析文件,并将解析结果缓存供下一次调用比对
    if (!persistent_script) {
        //...
		zend_op_array *op_array;

		persistent_script = opcache_compile_file(file_handle, type, key, &op_array);
		if (persistent_script) {
			persistent_script = cache_script_in_shared_memory(persistent_script, key, key ? key_length : 0, &from_shared_memory);
		}
        //...
	} else {
        //...
	}

    //返回结果(从缓存中取得,或新生成得)
    return zend_accel_load_script(persistent_script, from_shared_memory);
}
```