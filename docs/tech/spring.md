#### 项目中遇到过的问题

1.注意 @Transactionnal 注解失效的几个场景。因为注解是通过代理对象来实现的，因此当一个方法内部引用另外一个带有注解的方法是，实际上引用的对象 this 是当前类，而不是代理对象，因此注解失效

spiringboot 的解决方法：

通过AopContext.currentProxy获取当前代理对象，通过代理对象调用方法。最好的方法是避免在方法内部调用。

```java
public class UserService{
    @Transactional
    public void hello(){
        System.out.println("开始hello");
        try {
            //通过代理对象去调用saveUser()方法   
//直接 saveuser Transactional注解是失效的        
            (UserService)AopContext.currentProxy().saveUser();
        } catch (Exception  e) {
            logger.warn("发送消息异常");
        }
    }
    
    @Transactional
    public void saveUser(){
        User user = new User();
        user.setName("zhangsan");
        System.out.println("将用户存入数据库");
    }
//注意 ， 下面这种方式，虽然B的注解失效了，但是A上面的注解还是生效的
@TestAnnotation
pubic void A(){
	B()
}
@TestAnnotation
pubic void B(){
	xxxxxx
}
或者：
SpringUtils.getBean(this.getClass()).xxxx
SpringUtils.getBean(this.getClass()) 这个方法会返回通过cglib增强的代理类
而this.getClass 只会返回当前类

```

还有一种方法，之前在当前类中注入代理类

```java
public class KsOnboardingCandidateInfoServiceImpl implements
        KsOnboardingCandidateInfoService{
// 这个注入的不是当前类，是当前类的cgilib增强的代理类		
@Autowired
    private KsOnboardingCandidateInfoService candidateInfoService
//然后用这个类去调用在相应的方法中调用带有注解的方法
}
```

2. @schedule 注解 

如果未指定cron表达式，项目启动后会马上执行一次Task，这个延时就是项目启动后第一次执行的延时 即 `initialDelay` 所对应的时间

3. 在通过注解为字段加密的时候的一个坑：

```java
KsOnboardingBankInfo ksOnboardingBankInfo = 
bankInfoService.selectById(54257L);
ksOnboardingBankInfo.setBankNumber("11111111111111111113");
  // updateById 这里加的注解会导致上面的ksOnboardingBankInfo
  //对应的字段值改变
bankInfoService.updateById(ksOnboardingBankInfo);
 KsOnboardingBankInfo ksOnboardingBankInfo1 = bankInfoService.selectById(54257L);
```

本质是带有注解的方法，在执行前，会对该对象加了加密字段的属性进行加密，修改的属性值。因此如果调用方后的的代码又用到了此对象，会导致某个属性值发生变化了。

解：

```java
@Arround 
注解可以在方法调用前和方法调用后做一些操作。
这里可以在方法调用后对入参修改的字段值进行回滚即可
```

4. spirng 可以通过注解 @`RestControllerAdvice`  + `@ExceptionHandler`统一处理各种异常

##### AOP
几个术语

1. 连接点  Joinpoint

指程序执行过程中的一些点，比如方法调用，异常处理等。在 Spring AOP 中，仅支持方法级别的连接点。

连接点的定义：

```java
public interface Joinpoint {

    /** 用于执行拦截器链中的下一个拦截器逻辑 */
    Object proceed() throws Throwable;

    Object getThis();

    AccessibleObject getStaticPart();

}
```

1. 切点  pointcut

用于选择连接点的。spring中pointcut 定义：

```java
public interface Pointcut {

    /** 返回一个类型过滤器 */
    ClassFilter getClassFilter();

    /** 返回一个方法匹配器 */
    MethodMatcher getMethodMatcher();

    Pointcut TRUE = TruePointcut.INSTANCE;
}

public interface ClassFilter {
    boolean matches(Class<?> clazz);
    ClassFilter TRUE = TrueClassFilter.INSTANCE;

}

public interface MethodMatcher {
    boolean matches(Method method, Class<?> targetClass);
    boolean matches(Method method, Class<?> targetClass, Object... args);
    boolean isRuntime();
    MethodMatcher TRUE = TrueMethodMatcher.INSTANCE;
}
```

通常是用 AspectJ 表达式对连接点进行选择

1. 通知

通知 Advice 即我们定义的横切逻辑。如果说切点解决了通知在哪里调用的问题，那么现在还需要考虑了一个问题，即通知在何时被调用？是在目标方法前被调用，还是在目标方法返回后被调用，还在两者兼备呢？Spring帮我们定义了以下几种通知类型：

- 前置通知（Before advice）- 在目标方便调用前执行通知
- 后置通知（After advice）- 在目标方法完成后执行通知
- 返回通知（After returning advice）- 在目标方法执行成功后，调用通知
- 异常通知（After throwing advice）- 在目标方法抛出异常后，执行通知
- 环绕通知（Around advice）- 在目标方法调用前后均可执行自定义逻辑
1. 切面 Aspect

切面 Aspect 整合了切点和通知两个模块，切点解决了 where 问题，通知解决了 when 和 how 问题。切面把两者整合起来，就可以解决 对什么方法（where）在何时（when - 前置还是后置，或者环绕）执行什么样的横切逻辑（how）的三连发问题。

1. 织入  Weaving

织入就是将通知逻辑插入到方法调用上，使得我们的通知逻辑在方法调用时得以执行。

先来说说以何种方式进行织入，这个方式就是通过实现后置处理器 BeanPostProcessor 接口。该接口是 Spring 提供的一个拓展接口，通过实现该接口，用户可在 bean 初始化前后做一些自定义操作。那 Spring 是在何时进行织入操作的呢？答案是在 bean 初始化完成后，即 bean 执行完初始化方法（init-method）。Spring通过切点对 bean 类中的方法进行匹配。 若匹配成功，则会为该 bean 生成代理对象，并将代理对象返回给容器 。容器向后置处理器输入 bean 对象，得到 bean 对象的代理，这样就完成了织入过程。

