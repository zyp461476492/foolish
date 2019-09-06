---
title: spring-cloud-微服务实战-第四章学习笔记
date: 2019-09-03 09:33:01
tags:
    - 学习笔记
    - spring 
    - spring cloud
    - spring boot
categories: 学习笔记
---

> spring-cloud 学习笔记，第四章，介绍了 Spring-cloud-ribbon 相关内容 ，记录笔记，供个人回顾使用。

<!-- more -->

## RestTemplate 详解

目前理解功能是封装的通过 REST 远程访问经过 eureka 负载均衡的 spring 服务。

对于 HTTP 的 GET,POST,PUT,DELETE 等均有对应实现。

## 源码分析

@LoadBalanced 注解的作用是标记一个 RestTemplate Bean , 用 LoadBalancerClient 来对它进行配置。

LoadBalancerClient 是 org.springframework.cloud.client.loadbalancer 下的一个接口，它提供了三个方法。

```java
    /**
    * 使用来自 LoadBalancer 的 ServiceInstance 来执行请求对于指定的服务。
    * @param serviceId 待查找 LoadBalancer 的 serviceId
    * @param request 提供给实现者用于执行前和执行后的一些动作配置，比如新增 metrics 监控点等。
    * @return 对于选择的 ServiceInstance 的 LoadBalancerRequest 请求结果。
    */
    <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;

    <T> T execute(String serviceId, ServiceInstance serviceInstance,
            LoadBalancerRequest<T> request) throws IOException;
    /**
    * 构建一个合适的 HOST:PORT 的 URI 给分布式系统使用。
    * 有些系统会使用逻辑服务名作为 HOST，例如 http://myservice/path/to/service。
    * 这个方法会用ServiceInstance的 HOST:PORT 来替换上述的 URI。
    */
    URI reconstructURI(ServiceInstance instance, URI original);
```

LoadBalancerAutoConfiguration 是 org.springframework.cloud.client.loadbalancer 包下的，为了实现客户端负载均衡的自动化配置类。

```java
@Configuration
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerRetryProperties.class)
public class LoadBalancerAutoConfiguration {
    //...
}
```

@ConditionalOnClass, 指定类位于 classpath 时，匹配成功

@ConditionalOnBean，当所有依赖的 bean 已经存在于 BeanFactory 容器中时，匹配成功。

@ConditionalOnMissingBean 当前 bean 不存在 BeanFactory 中时，匹配成功。

@EnableConfigurationProperties 开启可以通过 ConfigurationProperties 的注解 bean 来对自动配置文件进行配置

所以 LoadBalancerAutoConfiguration，在 classpath 下有 RestTemplate.class 和 BeanFactory 存在 LoadBalancerClient 时，自动启用。同时，支持通过 LoadBalancerRetryProperties 类来对配置内容进行修改。

继续看源码,下面的配置是当缺少 RetryTemplate.class 时，进行配置的一些内容。

```java
    /**
    * 当不存在 LoadBalancerRequestFactory 时，会向 BeanFactory 中注册一个 LoadBalancerRequestFactory 类
    */
    @Bean
    @ConditionalOnMissingBean
    public LoadBalancerRequestFactory loadBalancerRequestFactory(
            LoadBalancerClient loadBalancerClient) {
        return new LoadBalancerRequestFactory(loadBalancerClient, this.transformers);
    }

    /**
    * classpath 缺少 RestTemplate.class 时，会注册 LoadBalancerInterceptor
    * classpath 缺少 RestTemplate.class 且不存在 RestTemplateCustomizer ，会注册 RestTemplateCustomizer
    */
    @Configuration
    @ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
    static class LoadBalancerInterceptorConfig {
        @Bean
        public LoadBalancerInterceptor ribbonInterceptor(
                LoadBalancerClient loadBalancerClient,
                LoadBalancerRequestFactory requestFactory) {
            return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
        }

        @Bean
        @ConditionalOnMissingBean
        public RestTemplateCustomizer restTemplateCustomizer(
                final LoadBalancerInterceptor loadBalancerInterceptor) {
            return restTemplate -> {
                List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                        restTemplate.getInterceptors());
                list.add(loadBalancerInterceptor);
                restTemplate.setInterceptors(list);
            };
        }
    }

    // 后面一些 RetryTemplate.class 相关的配置类暂时省略
```

下面，查看 LoadBalancerInterceptor 拦截器如何将一个普通的 RestTemplate 变成客户端负载均衡的

