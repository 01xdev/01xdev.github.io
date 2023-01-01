---
layout: post
title:  "Managing Dependencies - The BeanFactory"
tags: ['DEPENDENCY INJECTION']    
---

Dependency Injection(DI) Frameworks maintain a reductionistic outlook towards software systems - if components of a system can be considered as dependencies, then a sophisticated system can be put together by interconnecting these dependencies.Hence every DI framework should provide provisions for:
* Managing dependencies
* Injecting one dependency on to another

The spring framework uses the generic term *bean* for the components it manage. In spring verbose, the duties of a DI framework can be rephrased as:
* Registering beans
* Wiring beans together

Since most of my exposure to DI comes from spring, I will be using spring verbose throughout the article.

-------
# Managing Beans

Consider the following example 

```java
public class UserService {
    // Dependency
    private UserRepository userRepository;
    // Setter for injection
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    public User getUserDetails(Long id){
        return userRepository.findById(id);
    }
}
```

The `getUserDetails` method depends on `userRepositiory` to fetch details of a user from userId. The `UserService` class provides a setter method, `setUserRepository` to provision for the injection of `userRepository` into `UserService`.

If the system uses a DI framework, then the instances of `UserService` and `UserRepository` would have been maintained by the framework. As discussed in the previous article, the *Simple Factory Pattern* is a good candidate for a library that can manage instances for a system. But for the Simple Factory pattern to function, we need an abstraction to represent all the instances managed by the factory. 

Consider the following class:

```java
public class Bean {
    public String className;
    public Object instance;

    Bean(Object instance){
        this.className = instance.getClass().getName();
        this.instance = instance;
    }
}
```
The `Bean` class is a simple data structure with following fields
* **instance** - stores an instance of a dependency/component 
* **className** - the class whose instance is stored in the `instance` field

Given its simple structure, we can consider `Bean` as an abstraction for the dependencies(components) in a system. 

Now we can construct a *Simple Factory* as:
```java
public interface BeanFactory {
    public void registerBean(String name,Object instance);
    public Bean getBean(String beanName);
}
```

We can give an object for management as follows:
```java
beanFactory.registerBean("UserService",new UserService());
beanFactory.registerBean("UserRepository", new UserRepository());
```
This should create `Bean`s against `UserService` and `UserRepository`. The beans should be retrieved by mentioning the name passed while registering.

```java
UserService userService = (UserService)beanFactory.getBean("UserService").instance;
UserRepository userRepository = (UserRepository) beanFactory.getBean("UserRepository").instance;
```

For storing and retreiving beans against a name, we can use a `HashMap<String,Bean>`.

The following implementation of `BeanFactory` uses hashmap to store beans

```java

public class SimpleBeanFactory implements BeanFactory{
    private Map<String,Bean> beanContainer;
    private static volatile SimpleBeanFactory beanFactory;

    SimpleBeanFactory(){
        this.beanContainer = new HashMap<>();
    }

    @Override
    public void registerBean(String name,Object instance){
        Bean bean = new Bean(instance);
        beanContainer.put(name, bean);
    }

    @Override
    public Bean getBean(String beanName) {
       Bean bean = Optional.ofNullable(beanContainer.get(beanName))
               .orElseThrow(()->new RuntimeException("Bean not found"));
       return bean;
    }
}

```
The `SimpleBeanFactory` uses a map, `beanContainer` to store beans.The `registerBean` and `getBean` implementations are pretty straight forward.`registerBean` creates a new `Bean` for the object provided and would add it to the map.`getBean` would retrieve the bean from the map.

Consider the following `configure` method. This method initializes a new `SimpleBeanFactory`, registers beans, and *wire* them together.
```java
public void configure(){
    beanFactory = new SimpleBeanFactory();
       
    // Registering beans
    beanFactory.registerBean("UserService",new UserService());
    beanFactory.registerBean("UserRepository", new UserRepository());
    
    
    // Wiring beans
    // Obtaining userRepository and userService from bean factory
    UserRepository userRepository = (UserRepository) beanFactory.getBean("UserRepository").instance;
    UserService userService = (UserService)beanFactory.getBean("UserService").instance;
    // Using setter injection to add 
    userService.setUserRepository(userRepository);
}
```
-----

The code seems a bit wordy. We will have to call  `registerBean` for all the components in our system. Could this be moved out to a configuration file,  perhaps an XML file like so:

```xml
<?xml version="1.0" ?>
<Beans>
    <Bean name="UserService" classname="in.onexdev.testScenarios.UserService" />
    <Bean name="UserRepository" classname="in.onexdev.testScenarios.UserRepository" />
</Beans>
```
All the beans in the system are listed within the `<Beans>` tag. Each bean in the system is represented by the `<Bean>` tag. 

