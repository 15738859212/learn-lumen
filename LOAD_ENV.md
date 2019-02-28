### 加载配置文件

实现类自动载入和单一入口功能之后,Lumen框架便开始加载配置文件内容到环境变量,一般情况下会加载项目根目录下.env文件中的相关配置,实际的项目开发过程中,开发人员会根据项目的应用场景分别配置本地、测试和线上环境的配置文件。本节学习Lumen框架加载配置文件的过程。源码从 bootstrap/app.php 中开始：

```
    try {
        (new Dotenv\Dotenv(dirname(__DIR__)))->load();
    } catch (Dotenv\Exception\InvalidPathException $e) {
        //
    }
```

代码匿名实例化了Dotenv\Dotenv()类,并调用其load方法。我们来看实例化Dotenv\Dotenv()时程序做了哪些工作:

```
    protected $loader;

    ......

    public function __construct($path, $file = '.env')
    {
        $this->filePath = $this->getFilePath($path, $file);
        $this->loader = new Loader($this->filePath, true);
    }
```

程序做了两件事情:
 + 根据传递参数设置配置文件加载路径
 + 依赖注入Loader类

在构造函数中手动实例化类,并将其赋值给自身受保护的成员变量是解决“依赖注入”的一种方式，但是这种实现依赖注入的方法只适用简单的逻辑处理，后边我们会讲到Lumen中使用服务容器自动实现依赖注入的强大功能。

我们接着分析Dotenv\Dotenv()类的load方法:

```
    public function load()
    {
        return $this->loadData();
    }
```

继续分析Dotenv\Dotenv()类的loadData()方法:

```
    protected function loadData($overload = false)
    {
        return $this->loader->setImmutable(!$overload)->load();
    }
```

loadData方法使用loader成员变量(Loader对象)调用Loader类中的setImmutable方法和load方法。$overload默认为false,取反之后作为参数传入setImmutable()方法中。我们继续追踪查看setImmutable()方法:

```
    protected $immutable;

    public function __construct($filePath, $immutable = false)
    {
        $this->filePath = $filePath;
        $this->immutable = $immutable;
    }

    ......

    public function setImmutable($immutable = false)
    {
        $this->immutable = $immutable;

        return $this;
    }
```

Loader类初始化时默认赋值自身受保护的成员变量$immutable为false,通过自身开放成员方法setImmutable()方法修改设置其值。这种类库模式的设计符合“最小开放原则”,防止了自身受保护成员变量被无意篡改。结合上下文得知: $immutable表示配置文件是否被加载过,保证了每次请求生命周期配置文件只会被加载一次。setImmutable方法返回$this（当前对象）,这是实现链式调用的灵魂所在。我们继续追踪程序紧接着调用的load方法:

```
    public function load()
    {
        $this->ensureFileIsReadable();

        $filePath = $this->filePath;
        $lines = $this->readLinesFromFile($filePath);
        foreach ($lines as $line) {
            if (!$this->isComment($line) && $this->looksLikeSetter($line)) {
                $this->setEnvironmentVariable($line);
            }
        }
        return $lines;
    }
```

load方法首先调用ensureFileIsReadable方法判断环境变量文件是否合法(文件存在并可读):

```
    protected function ensureFileIsReadable()
    {
        if (!is_readable($this->filePath) || !is_file($this->filePath)) {
            throw new InvalidPathException(sprintf('Unable to read the environment file at %s.', $this->filePath));
        }
    }
```

然后调用readLinesFromFile方法将配置文件内容逐行读入数组$line:

```
    protected function readLinesFromFile($filePath)
    {
        // Read file into an array of lines with auto-detected line endings
        $autodetect = ini_get('auto_detect_line_endings');
        ini_set('auto_detect_line_endings', '1');
        $lines = file($filePath, FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);
        ini_set('auto_detect_line_endings', $autodetect);

        return $lines;
    }
```

php的内置函数file第二个参数 FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES 表示忽略文件中的空行和文件结束符EOF。紧接着程序循环遍历处理$lines数组中的原始数据,程序首先过滤掉了环境变量中的注释部分和不合法的行:

```
  ......

    protected function isComment($line)
    {
        $line = ltrim($line);

        return isset($line[0]) && $line[0] === '#';
    }

  ......

    protected function looksLikeSetter($line)
    {
        return strpos($line, '=') !== false;
    }

  ......   
```

之后调用setEnvironmentVariable方法进一步过滤、解析数据:

```
    public function setEnvironmentVariable($name, $value = null)
    {
        list($name, $value) = $this->normaliseEnvironmentVariable($name, $value);

        $this->variableNames[] = $name;

        // Don't overwrite existing environment variables if we're immutable
        // Ruby's dotenv does this with `ENV[key] ||= value`.
        if ($this->immutable && $this->getEnvironmentVariable($name) !== null) {
            return;
        }

        // If PHP is running as an Apache module and an existing
        // Apache environment variable exists, overwrite it
        if (function_exists('apache_getenv') && function_exists('apache_setenv') && apache_getenv($name) !== false) {
            apache_setenv($name, $value);
        }

        if (function_exists('putenv')) {
            putenv("$name=$value");
        }

        $_ENV[$name] = $value;
        $_SERVER[$name] = $value;
    }   
```

