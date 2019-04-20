### 路由解析与分发

Lumen中的路由解析为了快速解析响应用户请求，没有使用Symfony的路由组件,而是使用的fastRouter实现路由解析。

#### 路由解析

lumen框架在产生$app的时候会启动加载路由:

```
$app = new Laravel\Lumen\Application(
    dirname(__DIR__)
);
......
//Application的构造函数
public function __construct($basePath = null)
    {
        if (! empty(env('APP_TIMEZONE'))) {
            date_default_timezone_set(env('APP_TIMEZONE', 'UTC'));
        }

        $this->basePath = $basePath;

        $this->bootstrapContainer();
        $this->registerErrorHandling();
        $this->bootstrapRouter();
    }
 ......
    public function bootstrapRouter()
    {
        $this->router = new Router($this);
    }
```

有了 $this->router = new Router($this),Router类是Application类通过trait特性引入的，具体实现的类是Concerns\RoutesRequests,使用四百多行代码实现了路由的解析。

app.php类的最后添加了路由解析逻辑:

```
$app->router->group([
    'namespace' => 'App\Http\Controllers',
], function ($router) {
    require __DIR__ . '/../routes/web.php';
});
```

载入web.php中的代码后，上面代码变成了:

```
$app->router->group([
    'namespace' => 'App\Http\Controllers',
], function ($router) {
    $router->get('/', function () use ($router) {
    return $router->app->version();
    });

    $router->group(['middleware' => 'api'], function () use ($router) {
    $router->get('user/list', 'UserController@getList');
    $router->post('user/change', 'UserController@edit');
    });

    $router->group(['namespace' => '\App\Http\Controllers\Admin', 'prefix' => 'admin'], function () use ($router) {
    $router->addRoute(['POST'], '/auth/login', 'AuthController@login');
    });
});
```

继续看$router的group方法:

```
public function group(array $attributes, \Closure $callback)
    {
        if (isset($attributes['middleware']) && is_string($attributes['middleware'])) {
            $attributes['middleware'] = explode('|', $attributes['middleware']);
        }

        $this->updateGroupStack($attributes);

        call_user_func($callback, $this);

        array_pop($this->groupStack);
    }
```

先判断传入的$attributes是否有中间件middleware,有则解析成数组一并导入到$this->groupStack[]数组中,groupStack数组中key主要有:namespace、prefix、domain、as、suffix。代码很简单，直接贴出来:

```
protected function updateGroupStack(array $attributes)
    {
        if (! empty($this->groupStack)) {
            $attributes = $this->mergeWithLastGroup($attributes);
        }

        $this->groupStack[] = $attributes;
    }

public function mergeGroup($new, $old)
    {
        $new['namespace'] = static::formatUsesPrefix($new, $old);

        $new['prefix'] = static::formatGroupPrefix($new, $old);

        if (isset($new['domain'])) {
            unset($old['domain']);
        }

        if (isset($old['as'])) {
            $new['as'] = $old['as'].(isset($new['as']) ? '.'.$new['as'] : '');
        }

        if (isset($old['suffix']) && ! isset($new['suffix'])) {
            $new['suffix'] = $old['suffix'];
        }

        return array_merge_recursive(Arr::except($old, ['namespace', 'prefix', 'as', 'suffix']), $new);
}

protected function mergeWithLastGroup($new)
{
    return $this->mergeGroup($new, end($this->groupStack));
}
```

然后执行call_user_func($callback, $this),即回调函数。

代码中的get、post函数都是调用框架中的addRoute方法:

```
public function addRoute($method, $uri, $action)
    {
        $action = $this->parseAction($action);

        $attributes = null;

        if ($this->hasGroupStack()) {
            $attributes = $this->mergeWithLastGroup([]);
        }

        if (isset($attributes) && is_array($attributes)) {
            if (isset($attributes['prefix'])) {
                $uri = trim($attributes['prefix'], '/').'/'.trim($uri, '/');
            }

            if (isset($attributes['suffix'])) {
                $uri = trim($uri, '/').rtrim($attributes['suffix'], '/');
            }

            $action = $this->mergeGroupAttributes($action, $attributes);
        }

        $uri = '/'.trim($uri, '/');

        if (isset($action['as'])) {
            $this->namedRoutes[$action['as']] = $uri;
        }

        if (is_array($method)) {
            foreach ($method as $verb) {
                $this->routes[$verb.$uri] = ['method' => $verb, 'uri' => $uri, 'action' => $action];
            }
        } else {
            $this->routes[$method.$uri] = ['method' => $method, 'uri' => $uri, 'action' => $action];
        }
    }
```

