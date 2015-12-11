---
layout: post
title: "Java Arguments, by reference or by value?"
date: 2015-12-11 17:00:00
author: Damian Calabresi
categories: 
- blog 
- Java
img: 
thumb: 
comments: true
postid: 2
---

This could be a trivial question, but working with many people I have listen this question a few times.
I have seen bugs caused by this mistake. The question is:

Java arguments are passed by reference or by value?

The answer is **by value**. A more precise answer could be **the object pointer is passed by value**.

<!--more-->

This is a doubt that comes generally in developers that made their first steps in C. I am one of these, in C language you have a tight control of this behavior,
in some high level languages all is passed by reference, Java just stands in the middle.
Java has another particularity, integer, float, double, etc. are not objects, they are primitive types.

I will describe each case:

#### Object instance
If a function receives an object instance and modifies any property, the change will be seen outside this function.

If the function assigns another instance to the variable, it will change the pointer, so the change won't be seen outside.

#### Integer and other primitives
The behavior will be the same if you use the primitive **integer** or the class **Integer**. 
You can't set a new value to an integer, you can only assign a new integer, so there is no way to modify an integer inside a function and see new value outside. 
The value must be returned.

This case could be extended to other primitives values as double, float, char, etc.

#### String
The String object is special. It's an object like any other, but it's immutable. 
An String instance contains an array of chars, any function that modify the String will return a new instance.
Even if you use the concatenate function, you will be creating a new String. 
In order to pass a String by reference you have to use a container class, like StringBuilder.
 
#### Workaround - Pass primitive variables by reference
You should use a container, and object that has the primitive as a property.
In my opinion, the easiest way to do this is with the class **AtomicInteger, AtomicBoolean, AtomicLong, etc.**

### Example
In this code snippet you can see what happens in each case explained above:

{% highlight java %}
public class Application {

    public static void main( String[] args ) {
        Person person = new Person("Damian");
        System.out.println("Name: " + person.name);

        System.out.println("Test pointer by reference");
        testPointerByReference(person);
        System.out.println("Name: " + person.name);

        System.out.println("Test pointer by value");
        person = testPointerByValue(person);
        System.out.println("Name: " + person.name);

        System.out.println("Modify object");
        modifyObject(person);
        System.out.println("Name: " + person.name);

        System.out.println("Primitives - Integer");
        Integer aNumber = 5;
        modifyInt(aNumber);
        System.out.println("Integer: " + aNumber);

        System.out.println("Immutable - String (Not really a primitive)");
        String aString = "First value";
        modifyString(aString);
        System.out.println("String: " + aString);

        System.out.println("Pass an Integer by reference");
        AtomicInteger otherNumber = new AtomicInteger(5);
        modifyAtomicInt(otherNumber);
        System.out.println("Integer: " + otherNumber);
    }

    private static void testPointerByReference(Person person) {
        person = new Person("Mark");
    }

    private static Person testPointerByValue(Person person) {
        person= new Person("Thomas");
        return person;
    }

    private static void modifyObject(Person person) {
        person.name = "Paul";
    }

    private static void modifyInt(Integer aNumber) {
        aNumber = 7;
    }

    private static void modifyString(String aString) {
        // Even if you concatenate the String, Java will generate a copy.
        aString.concat("Another value");
    }

    private static void modifyAtomicInt(AtomicInteger otherNumber) {
        otherNumber.set(7);
    }

}
{% endhighlight %}

The output will be:

> Name: Damian  
> Test pointer by reference  
> Name: Damian  
> Test pointer by value  
> Name: Thomas  
> Modify object  
> Name: Paul  
> Primitives - Integer  
> Integer: 5  
> Immutable - String (Not really a primitive)  
> String: First value  
> Pass an Integer by reference  
> Integer: 7  

Remember this code example is uploaded in [github](https://github.com/damiancalabresi/blog-post-code/tree/master/02-java-arguments-by-value).

