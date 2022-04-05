### 1. 小红的疑惑
上回说到，还有一个问题让小红不解。
>既然`DataSourceTransactionManager`在JPA场景下会导致更新操作没法真实在数据库生效，那么之前其他的场景，比如用户注册是怎么将数据落库的呢？难道和定时调度的方式有关系吗？
为什么同样的事务管理器在不同场景下，会有不同的效果？

### 2. 重返现场
为了更好地描述清楚当时的情况，我们来一次情景重现。
首先，事务管理器使用的是错误的`DataSourceTransactionManager`:
```java
@Bean
@Primary
public PlatformTransactionManager transactionManager() {
    return new DataSourceTransactionManager(dataSource());
}
```
接着我们继续看下用户注册场景的代码情况，入口：
```java
@RestController
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @PostMapping("/users")
    public void createUser(@RequestBody @Valid User user) {
        userService.createUser(user);
    }
}
```

其中`userService#createUser`方法直接调用`UserRepository#save`, `UserRepository`具体如下：
```java
@Repository
public interface UserRepository extends CrudRepository<User, Integer> {

}
```

也是很简单的流程，对吧。和上回我们看到的定时统计用户注册量逻辑流程很类似，但为什么用户注册可以落库，而定时任务的用户统计却没能落库？

### 3. 真相只有一个
有了上次的经验，小红知道JPA操作在事务在提交时，才将对应的操作转换为真实数据库操作。这次小红直接从事务提交进行调查。小红这次站在更高的层面去看问题:`AbstractPlatformTransactionManager#commit`，这是事务提交时的入口。（_上回讲到的`DataSourceTransactionManager#doCommit`为事务提交中的一个环节，且`DataSourceTransactionManager#doCommit`也直接调用数据库连接进行事务提交，暂时没有什么值得我们深入下去了。_）
`AbstractPlatformTransactionManager#commit`内部会调用`processCommit`进行事务真实提交，具体如下：
```java
private void processCommit(DefaultTransactionStatus status) throws TransactionException {
    ...
    if (status.isNewTransaction()) {
        doCommit(status);
    }
    ...
    try {
        triggerAfterCommit(status);
    }
    finally {
        triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
    }
    ...
}
```
其中，`doCommit`为具体的事务管理器进行实现，这里也即`DataSourceTransactionManager#doCommit`。既然这一步没有将JPA的操作队列的操作转换为数据库操作，那么继续看`triggerAfterCommit`会触发什么后置操作。
事务提交后回调，最终会遍历当前线程的事务同步`TransactionSynchronization`，并调用其`afterCommit`。
难道这里会有什么猫腻吗？小红在这里添加了一个断点，来看看究竟有哪些事务同步，最终小红看到有如下两个同步器：
* ExtendedEntityManagerCreator.ExtendedEntityManagerSynchronization
* EntityManagerFactoryUtils.TransactionScopedEntityManagerSynchronization

先来后到，小红先拿`ExtendedEntityManagerCreator.ExtendedEntityManagerSynchronization`开刀：
```java
@Override
public void afterCommit() {
    super.afterCommit();
    try {
        this.entityManager.getTransaction().commit();
    }catch (RuntimeException ex) {
        throw convertException(ex);
    }
}
```
等等，第五行的这段代码不是和上回`JpaTransactionManager#doCommit`的逻辑一样吗？我们再来回顾下`JpaTransactionManager#doCommit`:
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
两者都是使用`EntityTransaction#commit`进行事务的提交，前面我们也聊到，会在`EntityTransaction#commit`中的真实提交事务之前的刷新操作中，会将JPA操作队列的操作转换为数据库操作，从而完成数据更新操作。

那这个同步器为什么没有帮助定时调度统计用户注册量的时候，生效对应的数据库操作呢？一番断点操作后，小红发现在定时任务中的数据库事务提交，并没有`ExtendedEntityManagerCreator.ExtendedEntityManagerSynchronization`，这什么情况？是谁，悄悄放了一个事务同步？

