---
layout: post
title:  spring中文说明文档有感
categories: [java]
tags: [java,spring,框架]
---



# Spring概览



## spring的设计理念

> spring框架的指导原则

- 在每个层面上提供选择。Spring让你尽可能晚的推迟设计决策。例如：你可以通过配置来切换持久化供应商，而不需要改变你的代码。对于许多其他基础设施问题和与第三方API的集成也是如此。
- 适应不同的观点。Spring拥抱灵活性，对事情应该如何做不持意见。它支持具有不同视角的广泛的应用需求。
- 保持强大的后向兼容性。Spring的演进是经过精心管理的，在不同的版本之间几乎不存在破坏性的变化。Spring支持一系列精心选择的JDK版本和第三方库，以方便维护依赖Spring的应用程序和库。
- 关心API的设计。Spring团队花了很多心思和时间来制作直观的API，并且在很多版本和很多年中都能保持良好的效果。
- 为代码质量设定高标准。Spring框架非常强调有意义的、最新的和准确的javadoc。它是为数不多的可以宣称代码结构干净、包与包之间没有循环依赖关系的项目之一。



# 核心技术



> IOC、AOP、AOT



# IOC容器



## 简介

IOC也被称为依赖注入（DI）。它是一个过程，对象仅通过构造参数、工厂方法的参数或在对象实例被构造或从工厂方法返回后在其上设置的属性来定义其依赖关系（即它们与之合作的其他对象）。然后容器在创建bean时注入这些依赖关系。这个过程从根本上说是Bean本身通过使用直接构建类或诸如服务定位模式的机制来控制其依赖关系的实例化或位置的逆过程（因此被称为控制翻转）。



## IOC容器概述



org.springframework.context.ApplicationContext接口代表SpringIoC容器，负责实例化、配置和组装Bean。容器通过读取配置元素数据来获取关于要实例化、配置和组装哪些对象的指示。配置元素以XML、Java注解或Java代码表示。









## 使用容器

ApplicationContext是一个高级工厂的接口，能够维护不同bean及其依赖关系的注册表。通过使用方法 T getBean（Stringname,Class<T> requiredType)，可以检索到bean的实例。



``` java
// 创建和配置bean
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");

// 检索配置的实例
PetStoreService service = context.getBean("petStore", PetStoreService.class);

// 使用配置的实例
List<String> userList = service.getUsernameList();

```



## Bean的概览



一个Spring Ioc容器管理着一个或多个Bean。这些Bean是用开发着提供给容器的配置元数据创建的。

在容器本身中，这些Bean定义被表示为BeanDefinition对象，它包含一下元数据。

- 一个Bean全路径类名：通常，被定义的Bean的实际实现类。
- Bean的行为配置元素，它说明了Bean在容器中的行为方式（Scope、生命周期回调，等等）。
- 对其他Bean的引用，这些Bean需要做它的工作。这些引用也被称为合作者或者依赖。
- 要在新创建的对象中设置的其他配置设置--例如，pool的大小限制或在管理连接池的Bean中使用的连接数。



### 属性



| 属性                     | 解释         |
| ------------------------ | ------------ |
| Class                    | 实例化Bean   |
| Name                     | Bean命名     |
| Scope                    | Bean Scope   |
| Constructor arguments    | 依赖注入     |
| Properties               | 依赖注入     |
| Autowiring mode          | 注入协作者   |
| Lazy initialization mode | 懒加载的Bean |
| Initialization method    | 初始化回调   |
| Destruction method       | 销毁回调     |



> Bean元数据和手动提供的单体实例需要尽早注册，以便容器在自动注入和其他内省步骤中正确推导它们。虽然在某种程度上支持覆盖现有的元数据和现有的单体实例，但官方不推荐在运行时注册新的Bean（与对工厂的实时访问同时进行），这可能会导致并发访问异常、bean容器中的不一致状态，或者两者都有。



## 依赖注入

依赖注入（DI）是一个过程，对象仅通过构造参数、工厂方法的参数或在对象实例被构造或从工厂方法返回后在其上设置的属性来定义他们的依赖（即与它们一起工作的其他对象）。然后，容器在创建bean时注入这些依赖。这个过程从根本上说是Bean本身通过使用类的直接构造或服务定位模式来控制其依赖的实例化或位置的逆过程（因此被称为控制反转）。



### 基于构成器的依赖注入

通过容器调用带有许多参数的构成函数来完成的，每个参数代表一个依赖。调用带有特定参数的static工厂方法来构造bean几乎是等价的。


