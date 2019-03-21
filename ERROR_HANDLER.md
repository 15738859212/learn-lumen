### 错误处理

Lumen框架设计的初衷是：所有的PHP错误都应该被处理，包括Notice和Warning。真实的生产环境中，我们不会将代码设置的这么严格，出现Notice或者Warning时，只需要记录到日志就可以了。另外Lumen自带的错误信息展示带有模版，在浏览器页面上比较直观；但在前后端分离的项目开发中，错误信息不简洁，往往需要用户自定义。

#### 一、php7错误处理

php7改变了大多数错误的报告方式,不同于传统(php5)的错误报告机制,现在大多数错误被作为Error异常抛出。

这种 Error 异常可以像 Exception 异常一样被第一个匹配的 try / catch 块所捕获。如果没有匹配的 catch 块，则调用异常处理函数（事先通过 set_exception_handler() 注册）进行处理。 如果尚未注册异常处理函数，则按照传统方式处理：被报告为一个致命错误（Fatal Error）。

Error 类并非继承自 Exception 类，所以不能用 catch (Exception $e) { ... } 来捕获 Error。你可以用 catch (Error $e) { ... }，或者通过注册异常处理函数（set_exception_handler()）来捕获 Error。

Error的层次结构:

- Throwable
    - Error
        - ArithmeticError
            - DivisionByZeroError
        - AssertionError
        - ParseError
        - TypeError
    - Exception
        - ......

#### 二、无错的运行容器

E_PARSE对大家来说既熟悉又陌生，在程序中经常会遇到(框架都有此类别的错误提示),如果想要捕获这种错误，首先要理解E_PARSE错误发生的时段。

E_PARSE：即语法解析错误，Syntax Error then Parse Error，PHP 将脚本载入 Zend Engine 后，最开始要做的就是检查基本语法是否有误，无误才会调用解释器，一行行的开始解释运行。

```
<?php
//认为关闭了错误捕获 不应该有错误报出才对
error_reporting(0);

echo 'Syntax Error then Parse Error'
```

上边的程序，关闭了错误提示，还是会报错。代码的确正确的书写了关闭错误提示的意图。但还没被执行到时，脚本就因为最初始的语法解析错误，被 Zend Engine 抛出 Parse Error 终止执行了。

同时还要理解，PHP include / require 只有在真的解释到这一行代码时，引用的文件才会被载入--解析--解释执行。举个例子:

```
<?php
//开启所有的错误提示
error_reporting(-1);
//将错误信息显示在页面上
ini_set('display_errors', 'ON');

$a()
```

上述代码中对错误处理的设置并不会生效，因为Zend Engine 在检查语法阶段，就因为E_PARSE而终止程序的执行了。如果使用include引入错误代码逻辑，则错误处理的设置就会生效:

```
<?php
//开启所有的错误提示
error_reporting(-1);
//将错误信息显示在页面上
ini_set('display_errors', 'ON');

include 'error_file.php';
```

error_file.php中有错误发生:

```
<?php
$a()
```

所以对于框架而言，需要一个无错误的运行容器，而用户的逻辑是被include/require到容器中执行的，并且在用户逻辑的上层使用try...catch...优雅的捕捉。Lumen框架的源码也是这样做的,先从脚本的入口文件分析:

```
<?php
$app = require __DIR__.'/../bootstrap/app.php';

//框架启动
$app->run();
```

我们找到框架启动方法实现部分:

```
public function run($request = null)
    {
        $response = $this->dispatch($request);

        if ($response instanceof SymfonyResponse) {
            $response->send();
        } else {
            echo (string)$response;
        }

        if (count($this->middleware) > 0) {
            $this->callTerminableMiddleware($response);
        }
    }
```

Lumen根据用户请求，通过dispatch分发给相应的路由方法处理，返回响应信息给客户端，之后做了一些其他工作。我们主要看dispatch方法的实现:

```
public function dispatch($request = null)
    {
        list($method, $pathInfo) = $this->parseIncomingRequest($request);

        try {
            $this->boot();
            return $this->sendThroughPipeline($this->middleware, function ($request) use ($method, $pathInfo) {
                $this->instance(Request::class, $request);

                if (isset($this->router->getRoutes()[$method . $pathInfo])) {
                    return $this->handleFoundRoute([true, $this->router->getRoutes()[$method . $pathInfo]['action'], []]);
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

dispatch方法先调用parseIncomingRequest方法对请求进行解析，接下来使用try...catch...对用户的逻辑进行包装。这样用户实现逻辑部分产生的几乎所有错误或异常都可以被框架捕捉到。这里的catch分别使用了Exception、Throwable分别捕捉用户逻辑中产生的异常和错误。而Lumen框架本身也是一个无错的运行容器。

#### 三、源码解析

Lumen容器本身的错误处理是在全局对象$app构建时:

```
$app = new Laravel\Lumen\Application(
    dirname(__DIR__)
);

//...Application对象的构造方法
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
```

实现逻辑在registerErrorHandling函数中:

```
 protected function registerErrorHandling()
    {
        error_reporting(-1);

        set_error_handler(function ($level, $message, $file = '', $line = 0) {
            if (error_reporting() & $level) {
                throw new ErrorException($message, 0, $level, $file, $line);
            }
        });

        set_exception_handler(function ($e) {
            $this->handleUncaughtException($e);
        });

        register_shutdown_function(function () {
            $this->handleShutdown();
        });
    }
