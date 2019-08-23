---
title: Tomcat新版本Host Name Validate问题
date: 2019-05-20 11:48:00
tags: 
- Tomcat
- Spring Boot
---

摘要：
1. Tomcat不同版本对HTTP请求中valid characters的配置。
2. Tomcat新版本中Host Name Validate的开启。
3. Spring Boot和Tomcat版本间的对应关系。

<!-- more -->

最近将一个项目由Spring Cloud Finchely.M5升级到Greenwich.SR1，在灰度环境正常，而线上则在网关层报400 Bad Request。

首先怀疑是特殊字符的问题，因为之前也遇到过这个问题。Tomcat 7.0.73中有一处改动使得带有{}|等符号的查询会返回400。

```
Add additional checks for valid characters to the HTTP request line parsing so invalid request lines are rejected sooner. 
```

在Tomcat 7.0.76中，添加了配置项。可以配置Tomcat行为

```
tomcat.util.http.parser.HttpParser.requestTargetAllow=
```

之前该项目也是通过这个配置来避免400，现在为什么不行了呢？查询Tomcat文档，该配置在[Tomcat 8](https://tomcat.apache.org/tomcat-8.5-doc/config/systemprops.html)中已标记为deprecated，在[Tomcat 9](https://tomcat.apache.org/tomcat-9.0-doc/config/systemprops.html)中移除，使用`relaxedPathChars`和`relaxedQueryChars`替代。

```
This system property is deprecated. Use the relaxedPathChars and relaxedQueryChars attributes of the Connector instead. These attributes permit a wider range of characters to be configured as valid.
```

Spring Boot 2.0.x到2.1.x的升级中，Tomcat版本由8.5升级至9.0，因此需要配置relaxedQueryChars和relaxedPathChars。

```java
    @Bean
    public WebServerFactoryCustomizer<TomcatServletWebServerFactory> containerCustomizer() {
        return new EmbeddedTomcatCustomizer();
    }

    public static class EmbeddedTomcatCustomizer implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {

        @Override
        public void customize(TomcatServletWebServerFactory factory) {
            factory.addConnectorCustomizers((TomcatConnectorCustomizer) connector -> {
                connector.setAttribute("relaxedPathChars", "\"<>[\\]^`{|}");
                connector.setAttribute("relaxedQueryChars", "\"<>[\\]^`{|}");
            });
        }
    }
```

但更新后依然报400。查看服务日志，找到一段异常。

```
http-nio-8080-exec-1 | INFO  | org.apache.coyote.http11.Http11Processor(175) KEY: | Error parsing HTTP request header
 Note: further occurrences of HTTP request parsing errors will be logged at DEBUG level.
java.lang.IllegalArgumentException: Invalid character found in the request target. The valid characters are defined in RFC 7230 and RFC 3986
	at org.apache.coyote.http11.Http11InputBuffer.parseRequestLine(Http11InputBuffer.java:467)
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:294)
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:66)
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:834)
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1415)
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
	at java.lang.Thread.run(Thread.java:748)
```

似乎是请求处理有问题，开启DEBUG日志，异常如下。

```
http-nio-8080-exec-5 | DEBUG | org.apache.coyote.http11.Http11Processor(175) KEY: | The host [finance_pools] is not valid
java.lang.IllegalArgumentException: The character [_] is never valid in a domain name.
	at org.apache.tomcat.util.http.parser.HttpParser$DomainParseState.next(HttpParser.java:926)
	at org.apache.tomcat.util.http.parser.HttpParser.readHostDomainName(HttpParser.java:822)
	at org.apache.tomcat.util.http.parser.Host.parse(Host.java:71)
	at org.apache.tomcat.util.http.parser.Host.parse(Host.java:45)
	at org.apache.coyote.AbstractProcessor.parseHost(AbstractProcessor.java:288)
	at org.apache.coyote.http11.Http11Processor.prepareRequest(Http11Processor.java:809)
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:384)
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:66)
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:834)
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1415)
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
	at java.lang.Thread.run(Thread.java:745)
```

原来是Tomcat 8.5.31/9.0.5后开启强制域名验证

```
Enable strict validation of the provided host name and port for all connectors. Requests with invalid host names and/or ports will be rejected with a 400 response. (markt)
```

而运维在nginx中配置的upstream名称为`finance_pools`，因此被tomcat拦截。

对比Spring Cloud不同版本中不同的Tomcat版本。可见升级前版本不需严格校验，升级后版本需要。因此导致400问题。

| Spring Cloud版本  | Spring Boot版本 | Tomcat版本 |
|-------------------|-----------------|------------|
| Finchley.M5       | 2.0.0.M7        | 8.5.23     |
| Greenwich.SR1     | 2.1.3.RELEASE   | 9.0.16     |

联系运维将upstream名称中的下划线去掉后正常。

# 总结

1. Tomcat 8 -> 9时，需要将`tomcat.util.http.parser.HttpParser.requestTargetAllow`替换为relaxedQueryChars和relaxedPathChars配置。
2. Tomcat新版本中host name validate为强制校验，因此无法使用带有下划线的host name。
