在一个风和日丽的下午，笔者正在开开心心地撸着代码。一切都那么水到渠成，好了，写完，坐等启动...
嗯？？？启动异常，乍一看`NPE`，再一看，Spring依赖的bean属性没有注入？
为了描述具体的场景，这里将问题用示例代码抽取出来。
我们有两个Spring Bean，`CandidateOne`和`CandidateTwo`，具体代码如下：
#### CandidateOne:

```
/**
 * 候选者1，依赖候选者2
 *
 * @author gavincook
 * @version $ID: CandidateOne.java, v0.1 2018-06-13 09:38 gavincook Exp $$
 */
@Component
public class CandidateOne implements ApplicationContextAware, Ordered {

    @Autowired
    private CandidateTwo candidateTwo;

    /**
     * 实例测试方法
     */
    public void testMethodForOne() {
        System.out.println(candidateTwo.toString());
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        candidateTwo.testMethodForTwo();
    }

    @Override
    public int getOrder() {
        return 1;
    }
}
```
#### CandidateTwo:

```
/**
 * 候选者2，依赖后选择1
 *
 * @author gavincook
 * @version $ID: CandidateTwo.java, v0.1 2018-06-13 09:38 gavincook Exp $$
 */
@Component
public class CandidateTwo implements ApplicationContextAware, Ordered {

    @Autowired
    private CandidateOne candidateOne;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        candidateOne.testMethodForOne();
    }

    /**
     * 实例测试方法
     */
    public void testMethodForTwo() {
        System.out.println(candidateOne.toString());
    }

    @Override
    public int getOrder() {
        return 2;
    }
}
```
可以看到两个Bean相互依赖，也即Spring中的循环依赖。启动后会在`CandidateOne#testMethodForOne`方法中抛出`NPE`异常，也即`candidateOne`对象并没有拿到`candidateTwo`属性对应的依赖。
似乎不太对，`ApplicationContextAware`切入点用于对应的Bean可以感应Spring上下文`ApplicationContext`，并且该切入点是会在Bean的属性注入之后调用。在`ApplicationContextAware`切入点进行了依赖实例的方法调用，拿到的实例怎么会是还没有完成依赖注入的实例（`candidateOne`中的`candidateTwo`为`null`）呢？

下面我们详细来分析下两个单例Bean的加载过程，这里由于我们通过`Ordered`接口指定了加载顺序。（`candidateOne`会先于`candidateTwo`加载）
1. 实例化`candidateOne`（调用CandidateOne无参构造函数）
2. 处理属性（依赖注入）--获取或创建上下文中`candidateTwo`对象
3. 实例化`candidateTwo`（调用CandidateTwo的无参构造函数）
4. 处理属性（依赖注入）--获取或创建上下文中`candidateOne`对象
5. 这时，会请求再次创建`candidateOne`对象，出现的同一个单例对象多次申请创建，抛出循环依赖异常
6. Spring检测到循环依赖异常时，则对需要循环依赖的对象`candidateOne`提前暴露（也即仅仅实例化，并没有完成属性的注入），提前暴露的对象会放入`DefaultSingletonBeanRegistry#earlySingletonObjects`中。
7. 提前暴露后，循环依赖处所得到的对象就是实例化但并没有完成属性注入的对象，也即`candidateTwo`中的`candidateOne`对象为刚刚初始化，并没有解决`candidateOne`内部属性的注入（`candidateTwo`属性为`null`）。此时的对象其实是不完整的，但总比没有对象好。
8. 然后继续走`candidateTwo`这个Bean的声明周期`ApplicationContextAware`切入点，此时在该切入点拿注入的属性`candidateOne`进行调用，那么其实是不完整的，也就出现了前文提到的`NPE`异常

如果在`candidateTwo`中的`ApplicationContextAware`切入点没有使用没有完全准备好的`candidateOne`，那么`candidateTwo`会在完全初始化好后，注入到`candidateOne`中，也即在`candidateOne`中的`ApplicationContextAware`切入点使用`candidateTwo`是安全的。

### 总结一下
1. `ApplicationContextAware`切入点会在Bean属性注入后进行调用，这里的Bean是指当前实现`ApplicationContextAware`接口的Bean。并不是指所有的Spring Bean。
2. `ApplicationContextAware`切入点由于在Bean完全准备好之前（如果不清楚Bean的声明周期，请自行补习），那么该切入点拿到的其他Bean对象有可能是不安全的，需要自行校验或者使用其他切入点。
3. 循环依赖，和Bean的加载顺序有关系。如果我们这里`candidateTwo`先加载，那么异常抛出点则会在`CandidateTwo#testMethodForTwo`处。
4. 对于构造方法的循环依赖，Spring无法解决。因为提前暴露未完全准备好的Bean需要调用构造方法。
5. 如果需要在启动就绪后使用Bean做一些逻辑，可以使用`ApplicationListener`切入点。


