分析，理解，优化Laravel

#前言
使用Laravel接近两年了， 从4.2到现在的5.2。大多数情况下，Laravel都能很好的支撑我们的系统。不管你是否使用或打算使用， 了解Laravel的设计对于提高程序水平也很有帮助。本文是我的一个学习laravel的过程， 也是我实现高性能server [stone](https://github.com/chefxu/stone)下一个目标的过程。限于本人水平有限，错误在所难免，希望大家不吝指正。

*注：因为是边写边改， 所以内容目录随时会有调整。*

#一个HTTP请求的执行流程
1. 创建app， app本质上是一个ioc容器（Illuminate\Container\Container），这个概念对于laravel很重要，laravel的核心组件都会注册到这个容器里面。

    ```php
    $app = new Illuminate\Foundation\Application(
        realpath(__DIR__.'/../')
    );
    ```
    
    * 初始化全局ioc容器app， 并把自己本身注册进去
    * 注册基本的Service Provider: EventServiceProvider, RoutingServiceProvider
    * 注册核心的class别名，参看：registerCoreContainerAliases
    
    
1. 核心接口绑定， 如果需替换自己实现的Kernel， 可以修改这里。

    ```php
    // http处理
    $app->singleton(
        Illuminate\Contracts\Http\Kernel::class,
        App\Http\Kernel::class
    );
    
    // console处理
    $app->singleton(
        Illuminate\Contracts\Console\Kernel::class,
        App\Console\Kernel::class
    );
    
    // 异常处理
    $app->singleton(
        Illuminate\Contracts\Debug\ExceptionHandler::class,
        App\Exceptions\Handler::class
    );
    ```
    
1. 创建Kernel，注意这里是基于接口来创建， 最终创建什么实例， 依赖于前面的接口绑定。 

    ```php
    $kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);
    ```
    
    * 这个代码最终实例化的是App\Http\Kernel，App\Http\Kernel继承自Illuminate\Foundation\Http\Kernel。
    * 接口绑定是laravel框架设计精华所在，laravel框架的核心组件全部都实现了接口，在laravel里统一放在Contracts下。这给予了开发者极大的自由，可以在程序中通过修改接口绑定来改变组件的实现。

        ``` bash
        chunfang@office:~/base.account$ ls vendor/laravel/framework/src/Illuminate/Contracts/
        Auth          Bus    composer.json  Console    Cookie    Debug       Events      
        Foundation  Http     Mail        Pipeline  Redis    Support     View
        Broadcasting  Cache  Config         Container  Database  Encryption  Filesystem  
        Hashing     Logging  Pagination  Queue     Routing  Validation
        ```
    
1. 基于全局变量创建request对象， 交由Kernel处理， 这是请求处理的核心所在。

    * 基于全局变量创建请求

        ```php
        $request = Illuminate\Http\Request::capture()
        ```
    * Kerner处理request

        ```php
        $response = $kernel->handle($request);
        ```
    
        请求方法关键细节：
        
        ```php
        /**
         * Handle an incoming HTTP request.
         *
         * @param  \Illuminate\Http\Request  $request
         * @return \Illuminate\Http\Response
         */
        public function handle($request)
        {
            try {
                // 允许使用_method来模拟PUT，DELETE等
                $request->enableHttpMethodParameterOverride();
    
                // 处理请求返回响应
                $response = $this->sendRequestThroughRouter($request);
            } catch (Exception $e) {
                $this->reportException($e);
    
                $response = $this->renderException($request, $e);
            } catch (Throwable $e) {
                $this->reportException($e = new FatalThrowableError($e));
    
                $response = $this->renderException($request, $e);
            }
    
            // 发射事件
            $this->app['events']->fire('kernel.handled', [$request, $response]);
    
            return $response;
        }

        ```
        处理请求返回响应关键细节：
        
        ```php
        /**
         * Send the given request through the middleware / router.
         *
         * @param  \Illuminate\Http\Request  $request
         * @return \Illuminate\Http\Response
         */
        protected function sendRequestThroughRouter($request)
        {
            $this->app->instance('request', $request);
    
            Facade::clearResolvedInstance('request');
    
            // 初始化请求资源
            $this->bootstrap();
    
            // 流水线处理
            return (new Pipeline($this->app))
                        ->send($request)
                        ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
                        ->then($this->dispatchToRouter());
        }
        ```
        
        初始化资源关键细节：
        
        ```php
        protected $bootstrappers = [
            'Illuminate\Foundation\Bootstrap\DetectEnvironment',
            'Illuminate\Foundation\Bootstrap\LoadConfiguration',
            'Illuminate\Foundation\Bootstrap\ConfigureLogging',
            'Illuminate\Foundation\Bootstrap\HandleExceptions',
            'Illuminate\Foundation\Bootstrap\RegisterFacades',
            'Illuminate\Foundation\Bootstrap\RegisterProviders',
            'Illuminate\Foundation\Bootstrap\BootProviders',
        ];
        ```
        
        流水线实现细节：
        [理解Laravel中的pipeline](http://www.jianshu.com/p/3c2791a525d0)
        
        路由处理关键细节：
        
        ```php
        /**
         * Dispatch the request to the application.
         *
         * @param  \Illuminate\Http\Request  $request
         * @return \Illuminate\Http\Response
         */
        public function dispatch(Request $request)
        {
            $this->currentRequest = $request;
    
            $response = $this->dispatchToRoute($request);
    
            return $this->prepareResponse($request, $response);
        }
        ```
        ```php
        /**
         * Dispatch the request to a route and return the response.
         *
         * @param  \Illuminate\Http\Request  $request
         * @return mixed
         */
        public function dispatchToRoute(Request $request)
        {
            // First we will find a route that matches this request. We will also set the
            // route resolver on the request so middlewares assigned to the route will
            // receive access to this route instance for checking of the parameters.
            $route = $this->findRoute($request);
    
            $request->setRouteResolver(function () use ($route) {
                return $route;
            });
    
            $this->events->fire(new Events\RouteMatched($route, $request));
    
            $response = $this->runRouteWithinStack($route, $request);
    
            return $this->prepareResponse($request, $response);
        }
        ```
        
        路由执行实际动作关键细节：
        
        ```php
        /**
         * Run the route action and return the response.
         *
         * @param  \Illuminate\Http\Request  $request
         * @return mixed
         */
        public function run(Request $request)
        {
            $this->container = $this->container ?: new Container;
    
            try {
                if (! is_string($this->action['uses'])) {
                    return $this->runCallable($request);
                }
    
                return $this->runController($request);
            } catch (HttpResponseException $e) {
                return $e->getResponse();
            }
        }
        ```

1. 输出到客户端

    ```php
    $response->send();
    ```
2. 依次调用结束动作

    ```php
    $kernel->terminate($request, $response);
    ```
# IOC容器的实现
编写中
# 实现自己的路由替代Laravel路由
编写中
# 浅析资源初始化
编写中
# Laravel Pipeline的实现
编写中
# 基于Stone实现高性能Kernel
编写中

    