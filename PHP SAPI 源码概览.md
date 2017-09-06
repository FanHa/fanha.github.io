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
static int php_cli_startup(sapi_module_struct *sapi_module)
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
static int do_cli(int argc, char **argv)
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
3. php_request_shutdown函数最终会调用sapi结构注册的flush或ub_write函数,以此来区分不同的sapi的输出方式;


### cgi,fcgi,和php-fpm
1. cgi:  
cgi是一种协议格式,客户端把http请求发送给web服务器,web服务器按一定的格式整理好,启动一个cgi程序,cgi程序处理完把结果原路返回;  

	客户端<-->web(http服务器)-->cgi程序

2. fcgi: 
原始的cgi程序每次处理请求都需要启动一个进程,很耗服务器资源;fcgi是一个与web服务器独立的进程,它自身初始化多个请求处理程序池,重复利用这些处理程序池接受web服务器按一定格式传来的请求,返回结果给web服务器.

	客户端<-->web(http服务器)<-->fcgi程序

3. php-fpm:
cgi只是一种协议,任何语言都可以有相应的实现,php-fpm就是用来处理php请求的fcgi进程.

#### cgi 和 fcgi
sapi结构如下
```c
/* sapi/cgi/cgi_main.c */
static sapi_module_struct cgi_sapi_module = {
	"cgi-fcgi",						/* name */
	"CGI/FastCGI",					/* pretty name */

	php_cgi_startup,				/* startup */
	php_module_shutdown_wrapper,	/* shutdown */

	sapi_cgi_activate,				/* activate */
	sapi_cgi_deactivate,			/* deactivate */

	sapi_cgi_ub_write,				/* unbuffered write */
	sapi_cgi_flush,					/* flush */
	NULL,							/* get uid */
	sapi_cgi_getenv,				/* getenv */

	php_error,						/* error handler */

	NULL,							/* header handler */
	sapi_cgi_send_headers,			/* send headers handler */
	NULL,							/* send header handler */

	sapi_cgi_read_post,				/* read POST data */
	sapi_cgi_read_cookies,			/* read Cookies */

	sapi_cgi_register_variables,	/* register server variables */
	sapi_cgi_log_message,			/* Log message */
	NULL,							/* Get request time */
	NULL,							/* Child terminate */

	STANDARD_SAPI_MODULE_PROPERTIES
};
```

基本执行流程
```c
/* sapi/cgi/cgi_main.c */
int main(int argc, char *argv[])
{
	//...
	/* 判断自己是fcgi还是cgi */
	fastcgi = fcgi_is_fastcgi();
	//...
	if (!fastcgi) {
		/* Make sure we detect we are a cgi - a bit redundancy here,
		 * but the default case is that we have to check only the first one. */
		if (getenv("SERVER_SOFTWARE") ||
			getenv("SERVER_NAME") ||
			getenv("GATEWAY_INTERFACE") ||
			getenv("REQUEST_METHOD")
		) {
			cgi = 1;
		}
	}
	/* 将请求的query_string保存下来 */
	if((query_string = getenv("QUERY_STRING")) != NULL && strchr(query_string, '=') == NULL) {
		/* we've got query string that has no = - apache CGI will pass it to command line */
		unsigned char *p;
		decoded_query_string = strdup(query_string);
		php_url_decode(decoded_query_string, strlen(decoded_query_string));
		for (p = (unsigned char *)decoded_query_string; *p &&  *p <= ' '; p++) {
			/* skip all leading spaces */
		}
		if(*p == '-') {
			skip_getopt = 1;
		}
		free(decoded_query_string);
	}
	//...
	/* 如果是fcgi将一些sapi函数替换 */
	if (fastcgi || bindpath) {
		/* Override SAPI callbacks */
		cgi_sapi_module.ub_write     = sapi_fcgi_ub_write;
		cgi_sapi_module.flush        = sapi_fcgi_flush;
		cgi_sapi_module.read_post    = sapi_fcgi_read_post;
		cgi_sapi_module.getenv       = sapi_fcgi_getenv;
		cgi_sapi_module.read_cookies = sapi_fcgi_read_cookies;
	}
	//...
	/* 将要执行的文件路径保存 */
	cgi_sapi_module.executable_location = argv[0];
	//...
	/* 执行sapi结构的startup函数, 这里调用的是php_cgi_startup,实际主要就是执行每个php模块的 MINIT */
	if (cgi_sapi_module.startup(&cgi_sapi_module) == FAILURE){
		//...
	}
	//...
	/* 这里依然是判断是否是cgi,从警告可以看出这种方式确实存在安全隐患,已经不再被php新版本支持了*/
	if (cgi && CGIG(force_redirect)) {
		//...
		SG(sapi_headers).http_response_code = 400;
				PUTS("<b>Security Alert!</b> The PHP CGI cannot be accessed directly.\n\n\
<p>This PHP CGI binary was compiled with force-cgi-redirect enabled.  This\n\
means that a page will only be served up if the REDIRECT_STATUS CGI variable is\n\
set, e.g. via an Apache Action directive.</p>\n\
<p>For more information as to <i>why</i> this behaviour exists, see the <a href=\"http://php.net/security.cgi-bin\">\
manual page for CGI security</a>.</p>\n\
<p>For more information about changing this behaviour or re-enabling this webserver,\n\
consult the installation file that came with this distribution, or visit \n\
<a href=\"http://php.net/install.windows\">the manual page</a>.</p>\n");
	}
	//...
	/* 接下来进入fcgi方式的执行流程了 */
	if (fastcgi) {
		//...
		/* 首先初始化请求的结构(主要是分配空间,属性的初始化在后头) */
		request = fcgi_init_request(fcgi_fd, NULL, NULL, NULL);
		//...
		/* 这里parent是一个全局变量,初始化为1,即父进程,父进程进入无限循环保证正在运行的子进程数量*/
		while (parent) {
			do {
					pid = fork();
					switch (pid) {
					case 0:
						/* 子进程把parent变量置为0 ,一遍子进程跳出此循环 */
						parent = 0;

						/* don't catch our signals */
						sigaction(SIGTERM, &old_term, 0);
						sigaction(SIGQUIT, &old_quit, 0);
						sigaction(SIGINT,  &old_int,  0);
						zend_signal_init();
						break;
					case -1:
						perror("php (pre-forking)");
						exit(1);
						break;
					default:
						/* Fine */
						running++;
						break;
					}
			} while (parent && (running < children));
			//...
		}
	}
	//...
	/* 循环调用等待收到一个fcgi请求时,做出处理(TODO:这里有疑问,从哪里监听了请求事件?) */
	while (!fastcgi || fcgi_accept_request(request) >= 0) {
			SG(server_context) = fastcgi ? (void *)request : (void *) 1;
			/* 这里是真正的把请求的内容保存到了本地的request结构里*/
			init_request_info(request);
			//... 
			/* 保存cgi,fcgi 命令需要执行的脚本文件名字 */
			if (SG(request_info).path_translated || cgi || fastcgi) {
				file_handle.type = ZEND_HANDLE_FILENAME;
				file_handle.filename = SG(request_info).path_translated;
				file_handle.handle.fp = NULL;
			} else {
				file_handle.filename = "-";
				file_handle.type = ZEND_HANDLE_FP;
				file_handle.handle.fp = stdin;
			}
			//...
			/* 把请求的各种信息都初始化到file_handle结构后,就把整个程序流程交给php引擎去解析和执行了 */
			switch (behavior) {
				case PHP_MODE_STANDARD:
					php_execute_script(&file_handle);
					break;
					//...
			}

	}

}
```
### apache(php_mod)

