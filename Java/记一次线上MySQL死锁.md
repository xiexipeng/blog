# 记一次线上MySQL死锁

> MySQL在不同的事务隔离级别下，为了避免出现幻读、脏读等情况，在读写数据的时候会对某一行或者一系列行数据进行加锁，锁的种类包括共享锁和独占锁。本文基于一次线上数据库出现的死锁场景，分析了MySQL的加锁原理，对MySQL的死锁场景进行了重现。

## 线上数据库死锁回顾

线上有用户反馈某个数据状态不对，针对当时的业务场景，立马就定位到相关的服务和相关的接口，直接去线上服务器上查看接口的调用日志，发下接口报错了，错误日志如下：

```java
Cause: com.mysql.jdbc.exceptions.jdbc4.MySQLTransactionRollbackException: Deadlock found when trying to get lock; try restarting transaction
; SQL []; Deadlock found when trying to get lock; try restarting transaction; nested exception is com.mysql.jdbc.exceptions.jdbc4.MySQLTransactionRollbackException: Deadlock found when trying to get lock; try restarting transaction
	at org.springframework.jdbc.support.SQLErrorCodeSQLExceptionTranslator.doTranslate(SQLErrorCodeSQLExceptionTranslator.java:263)
	at org.springframework.jdbc.support.AbstractFallbackSQLExceptionTranslator.translate(AbstractFallbackSQLExceptionTranslator.java:73)
	at org.mybatis.spring.MyBatisExceptionTranslator.translateExceptionIfPossible(MyBatisExceptionTranslator.java:73)
	at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:446)
	at com.sun.proxy.$Proxy74.update(Unknown Source)
	at org.mybatis.spring.SqlSessionTemplate.update(SqlSessionTemplate.java:294)
	at org.apache.ibatis.binding.MapperMethod.execute(MapperMethod.java:63)
	at org.apache.ibatis.binding.MapperProxy.invoke(MapperProxy.java:59)
	at com.sun.proxy.$Proxy75.updateById(Unknown Source)
```

通过异常日志可以看到是在更新数据的时候出现了死锁，这个时候去查看接口的具体实现逻辑，

```java
@Override
@Transactional(rollbackFor = Throwable.class)
public void batchUpdateCarLoanOrderInfo(List<CarLoanOrderDetailDTO> carLoanOrderDetailDTOS) {
    if (CollectionUtils.isEmpty(carLoanOrderDetailDTOS)) {
        return;
    }
    carLoanOrderDetailDTOS.parallelStream().forEach(carLoanOrderDetailDTO -> updateCarLoanOrderInfo(carLoanOrderDetailDTO));
}
```

这段代码的核心逻辑就是遍历订单集合，依次更新每一条记录的状态，并且开启了事务异常回滚。通过查看调用日志发现，在很短的时间内，该接口被调用了两次，且两次更新的数据是一样的，通过由于开启了事务异常回滚，事务的提交不是自动提交，依次更新请求必须等所有列表全部执行完才提交事务，这里就可能发生死锁，具体流程如下：

![dead_lock.png](https://i.loli.net/2019/07/20/5d32ca8840abd74724.png)

大致过程就是当开启两个事务同时更新记录A和记录B的时候，事务1更新A记录，同时事务B更新B记录，当事务1再想去更新B记录的时候，必须等待事务2提交，同样的，事务2想要更新A记录的时候，也必须等待事务1的提交，这个时候就事务1和事务2就出现了死锁。

**解决办法：**

1. 去掉事务异常回滚，也就是去掉`@Transactional(rollbackFor = Throwable.class)`，使每次更新变成自动提交事务，记录A依旧是更新两次，但是每次更新自动开启事务，自动提交
2. 更合理的方法是对需要更新的集合进行排序，按相同的顺序更新，这个也是解决死锁问题的方案之一



## MySQL加锁原理分析

mysql在并发事务执行的时候可能会出现脏写、脏读、幻读和重复读的情况，为了避免这些事务读写问题，mysql提供了两种解决方案：

1. 通过MVVC版本链来控制，记录每个事务对数据的操作记录，同一行记录的所有历史版本将形成一个版本链，针对不同的事务隔离级别，定义不同的规则，决定查询的是版本链上的哪一行数据
2. 通过锁的方式实现

当一个事务修改某一行数据的时候，会同时生成一个锁结构关联这一行数据记录，锁结构简单来说包含一个事务ID和一个是否等待字段，当一个事务1修改某行记录A的之后，在它提交之前，有其它事务例如事务2也想操作A的话，事务2将获取锁失败，进入等待状态，当事务1提交之后，事务2才可以继续操作。通过锁我们也可以实现并发事务的脏读、脏写等问题。例如当一个事务在操作一行数据的时候，另一个事务想要读取这一行数据记录，只能等待当前事务处理完毕（当前事务有可能提交，也有可能回滚），这也就避免了脏读问题，避免后面的事务读取到前一个事务未提交的数据。

![lock_and_tx](https://i.loli.net/2019/07/20/5d32de55a64c283561.png)



###  MySQL的锁类型

MySQL的锁类型大类可以分为两种，分别是**`只读锁`**和**`读写锁`**。

只读锁：当事务1对某一行记录A加上只读锁之后，其它事务可以获取这一行记录A的只读锁，但是无法获取记录A读写锁。

读写锁：当事务1对某一行记录A加上读写锁之后，其它事务既不能获取记录A的只读锁，也无法获取记录A的读写锁。

1. 行级锁：当事务1获取到某一行记录A的行级锁后，其它事务不能获取记录A的行级锁
2. 间隙锁

**加锁的方式**

1. `select * from lock_demo where id = 1 for update;`
2. `select * from lock_demo where id = 1 lock in share mode;`



## MySQL死锁场景