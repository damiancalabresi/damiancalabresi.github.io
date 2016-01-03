---
layout: post
title: "Automatic Logging with AOP"
date: 2016-01-02 21:47:00
author: Damian Calabresi
categories: 
- blog 
- Java
img: 
thumb: 
comments: true
postid: 7
---

In an enterprise application is a common thing to log all the actions a user makes. 
This can be easily achieved with Aspect Oriented Programming.

I will declare and aspect that logs an entry before and after every service call, only if these service has the related annotation.

<!--more-->

## Aspect Oriented Programming

If you don't know what AOP is I recommend you to read about it and discover what benefits this paradigm could bring to you.
Here are some links:

<https://en.wikipedia.org/wiki/Aspect-oriented_programming>

<https://en.wikipedia.org/wiki/AspectJ>

<http://docs.spring.io/spring/docs/current/spring-framework-reference/html/aop.html>

I use Spring AOP library that relies on AspectJ under the hood. 
To understand the aspect I created on this post is only necessary to know what a pointcut and a join-point are.

## LoggingAspect

I created this aspect. It's called before and after any bean method that have the annotation **@Loggable**

An interesting function I added is the possibility to use the annotation **@NotLog** in a specific argument that you don't want to be logged, like a password.

{% highlight java %}
@Aspect
public class LoggingAspect {

    private Logger logger = LoggerFactory.getLogger(LoggingAspect.class);

    public LoggingAspect() {
    }

    @Pointcut("execution(public * *(..))")
    public void publicMethod() {}

    @Pointcut("@annotation(Loggable)")
    public void methodLoggable() {}

    @Pointcut("publicMethod() && methodLoggable()")
    public void publicAndLoggableMethod() {}

    @Before("publicAndLoggableMethod()")
    public void logServiceCall(JoinPoint joinPoint) throws NoSuchMethodException {
        logger.info("Call      " + this.generateMethodCallDescription(joinPoint) + " - Arguments - " + ReflectionHelper.generateMethodArgumentsDescription(joinPoint));
    }

    @SuppressWarnings("unchecked")
    @AfterReturning(value = "publicAndLoggableMethod()")
    public void logServiceReturn(JoinPoint joinPoint) throws NoSuchMethodException {
        logger.info("Return    " + this.generateMethodCallDescription(joinPoint) + " - Success");
    }

    @AfterThrowing(value = "publicAndLoggableMethod()", throwing = "ex")
    public void logServiceException(JoinPoint joinPoint, Throwable ex) throws NoSuchMethodException {
        logger.info("Exception " + this.generateMethodCallDescription(joinPoint) + " - Error - " + ex.getClass().getSimpleName() + " - " + ex.getMessage());
    }

    @SuppressWarnings("unchecked")
    private String generateMethodCallDescription(JoinPoint joinPoint) throws NoSuchMethodException {
        StringBuilder builder = new StringBuilder();

        Class aClass = joinPoint.getSignature().getDeclaringType();
        MethodSignature method = (MethodSignature) joinPoint.getSignature();

        String className = aClass.getSimpleName();
        String methodName = method.getName();

        builder.append(className);
        builder.append(" - ");
        builder.append(methodName);

        return builder.toString();
    }

}
{% endhighlight %}

Also, there are the annotations @Loggable and @NotLog

{% highlight java %}
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Loggable {
}
{% endhighlight %}

{% highlight java %}
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface NotLog {
}
{% endhighlight %}

The LoggingAspect also needs a class named **ReflectionHelper** that gets the arguments of the methods from the JoinPoint.

This class is so verbose and probably it could be done in a better way. 

{% highlight java %}
public class ReflectionHelper {

    public static Class getClass(JoinPoint joinPoint) {
        return joinPoint.getSignature().getDeclaringType();
    }

    public static MethodSignature getMethod(JoinPoint joinPoint) {
        return (MethodSignature) joinPoint.getSignature();
    }

