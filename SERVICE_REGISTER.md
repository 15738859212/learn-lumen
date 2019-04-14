### 服务注册

Lumen框架拥有非常强大的容器，在框架启动的过程中，可以通过**服务提供者**向容器中注册服务，开发者的应用以及所有Lumen的核心服务都是通过服务提供者启动，服务提供者是Lumen框架的配置中心。服务提供者通过$app->register()进行服务注册。

所有的服务提供者继承自Illuminate\Support\ServiceProvider类。继承该抽象类要求至少在服务提供者中定义一个方法：register。在register方法内，你唯一要做的事情就是绑定依赖的服务到服务容器，不要尝试在其中注册任何事件监听器，路由或者任何其它功能：否则有可能会使用到尚未注册到服务容器的功能而造成错误。

注册服务容器的代码在app.php中有示例:

```
$app->register(App\Providers\AppServiceProvider::class);
$app->register(App\Providers\AuthServiceProvider::class);
$app->register(App\Providers\EventServiceProvider::class);
```

我们来看一下register方法:

```
public function register($provider)
    {
        if (! $provider instanceof ServiceProvider) {
            $provider = new $provider($this);
        }

        if (array_key_exists($providerName = get_class($provider), $this->loadedProviders)) {
            return;
        }

        $this->loadedProviders[$providerName] = $provider;

        if (method_exists($provider, 'register')) {
            $provider->register();
        }

        if ($this->booted) {
            $this->bootProvider($provider);
        }
    }
```

服务提供者注册的服务，一般都是ServiceProvider的子类。已经注册的服务会存储到loadProvider数组中，如果服务存在register方法，则会调用该方法注入准备服务所需要的依赖。$booted属性控制服务是在注册时直接启动还是服务注册完成后启动，默认为false,所有服务的boot方法会在服务注册完成后启动，源码如下:

```
public function dispatch($request = null)
    {
        list($method, $pathInfo) = $this->parseIncomingRequest($request);

        try {
            $this->boot();
            ......
        } catch (Exception $e) {
            ......
        } catch (Throwable $e) {
            ......
        }
    }
```

boot方法就是循环loadProvider数组中的服务，并调用每一个服务的boot方法：

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
......
protected function bootProvider(ServiceProvider $provider)
    {
        if (method_exists($provider, 'boot')) {
            return $this->call([$provider, 'boot']);
        }
    }
```

服务注册中添加的服务非常适合做一些用户验证、事件监听等工作。比如可以监听用户逻辑中的sql语句，将每一条执行的sql都记录到日志；查看请求接口中是否带有token；做HTTP基本验证等。