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



org.springframework.beans 和 org.springframework.context 包是Spring Framework的IoC容器的基础。

BeanFactory接口提供了一种高级配置机制，能够管理任何类型的对象。ApplicationContext是BeanFactory的一个子接口。它增加了：

- 更容易和Spring的AOP功能集成
- Message resource处理（用于国际化）
- 事件发布
- 应用层的特定上下文，如WebApplicationContext，用于web应用



> 在Spring中，构成应用程序的骨干并由Spring IoC容器管理的对象被称为Bean。Bean是一个由Spring IOC容器实例化、组装和管理的对象。Bean以及它们之间的依赖关系都反应在容器使用的配置元数据中。



## IOC容器概述



org.springframework.context.ApplicationContext接口代表SpringIoC容器，负责实例化、配置和组装Bean。容器通过读取配置元素数据来获取关于要实例化、配置和组装哪些对象的指示。配置元素以XML、Java注解或Java代码表示。



- 基于注解的配置

