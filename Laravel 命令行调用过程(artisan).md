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

### Console Kernel 的 handle 处理

```php
/* vendor/laravel/framework/src/Illuminate/Foundation/Console/Kernel.php */

    public function handle($input, $output = null)
    {
        try {
            $this->bootstrap();

            if (! $this->commandsLoaded) {
                $this->commands();

                $this->commandsLoaded = true;
            }
            /* Console Kernel初始化后调用getAtrisan()->run() */
            return $this->getArtisan()->run($input, $output);
        } 
        //...
    }
    //...
    /**
     * Get the Artisan application instance.
     *
     * @return \Illuminate\Console\Application
     */
    protected function getArtisan()
    {
        if (is_null($this->artisan)) {
            return $this->artisan = (new Artisan($this->app, $this->events, $this->app->version()))
                                ->resolveCommands($this->commands);
        }

        return $this->artisan;
    }
```
由上面的的getArtisan方法可以看出kernel通过"懒加载"生成了一个\Illuminate\Console\Application 类;

### Application->run() 方法
```php
/* vendor/laravel/framework/src/Illuminate/Console/Application.php */
class Application extends SymfonyApplication implements ApplicationContract
{
    ...
}
```
Application本身并没有实现run方法,我们从它集成的SymfonyApplication类中找到了run
> Laravel 最初是由symfony分裂出来的（精简版?）
```php
/* vendor/symfony/console/Application.php */
    public function run(InputInterface $input = null, OutputInterface $output = null)
    {
        //...

        try {
            $e = null;
            $exitCode = $this->doRun($input, $output);
        } catch (\Exception $x) {
            $e = $x;
        } catch (\Throwable $x) {
            $e = new FatalThrowableError($x);
        }

        //...
        return $exitCode;
    }

    //...
    public function doRun(InputInterface $input, OutputInterface $output)
    {
        //...
        /* 由input的内容得到命令的名字,这里调试时可以打印出来看,$name的值就是"schedule:run"*/
        $name = $this->getCommandName($input);
        //...

        /* 然后由命令的名字得到具体的command类,同样这里也通过调试打印出$command类为 Illuminate\Console\Scheduling\ScheduleRunCommand*/
        try {
            $e = $this->runningCommand = null;
            // the command name MUST be the first element of the input
            $command = $this->find($name);
        } catch (\Exception $e) {
        } catch (\Throwable $e) {
        }
        //...

        $this->runningCommand = $command;
        $exitCode = $this->doRunCommand($command, $input, $output);
        $this->runningCommand = null;

        return $exitCode;
    }

    //...
    protected function doRunCommand(Command $command, InputInterface $input, OutputInterface $output)
    {
        //...
        /* 去掉一些检测和监听事件分发处理 最终调用了$command->run,这里就是ScheduleRunCommand->run()方法 */
        return $command->run($input, $output);
        //...
    }
```

### ScheduleRunCommand类
```php
/* vendor/laravel/framework/src/Illuminate/Console/Scheduling/ScheduleRunCommand.php */
class ScheduleRunCommand extends Command
{
   //... 
}
```
ScheduleRunCommand类调用的是继承来的Command类的run方法

```php
/* vendor/symfony/console/Command/Command.php */
    public function run(InputInterface $input, OutputInterface $output)
    {
        //...
        $statusCode = $this->execute($input, $output);
        //...
    }
```

```php
/* vendor/laravel/framework/src/Illuminate/Console/Command.php */
    protected function execute(InputInterface $input, OutputInterface $output)
    {
        
        $method = method_exists($this, 'handle') ? 'handle' : 'fire';

        return $this->laravel->call([$this, $method]);
    }
```

最终程序执行到了ScheduleRunCommand的fire方法,即最终实现命令的地方.
```php
/* vendor/laravel/framework/src/Illuminate/Console/Scheduling/ScheduleRunCommand.php */
class ScheduleRunCommand extends Command
{
   //...
    public function fire()
    {
        $eventsRan = false;

        foreach ($this->schedule->dueEvents($this->laravel) as $event) {
            if (! $event->filtersPass($this->laravel)) {
                continue;
            }

            $this->line('<info>Running scheduled command:</info> '.$event->getSummaryForDisplay());

            $event->run($this->laravel);

            $eventsRan = true;
        }

        if (! $eventsRan) {
            $this->info('No scheduled commands are ready to run.');
        }
    }
    //...
}
```