---
layout: post
title:  "Motan源码解读-Protocol"
date:  2016-09-21
categories: [Motan源码解读]
keywords: motan,源码,Protocol,点对点通讯
---

protocol可以说是motan最重要的部分之一。
暴露一个服务引用一个服务都是在这块实现的。

``` java
@Spi(scope = Scope.SINGLETON)
public interface Protocol {
    /**
     * 暴露服务
     * 
     * @param <T>
     * @param provider
     * @param url
     * @return
     */
    <T> Exporter<T> export(Provider<T> provider, URL url);

    /**
     * 引用服务
     * 
     * @param <T>
     * @param clz
     * @param url
     * @param serviceUrl
     * @return
     */
    <T> Referer<T> refer(Class<T> clz, URL url, URL serviceUrl);

    /**
     * <pre>
	 * 		1） exporter destroy
	 * 		2） referer destroy
	 * </pre>
     * 
     */
    void destroy();
}

```

- Provider

> 服务提供方，通常可以认为是server端

- Exporter

>服务提供Provider暴露出去就变成Exporter，服务使用方（客户端）可以调用暴露出去的Provider

- Referer

> 引用服务，引用的服务会通过配置去调用服务端暴露的服务

现在我们应该会有一定的认识，对于protocol。不管是暴露服务还是引用服务都是通过protocol来实现的。可以说他是学习motan的一个入口。

我们看到这个接口上面也有@SPI注解，就是说protocol可以基于spi机制扩展，`也就说你可以对motan进行为所欲为的扩展`。

## protocol的实现

![](/images/motan/Protocol.png)

这里面再次用到了`模板方法设计模式`。

下面看我们看AbstractProtocol抽象类：

``` java
public abstract class AbstractProtocol implements Protocol {
    // 存放本地export的服务，用于injvm的时候查找本地服务
    protected ConcurrentHashMap<String, Exporter<?>> exporterMap = new ConcurrentHashMap<String, Exporter<?>>();

    public Map<String, Exporter<?>> getExporterMap() {
        return Collections.unmodifiableMap(exporterMap);
    }

    @SuppressWarnings("unchecked")
    @Override
    public <T> Exporter<T> export(Provider<T> provider, URL url) {
        if (url == null) {
            throw new MotanFrameworkException(this.getClass().getSimpleName() + " export Error: url is null",
                    MotanErrorMsgConstant.FRAMEWORK_INIT_ERROR);
        }

        if (provider == null) {
            throw new MotanFrameworkException(this.getClass().getSimpleName() + " export Error: provider is null, url=" + url,
                    MotanErrorMsgConstant.FRAMEWORK_INIT_ERROR);
        }

        String protocolKey = MotanFrameworkUtil.getProtocolKey(url);

        synchronized (exporterMap) {
            Exporter<T> exporter = (Exporter<T>) exporterMap.get(protocolKey);

            if (exporter != null) {
                throw new MotanFrameworkException(this.getClass().getSimpleName() + " export Error: service already exist, url=" + url,
                        MotanErrorMsgConstant.FRAMEWORK_INIT_ERROR);
            }

            exporter = createExporter(provider, url);
            exporter.init();

            exporterMap.put(protocolKey, exporter);

            LoggerUtil.info(this.getClass().getSimpleName() + " export Success: url=" + url);

            return exporter;
        }


    }

    public <T> Referer<T> refer(Class<T> clz, URL url) {
        return refer(clz, url, url);
    }

    @Override
    public <T> Referer<T> refer(Class<T> clz, URL url, URL serviceUrl) {
        if (url == null) {
            throw new MotanFrameworkException(this.getClass().getSimpleName() + " refer Error: url is null",
                    MotanErrorMsgConstant.FRAMEWORK_INIT_ERROR);
        }

        if (clz == null) {
            throw new MotanFrameworkException(this.getClass().getSimpleName() + " refer Error: class is null, url=" + url,
                    MotanErrorMsgConstant.FRAMEWORK_INIT_ERROR);
        }

        Referer<T> referer = createReferer(clz, url, serviceUrl);
        referer.init();

        LoggerUtil.info(this.getClass().getSimpleName() + " refer Success: url=" + url);

        return referer;
    }

    // 用于子类扩展，通过扩展这个方法，可以使服务端暴露出自定义的服务。
    protected abstract <T> Exporter<T> createExporter(Provider<T> provider, URL url);

  // 用于子类扩展，通过扩展这个方法，可以使客户端有自定义的服务引用。
    protected abstract <T> Referer<T> createReferer(Class<T> clz, URL url, URL serviceUrl);

    @Override
    public void destroy() {
        for (String key : exporterMap.keySet()) {
            Node node = exporterMap.remove(key);

            if (node != null) {
                try {
                    node.destroy();

                    LoggerUtil.info(this.getClass().getSimpleName() + " destroy node Success: " + node);
                } catch (Throwable t) {
                    LoggerUtil.error(this.getClass().getSimpleName() + " destroy Error", t);
                }
            }
        }
    }
}
```

如果我们需要实现自己的Export或者Referer，我们继承AbstractProtocol这个方法覆盖里面的两个抽象方法就好了。

比如说，我们希望motan支持protobuf或者thrift，那么我们继承AbstractProtocol这个抽象类，分别实现createExporter，createReferer方法。

那么这两个方法怎么实现呢？

我们来看motan给出的一个默认实现：

