### 1. 小红的定时任务秀逗了
今天搬砖大师小红又犯愁了，明明定时任务都按时调度了，但数据库里的数据却原封不动，好像定时任务从未执行过一样。这就好比明明干完活了，临了领工资的时候，包工头却说你没干活。

### 2. 案发现场
最近，老板想要知道每天系统增加了多少用户，以查看推广活动是否有带来用户量。这么关键的指标最终落到了小红身上。
老板这么信任自己，小红丝毫不敢怠慢。立马着手开始分析开发指标功能，考虑到用户数据表的数据量非常大，每次查看指标时，实时地进行汇总会花老板不少时间。
老板日理万机，一定得让老板有丝滑的用户体验。终于，小红思得一妙计，老板不是想要每天的用户增加量吗？那把每天的用户增加量形成一张统计表，老板在查看统计指标时，就直接从统计表拉取数据即可。按天的统计数据量很小，老板的丝滑用户体验也就得以实现了。
那谁来进行每天数据统计呢？小红是肯定不会干这种重复性的工作的，这辈子都不可能。那就交给定时任务来做吧。
说时迟，那时快，不一会小红就撸好了代码。
首先定义了一个用户注册量统计的定时任务调度器，如下：
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class UserStatisticScheduler {

    private final UserService userService;

    private final UserStatisticService userStatisticService;

    @Scheduled(cron = "1 0 0 * * *")
    public void userRegisterStatistic() {
        log.info("开始统计前一天用户注册量");
        int userRegisterCount = userService.count(LocalDateTime.now().minusDays(1));
        UserStatistic userStatistic = new UserStatistic(LocalDate.now().minusDays(1), userRegisterCount);
        userStatisticService.saveUserStatistic(userStatistic);
        log.info("前一天用户注册量统计完成");
    }
}
```
定时任务会在每天00:00:01进行前一天数据的统计。

其中`userStatisticService`底层直接的调用JPA实现：`UserStatisticRepository#save`进行保存，代码如下：
```java
@Repository
public interface UserStatisticRepository 
    extends CrudRepository<UserStatistic, Integer> {

}
```
很简单的逻辑对不对，小红也这么觉得。明天就能看到统计数据了，可以给老板一个惊喜。于是开心地提交了代码，就下班了。

第二天一早，小红早早地到了公司，迫不及待地打开系统查询昨天的用户注册量统计。咦，怎么没数据？难道定时任务没跑吗？
小红迅速打开日志，查阅具体情况：
```
2020-05-03 00:00:01.006  INFO 60094 --- [   scheduling-1] m.g.transaction.UserStatisticScheduler   : 开始统计前一天用户注册量
2020-05-03 00:00:01.143  INFO 60094 --- [   scheduling-1] m.g.transaction.UserStatisticScheduler   : 前一天用户注册量统计完成
```
从日志来看，定时任务执行了，并且没有任何异常信息。这是怎么回事呢？

### 3. 拨开乌云方见日
小红收拾了期待的心情，转而进行了问题排查。既然调用了`UserStatisticRepository#save`方法保存未生效，那么就从该`save`方法进行调试。很快小红定位到`save`方法最终会调用`AbstractSaveEventListener#performSaveOrReplicate`，如下：
```java
protected Serializable performSaveOrReplicate(
            Object entity,
            EntityKey key,
            EntityPersister persister,
            boolean useIdentityColumn,
            Object anything,
            EventSource source,
            boolean requiresImmediateIdAccess) {
    ...
    AbstractEntityInsertAction insert = addInsertAction(
            values, id, entity, persister, useIdentityColumn, source, shouldDelayIdentityInserts
    );

    ...
    return id;
}
```
这里我们只展现了部分重要代码，新增操作最终会映射为一个`InsertAction`，那个`InsertAction`会怎么处理呢？且看如下具体处理：
```java
private AbstractEntityInsertAction addInsertAction(
        Object[] values,
        Serializable id,
        Object entity,
        EntityPersister persister,
        boolean useIdentityColumn,
        EventSource source,
        boolean shouldDelayIdentityInserts) {
    if ( useIdentityColumn ) {
        EntityIdentityInsertAction insert = ...
        source.getActionQueue().addAction( insert );
        return insert;
    }
    else {
        EntityInsertAction insert = ...
        source.getActionQueue().addAction( insert );
        return insert;
    }
}
```
这里我们看到会将`InsertAction`添加到操作队列中，并没有真正执行新增操作。嗯，到目前还是符合我们预期的，在通常的事务隔离级别下，数据库更新操作都是会在事务提交的时候才会真实改变数据库的数据。
于是小红继续跟进事务提交的操作，在系统中小红使用事务管理器是`DataSourceTransactionManager`，具体配置如下：
```java
@Bean
@Primary
public PlatformTransactionManager transactionManager() {
    return new DataSourceTransactionManager(dataSource());
}
```