我们进一步来解析:

```
protected function parseAction($action)
    {
        if (is_string($action)) {
            return ['uses' => $action];
        } elseif (! is_array($action)) {
            return [$action];
        }

        if (isset($action['middleware']) && is_string($action['middleware'])) {
            $action['middleware'] = explode('|', $action['middleware']);
        }

        return $action;
    }
```

parseAction方法首先分析$action是否有中间件，并解析。然后使用hasGroupStack()方法判断是否解析函数是否在group的堆栈当中,进而进行合并。

```
 public function hasGroupStack()
    {
        return ! empty($this->groupStack);
    }
```
然后判断路由是否有别名,有的话存储到namedRoutes属性中,最后完成路由的解析，存储到变量routes中。我们在app.php最后打印routes信息:$app->router->getRoutes():

```
array:4 [▼
  "GET/" => array:3 [▼
    "method" => "GET"
    "uri" => "/"
    "action" => array:1 [▼
      0 => Closure() {#13 ▶}
    ]
  ]
  "GET/user/list" => array:3 [▼
    "method" => "GET"
    "uri" => "/user/list"
    "action" => array:2 [▼
      "uses" => "App\Http\Controllers\UserController@getList"
      "middleware" => array:1 [▼
        0 => "api"
      ]
    ]
  ]
  "POST/user/change" => array:3 [▼
    "method" => "POST"
    "uri" => "/user/change"
    "action" => array:2 [▼
      "uses" => "App\Http\Controllers\UserController@edit"
      "middleware" => array:1 [▼
        0 => "api"
      ]
    ]
  ]
  "POST/admin/auth/login" => array:3 [▼
    "method" => "POST"
    "uri" => "/admin/auth/login"
    "action" => array:1 [▼
      "uses" => "App\Http\Controllers\Admin\AuthController@login"
    ]
  ]
]
```

#### 路由分发(中间件实现原理)

路由解析完成之后，就需要根据用户的请求获取执行逻辑的相关代码了，index.php中$app->run()的实现就是实现这一功能的：

```
public function run($request = null)
    {
        $response = $this->dispatch($request);

        if ($response instanceof SymfonyResponse) {
            $response->send();
        } else {
            echo (string) $response;
        }

        if (count($this->middleware) > 0) {
            $this->callTerminableMiddleware($response);
        }
    }
```

run方法中根据用户请求分发路由，得到响应信息，返回响应信息给客户端。然后判断是否有后台其他任务，这些任务的处理是在响应给客户端之后进行的。Lumen框架默认处理Response信息的组件是基于SymfonyResponse构建的,我们来看一下发送响应信息给客户端的send方法:

```
public function send()
    {
        $this->sendHeaders();
        $this->sendContent();

        if (\function_exists('fastcgi_finish_request')) {
            fastcgi_finish_request();
        } elseif (!\in_array(\PHP_SAPI, ['cli', 'phpdbg'], true)) {
            static::closeOutputBuffers(0, true);
        }

        return $this;
    }
...
   public function sendHeaders()
    {
        // headers have already been sent by the developer
        if (headers_sent()) {
            return $this;
        }

        // headers
        foreach ($this->headers->allPreserveCaseWithoutCookies() as $name => $values) {
            $replace = 0 === strcasecmp($name, 'Content-Type');
            foreach ($values as $value) {
                header($name.': '.$value, $replace, $this->statusCode);
            }
        }

        // cookies
        foreach ($this->headers->getCookies() as $cookie) {
            header('Set-Cookie: '.$cookie->getName().strstr($cookie, '='), false, $this->statusCode);
        }

        // status
        header(sprintf('HTTP/%s %s %s', $this->version, $this->statusCode, $this->statusText), true, $this->statusCode);

        return $this;
    }

    public function sendContent()
    {
        echo $this->content;

        return $this;
    } 
```