``` java
@SpiMeta(name = "motan")
public class DefaultRpcProtocol extends AbstractProtocol {
    // 多个service可能在相同端口进行服务暴露，因此来自同个端口的请求需要进行路由以找到相应的服务，同时不在该端口暴露的服务不应该被找到
    // ProviderMessageRouter负责了服务端消息的路由调用，同时也负责了心跳的逻辑
    private Map<String, ProviderMessageRouter> ipPort2RequestRouter = new HashMap<String, ProviderMessageRouter>();
    @Override
    protected <T> Exporter<T> createExporter(Provider<T> provider, URL url) {
        return new DefaultRpcExporter<T>(provider, url);
    }
    @Override
    protected <T> Referer<T> createReferer(Class<T> clz, URL url, URL serviceUrl) {
        return new DefaultRpcReferer<T>(clz, url, serviceUrl);
    }
    /**
     * rpc provider
     *
     * @param <T>
     * @author maijunsheng
     */
    class DefaultRpcExporter<T> extends AbstractExporter<T> {
        private Server server;
        private EndpointFactory endpointFactory;
        public DefaultRpcExporter(Provider<T> provider, URL url) {
            super(provider, url);
            ProviderMessageRouter requestRouter = initRequestRouter(url);
            endpointFactory =
                    ExtensionLoader.getExtensionLoader(EndpointFactory.class).getExtension(
                            url.getParameter(URLParamType.endpointFactory.getName(), URLParamType.endpointFactory.getValue()));
            //创建server，目前motan仅实现了NettyServer（ExtensionLoader出现代表是可以扩展的，可以扩展自己的server）
            server = endpointFactory.createServer(url, requestRouter);
        }
        @SuppressWarnings("unchecked")
        @Override
        public void unexport() {
            String protocolKey = MotanFrameworkUtil.getProtocolKey(url);
            String ipPort = url.getServerPortStr();
            Exporter<T> exporter = (Exporter<T>) exporterMap.remove(protocolKey);
            if (exporter != null) {
                exporter.destroy();
            }
            synchronized (ipPort2RequestRouter) {
                ProviderMessageRouter requestRouter = ipPort2RequestRouter.get(ipPort);

                if (requestRouter != null) {
                    requestRouter.removeProvider(provider);
                }
            }
            LoggerUtil.info("DefaultRpcExporter unexport Success: url={}", url);
        }

        @Override
        protected boolean doInit() {
           //打开服务，
            boolean result = server.open();
            return result;
        }
        @Override
        public boolean isAvailable() {
            return server.isAvailable();
        }
        @Override
        public void destroy() {
            endpointFactory.safeReleaseResource(server, url);
            LoggerUtil.info("DefaultRpcExporter destory Success: url={}", url);
        }
        private ProviderMessageRouter initRequestRouter(URL url) {
            ProviderMessageRouter requestRouter = null;
            String ipPort = url.getServerPortStr();
            synchronized (ipPort2RequestRouter) {
                requestRouter = ipPort2RequestRouter.get(ipPort);
                if (requestRouter == null) {
                    requestRouter = new ProviderProtectedMessageRouter(provider);
                    ipPort2RequestRouter.put(ipPort, requestRouter);
                } else {
                    requestRouter.addProvider(provider);
                }
            }
            return requestRouter;
        }
    }
    /**
     * rpc referer
     *
     * @param <T>
     * @author maijunsheng
     */
    class DefaultRpcReferer<T> extends AbstractReferer<T> {
        private Client client;
        private EndpointFactory endpointFactory;

        public DefaultRpcReferer(Class<T> clz, URL url, URL serviceUrl) {
            super(clz, url, serviceUrl);

            endpointFactory =
                    ExtensionLoader.getExtensionLoader(EndpointFactory.class).getExtension(
                            url.getParameter(URLParamType.endpointFactory.getName(), URLParamType.endpointFactory.getValue()));
            // 创建client
            client = endpointFactory.createClient(url);
        }
        // 用这个协议的服务，每次调用必经过这个方法
        @Override
        protected Response doCall(Request request) {
            try {
                // 为了能够实现跨group请求，需要使用server端的group。
                request.setAttachment(URLParamType.group.getName(), serviceUrl.getGroup());
                return client.request(request);
            } catch (TransportException exception) {
                throw new MotanServiceException("DefaultRpcReferer call Error: url=" + url.getUri(), exception);
            }
        }
        @Override
        protected void decrActiveCount(Request request, Response response) {
            if (response == null || !(response instanceof Future)) {
                activeRefererCount.decrementAndGet();
                return;
            }
            Future future = (Future) response;
            future.addListener(new FutureListener() {
                @Override
                public void operationComplete(Future future) throws Exception {
                    activeRefererCount.decrementAndGet();
                }
            });
        }
        @Override
        protected boolean doInit() {
            // 打开client
            boolean result = client.open();
            return result;
        }

        @Override
        public boolean isAvailable() {
            return client.isAvailable();
        }
        @Override
        public void destroy() {
            endpointFactory.safeReleaseResource(client, url);
            LoggerUtil.info("DefaultRpcReferer destory client: url={}" + url);
        }
    }
}


```

ProviderMessageRouter：这个类也比较核心，call方法处理了Server消息路由调用的逻辑，handle方法理了心跳的逻辑

----
后记：看到这里，觉得motan真心不错，虽然功能算不上丰富，但是处处留了扩展的入口。所以还在纠结要不要用motan的可以不用纠结了，即便你需要的功能motan没有覆盖到，
但是你多半也可以扩展去改造它。
