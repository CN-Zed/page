# 如何在Spring Data Jpa下快速实现读写分离

首先，先交待下我所使用的框架，Spring Framework + Spring MVC + Spring Data Jpa (Hibrenate实现)。

另外还需用到aspectjweaver和Spring AOP。

**简述：本文通过实现Spring JDBC下AbstractRoutingDataSource，通过AOP切入Repository的各个继承接口的方法调用，根据JPA方法命名规范定义执行类型（查询/非查询），动态切换数据源。**

通过本文，可以快速实现一个**双节点**读写分离，且基本不入侵代码的实现。

## 1_实现一个DynamicDataSource

```java
package com.zedcn.configure;

import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;
import org.springframework.transaction.support.TransactionSynchronizationManager;

public class DynamicDataSource extends AbstractRoutingDataSource {
    private static final Type defaultType = Type.WRITE;

    @Override
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.hasResource("DB_TYPE") ?
                TransactionSynchronizationManager.getResource("DB_TYPE") : defaultType;
    }

    public enum Type {
        WRITE,
        READONLY
    }
}

```

`AbstractRoutingDataSource`是Spring JDBC抽象实现的一个路由数据源，其内部有一个`Map<Object,Object>`用于存放多个不同的数据源。

在其实现的方法中`determineTargetDataSource()`会返回一个数据源以供本次SQL操作使用，在该方法中，会调用`determineCurrentLookupKey()`决定Key，然后从`Map<Object,Object>`取出对应的DataSource。

**所以，我们这里只需要重写`determineCurrentLookupKey()`来决定本次操作返回哪一个Key，这样便达成了动态切换数据源。**

在上述代码中，我们写了一个枚举类用作为不同数据源的Key，之后设置数据源的时候会讲到。如果你有多个读/写数据源，则应该有对应数量的枚举类型。或者，也可以使用Prefix+序号等方式标识不同数据源，这里仅以两个节点做示例，不做太多讨论。



这里没有太多逻辑，只用到了`TransactionSynchronizationManager`线程专用资源的取出，`TransactionSynchronizationManager`后续我们再讲。为了安全，这里做了判断如果没有相应资源则返回默认类型，默认类型设置为了读写。



## 2_实现Repository的切入点

```java
package com.zedcn.aop;

import com.zedcn.configure.DynamicDataSource;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.context.annotation.EnableAspectJAutoProxy;
import org.springframework.stereotype.Component;
import org.springframework.transaction.support.TransactionSynchronizationManager;

@Aspect
@EnableAspectJAutoProxy
public class JpaHandler {
    private static final String[] READ_PREFIX = {"find", "count"};

    @Pointcut("this(org.springframework.data.repository.Repository)")
    void jpaAspect() {
    }

    @Before("jpaAspect()")
    public void beforeExecute(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().getName();
        if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
            TransactionSynchronizationManager.bindResource("DB_TYPE", DynamicDataSource.Type.READONLY);
            return;
        }
        for (String prefix : READ_PREFIX) {
            if (methodName.startsWith(prefix)) {
                TransactionSynchronizationManager.bindResource("DB_TYPE", DynamicDataSource.Type.READONLY);
                return;
            }
        }
        TransactionSynchronizationManager.bindResource("DB_TYPE", DynamicDataSource.Type.WRITE);
    }

    @After("jpaAspect()")
    public void afterExecute() {
        TransactionSynchronizationManager.unbindResource("DB_TYPE");
    }
}

```

AOP的相关知识这里不作说明，大家可以自行Google。

定义一个`String[] READ_PREFIX`用来存放JPA规范下的查询方法名称定义的前缀，这里以`findBy...`取find等举例。

通过`@Pointcut("this(org.springframework.data.repository.Repository)")`切入到所有Repository的子类（子接口）的方法中。

在获得切入点以后通过`@Before`在JPA层的方法调用前做预处理，大致思路很简单：

1. 预先判断本次执行的方法是否在代码层面指定了`@Transactions(readonly=true)`，如果设为只读则直接在此处判定为READ_ONLY的数据源。
2. 通过`JoinPoint`实例获取当前切入点执行的方法名。
3. 遍历所有Prefix，如果存在任意一个方法名`startWith()`的匹配，则算作READ_ONLY数据源。
4. 通过`TransactionSynchronizationManager`存入此次的判断结果。即DataSource的Key。

`TransactionSynchronizationManager`是一个当前线程事物的管理器。他的内部包装了各种类型的`ThreadLocal`用来存放线程专用资源（或者表述为线程间不共享的资源）。通过这个管理器，可以保证在线程A下操作的数据，只有线程A能够使用。这样便不会出现并发时，线程A判定的数据源状态不会被其他线程覆盖或者被其他线程使用。

这里我建议在`@After`里取消之前所绑定的资源。暂时没有做相应的对比，是否存在内存溢出等问题。

## 3_在DatabaseConfig里设置数据源

这里我们仅举例Spring的Java Base Config，XML根据对应语法设置即可。

```java
    @Bean
    public DataSource dataSourceApp() {
        DynamicDataSource dynamicDataSource = new DynamicDataSource();
        dynamicDataSource.setTargetDataSources(new HashMap<Object, Object>() {{
            put(DynamicDataSource.Type.WRITE, masterDataSourceApp());
            put(DynamicDataSource.Type.READONLY, slaveDataSourceApp());
        }});
        return dynamicDataSource;
    }
```

这里只需要做一个`DynamicDataSource`的实例化，然后通过`setTargetDataSources(Map<Object,Object>)`方法将所有Type的数据源都设置进去，`Map<Object,Object>`即上文提到的，Key和数据源的对应。这里的`masterDataSourceApp()`和`slaveDataSourceApp()`代表两个不同的数据源，因为每个人用的库可能不一样，所以这里就不在贴具体的代码。

## 4_总结

至此，基于Spring Data JPA的读写分离已经实现，因为使用了AOP切入到所有继承自Repository的子类（子接口），所以无需使用其他的方式（如：事务设置为只读、注解），这样不会入侵到代码里，也就不存在现有代码要覆盖一遍。个人认为还是很好的。