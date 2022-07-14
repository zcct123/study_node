# IoC的基本概念

## **三种依赖注入**
### 构造方法注入
    构造方法注入方式比较直观，对象被构造完成后，即进入就绪状态，可以马上使用
``` java
    public FXNewsProvider(IFXNewsListener newsListner,IFXNewsPersister newsPersister) {
        this.newsListener = newsListner; this.newPersistener = newsPersister;
    }
```

### setter方法注入

setter方法注入虽不像构造方法注入那样，让对象构造完成后即可使用，但相对来说更宽松一些，
可以在对象构造完成后再注入

### 接口注入
接口注入比较死板和烦琐。如果需要注入依赖对象，被注入对象就必须声明和实现另外的接口



* 接口注入：
  从注入方式的使用上来说，接口注入是现在不甚提倡的一种方式，基本处于“退 役状态”。因为它强制被注入对象实现不必要的接口，带有侵入性。而构造方法注入和setter 方法注入则不需要如此。
* 构造方法注入：
  这种注入方式的优点就是，对象在构造完成之后，即已进入就绪状态，可以 马上使用。缺点就是，当依赖对象比较多的时候，构造方法的参数列表会比较长。而通过反 射构造对象的时候，对相同类型的参数的处理会比较困难，维护和使用上也比较麻烦。而且 在Java中，构造方法无法被继承，无法设置默认值。对于非必须的依赖处理，可能需要引入多 个构造方法，而参数数量的变动可能造成维护上的不便。
* setter方法注入：
  因为方法可以命名，所以setter方法注入在描述性上要比构造方法注入好一些。 另外，setter方法可以被继承，允许设置默认值，而且有良好的IDE支持。缺点当然就是对象无 法在构造完成后马上进入就绪状态。


 ## BeanFactory

 Spring提供了两种容器类型：BeanFactory和ApplicationContext。
* BeanFactory。基础类型IoC容器，提供完整的IoC服务支持。如果没有特殊指定，默认采用延 迟初始化策略（lazy-load）。只有当客户端对象需要访问容器中的某个受管对象的时候，才对 该受管对象进行初始化以及依赖注入操作。所以，相对来说，容器启动初期速度较快，所需 要的资源有限。对于资源有限，并且功能要求不是很严格的场景，BeanFactory是比较合适的 IoC容器选择。
* ApplicationContext。ApplicationContext在BeanFactory的基础上构建，是相对比较高 级的容器实现，除了拥有BeanFactory的所有支持，ApplicationContext还提供了其他高级特性，比如事件发布、国际化信息支持等，这些会在后面详述。*ApplicationContext所管理 的对象，在该类型容器启动之后，默认全部初始化并绑定完成。所以，相对于BeanFactory来 说，ApplicationContext要求更多的系统资源，同时，因为在启动时就完成所有初始化，容 器启动时间较之BeanFactory也会长一些。在那些系统资源充足，并且要求更多功能的场景中， ApplicationContext类型的容器是比较合适的选择 


<img src="./../spring_img/屏幕截图%202022-07-14%20124025.png" width="100%">

``` java
    BeanFactory container = ➥new XmlBeanFactory(new ClassPathResource("配置文件路径"));
    FXNewsProvider newsProvider = (FXNewsProvider)container.getBean("djNewsProvider"); 
    ewsProvider.getAndPersistNews();
    //或者如以下代码所示：
    ApplicationContext container = ➥ new ClassPathXmlApplicationContext("配置文件路径");
    FXNewsProvider newsProvider = (FXNewsProvider)container.getBean("djNewsProvider");
    ewsProvider.getAndPersistNews(); 
    //亦或如以下代码所示：
    ApplicationContext container = FileSystemXmlApplicationContext("配置文件路径");
    FXNewsProvider newsProvider = (FXNewsProvider)container.getBean("djNewsProvider"); newsProvider.getAndPersistNews();
```


BeanFactory只是一个接口，我们最终需要一个该接口的实现来进行实际Bean的管理，**DefaultListableBeanFactory**就是这么一个比较通用的BeanFactory实现类。

DefaultListableBeanFactory除了间接地实现了BeanFactory接口，还实现了**BeanDefinitionRegistry**接口，该接口才是在BeanFactory的实现中担当Bean注册管理的角色。

<img src="./../spring_img/屏幕截图%202022-07-14%20124615.png" width="100%">
* DefaultListableBeanFactory 比较通用的BeanFactory实现类。
* BeanDefinitionRegistry 担当Bean注册管理的角色
* BeanDefinition 的实例负责保存对象的所有必要信息，包括其对应的对象的class类型、是否是抽象类、构造方法参数

### 
Spring的IoC容器支持两种配置文件格式：Properties文件格式和XML文件格式。

根据不同的外 部配置文件格式，给出相应的**BeanDefinitionReader**实现类，由BeanDefinitionReader的相应实现类负责将相应的配置文件内容读取并 **映射到BeanDefinition**，然后将映射后的BeanDefinition注册到一个**BeanDefinitionRegistry**，之后，**BeanDefinitionRegistry即完成Bean的注册和加载**。 当然，**大部分工作，包括解析文件格式、装配BeanDefinition之类的工作，都是由BeanDefinitionReader的相应实现类来做的，BeanDefinitionRegistry只不过负责保管而已**

