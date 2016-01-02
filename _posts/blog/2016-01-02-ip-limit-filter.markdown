---
layout: post
title: "Limit requests by IP with a Servlet Filter"
date: 2016-01-02 19:00:00
author: Damian Calabresi
categories: 
- blog 
- Java
img: 
thumb: 
comments: true
postid: 6
---

In order to meet the requirements of a project at my work I had to develop a filter that limits the requests by IP.

*"The application has to limit the amount of requests each IP address could make in a period of time."*

This Java Servlet Filter is compound by the **IpTimeWindowManager** and the specific **IpLimitFilter**.

<!--more-->

## IpTimeWindowManager

Here the period of time and the number of requests are constants. This should be modified if you want the limits to be configurable.

{% highlight java %}
public class IpTimeWindowManager {


    public static final int WINDOW_SIZE_IN_MINUTES = 30;
    public static final int MAX_REQUEST_PER_IP_IN_WINDOW = 5;
    private long lastEpochMinute;

    private LinkedListMultimap<String, Long> requestsPerIp;

    public IpTimeWindowManager() {
        requestsPerIp = LinkedListMultimap.create();
        lastEpochMinute = 0;
    }

    public synchronized void addIpRequest(String ipAddress) {
        long epochSecond = LocalDateTime.now().atZone(ZoneId.systemDefault()).toEpochSecond();
        requestsPerIp.put(ipAddress, epochSecond);

        long epochMinute = epochSecond - (epochSecond % 60);
        if (epochMinute > lastEpochMinute) {
            lastEpochMinute = epochMinute;
            cleanExpiredRequests();
        }
    }

    private void cleanExpiredRequests() {
        long expiredEpochMinute = lastEpochMinute - (WINDOW_SIZE_IN_MINUTES * 60);

        for (String ipAddress : requestsPerIp.keySet()) {
            List<Long> requests = requestsPerIp.get(ipAddress);

            for (Long request : requests) {
                if (request < expiredEpochMinute) {
                    requests.remove(request);
                }
            }

            if (requests.isEmpty()) {
                requestsPerIp.removeAll(ipAddress);
            }

        }
    }

    public synchronized boolean ipAddressReachedLimit(String ipAddress) {
        int amountRequests = requestsPerIp.get(ipAddress).size();
        return (amountRequests > MAX_REQUEST_PER_IP_IN_WINDOW);
    }
}
{% endhighlight %}

## IpLimitFilter

The same happens here, the limited paths are constants of the class, this could be improved.

Notice that the limited path is just the prefix, so, for example, **/test/limited/anotherTest** will also be restricted.

{% highlight java %}
public class IpLimitFilter implements Filter {

    private static String[] LIMITED_PATHS = new String[]{"/test/limited"};

    Logger logger = LoggerFactory.getLogger(IpLimitFilter.class);

    IpTimeWindowManager ipTimeWindowManager;

    public IpLimitFilter(IpTimeWindowManager ipTimeWindowManager) {
        this.ipTimeWindowManager = ipTimeWindowManager;
    }

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        HttpServletRequest request = getHttpServletRequest(servletRequest);

        boolean isRestServicePostCall = isRestPublicUserServicePostCall(request);

        if(isRestServicePostCall) {
            String ipAddress = request.getRemoteAddr();
            ipTimeWindowManager.addIpRequest(ipAddress);

            if (ipTimeWindowManager.ipAddressReachedLimit(ipAddress)) {
                String message = "The ip address: " + ipAddress + " made more than " + IpTimeWindowManager.MAX_REQUEST_PER_IP_IN_WINDOW + " requests in " + IpTimeWindowManager.WINDOW_SIZE_IN_MINUTES + " minutes. It's suspicious.";
                logger.error(message);
                throw new ServletException(message);
            }
        }

        filterChain.doFilter(servletRequest, servletResponse);
    }

    private HttpServletRequest getHttpServletRequest(ServletRequest servletRequest) throws ServletException {
        if (servletRequest instanceof HttpServletRequest) {
            return (HttpServletRequest) servletRequest;
        } else {
            Request springRequest = (Request) servletRequest.getAttribute("org.springframework.web.context.request.RequestContextListener.REQUEST_ATTRIBUTES");
            if (springRequest instanceof HttpServletRequest) {
                return springRequest;
            } else {
                throw new ServletException("At least the inner request should be a HttpServletRequest");
            }
        }
    }

    private boolean isRestPublicUserServicePostCall(HttpServletRequest request) {
        String requestedUri = request.getRequestURI();
        return Arrays.stream(LIMITED_PATHS).anyMatch(limitedPath -> requestedUri.startsWith(limitedPath));
    }

    @Override
    public void destroy() {
    }

}
{% endhighlight %}

## Spring Boot Filter Configuration

In Spring Boot the filter is configured creating the FilterRegistrationBean. 
The filter has the lower precedence in order to be one of the first filters to be executed in the filter chain.

So, this is the **Application.java** file:

{% highlight java %}
@EnableAutoConfiguration
@Import({RestConfiguration.class})
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @Bean
    @Scope(value="singleton")
    @Order(Ordered.LOWEST_PRECEDENCE)
    public IpTimeWindowManager ipTimeWindowManager() {
        return new IpTimeWindowManager();
    }

    @Bean
    @Order(Ordered.LOWEST_PRECEDENCE)
    public IpLimitFilter ipLimitFilter(IpTimeWindowManager ipTimeWindowManager) {
        return new IpLimitFilter(ipTimeWindowManager);
    }

    @Bean
    @Order(Ordered.LOWEST_PRECEDENCE)
    public FilterRegistrationBean ipLimitFilterRegistrationBean(IpLimitFilter ipLimitFilter) {
        FilterRegistrationBean registrationBean = new FilterRegistrationBean();
        registrationBean.setFilter(ipLimitFilter);
        registrationBean.setOrder(Ordered.LOWEST_PRECEDENCE);
        return registrationBean;
    }

}
{% endhighlight %}

## Run the example
To test the filter, download the code in:

<https://github.com/damiancalabresi/blog-post-code/tree/master/05-ip-limit-filter>

And run:

> mvn spring-boot:run

Then hit more than five times the URL:

<http://localhost:8080/test/>

And hit more than five times:

<http://localhost:8080/test/limited/> (After five requests the service should fail)

Remember this code example is uploaded in [github](https://github.com/damiancalabresi/blog-post-code/tree/master/05-ip-limit-filter).