``` java
public class SimpleMovieLister {

    // the SimpleMovieLister has a dependency on a MovieFinder
    private final MovieFinder movieFinder;

    // a constructor so that the Spring container can inject a MovieFinder
    public SimpleMovieLister(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // business logic that actually uses the injected MovieFinder is omitted...
}
```



### 基于Setter的依赖注入

通过容器在调用无参构造或无参数的static工厂方法来实例化你的bean之后调用Setter方法来实现的。



``` java
public class SimpleMovieLister {

    // the SimpleMovieLister has a dependency on the MovieFinder
    private MovieFinder movieFinder;

    // a setter method so that the Spring container can inject a MovieFinder
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // business logic that actually uses the injected MovieFinder is omitted...
}

```



- 应该选用哪种DI方式？

由于可以混合使用基于构造函数和基于Setter的DI，一个好的经验法则是对强制依赖使用构造函数，对可选依赖使用setter方法或配置方法。请注意，在setter方法上使用@Autowired注解可以使属性成为必须得依赖；然而，带有参数程序化验证的构造注入是更好的。

Spring团体通常提倡构造函数注入，因为它可以让你将应用组件实现为不可变的对象，并确保所需的依赖不为null。此外，构造函数注入的组件总是以完全初始化的状态返回给客户端（调用）代码。顺便提一下，大量的构造函数参数是一种不好的代码，意味着该类可能有太多的责任，应该重构以更好地解决适当的分离问题。

Settter注入主要应该只用于在类中可以分配合理默认值的可选依赖。否则，必须在代码使用依赖的所有地方进行非null值检查。Setter注入的好处：Setter方法使该类的对象可以在以后重新配置或重新注入。

### 依赖的解析过程

Bean解析的过程

1. ApplicationContext是用来描述所有bean的配置元数据创建和初始化的。配置元数据可以由XML、Java代码或注解来指定。
2. 对于每个Bean来说，它的依赖是以属性、构造函数参数或静态工厂方法的参数（如果你用它代替正常的构造函数）的形式表达。在实际创建Bean时，这些依赖被提供给Bean。
3. 每个属性或构造函数参数都是要设置的值的实际定义，或对容器中另一Bean的引用。
4. 每个作为值的属性或构造函数参数都会从其指定格式转换为该属性或构造函数参数的实际类型。默认情况下，Spring可以将以字符串格式提供的值转换为所有内置类型，如int、long、String、boolean等。



- 循环依赖

> 如果你使用主要的构造函数注入，就有可能产生一个无法解决的循环依赖情况。
>
> 比如说，类A通过构造函数注入需要类B的一个实例，而类B通过构造函数注入需要类A的实例。如果你将类A和类B的Bean配置为相互注入，Spring IOC容器会在运行时检测到这些循环引用，并抛出一个BeanCurrentlyInCreationException。
>
> 解决方法：在编写类源码时，使其通过Setter而不是构造器进行配置。或者，避免构造器注入，只使用setter注入。
>
> 还有一种方法是在一个Bean被完全初始化之前就被注入到另一个Bean中。





## 注入协作者（Autowiring Collaborators）自动注入

Spring容器可以自动连接协作Bean之间的关系。你可以让Spring通过检查ApplicationContext的内容为你的Bean自动解决协作者（其他Bean）。



- 自动注入的优点

自动注入可以随着你的对象的发展而更新配置。例如，如果你需要给一个类添加一个依赖，这个依赖可以自动满足，而不需要你修改配置。因此，自动在开发过程中可能特别有用。而不会否定在代码库变得更加稳定时切换到显式注入的选择。

- 四种自动注入的方式

| 模式        | 解释                                                         |
| ----------- | ------------------------------------------------------------ |
| no          | （默认）没有自动注入。Bean引用必须由ref元素来定义。对于大型部署来说，不建议改变默认设置，因为明确指定协作者会带来更大的控制力和清晰度。在某种程度上，它记录了一个系统的结构。 |
| byName      | 通过属性名称自动注入。Spring寻找一个与需要自动注入的属性同名的Bean。例如，如果一个Bean定义被设置为按名称自动注入，并且包含一个master属性（也就是说，它有一个setMaster（..）方法），Spring会寻找一个名为master的Bean定义并使用它来设置该属性。 |
| byType      | 如果容器中正好有一个property类型的Bean存在，就可以自动注入该属性。如果存在一个以上的bean，就会抛出一个致命的exception，这表明你不能对该bean使用byType自动注入。如果没有匹配的bean，就不会发生任何事情（该属性没有被设置）。 |
| constructor | 类似于byType，但适用于构造函数参数。如果容器中没有一个构造函数参数类型的Bean，就会产生一个致命的错误。 |



