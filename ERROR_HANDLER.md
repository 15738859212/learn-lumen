### 错误处理

Lumen框架设计的初衷是：所有的PHP错误都应该被处理，包括Notice和Warning。真实的生产环境中，我们不会将代码设置的这么严格，出现Notice或者Warning时，只需要记录到日志就可以了。另外Lumen自带的错误信息展示带有模版，在浏览器页面上比较直观；但在前后端分离的项目开发中，错误信息不简洁，往往需要用户自定义。

#### 一、php7错误处理

php7改变了大多数错误的报告方式,不同于传统(php5)的错误报告机制,现在大多数错误被作为Error异常抛出。

这种 Error 异常可以像 Exception 异常一样被第一个匹配的 try / catch 块所捕获。如果没有匹配的 catch 块，则调用异常处理函数（事先通过 set_exception_handler() 注册）进行处理。 如果尚未注册异常处理函数，则按照传统方式处理：被报告为一个致命错误（Fatal Error）。

Error 类并非继承自 Exception 类，所以不能用 catch (Exception $e) { ... } 来捕获 Error。你可以用 catch (Error $e) { ... }，或者通过注册异常处理函数（ set_exception_handler()）来捕获 Error。

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


