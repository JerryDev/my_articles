---
title: Lumen源码分析之门面模式
date: 2017-07-18 22:57:59
tags: Lumen
---

最近项目用Lumen来做接口开发，主要看重Lumen比Laravel更轻量，更适合做接口开发。

Laravel的源代码平时用的时候也经常去翻，但总有种云里雾里的感觉，主要是Laravel运用了很多设计模式，还有许多PHP的OOP特性，所以看起来略显繁复。

刚好趁着应用Lumen，就以Lumen来分析一下其所用的门面模式（Facade pattern）。

Lumen的代码比Laravel简洁很多，砍掉了很多自定义配置，还替换了路由的包，以此来获得跟快的运行速度。

<!--more-->

从入口文件开始。

public/index.php

```php
$app = require __DIR__.'/../bootstrap/app.php';

$app->run();
```

可以看到，入口文件只是引用了bootstrap下的app.php文件，通过返回值来看，是返回了一个对象，如此，下面就直接运行了这个对象run方法。我们再来看看bootstrap/app.php文件。

```php
// 首先引入composer提供的自动加载器
require_once __DIR__.'/../vendor/autoload.php';

// 解析并注入配置信息
try {
    (new Dotenv\Dotenv(__DIR__.'/../'))->load();
} catch (Dotenv\Exception\InvalidPathException $e) {
    //
}

/* Application就是整个框架的核心，也是一个容器，里面保存有整个框架的很多实例对象
 * 我们通过 app()  app()->make() 获得对象都是从这里来的
 */
$app = new Laravel\Lumen\Application(
    realpath(__DIR__.'/../')
);

// 引入门面模式，默认没开启，也是为了提升运行速度，不过个人觉得开启的话方便些
// $app->withFacades();

// 引入ORM，也就是对象关系映射，Models类映射到数据库表进行操作，会影响运行速度，个人还是习惯用DB操作去实现业务逻辑，很少用ORM，所以就不开启
// $app->withEloquent();

// 单例模式 注入异常处理类，所以用Lumen可以直接抛出异常，由Lumen框架的这个类统一处理
$app->singleton(
    Illuminate\Contracts\Debug\ExceptionHandler::class,
    App\Exceptions\Handler::class
);

// 单例模式 注入命令行运行所需类
$app->singleton(
    Illuminate\Contracts\Console\Kernel::class,
    App\Console\Kernel::class
);

// 中间件
// $app->middleware([
//    App\Http\Middleware\ExampleMiddleware::class
// ]);

// 路由中间件
// $app->routeMiddleware([
//     'auth' => App\Http\Middleware\Authenticate::class,
// ]);

// 一些ServiceProvider
// $app->register(App\Providers\AppServiceProvider::class);
// $app->register(App\Providers\AuthServiceProvider::class);
// $app->register(App\Providers\EventServiceProvider::class);

/* 
 * 这里就是读取路由文件里定义的路由列表，保存到Application对象中
 * 后续就会开始处理http请求，从路由列表中匹配去运行相应的控制器方法
 */
$app->group(['namespace' => 'App\Http\Controllers'], function ($app) {
    require __DIR__.'/../routes/web.php';
});

return $app;
```

大致介绍了一下Lumen的加载过程，本次我们主要看Lumen里的门面模式

就是下面这句代码：
```php
$app->withFacades();
```

我们可以跟进去看看代码
```php
/**
 * Register the facades for the application.
 *
 * @param  bool  $aliases
 * @param  array $userAliases
 * @return void
 */
public function withFacades($aliases = true, $userAliases = [])
{
    Facade::setFacadeApplication($this);

    if ($aliases) {
        $this->withAliases($userAliases);
    }
}
```

可以看到，此方法做了两件事：
把 Application 对象注入Facade类
执行 withAliases() 方法

