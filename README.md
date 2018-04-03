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



> 3.app(Dispatcher::class)方法是laravel 容器实例化的过程，具体是调到了类 Illuminate\Bus\Dispatcher里面,下面我们看看具体的方法



```php

public function dispatch($command)

{

   if ($this->queueResolver && $this->commandShouldBeQueued($command)) {

   return $this->dispatchToQueue($command);

  }

   return $this->dispatchNow($command);

}

```



> 4.dispatch方法中，$command是我们传入的job SendEmail,那么$this->queueResolver又是什么呢？我们来看看Dispatcher的构造函数就知道了



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

app的实例化以及绑定过程在BusServiceProvider中

```php
public function register()
    {
        $this->app->singleton(Dispatcher::class, function ($app) {
            return new Dispatcher($app, function ($connection = null) use ($app) {
                return $app[QueueFactoryContract::class]->connection($connection);
            });
        });

        $this->app->alias(
            Dispatcher::class, DispatcherContract::class
        );

        $this->app->alias(
            Dispatcher::class, QueueingDispatcherContract::class
        );
    }
```



> Register 方法中绑定了 new Dispatcher()类，构造方法传入了$app容器以及一个闭包

> return $app[QueueFactoryContract::class]->connection($connection) 返回的是一个RedisQueue,执行了
>
> QueueManager的connection()方法



```php 
 public function connection($name = null)
    {
        $name = $name ?: $this->getDefaultDriver();

        // If the connection has not been resolved yet we will resolve it now as all
        // of the connections are resolved when they are actually needed so we do
        // not make any unnecessary connection to the various queue end-points.
        if (! isset($this->connections[$name])) {
            $this->connections[$name] = $this->resolve($name);

            $this->connections[$name]->setContainer($this->app);
        }

        return $this->connections[$name];
    }
```



> 5.再看看$this>commandShouldBeQueued($command)方法



```php
    protected function commandShouldBeQueued($command)
    {
        return $command instanceof ShouldQueue;
    }
```



> 6.这段代码的意义在于判断我们的job是异步执行还是立刻执行，用过job的朋友都应该知道，如果我们的job SendEmail 实现 ShoulduQueue这个接口的话，我们就可以把job放到队列中进行异步处理，具体原理判断其实就是这句代码再回到刚才的dispatch代码，如果job是要放到队列里面，那就会走到下面这个方法里面



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

> 由于我们没有在job里面写queue方法，所以直接执行了$this->pushCommandToQueue()方法

```php
 protected function pushCommandToQueue($queue, $command)
    {
        if (isset($command->queue, $command->delay)) {
            return $queue->laterOn($command->queue, $command->delay, $command);
        }

        if (isset($command->queue)) {
            return $queue->pushOn($command->queue, $command);
        }

        if (isset($command->delay)) {
            return $queue->later($command->delay, $command);
        }

        return $queue->push($command);
    }
```



> 对sendEmail进行延迟判断，是否制定队列判断，最后就是调到了RedisQueue类里面把数据存在redis里面了



##### 如果没有进入到队列里面，那job就是立即执行了，程序就执行到方法 dispatchNow()里面



```php
public function dispatchNow($command, $handler = null)
    {

        if ($handler || $handler = $this->getCommandHandler($command)) {
            $callback = function ($command) use ($handler) {
                return $handler->handle($command);
            };
        } else {
            $callback = function ($command) {
                return $this->container->call([$command, 'handle']);
            };
        }

        return $this->pipeline->send($command)->through($this->pipes)->then($callback);
    }
```



> handler为自定义的处理类，这边我们没有自定义，所以会方法执行到else，$callback值为call方法调用SendEmail的handler方法，方法最后通过管道模式进行处理，对$command的数据进行callback的操作