既然数据库同步器不知道是谁放进来的，那就对事务同步的注册环节进行断点，也即：`TransactionSynchronizationManager#registerSynchronization`。最终小红揭开了这位神秘侠的面纱：`EntityManagerFactoryUtils#doGetTransactionalEntityManager`，该工具方法逻辑较多，我们会省去无关当前问题的代码：
```java
public static EntityManager doGetTransactionalEntityManager(
        EntityManagerFactory emf, @Nullable Map<?, ?> properties, boolean synchronizedWithTransaction)
        throws PersistenceException {

    EntityManagerHolder emHolder = (EntityManagerHolder) TransactionSynchronizationManager.getResource(emf);
    if (emHolder != null) {
        if (synchronizedWithTransaction) {
            if (!emHolder.isSynchronizedWithTransaction()) {
                if (TransactionSynchronizationManager.isActualTransactionActive()) {
                    try {
                        emHolder.getEntityManager().joinTransaction();
                    }
                    catch (TransactionRequiredException ex) {
                        logger.debug("Could not join transaction because none was actually active", ex);
                    }
                }
                if (TransactionSynchronizationManager.isSynchronizationActive()) {
                    Object transactionData = prepareTransaction(emHolder.getEntityManager(), emf);
                    TransactionSynchronizationManager.registerSynchronization(
                            new TransactionalEntityManagerSynchronization(emHolder, emf, transactionData, false));
                    emHolder.setSynchronizedWithTransaction(true);
                }
            ...
        }
       ...
    }
    ...
    return em;
}
```

这里着重留意第5行和第11行代码：
1. 第5行，首先从事务同步的管理器中获取`EntityManagerHolder`
2. 第11行，如果有对应的资源，且该资源需要被同步但目前还未同步时，就将获取到的资源中的`EntityManager`加入到当前事务中。

加入事务的方式，为打开一个事务，并将`ExtendedEntityManagerSynchronization`注册到事务同步中。以配合我佛大慈大悲，错了，以配合事务提交后回调完成`EntityTransaction`的提交。具体逻辑如下：

```java
private void enlistInCurrentTransaction() {
    EntityTransaction et = this.target.getTransaction();
    et.begin();
    ...
    ExtendedEntityManagerSynchronization extendedEntityManagerSynchronization =
            new ExtendedEntityManagerSynchronization(this.target, this.exceptionTranslator);
    TransactionSynchronizationManager.bindResource(this.target, extendedEntityManagerSynchronization);
    TransactionSynchronizationManager.registerSynchronization(extendedEntityManagerSynchronization);
}
```
到这里，我们已经知道了谁来注册的`ExtendedEntityManagerSynchronization`。所以大功告成了吗？
我们是要调查为什么定时任务没有这个事务同步。
别急，我们回到`EntityManagerFactoryUtils#doGetTransactionalEntityManager`，前面说到会从事务同步的管理器中获取`EntityManagerHolder`这个资源，那么这个资源是谁绑定设置的呢？
老规矩，遇事不定上DEBUG。对资源的绑定`TransactionSynchronizationManager#bindResource`进行断点，最终神秘侠的幕后操作浮出了水面：`OpenEntityManagerInViewInterceptor#preHandle`:
```java
@Override
public void preHandle(WebRequest request) throws DataAccessException {
    ...
    if (TransactionSynchronizationManager.hasResource(emf)) {
        ...
    }
    else {
        ...
        TransactionSynchronizationManager.bindResource(emf, emHolder);
        ...
    }
}
```
我们看到这里，如果在当前事务同步管理器中没有找到`EntityManagerHolder`资源，则会进行绑定。
而`OpenEntityManagerInViewInterceptor`是`WebRequestInterceptor`的一种具体实现，为一种web请求的拦截器。
真相大白了，定时任务场景下并不是web请求，不会经过`OpenEntityManagerInViewInterceptor`，自然在搭配`DataSourceTransactionManager`时，也就没有神秘侠来对事务提交助一臂之力了。

插一句，上面提到的另外一个事务同步：`EntityManagerFactoryUtils.TransactionScopedEntityManagerSynchronization`中的`afterCommit`，会直接调用其父类：`ResourceHolderSynchronization#afterCommit`，而这也是`ExtendedEntityManagerCreator.ExtendedEntityManagerSynchronization`的父类。所以这个路人甲事务同步器就不再详细查看了。
### 4. 片尾彩蛋
`OpenEntityManagerInViewInterceptor`虽然帮助我们修正了`DataSourceTransactionManager` + JPA的组合下，事务提交的问题。但这是`OpenEntityManagerInViewInterceptor`的本意吗？那如果没有定时任务场景，我们是否还应该继续使用：`DataSourceTransactionManager` + JPA组合呢？

`OpenEntityManagerInViewInterceptor`在view层会持续持有可用的`EntityManager`，其目的是为了在view层可以访问懒加载的数据，而不用在service或者dao层将数据全部强制加载出来。在spring boot中，该拦截器默认是打开的，可以通过`spring.jpa.open-in-view`进行配置。

我们看到，`OpenEntityManagerInViewInterceptor`只是在解决另一方面的问题时，"顺带"也处理了`DataSourceTransactionManager` + JPA组合下的事务问题。并且该拦截器也只会在web请求场景下生效。在实际场景中，针对JPA，还是应该使用正确的事务管理器：`JpaTransactionManager`。