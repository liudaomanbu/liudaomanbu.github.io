---
layout:     post
title:      "debug记录:连接池和事务引发的bug"
subtitle:   "JUnit4测试框架中报数据库连接已关闭错误的一次debug"
date:       2016-9-12
author:     "caotc"
header-img: "img/home-bg.jpg"
tags:
    - Java
    - java
    - Debug
    - debug
    - Bug
    - bug
    - junit
    - Junit
    - transactional
    - Transactional
    - removeAbandoned
    - RemoveAbandoned
---


这两天在工作中遇到一个BUG.
项目里有一个定时计算任务,主要功能就是从原始数据表计算汇总一些数据然后更新到汇总表.这个定时任务是多线程同时进行计算更新的,因为要汇总的数据包括很多家企业的以天为单位的数据,单线程执行太耗费时间.
由于项目并没有为这个定时计算任务做允许手动执行的功能,所以项目在test中做了一个测试方法用作手动执行定时任务.
然而在计算任务跑完后控制台报错了如下错误:
> java.sql.SQLException: Connection has already been closed.

这个错误从字面上看,直接原因就是程序试图向数据库发送指令或数据,然后发现这个数据库连接实际上已经关闭了,于是报了使用的数据库连接已经关闭的错误.

**那么我的第一反应就是数据库的锅**,因为在项目的数据库配置中有
> edba.default.testOnBorrow="true"
>
> edba.default.validationQuery="SELECT 1"

这两行配置,testOnBorrow参数为true之后,从连接池池中取出数据库连接前会使用validationQuery参数配置的检查语句,进行检验该连接是否还保持连接状态,如果检验失败,则从池中去除连接并尝试取出另一个.

**也就是说这个参数为true已经保证了所有线程取得的数据库连接必然是正常的.**
**那么大概是数据库方面主动关闭了这个连接?**
但是我又转念一想,数据库主动关闭连接应该也只会发生在闲置了一段时间没有发送过命令的连接上,而这个项目的每个连接都在一直发送命令啊,**难道数据库会因为某个连接的连接时间过长而关闭连接?还是闲置连接参数被设的太短了,某个连接的sql执行时间太长,超过了闲置时间?**

不管怎么样,还是先查一下数据库吧.我们项目用的是mysql的数据库.
于是使用show variables去查询相关的参数,并且查看各参数的含义.
**结果发现wait_timeout这个参数是默认的28800秒即8小时,并且该参数是等待超时时间,即连接闲置时间.
mysql也没有自动关闭长时间的连接的参数.**

看来是我错怪数据库了,但是这样的话,就有点陷入僵局了,既然数据库没有关闭过连接,连接池又保证给定时任务线程的连接是保持连接状态的.那么为什么会发生连接已经关闭的情况呢?只能回去再仔细看看日志了.

> 14:37:30.671 [main] INFO  c.u.t.s.r.EnterpriseGroupAndRegionTask - electric stat report end! [2016-09-09 02:37:30]
>
> 14:37:30.673 [main] INFO  c.u.t.s.r.EnterpriseGroupAndRegionTask - electric stat report leader statistics use time :  36 second415 milliSecond

这几行日志是定时任务执行结束时打印的,定时任务都已经结束了,**线程都已经把连接还给连接池了,为什么还会用到数据库连接导致报错?**
直接DEBUG跟进去看一下吧.
找到报错的地方了,一看类名org.springframework.transaction.interceptor.TransactionAspectSupport
这是一个spring框架的类报的错,Transaction这个词有点眼熟,百度一查,事务(Transaction).
然后在定时任务测试启动类里看见,继承的类是AbstractTransactionalJUnit4SpringContextTests.
计算任务里面也使用了事务注解@Transactional(propagation = Propagation.REQUIRED, readOnly = false)
那么先找一下AbstractTransactionalJUnit4SpringContextTests这个类和AbstractJUnit4SpringContextTests类的区别在哪里.
> 测试类应该继承与 AbstractJUnit4SpringContextTests或AbstractTransactionalJUnit4SpringContextTests.
>
> 对于AbstractJUnit4springcontextTests和AbstractTransactionalJUnit4SpringContextTests类的选择：如果再你的测试类中,需要用到事务管理(比如要在测试结果出来之后回滚测试内容),就可以使用AbstractTransactionalJUnit4SpringTests类.
> AbstractTransactionalJUnit4SpringContextTests执行默认是会回滚

回滚事务?立刻倒回去看之前的报错日志.
果然这就是元凶.看来是测试方法执行完毕了之后,默认要进行回滚事务的操作,结果进行操作时发现连接已经关闭了.
但是这里还有一个问题,上面说过了,testOnBorrow="true"的配置已经保证了取得的连接是正常的,为什么后来又是报连接已经关闭呢?
**推测只能是,由于事务管理,会提前从连接池取得连接,然后在测试方法结束后再执行回滚事务操作,执行回滚操作时才发现连接已经关闭了.**
重新执行一次测试方法,来验证猜测.
看见了如下日志.
> 19:49:52.649 [main] DEBUG o.s.j.d.DataSourceTransactionManager - Acquired Connection [ProxyConnection[PooledConnection[com.mysql.jdbc.JDBC4Connection@4acb6d5]]] for JDBC transaction
>
> 19:49:52.649 [main] DEBUG o.s.j.d.DataSourceTransactionManager - Switching JDBC Connection [ProxyConnection[PooledConnection[com.mysql.jdbc.JDBC4Connection@4acb6d5]]] to manual commit

