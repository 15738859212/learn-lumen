### 控制反转(Ioc)和依赖注入(DI)

**控制反转**是框架设计的一种原则，在很大程度上降低代码模块之间的耦合度，有利于框架维护和拓展。实现控制反转最常见的方法是“依赖注入”，还有一种方法叫“依赖查找”。控制反转将框架中解决依赖的逻辑从实现代码类库的内部提取到了外部来管理实现。

我们用简单代码模拟一下Lumen处理用户请求的逻辑，框架中要使用到最简单的Request请求模块、Response请求模块,我们就使用单例模式简单实现一下:

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

可以看到我们使用工厂模式管理依赖的时候，可以在处理业务逻辑外部根据处理请求需要依赖的模块自行进行注入。比如例子中就注入了request、response模块。这种模式虽然解决了我们处理逻辑对外部模块的依赖管理问题，但是并不是太完美，我们的程序只是将原来逻辑对一个个实例子对象的依赖转换成了工厂对这些实例子对象的依赖，工厂和这些实例子对象之间的耦合还存在，随着工厂越来越大，用户逻辑实现越来越复杂，这种“依赖查找”实现控制反转的模式对于用户来讲，依然比较痛苦。