我们再来看看withAliases() 干了啥事
```php
/**
 * Register the aliases for the application.
 *
 * @param  array  $userAliases
 * @return void
 */
public function withAliases($userAliases = [])
{
    $defaults = [
        'Illuminate\Support\Facades\Auth' => 'Auth',
        'Illuminate\Support\Facades\Cache' => 'Cache',
        'Illuminate\Support\Facades\DB' => 'DB',
        'Illuminate\Support\Facades\Event' => 'Event',
        'Illuminate\Support\Facades\Gate' => 'Gate',
        'Illuminate\Support\Facades\Log' => 'Log',
        'Illuminate\Support\Facades\Queue' => 'Queue',
        'Illuminate\Support\Facades\Schema' => 'Schema',
        'Illuminate\Support\Facades\URL' => 'URL',
        'Illuminate\Support\Facades\Validator' => 'Validator',
    ];

    if (! static::$aliasesRegistered) {
        static::$aliasesRegistered = true;

        $merged = array_merge($defaults, $userAliases);

        foreach ($merged as $original => $alias) {
            class_alias($original, $alias);
        }
    }
}
```

哦，原来是定义了一些别名，门面模式的目的就是简化、方便使用
这里就定义了一个门面列表，有常用的DB、Schema、Log等

到这里肯定是还不够的，我们只是看到了表象，还不知道其内部到底是如何实现的，我们继续翻源码

先拿DB来举例，通过这里的别名列表我们知道DB实际上映射到了 Illuminate\Support\Facades\DB

打开这个文件
```php
<?php

namespace Illuminate\Support\Facades;

/**
 * @see \Illuminate\Database\DatabaseManager
 * @see \Illuminate\Database\Connection
 */
class DB extends Facade
{
    /**
     * Get the registered name of the component.
     *
     * @return string
     */
    protected static function getFacadeAccessor()
    {
        return 'db';
    }
}
```

瞬间懵了，什么鬼，就一个方法，还是只返回一个'db'字符串的方法，接着找

我们看到这个类继承了Facade类，so，我们继续翻看Facade的代码

Facade类里也没有DB::table()这种方法，但是，我们找到了魔术方法

```php
public static function __callStatic($method, $args)
{
    $instance = static::getFacadeRoot();

    if (! $instance) {
        throw new RuntimeException('A facade root has not been set.');
    }

    switch (count($args)) {
        case 0:
            return $instance->$method();
        case 1:
            return $instance->$method($args[0]);
        case 2:
            return $instance->$method($args[0], $args[1]);
        case 3:
            return $instance->$method($args[0], $args[1], $args[2]);
        case 4:
            return $instance->$method($args[0], $args[1], $args[2], $args[3]);
        default:
            return call_user_func_array([$instance, $method], $args);
    }
}
```
这个魔术方法会调用 getFacadeRoot() 获得实例，我们跟进去看看

```php
public static function getFacadeRoot()
{
    return static::resolveFacadeInstance(static::getFacadeAccessor());
}

protected static function getFacadeAccessor()
{
    throw new RuntimeException('Facade does not implement getFacadeAccessor method.');
}

protected static function resolveFacadeInstance($name)
{
    if (is_object($name)) {
        return $name;
    }

    if (isset(static::$resolvedInstance[$name])) {
        return static::$resolvedInstance[$name];
    }

    return static::$resolvedInstance[$name] = static::$app[$name];
}
```

重点就是这三个方法了，首先调用getFacadeAccessor()，我们看到在这里是抛出异常，但是这个方法我们在DB里看到被重写了，返回了要给'db'字符串

然后调用resolveFacadeInstance()，传进来一个'db'字符串，我们看到代码里，首先看Facade里有没有存储这个实例，没有的话就去Application对象里去拿实例，so，这里也可以知道Application已经保存了所有Facade需要的实例。

返回实例以后，看上面的魔术方法，就会去调用此实例相应的方法，到这里，门面模式的运行过程就剖析完了。

我们也得出个结论，Application类是Lumen框架的核心，保存了所有用过的实例，这样我们就不用每次都去new Class()，只需要app() app()->make()，获得实例即可。

这里也看得出来，启用了门面模式，用起来比较方便，稍微牺牲了一点性能，具体性能却没有测试过，这就看开发的时候如何取舍了。