返回给客户端的信息分为Header和Content两部分，程序还会判断是否存在fastcgi_finish_request函数，如果存在，会调用fastcgi_finish_request发送请求给客户端并断开链接。这个时候服务端的程序还正在运行，Lumen框架会判断有没有需要执行的服务端任务然后继续执行，进程中的环境变量和数据也都存在，可以利用这个机制做处理异步脚本之类的事情，但是对内存的管理不太好把控。

回到主题，我们再来看dispatch方法是通过路由分发请求给指定的控制器方法的:

```
public function dispatch($request = null)
    {
        list($method, $pathInfo) = $this->parseIncomingRequest($request);

        try {
            $this->boot();

            return $this->sendThroughPipeline($this->middleware, function ($request) use ($method, $pathInfo) {
                $this->instance(Request::class, $request);

                if (isset($this->router->getRoutes()[$method.$pathInfo])) {
                    return $this->handleFoundRoute([true, $this->router->getRoutes()[$method.$pathInfo]['action'], []]);
                }

                return $this->handleDispatcherResponse(
                    $this->createDispatcher()->dispatch($method, $pathInfo)
                );
            });
        } catch (Exception $e) {
            return $this->prepareResponse($this->sendExceptionToHandler($e));
        } catch (Throwable $e) {
            return $this->prepareResponse($this->sendExceptionToHandler($e));
        }
    }
```

程序先调用parseIncomingRequest方法得到用户请求的方法（get、post等）和请求路径信息:

```
protected function parseIncomingRequest($request)
    {
        if (! $request) {
            $request = LumenRequest::capture();
        }

        $this->instance(Request::class, $this->prepareRequest($request));

        return [$request->getMethod(), '/'.trim($request->getPathInfo(), '/')];
    }
```

lumen用户默认创建了自己的的$request实例并绑定到了容器。程序接下来调用了boot方法注册启动了用户添加的所有服务:

```
 public function boot()
    {
        if ($this->booted) {
            return;
        }

        array_walk($this->loadedProviders, function ($p) {
            $this->bootProvider($p);
        });

        $this->booted = true;
    }
...
protected function bootProvider(ServiceProvider $provider)
    {
        if (method_exists($provider, 'boot')) {
            return $this->call([$provider, 'boot']);
        }
    }
```

有了method，有了请求uri，我们就可以根据之前解析出来的路由信息，利用服务容器make(解析)得到服务，执行逻辑代码，得到响应信息了。

#### 中间件实现原理
服务容器在解析服务时会根据路由设置的中间件，构建一个洋葱式的程序调用栈，lumen的中间件的实现原理是**管道(Pipeline)**。管道实现的核心是array_reduce函数，我们来看一个例子：

```
<?php
$a = array(1, 2, 3, 4, 5);
function sum($carry, $item)
{
    $carry += $item;
    return $carry;
}
var_dump(array_reduce($a, "sum"));
var_dump(array_reduce($a, "sum", 10));
```

以上程序会输出15 、25

官网上对array_reduce函数的介绍是:

array_reduce — 用回调函数迭代地将数组简化为单一的值

说明 ¶
array_reduce ( array $array , callable $callback [, mixed $initial = NULL ] ) : mixed
array_reduce() 将回调函数 callback 迭代地作用到 array 数组中的每一个单元中，从而将数组简化为单一的值。

参数 ¶
array
输入的 array。

callback
callback ( mixed $carry , mixed $item ) : mixed
carry
携带上次迭代里的值； 如果本次迭代是第一次，那么这个值是 initial。

item
携带了本次迭代的值。

initial
如果指定了可选参数 initial，该参数将在处理开始前使用，或者当处理结束，数组为空时的最后一个结果。

试想我们如果将数组中的值设置为匿名对象实例,并通过参数传递一个实例($initial参数)经过这些匿名对象实例，不就能构建一个洋葱式的程序调用栈了吗？实际上lumen框架也是这么做的。下边我们看一下源码,重点是sendThroughPipeline方法:

```
 protected function sendThroughPipeline(array $middleware, Closure $then)
    {
        if (count($middleware) > 0 && ! $this->shouldSkipMiddleware()) {
            return (new Pipeline($this))
                ->send($this->make('request'))
                ->through($middleware)
                ->then($then);
        }

        return $then($this->make('request'));
    }           
```

