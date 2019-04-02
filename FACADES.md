### 开启门面

#### 一、介绍
Lumen框架中在app.php中去掉withFacades就开启了门面。Lumen中的Facades也是在框架级提供抽象调用，从而使开发者无需关心内容实现。门面的主要特性是屏蔽内部实现，开放接口供外部使用，提高模块抽象。Lumen中的Auth、Cache、DB、Log、Storage等模块，都默认提供了门面,供开发者简化接口的使用。

我们看看源码中的部分：

```
public function withFacades($aliases = true, $userAliases = [])
    {
        Facade::setFacadeApplication($this);

        if ($aliases) {
            $this->withAliases($userAliases);
        }
    }
```

代码调用Facade::setFacadeApplication将全局容器注入到Facade抽象基类中,将来根据门面解析调用模块相关的方法：

```
public static function setFacadeApplication($app)
    {
        static::$app = $app;
    }
```

之后代码使用withAliases默认在Facades类中默认注册了框架常用的模块门面:

```
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
            'Illuminate\Support\Facades\Route' => 'Route',
            'Illuminate\Support\Facades\Schema' => 'Schema',
            'Illuminate\Support\Facades\Storage' => 'Storage',
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

我们逻辑中常用的Log::info()、Route::get()等方法中的Log、Route分别是此处对应的别名：Illuminate\Support\Facades\Log、Illuminate\Support\Facades\Route,而具体调用的方法是从容器解析出来的服务;框架中的Log、Route、Cache、Event等模块本身特别复杂，但是通过门面模式大大降低了开发者的使用难度，通过简单的静态方法调用即可。Lumen中实现门面模式的原理比较简单，不过首先我们还是看一下：后期静态绑定的相关知识。

#### 二、后期静态绑定
自 PHP 5.3.0 起，PHP 增加了一个叫做后期静态绑定的功能，用于在继承范围内引用静态调用的类。

该功能从语言内部角度考虑被命名为“后期静态绑定”。“后期绑定”的意思是说，static:: 不再被解析为定义当前方法所在的类，而是在实际运行时计算的。也可以称之为“静态绑定”，因为它可以用于（但不限于）静态方法的调用。 

+ self::的限制

使用self::或者__CLASS__对当前类的静态引用，取决于定义当前方法所在的类,例如:

```
<?php
class A {
    public static function who() {
        echo __CLASS__;
    }
    public static function test() {
        self::who();
    }
}

class B extends A {
    public static function who() {
        echo __CLASS__;
    }
}

B::test();
?>
```
输出： A

后期静态绑定使用预留的static关键字，表示运行时最初调用的类;将上述代码中的A类中的test()中的self改为static,就会输出: B

更多关于PHP静态后期绑定的问题，可以参考官网：[后期静态绑定](https://www.php.net/manual/zh/language.oop5.late-static-bindings.php)

#### 三、源码实现

我们以代码中常用的Log::info()为例,Lumen框架中每一个实现门面模式的模块类都继承自抽象类:Facades,使用方法getFacadeAccessor为服务模块起别名(服务容器根据这里的别名查找并解析相关服务):

```
class Log extends Facade
{
    /**
     * Get the registered name of the component.
     *
     * @return string
     */
    protected static function getFacadeAccessor()
    {
        return 'log';
    }
}
```

程序静态调用info方法，会调用Facades类中的__callStatic魔术方法:

```
public static function __callStatic($method, $args)
    {
        $instance = static::getFacadeRoot();

        if (! $instance) {
            throw new RuntimeException('A facade root has not been set.');
        }

        return $instance->$method(...$args);
    }
```

方法首先通过static::getFacadeRoot()方法取得从服务容器中解析出来的相关模块中的实例,然后通过$instance->$method(...$args)代理方法，实际调用相关服务。

我们具体分析一下getFacadeRoot()方法:

```
public static function getFacadeRoot()
    {
        return static::resolveFacadeInstance(static::getFacadeAccessor());
    }
```

代码中调用了resolveFacadeInstance方法，传入的参数是static::getFacadeAccessor解析出来的别名，根据前边介绍的后期静态绑定的知识，这里得到的是log,接着看resolveFacadeInstance方法:

```
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

代码中查看服务是否已经解析好，如果已经解析过，直接返回解析好的单例。否则使用static::$app[$name]解析相关的服务实例即可(服务容器Container实现了数组式访问ArrayAccess接口，所以可以直接以数组的方式解析服务)。解析出实例之后就可以通过$instance->$method(...$args)转发调用实例的相关方法了。