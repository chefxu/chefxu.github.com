+++
date = "2016-04-23T00:25:08+08:00"
draft = false
title = "关于Stone的一些计划"

+++
# 关于
Stone是我在趣分期工作中发起的一个性能优化的项目，旨在解决基于laravel编写的接口的性能问题。根据我的经验，在一台8核的PC上，一个laravel的"hello，world"程序， RPS也只有100多。

> 已经开启opcode cache， 关闭debug， 运行 php ./artisan optimize 优化过

年底的时候要求给出一个流量提升10倍的解决方案， 于是就有了Stone这个项目。 Stone最开始是定位于一个TCP业务Server。使用lumen对外提供HTTP接口。目前已经支撑了核心业务白条接口的请求以及一些读写cookie的请求。目前请求次数已经超过13亿次， 两台服务器，负载都很低。

# 目前的一些进展
* 实现了FastCGI协议， 直接与nginx通信， 摆脱了lumen的依赖， 性能高达7500RPS
* 实现了基于Unix domain socket的进程通信， 进一步提高性能10%
* 实现了基本的RAML接口设计规范
* 适配Laravel5.1框架进展顺利

# 计划
* 将拆分为两部分， Stone-Web和Stone-API，名字暂定。
* Stone-Web目标100%兼容laravel5.1，提升laravel框架性能10倍以上，让大象可以优雅地跳舞
* Stone-API专注于高性能API，围绕RAML规范，打造一整套API设计工具集
* 持续反馈Swoole社区



