## 前言
源码基于php 7.1.8
SAPI是php封装的一层对外接口,可以让应用以不同的方式调用php,如命令行,php-fpm等,源码主要在sapi目录下.

## PHP启动方式
常见的php启动方式有:
1. 直接从shell输入命令 php xxxx.php,即cli方式;
2. php-fpm启动,接受从web服务器转发来的php请求,处理请求,输出返回给web服务器,即cgi方式;
3. 作为一个模块嵌在web服务器中,web服务器将程序执行流交给PHP模块,执行完保存结果,并把结果交给web服务器继续执行,即php_mod方式;

## 重要数据结构

### _sapi_module_struct
_sapi_module_struct 是每一种sapi都需要实现的的一个结构,囊括了每种调用方式的最基本的信息,每个属性的基本功能可以从结构中属性的名字看出,如 int (*startup)代表sapi启动函数等等;

各种SAPI方式对于php文件本身的处理,(几乎?)是走的同一个流程,没有区别.区别在于
1. 请求信息从哪里输入;
3. 处理完的结果怎样返回,返回给谁;
4. 遇到错误怎么处理;
5. .....

每种SAPI方式各自实现了一套_sapi_module_struct,控制基本的输入输出.

```c
/* main/SAPI.h*/
struct _sapi_module_struct {
	char *name;
	char *pretty_name;

	int (*startup)(struct _sapi_module_struct *sapi_module);
	int (*shutdown)(struct _sapi_module_struct *sapi_module);

	int (*activate)(void);
	int (*deactivate)(void);

	size_t (*ub_write)(const char *str, size_t str_length);
	void (*flush)(void *server_context);
	zend_stat_t *(*get_stat)(void);
	char *(*getenv)(char *name, size_t name_len);

	void (*sapi_error)(int type, const char *error_msg, ...) ZEND_ATTRIBUTE_FORMAT(printf, 2, 3);

	int (*header_handler)(sapi_header_struct *sapi_header, sapi_header_op_enum op, sapi_headers_struct *sapi_headers);
	int (*send_headers)(sapi_headers_struct *sapi_headers);
	void (*send_header)(sapi_header_struct *sapi_header, void *server_context);

	size_t (*read_post)(char *buffer, size_t count_bytes);
	char *(*read_cookies)(void);

	void (*register_server_variables)(zval *track_vars_array);
	void (*log_message)(char *message, int syslog_type_int);
	double (*get_request_time)(void);
	void (*terminate_process)(void);

	char *php_ini_path_override;

	void (*default_post_reader)(void);
	void (*treat_data)(int arg, char *str, zval *destArray);
	char *executable_location;

	int php_ini_ignore;
	int php_ini_ignore_cwd; /* don't look for php.ini in the current directory */

	int (*get_fd)(int *fd);

	int (*force_http_10)(void);

	int (*get_target_uid)(uid_t *);
	int (*get_target_gid)(gid_t *);

	unsigned int (*input_filter)(int arg, char *var, char **val, size_t val_len, size_t *new_val_len);

	void (*ini_defaults)(HashTable *configuration_hash);
	int phpinfo_as_text;

	char *ini_entries;
	const zend_function_entry *additional_functions;
	unsigned int (*input_filter_init)(void);
};
```

### cli
平时通过运行命令php xxxx.php来运行php是通过cli方式,cli方式实现的sapi_module_struct 如下:
```c
/* cli/php_cli.c */
static sapi_module_struct cli_sapi_module = {
	"cli",							/* name */
	"Command Line Interface",    	/* pretty name */

	php_cli_startup,				/* startup */
	php_module_shutdown_wrapper,	/* shutdown */

	NULL,							/* activate */
	sapi_cli_deactivate,			/* deactivate */

	sapi_cli_ub_write,		    	/* unbuffered write */
	sapi_cli_flush,				    /* flush */
	NULL,							/* get uid */
	NULL,							/* getenv */

	php_error,						/* error handler */

	sapi_cli_header_handler,		/* header handler */
	sapi_cli_send_headers,			/* send headers handler */
	sapi_cli_send_header,			/* send header handler */

	NULL,				            /* read POST data */
	sapi_cli_read_cookies,          /* read Cookies */

	sapi_cli_register_variables,	/* register server variables */
	sapi_cli_log_message,			/* Log message */
	NULL,							/* Get request time */
	NULL,							/* Child terminate */

	STANDARD_SAPI_MODULE_PROPERTIES
};
```

cli模式的主要流程
```c
/* cli/php_cli.c */
int main(int argc, char *argv[]){
    //...
    sapi_module_struct *sapi_module = &cli_sapi_module;
    //...
    sapi_module->ini_defaults = sapi_cli_ini_defaults;
	sapi_module->php_ini_path_override = ini_path_override;
	sapi_module->phpinfo_as_text = 1;
	sapi_module->php_ini_ignore_cwd = 1;
	sapi_startup(sapi_module);
	sapi_started = 1;
    //...
    if (sapi_module->startup(sapi_module) == FAILURE) {
		/* there is no way to see if we must call zend_ini_deactivate()
		 * since we cannot check if EG(ini_directives) has been initialised
		 * because the executor's constructor does not set initialize it.
		 * Apart from that there seems no need for zend_ini_deactivate() yet.
		 * So we goto out_err.*/
		exit_status = 1;
		goto out;
	}
    module_started = 1;
    //...
    exit_status = do_cli(argc, argv);
    //...
}
//...
static int php_cli_startup(sapi_module_struct *sapi_module) /* {{{ */
{
	if (php_module_startup(sapi_module, NULL, 0)==FAILURE) {
		return FAILURE;
	}
	return SUCCESS;
}

```
1. 将sapi_model指向cli模式的cli_sapi_module;
2. 执行一些与sapi相关的初始化;
3. 调用cli_sapi_module的startup函数:  
    这里具体是php_cli_startup,而php_cli_startup只是简单的对php_module_startup函数的封装.php_module_startup函数与sapi无关,初始化所有php模块的MINIT函数;
4. 将module_started标记置为1;
5. 调用do_cli函数处理请求:

#### do_cli
```c
/* cli/php_cli.c */
static int do_cli(int argc, char **argv) /* {{{ */
{
    //...
    if (php_request_startup()==FAILURE) {
			*arg_excp = arg_free;
			fclose(file_handle.handle.fp);
			PUTS("Could not startup.\n");
			goto err;
	}
    //...
    switch (behavior) {
		case PHP_MODE_STANDARD:
			if (strcmp(file_handle.filename, "-")) {
				cli_register_file_handles();
			}

			if (interactive && cli_shell_callbacks.cli_shell_run) {
				exit_status = cli_shell_callbacks.cli_shell_run();
			} else {
				php_execute_script(&file_handle);
				exit_status = EG(exit_status);
			}
			break;
        //...
    }
    //...
    out:
	if (request_started) {
		php_request_shutdown((void *) 0);
	}
    //...
}
```
1. do_cli首先调用php_request_startup函数,这个函数里调用了每个php模块的MINIT函数;
2. 执行php文件;
3. php_request_shutdown函数会根据sapi的不同把php执行结果输出到不同的输出流;


### cgi(php-fpm)
### apache(php_mod)

