---
title: 【Spring】单应用多数据库的事务管理
published: true
date: '27-03-2016 00:00'
visible: true
summary:
    enabled: '1'
    format: short
taxonomy:
    tag:
        - 原创
        - database
---

#### 单应用多数据库的事务管理

[上篇](http://emacoo.cn/blog/spring-boot-multi-db)讲到单应用多数据库的配置，这次我们聊聊单应用多数据库的事务管理。首先我们来了解一下事务。

#### 什么是数据库事务？

> 数据库事务(Database Transaction) ，是指作为单个逻辑工作单元执行的一系列操作，要么完全地执行，要么完全地不执行。事务处理可以确保除非事务性单元内的所有操作都成功完成，否则不会永久更新面向数据的资源。一个逻辑工作单元要成为事务，必须满足所谓的ACID（原子性、一致性、隔离性和持久性）属性。
> http://baike.baidu.com/view/1298364.htm

举个栗子，银行的一次转账操作就可以理解成一个事务，A打钱给B，银行首先从A的账户里扣钱，然后把钱转到B的账户。如果只执行前一步，A肯定不乐意，如果是后一步，换银行不乐意。所以两步要么都执行，要么都不执行。

#### 单库事务和跨库事务有什么区别？

一般而言，所谓的数据库事务都是针对单个数据库的事务，即单库事务。而跨库事务，顾名思义，是指涉及多个数据库的事务，理论上也必须满足ACID属性。两者最核心的区别在于，单库事务一般是由数据库保证的，俗称物理事务，而跨库事务一般是由应用保证的，俗称逻辑事务。与单库事务相比，跨库事务执行成本高，稳定性差，管理也更复杂，但在某些场景下，尤其是分布式应用环境下，又是不得不使用的技术。

再举个栗子，单库事务好比你从北京飞上海，到东航官网买张票就搞定了，而跨库事务好比北京飞纽约，到上海转机，就得买东航转上航的联票，出票就转由携程保证了。

#### 多数据库下的三种事务使用场景

了解了单库事务和跨库事务之后，我们再来看看多数据库下的三种事务使用场景。假设有DB1，DB2两个数据库，分别对应ServiceA和ServiceB两个带上事务注解的服务类，根据调用关系，可细分为三种场景。

##### 场景一：仅调用ServiceA，ServiceA不调用ServiceB

这种情况等同于单库事务，无需特殊处理。

##### 场景二：仅调用ServiceA，ServiceA再调用ServiceB

![](QQ20160327-0.png)


##### 场景三：先调用ServiceA，再调用ServiceB

![](QQ20160327-1.png)

场景二和场景三是两种典型的跨库事务，Spring默认的事务管理并无法保证事务的属性。对于场景二，在调用ServiceB之后，如果ServiceA出错，ServiceB并不会回滚。而对于场景三，在调用ServiceB之前，ServiceA的事务已经完成，因此当ServiceB出错回滚时，ServiceA并不会同步回滚。

如何解决？前面说过，跨库事务一般是由应用保证，因此办法有很多。标准的方法是使用JTA框架进行两段式提交，比如开源的[Atomikos](https://www.atomikos.com/Main/WebHome)，[Bitronix](https://github.com/bitronix/btm)。粗暴一点，可以显式创建两个事务，将所有的服务调用包在其中。考虑到本文单应用的环境，还有第三种方式，根据所涉及的事务列表，动态构造调用链，把所有的服务调用封装到最内层，由外层的事务注解链保证跨库事务。

定义事务代理类，每一个类代理一个数据库事务：
    
    @Component
    public class Db1TxBroker {
        @Transactional(DbConstants.TX_DB1)
        public <V> V inAccount(Callable<V> callable) {
            try {
                return callable.call();
            } catch (Exception e) {
                throw new ServiceException(e);
            }
        }
    }

负责生成调用链的服务基类：

    public abstract class BaseComboService {

        @Autowired
        private Db1TxBroker db1TxBroker;

        @Autowired
        private Db2TxBroker db2TxBroker;

        /**
         * 根据传入的事务链构造调用链,在最内层调用包含业务逻辑的callable.
         *
         * @param callable
         * @param txes 所涉及的完整事务列表(顺序无关)
         */
        protected <V> V combine(Callable<V> callable, TX... txes) {
            if (callable == null) {
                return null;
            }

            Callable<V> combined = Stream.of(txes).filter(Objects::nonNull).distinct().reduce(callable, (r, tx) -> {
                switch (tx) {
                    case DB1:
                        return () -> db1TxBroker.inDb1(r);
                    case DB2:
                        return () -> db2TxBroker.inDb2(r);
                    default:
                        // should not happen
                        return null;
                }
            }, (r1, r2) -> r2);
            try {
                return combined.call();
            } catch (Exception e) {
                throw new ServiceException(e);
            }
        }
    }

使用示例：

    @Service
    public class DemoComboService extends BaseComboService {

        @Autowired
        private ServiceA serviceA;

        @Autowired
        private ServiceB serviceB;

        public void demo() {
            combine(() -> {
                serviceA.flyToShanghai();
                serviceB.flyToNewYork();
                return null;
            }, TX.DB1, TX.DB2);
        }
    }

相比JTA，上述第三种方法最大的优点是更轻量，配置更简单，但只能工作在单个应用的环境。对于分布式应用，后者就无能为力了。这种方法本质上还是借助Spring的事务注解来保证跨库事务，如果将来Spring的事务注解支持JDK8的@Repeatable特性，那就可以直接在方法上加上多个事务注解来达到同样目的。

#### 参考

- [Configuring Spring and JTA without full Java EE](https://spring.io/blog/2011/08/15/configuring-spring-and-jta-without-full-java-ee/)
- [Spring的事务之编程式事务](http://jinnianshilongnian.iteye.com/blog/1441271)