### 自动注入的限制和缺点

- property和constructor-arg设置中的明确依赖关系总是覆盖自动注入。你不能自动注入简单的属性，如基本数据、String和Class。
- 自动注入不如显式注入精确。尽管正如前面的表格中所指出的，Spring很小心地避免在模糊不清的情况进行猜测，这可能会产生意想不到的结果。你的Spring管理的对象之间的关系不再被明确地记录下来
- 对于可能从Spring容器中生成文档的工具来说，注入信息可能无法使用。
- 容器中的多个Bean定义可以与setter方法或构造参数指定的类型相匹配，以实现自动注入。对于数组、集合或Map实例，这不一定是个问题。然而，对于期待单一值的依赖关系，这种模糊性不会被任意地解决。如果没有唯一的Bean的定义，就会抛出一个异常。

无法根据类型指定唯一bean的几种解决方案：

1. 放弃自动注入，改用明确注入。
2. 通过将Bean定义的Autowire-condidate属性设置为false来避免bean定义的自动注入。
3. 通过将<bean/> 元素的primary属性设置为true，将单个bean定义指定为主要候选者。
4. 实现基于注解的配置所提供的更精细的控制。





### 通过方法来注入Bean

> 大多数情况下，Bean都是单例的，当一个单例Bean需要另一个单例Bean协作或一个非单例的Bean需与另一个非单例Bean协作，需要将一个Bean定义为另一个Bean的属性。如果需要Bean，A每次使用Bean，B时，都会重新创建Bean，B，则需要通过方法注入的方式解决。

放弃一些控制的反转。可以通过实现ApplicationContextAware接口给bean A注入容器，并在bean A需要时让容器的getBean("B")调用询问（一个典型的new）bean B实例。



``` java
package com.springboot.demo;


import org.springframework.beans.BeansException;
import org.springframework.beans.PropertyValue;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;

import java.util.Map;

class  Command{
    private Map map;

    public void setMap(Map map) {
        this.map = map;
    }
    public int execute(){
        return 1;
    }
}

public class CommandManager implements ApplicationContextAware {
    private ApplicationContext applicationContext;
    public Object process(Map commandState){
        Command command = createCommand();
        command.setMap(commandState);
        return command.execute();
    }

    protected Command createCommand(){
//        每次调用时，用getBean方法，通过BeanFactory来创建新的Bean。
        return this.applicationContext.getBean("command",Command.class);
    }

//获取ApplicationContext
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext=applicationContext;
    }
}

```



### 查找方法依赖注入

查找方法注入是指容器能够覆盖容器管理的Bean上的方法并返回容器中另一个命名的bean的查询结果。（动态代理）Spring框架通过使用CGLIB的字节码生成来实现这种方法注入，动态地生成一个覆盖该方法的子类。



1. 为了实现此动态子类功能，被Spring bean容器子类化的类不能为final。
2. 对一个包含abstract方法的类进行单元测试，需要你自己对这个类进行子类化，并提供一个abstract方法的stub实现。
3. 有方法体的方法对于组件扫描也是必要的，这需要具体的方法来实现。
4. 另一个关键的限制是，查找方法对工厂方法不起作用，特别是对配置类中的@Bean方法不起作用，因为这种情况下，容器不负责创建实例，因此不能即时创建运行时生成的子类。



## Bean Scope

> 当你创建一个Bean定义时，你创建了一个“构造方法”，用于创建该Bean定义（definition）是所定义的类的实际实例。Bean定义（definitiion）是一个“构造方法”的想法很重要，因为它意味着，就像一个类一样，你可以从一个“构造方法”中创建许多实例。



Spring框架支持六个scope

| scope       | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| singleton   | （默认情况下）为每个Spring IOC容器将单个Bean定义的Scope扩大到单个对象实例。 |
| prototype   | 将单个Bean定义的Scope扩大到任何数量的对象实例。              |
| request     | 将单个Bean定义的Scope扩大到单个HTTP请求的生命周期。也就是说，每个HTTP请求都有自己的Bean实例，该实例是在单个Bean定义的基础上创建的。只在Web感知的Spring ApplicationContext的上下文中有效。 |
| session     | 将单个Bean定义的Scope扩大到一个HTTP Session的生命周期。只在Web感知的Spring ApplicationContext的上下文中有效 |
| Application | 将单个Bean定义的Scope扩大到ServletContext的生命周期中。只在Web感知的Spring ApplicationContext的上下文中有效。 |
| websocket   | 将单个Bean定义的Scope扩大到WebSocket的生命周期。仅在具有Web感知的Spring ApplicationContext的上下文中有效。 |



