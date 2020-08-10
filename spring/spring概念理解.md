1.依赖查找与依赖注入

依赖查找：自己依赖什么，自己通过显示调用框架API获取到，然后进行相应的业务操作。

依赖注入：自己依赖什么，不用关心依赖对象的构造和初始化，只需要关心依赖对象在业务中的操作就行。

举例：

依赖查找

定义一个worker.xml和Worker对象

```xml
<bean id="worker" class="com.github.rookie45.spring.ioc.lookup.domain.Worker">
```

```java
public class Worker {
    public void doWork(){
        System.out.println("rookie45 want to sleep...");
    }
}
```

业务代码中通过框架API获取Worker对象，然后进行相关业务操作

```java
    BeanFactory beanFactory =
            new ClassPathXmlApplicationContext("classpath:worker.xml");
    Worker bean = (Worker) beanFactory.getBean("worker");
    bean.doWork();
    System.out.println("no sleep, continue do work ...");
```
依赖注入

定一个business.xml和Business对象

```xml
<import resource="worker.xml"/>
<bean id="business" class="com.github.rookie45.spring.ioc.lookup.domain.Business">
    <property name="name" ref="worker"/>
</bean>
```

```java
public class Business {
    private Worker worker;
    public void setWorker(Worker worker){
        this.worker = worker;
    }
    public void doWork(){
        worker.doWork();
        System.out.println("no sleep, continue do work ...");
    }
}
```

业务代码中不用关心依赖对象的构造和初始化，交由框架完成，只需在自己的业务代码中进行相关操作即可。

两者对比：

| 名称     | 依赖处理 | 易编码性 | 代码侵入性 | API依赖 | 可读性 |
| -------- | -------- | -------- | ---------- | ------- | ------ |
| 依赖查找 | 主动获取 | 繁琐     | 侵入业务   | 高依赖  | 良好   |
| 依赖注入 | 被动提供 | 便利     | 低侵入性   | 低依赖  | 一般   |

