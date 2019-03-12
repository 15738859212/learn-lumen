### 控制反转(Ioc)和依赖注入(DI)

**控制反转**是框架设计的一种原则，在很大程度上降低了代码模块之间的耦合度，有利于框架维护和拓展。实现控制反转最常见的方法是“依赖注入”，还有一种方法叫“依赖查找”。控制反转将框架中解决依赖的逻辑从实现代码类库的内部提取到了外部来管理实现。

我们用简单代码模拟一下Lumen处理用户请求的逻辑，框架中要使用到最简单的Request请求模块、Response请求模块,我们使用单例模式简单实现一下:

```
//Request模块实现
class Request
{
    static private $instance = null;

    private function __construct()
    {
    }

    private function __clone()
    {
    }

    static function getInstance()
    {
        if (self::$instance == null) self::$instance = new self();
        return self::$instance;
    }

    public function get($key)
    {
        return $_GET[$key] ? $_GET[$key] : '';
    }

    public function post($key)
    {
        return $_POST[$key] ? $_POST[$key] : '';
    }
}

//Response模块实现
class Response
{
    static private $instance = null;

    private function __construct()
    {
    }

    private function __clone()
    {
    }

    static function getInstance()
    {
        if (self::$instance == null) self::$instance = new self();
        return self::$instance;
    }

    public function json($data)
    {
        return json_encode($data);
    }
}
```

我们先来使用“依赖查找”的工厂模式来实现控制反转,我们需要一个工厂，简单实现一下：

```
include_once 'Request.php';
include_once 'Response.php';
include_once 'ExceptionHandler.php';

abstract class Factory
{
    static function Create($type, array $params = [])
    {
        //根据接收到的参数确定要生产的对象
        switch ($type) {
            case 'request':
                return Request::getInstance();
                break;
            case 'response':
                return Response::getInstance();
                break;
            case 'exception':
                return new ExceptionHandler();
                break;
        }
    }
}
```

接下来就开始实现用户逻辑，我们首先加入错误处理的简单实现:

```
//开启报告代码中的错误处理
class ExceptionHandler
{
    public function __construct()
    {
        error_reporting(-1);
        ini_set('display_errors', true);
    }
}
```

我们模拟一个请求用户列表的逻辑:

```
include_once 'Factory.php';

Factory::Create('exception');

//用户逻辑
class UserLogic
{
    private $modules = [];

    public function __construct(array $modules)
    {
        foreach ($modules as $key => $module) {
            $this->modules[$key] = Factory::Create($module);
        }
    }

    public function getUserList()
    {
        if ($this->modules['request']->get('path') == 'userlist') {
            $userList = [
                ['name' => '张三', 'age' => 18],
                ['name' => '李四', 'age' => 22]
            ];

            return $this->modules['response']->json($userList);
        }
    }
}

try {
    $userLogic = new UserLogic(['request' => 'request', 'response' => 'response']);
    echo $userLogic->getUserList();
} catch (\Error $e) {
    var_dump($e);
    exit();
}
```

可以看到我们使用工厂模式管理依赖的时候，可以在处理业务逻辑外部根据处理请求需要依赖的模块自行进行注入。比如例子中就注入了request、response模块。这种模式虽然解决了我们处理逻辑对外部模块的依赖管理问题，但是并不是太完美，我们的程序只是将原来逻辑对一个个实例子对象的依赖转换成了工厂对这些实例子对象的依赖，工厂和这些实例子对象之间的耦合还存在，随着工厂越来越大，用户逻辑实现越来越复杂，这种“依赖查找”实现控制反转的模式对于用户来讲依然很痛苦。

接下来我们使用**Ioc服务容器**来实现依赖注入,下边先实现一个简单的服务容器:

```
class Container
{
    //用于装提供实例的回调函数，真正的容器还会装实例等其他内容
    protected $bindings = [];

    //容器共享实例数组(单例)
    protected $instances = [];

    public function bind($abstract, $concrete = null, $shared = false)
    {
        if (! $concrete instanceof Closure) {
            //如果提供的参数不是回调函数，则产生默认的回调函数
            $concrete = $this->getClosure($abstract, $concrete);
        }

        $this->bindings[$abstract] = compact('concrete', 'shared');
    }

    public function getBuildings()
    {
        return $this->bindings;
    }

    //默认生成实例的回调函数
    protected function getClosure($abstract, $concrete)
    {
        return function ($c) use ($abstract, $concrete)
        {
            $method = ($abstract == $concrete) ? 'build' : 'make';
            //调用的是容器的build或make方法生成实例
            return $c->$method($concrete);
        };
    }

    //生成实例对象，首先解决接口和要实例化类之间的依赖关系
    public function make($abstract)
    {
        $concrete = $this->getConcrete($abstract);
        if ($this->isBuildable($concrete, $abstract)) {
            $object = $this->build($this->build($concrete));
        } else {
            $object = $this->make($concrete);
        }

        return $object;
    }

    protected function isBuildable($concrete, $abstract)
    {
        return $concrete === $abstract || $concrete instanceof Closure;
    }

    //获取绑定的回调函数
    protected function getConcrete($abstract)
    {
        if (!isset($this->bindings[$abstract]))
        {
            return $abstract;
        }

        return $this->bindings[$abstract]['concrete'];
    }

    //实例化一个对象
    public function build($concrete)
    {
        if ($concrete instanceof Closure) {
            return $concrete($this);
        }
        $reflector = new ReflectionClass($concrete);
        if (! $reflector->isInstantiable()) {
            echo $message = "Target [$concrete] is not instantiable.";
        }
        $constructor = $reflector->getConstructor();
        if(is_null($constructor)) {
            return new $concrete;
        }
        $dependencies = $constructor->getParameters();
        $instances = $this->getDependencies($dependencies);

        return $reflector->newInstanceArgs($instances);
    }

    //通过反射机制实例化对象时的依赖
    protected function getDependencies($parameters)
    {
        $dependencies = [];
        foreach($parameters as $parameter)
        {
            $dependency = $parameter->getClass();
            if(is_null($dependency)) {
                $dependencies[] = NULL;
            } else {
                $dependencies[] = $this->resolveClass($parameter);
            }
        }

        return (array) $dependencies;
    }

    protected function resolveClass(ReflectionParameter $parameter)
    {
        return $this->make($parameter->getClass()->name);
    }

    //注册一个实例并绑定到容器中
    public function singleton($abstract, $concrete = null){
        $this->bind($abstract, $concrete, true);
    }
}
```

该服务容器可以称为Lumen服务容器的简化版，但是它实现的功能和Lumen服务容器是一样的，虽然只有一百多行的代码，但是理解起来有难度，这里就详细讲解清楚简化版容器的代码和原理，接下来章节对Lumen服务容器源码分析时就仅仅只对方法做简单介绍。

根据对服务容器介绍章节所讲:容器中有两个关键属性$bindings和$instance,其中$bindings中存在加入到容器中的回调函数，而$instance存放的是容器中绑定的实例对象。我们还知道$singleton方法用来绑定单例对象，其底层只是调用了bind方法而已，只不过$shared属性为true，意为容器中全局共享:

```
//注册一个实例并绑定到容器中
public function singleton($abstract, $concrete = null){
    $this->bind($abstract, $concrete, true);
}
```

bind方法的实现也很简单,只是将用户指定的服务解析好之后存放入相应的属性当中:

```
public function bind($abstract, $concrete = null, $shared = false)
    {
        if (! $concrete instanceof Closure) {
            //如果提供的参数不是回调函数，则产生默认的回调函数
            $concrete = $this->getClosure($abstract, $concrete);
        }

        $this->bindings[$abstract] = compact('concrete', 'shared');
    }
```

Closure是php中的匿名函数类类型。$abstract和$concrete可以抽象理解为KV键值对，K就是$abstract，是服务名；V是$concrete，是服务的具体实现。我们理解容器，首先要将思维从平常的业务逻辑代码中转换回来。业务逻辑中操作的一般是用户数据，而容器中，我们操作的是对象、类、接口之类的,在框架中可称为“服务”。如果用户要绑定的具体实现$concrete不是匿名函数，则调用getClosure方法生成一个匿名函数：

```
//获取绑定的回调函数
    //默认生成实例的回调函数
    protected function getClosure($abstract, $concrete)
    {
        return function ($c) use ($abstract, $concrete)
        {
            $method = ($abstract == $concrete) ? 'build' : 'make';
            //调用的是容器的build或make方法生成实例
            return $c->$method($concrete);
        };
    }
```

