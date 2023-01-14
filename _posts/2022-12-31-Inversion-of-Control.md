---
layout: post
title:  "Inversion of Control"
tags: ['DEPENDENCY INJECTION']    
---

Inversion of Control(IoC) is a technique we use to plug-in custom behaviors into a generic framework. This can be achieved in two ways:
* Creating instances of  classes exposed by framework
* Creating subtypes of classes/interfaces exposed by framework. 

An example for IoC would be callbacks for UI events. Consider the following code for a button in Java Swing.

```java
JButton submit = new JButton("submit");
b.addActionListener(new ActionListener() {
    @Override
    public void actionPerformed(ActionEvent e) {
        //custom code to be run on click
    }
});
```

The code within `actionPerformed` will be called by the Swing Framework when user interacts with the `submit` button. This is an inversion from normal flow of control where the methods we write will be called by ourselves. Hence the name Inversion of Control. This act of delegating control over a method to a framework is called `Hollywood Principle` - *Don't call us, we will call you*. 

Inversion of Control has a slightly different meaning in context of IoC containers. While writing code against an IoC container, we delegate the responsibility of injecting/wiring dependencies, to the framework. Though there is an Inversion of Control in this pattern, it doesnt conform to the general idea of Inversion of Control principle. To avoid confusion, we call the Inversion of Control proposed by IoC containers, Dependency Injection.

-----

# Setter Injection

In the previous article, we created a framework that manages beans for us. In this article we will be focusing on wiring these beans together.

As discussed in the first article there are several approaches to injecting beans. Most straightforward approach would be setter injection.

Consider the following config file for our DI framework:

```xml
<Beans>
    <Bean name="UserRepository" classname="in.onexdev.testScenarios.UserRepository" />
    <Bean name="UserService" classname="in.onexdev.testScenarios.UserService">
        <setter name="setUserRepository" bean="UserRepository"/>
    </Bean>
</Beans>
```

We have defined the `UserRepository` and `UserService` bean as before.The only difference we is that  the `UserService` now has a `setter` node as child.

The `UserRepository` class is designed like so:

```java
public class UserService {
    private UserRepository userRepository;

    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User getUserDetails(Long id){
        return userRepository.findById(id);
    }
}
```

The `setUserRepository` accepts an instance of `UserRepository`, which it later persists in the `userRepository` field. All our framework have to do is invoke this method with a `UserRepository` bean

The config
```xml
 <Bean name="UserService" classname="in.onexdev.testScenarios.UserService">
        <setter name="setUserRepository" bean="UserRepository"/>
</Bean>
```
can be roughly transalted as 
* Create an instance of `UserService`
* Invoke the setter method `setUserRepository` method with `UserRepository` bean as argument.

To acheive this, our DI framework should traverse through each `Bean` node in parsed xml document. If any `Bean` node has a `setter` child node, the framwork uses java reflection to call corresponding setter method with the given argument.

The layout of our framwork is mostly unchanged

```java
 public XMLBeanFactory(String xmlPath) {
    // initalize a SimpleBeanFactory
    simpleBeanFactory = new SimpleBeanFactory();
    // parse the bean configuration XML
    Document document = parseXmlDocument(xmlPath);
    // obtain all Bean nodes in file
    NodeList beans = document.getElementsByTagName("Bean");
    // register all beans to the SimpleBeanFactory
    registerBeansFromNodeList(beans);
    // persist the newly created XMLBeanFactory for later access
    instance = this;
}
```
 
The `registerBeansFromNodeList` method has following implementation :
```java
private void registerBeansFromNodeList(NodeList beanNodes){
    // Iterate through beans
    for(int i = 0; i< beanNodes.getLength(); i++){
        Node beanNode = beanNodes.item(i);
        NamedNodeMap attributes = beanNode.getAttributes();

        // Obtain classname of each bean and intialize it using reflection 
        String classname = attributes.getNamedItem("classname").getTextContent();
        Object instance = Class.forName(classname).getConstructor().newInstance();

        // apply setter injection on each Bean   
        injectSetterDependencies(instance, beanNode);

        // finally, save the bean instance in the simple bean factory
        String beanName = attributes.getNamedItem("name").getTextContent();
        simpleBeanFactory.registerBean(beanName,instance);
    }
}

```

As you can see, we have intoduced a new method, `injectSetterDependencies(instance, beanNode)` 

```java
private  void injectSetterDependencies(Object instance, Node beanNode)  {
    // get all "setter" child nodes of a Bean node
    NodeList setterNodes = ((Element)beanNode).getElementsByTagName("setter");
    // iterate though the setter nodes
    for(int i=0; i<setterNodes.getLength(); i++){
        Node setterNode = setterNodes.item(i);
        NamedNodeMap attributes = setterNode.getAttributes();

        // obtain method name from "name" attribute of setter
        String methodName = attributes.getNamedItem("name").getTextContent();

        // obtain dependency  from "bean" attribute of setter
        String dependencyName = attributes.getNamedItem("bean").getTextContent();
        Bean dependency = simpleBeanFactory.getBean(dependencyName);

        // invoke method using reflection
        Method setterMethod = instance.getClass().getMethod(methodName,Class.forName(dependency.className));
        setterMethod.invoke(instance,dependency.instance);
    }
}
```

We can now test our code as :

```java
beanFactory = XMLBeanFactory.getInstance();

UserService userService = (UserService) beanFactory.getBean("UserService").instance;

User userDetails = userService.getUserDetails(10L);
assertEquals(userDetails.getAge(), 25);
```

We are directly invoking `getUserDetails` method from `userService` instance obtained from the framework. The framework has taken care of binding `userRepository` into `UserService`.



