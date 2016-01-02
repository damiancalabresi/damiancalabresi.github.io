---
layout: post
title: "Spring Boot and Java Server Faces"
date: 2015-12-30 23:00:00
author: Damian Calabresi
categories: 
- blog 
- Java
- Java Server Faces
- JSF
- PrimeFaces
img: 
thumb: 
comments: true
postid: 4
---

Have in the same application a Spring MVC context and a JSF servlet is not easy, specially if you are using Spring Boot.
In this example I'll create a JSF Servlet with Primefaces, in my opinion the best implementation.
<!--more-->

## War instead of jar
By default, Spring Boot creates a Jar file with a Tomcat embedded. 
The jar does not contain a webapp directory, in order to have one, you have to change the type in the pom file.
The Java Server Faces servlet will serve the files located in webapp and read the file **faces-config.xml** located in webapp/WEB-INF.


The war file will also be an executable file with a Tomcat embedded, that can be executed using:

> java -jar \<project-name\>.war

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.dcalabresi</groupId>
	<artifactId>spring-boot-jsf</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>war</packaging>

	<name>spring-boot-jsf</name>
	<description>Example of a Java Server Face in a Spring Boot application</description>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.3.0.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<java.version>1.8</java.version>
		<tomcat.version>8.0.23</tomcat.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>com.sun.faces</groupId>
			<artifactId>jsf-impl</artifactId>
			<version>2.2.10</version>
			<scope>compile</scope>
		</dependency>
		<dependency>
			<groupId>com.sun.faces</groupId>
			<artifactId>jsf-api</artifactId>
			<version>2.2.10</version>
			<scope>compile</scope>
		</dependency>
		<dependency>
			<groupId>org.primefaces</groupId>
			<artifactId>primefaces</artifactId>
			<version>5.2</version>
			<scope>compile</scope>
		</dependency>
		<dependency>
			<groupId>org.primefaces.themes</groupId>
			<artifactId>all-themes</artifactId>
			<version>1.0.10</version>
		</dependency>
		<dependency>
			<groupId>commons-fileupload</groupId>
			<artifactId>commons-fileupload</artifactId>
			<version>1.3.1</version>
		</dependency>
		<dependency>
			<groupId>org.apache.tomcat.embed</groupId>
			<artifactId>tomcat-embed-jasper</artifactId>
			<version>${tomcat.version}</version>
			<scope>compile</scope>
		</dependency>

		<dependency>
			<groupId>com.google.guava</groupId>
			<artifactId>guava</artifactId>
			<version>19.0</version>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<repositories>
		<repository>
			<id>prime-repo</id>
			<name>PrimeFaces Maven Repository</name>
			<url>http://repository.primefaces.org</url>
			<layout>default</layout>
		</repository>
	</repositories>
	
	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
	

</project>

{% endhighlight %}

## Declare Faces Servlet

Now that the project has a WEB-INF directory, you should declare the Faces servlet in the **web.xml**.
(The web.xml does not need to have the Spring servlet configuration)

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://java.sun.com/xml/ns/javaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
          http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
	version="2.5">

	<servlet>
		<servlet-name>Faces Servlet</servlet-name>
		<servlet-class>javax.faces.webapp.FacesServlet</servlet-class>
		<load-on-startup>1</load-on-startup>
	</servlet>
	<servlet-mapping>
		<servlet-name>Faces Servlet</servlet-name>
		<url-pattern>*.jsf</url-pattern>
	</servlet-mapping>
	<context-param>
   	 <param-name>primefaces.CLIENT_SIDE_VALIDATION</param-name>
   	 <param-value>true</param-value>
    </context-param>
</web-app>
{% endhighlight %}

Also, in the **WEB-INF** directory, there should be the **faces-config.xml** file

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<faces-config xmlns="http://xmlns.jcp.org/xml/ns/javaee"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee/web-facesconfig_2_2.xsd"
              version="2.2">
    <application>
        <el-resolver>
            org.springframework.web.jsf.el.SpringBeanFacesELResolver
        </el-resolver>
    </application>
</faces-config>
{% endhighlight %}

With this configuration the JSF pages will access to the Spring Beans and the JSF Managed Beans will not be necessary.

## Spring Boot configuration

First, you have to create an initializer that set the PrimeFaces parameters at startup:

{% highlight java %}
@Configuration
public class FacesInitializer implements ServletContextInitializer {

    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {
        System.err.println("------------------------------------");
        servletContext.setInitParameter("primefaces.CLIENT_SIDE_VALIDATION", "true");
        servletContext.setInitParameter("primefaces.THEME", "bootstrap");
        servletContext.setInitParameter("primefaces.UPLOADER", "commons");
    }

}
{% endhighlight %}

Next, you must declare the Faces Servlet in the main Spring Boot Java class and configure it to load the Initializer class.

**Application.java** will look like this:

{% highlight java %}
@EnableAutoConfiguration
@Import({RestConfiguration.class, FacesConfiguration.class})
public class Application extends SpringBootServletInitializer {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(new Class[] { Application.class, FacesInitializer.class});
    }

    @Bean
    public FilterRegistrationBean fileUploadFilterRegistrationBean() {
        FilterRegistrationBean registrationBean = new FilterRegistrationBean();
        registrationBean.addUrlPatterns("*.jsf");
        registrationBean.setFilter(new FileUploadFilter());
        return registrationBean;
    }

    @Bean
    public ServletRegistrationBean servletRegistrationBean(MultipartConfigElement multipartConfigElement) {
        FacesServlet servlet = new FacesServlet();
        ServletRegistrationBean servletRegistrationBean = new ServletRegistrationBean(servlet, "*.jsf");
        servletRegistrationBean.setMultipartConfig(multipartConfigElement);
        return servletRegistrationBean;
    }

}
{% endhighlight %}

As you can see, I import two configuration class. In RestConfiguration a test controller is created, in FacesConfiguration an ExampleManagedBean is created.

## Create JSF pages

Under the **webapp** directory I created a page located in /faces/index.xhtml with these code:

{% highlight xml %}
<f:view xmlns="http://www.w3c.org/1999/xhtml" xmlns:f="http://java.sun.com/jsf/core"
  xmlns:h="http://java.sun.com/jsf/html" xmlns:p="http://primefaces.org/ui"
  xmlns:ui="http://java.sun.com/jsf/facelets">
  <h:head />
  <h:body>
    <h:outputStylesheet library="default" name="css/default.css" />

    <p:outputPanel>
      <h1>Admin Site #{exampleManagedBean.random}</h1>
    </p:outputPanel>

  </h:body>
</f:view>
{% endhighlight %}

The **exampleManagedBean** is a Spring bean that creates a random Integer in each session.
 
## Run the example
In order to see a full working example I suggest to download the code from the repository:

<https://github.com/damiancalabresi/blog-post-code/tree/master/04-spring-boot-jsf>

Go to the directory and run

> mvn spring-boot:run

Then, go to <http://localhost:8080/test/> and <http://localhost:8080/faces/index.jsf> to see an example of a rest service and an example of a JSF page.

I hope this explanation helped you.

Bye!

Remember this code example is uploaded in [github](https://github.com/damiancalabresi/blog-post-code/tree/master/04-spring-boot-jsf).