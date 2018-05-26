layout: post
title: "Laravel5.4 生命流程与源码分析"
date: 2017-12-19 14:48
comments: true
tags: 
	- PHP
	- Laravel
---

明白流程不代表精通Laravel。因为还需要把这个流程与实际开发需要融汇贯通：学以致用。比如，如何修改Laravel日志存放目录？Laravel并没有自定义日志目录的配置，需要修改源代码，这就需要了解Laravel的生命流程，明白Laravel是在哪里，什么时候进行日志目录设置。
<!--more-->
## 一、 整体流程

![](https://note.youdao.com/yws/api/personal/file/4F19E325A4C643ACAB51EBC5E4641E7F?method=download&shareKey=36c7260a223a5a110def4bf4216ad92a)

绑定的异常处理：App\Exceptions\Handler::class 可以在这里自定义

## 二、 Application初始化流程

![](https://note.youdao.com/yws/api/personal/file/D5094DACC2A24745A75FD728EC7E247A?method=download&shareKey=7f94729f8550bae15f00667f1bcc9554)

### 2.1 App初始化解释
由此看来，Applicantion的初始化并没有很复杂

1. 初始化容器
2. Events/Log/Router注册基础服务
3. 设置容器中abstract与alias的映射关系

####  Events ：单例绑定 events => `Illuminate\Events\Dispatcher`
    - 并且设置了`setQueueResolver` Resolver 一般是一个callable类型，用于向容器获取某个对象。比如这里的是一个队列
#### Log : 单例绑定 log => \Illuminate\Log\Writer
```
public function createLogger()
{
    $log = 
    // \Monolog\Logger
    new Monolog($this->channel()),
    // \Illuminate\Contracts\Events\Dispatcher 上面Events的绑定
    $this->app['events']

    if ($this->app->hasMonologConfigurator()) {
        call_user_func($this->app->getMonologConfigurator(), $log->getMonolog());
    } else {
        // 默认会走这里
        $this->configureHandler($log);
    }

    return $log;
}

// channel 只是得到当前的环境
protected function channel()
{
    return $this->app->bound('env') ? $this->app->environment() : 'production';
}
```

默认会走`configureHandler`方法：
```php
protected function configureHandler(Writer $log)
{
    $this->{'configure'.ucfirst($this->handler()).'Handler'}($log);
}

// 从配置中得到 是single/daily/Syslog/Errorlog
protected function handler()
{
    if ($this->app->bound('config')) {
        return $this->app->make('config')->get('app.log', 'single');
    }

    return 'single';
}

// single 调用: $log->useFiles
// daily  调用：$log->useDailyFiles
// Syslog 调用：$log->useSyslog
// Errorlog 调用：$log->useErrorLog
```
这里是要从配置上获取啊，关键是现在还没有加载配置！！这是什么情况?
```
$this->app->singleton('log', function () {
    return $this->createLogger();
});
```
虽然是绑定，但只是绑定一个`Resolver` ，一个闭包，并不是马上实例化Logger的。所以在加载配置前，将会使用默认值形式记录日志的。

意思是设置monolog 该把日志写在哪里，怎么写，比如 daily。
```php
protected function configureDailyHandler(Writer $log)
{
    $log->useDailyFiles(
        // 固定storage目录和文件名 可以在这修改
        $this->app->storagePath().'/logs/laravel.log', $this->maxFiles(),
        $this->logLevel()
    );
}
```
如果想要自定义处理，则：
```php
//  设置一个回调即可
$app->configureMonologUsing('myLogCallback');

function myLogCallback($log)
{
    $log->useDailyFiles(
        '/tmp/logs/laravel.log', 
        5, // 最多几个日志文件，可以写成配置
        'debug' // 日志等级，可以写成配置
    );
}
//关键是在哪里调用这个设置比较好？
```
如果只是修改日志路径，我建议修改对应handler的路径即可，比如修改`configureDailyHandler`的路径修改一下。甚至可以在服务提供者下添加一个自定义的handler。我认为Laravel应该把这个Log服务提供者做成Kernel这样的继承。

比如：
App 的 registerBaseServiceProviders 中：

```php
$this->register(new \Illuminate\Log\LogServiceProvider($this));
```

改成：

```php
$this->register(new \App\Providers\LogServiceProvider($this));
```

然后再app/Providers目录下新增一个`LogServiceProvider extens \Illuminate\Log\LogServiceProvider`。 这样我们就可以重写或者添加日志的configuredHandler`日志配置handler`了.


#### `Illuminate\Routing\RoutingServiceProvider` 的注册 `register`

```php
/**
 * Register the service provider.
 *
 * @return void
 */
public function register()
{
    /*
    绑定 ：router => Illuminate\Routing\Router
    Router 定义了 get post put
    */
    $this->registerRouter();

    /*
    绑定： url => Illuminate\Routing\UrlGenerator
    绑定： routes => \Illuminate\Routing\RouteCollection
    回调： session :setSessionResolver
    回调： 上述绑定的 rebind
    */
    $this->registerUrlGenerator();

    /*
    绑定：redirect => Illuminate\Routing\Redirector
    */
    $this->registerRedirector();

    $this->registerPsrRequest();

    $this->registerPsrResponse();

    $this->registerResponseFactory();
}
```

暂时可以把Kernel看作一个黑匣子：把请求往里放，响应返回出来。

##### registerRouter 注册了 Router路由
> Facade 中的Route实际上调用的就是 Illuminate\Routing\Router 实例的方法。Router还有魔术方法__call  类似 `as`, `domain`, `middleware`, `name`, `namespace`, `prefix` 这些Router没有的方法，会从新创建一个新的 `Illuminate\Routin\RouteRegistrar` 实例调用其`attribute` 方法 (Registrar是登记的意思)

这些都回在`RouteServiceProvider` 上调用：

```php
/**
 * Define the "web" routes for the application.
 *
 * These routes all receive session state, CSRF protection, etc.
 *
 * @return void
 */
protected function mapWebRoutes()
{
    Route::middleware('web') // 这里就创建了RouteRegistrar 并且返回自身
         ->namespace($this->namespace)
         ->group(base_path('routes/web.php'));
}
```
最终 RouteRegistrar 还是对router对象进行操作。
疑问：为什么不直接操作router,而走这么复杂的路？

主要是给RouteServiceProvider 的`mapWebRoutes` 和 `mapApiRoutes` 方法使用。作用是区分并且保存 路由设置的属性。因为每次调用都会重新创建一个`RouteRegistrar`实例，因此 其attributes都是不共享的，起到Web与Api理由设置的属性不会相互影响。

**要注意**：服务提供者注册的时候没有进行依赖注入，也不需要使用依赖注入；因为实例化的时候就把`app`作为参数传递进去，如果需要直接使用`app`获取依赖的对象即可

## 三、Http Kernal 的流程

看代码：
```php
$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);

$response = $kernel->handle(
    $request = Illuminate\Http\Request::capture()
);

$response->send();

$kernel->terminate($request, $response);
```

上面看，**Kernel大体做了4件事：**
* 初始化Kernel
* 捕捉请求与handle请求
* 发送请求
* terminate

![](https://note.youdao.com/yws/api/personal/file/D79214AEDE3E49D48C2B1E86DD67A7E2?method=download&shareKey=a8a411cc1e721ca79c277123c1adc16a)

### 3.1 Kernel初始化
```php
public function __construct(Application $app, Router $router)
{
    $this->app = $app;
    $this->router = $router;

    // 设置中间件优先级列表
    $router->middlewarePriority = $this->middlewarePriority;

    // 中间件组 web 和 api  的中间件组
    foreach ($this->middlewareGroups as $key => $middleware) {
        $router->middlewareGroup($key, $middleware);
    }

    // 路由中间件
    foreach ($this->routeMiddleware as $key => $middleware) {
        $router->aliasMiddleware($key, $middleware);
    }
}
```

Kernel初始化做了3件事，总体是一件事（设置中间件）：
1. 给路由设置中间件固定优先级列表
2. 设置中间件分组(Web与Api的分组)
3. 设置路由中间件(在路由中设置的中间件)

后两步的中间件属性都是空的，具体的值在`App\Http\Kernel`中

在设置路由中间件中`aliasMiddleware` 顾名思义，就是给中间件设置一个别名而已。

### 3.2 Kernel处理请求
> 捕获请求后面再说

这里是核心部分了：路由调度，中间件栈(闭包)，自定义异常处理。在此之前都没有进行过一次的`依赖注入`。因为还没有在容器当中。

```php
public function handle($request)
{
    try {
        $request->enableHttpMethodParameterOverride();

        // 把请求通过以下路由，就变成了响应
        $response = $this->sendRequestThroughRouter($request);
    } catch (Exception $e) {
        $this->reportException($e);

        $response = $this->renderException($request, $e);
    } catch (Throwable $e) {
        $this->reportException($e = new FatalThrowableError($e));

        $response = $this->renderException($request, $e);
    }

    event(new Events\RequestHandled($request, $response));

    return $response;
}
```
**注意：**`Throwable`能够捕捉 Exception 和 Error。这是php7 的特征。

`enableHttpMethodParameterOverride` 做什么？方法欺骗！

```php
/**
 * Enables support for the _method request parameter to determine the intended HTTP method.
 *
 * Be warned that enabling this feature might lead to CSRF issues in your code.
 * Check that you are using CSRF tokens when required.
 * If the HTTP method parameter override is enabled, an html-form with method "POST" can be altered
 * and used to send a "PUT" or "DELETE" request via the _method request parameter.
 * If these methods are not protected against CSRF, this presents a possible vulnerability.
 *
 * The HTTP method can only be overridden when the real HTTP method is POST.
 */
public static function enableHttpMethodParameterOverride()
{
    self::$httpMethodParameterOverride = true;
}
```

主要看注释。大概意思：
> 为了能够通过_method参数进行方法欺骗.被警告，启用这个特性可能会导致你的代码中的CSRF问题。检查您是否在需要时使用CSRF令牌。如果启用HTTP方法参数覆盖，则可以更改具有方法“POST”的html表单，并通过_method请求参数发送“PUT”或“DELETE”请求。如果这些方法不受CSRF保护，就会出现一个可能的漏洞。方法前片只有在真正的HTTP方法是POST的时候才会被修改


#### sendRequestThroughRouter做的几件事

1. 绑定request
2. 启动App容器(重要)
    * 加载.env环境配置
    * 加载config目录配置
    * 设置错误和异常的handler
    * 设置Facade别名自动加载
    * 注册服务提供者
    * 启动服务提供者** (这个时候才开始可以使用依赖注入)**
3. 把请求通过中间件和路由 
    * 使用管道(闭包栈)
    * 路由调度

```php
protected function sendRequestThroughRouter($request)
{
    // 通过路由的时候才绑定请求到容器中 
    $this->app->instance('request', $request);

    // 清除Facade的$resolvedInstance属性保存的对象。
    // 当使用Facade的时候如果不存在则从app容器获取对象
    Facade::clearResolvedInstance('request');

    // 启动App 其实是调用一些类
    $this->bootstrap();

    return (new Pipeline($this->app))
                ->send($request)
                ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
                ->then($this->dispatchToRouter());
}
```

##### 启动App 其实是调用一些类
App 下的
```
public function bootstrapWith(array $bootstrappers)
{
    // 修改为已经启动
    $this->hasBeenBootstrapped = true;

    // 启动类 触发事件
    foreach ($bootstrappers as $bootstrapper) {
        $this['events']->fire('bootstrapping: '.$bootstrapper, [$this]);

        // 调用这些类的bootstrap方法
        $this->make($bootstrapper)->bootstrap($this);

        $this['events']->fire('bootstrapped: '.$bootstrapper, [$this]);
    }
}
```

看一下App启动流程有哪些？
```
protected $bootstrappers = [
   // 加载 .env环境变量
    \Illuminate\Foundation\Bootstrap\LoadEnvironmentVariables::class,
    // 加载config目录的配置
    \Illuminate\Foundation\Bootstrap\LoadConfiguration::class,
    \Illuminate\Foundation\Bootstrap\HandleExceptions::class,
    \Illuminate\Foundation\Bootstrap\RegisterFacades::class,
    // 注册配置的服务提供者
    \Illuminate\Foundation\Bootstrap\RegisterProviders::class,
    // 启动服务提供者 即调用 boot方法
    \Illuminate\Foundation\Bootstrap\BootProviders::class,
];
```

注意：环境配置与config配置的加载之前已经说过了，这里就不说了：
1. [.env读取](http://note.youdao.com/noteshare?id=0a7ffe86953dce8cb27eab0a8e159326&sub=3672DE1ADCBC4DBEB9AA804667A18246)
2. [config配置加载](http://note.youdao.com/noteshare?id=a2ecdcc23fa7d3b9333edb79fce4bf27&sub=FD051E2A38614647BDB78E3CC3A3F20C)

**这里简单总结一下上面两个链接：**
* 不能在config目录外使用env辅助函数
* 不能在config目录内定义配置以外的内容，比如定义类

为什么有这样的要求？因为如果使用 php artisan config:cache 把配置缓存后，env函数将不再去加载.env文件，config函数也不会加载config目录(目录中的类也就不能加载了)

##### 异常错误handler
> `Illuminate\Foundation\Bootstrap\HandleExceptions`的`bootstrap`方法:

```
public function bootstrap(Application $app)
{
    $this->app = $app;
    // 设置报所有的错 -1 的补码全都是1
    error_reporting(-1);
    // 设置系统错误地handler：warning notice等
    set_error_handler([$this, 'handleError']);
    // 设置异常处理
    set_exception_handler([$this, 'handleException']);
    // 设置php结束时的处理函数
    register_shutdown_function([$this, 'handleShutdown']);

    if (! $app->environment('testing')) {
        ini_set('display_errors', 'Off');
    }
}
```
其实就是设置一些错误和异常处理回调,最终的handler是之前说的`App\Exceptions\Handler::class`类。

我们看看：
```
public function handleError($level, $message, $file = '', $line = 0, $context = [])
{
    if (error_reporting() & $level) {
        throw new ErrorException($message, 0, $level, $file, $line);
    }
}
```
只是把一些可以捕获的PHP Error转为Exception，然后再由ExceptionHandler处理。

```
public function handleException($e)
{
    if (! $e instanceof Exception) {
        $e = new FatalThrowableError($e);
    }

    // 报告错误，比如写入错误日志
    $this->getExceptionHandler()->report($e);

    // 根据不同的运行环境输出不一样的错误形式
    if ($this->app->runningInConsole()) {
        $this->renderForConsole($e);
    } else {
        $this->renderHttpResponse($e);
    }
}
```
好了，看到这里大家明白了自定义异常处理是在:``App\Exceptions\Handler::class` 处理的.

##### Facade的注册：
这个是需要在`app.aliases`上进行配置的。

调用了一个`AliasLoader`类：顾名思义，就是一个类别名自动加载者；就是为这些Facade设置自动加载。

```
/**
 * AliasLoader类
 * Prepend the load method to the auto-loader stack.
 *
 * @return void
 */
protected function prependToLoaderStack()
{
    spl_autoload_register([$this, 'load'], true, true);
}
```
[spl_autoload_register官网解释](http://php.net/manual/zh/function.spl-autoload-register.php)

```
bool spl_autoload_register ([ callable $autoload_function [, bool $throw = true [, bool $prepend = false ]]] )
```
**Facade性能：** Laravel把Facade的加载放在最前面。因此使用Facade是可以提高自动加载速度的。但是如果在app.aliases加了一个不是Facade的一个别名，这将会降低性能（因为会去写文件，创建一个Facade）

看看Facade是怎么加载的：
```
public function load($alias)
{
    // 如果没有Facade的命名空间,说明不是Facade
    if (static::$facadeNamespace && strpos($alias, static::$facadeNamespace) === 0) {
        // 使用Facade的模板自动创建一个与别名一样的Facade，并且加载它：ensureFacadeExists
        $this->loadFacade($alias);
        // 返回 true 告诉PHP不用再去加载，已经加载了
        return true;
    }

    // 有Facade这个命名空间: Facades\\
    if (isset($this->aliases[$alias])) {
        // 设置别名，并且自动加载
        return class_alias($this->aliases[$alias], $alias);
    }
}
// PHP class_alias原生函数
bool class_alias ( string $original , string $alias [, bool $autoload = TRUE ] )
```
##### register服务提供者

```
public function registerConfiguredProviders()
{
    (new ProviderRepository($this, new Filesystem, $this->getCachedServicesPath()))
                ->load($this->config['app.providers']);
}
```
服务提供者的缓载介绍：[服务提供者缓载](http://note.youdao.com/noteshare?id=49f0ffc3175a7ed3a3562a21bc9ade2c&sub=B7452693C6314F67874338051B0EAC35)

这里简单说：
1. 首先使用`$this->getCachedServicesPath()`得到服务提供者缓存起来的文件
2. 是否有缓存或者配置是否变化（变化就更新）
3. 使用配置的进行判断哪些是缓载(需要时才加载)

主要看`Illuminate\Foundation\ProviderRepository`的`load`方法:
```
public function load(array $providers)
{
    // 加载缓存的services.php
    $manifest = $this->loadManifest();
    
    // 是否需要从新生成缓存文件
    if ($this->shouldRecompile($manifest, $providers)) {
        $manifest = $this->compileManifest($providers);
    }

    // 对于 when形式的服务提供者 在某事件触发才注册
    foreach ($manifest['when'] as $provider => $events) {
        $this->registerLoadEvents($provider, $events);
    }

    // 这些是马上就注册的
    foreach ($manifest['eager'] as $provider) {
        $this->app->register($provider);
    }
    
    // 这个是缓载的。只是添加到App的deferredServices属性
    $this->app->addDeferredServices($manifest['deferred']);
}

// 添加事件监听
protected function registerLoadEvents($provider, array $events)
{
    if (count($events) < 1) {
        return;
    }

    $this->app->make('events')->listen($events, function () use ($provider) {
        $this->app->register($provider);
    });
}
```

##### boot 启动服务提供者
> 这个时候会使用App 的 call方法来调用所有注册的服务提供者。为什么使用 `call`方法，而不是直接调用？因为使用依赖注入。要知道这个`call`方法是public的，我们可以在其他地方也可以显示使用


#### 依赖注入
```
/**
 * Call the given Closure / class@method and inject its dependencies.
 *
 * @param  callable|string  $callback
 * @param  array  $parameters
 * @param  string|null  $defaultMethod
 * @return mixed
 */
public function call($callback, array $parameters = [], $defaultMethod = null)
{
    return BoundMethod::call($this, $callback, $parameters, $defaultMethod);
}
```

`Illuminate\Container\BoundMethod`就是依赖注入实现的类了(需要传递App容器进去才行)

#### 依赖注入流程
> 从`Illuminate\Container\BoundMethod`的`call`方法开始
```
graph TD
    A[A BoundMethod::call] --> B
    B{B callback 是字符串型} -->|Yes|C
    B --> |No|E
    C[C 转为 array 型callable] --> E
    E[E callBoundMethod] --> F
    F{F callback 是 array 型} --> |No|G
    F --> |Yes|H
    G[G 直接调用default并且返回] --> Z
    H{H callback 是 app的bindingMethod} --> |Yes|I
    I[I 直接调用app的bindingethod] --> Z
    H --> |No|J
    J[J 直接调用default并且返回] --> Z
    Z[Z 返回call结果]
```
注释：
* B中字符串型：是字符串；且有‘@’字符或者有默认方法。
* F判断是否为array型。因为callback仍可能是字符串，且没有满足B中条件，这就直接到G。为什么？
比如你要对某个**函数进行依赖注入**
G情况的伪代码：
```
function test(Request $request)
{
    ...
}
$app->call('test');
```
* G中直接调用或者返回$default，是？
```
return $default instanceof Closure ? $default() : $default;
```
* default 个是实现依赖注入的关键！！！下面是 这个闭包的代码：
```
function () use ($container, $callback, $parameters) {
    return call_user_func_array(
        $callback, static::getMethodDependencies($container, $callback, $parameters)
    );
}
```
看出，进行判断$callback 需要哪些依赖和参数的方法是:`getMethodDependencies`
```
// 得到方法或者函数的参数列表 包括依赖注入的和$parameters本身传入的
protected static function getMethodDependencies($container, $callback, array $parameters = [])
{
    $dependencies = [];

    foreach (static::getCallReflector($callback)->getParameters() as $parameter) {
        static::addDependencyForCallParameter($container, $parameter, $parameters, $dependencies);
    }

    
    return array_merge($dependencies, $parameters);
}

// 通过反射得到方法或者函数定义的参数列表
protected static function getCallReflector($callback)
{
    if (is_string($callback) && strpos($callback, '::') !== false) {
        $callback = explode('::', $callback);
    }

    return is_array($callback)
                    ? new ReflectionMethod($callback[0], $callback[1])
                    : new ReflectionFunction($callback); // 这里兼容了函数的依赖注入
}

// 计算方法或者函数的 依赖参数 or $parameters or 本身的默认值
protected static function addDependencyForCallParameter($container, $parameter,
                                                        array &$parameters, &$dependencies)
{
    // 如果 定义的参数列表顺序 指定 `手动参数` 放在`自动参数`前面。
    // $parameters 是关联数组的时候才有用
    if (array_key_exists($parameter->name, $parameters)) {
        $dependencies[] = $parameters[$parameter->name];

        unset($parameters[$parameter->name]);
    } elseif ($parameter->getClass()) {
        // 类的依赖注入
        $dependencies[] = $container->make($parameter->getClass()->name);
    } elseif ($parameter->isDefaultValueAvailable()) {
        // 有默认值
        $dependencies[] = $parameter->getDefaultValue();
    }
}
```

可以使用依赖注入的地方：
* 服务提供者 boot方法
* 路由中指定的控制器的处理方法和控制器的构造函数
* 凡事调用App的 call进行调用的方法或函数都可以使用依赖注入

**注意：**
* `需要注入或有默认值的参数` 这里统一称为`自动参数`;其他为`手动参数`，就是需要显示传递的参数
* 依赖注入不仅仅是把依赖注入到类方法中，还可以注入到函数中
* 最好把`自动参数`放在前面，`手动参数`放后面
* 如果无需注入且无默认值的参数定义在 需要注入或有默认值的前面，那么$parameters需要为关联数组(并且需要与接下来的`手动参数`顺序保持一致)
    - 原因是：`call_user_func_array` 调用的时候参数列表只是按照顺序来传递
   
```php
$array = [
    'param2' => '1111',
    'param1' => '2222'
];
function test($param1, $param2){
    var_dump([$param1, $param2]);
}

call_user_func_array('test', $array);
// 输出：只是按照顺序传递
array(2) {
  [0] =>
  string(4) "1111"
  [1] =>
  string(4) "2222"
}
```

按照 [官网文档](http://php.net/manual/zh/function.call-user-func-array.php) 上说第二个的`$param_arr`需要时索引数组。但是上面的例子使用了一个关联数组也是可以正常运行的；难道是低版本才会有这种要求？这个留给读者去研究了！

### 3.3 管道
> Laravel有管道？什么鬼？怎么还不说`中间件`和`路由`？Laravel的`中间件`和`路由`是通过管道传输的，在传输的过程中就把请求转化为响应。

```php
// 管道就只有四个方法
interface Pipeline
{
    /**
     * Set the traveler object being sent on the pipeline.
     * 把需要加工的东西放入管道
     *
     * @param  mixed  $traveler
     * @return $this
     */
    public function send($traveler);
    
    /**
     * Set the stops of the pipeline.
     * 管道的停留点。就管道流程的加工点
     *
     * @param  dynamic|array  $stops
     * @return $this
     */
    public function through($stops);
    
    /**
     * Set the method to call on the stops.
     * 设置停留点调用的方法名，比如中间件的handle
     *
     * @param  string  $method
     * @return $this
     */
    public function via($method);
    
    /**
     * Run the pipeline with a final destination callback.
     * 运行管道(重点)，并且把结果送到目的地
     *
     * @param  \Closure  $destination
     * @return mixed
     */
    public function then(Closure $destination);
}
```


#### 管道实现中间件栈

![image](https://note.youdao.com/yws/api/personal/file/8CEEDE817C84464F87ECA7DD672FB352?method=download&shareKey=715d761053e41e918dfee64c56fc4877)

为什么叫中间件栈？因为这与栈的思想是一致的。请求最先进入的中间件，最后才会出去：先进后出。则么做到的？使用闭包一层一层地包裹着控制器！！先把`middleware 3` 包裹 `控制器`，然后`middleware 2`再对其进行包装一层，最后包上最后一层`middlware 1`。

看一下管道是怎么使用的：
```
return (new Pipeline($this->app))
    ->send($request)
    ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
    ->then($this->dispatchToRouter());
    
// Pipeline 下：
public function then(Closure $destination)
{
    // array_reduce 是PHP原生的函数，用于数组迭代
    $pipeline = array_reduce(
        // 这里对pipes进行反转 是需要从最后的 middleware 3 开始包裹
        array_reverse($this->pipes), $this->carry(), $this->prepareDestination($destination)
    );
    // 运行得到的闭包栈
    return $pipeline($this->passable);
}

// PHP 版本的实现
function array_reduce($array, $callback, $initial=null)
{
    $acc = $initial;
    foreach($array as $a)
        $acc = $callback($acc, $a);
    return $acc;
}

protected function prepareDestination(Closure $destination)
{
    return function ($passable) use ($destination) {
        return $destination($passable);
    };
}

protected function carry()
{
    // 返回一个处理 $pipes的callback;$stack=上次迭代完成的中间件栈; $pipe 本次迭代的中间件;
    // $passable每一层中间件都需要处理的$request请求
    return function ($stack, $pipe) {
        // 本次迭代 包裹完成的$stack 闭包栈。每一层闭包栈都只有一个 $passable 参数
        return function ($passable) use ($stack, $pipe) {
            if ($pipe instanceof Closure) {
                // 如果管道是一个Closure的实例，我们将直接调用它，否则我们将把这些管道从容器中解出来，并用适当的方法和参数调用它，并返回结果。
                return $pipe($passable, $stack);
            } elseif (! is_object($pipe)) {
                // 字符串形式可以 传递参数
                list($name, $parameters) = $this->parsePipeString($pipe);

                // 如果管道是一个字符串，我们将解析字符串并将该类从依赖注入容器中解析出来。 然后，我们可以构建一个可调用的函数，并执行所需参数中的管道函数。
                $pipe = $this->getContainer()->make($name);

                $parameters = array_merge([$passable, $stack], $parameters);
            } else {
                // 如果管道已经是一个对象，我们只需要调用一个对象，然后将它传递给管道。 没有必要做任何额外的解析和格式化，因为我们提供的对象已经是一个完全实例化的对象。
                $parameters = [$passable, $stack];
            }
            // 运行中间件 参数顺序： $request请求, 下一层闭包栈,自定义参数
            return $pipe->{$this->method}(...$parameters);
        };
    };
}
```

看了源码之后，其实管道并不是真的把中间件闭包一层一层地进行包裹的。而是把闭包连成一个链表!，把相邻的中间件连接起来:
![](https://note.youdao.com/yws/api/personal/file/8800D1B2A040419283FF9F7A7EC1D8FA?method=download&shareKey=abf6f75fd7163ac0c89b0d5e34840351)
middleware 的 $next 参数其实就是 下一个中间件，可以选择在什么时候调用。比如在`middleware`的`handle`方法中：
```
前置操作... 可以进行权限控制
$response = $next($request);
后置操作... 可以统一处理返回格式，比如返回jsonp格式
```

注意：上面使用通过管道经过的中间件是在Http Kernel上配置的必然经过的中间件。但是路由的中间件/控制器中间件呢？接下来就是路由调度的是了！！

### 3.4 路由调度
> 其实上面流程图最后的节点是`控制器`，其实并不是，而是被路由中间件栈`包裹`的控制器。这也是需要使用管道的

```
public function dispatchToRoute(Request $request)
{
    // First we will find a route that matches this request. We will also set the
    // route resolver on the request so middlewares assigned to the route will
    // receive access to this route instance for checking of the parameters.
    // 路由调度是这里
    $route = $this->findRoute($request);

    // 给$request 设置$route实例
    $request->setRouteResolver(function () use ($route) {
        return $route;
    });

    // 事件配发 就是触发某事件
    $this->events->dispatch(new Events\RouteMatched($route, $request));

    // 把请求通过管道(中间件是管道节点) 得到请求 这里就不说了
    $response = $this->runRouteWithinStack($route, $request);
    // 这里只是封装以下请求 不是路由调度的内容
    return $this->prepareResponse($request, $response);
}
```
* 加载路由
    - 等等？什么时候加载路由？这个在App初始化的时候就加载了。不过本文并没有详细说明路由是如何实现：比如路由组是怎么实现？还是使用了栈的原理来保存路由组共有的属性！！
* 匹配路由
    - `Illuminate\Routing\RouteCollection`:所有路由到保存在这里。

```
public function match(Request $request)
{
    $routes = $this->get($request->getMethod());

    // First, we will see if we can find a matching route for this current request
    // method. If we can, great, we can just return it so that it can be called
    // by the consumer. Otherwise we will check for routes with another verb.
    // 翻译：首先，我们将看看我们是否可以找到这个当前请求方法的匹配路由。 如果我们可以的话，那么我们可以把它归还给消费者consumer。 否则，我们将检查另一个verb的路线。
    $route = $this->matchAgainstRoutes($routes, $request);

    // 找到匹配路由
    if (! is_null($route)) {
        return $route->bind($request);
    }

    // If no route was found we will now check if a matching route is specified by
    // another HTTP verb. If it is we will need to throw a MethodNotAllowed and
    // inform the user agent of which HTTP verb it should use for this route.
    // 翻译：如果没有找到路由，我们现在检查一个匹配路由是否由另一个HTTP verb指定。 如果是这样的话，我们需要抛出一个MethodNotAllowed方法，并告知用户代理它应该为这个路由使用哪个HTTP verb。意思是提示：方法不被允许的Http 405 错误
    $others = $this->checkForAlternateVerbs($request);

    if (count($others) > 0) {
        return $this->getRouteForMethods($request, $others);
    }

    throw new NotFoundHttpException;
}

protected function matchAgainstRoutes(array $routes, $request, $includingMethod = true)
{
    // Arr::first 返回提一个符合条件的数组元素
    return Arr::first($routes, function ($value) use ($request, $includingMethod) {
        return $value->matches($request, $includingMethod);
    });
}

// Illuminate\Routing\Route 类的 matches
public function matches(Request $request, $includingMethod = true)
{
    // 
    $this->compileRoute();

    foreach ($this->getValidators() as $validator) {
        if (! $includingMethod && $validator instanceof MethodValidator) {
            continue;
        }
        // 任意一个不符合 就不符合要求
        if (! $validator->matches($this, $request)) {
            return false;
        }
    }

    return true;
}

public static function getValidators()
{
    if (isset(static::$validators)) {
        return static::$validators;
    }

    // To match the route, we will use a chain of responsibility pattern with the
    // validator implementations. We will spin through each one making sure it
    // passes and then we will know if the route as a whole matches request.
    // 翻译：为了匹配路线，我们将使用验证器实现的责任模式链。 我们将通过每一个确保它通过，然后我们将知道是否整个路线符合要求。
    return static::$validators = [
        new UriValidator, new MethodValidator,
        new SchemeValidator, new HostValidator,
    ];
}
```
注意：
* HTTP verb是指http 动词，比如POST，GET等
* 这里不详细说明了，因为没有什么特别的
* 可能重要的一点就是：路由匹配只会使用第一条符合条件的路由。

## 4 请求输出
> Symfony\Component\HttpFoundation\Response

```
public function send()
{
    // 设置 header 和 cookie
    $this->sendHeaders();
    // echo $this->content
    $this->sendContent();

    if (function_exists('fastcgi_finish_request')) {
        fastcgi_finish_request();
    } elseif ('cli' !== PHP_SAPI) {
        // ob_end_flush()
        static::closeOutputBuffers(0, true);
    }

    return $this;
}

public static function closeOutputBuffers($targetLevel, $flush)
{
    // 获取有效的缓冲区嵌套层数
    $status = ob_get_status(true);
    $level = count($status);
    // PHP_OUTPUT_HANDLER_* are not defined on HHVM 3.3
    $flags = defined('PHP_OUTPUT_HANDLER_REMOVABLE') ? PHP_OUTPUT_HANDLER_REMOVABLE | ($flush ? PHP_OUTPUT_HANDLER_FLUSHABLE : PHP_OUTPUT_HANDLER_CLEANABLE) : -1;

    // 一层一层进行清除
    while ($level-- > $targetLevel && ($s = $status[$level]) && (!isset($s['del']) ? !isset($s['flags']) || $flags === ($s['flags'] & $flags) : $s['del'])) {
        if ($flush) {
            ob_end_flush();
        } else {
            ob_end_clean();
        }
    }
}
```

## 5 kernel terminate
> http Kernel的terminate终止。一般情况都是不需要做些什么的，因为很少去设置终止回调

* 先 终止中间件(这里不会使用链表的形式了，直接遍历每个中间件的`terminate`方法)
* 后 终止app程序

```
public function terminate($request, $response)
{
    // 逐个调用中间件的terminate 。没有 $next 参数
    $this->terminateMiddleware($request, $response);

    $this->app->terminate();
}

// App
public function terminate()
{
    // 使用依赖注入形式 调用设置的回调
    foreach ($this->terminatingCallbacks as $terminating) {
        $this->call($terminating);
    }
}
```

**本文还不是还完整：比如请求捕获；路由如何调用控制器等等。之后有时间会继续更新的！！**