Every `<Bean>` in the XML has two attributes - `name` and `classname`. `name` can be used as the unique identifier of the bean. `classname` can be used to create corresponding `Bean` instance.

We can use XML DOM parser to parse the document
```java
Document document = documentBuilder.parse(beanXmlConfigPath);
```

Once the XML Document is parsed, we can obtain all Nodes with the `Bean` tag 
```java
NodeList beans = document.getElementsByTagName("Bean");
```

Now that we have all the beans as XML nodes, we can use the attributes in each bean node to create corresponding `Bean` instances.

```java
private void registerBeansFromNodeList(NodeList beans) {
    for(int i = 0; i< beans.getLength(); i++){
        Node bean = beans.item(i);
        // Obtaining attributes of each Bean
        NamedNodeMap attributes = bean.getAttributes();

        // Creating instance from classname attribute 
        String classname = attributes.getNamedItem("classname").getTextContent();
        Object instance = Class.forName(classname).getConstructor().newInstance();

        String beanName = attributes.getNamedItem("name").getTextContent();

        // Finally registering the bean to a beanfactory 
        simpleBeanFactory.registerBean(beanName,instance);
        }
    }
```

We iterate over the beans list to obtain each `<Bean>` node. The attributes of the node can be saved into a map

```java
NamedNodeMap attributes = bean.getAttributes();
```

Individual attributes can be fetched as `attributes.getNamedItem("attribute name").getTextContent()`

```java
String classname = attributes.getNamedItem("classname").getTextContent();
String beanName = attributes.getNamedItem("name").getTextContent();
```

We can obtain the instance of the dependency by using Reflection API
```java
Object instance = Class.forName(classname).getConstructor().newInstance();
```

Finally this instance can be registered against the `beanName` in a BeanFactory as follows:
```java
simpleBeanFactory.registerBean(beanName,instance);
```

Consider the following implementation of BeanFactory, which uses an XML file create Beans.

```java
public class XMLBeanFactory implements BeanFactory {

    private SimpleBeanFactory simpleBeanFactory;

    private static XMLBeanFactory instance = null;
    public static XMLBeanFactory getInstance(){
        if(null == instance)
            throw new RuntimeException("BeanFactory not initialized");
        return instance;
    }


    // Initializing Bean Factory
    public XMLBeanFactory(String xmlPath) {
        simpleBeanFactory = new SimpleBeanFactory();
        Document document = parseXmlDocument(xmlPath);
        NodeList beans = document.getElementsByTagName("Bean");
        registerBeansFromNodeList(beans);
        // Storing last configured XMLBeanFactory
        instance = this;
    }

    private static Document parseXmlDocument(String xmlPath) {
        DocumentBuilder documentBuilder = DocumentBuilderFactory.newInstance().newDocumentBuilder();
        ClassLoader classloader = Thread.currentThread().getContextClassLoader();
        String beanXmlConfigPath = classloader.getResource(xmlPath).getPath();
        Document document = documentBuilder.parse(beanXmlConfigPath);
        return document;
    }

    private void registerBeansFromNodeList(NodeList beans) {
        for(int i = 0; i< beans.getLength(); i++){
            Node bean = beans.item(i);
            NamedNodeMap attributes = bean.getAttributes();

            String classname = attributes.getNamedItem("classname").getTextContent();
            Object instance = Class.forName(classname).getConstructor().newInstance();

            String beanName = attributes.getNamedItem("name").getTextContent();

            simpleBeanFactory.registerBean(beanName,instance);
        }
    }

    @Override
    public void registerBean(String name, Object instance) {
        simpleBeanFactory.registerBean(name, instance);
    }

    @Override
    public Bean getBean(String beanName) {
        return simpleBeanFactory.getBean(beanName);
    }
}
```

The `XMLBeanFactory` internally uses a `SimpleBeanFactory` to manage beans.The `registerBean` and `getBean` simple delegates calls to `simpleBeanFactory`.

Upon calling the `XMLBeanFactory` with the path to XML file, it would parse the XML document, obtain the beans, and then call the `registerBeansFromNodeList` we saw earlier to populate the BeanFactory. We have gone through much of the code already.The `getInstance` method provides provision to obtain the last configured `XMLBeanFactory`

Now, the `configure` method can be refactored as follows
```java
public void configure(){
    beanFactory = new XMLBeanFactory("beans.xml");
    
    // wiring beans
    UserRepository userRepository = (UserRepository) beanFactory.getBean("UserRepository").instance;
    UserService userService = (UserService)beanFactory.getBean("UserService").instance;
    userService.setUserRepository(userRepository);
}
```
Once configured, the dependencies can be obtained like so:
```java
beanFactory = XMLBeanFactory.getInstance();
UserService userService = (UserService) beanFactory.getBean("UserService").instance;
```

