---
title: lumen定时任务
<!-- description: 这是一个副标题 -->
date: 2022-07-20
slug: lumen-schedule
categories:
    - other

tags:
    - other
---

`Lumen`实现定时任务功能的步骤如下。

# 任务逻辑实现

在目录`app/Console/Commands`目录下新建`TestSchedule.php`文件，用来实现定时任务的具体逻辑，代码结构如下：
```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Log;

class TestSchedule extends Command
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'command:test-schedule';

    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = '测试';

    /**
     * Create a new command instance.
     *
     * @return void
     */
    public function __construct() {
        parent::__construct();
    }

    /**
     * Execute the console command.
     *
     * @return mixed
     */
    public function handle()
    {
        Log::info('Begin schedule...');
    }
}
```

上述代码中，`$signature`代码执行命令，`handler`函数即是具体逻辑的实现函数，这里我们实现一个简单的LOG打印。


# 定时任务配置

然后在`app/Console/Kernel.php`增加对该定时任务的配置：
```php
<?php

namespace App\Console;

use Illuminate\Console\Scheduling\Schedule;
use Laravel\Lumen\Console\Kernel as ConsoleKernel;
use Log;

class Kernel extends ConsoleKernel
{
    /**
     * The Artisan commands provided by your application.
     *
     * @var array
     */
    protected $commands = [
        \App\Console\Commands\TestSchedule::class,
    ];

    /**
     * Define the application's command schedule.
     *
     * @param  \Illuminate\Console\Scheduling\Schedule  $schedule
     * @return void
     */
    protected function schedule(Schedule $schedule)
    {
        $schedule->command('command:test-schedule')
            ->everyMinute()
            ->when(function () {
               return true;
            });
            //->withoutOverlapping();
    }
}
```

如上述代码，`$commands`中配置定时任务的实现类，然后在`schedule`函数中增加对该定时任务的时间等配置。
`everyMinute`表示每分钟，可以使用更多的时间配置，具体查看相关文档。
`when（Closure）`用来根据给定的测试的结果来限制任务的执行，只有当`Closure`返回`true`时才执行。
默认情况，即便有相同的任务还在运行，调度内的任务依旧会被执行，`withoutOverlapping`可以用来解决这个问题。
**值得注意的是，`withoutOverlapping`不能和`when`共用，即使`when`返回值为`true`。所以当有`withoutOverlapping`需求时，是否执行该任务的判断需要放在`handler`函数中。**


# 执行

上述代码创建完成后，我们可以通过命令`php artisan command:test-schedule`来执行一次定时任务，用来验证代码逻辑是否正确，该命令会立即运行定时任务，并且会忽略`when`的判断。

确认逻辑正确后，需要把定时任务加到`linux cron`配置中去，让它可以定时执行。

这里可以配置两个位置，一个是在`/etc/cron.d`目录下，新增定时任务文件，形式如下：
```c
* * * * * laradock php /var/www/artisan schedule:run >> /dev/null 2>&1
```
`etc/cron.d`里面的配置是系统级别的，需要指定用户执行。`schedule:run`表示开始执行全部的定时任务。

还有一种方法是输入命令 `crontab -e` 进行编辑，文件在`/var/spool/cron/crontabs`目录下，`crontab -e`会以当前用户去执行脚本。`crontab -l`可以查看当前已经设定的定时任务。