```java
    @Override
    public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
            final ClientHttpRequestExecution execution) throws IOException {
        final URI originalUri = request.getURI();
        String serviceName = originalUri.getHost();
        Assert.state(serviceName != null,
                "Request URI does not contain a valid hostname: " + originalUri);
        return this.loadBalancer.execute(serviceName,
                this.requestFactory.createRequest(request, body, execution));
    }
```

之前也提到过，RestTemplate 的 host 为服务名，所以通过 URI 获取的 host 就是 service 的服务名。负载均衡，通过 serviceName 选择实例，并发起请求。

接下来，我们查看一下 ribbo 对于 LoadBalancerClient 的具体实现。

```java
public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint)
            throws IOException {
        ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
        Server server = getServer(loadBalancer, hint);
        if (server == null) {
            throw new IllegalStateException("No instances available for " + serviceId);
        }
        RibbonServer ribbonServer = new RibbonServer(serviceId, server,
                isSecure(server, serviceId),
                serverIntrospector(serviceId).getMetadata(server));

        return execute(serviceId, ribbonServer, request);
    }
```

新加入了一个 hint 来和 execute(serviceId, ServiceInstance, request) 区分，默认 hint 传递的是 null。

开始会通过 getServer 来获取 server。代码如下,这里通过 com.netflix.loadbalancer 的 ILoadBalancer 来选择 server

```java
protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
        if (loadBalancer == null) {
            return null;
        }
        // Use 'default' on a null hint, or just pass it on?
        return loadBalancer.chooseServer(hint != null ? hint : "default");
    }
```

ILoadBalancer 接口提供了下列方法

```java
public interface ILoadBalancer {
    public void addServers(List<Server> newServers);
    public Server chooseServer(Object key);
    public void markServerDown(Server server);
    public List<Server> getReachableServers();
    public List<Server> getAllServers();
}
```

通过查看 RibbonClientConfiguration 可以发现，Ribbon 用 ZoneAwareLoadBalancer 来默认实现 ILoadBalancer 来实现负载均衡器。

```java
    @Bean
    @ConditionalOnMissingBean
    public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
            ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
            IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
        if (this.propertiesFactory.isSet(ILoadBalancer.class, name)) {
            return this.propertiesFactory.get(ILoadBalancer.class, config, name);
        }
        return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList,
                serverListFilter, serverListUpdater);
    }
````

在通过 ILoadBalancer 来获取到 server 后，将 server 包装成了 RibbonServer , 随后获取了 serviceId 对应的 负载均衡的上下文，记录了状态，再回调 LoadBalancerRequest 的 apply 方法进行处理。

```java
@Override
    public <T> T execute(String serviceId, ServiceInstance serviceInstance,
            LoadBalancerRequest<T> request) throws IOException {
        Server server = null;
        if (serviceInstance instanceof RibbonServer) {
            server = ((RibbonServer) serviceInstance).getServer();
        }
        if (server == null) {
            throw new IllegalStateException("No instances available for " + serviceId);
        }

        RibbonLoadBalancerContext context = this.clientFactory
                .getLoadBalancerContext(serviceId);
        RibbonStatsRecorder statsRecorder = new RibbonStatsRecorder(context, server);

        try {
            T returnVal = request.apply(serviceInstance);
            statsRecorder.recordStats(returnVal);
            return returnVal;
        }
        // catch IOException and rethrow so RestTemplate behaves correctly
        catch (IOException ex) {
            statsRecorder.recordStats(ex);
            throw ex;
        }
        catch (Exception ex) {
            statsRecorder.recordStats(ex);
            ReflectionUtils.rethrowRuntimeException(ex);
        }
        return null;
    }
```

传入 apply 的 ServiceInstance 是对服务实例的抽象定义，在该接口中暴露出了每个服务实例需要的基本信息，例如 serviceId, host, port 等。

```java
public interface ServiceInstance {
    default String getInstanceId() {
        return null;
    }

    String getServiceId();

    String getHost();

    int getPort();

    boolean isSecure();

    URI getUri();

    Map<String, String> getMetadata();