### 单例

只有一个单例的Bean的共享实例被管理，所有对符合该bean定义的id的bean的请求都会被Spring容器返回该特定的Bean实例。

> 当你定义了一个Bean的定义，并且它被定义为singleton（单例），Spring Ioc容器就会为该Bean定义的对象创建一个确切的实例。这个单例的实例被存储在单例Bean的缓存中，所有后续的请求和对该命名Bean的引用都会返回缓存中的对象。



![2024-4-2414:31:42-1713940302216.png](https://gitee.com/grsswh/drawing-bed/raw/master/image/2024-4-2414:31:42-1713940302216.png)

Spring的单例Bean概念和GOF设计模式书中定义的单例模式不同。Gof单例模式对对象的范围进行了硬编码，即每个ClassLoader创建一个且仅有一个特定类的实例。Spring单例的范围最好被描述为每个容器和每个bean。这意味着，如果你在一个Spring容器中为一个特定的类定义了一个Bean，Spring容器就会为该Bean定义的类创建一个且只有一个实例。Singleton Scope是Spring默认的Scope。



### 原型

每次对该特定Bean的请求都会创建一个新的Bean实例。也就是说，该Bean被注入到另一个bean中，或者你通过容器上的getBean（）方法调用来请求它。

> 应该对所有有状态的Bean使用原型范围，对无状态的bean使用单例范围。

![2024-4-2414:38:30-1713940709879.png](https://gitee.com/grsswh/drawing-bed/raw/master/image/2024-4-2414:38:30-1713940709879.png)



与其他作用域相比，Spring不会管理原型模式的Bean的生命周期。容器对原型对象进行实例化，配合和其他方面的组装，并将其交给客户端，而对该原型实例没有进一步的记录。后续销毁需要客户端自己来。



> 一个单例bean中，如果有依赖一个原型的Bean，那么这个原型Bean也只会被创建一次。



## 自定义Bean的性质



### 生命周期回调

为了与容器对Bean生命周期的管理进行交互，你可以实现Spring InitializingBean和DispobleBean接口。容器为前者调用afterPropertiesSet()，为后者调用destroy（），让Bean在初始化和销毁你的Bean时执行某些动作。

> @PostConstruct和@PreDestroy注解通常被认为是在现代Spring应用程序中接收生命周期回调的最佳实践。使用这些注解意味着你的Bean不会被耦合到Spring特定的接口。



- 在非Web应用中优雅地关闭Spring IoC容器

如果你在非Web应用环境中使用Spring的Ioc容器，请向JVM注册一个shutdown hook。这样做可以确保优雅地关闭，并在你的单例Bean上调用相关的destroy方法，从而释放所有资源。



> 要注册一个shutdown hook，请调用registerShutdown（）方法，该方法在ConfigurableApplicationContext接口上声明。

``` java
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public final class Boot {

    public static void main(final String[] args) throws Exception {
        ConfigurableApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");

        // add a shutdown hook for the above context...
        ctx.registerShutdownHook();

        // app runs here...

        // main method exits, hook is called prior to the app shutting down...
    }
}


```



org.springframework.beans 和 org.springframework.context 包是Spring Framework的IoC容器的基础。

BeanFactory接口提供了一种高级配置机制，能够管理任何类型的对象。ApplicationContext是BeanFactory的一个子接口。它增加了：

- 更容易和Spring的AOP功能集成
- Message resource处理（用于国际化）
- 事件发布
- 应用层的特定上下文，如WebApplicationContext，用于web应用



> 在Spring中，构成应用程序的骨干并由Spring IoC容器管理的对象被称为Bean。Bean是一个由Spring IOC容器实例化、组装和管理的对象。Bean以及它们之间的依赖关系都反应在容器使用的配置元数据中。



### Bean定义（Definition）的继承

一个Bean定义可以包含很多配置信息，包括构造函数参数、数据值和容器特有的信息，如初始化方法、静态工厂方法名称等等。一个子Bean定义从父定义继承配置数据。自定义可以覆盖一些值或根据需要添加其他值。使用父Bean定义和子Bean定义可以节省大量的打字工作量。

> ApplicationContext默认设置了所有的Singleton。因此，如果你有一个（父）Bean定义，你打算作为模版使用，并且这个定义指定了一个类，你必须确保将abstract属性设置为true，否则应用上下文将试图预实例化abstract Bean。



### 容器的扩展



#### 使用BeanPostProcessor自定义bean

> BeanPostProcessor接口定义了回调方法，可以实现这些方法来提供你自己的实例化逻辑、依赖性解析逻辑等。

BeanPostProcessor实例对Bean（或对象）实例进行操作。也就是说，Spring Ioc容器实例化一个Bean实例，然后由BeanPostProcessor实例来完成其工作。

BeanPostProcessor实例是按容器范围的。只有在你使用容器结构时才有意义。如果你在一个容器中定义了一个BeanPostProcessor，他只对该容器的bean进行后置处理。换句话说，在一个容器中定义的BeanPostProcessor不会对另一个容器中定义的BeanPostProcessor进行后置处理，即使两个容器都是同一层次结构的一部分。


``` java
package org.springframework.beans.factory.config;

import org.springframework.beans.BeansException;
import org.springframework.lang.Nullable;

public interface BeanPostProcessor {
    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}

```
BeanPostProcessor接口正常由两个回调方法组成。当这样的类被注册为容器的后置处理器时，对于容器创建的每个bean实例，后置处理器在容器初始化方法被调用之前和任何bean初始化回调之后都会从容器获得一个回调。后置处理程序可以对Bean实例采取任何行动，包括完全忽略回调。bean类后置处理器通常会检查回调接口，或者用代理来包装bean类。一些Spring AOP基础设施类被实现为Bean后置处理器，以提供代理封装逻辑。

ApplicationContext会自动检测在配置元数据中定义实现BeanPostProcessor 接口的任何Bean。ApplicationContext将这些bean注册为后置处理器，以便以后在bean创建时可以回调它们。Bean后置处理器可以像其他Bean一样被部署在容器中。

> 以编程方式注册BeanPostProcessor实例
>
> 虽然推荐的BeanPostProcessor 虽然推荐的 BeanPostProcessor 注册方法是通过 ApplicationContext 自动检测（如前所述），但你可以通过使用 addBeanPostProcessor 方法，针对 ConfigurableBeanFactory 以编程方式注册它们。
> 以编程方式添加的BeanPostProcessor实例并不遵循Order接口的顺序来执行。 

- BeanPostProcessor实例和AOP自动代理

实现了BeanPostProcessor接口的类是特殊的，会被容器区别对待。所有BeanPostProcessor实例和他们直接引用的Bean在启动时被实例化，作为ApplicationContext特殊启动阶段的一部分。接下来，所有BeanPostProcessor实例被分类注册，并应用于容器中的所有其他Bean。因为AOP的自动代理是作为BeanPostProcessor本身实现的，所以BeanPostProcessor实例和它们直接引用的Bean都不符合自动代理的条件，因此，没有切面被织入进去。    

#### 用BeanFactoryPostProcessor定制配置元数据

BeanFactoryPostProcessor对Bean配置元数据进行操作。Spring IoC容器让BeanFactoryPostProcessor读取配置元数据，并在容器实例化BeanFactoryPostProcessor实例以外的任何Bean之前对其进行潜在的修改。

``` java
package org.springframework.beans.factory.config;

import org.springframework.beans.BeansException;

@FunctionalInterface
public interface BeanFactoryPostProcessor {
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}


```


> 如果你想改变实际的Bean实例（即从配置元数据中创建的对象），那么你需要使用 BeanPostProcessor（如前面 使用 BeanPostProcessor 自定义 Bean 中的描述）。虽然在技术上可以在BeanFactoryPostProcessor中处理Bean实例（例如，通过使用BeanFactory.getBean())，但这样做会导致过早的Bean实例化，违反了标准容器的生命周期。这可能会导致负面的副作用，比如绕过Bean的后置处理器。


## 基于注解的容器配置

### 使用@Autowired

> @Inject注解可以代替Spring的@Autowired注解

从Springframework 4.3开始，如果目标Bean一开始就只定义了一个构造函数，那么在这样的构造函数上就不再需要@Autowired注解。然而，如果有几个构造函数，而且没有主要/默认构造函数，那么至少有一个构造函数必须用@Autowired注解，以便指示容器使用哪一个。
