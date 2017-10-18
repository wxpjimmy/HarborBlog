title: Google guice学习(1)
date: 2016-08-21 20:43:07
categories: java
tags: guice,google
---

最近在研究DRUID，druid整个设计采用了Google的guice依赖注入框架，所以要想学习druid源代码，首先要弄明白guice框架是怎么回事。

## 什么是GUICE   
Guice是Google开发的一个轻量级，基于Java5（主要运用泛型与注释特性）的依赖注入框架(IOC)。Guice非常小而且快。Guice是类型安全的，它能够对构造函数，属性，方法（包含任意个参数的任意方法，而不仅仅是setter方法）进行注入。

guice还具有一些可选的特性比如：自定义scopes，传递依赖，静态属性注入，与Spring集成和AOP联盟方法注入等。一部分人认为，Guice可以完全替代spring, 因为对于DI组件框架来说, 性能是很重要的, guice比spring快十倍左右, 另外, 也是最重要的一点, 使用spring很容易写成service locator的风格, 而用guice, 你会很自然的形成DI风格.甚至说，guice简单超轻量级的DI框架效率是spring的1.6倍，Spring使用XML使用将类与类之间的关系隔离到xml中，由容器负责注入被调用的对象，而guice将类与类之间的关系隔离到Module中，声明何处需要注入，由容器根据Module里的描述，注入被调用的对象,使用Annotation使用支持自定义Annotation标注，对于相同的接口定义的对象引用，为它们标注上不同的自定义Annotation注释，就可以达到同一个类里边的同一个接口的引用，注射给不同的实现，在Module里用标注做区分，灵活性大大增加


## guice跟spring对比  
- 使用XML  
 Spring 使用将类与类之间的关系隔离到xml中，由容器负责注入被调用的对象，因此叫做依赖注入。
 Guice 不使用xml,将类与类之间的关系隔离到Module中，声名何处需要注入，由容器根据Module里的描述，注入被调用的对象。  
- 使用Annotation  
 Guice 使用，支持自定义Annotation标注  
- 运行效率  
 Spring 装载spring配置文件时，需解析xml，效率低，getBean效率也不高，不过使用环境不会涉及到getBean，只有生产环境的时候会用到getBean,在装载spring应用程序的时候，已经完成全部的注射，所以这个低效率的问题不是问题。
 Guice 使用Annotation，cglib, 效率高与spring最明显的一个区别，spring是在装载spring配置文件的时候把该注入的地方都注入完，而Guice呢，则是在使用的时候去注射，运行效率和灵活性高。  
- 类耦合度  
 Spring 耦合度低，强调类非侵入，以外部化的方式处理依赖关系，类里边是很干净的，在配置文件里做文章，对类的依赖性极低。
 Guice 高，代码级的标注，DI标记@inject侵入代码中，耦合到了类层面上来，何止侵入，简直侵略，代码耦合了过多guice的东西，大大背离了依赖注入的初衷，对于代码的可维护性，可读性均不利  
- 类编写时  
 Spring 需要编写xml，配置Bean，配置注入。
 Guice 只需声明为@inject,等着被注入，最后在统一的Module里声明注入方式。  
- 仅支持IoC  
 Spring 否，spring目前已经涉猎很多部分。
 Guice 是，目前仅仅是个DI容器。  
- 是否易于代码重构  
 Spring 统一的xml配置入口，更改容易。
 Guice 配置工作是在Module里进行，和spring异曲同功  
- 支持多种注入方式  
 Spring 构造器，setter方法。
 Guice Field,构造器，setter方法.  
- 灵活性  
 Guice 1.如果同一个接口定义的引用需要注入不同的实现，就要编写不同的Module，烦琐 2.动态注入，如果你想注射的一个实现，你还未知呢，怎么办呢，spring是没办法，事先在配置文件里写死的，而Guice就可以做到，就是说我想注射的这个对象我还不知道注射给谁呢，是在运行时才能得到的的这个接口的实现，所以这就大大提高了依赖注射的灵活性，动态注射。
与现有框架集成度
 Spring 1.高，众多现有优秀的框架（如struts1.x等）均提供了spring的集成入口，而且spring已经不仅仅是依赖注入，包括众多方面。2. Spring也提供了对Hibernate等的集成，可大大简化开发难度。3.提供对于orm,rmi,webservice等等接口众多，体系庞大。
 Guice 1.可以与现有框架集成，不过仅仅依靠一个效率稍高的DI，就想取代spring的地位，有点难度。  
- 配置复杂度  
 Spring 在xml中定位类与类之间的关系,难度低。
 Guice 代码级定位类与类之间的关系,难度稍高。