    default String getScheme() {
        return null;
    }
}
```

到这里，重新整理一下逻辑。

在 LoadBalancerAutoConfiguration 中，通过 LoadBalancerInterceptor bean 来拦截负载均衡的 RestTemplate 请求，从而达到负载均衡的效果。

LoadBalancerInterceptor 是由 LoadBalancerClient 和 LoadBalancerRequestFactory 两个参数生成。

- LoadBalancerClient 主要功能是执行请求，上文也分析了 excute 方法。
- LoadBalancerRequestFactory 是用来构造 LoadBalancerRequest 请求对象。

现在分析一下 LoadBalancerRequestFactory, 该对象由一个 LoadBalancerClient 对象和一个 ```List<LoadBalancerRequestTransformer>``` 生成。

```java
public LoadBalancerRequestFactory(LoadBalancerClient loadBalancer,
            List<LoadBalancerRequestTransformer> transformers) {
        this.loadBalancer = loadBalancer;
        this.transformers = transformers;
    }
```

LoadBalancerRequestTransformer 的主要功能就是传递负载均衡的 HttpRequest 给指定的 ServiceInstance。

```java
@Order(LoadBalancerRequestTransformer.DEFAULT_ORDER)
public interface LoadBalancerRequestTransformer {

    /**
     * Order for the load balancer request tranformer.
     */
    int DEFAULT_ORDER = 0;

    HttpRequest transformRequest(HttpRequest request, ServiceInstance instance);

}
```

接着看 LoadBalancerRequestFactory，其中核心的方法是 createRequest 方法，目的是生成一个 LoadBalancerRequest 对象。

```java
public LoadBalancerRequest<ClientHttpResponse> createRequest(
            final HttpRequest request, final byte[] body,
            final ClientHttpRequestExecution execution) {
        return instance -> {
            HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance,
                    this.loadBalancer);
            if (this.transformers != null) {
                for (LoadBalancerRequestTransformer transformer : this.transformers) {
                    serviceRequest = transformer.transformRequest(serviceRequest,
                            instance);
                }
            }
            return execution.execute(serviceRequest, body);
        };
    }
```

这里通过 ServiceRequestWrapper 来包装 request 和 serviceInstance，来看下 ServiceRequest 方法，构造的时候要传入 LoadBalancerClient 对象，通过调 this.loadBalancer.reconstructURI 来对 URI 进行重构。同时，该对象继承了 HttpRequestWrapper ，最后就可以生成一个 HttpRequest 对象，到这里，就把请求转换为了 HttpRequest。

```java
public class ServiceRequestWrapper extends HttpRequestWrapper {

    private final ServiceInstance instance;

    private final LoadBalancerClient loadBalancer;

    public ServiceRequestWrapper(HttpRequest request, ServiceInstance instance,
            LoadBalancerClient loadBalancer) {
        super(request);
        this.instance = instance;
        this.loadBalancer = loadBalancer;
    }

    @Override
    public URI getURI() {
        URI uri = this.loadBalancer.reconstructURI(this.instance, getRequest().getURI());
        return uri;
    }

}
```

这里再调用前面的 LoadBalancerRequestTransformer , 根据 ServiceInstance 的信息，将请求发送到对应的服务实例上。

```java
if (this.transformers != null) {
                for (LoadBalancerRequestTransformer transformer : this.transformers) {
                    serviceRequest = transformer.transformRequest(serviceRequest,
                            instance);
                }
            }
```

到这里，就说明白了 LoadBalancerClient 和 LoadBalancerRequestFactory 在 LoadBalancerInterceptor 下的作用。

继续看 LoadBalancerInterceptor 的 intercept 方法，也就是实际对请求进行拦截的方法。

```java
@Override
    public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
            final ClientHttpRequestExecution execution) throws IOException {
        final URI originalUri = request.getURI();
        String serviceName = originalUri.getHost();
        Assert.state(serviceName != null,
                "Request URI does not contain a valid hostname: " + originalUri);
        return this.loadBalancer.execute(serviceName,
                this.requestFactory.createRequest(request, body, execution));
    }
```

最后 execute 就用到了 RibbonLoadBalancerClient 下的 execute 方法，具体分析见上文，一直到使用 apply 方法，就调用了 LoadBalancerRequestFactory 的 createRequest 里的方法，这里的具体执行方式，上文也进行过了分析。

```java
return instance -> {
            HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance,
                    this.loadBalancer);
            if (this.transformers != null) {
                for (LoadBalancerRequestTransformer transformer : this.transformers) {
                    serviceRequest = transformer.transformRequest(serviceRequest,
                            instance);
                }
            }
            return execution.execute(serviceRequest, body);
        };
```