setEnvironmentVariable方法做了加载配置文件到环境变量的最后工作,函数首先调用normaliseEnvironmentVariable方法规范化整理数据（解析成KV键值对）。追加到variableNames数组中，然后将数据分别写入到全局变量$_ENV和 $_SERVER中去,在开发过程中可以通过这这两个全局变量取的配置文件中的相关数据。我们再来细看normaliseEnvironmentVariable方法的实现:

```
    protected function normaliseEnvironmentVariable($name, $value)
    {
        list($name, $value) = $this->processFilters($name, $value);

        $value = $this->resolveNestedVariables($value);

        return array($name, $value);
    }
```

代码主要做了两件事:首先调用processFilters方法过滤整理数据,然后调用resolveNestedVariables解析配置文件中嵌套变量的替换。前者代码比较简单,我们就不再分析了,主要来看后者的实现:

```
    protected function resolveNestedVariables($value)
    {
        if (strpos($value, '$') !== false) {
            $loader = $this;
            $value = preg_replace_callback(
                '/\${([a-zA-Z0-9_.]+)}/',
                function ($matchedPatterns) use ($loader) {
                    $nestedVariable = $loader->getEnvironmentVariable($matchedPatterns[1]);
                    if ($nestedVariable === null) {
                        return $matchedPatterns[0];
                    } else {
                        return $nestedVariable;
                    }
                },
                $value
            );
        }

        return $value;
    }
```

代码首先判断数据中是否存在$符号判断是否有可解析的变量,然后调用内置函数preg_replace_callback对可解析变量进行替换，替换规则/\${([a-zA-Z0-9_.]+)}/ 表示是查找类似${...}的字符串,然后将匹配到的字符串解析替换为全局变量中匹配的值。代码:

```
function ($matchedPatterns) use ($loader) {
                   ......
}
```

是闭包的实现,在函数内部可以调用$loader这个外部变量,$matchedPatterns是正则表达式匹配到的数组,其中$matchedPatterns[1]表示子匹配([a-zA-Z0-9_.]+)的结果。getEnvironmentVariable方法是从全局变量中查找指定的值:

```
    public function getEnvironmentVariable($name)
    {
        switch (true) {
            case array_key_exists($name, $_ENV):
                return $_ENV[$name];
            case array_key_exists($name, $_SERVER):
                return $_SERVER[$name];
            default:
                $value = getenv($name);
                return $value === false ? null : $value; // switch getenv default to null
        }
    }
```

以上简单的代码就实现了解析配置文件中嵌套变量的强大功能。

我们来简单配置一下本地的.env文件

```
APP_NAME=Lumen
APP_ENV=local
APP_NAME_ENV=Laravel${APP_NAME}
APP_KEY=4fff29bf2144a5ade1f177b3cb0fa94f
APP_DEBUG=true
APP_URL=http://localhost
APP_TIMEZONE=UTC

LOG_CHANNEL=stack
LOG_SLACK_WEBHOOK_URL=

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=homestead
DB_USERNAME=homestead
DB_PASSWORD=secret

CACHE_DRIVER=file
QUEUE_CONNECTION=sync
```

然后在代码中配置文件加载完成之后打印一下$_ENV的内容,我们就得到加载到程序全局变量中的内容:

```
array:17 [▼
  "APP_NAME" => "Lumen"
  "APP_ENV" => "local"
  "APP_NAME_ENV" => "LaravelLumen"
  "APP_KEY" => "4fff29bf2144a5ade1f177b3cb0fa94f"
  "APP_DEBUG" => "true"
  "APP_URL" => "http://localhost"
  "APP_TIMEZONE" => "UTC"
  "LOG_CHANNEL" => "stack"
  "LOG_SLACK_WEBHOOK_URL" => ""
  "DB_CONNECTION" => "mysql"
  "DB_HOST" => "127.0.0.1"
  "DB_PORT" => "3306"
  "DB_DATABASE" => "homestead"
  "DB_USERNAME" => "homestead"
  "DB_PASSWORD" => "secret"
  "CACHE_DRIVER" => "file"
  "QUEUE_CONNECTION" => "sync"
]
```

总结：

+ 我们可以在一个类的构造函数中实例化依赖的类来手动实现依赖注入。
+ 类的设计要符合“最小开放原则”,要综合考量哪些变量、哪些方法应该开放给用户使用,哪些变量和方法又是避免用户修改和调用的。
+ 实现链式调用的核心是返回当前对象。
+ 闭包在程序中实现特定的功能时非常的实用,可以通过use关键字引用外部变量。
+ 正则表达式在解析字符串时的非常的强大。