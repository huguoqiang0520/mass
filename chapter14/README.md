欢迎来到[「我是真的狗杂谈世界」](https://github.com/huguoqiang0520/mass/blob/main/README.md)，关注不迷路

# 略读

- CGI+同步阻塞方案异步任务方案有点重，但又不想放弃傻瓜方案的优点；
- 思考后将问题转换成：
  - 提前返回响应后继续同步执行非重要任务；
  - 顺序编写逻辑，延迟执行部分非重要任务；
- 解决这两个问题：
  - 使用fastcgi_finish_request；
  - 借鉴golang defer；
- 实现、效果和注意事项。

# 背景

## 团队技术背景

目前我们团队小组的技术情况如下：

- 以PHP作为开发语言开发Web类接口服务；
- 采用传统的Nginx+FPM模式运行服务；
- 我创建并维护了新的开发框架；
- 框架基于Slim v3.7，是小组之前依赖的，因考虑过渡成本暂时没改变。

## 场景问题

近期在一些项目中发现经常遇到业务接口有以下特点：

- 有主：一个接口中部分逻辑（比如下方栗子中的1/3/5，后续简称主逻辑）是需要保障处理成功并将结果反馈给调用方的；
- 有支：另一部分逻辑（比如下方栗子中的2/4，后续简称支逻辑）则可容忍（暂时）失败，甚至调用方并不关心结果或说感知不明显；
- 简单：大部分支逻辑比较简单，简单判断加上打文件日志、写条MySQL日志记录、发个HTTP请求等类；
- 混合：主支逻辑在代码编写顺序上往往是交叉混淆而非泾渭分明的。

> 关于代码编写顺序当然也可以特意把两部分分开，但这样并不符合常规开发同学实现的思路脉络，也不利于代码阅读理解和变量控制

---

举一个接口栗子（瞎编的）：

1. 一个重要的扣除逻辑，成功才能继续；
2. 扣除失败则发送一条info级别的邮件消息，成功与否都行；
3. 一个重要的发货逻辑，成功才能继续；
4. 发货失败则发送一条error级别的邮件消息，发送失败则记录本地文件日志；
5. 组装结果并返回。

## 同步模型

先闭上眼睛一把梭顺序编写代码实现，同步执行流程如下（事实上一开始我真是这么一把梭实现的）：

![同步模式请求处理时序](https://raw.githubusercontent.com/huguoqiang0520/mass/main/chapter14/img1-同步模式请求处理时序.png)

## 异步模型

- 遗憾的是有一天QA同学说你这个接口响应耗时太高了，压测时的表现更明显；
- 更遗憾的是线上居然还发生了2/4步骤DNS解析超时问题（后面发现是整个libcurl问题，不仅DNS解析，当然这是另一个话题了）

---

这种场景下传统常见的方案也许我们不会陌生——进程级别的异步任务方案！

> Laravel/Lumen也是采用这个方案的
> 我们组内另一位同学也为我们的开发框架支持了这种异步任务的方案，其实是可以直接采用的，只是它并不是本文的主角～

执行流程如下：

![异步模式请求处理时序](https://raw.githubusercontent.com/huguoqiang0520/mass/main/chapter14/img2-异步模式请求处理时序.png)

## 多线程、协程+异步IO调度模型

当然还有很多其他方案，不过也都比较复杂，且不利于保持CGI+同步阻塞这套模型给团队同学的门槛好处和对服务的稳定安全，所以也不是本文的主角～

> 特别是团队今年的主基调是质量、与效率，更加不想这个时候搞事情啦～

# 思路

上述异步模型其实是比较成熟的方案选择，只是它也存在着一些问题/弊端：

- 队列服务依赖：需要额外依赖一个队列服务，这也一定程度上依赖了其可用性、同时本身也是一条网络IO开销；
- 消费进程管理：需要独立的消费进程消费队列中的异步任务，独立消费进程也需要额外考虑其运行状态维护和控制；
- 处理链路增长：这个就不多说了，虽然这对于异步任务而言倒不算什么。

## 分析转换

那有没有（使用成本和运行效率上）轻量级又保留现有优势的方案呢？

---

思考分析：

- 既然仍然是采用同步阻塞方案，也就是说不去做串行改并行的优化，仍旧是原来的执行时间开销；
- 同时想要轻量级，那需要将队列服务、消费进程都干掉，只能仍旧由当前处理该请求的FPM处理进程来执行全部逻辑；
- 那能不能搞一个伪"异步"呢？让调用方在感知上提前结束，但该FPM处理进程仍旧会完成剩下逻辑。

---

问题转换：

1. PHP在FPM运行模式下能不能让请求响应提前返回给调用方？
2. 能不能顺序编写代码逻辑，但执行时将指定部分的代码逻辑块延迟到某个指定逻辑之后？（这是因为我需要考虑封装成方便大家使用的框架能力）

## 问题1解法：fastcgi_finish_request

很容易想到[fastcgi_finish_request](https://www.php.net/manual/zh/function.fastcgi-finish-request.php)
，FPM正好又是FastCGI模式，可以使用

> 当然使用它是要注意一些点的，具体我放在本文最后了～

---

另外印象里Laravel/Lumen有个终结者中间件，好像也是实现了类似功能（先返回响应给调用方，再继续执行终结者中间件逻辑），于是打算去翻翻源码回忆验证一下其实现原理：

> - 对这个有印象是因当年有同事使用Laravel时遇到通过框架提供的session修改和保存方法但是并没有生效的问题
> - 协助排查时大概看到过框架对于session的操作都是在进程内存级别的，只在终结者中间件中才会将session完整覆盖到存储驱动中
> - 问题出在同事一通逻辑处理后没有采用框架提供的response方法返回响应，直接echo然后exit，以至于没法走到框架后续返回响应和执行终结者中间件。。。
> - 所以记忆犹新～～

---

通过Laravel源码验证也是通过fastcgi_finish_request来实现此功能的：

> 阅读理解为：kernel->handle后得到response，response->send中执行了header和body的设置输出后，执行了fastcgi_finish_request
> 当然它对其他运行模式也做了兼容，但是我暂时用不到

```php
$app = require_once __DIR__.'/../bootstrap/app.php';

$kernel = $app->make(Kernel::class);

$response = $kernel->handle(
    $request = Request::capture()
)->send();

$kernel->terminate($request, $response);
```

```php
    /**
     * Sends HTTP headers.
     *
     * @return $this
     */
    public function sendHeaders(): static
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
            header('Set-Cookie: '.$cookie, false, $this->statusCode);
        }

        // status
        header(sprintf('HTTP/%s %s %s', $this->version, $this->statusCode, $this->statusText), true, $this->statusCode);

        return $this;
    }

    /**
     * Sends content for the current web response.
     *
     * @return $this
     */
    public function sendContent(): static
    {
        echo $this->content;

        return $this;
    }

    /**
     * Sends HTTP headers and content.
     *
     * @return $this
     */
    public function send(): static
    {
        $this->sendHeaders();
        $this->sendContent();

        if (\function_exists('fastcgi_finish_request')) {
            fastcgi_finish_request();
        } elseif (\function_exists('litespeed_finish_request')) {
            litespeed_finish_request();
        } elseif (!\in_array(\PHP_SAPI, ['cli', 'phpdbg'], true)) {
            static::closeOutputBuffers(0, true);
        }

        return $this;
    }
```

## 问题2解法：参考golang defer

前文说到我要考虑框架封装方便组内同学使用的，所以不能裸写而要考虑大家使用时的便捷性

我想如果可以同步编写代码逻辑块，但在执行顺序上将标记的代码逻辑块延迟到请求响应返回后继续执行就很理想了～

---

等等，这不是跟golang中的defer很像吗？而且都是延迟执行的意思！当然它们本质是有区别的：

-
golang的defer是语言级别的，代表的是将逻辑块的执行延迟到函数执行结束栈帧要销毁前执行的意思，具体参考[「Chapter 15.Golang defer介绍与实现浅析」][1]
；
- 而我需要的是在框架层面的（虽然我也觉得如果php有语言层面的defer挺好），用来将逻辑块的执行延迟到请求响应结束后的意思。

# 实现

golang的defer是通过链表实现的，而php吗——array走天下哈哈！

---

先定义一个Defer类用来注册、判断和执行要被'defer'的逻辑块（callable）：

```php
class Defer
{
    protected array $deferList = [];

    public function defer(callable $function): void
    {
        $this->deferList[] = $function;
    }

    public function isEmpty(): bool
    {
        return empty($this->deferList);
    }

    public function run(): void
    {
        foreach ($this->deferList as $function) {
            $function();
        }
    }
}
```

---

- 实例化Defer并挂载到Container上；
- 执行slim App run，其中可以任由业务逻辑中进行'defer'注册；
- 如果注册了'defer'，则提前返回请求响应后执行各'defer'：

```php
$container['defer'] = new Defer();

$app->run();

// 响应后任务
$container = $app->getContainer();
if ($container->has('defer')) {
    $defer = $container['defer'];
    if (! $defer->isEmpty()) {
        fastcgi_finish_request();
        $defer->run();
    }
}
```

---

业务逻辑中的'defer'注册：

```php

    // TODO 1

    $container['defer']->defer(function () {
        // TODO 2
    });
    
    // TODO 3
    
    $container['defer']->defer(function () {
        // TODO 4
    });
    
    // TODO 5
```

# 效果

## 关于嵌套defer

理论上也可以做成像go的defer任意嵌套，但既然是延迟到请求返回响应后嵌套就没意义，反而会增加编码和阅读的复杂性，所以干脆不支持。

## 关于闭包变量

使用时需注意变量生命周期，毕竟不像go的defer是语言层面执行到defer时先处理函数签名再继续的，具体闭包与变量生命周期相关参考：

- [「Chapter 16.PHP 变量生命周期」][1]
- [「Chapter 17.PHP 闭包」][1]

## defer运行模型

经过上述一系列的姿势，咱们按照顺序编写的代码逻辑执行顺序就发生了变化：

![伪异步模式请求处理时序](https://raw.githubusercontent.com/huguoqiang0520/mass/main/chapter14/img3-伪异步模式请求处理时序.png)

## 理论效果

从上面的处理时序不难看出，伪'异步'模型能带来：

- 提升接口响应速度（因为提前返回响应了）；
- 在CPU等硬件资源和FPM等软件资源未达瓶颈时提升QPS表现（QPS=并发度*平均响应耗时，并发度不变，响应耗时降低，收到响应后继续请求，由于资源够用，QPS升高）；
- 在CPU等硬件资源和FPM等软件资源达到瓶颈时无法提升QPS表现（请求提前返回后，再次发起请求，原先的逻辑还没处理完，又达到瓶颈了，故此无法提升QPS）。

## 使用注意

另外伪'异步'仍旧在FPM进程中，可不能像前面的传统异步一样处理长任务：

- 仍旧受到PHP最大CPU执行时间（php.ini的max_execution_time）的限制；
- 但不再受到FPM请求终结超时（fpm.conf的request_terminate_timeout）的限制；
- 想想长任务占据光了FPM的话，新的请求怎么办呢？

[1]: https://github.com/huguoqiang0520/mass/blob/main/README.md