---
layout: post
title: "Send transactional emails with JMS and ActiveMQ"
date: 2015-12-12 13:47:00
author: Damian Calabresi
categories: 
- blog 
- Java
- Jms
img: 
thumb: 
comments: true
postid: 3
---

## Make a rollback could be important

This technique allows you to:

* Disconnect email queueing from email sending
* Make a rollback and don't send the email if the flow have an error
* Resend the email if there's any trouble or the SMTP server doesn't respond
* Provide multiple email senders (STMP, Mandrill, MailGun) and select one with a simple configuration

In a web flow, one common action could be send an email. 
For example, when a user do a sign up, the application saves the user data and sends an activation email.
To manipulate this data and save the user, in many databases, like MySQL, you use transactions. 
If an error happens, the data isn't saved and all the data manipulation never happened. 
But **when an email has been sent, and any error occurs, there is no way back**.
A common practice is to put all the email related actions at the end of the transaction, then, if there is an error, it's more likely it will be before sending the email.
This is a simple solution, not the best, for example, the SQL constraints are checked when the transaction is committed, and the email has already been sent.

A nice solution could be **delay the email sending with JMS** (Java Message Service). Doing this, a rollback is possible.
<!--more-->

## Java Message Service
I won't explain here what JMS is, the main functionality of this API is not to delay actions inside an application but it's to simplify communication between many applications.
You can read about it in [Wikipedia](https://en.wikipedia.org/wiki/Java_Message_Service).
We are going to take advantage of the point-to-point model, queue a message in the application and dequeue it after the transaction finishes.

Any implementation of the JMS api can be used, I prefer to use ActiveMQ, one of the most known messaging applications.

## Implementation

First, I based the source on Spring Boot, so here is the Application.java

{% highlight java %}
@EnableAutoConfiguration
@Import({RestConfiguration.class})
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
{% endhighlight %}

Then, following the Spring annotation configuration convention, there are the Configuration classes:

{% highlight java %}
@Configuration
@Import(ServiceConfiguration.class)
public class RestConfiguration {

    Logger logger = LoggerFactory.getLogger(RestConfiguration.class);

    @Bean
    public TestController testController() {
        return new TestController();
    }

}
{% endhighlight %}

{% highlight java %}
@Configuration
@EnableTransactionManagement
@Import(MailConfiguration.class)
public class ServiceConfiguration {

    Logger logger = LoggerFactory.getLogger(ServiceConfiguration.class);

    @Bean
    public EmailService emailService() {
        return new EmailService();
    }

}
{% endhighlight %}

The mail configuration class has the job that takes the mails from the queue and send them. 
To achieve this, the annotation **@EnableScheduling** is a must. Spring Schedule works similar to quartz.

{% highlight java %}
@Configuration
@EnableScheduling
@Import({JmsConfiguration.class})
public class MailConfiguration {

    Logger logger = LoggerFactory.getLogger(MailConfiguration.class);

    @Autowired
    EmailDequeuer emailDequeuer;

    @Scheduled(initialDelay = 10000, fixedDelay = 5000)
    public void startSendEnqueuedEmails() {
        emailDequeuer.sendEnqueuedEmails();
    }

    @Bean
    public EmailQueuer emailQueuer() {
        return new EmailQueuer();
    }

    @Bean
    public EmailDequeuer emailDequeuer() {
        return new EmailDequeuer();
    }

    @Bean
    public EmailSender emailSender() {
        return new EmailSender();
    }

}
{% endhighlight %}

The JmsConfiguration class includes the configurations related to JMS and ActiveMQ, like the in-memmory ActiveMQ, the JmsTemplate and the TransactionManager.

{% highlight java %}
@Configuration
public class JmsConfiguration {

    @Bean
    public ConnectionFactory jmsConnectionFactory() {
        ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory("vm://localhost?broker.persistent=false");
        connectionFactory.setObjectMessageSerializationDefered(true);
        connectionFactory.setCopyMessageOnSend(false);

        CachingConnectionFactory cachingConnectionFactory = new CachingConnectionFactory(connectionFactory);
        return cachingConnectionFactory;
    }

    @Bean
    public JmsTemplate JmsTemplate() {
        JmsTemplate jmsTemplate = new JmsTemplate();
        jmsTemplate.setConnectionFactory(jmsConnectionFactory());
        jmsTemplate.setDefaultDestination(new ActiveMQQueue(Constants.MAIL_QUEUE));
        jmsTemplate.setSessionTransacted(true);
        jmsTemplate.setReceiveTimeout(2000);
        return jmsTemplate;
    }


	@Bean(name = Constants.TRANSACTION_MANAGER_JMS)
	public JmsTransactionManager jmsTransactionManager() {
		return new JmsTransactionManager(jmsConnectionFactory());
	}

}

{% endhighlight %}

Now, I'll go to the specific code. 
The Test Controller is a simple controller that, when executed, calls the service that sends an example email and return the random number sent.

{% highlight java %}
@RestController
@RequestMapping(value = "/test")
public class TestController {

	@Autowired
	EmailService emailService;
	
	@RequestMapping(value="/", method= RequestMethod.GET)
	public String test() {
		return emailService.sendExampleEmail();
	}

}
{% endhighlight %}

There is a simple service called EmailService that makes a simple email.

{% highlight java %}
@Service
public class EmailService {

	private Logger logger = LoggerFactory.getLogger(EmailService.class);

	private Random random = new Random();

	@Autowired
	EmailQueuer emailQueuer;

	@Transactional(transactionManager = Constants.TRANSACTION_MANAGER_JMS)
	public String sendExampleEmail() {
		String text = "Mail number: " + random.nextInt(100);
		emailQueuer.sendMail("from@dcalabresi.com", "Dcalabresi", "to@dcalabresi.com", "",
				"A Subject", text, MimeType.TEXT, null, null);
		return text;
	}

}
{% endhighlight %}

The EmailQueuer is the abstraction that receives the email information and put it in the queue, waiting to be sent.

{% highlight java %}
@Service
public class EmailQueuer {

	private Logger logger = LoggerFactory.getLogger(EmailQueuer.class);

	@Autowired
	JmsTemplate jmsTemplate;


	public void sendMail(String from, String fromName, String to, String unsubscribeUrl, String subject, String body,
						 MimeType mimeTypee, String fileBase64, String fileName) {
		EmailDto emailDto = new EmailDto(from, fromName, to, unsubscribeUrl, subject, body, mimeTypee, fileBase64, fileName);
		validateEmail(emailDto);
		jmsTemplate.convertAndSend(emailDto);
	}

	private void validateEmail(EmailDto emailDto) {
		// HERE SHOULD BE THE VALIDATIONS
	}

}
{% endhighlight %}

The, there is the EmailDequeuer. 
The method sendEnqueuedEmails is executed by Spring Schedule and it sends, one by one, all the pending emails in the JMS queue.

{% highlight java %}
@Service
public class EmailDequeuer {

	private Logger logger = LoggerFactory.getLogger(EmailDequeuer.class);

	@Autowired
	JmsTemplate jmsTemplate;

	@Autowired
	EmailSender emailSender;

	@Transactional(transactionManager = Constants.TRANSACTION_MANAGER_JMS)
	public void sendEnqueuedEmails() {
		while (sendEmailInQueue()) {}
	}

	private boolean sendEmailInQueue() {
		Object received = jmsTemplate.receiveAndConvert();
		if (received == null) {
			return false;
		} else if (received instanceof EmailDto) {
			EmailDto emailDto = (EmailDto) received;
			logger.info("Sending email from [" + emailDto.getFrom() + "] to [" + emailDto.getTo() + "] subject [" + emailDto.getSubject() + "]");
			EmailStatus emailStatus = emailSender.sendEmail(emailDto);
			return true;
		} else {
			logger.error("The mail has not the corresponding format - class: " + received.getClass().toString());
			return true;
		}
	}

}
{% endhighlight %}

The key part is the sender. Here I developed a simple email sender that prints the content to the standard output.
You can send the email using Java Mail, Spring Mail or a REST service like Mandrill or MailGun.

{% highlight java %}
@Service
public class EmailSender {

	private Logger logger = LoggerFactory.getLogger(EmailSender.class);

	public EmailStatus sendEmail(EmailDto emailDto) {
		logger.info("Sending an email - This sender only writes to the standard output");
		System.out.println("SEND EMAIL:");
		System.out.print(emailDto.toString());
		return new EmailStatus(emailDto.getTo(), "OK", "");
	}

}
{% endhighlight %}

### Test the application
To test it, download the source code in [github](https://github.com/damiancalabresi/blog-post-code/tree/master/03-jms-transactional-email).
Execute it like a simple jar (It's Spring Boot magic) and go to http://localhost:8080/test/
You will see a random number as a response:

> Mail number: 15

A few seconds later, in the console or the standard output will be a line like this:

> SEND EMAIL:
> EmailDto{from=from@dcalabresi.com, fromName=Dcalabresi, to=to@dcalabresi.com, unsubscribeUrl=, subject=A Subject, body=Mail number: 15, mimeType=TEXT, fileBase64=null, fileName=null}

Remember this code example is uploaded in [github](https://github.com/damiancalabresi/blog-post-code/tree/master/03-jms-transactional-email).

