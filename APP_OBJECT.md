### $app对象简介

Lumen加载完配置文件到全局环境变量以后，紧接着开始实例化全局服务对象$app:

```
$app = new Laravel\Lumen\Application(
    dirname(__DIR__)
);
```

$app是唯一贯穿Lumen框架生命周期的对象，包含了处理请求过程中需要的所有服务，例如：错误处理、路由解析、注册服务、生产对象等等。$app对象实现的每一个功能模块都值得深入研究，本节先从全局架构的角度来分析$app对象的设计组成。之后章节再对其重要功能模块实现源码做深入研究。

接着看Application对象的构造方法：

```
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

Application类在实例化的过程中,做了五件事:

1. 设置时区
2. 设置项目根路径
3. 初始化容器
4. 注册自定义错误处理模块
5. 加载路由对象

Application类继承自Container类。“服务容器”是$app对象的最核心功能，它不仅装载了框架在处理请求中使用到的所有服务，还解决了容器内部服务之间的依赖关系，另外它还提供了构建对象、解析对象的功能，是整个框架的发动机。我们再来看看初始化容器做了哪些工作：

```
protected function bootstrapContainer()
    {
        static::setInstance($this);

        $this->instance('app', $this);
        $this->instance(self::class, $this);

        $this->instance('path', $this->path());

        $this->instance('env', $this->environment());

        $this->registerContainerAliases();
    }
```

初始化容器的时候，程序现将当前对象注册为单例，保存到容器对象的$instance静态成员变量中。然后将'app'、自身路径（Laravel\Lumen\Application）都注册到容器中，添加到单例数组中；将当前路径path，当前环境也都添加到容器中的单例类型中，最后registerContainerAliases方法设置容器初始的一些“服务别名”,具体容器中的“服务别名”是什么,怎么用,在服务容器章节会详细说明。

Application类除了拥有“服务容器”的强大功能外，还有路由解析和错误处理相关的功能。php是单继承语言，无法同时继承两个父类。php的Traits和Go语言的组合功能类似，通过在类中使用use关键字声明要组合的Trait名称，而具体某个Trait的声明使用trait关键词，Trait不能直接实例化。Application类中提供路由解析和错误处理相关的功能由两个Trait提供的方法做支持：

```
class Application extends Container
{
  use Concerns\RoutesRequests,
        Concerns\RegistersExceptionHandlers;
    ......
```

Trait为程序模块化设计提供了更小粒度的代码复用和更灵活的功能组合方式。当然Trait也有自己的使用场景，如果类与类之间存在父类和子类的关系，当然还是使用继承比较好。这里Application类使用的就恰到好处，因为路由解析和错误处理只是它两个功能模块而已，它最主要的功能还是要实现“服务容器”。

程序接下来注册自定义错误处理后边章节会详细介绍，加载路由对象也只是实例化路由处理对象并赋值给自身属性变量：

 ```
public function bootstrapRouter()
    {
        $this->router = new Router($this);
    }
 ```

 Application类包含很多实用的方法，例如获取框架资源文件存储路径、注册各种服务、解析数据库ORM类库等等，但是注册服务、解析类库等功能还是要依赖强大的容器来完成。

 可以总结一下$app对象的设计组成了：$app对象最主要的功能是实现了“服务容器”，另外还提供了处理请求中路由解析、错误处理、服务注册等所有使用到的功能。

 总结：

 + “服务容器”是Lumen框架提供服务的关键：它不仅装载了框架在处理请求中使用到的所有服务，还解决了容器内部服务之间的依赖关系，提供了构建、解析对象的强大功能。

 + php中的Trait为程序模块化设计提供了更小粒度的代码复用和更灵活的功能组合方式。

 + $app对象是Lumen框架全局唯一的服务对象，它最主要实现了服务容器的功能，另外提供了处理请求中使用到的其他功能。