getClosure是根据用户传入的参数来决定调用系统的build和make方法。其中build方法就是构建匿名函数和类实例的关键实现，使用了php中的反射机制，解析出类实例:

```
//实例化一个对象
    public function build($concrete)
    {
        if ($concrete instanceof Closure) {
            return $concrete($this);
        }
        $reflector = new ReflectionClass($concrete);
        if (! $reflector->isInstantiable()) {
            echo $message = "Target [$concrete] is not instantiable.";
        }
        $constructor = $reflector->getConstructor();
        if(is_null($constructor)) {
            return new $concrete;
        }
        $dependencies = $constructor->getParameters();
        $instances = $this->getDependencies($dependencies);

        return $reflector->newInstanceArgs($instances);
    }
```

build首先判断参数$concrete是一个匿名函数，就返回调用匿名函数的一个闭包。否则$concrete是一个类,利用反射机制解析类的信息,首先判断类是否能够被实例化(例如单例就不能被实例化，容器中的单例是通过属性$shared来区分的);确保了类能够被实例化以后,使用getConstructor()判断类是否定义了构造函数，如果没有定义构造函数，直接实例化得到一个类的实例。否则就再次调用getParameters获取构造函数中都传入了哪些参数(也就是判断$concrete类都有哪些依赖)，getDependencies方法就是来生成$concrete依赖的函数:

``` 
    //通过反射机制实例化对象时的依赖
    protected function getDependencies($parameters)
    {
        $dependencies = [];
        foreach($parameters as $parameter)
        {
            $dependency = $parameter->getClass();
            if(is_null($dependency)) {
                $dependencies[] = NULL;
            } else {
                $dependencies[] = $this->resolveClass($parameter);
            }
        }

        return (array) $dependencies;
    }
```

得到了类依赖的实例以后，就调用newInstanceArgs($instances)来生成类的实例。

服务解析函数make主要由build函数实现：

```
    //生成实例对象，首先解决接口和要实例化类之间的依赖关系
    public function make($abstract)
    {
        $concrete = $this->getConcrete($abstract);
        if ($this->isBuildable($concrete, $abstract)) {
            $object = $this->build($this->build($concrete));
        } else {
            $object = $this->make($concrete);
        }

        return $object;
    }
```

有了服务容器以后,我们就可以使用服务容器来存储处理请求中需要的服务，并实现服务中的依赖自动注入。不过首先我们需要将Request、Response单例做修改，因为服务容器对单例的管理，是通过$shared属性进行设置的。所以Request、Response要能够被实例化，才能保存到容器的$bindings数组中:

```
class Request
{

    public function __construct()
    {
    }

    public function get($key)
    {
        return $_GET[$key] ? $_GET[$key] : '';
    }

    public function post($key)
    {
        return $_POST[$key] ? $_POST[$key] : '';
    }
}

class Response
{
    public function __construct()
    {
    }

    public function json($data)
    {
        return json_encode($data);
    }
}
```

我们再来看使用容器后处理用户请求的源代码:

```
include_once 'Container.php';
include_once 'Request.php';
include_once 'Response.php';
include_once 'ExceptionHandler.php';

$app = new Container();
//绑定错误处理
$app->bind('exception', 'ExceptionHandler');
//将请求、响应单例组件添加到容器中
$app->singleton('request', 'Request');
$app->singleton('response', 'Response');
//解析错误处理
$app->make('exception');

//用户逻辑
class UserLogic
{
    public $app = null;

    public function __construct(Container $app)
    {
        $this->app = $app;
    }

    public function getUserList()
    {
        if ($this->app->make('request')->get('path') == 'userlist') {
            $userList = [
                ['name' => '张三', 'age' => 18],
                ['name' => '李四', 'age' => 22]
            ];

            return $this->app->make('response')->json($userList);
        }
    }
}

try {
    $userLogic = new UserLogic($app);
    echo $userLogic->getUserList();
} catch (\Error $e) {
    var_dump($e);
    exit();
}
```

我们还是按照之前的步骤，使用容器将错误处理类绑定到容器中，然后解析出来使用。使用singleton方法将Request和Response类绑定到容器中，类型是单例。这样我们管理服务模块、实现依赖注入这些问题全都交给容器来做就好了。我们想要什么样的服务，就向容器中添加，在需要使用的时候，就利用容器解析使用就可以了。lumen框架中的服务容器是全局的，不需要像例子中一样，手动注入到逻辑代码中使用。