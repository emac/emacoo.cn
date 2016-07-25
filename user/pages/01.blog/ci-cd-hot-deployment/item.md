---
title: 【CI/CD】几种常见的不停机发布方式
published: true
date: '09-06-2016 00:00'
visible: true
summary:
    enabled: '1'
    format: short
taxonomy:
    tag:
        - 原创
        - ci/cd
---

#### 何为不停机发布？

本文所说的不停机发布，是指在**不停止对外服务**的前提下完成应用的更新。与[热部署](http://emacoo.cn/blog/play-hotdeploy)的区别在于，热部署关注于**应用**层面并且以**不重启应用**为前提，而不停机发布则关注于**服务**层面。随着摩尔定律逐渐逼近极限和多核时代的到来，分布式应用已经成为事实上的主流。下文首先给出一种通用的适用于分布式应用环境的不停机发布方式，然后再介绍Master/Worker这种常见的适用于单机应用的不停机发布方式。

#### Cluster模式

对于运行于集群环境的分布式应用，一般在应用之上都有一层负载均衡（LB）。如果在发布过程中，在更新任一节点（也可以是一组节点）前先关闭该节点对应的负载，更新完再打开负载，即可实现整体服务的不停机发布。在此基础上，为了保证服务的稳定性，可以加上备机的支持，即更新某一节点时，先挂上备机，更新完再卸下，依次轮换更新完所有节点后最后再升级备机，如下图所示：

![](http://static.zybuluo.com/emac/zijlzk2pruasmrbmbaz8eev1/QQ20160405-3.png)

_* 完整设计可以参考我写的另一篇[文章](https://www.zybuluo.com/emac/note/330205)_

上述发布过程其实就是一个简单的CD（Continuous Deployment）系统。作为一个参考实现，可以使用[Jenkins 2.0 Pipeline](http://emacoo.cn/blog/jenkins-2-0-from-ci-to-cd)特性定义整个发布流程，使用[Nginx Dynamic Upstream](https://github.com/cubicdaiya/ngx_dynamic_upstream)插件操纵Nginx，然后配合脚本完成应用的启停和检测。

![](QQ20160609-0.png)

#### Master/Worker模式

对于单机应用，由于不存在LB，一般由应用容器实现不停机发布特性，最常见是Master/Worker模式。容器中常驻一个master进程和多个work进程，master进程只负责加载程序和分发请求，由fork出来的worker进程完成具体工作。当容器收到更新应用的信号时，master进程重新加载更新后的程序，然后fork新的worker进程处理新的请求，而老的worker进程在处理完当前请求后就自动销毁。Ruby的[Unicorn](https://github.com/blog/517-unicorn)，PHP的[FPM](http://php-fpm.org/about/)都是采用了这套机制。

#### 延伸阅读

不同于Master/Worker模式，erlang采用了另一种独特的方式实现了不停机发布。

> erlang VM为每个模块最多保存2份代码，当前版本'current'和旧版本'old'，当模块第一次被加载时，代码就是'current'版本。如果有新的代码被加载，'current'版本代码就变成了'old'版本，新的代码就成了'current'版本。erlang用两个版本共存的方法来保证任何时候总有一个版本可用，对外服务就不会停止。
> <br><br>—— 引自[分析erlang热更新实现机制](http://blog.csdn.net/mycwq/article/details/43372687)

#### 小结

不管是LB，还是Master/Worker，其基本思想都是在发布过程中，通过某种机制使得服务请求始终能够被系统的某个节点或者某个进程处理，从而保证了服务的可用性。
