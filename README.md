# job

#### larval 里面创建一个job非常简单，只要执行指令 php artisan make:job JobName即可，文章主要介绍job被调度后的源码分析

> 1.首先，我们有一个job SendEmail,执行内容也是非常简单

```php
<?php

namespace App\Jobs;

use App\User;
use Carbon\Carbon;
use Illuminate\Bus\Queueable;
use Illuminate\Queue\SerializesModels;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Support\Facades\Log;

class SendEmail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;


    protected $user;

    protected $tries = 2;
    /**
     * Create a new job instance.
     *
     * @return void
     */
    public function __construct(User $user)
    {
        $this->user = $user;
    }

    /**
     * Execute the job.
     *
     * @return void
     */
    public function handle()
    {
        Log::info('handel in sendemail');
        $user = $this->user;

        $user->remark = Carbon::now()->toDateTimeString();

        //重试测试
//        $user->heh = Carbon::now()->toDateTimeString();


        $user->save();
    }

    public function failed(\Exception $exception)
    {
        Log::error($exception->getMessage());
    }
}


```



> 2.接下来，我们要对job进行调度，laravel 里面自带了方法如下,该方式我们在控制器里调用
>

```php
$user = User::find(1);

$this->dispatch((new SendEmail($user))


```



> app(Dispatcher::class)方法是laravel 容器实例化的过程，具体是调到了类 Illuminate\Bus\Dispatcher里面,下面我们看看具体的方法



```php

public function dispatch($command)

{

   if ($this->queueResolver && $this->commandShouldBeQueued($command)) {

   return $this->dispatchToQueue($command);

  }

   return $this->dispatchNow($command);

}

```



> dispatch方法中，$command是我们传入的job SendEmail,那么$this->queueResolver又是什么呢？我们来看看Dispatcher的构造函数就知道了



```php

    /**
     * Create a new command dispatcher instance.
     *
     * @param  \Illuminate\Contracts\Container\Container  $container
     * @param  \Closure|null  $queueResolver
     * @return void
     */
    public function __construct(Container $container, Closure $queueResolver = null)
    {
        $this->container = $container;
        $this->queueResolver = $queueResolver;
        $this->pipeline = new Pipeline($container);
    }
```



> $queueResolver 默认为null，我们实例化的时候也没有传值，所以不用管再看看$this>commandShouldBeQueued($command)方法



```php
    protected function commandShouldBeQueued($command)
    {
        return $command instanceof ShouldQueue;
    }

```

这段代码的意义在于判断我们的job是异步执行还是立刻执行，用过job的朋友都应该知道，如果我们的job SendEmail 实现 ShoulduQueue这个接口的话，我们就可以把job放到队列中进行异步处理，具体原理判断其实就是这句代码再回到刚才的dispatch代码，如果job是要放到队列里面，那就会走到下面这个方法里面



```php
 /**
     * Dispatch a command to its appropriate handler behind a queue.
     *
     * @param  mixed  $command
     * @return mixed
     *
     * @throws \RuntimeException
     */
    public function dispatchToQueue($command)
    {
        $connection = $command->connection ?? null;

        $queue = call_user_func($this->queueResolver, $connection);

        if (! $queue instanceof Queue) {
            throw new RuntimeException('Queue resolver did not return a Queue implementation.');
        }

        if (method_exists($command, 'queue')) {
            return $command->queue($queue, $command);
        }

        return $this->pushCommandToQueue($queue, $command);
    }
```



$sendEmail->connecton肯定是空，如果要设置的话在任务调度的时候链式调用onConnection('connection')