程序判断是否有中间件并且中间件不应该被忽略,执行:

```
(new Pipeline($this))
                ->send($this->make('request'))
                ->through($middleware)
                ->then($then);
```

否则就执行$then($this->make('request'));即匿名函数，参数是从服务容器解析出来的request组件。我们考虑有中间件的情况，接着看源码:

```
public function send($passable)
    {
        $this->passable = $passable;

        return $this;
    }
......
public function through($pipes)
{
    $this->pipes = is_array($pipes) ? $pipes : func_get_args();

    return $this;
}
......
public function then(Closure $destination)
    {
        $pipeline = array_reduce(
            array_reverse($this->pipes), $this->carry(), $this->prepareDestination($destination)
        );

        return $pipeline($this->passable);
    }
```

send和through方法都返回$this，可以链式调用。send方法将要发送的参数存储$this->passable成员变量中，这里指的是从服务容器解析出来的request服务；through是将调用栈（这里指中间件存储到$this->pipes成员变量中）;接下来的then方法是重点，调用array_reduce方法根据上述参数构建程序执行栈,重点是carry方法:

```
protected function carry()
    {
        return function ($stack, $pipe) {
            return function ($passable) use ($stack, $pipe) {
                if (is_callable($pipe)) {
                    return $pipe($passable, $stack);
                } elseif (! is_object($pipe)) {
                    [$name, $parameters] = $this->parsePipeString($pipe);

                    $pipe = $this->getContainer()->make($name);

                    $parameters = array_merge([$passable, $stack], $parameters);
                } else {
                    instantiated object.
                    $parameters = [$passable, $stack];
                }

                $response = method_exists($pipe, $this->method)
                                ? $pipe->{$this->method}(...$parameters)
                                : $pipe(...$parameters);

                return $response instanceof Responsable
                            ? $response->toResponse($this->getContainer()->make(Request::class))
                            : $response;
            };
        };
    }
......
protected function prepareDestination(Closure $destination)
    {
        return function ($passable) use ($destination) {
            return $destination($passable);
        };
    }
```

程序判断当前要执行的实例$pipe是方法还是对象，如果是方法直接调用执行，如果是对象，则调用对象的handle方法，将$passable和$stack作为参数传入其中。我们使用下面例子简化一下上边构建洋葱式程序栈的过程:

```
<?php

/**
 * Description: File basic description here...
 * User: guozhaoran<guozhaoran@cmcm.com>
 * Date: 2019-02-15
 */
interface Step
{
    public static function go(Closure $next);
}

class FirstStep implements Step
{
    public static function go(Closure $next)
    {
        echo "Start FirstStep" . '<br/>';
        $next();
        echo "End FirstStep" . '<br/>';
    }
}

class SecondStep implements Step
{
    public static function go(Closure $next)
    {
        echo "Start SecondStep" . '<br/>';
        $next();
        echo "End SecondStep" . '<br/>';
    }
}

class ThirdStep implements Step
{
    public static function go(Closure $next)
    {
        echo "Start ThirdStep" . '<br/>';
        $next();
        echo "End ThirdStep" . '<br/>';
    }
}

function goFunc($step, $className)
{
    return function () use ($step, $className) {
        return $className::go($step);
    };
}

function then()
{
    $steps = ["FirstStep", "SecondStep", "ThirdStep"];
    $prepare = function () {
        echo "请求向路由传递,返回响应" . '<br/>';
    };
    $go = array_reduce($steps, "goFunc", $prepare);
    $go();
}

then();
```

以上程序的输入结果是:
Start ThirdStep
Start SecondStep
Start FirstStep
请求向路由传递,返回响应
End FirstStep
End SecondStep
End ThirdStep

上述简单例子中$prepare函数作为goFunc函数的第一个迭代参数执行，返回一个闭包函数，然后第二层、第三层迭代，最终返回一个洋葱迭代栈闭包。调用函数就达到了预期的响应结果。

到这里，Lumen框架实现中间件的原理已经基本明了了。路由解析完成之后，根据用户请求，服务容器解析Controller的入口函数为一个实例；调用Pipeline方法构造一个洋葱式调用栈。调用次函数，得到用户响应结果，然后经过加工返回给用户。一次request -> response 就是这样完成的。