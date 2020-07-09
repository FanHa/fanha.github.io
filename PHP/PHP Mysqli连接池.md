### 流程源码
```c
// mysqli_api.c
PHP_FUNCTION(mysqli_real_connect)
{
	mysqli_common_connect(INTERNAL_FUNCTION_PARAM_PASSTHRU, TRUE, FALSE);
}
```



```c
// mysqli_noapi.c
void mysqli_common_connect(INTERNAL_FUNCTION_PARAMETERS, zend_bool is_real_connect, zend_bool in_ctor)
{
    // ...
    // 解析参数
    if (zend_parse_method_parameters(...) {
		return;
	}
    
    // ...
    // 如果连接制定了'p:'持久连接,则进入连接池逻辑
    if (strlen(SAFE_STR(hostname)) > 2 && !strncasecmp(hostname, "p:", 2)) {
		hostname += 2;
		if (!MyG(allow_persistent)) {
            // ...
		} else {
			mysql->persistent = persistent = TRUE;

            // 算出连接hash值
			hash_key = strpprintf(0, "mysqli_%s_%s" ZEND_LONG_FMT "%s%s%s", SAFE_STR(hostname), SAFE_STR(socket),
								port, SAFE_STR(username), SAFE_STR(dbname),
								SAFE_STR(passwd));

			mysql->hash_key = hash_key;

            // 寻找连接池中是否有满足hash值的连接 (`EG`相关 见后文)
			if ((le = zend_hash_find_ptr(&EG(persistent_list), hash_key)) != NULL) {
				if (le->type == php_le_pmysqli()) {
					plist = (mysqli_plist_entry *) le->ptr;

                    // 循环所有满足hash值的连接,找到空闲连接
					do {
						if (zend_ptr_stack_num_elements(&plist->free_links)) {
                            // 将找到了连接存入 mysql-mysql,便于后面判断是否从连接池里找到了连接
							mysql->mysql = zend_ptr_stack_pop(&plist->free_links);

							MyG(num_inactive_persistent)--;
                            // 检测该连接是否仍然有效
							if (!mysql_ping(mysql->mysql)) {
								MyG(num_active_persistent)++;
								/* clear error */
								php_mysqli_set_error(mysql_errno(mysql->mysql), (char *) mysql_error(mysql->mysql));	

                                // 找到了有效的连接,直接进入end,复用该连接						
								goto end;
							} else {
                                // 如果连接失效, 关闭该连接
								mysqli_close(mysql->mysql, MYSQLI_CLOSE_IMPLICIT);
								mysql->mysql = NULL;
							}
						}
					} while (0);
				}
			} else {
                // 没有从连接池找到能复用的连接,需要为新建的连接占一个连接池的坑
				zend_resource le;
				le.type = php_le_pmysqli();
				le.ptr = plist = calloc(1, sizeof(mysqli_plist_entry));

				zend_ptr_stack_init_ex(&plist->free_links, 1);

                // `EG` 见后文
				zend_hash_str_update_mem(&EG(persistent_list), ZSTR_VAL(hash_key), ZSTR_LEN(hash_key), &le, sizeof(le));
			}
		}
	}
    // ...
    // 由前面的判断决定需要重新建立连接
    if (!mysql->mysql) {
		if (!(mysql->mysql = mysql_init(NULL))) {
			goto err;
		}
		new_connection = TRUE;
	}
    // 调用mysql_real_connect 来建立一个新连接
    if (mysql_real_connect(mysql->mysql, hostname, username, passwd, dbname, port, socket, flags) == NULL){
        // ...
    }

end:	
	if (!is_real_connect) {
		return;
	} else {
		RETURN_TRUE;
	}


}
```

### 寻找连接池的连接,新加入连接用到的`EG`
+ `EG`意思是php执行器的全局变量;
+ 通常PHP的各个请求独立执行,要想在各个请求间共用资源,必须有每个请求都能访问得到的'全局(或类似全局)'的区域;
+ 这里用到的就是`EG`中的`persistent_llist`;
+ 实际上`persistent_list`并不局限于mysql连接,各种各样的连接都可以;
+ 例如使用php-fpm运行php服务器时,有一块公共区域(通常在MINIT阶段创建),不会因为每个请求的周期(RINIT -> RSHUTDOWN)而释放,这样不同请求通过访问这片公共区域可以实现共享一些资源;