```

程序调用了error_reporting()函数，设置所有的错误都被PHP标准错误处理监听。error_reporting函数的设置和读取只对PHP标准错误处理监听有效。接下来分别使用set_error_handler和set_exception_handler设置了全局用户自定义用户处理。下面分别介绍这两个函数:

+ set_error_handler 并非可以捕获所有错误，且 set_error_handler 不会终止程序继续执行。处理后若返回 false，则错误会被继续递交给 PHP 标准错误处理 流程。可以捕获： E_WARNING & E_NOTICE & E_DEPRCATED & E_USER_* & 部分 E_STRICT 级的错误。无法捕获： E_ERROR & E_PARSE & E_CORE_* & E_COMPLIE_* 级的错误。有自身的错误捕获级别，默认E_ALL | E_STRICT，且不受 error_reporting 设定的级别的影响。这里要理解，用户自定义错误处理 和 PHP 标准错误处理 是两层错误捕捉器，有独立的捕获级别。

+ set_exception_handler 用户自定义捕获异常 handler，异常没有被 try ... catch 捕获处理的话会被抛出，此时系统会检查上下文是否注册了 set_exception_handler。
如果未注册 则进入 PHP 标准错误处理 致命错误退出执行。
如果已注册 则进入 set_exception_handler 处理 程序依然会退出执行。
而 try ... catch ... 捕获异常后仍不会退出执行，故强烈建议将有异常的执行逻辑放入 try ... catch 中

紧接着，程序注册了register_shutdown_function函数，在程序执行完成时，如果还有错误未被捕获，则交给回调函数进行处理，这是Lumen错误处理的最后一道关卡。诸如：php执行超时、内存溢出等异常情况，都会通过此处的回调函数进行处理。

Lumen框架默认的错误处理类是Laravel\Lumen\Exceptions\Handler

```
protected function resolveExceptionHandler()
    {
        if ($this->bound('Illuminate\Contracts\Debug\ExceptionHandler')) {
            return $this->make('Illuminate\Contracts\Debug\ExceptionHandler');
        } else {
            return $this->make('Laravel\Lumen\Exceptions\Handler');
        }
    }
```

该方法中有两个重要的方法，分别是:report和render，其中report方法是将错误信息记录到日志；render方法中复用了SymfonyExceptionHandler中的组件方法，将错误信息更加细化，然后匹配相关的模版将错误信息展示。

```
public function report(Exception $e)
    {
        if ($this->shouldntReport($e)) {
            return;
        }

        try {
            $logger = app('Psr\Log\LoggerInterface');
        } catch (Exception $ex) {
            throw $e; // throw the original exception
        }

        $logger->error($e, ['exception' => $e]);
    }

...

public function render($request, Exception $e)
    {
        if ($e instanceof HttpResponseException) {
            return $e->getResponse();
        } elseif ($e instanceof ModelNotFoundException) {
            $e = new NotFoundHttpException($e->getMessage(), $e);
        } elseif ($e instanceof AuthorizationException) {
            $e = new HttpException(403, $e->getMessage());
        } elseif ($e instanceof ValidationException && $e->getResponse()) {
            return $e->getResponse();
        }

        $fe = FlattenException::create($e);

        $handler = new SymfonyExceptionHandler(env('APP_DEBUG', config('app.debug', false)));

        $decorated = $this->decorate($handler->getContent($fe), $handler->getStylesheet($fe));

        $response = new Response($decorated, $fe->getStatusCode(), $fe->getHeaders());

        $response->exception = $e;

        return $response;
    }  
```

如果想对框架抛出的错误信息改写，可以直接在render方法中改造重写。

```
public function render($request, Exception $e)
    {
        // 404 not found
        if ($e instanceof NotFoundHttpException) {
            return response('404. Not Found', 404);
        }
        // 403 forbidden
        if ($e instanceof MethodNotAllowedHttpException) {
            return response('403. Forbidden', 403);
        }

        // 本地和测试环境直接输出错误
        if (app()->environment('local')) {
            return parent::render($request, $e);
        }

        // SQL Duplicate Unique Key
        if ($e instanceof QueryException && $e->getCode() == 23000) {
            $data = ['code' => 23000, 'msg' => '新增重复数据'];
            return response()->json($data);
        }

        if (app()->environment('qa')) {
            $code = $e->getCode() ?? -1;
            $data = ['code' => $code, 'msg' => $e->getMessage(), 'trace' => $this->getTopTrace($e)];
            return response()->json($data);
        }

        // 自定义错误信息展示出来
        if ($e instanceof CustomException) {
            $data = ['code' => $e->getCode(), 'msg' => $e->getMessage()];
            return response()->json($data);
        }

        // 未知的错误信息
        $data = ['code' => -1, 'msg' => 'sorry, error occurred.'];
        return response()->json($data);
    }
```

总结：

+ PHP 将脚本载入 Zend Engine 后，最开始要做的就是检查基本语法是否有误，无误才会调用解释器，一行行的开始解释运行。PHP include / require 只有在真的解释到这一行代码时，引用的文件才会被载入--解析--解释执行。

+ Lumen框架本身是一个无错的容器，用户逻辑处理部分使用try...catch...包裹，基本可以捕获大多数异常和错误。

+ set_error_handler可以捕获E_WARNING & E_NOTICE & E_DEPRECATED & E_USER_* 和 部分 E_STRICT 级的错误。set_error_handler 如果返回了 false 错误会递交给 PHP 标准错误处理。set_error_handler 不会终止程序执行。set_exception_handler 用户自定义异常捕获，捕获后程序依然会终止运行，但不会再将异常递交给 PHP 标准异常处理。

+ Lumen框架容器中使用registerErrorHandling设置了全局用户自定义错误、异常处理程序。用来处理没有被try...catch...捕捉到的错误，另外程序使用register_shutdown_function注册了回调函数，这是错误处理的最后一道关卡，用来捕捉用户自定义错误处理程序和try...catch...未能捕捉到的异常情况，例如：执行超时或内存溢出。