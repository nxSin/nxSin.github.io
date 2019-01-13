---

layout: post
title: Android-AOP
date: 2019-01-13
categories: [其他]
author: 小鬼
tags: [AOP]
description: Android AOP使用介绍

---

AOP，面向切面的方式，用起来是非常的棒。

### 一. 介绍

以下介绍的其中部分内容是摘抄修改自网络：

AOP为Aspect Oriented Programming的缩写，意为：面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。

它并没有帮助我们解决任何新的问题，它只是提供了一种更好的办法，能够用更少的工作量来解决现有的一些问题，并且使得系统更加健壮，可维护性更好。同时，它让我们在进行系统架构和模块设计的时候多了新的选择和新的思路。

AOP通过对切面进行切入执行代码，达到对原代码无侵入，比如如下图：有A->B，C->D流程，在不改变原流程代码情况下，咱们可以横向插入Aspect流程执行。

![AOP](https://github.com/nxSin/BlogPic/blob/8a80cd47bc3aec79a83ece76fc92ebf596208cd0/Android/AOP/introduce.png?raw=true)

使用AOP，咱们这里将讲一个AOP框架[Aspectj](https://www.eclipse.org/aspectj/)，咱们利用它来实现AOP。

1. AspectJ是什么

AspectJ是一个代码生成工具（Code Generator）。 
AspectJ语法就是用来定义代码生成规则的语法。您如果使用过Java Compiler Compiler (JavaCC)，您会发现，两者的代码生成规则的理念惊人相似。 
AspectJ有自己的语法编译工具，编译的结果是Java Class文件，运行的时候，classpath需要包含AspectJ的一个jar文件（Runtime lib）。 
AspectJ和xDoclet的比较。AspectJ和EJB Descriptor的比较。

2. AspectJ能干什么？

 AOP是Object Oriented Programming（OOP）的补充。
OOP能够很好地解决对象的数据和封装的纵向问题，却不能很好的解决Aspect（"方面"）分离的横向问题

3. 使用价值

认证、事务、日志等等

如：你已经写好一个功能，有一天客户提出一个需求，需要对调用这个服务的用户进行权限认证，这时候通过AspectJ实现一个AOP是你的首选。



### 二. 使用

#### (一) 切入点Pointcuts

咱们要切入执行代码，需要有一个切入点，可以作为切入点的描述，[Pointcuts](https://www.eclipse.org/aspectj/doc/released/progguide/quick.html#quick-pointcuts)，以下列举几个常用的Pointcuts：

常用Pointcuts| 使用|说明
---|---|---
方法与构造方法(Methods and Constructors) |Call 或者 execution| 在切入点方法前后插入
字段(Fields) | get 或 set | 通过插入字段相应的get与set方法，来达到对字段值修改和返回的变相处理
异常(Exception Handlers)| handler| 通过handler方式，得到异常信息，做一些操作

#### (二) 切入点使用介绍

咱们主要切入的都是在方法上，构造方法也好普通方法也好，接下来看下如何使用进行切入代码执行。

##### 1. 配置Gradle依赖

咱们这里以Android代码为例：

1. Project的build.gradle中添加

```
buildscript {
    dependencies {
        classpath group: 'org.aspectj', name: 'aspectjtools', version: '1.9.2'
        // https://mvnrepository.com/artifact/org.aspectj/aspectjweaver
        classpath group: 'org.aspectj', name: 'aspectjweaver', version: '1.9.2'
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}
```

2. App工程的build.gradle中添加依赖

```
//aspectj库
    implementation group: 'org.aspectj', name: 'aspectjrt', version: '1.9.2'
```

3. App工程的build.gradle中添加配置

```
/*Aspectj配置*/
import org.aspectj.bridge.IMessage
import org.aspectj.bridge.MessageHandler
import org.aspectj.tools.ajc.Main

//log 打印工具和构建配置
final def log = project.logger
final def variants = project.android.applicationVariants

variants.all { variant ->
    // 通过debug判断是否需要打入aspectj，比如咱们切入的是打印执行时间工具代码，那如果release不需要打则return掉
    if (!variant.buildType.isDebuggable()) {
        log.debug("Skipping non-debuggable build type '${variant.buildType.name}'.")
        return;
    }

    //执行到这里，使 aspectj 配置生效
    JavaCompile javaCompile = variant.javaCompile
    javaCompile.doLast {
        String[] args = ["-showWeaveInfo",
                         "-1.8",
                         "-inpath", javaCompile.destinationDir.toString(),
                         "-aspectpath", javaCompile.classpath.asPath,
                         "-d", javaCompile.destinationDir.toString(),
                         "-classpath", javaCompile.classpath.asPath,
                         "-bootclasspath", project.android.bootClasspath.join(File.pathSeparator)]
        log.debug "ajc args: " + Arrays.toString(args)

        MessageHandler handler = new MessageHandler(true);
        new Main().run(args, handler);
        //在编译时打印信息如警告、error 等等
        for (IMessage message : handler.getMessages(null, true)) {
            switch (message.getKind()) {
                case IMessage.ABORT:
                case IMessage.ERROR:
                case IMessage.FAIL:
                    log.error message.message, message.thrown
                    break;
                case IMessage.WARNING:
                    log.warn message.message, message.thrown
                    break;
                case IMessage.INFO:
                    log.info message.message, message.thrown
                    break;
                case IMessage.DEBUG:
                    log.debug message.message, message.thrown
                    break;
            }
        }
    }
}
```

##### 2. 切入执行使用方式

1. 创建切入处理对象，使用@Aspect注解供库识别，比如：

```
@Aspect
public class AspectJAfterBefore {
}
```

2. 在第1步类中声明切入点，使用@Pointcuts注解来标明匹配：

```
/**
     * 切入方法
     */
    @Pointcut("call(* shixin.aopdemo.aspect_afterbefore.AfterBeforeTestHelper.aspectjTest(..))")
    public void callWith() {
    }
```

**说明：call代表方式，括号中的第一个 * 代表方法的返回值为任意，后面则是切入方法的全路径，方法括号（..），其中..代码参数任意**

**注：当然这个不是必须,可以将Pointcut注解中的value直接放入之后的切入代码，比 如@After注解中的value，不过使用@Pointcut来声明一下会更方便**

3. 在第1步类中加切入代码编写

比如咱们这里使用@After注解，该注解的方法会在切入点后执行：

```
/**
     * 使用After，那么将在切入点方法执行之后执行该切入方法代码
     *
     * @param jp 切入点信息参数（如果不需要相关信息，可以不填这个参数）
     */
    @After("callWith()")
    public void testCallAft(JoinPoint jp) {
        //打印返回值
        Log.i(TAG, "AspectJAfterBefore after,args:" + Arrays.toString(jp.getArgs()));
    }
```

**说明：那么testCallAft方法则会在切入点"callWith()"之后执行，这个callWith()即第2步的方法名称，所以就是切入到第2步指定的方法代码shixin.aopdemo.aspect_afterbefore.AfterBeforeTestHelper.aspectjTest()方法之后会执行testCallAft中的打印**

#### (三) 方法切入-Advice：
>对于方法切入方式有call和execution，咱们使用这两种方式来对方法切入，然后切入代码Advice，对应After、Before、Around等

call与execution的区别，execution是在方法内部插入，call是在方法外部插入，如下图：

![](https://github.com/nxSin/BlogPic/blob/master/Android/AOP/AOP.png?raw=true)

那么咱们看看常用的几种Advice：

##### 1. After 与 Before：

就如字面意思，就是在插入点之前和之后执行的代码。

1. 测试对象类:testAfterBeforeOne()调用aspectjTest方法

```
public void testAfterBeforeOne() {
        Log.d(TAG, "testAfterBeforeOne");
        int i = aspectjTest();
        Log.i(TAG, "testAfterBeforeOne over");
    }

    /**
     * 被调用
     */
    private int aspectjTest() {
        Log.i(TAG, "exe aspectjTest");
        return 5;
    }
```

2. 切入代码：切入点为aspectjTest

```
/**
 * afterreturn切入代码
 */
@Aspect
public class AspectJAfterBefore {
    private static final String TAG = AspectJAfterBefore.class.getSimpleName();
    /**
     * 切入方法
     */
    @Pointcut("call(* shixin.aopdemo.aspect_afterbefore.AfterBeforeTestHelper.aspectjTest(..))")
    public void callWith() {
    }

    /**
     * 使用After
     *
     * @param jp 切入点信息参数（如果不需要相关信息，可以不填这个参数）
     */
    @After("callWith()")
    public void testCallAft(JoinPoint jp) {
        //打印返回值
        Log.i(TAG, "AspectJAfterBefore after,args:" + Arrays.toString(jp.getArgs()));
    }

    /**
     * 使用before
     */
    @Before("callWith()")
    public void testCallBef() {
        Log.d(TAG, "AspectJAfterBefore before");
    }
}
```

3. 打印：

```
D/AfterBeforeTestHelper: testAfterBeforeOne
D/AspectJAfterBefore: AspectJAfterBefore before
I/AfterBeforeTestHelper: exe aspectjTest
I/AspectJAfterBefore: AspectJAfterBefore after,args:[]
I/AfterBeforeTestHelper: testAfterBeforeOne over
```

从打印即可看出，在咱们切入的aspectjTest方法前执行了切入了指定的before方法，之后切入了指定的after方法。

##### 2. Around

Around相当于是After和Before的结合，是在切入方法的前后进行的。

1. 测试对象类：

```
public void testAroundOne() {
        Log.d(TAG, "testAroundOne");
        aspectjTest();
        Log.i(TAG, "testAroundOne over");
    }
    /**
     * 被调用
     */
    private void aspectjTest() {
        Log.d(TAG, "aspectjTest");
    }
```

2. 切入点类：

```
@Aspect
public class AspectJAround {
    private static final String TAG = AspectJAround.class.getSimpleName();
    /**
     * 切入方法
     */
    @Pointcut("call(* shixin.aopdemo.aspect_around.AroundTestHelper.aspectjTest(..))")
    public void callWith() {
    }
    @Around("callWith()")
    public void testCallAround(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        Log.d(TAG, "testCallAround before");
        //采用proceed来调用方法的执行
        proceedingJoinPoint.proceed();

        //如果方法是有参数的话，可以采用调用参数方式来执行
//        Object[] args = proceedingJoinPoint.getArgs();
//        proceedingJoinPoint.proceed(args);
        Log.d(TAG, "testCallAround after");
    }
}
```

3. 打印：

```
D/AroundTestHelper: testAroundOne
D/AspectJAround: testCallAround before
D/AroundTestHelper: aspectjTest
D/AspectJAround: testCallAround after
I/AroundTestHelper: testAroundOne over
```

如代码一样，around里咱们可以通过ProceedingJoinPoint的proceed方法来让方法执行，那么切入点不调用这个方法，那方法不就不会执行了吗？哈哈，利用这个可以做一些事，比如单点登陆的权限判断。

**注：Around，对于这种确定方法的切入点是不支持带返回值的，只能返回类型为void**

##### 3. AfterReturn 

如果想在after之后，还想获取到切入方法的返回值，那么AftetReturn就可以上了,看看用法

1. 测试对象类：这里咱们从testAfterReturnOne进入，去调用aspectjTest方法，传入两个参数并得到返回值

```
/**
 * AfterReturn测试
 */
public class AfterReturnTestHelper {

    private static final String TAG = AfterReturnTestHelper.class.getSimpleName();

    public void testAfterReturnOne() {
        Log.d(TAG, "testAfterReturnOne");
        //调用带返回值方法，传入1，3参数
        int i = aspectjTest(1, 3);
        Log.i(TAG, "testAfterReturnOne over");
    }

    /**
     * 被调用
     */
    private int aspectjTest(int a, int b) {
        return 5;
    }
}
```

2. 那么看看切入核心方法：切入点为第一步的aspectjTest方法

```
/**
 * afterreturn
 */
@Aspect
public class AspectJAftwerReturn {
    private static final String TAG = AspectJAftwerReturn.class.getSimpleName();

    /**
     * 切入方法
     */
    @Pointcut("call(* shixin.aopdemo.aspect_afterreturn.AfterReturnTestHelper.aspectjTest(..))")
    public void callWith() {
    }

    /**
     * 使用AfterReturning
     * @param jp JoinPoint，可获取方法、类、参数等信息(这个参数不是AfterReturning特有，不用关心)
     * @param ret 返回值 AfterReturning特有,注意：注解中的参数名称与方法中的名称要一样，比如这里的ret
     */
    @AfterReturning(returning = "ret",  pointcut = "callWith()")
    public void testCallAft(JoinPoint jp, Object ret) {
        //打印返回值
        Log.d(TAG, "AspectJAftwerReturn after,return:" + ret);
        Log.i(TAG, "AspectJAftwerReturn after,args:" + Arrays.toString(jp.getArgs()));
    }

    @Before("callWith()")
    public void testCallBef() {
        Log.d(TAG, "AspectJAftwerReturn before");
    }
}
```

3. 打印：

```
D/AfterReturnTestHelper: testAfterReturnOne
D/AspectJAftwerReturn: AspectJAftwerReturn before
D/AspectJAftwerReturn: AspectJAftwerReturn after,return:5
I/AspectJAftwerReturn: AspectJAftwerReturn after,args:[1, 3]
I/AfterReturnTestHelper: testAfterReturnOne over
```

可以看到，通过AfterReturning中的'returning'则可以获取到方法的返回值了。

#### (四) 字段切入-Field

咱们看看字段切入，字段切入其主要是针对字段的get和set方法做切入处理

1. 测试代码：对age做赋值操作，并打印。对name做赋值，并打印

```
private static final String TAG = .class.getSimpleName();
    /**
     * 姓名，String类型
     */
    private String name;
    /**
     * 年龄 int
     */
    private int age;

    public void testStart() {
        Log.d(TAG, "testStart");
        //给age赋值，那么代表了做了set
        this.age = 5;
        Log.i(TAG, "get age value:" + age);

        this.name = "小宝贝";
        //打印name，等于获取name值，代表了get
        Log.i(TAG, "get name value:" + name);
        Log.i(TAG, "testStart over");
    }
```

2. 切入代码: 对age的赋值set做切入，切入代码将赋值减2。对name的获取get做切入，添加后缀"老大"后返回。

```
/**
 * Field切入
 */
@Aspect
public class AspectJFiled {
    private static final String TAG = AspectJFiled.class.getSimpleName();

    /**
     * 切入方法
     */
    @Pointcut("get(String shixin.aopdemo.aspect_filed.FiledHelper.name)")
    public void getMethod() {
    }

    @Pointcut("set(int shixin.aopdemo.aspect_filed.FiledHelper.age)")
    public void setMethod() {
    }

    @Around("getMethod()")
    public String testGetAround(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        Log.d(TAG, "testGetAround before");
        //采用proceed来调用方法的执行,得到返回String的字段
        String proceedStr = (String) proceedingJoinPoint.proceed();
        //修改返回字段
        String returnStr = proceedStr + "老大";
        Log.d(TAG, "testGetAround after");
        return returnStr;
    }

    @Around("setMethod()")
    public void testSetAround(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        Log.d(TAG, "testSetAround before");
        //采用getArgs方法获取设置的参数String的字段
        Integer argsAge = (Integer) proceedingJoinPoint.getArgs()[0];
        //修改设置字段,将年龄改小两岁
        int modifyAge = argsAge - 2;
        proceedingJoinPoint.proceed(new Integer[]{modifyAge});
        Log.d(TAG, "testSetAround after");
    }
}
```

3. 调用testStart()后的打印：

```
D/FiledHelper: testStart
D/AspectJFiled: testSetAround before
D/AspectJFiled: testSetAround after
I/FiledHelper: get age value:3
D/AspectJFiled: testGetAround before
D/AspectJFiled: testGetAround after
I/FiledHelper: get name value:小宝贝老大
I/FiledHelper: testStart over
```

由此可看出，虽然我们没有明写get和set方法，但是由于切入field的原因，在赋值和获取值的地方会自动切入我们设置定代码。

**注：使用Filed切入的时候需要考虑好，避免因为使用不当造成的一些奇怪现象哦**

#### (五) 异常捕获切入-Exception

咱们说说对指定异常进行代码切入，比如来看看捕获应用中的IOException，并做一些操作

1. 测试代码:

```
public void testStart() {
        Log.d(TAG, "testStart");
        try {
            testException();
        } catch (IOException e) {
            e.printStackTrace();
        }
        Log.i(TAG, "testStart over");
    }

    private void testException() throws IOException {
        //抛出异常
        throw new IOException("testStart");
    }
```

2. 切入代码：

```
@Aspect
public class AspectJException {
    private static final String TAG = AspectJException.class.getSimpleName();

    /**
     * 异常产生之后插入
     * handler方式，指定IO异常，并获取参数ex
     */
    @Before(value = "handler(java.io.IOException+) && args(ex)")
    public void testHandleBefore(IOException ex) throws Throwable {
        Log.d(TAG, "testHandleBefore before");
        Log.d(TAG, "出现io异常:" + ex);
        //打印栈信息
        ex.printStackTrace();
        //还可以统一写入文件等
    }
}
```

3. 打印:

```
D/ExceptionHelper: testStart
D/AspectJException: testHandleBefore before
D/AspectJException: 出现io异常:java.io.IOException: testStart
W/System.err: java.io.IOException: testStart
W/System.err:     at shixin.aopdemo.aspect_handle_exception.ExceptionHelper.testStart(ExceptionHelper.java:17)
W/System.err:     at shixin.aopdemo.AspectHelper.test(AspectHelper.java:27)
W/System.err:     at shixin.aopdemo.activity.MainActivity.testAspect(MainActivity.java:36)
W/System.err:     at shixin.aopdemo.activity.MainActivity.onCreate(MainActivity.java:32)
W/System.err:     at android.app.Activity.performCreate(Activity.java:6975)
W/System.err:     at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1213)
W/System.err:     at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2770)
I/ExceptionHelper: testStart over
```

由示例代码和打印可看出，我们只是指定了IO异常，就将IO异常相关给收集了，那么如果handler括号里的是*，那就将所有的都给收集了，那么通过切入对某种异常捕获，可以将应用中对应的异常代码都进行切入，做日志记录啊等等都可以了。

**注意： 这里切入代码没有用@Pointcuts注解，因为需要用到args(e)来获取注解参数信息，所以需要直接放到@Before等的value中**

#### (六) 组合-Combine

前面将到了切入点的匹配，除了单一的匹配，其还支持 ！、||、&& 这样的组合匹配。在讲这个之前先提一下：within与withincode

词 | 说明
---|---
within | 匹配定义类型在指定位置内，传入指定类型为TypePattern，即类
withincode | 匹配定义类型在指定位置内，传入类型为Signature，即方法

好了，那么我们结合withincode来讲一个带逻辑与的操作：

1. 测试类: 咱们要切入的方法为aspectJ，它被test1和test2调用

```
/**
 * Combine测试
 */
public class CombineHelper {

    private static final String TAG = CombineHelper.class.getSimpleName();

    public void testStart() {
        Log.d(TAG, "testStart");
        test1();
        test2();
        Log.i(TAG, "testStart over");
    }

    /**
     * 测试方法1，调用aspectJ()
     */
    private void test1() {
        Log.d(TAG, "test1");
        aspectJ();
    }

    /**
     * 测试方法2，调用aspectJ()
     */
    private void test2() {
        Log.d(TAG, "test2");
        aspectJ();
    }

    /**
     * 被切入的方法
     */
    private void aspectJ() {
        Log.d(TAG, "conbineNotPickedOut2");
    }
}
```

如上代码，那么正常情况下，咱们执行testStart()的时候，aspectJ()会被调用两次就会被插入执行两次，但是我们只想在test1调用aspectJ的时候插入，test2调用aspectJ方法的时候不做插入，那就需要&&组合上场了，不多话，上代码：

2. 切入代码：切入aspectJ() && 非test2()方法内调用

```
/**
 * 逻辑切入
 */
@Aspect
public class AspectJCombine {
    private static final String TAG = AspectJCombine.class.getSimpleName();

    /**
     * 切入方法aspectJ
     */
    @Pointcut("call(void shixin.aopdemo.aspect_combine.CombineHelper.aspectJ())")
    public void exeMethod() {
    }

    /**
     * !withincode  代表不是指定方法内
     */
    @Pointcut("!withincode(void shixin.aopdemo.aspect_combine.CombineHelper.test2())")
    public void notPickedOut1(){
    }

    /**
     * before 其中带 && 也就是说两个都满足才进行插入
     */
    @Before("exeMethod()&&notPickedOut1()")
    public void testExeAround() throws Throwable {
        Log.d(TAG, "testCombination before");
    }
}
```

就这样就能达到在test2方法内调用aspectJ的时候不切入代码了，对于其他逻辑 || 等等，都差不多，根据实际场景实际使用即可。

**注：在这里切入方法需要用call，不用execution，否则不起效果哦！至于为什么，想想call与execution的区别就知道了。**

#### (七) 自定义处理-Annotation

前面讲的切入方法都是固定的方法，但是实际使用中，我们一般切入的方法都是不固定的，在需要的时候才切入，那么就采用注解来做切入，咱们来做一个切入输出方法执行时间的工具

1. 定义注解:因为aspectj也是编译时注解，所以类型为CLASS即可，目标为方法

```
@Retention(RetentionPolicy.CLASS)
@Target(ElementType.METHOD)
public @interface AnnoExeTimeLogAspect {
}
```

2. 切入方法：

```
@Aspect
public class LogAspect {
    /*下面是使用注解自定义方式*/
    @Pointcut("execution(@shixin.aopdemo.logh.AnnoExeTimeLogAspect * *(..))")
    public void testExeMethodTime() {
    }

    @Around("testExeMethodTime()")
    public void timeEva(ProceedingJoinPoint joinPoint) throws Throwable {
        long timeBefore = SystemClock.currentThreadTimeMillis();
        Object[] args = joinPoint.getArgs();
        /*分带参数执行和无参数执行*/
        if (args != null && args.length > 0) {
            Log.d("timeEva", "have args:" +  Arrays.toString(args));
            joinPoint.proceed(args);
        } else {
            Log.d("timeEva", "have no args");
            joinPoint.proceed();
        }
        long timeAfter = SystemClock.currentThreadTimeMillis();
        long time = timeAfter - timeBefore;
        Signature signature = joinPoint.getSignature();
        if (!(signature instanceof MethodSignature)) {
            throw new IllegalStateException("LoginFilter 注解只能用于方法上");
        }
        MethodSignature methodSignature = (MethodSignature) signature;
        Log.i(methodSignature.getDeclaringType().getSimpleName(), joinPoint.getSignature().getName() + "，花费时间:" + time);
    }
}
```

3. 被调用的方法:包含咱们要切入的注解，被注解的方法即会被切入

```
//包含咱们的注解
@AnnoExeTimeLogAspect()
    private void testLog(int value) {
        Log.d("MainActivity", "日志:" + value);
    }
```

4. 打印:

```
D/timeEva: have args:[7]
D/MainActivity: 日志:7
I/MainActivity: testLog，花费时间:0
```

可以看到，咱们将注解放到方法上，方法被调用后，就会执行咱们切入的方法，先打印参数，再proceed执行方法，再打印话费时间。就这样一个简单打印方法执行时间的注解处理就OK了。
