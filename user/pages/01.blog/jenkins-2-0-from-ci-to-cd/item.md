---
title: 【Jenkins】2.0新时代：从CI到CD
published: true
date: '01-05-2016 00:00'
visible: true
summary:
    enabled: '1'
    format: short
taxonomy:
    tag:
        - 原创
        - jenkins
---

#### 2.0 破茧重生

自从去年9月底Jenkins的创始人[Kohsuke Kawaguchi](https://github.com/kohsuke)提出Jenkins 2.0（后称2.0）的[愿景](https://docs.google.com/presentation/d/12ikbbQoMvus_l_q23BxXhYXnW9S5zsVNwIKZ9N8udg4/edit#slide=id.p)和[草案](https://groups.google.com/forum/#!topic/jenkinsci-dev/vbXK7JJekFw/overview)之后，整个Jenkins社区为之欢欣鼓舞，不管是官方博客还是Google论坛，大家都在热烈讨论和期盼2.0的到来。4月20日，历经Alpha(2/29)，Beta(3/24)，RC(4/7)3个版本的迭代，2.0终于正式发布。这也是Jenkins面世11年以来（算上前身Hudson）的首次大版本升级。那么，这次升级具体包含了哪些内容呢？

##### 外部

从外部来看，2.0最大的三个卖点分别是Pipeline as Code，全新的开箱体验和1.x兼容性。

**Pipeline as Code**是2.0的精髓所在，是帮助Jenkins实现CI(Continuous Integration)到CD(Continuous Delivery)华丽转身的关键推手。所谓Pipeline，简单来说，就是一套运行于Jenkins上的工作流框架，将原本独立运行于单个或者多个节点的任务连接起来，实现单个任务难以完成的复杂发布流程（例如下图）。Pipeline的实现方式是一套Groovy DSL(类似Gradle)，任何发布流程都可以表述为一段Groovy脚本，并且Jenkins支持从代码库直接读取脚本，从而实现了Pipeline as Code的理念。

![](https://jenkins.io/images/pipeline/realworld-pipeline-flow.png)

**全新的开箱体验**力图扭转我们印象中Jenkins十年不变的呆滞界面风格，不光Jenkins应用本身，官网排版、博客样式乃至域名都被重新设计。这些变化除了极大的改善了用户体验，更重要的是给人们传达一个清晰的信号，Jenkins不再仅仅是一个CI工具，而是蕴含着无限可能。

![](https://jenkins.io/images/2.0-create-item.png)

![](https://jenkins.io/images/2.0-config-dialog.png)

**1.x兼容性**给所有老版本用户吃了一颗大大的定心丸，注意，是完全兼容哦。

##### 内部

从内部来看，2.0主要包含了一些组件升级和JS模块化改造。

- 升级Servlet版本到3.1，获取Web Sockets支持
- 升级内嵌的Groovy版本到2.4.6
  - 未来版本的Jenkins将会[把Groovy彻底从内核中剥离](https://issues.jenkins-ci.org/browse/JENKINS-29068)，此次Groovy升级只是第一步
- 提供一个简化的JS类库给Plugin开发者使用

##### 更好的容器化支持

随着容器化技术（以Docker为代表）的不断升温，Jenkins紧随潮流，不仅同步上传2.0的Docker镜像，同时也在Pipeline中提供了默认的[Docker支持](https://jenkins.io/doc/pipeline/steps/docker-workflow/)。

除了上述内容，2.0还有一个比较有意思的改动，全局重命名Slave为Agent，看来在美国做IT政治正确性也是很重要啊。

#### Pipeline as Code

了解了2.0的概貌之后，回过来我们再看一下Pipeline as Code(后称Pipeline)产生的背景和具体构成。

##### 产生背景

作为2.0的核心插件，Pipeline并不是一个新事物，它的前身是[Workflow Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Workflow+Plugin)，而Workflow的诞生是受更早的[Build Flow Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Build+Flow+Plugin)启发，由[Nicolas De Loof](https://github.com/ndeloof)于2012年4月发布第一个版本。而纵观Jenkins的几个竞争对手（[Travis CI](https://docs.travis-ci.com/user/customizing-the-build/)、[phpci](https://www.phptesting.org/wiki/Adding-PHPCI-Support-to-Your-Projects)、[circleci](https://circleci.com/docs/configuration/)），Pipeline早已不是什么新鲜概念。可以说这次Jenkins 2.0的发布是顺势而为，同时也是大势所趋。

如果要在更大范围探讨Pipelined的产生背景，我认为有三个层面的原因。

- 第一层面，与不断增长的发布复杂度有关，其中一个典型场景就是灰度发布。原本只有大公司才有的灰度发布，随着敏捷开发实践的广泛采用、产品迭代周期的不断缩短、数据增长理念的深入人心，越来越多的中小公司也开始这一方面的探索，这对发布的需求也从点状的CI升级到线状的CD。这是Pipeline产生的第一个原因。
- 第二层面，与应用架构的模块化演变有关，以[微服务](http://martinfowler.com/articles/microservices.html)为代表，一次应用升级往往涉及到多个模块的协同发布，单个CI显然无法满足此类需求。这是Pipeline产生的第二个原因。
- 第三层面，与日益失控的CI数量有关。一方面，类似于Maven、pip、RubyGems这样的包管理工具使得有CI需求的应用呈爆发性增长，另一方面，受益于便捷的Git分支特性，即便对于同一个应用，往往也需要配置多个CI。随着CI数量的不断增长，集中式的任务配置就会成为一个瓶颈，这就需要把任务配置的职责下放到应用团队。这是Pipeline(as Code)产生的第三个原因。


##### 具体构成

说完背景，再看一下Pipeline的具体构成和特性。

基本概念：

- Stage: 一个Pipeline可以划分为若干个Stage，每个Stage代表一组操作。注意，Stage是一个逻辑分组的概念，可以跨多个Node。
- Node: 一个Node就是一个Jenkins节点，或者是Master，或者是Agent，是执行Step的具体运行期环境。
- Step: Step是最基本的操作单元，小到创建一个目录，大到构建一个Docker镜像，由各类Jenkins Plugin提供。

具体构成：

- Jenkinsfile: Pipeline的定义文件，由Stage，Node，Step组成，一般存放于代码库根目录下。
- Stage View: Pipeline的视觉展现，类似于下图。

![](QQ20160502-0.png)

2.0默认支持三种类型的Pipeline，普通Pipeline，Multibranch Pipeline和Organization Folders，后两种其实是批量创建一组普通Pipeline的快捷方式，分别对应于多分支的应用和多应用的大型组织。注意，要获取Organization Folders的支持需要额外安装Plugin。

值得一提的是，2.0有两个很重要的特性：

- Pausable: 类似于Bash的read命令，2.0允许暂停发布流程，等待人工确认后再继续，这个特性对于保证应用HA尤为重要。

![](QQ20160502-1.png)

- Durable: 发布过程中，如果Jenkins挂掉，正在运行中的Pipeline并不会受影响，也就是说Pipeline的进程独立于Jenkins进程本身。

##### 示例Pipeline

上文所涉及的示例Pipeline可以在我的[GitHub]((https://github.com/emac/pagination/blob/master/Jenkinsfile))找到，如果有问题想跟我探讨，可以加我QQ: 7789059。

#### 参考

- [The Need for Jenkins Pipeline](https://jenkins.io/blog/2016/04/15/the-need-for-pipeline/?utm_source=feedburner&utm_medium=feed&utm_campaign=Feed%3A+ContinuousBlog+%28Jenkins%29)
- [Jenkins 2.0](https://wiki.jenkins-ci.org/display/JENKINS/Jenkins+2.0)
- [Why Pipeline?](https://github.com/jenkinsci/pipeline-plugin/blob/master/TUTORIAL.md)
- [Pipeline-as-code with Multibranch Workflows in Jenkins](https://jenkins.io/blog/2015/12/03/pipeline-as-code-with-multibranch-workflows-in-jenkins/)
