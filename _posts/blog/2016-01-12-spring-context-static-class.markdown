---
layout: post
title: "Access Spring Context from static methods"
date: 2016-01-12 22:00:00
author: Damian Calabresi
categories: 
- blog 
- Java
img: 
thumb: 
comments: true
postid: 8
---

Many Spring based projects have static classes that plays an important role in the application. 
In a Spring application, the use of static classes should be avoided and is not recommended. 
They doesn't have access to the Spring context given that the application context is just an instance, and all the beans depends on it.

But, when you have to deal with legacy code, you will find these static classes everywhere. 
I have seen many times an static method or a singleton class creating and initializing the Spring Context when it already exists.

There is a way to retrieve the only existing ApplicationContext. It's creating an ApplicationContextProvider

<!--more-->

## ApplicationContextProvider

You can create a bean of a class that extends **ApplicationContextAware**. 
When Spring creates this bean it will detect it and call the method **setApplicationContext**.
If you put the ApplicationContext received in an static property, it will be available everywhere.

So, you have to create the following class:

{% highlight java %}
public class ApplicationContextProvider implements ApplicationContextAware {
    
    private static ApplicationContext context;
 
    public ApplicationContext getApplicationContext() {
        return context;
    }
 
    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        context = ctx;
    }
}
{% endhighlight %}

Then, declare it as a Spring Bean.

{% highlight java %}
@Configuration
public class SpringConfiguration() {

    @Bean
    public static ApplicationContextProvider contextProvider() {
        return new ApplicationContextProvider();
    }
    
}
{% endhighlight %}

**It's important to declare the bean creation method in the configuration class as a static method.**
If it isn't static the Spring Configuration won't detect the bean as ApplicationContextAware.

Another way is to activate the ComponentScan and add the @Component annotation to the ApplicationContextProvider.