于是在事务提交时，可以看到如下操作（`DataSourceTransactionManager#doCommit`）：
```java
protected void doCommit(DefaultTransactionStatus status) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
    Connection con = txObject.getConnectionHolder().getConnection();
    if (status.isDebug()) {
        logger.debug("Committing JDBC transaction on Connection [" + con + "]");
    }
    try {
        con.commit();
    }
    catch (SQLException ex) {
        throw new TransactionSystemException("Could not commit JDBC transaction", ex);
    }
}
```
这个逻辑很简单，就是直接调用数据库连接的提交嘛。等等，那我们前面看到的操作队列里的那些操作呢？怎么没有任何地方去执行？怪不得，用户注册量统计没有写入到数据库。
这事务管理器`DataSourceTransactionManager`怎么不"务实"呢？此时，小红留意到事务管理器有很多实现，其中有一个叫`JpaTransactionManager`引起了小红的注意。为啥呢，因为我们数据库操作都是jpa呀。
闲话少叙，直接将`DataSourceTransactionManager`切换为`JpaTransactionManager`，再跑一次看看。果不其然，数据库里有数据了。说明`JpaTransactionManager`将操作队列中的操作都转换为了数据库操作，具体是怎么做的呢？如下为`JpaTransactionManager#doCommit`方法：
```java
protected void doCommit(DefaultTransactionStatus status) {
    JpaTransactionObject txObject = (JpaTransactionObject) status.getTransaction();
    ...
    try {
        EntityTransaction tx = txObject.getEntityManagerHolder().getEntityManager().getTransaction();
        tx.commit();
    }
    ...
}
```
我们可以看到，其不再是直接调用数据库连接的提交动作，而是使用`EntityTransaction`进行事务提交。进一步跟进`EntityTransaction#commit`，最终小红在`JdbcResourceLocalTransactionCoordinatorImpl#commit`中看到如下代码：
```java
public void commit() {
    ...
    JdbcResourceLocalTransactionCoordinatorImpl.this.beforeCompletionCallback();
    jdbcResourceTransaction.commit();
    JdbcResourceLocalTransactionCoordinatorImpl.this.afterCompletionCallback( true );
    ...
}
```
在提交事务之前，会回调`beforeCompletionCallback`，回调里会进行事务提交的准备，将操作队列里的操作都刷新为数据库操作，核心处理逻辑为`DefaultFlushEventListener#onFlush`:
```java
public void onFlush(FlushEvent event) throws HibernateException {
    ...
    flushEverythingToExecutions( event );
    performExecutions( source );
    postFlush( source );
    ...
}
```
其中`performExecutions(source)`方法逻辑为：
```java
protected void performExecutions(EventSource session) {
...
    session.getActionQueue().prepareActions();
    session.getActionQueue().executeActions();
...
}
```
终于，我们看到这里处理了操作队列，并将里面的所有操作都进行了执行。所以这样在真正事务提交时，可以将新增操作的数据提交到数据库了。

小红这里的问题就是使用了错误的事务管理器`DataSourceTransactionManager`，**不同的事务管理器使用场景如下**：
* JTA: `JtaTransactionManager`
* JDBC: `DataSourceTransactionManager`
* JPA: `JpaTransactionManager`
* Hibernate: `HibernateTransactionManager`

### 4. 真相大白了吗
前面小红分析出了问题的根因，并整理出了不同场景下应该使用的事务管理器。但小红并没有面露喜色，因为还有一个问题让小红不解。
既然`DataSourceTransactionManager`在JPA场景下会导致更新操作没法真实在数据库生效，那么之前其他的场景，比如用户注册是怎么将数据落库的呢？难道和定时调度的方式有关系吗？
且听小红后续带来的揭秘！！！