    public static Map<String, Object> getMethodArguments(JoinPoint joinPoint) {
        MethodSignature method = getMethod(joinPoint);
        String[] parameterNames = method.getParameterNames();
        Object[] params = joinPoint.getArgs();

        Map<String, Object> paramsMap = Maps.newHashMap();
        for (int i=0; i<parameterNames.length; i++) {
            paramsMap.put(parameterNames[i], params[i]);
        }

        return paramsMap;
    }

    @SuppressWarnings("unchecked")
    public static String generateMethodArgumentsDescription(JoinPoint joinPoint) throws NoSuchMethodException {
        StringBuilder builder = new StringBuilder();

        Class aClass = getClass(joinPoint);

        MethodSignature method = getMethod(joinPoint);

        Class<?>[] parameterTypes = method.getParameterTypes();
        Annotation[][] paramAnnotations = getMethodAnnotations(joinPoint);
        Object[] params = joinPoint.getArgs();

        builder.append(" ( ");

        for (int i=0; i<params.length; i++) {
            StringBuilder paramDesc = generateParamDescription(parameterTypes[i],params[i],paramAnnotations[i]);
            builder.append(paramDesc);
            builder.append(", ");
        }
        builder.delete(builder.length() - 2, builder.length());
        builder.append(" )");

        return builder.toString();

    }

    public static Annotation[][] getMethodAnnotations(JoinPoint joinPoint) throws NoSuchMethodException {
        Class aClass = getClass(joinPoint);
        MethodSignature method = getMethod(joinPoint);
        Class<?>[] parameterTypes = method.getParameterTypes();
        String methodName = method.getName();
        return aClass.getMethod(methodName,parameterTypes).getParameterAnnotations();
    }

    private static StringBuilder generateParamDescription(Class<?> parameterType, Object value, Annotation[] annotations) {
        StringBuilder builder = new StringBuilder();
        builder.append(parameterType.getSimpleName());
        builder.append(": ");

        boolean canBeLogged = true;
        for (Annotation annotation : annotations) {
            if(annotation instanceof NotLog) canBeLogged = false;
        }

        if(canBeLogged) {
            builder.append(getObjectDescription(value));
        } else {
            builder.append("****");
        }

        return builder;
    }

    @SuppressWarnings("unchecked")
    private static String getObjectDescription(Object value) {
        if (value==null) {
            return "null";
        } else if (value instanceof Iterable) {
            Iterable<Object> iterValue = (Iterable<Object>) value;
            return getIterableObjectDescription(iterValue);
        } else {
            return value.toString();
        }
    }

    private static String getIterableObjectDescription(Iterable<Object> iterValue) {
        StringBuilder valueBuilder = new StringBuilder("[ ");
        for (Object element : iterValue) {
            valueBuilder.append(element);
            valueBuilder.append(", ");
        }
        valueBuilder.delete(valueBuilder.length() - 2, valueBuilder.length());
        valueBuilder.append(" ]");
        return valueBuilder.toString();
    }

}
{% endhighlight %}

## Declare an aspect in Spring

If you are using annotation configuration in Spring you just have to add the annotation **@EnableAspectJAutoProxy** in a @Configuration class and declare the aspect class as a normal bean.

{% highlight java %}
@Configuration
@EnableAspectJAutoProxy
public class ServiceConfiguration {

    Logger logger = LoggerFactory.getLogger(ServiceConfiguration.class);

    @Bean
    public LoggingAspect loggingAspect() {
        return new LoggingAspect();
    }

    @Bean
    public TestService testService() {
        return new TestService();
    }

}
{% endhighlight %}

## Run the example

If you want to see the log working, just download the code from Github:

<https://github.com/damiancalabresi/blog-post-code/tree/master/06-logging-aop>

Run in the terminal *mvn spring-boot:run* and go to the url:

<http://localhost:8080/test/aUser/aPassword>

You will see in the browser the message:

> Test - Arguments received - User: aUser - Password: aPassword

And in the application log (In the terminal):

> Call      TestService - test - Arguments -  ( String: aUser, String: **** )  
> Return    TestService - test - Success

I hope this post helped you to manage your service logs in a cleaner way.

Remember this code example is uploaded in [github](https://github.com/damiancalabresi/blog-post-code/tree/master/06-logging-aop).