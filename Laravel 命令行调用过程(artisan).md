## 前言
laravel 源码版本 5.4
以命令" php artisan schedule:run "为例

## artisan
artisan是laravel官方用来从命令行调用各个组件函数的入口,本身代码很简洁,初始化整个laravel应用,然后将命令行的内容交给Application核心类解析并执行.
```php
/* artisan */

/* 常见的自动加载php第三方类,函数的方式 */
require __DIR__.'/bootstrap/autoload.php';

/* 初始化整个核心应用类Application */
$app = require_once __DIR__.'/bootstrap/app.php';

/* 生成一个类型为Console的Kernel(常用的kernel类型有HTTP 和 Console),这里生成的$kernel的实体类是Illuminate\Foundation\Console\Kernel */
$kernel = $app->make(Illuminate\Contracts\Console\Kernel::class);

/* 调用ConsoleKernel的handle方法处理整个命令行 的输入和输出.
这里第一个参数为新建的一个默认输入类,是根据命令行的信息("schedule:run")构造的;
第二个参数是默认输出类, 一般就是标准输出 */
$status = $kernel->handle(
    $input = new Symfony\Component\Console\Input\ArgvInput,
    new Symfony\Component\Console\Output\ConsoleOutput
);

/* 处理结束 */
$kernel->terminate($input, $status);

exit($status);
```