最后报错信息前的日志则是
> 20:03:23.702 [main] DEBUG o.s.j.d.DataSourceTransactionManager - Initiating transaction rollback
>
> 20:03:23.702 [main] DEBUG o.s.j.d.DataSourceTransactionManager - Rolling back JDBC transaction on Connection [ProxyConnection[null]]
>
> 20:03:23.712 [main] DEBUG o.s.j.d.DataSourceTransactionManager - Could not reset JDBC Connection after transaction

再重新执行一次测试方法,这次把计算的企业改为一家,测试是否报错,如果不报错的话,那么最后回滚事务使用的连接对象是不是和开始执行测试方法之前使用的是否是同一个对象.
结果如下
> 20:06:12.925 [main] DEBUG o.s.j.d.DataSourceTransactionManager - Creating new transaction with name [testRun]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT; ''
>
> 20:06:12.925 [main] DEBUG o.s.j.d.DataSourceTransactionManager - Acquired Connection [ProxyConnection[PooledConnection[com.mysql.jdbc.**JDBC4Connection@48b4da9b**]]] for JDBC transaction
>
> 20:06:12.925 [main] DEBUG o.s.j.d.DataSourceTransactionManager - Switching JDBC Connection [ProxyConnection[PooledConnection[com.mysql.jdbc.**JDBC4Connection@48b4da9b**]]] to manual commit
>
> 20:06:15.925 [main] DEBUG o.s.j.d.DataSourceTransactionManager - Initiating transaction rollback
>
> 20:06:15.925 [main] DEBUG o.s.j.d.DataSourceTransactionManager - Rolling back JDBC transaction on Connection [ProxyConnection[PooledConnection[com.mysql.jdbc.**JDBC4Connection@48b4da9b**]]]
>
> 20:06:15.926 [main] DEBUG o.s.j.d.DataSourceTransactionManager - Releasing JDBC Connection [ProxyConnection[PooledConnection[com.mysql.jdbc.**JDBC4Connection@48b4da9b**]]] after transaction

很明显,我的猜测正确,回滚事务所用的连接是在执行测试方法之前就获取的,然后由于执行测试方法时间太长,期间这个连接被关闭了,所以最后回滚事务之前报错-连接已关闭.

那么又回到原来的问题,为什么这个连接被关闭了?
根据目前情况来看,mysql没有进行关闭连接的操作,只能是连接池做的.再去检查连接池的配置.
看见了这几行配置.
edba.default.removeAbandonedTimeout="300"
edba.default.removeAbandoned="true"
百度查询参数含义.
> 有时粗心的程序编写者在从连接池中获取连接使用后忘记了连接的关闭,这样连池的连接就会逐渐达到maxActive直至连接池无法提供服务.现代连接池一般提供一种“智能”的检查,但设置了removeAbandoned="true"时,当连接池连接数到达(getNumIdle() < 2) and (getNumActive() > getMaxActive() - 3)时便会启动连接回收,那种活动时间超过removeAbandonedTimeout="60"的连接将会被回收,同时如果logAbandoned="true"设置为true,程序在回收连接的同时会打印日志.removeAbandoned是连接池的高级功能,理论上这中配置不应该出现在实际的生产环境,因为有时应用程序执行长事务,可能这种情况下,会被连接池误回收,该种配置一般在程序测试阶段,为了定位连接泄漏的具体代码位置,被开启,生产环境中连接的关闭应该靠程序自己保证.

那么真相大白,数据库的这个配置代表了超过300秒即5分钟的数据库连接将会被回收.
由于DataSourceTransactionManager这个类为了回滚事务而提前从连接池请求了数据库连接,但是由于测试方法耗时太长,所以这个长时间没有活动的连接超过了300秒后被连接池回收.最后回滚事务时报错.

**那么解决方法自然很简单:**

**1.将事务操作去掉**

**2.将removeAbandoned参数改为false或者removeAbandonedTimeout的参数设置时间更长.**

因为这个测试类本来就是当做让定时任务手动执行的方法存在的,并不怕产生脏数据,也不需要回滚,所以自然我的做法就是把AbstractTransactionalJUnit4SpringContextTests类改为AbstractJUnit4SpringContextTests类.
好的,就这样愉快的修改掉测试类的父类.
然后再跑一次.

结果居然又报错了.
> 20:22:28.852 [main] DEBUG o.s.j.d.DataSourceTransactionManager - Initiating transaction commit
>
> 20:22:28.852 [main] DEBUG o.s.j.d.DataSourceTransactionManager - Committing JDBC transaction on Connection [ProxyConnection[null]]
>
> 20:22:28.862 [main] DEBUG o.s.j.d.DataSourceTransactionManager - Could not reset JDBC Connection after transaction
>
> java.sql.SQLException: Connection has already been closed.

很明显,还是同样的原因,只不过是变成要确认事务了.
仔细在测试方法里寻找问题,一下就找到了
> @Transactional(propagation = Propagation.REQUIRED, readOnly = false)

很明显,这是事务的注解,REQUIRED代表业务方法需要在一个事务中运行,如果方法运行时,已处在一个事务中,那么就加入该事务,否则自己创建一个新的事务.这是spring默认的传播行为.readOnly=true只读,不能更新,删除.
好吧,方法级事务还是需要的,如果计算任务出错,的确不应该提交事务.
如果删除这个注解,那么就变成每句sql单独持有事务了.
只能选择方法二,去把removeAbandoned参数改为false.

再运行一次,顺利结束,问题解决.