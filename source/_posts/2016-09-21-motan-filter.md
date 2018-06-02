---
layout: post
title:  "Motan源码解读-Filter"
date:  2016-09-21
categories: [Motan源码解读]
keywords: motan,源码,filter
---

很多服务都有filter这个概念，filter就像在管道前的过滤器，只要通过管道的请求必须先经过过滤器。
过滤器可以做很多事情，类似权限控制，限流，统计，上报，监控等等。有些为服务架构会把filter单独抽出来作为getway服务。

## motan filter：

### 目前实现的filter

- AccessLogFilter

>统计整个call的执行状况，尽量到最上层，最后执行.
> 此filter会对性能产生一定影响，请求量较大时建议关闭。

- AccessStatisticFilter

>

- ActiveLimitFilter

> limit active count，判断某个接口并发数是否超限，如果超过限制，则上抛异常,同时做简单的统计。 此filter比较严格，尽量放到底层较早执行。

- ServiceMockFilter

>mock serivce。测试场景使用

- SwitcherFilter

>服务降级的filter，



###  按使用的地方划分

- **服务端的filter**
- **客户端的filter**

motan的服务端和客户端都可以添加filter，可以是相同的filter也可以是不同的filter

### 代码分析
下面我们来看filter的ProtocolFilterDecorator 代码(代码仅保留了分析相关部分)：

``` java
public class ProtocolFilterDecorator implements Protocol {
    ...
    private <T> Referer<T> decorateWithFilter(Referer<T> referer, URL url) {
        List<Filter> filters = getFilters(url, MotanConstants.NODE_TYPE_REFERER);
        Referer<T> lastRef = referer;
        for (Filter filter : filters) {
            final Filter f = filter;
            final Referer<T> lf = lastRef;
            lastRef = new Referer<T>() {
                @Override
                public Response call(Request request) {
	                // 这里可以通过配置来控制，对于重试的请求要不要经过filter，比如有些安全校验的，第一次已经校验通过，重试自然不用在校验一次，但是对于有些统计来说，可能会认为不管是不是重试，都需要统计，这里通过配置把这种情况暴露给使用者，把决定权留给使用方
                    if (!f.getClass().getAnnotation(Activation.class).retry() && request.getRetries() != 0) {
                        return lf.call(request);
                    }
                    return f.filter(lf, request);
                }

               ...
            };
        }
        return lastRef;
    }

    private <T> Provider<T> decorateWithFilter(Provider<T> provider, URL url) {
        List<Filter> filters = getFilters(url, MotanConstants.NODE_TYPE_SERVICE);
        if (filters == null || filters.size() == 0) {
            return provider;
        }
        Provider<T> lastProvider = provider;
        for (Filter filter : filters) {
            final Filter f = filter;
            final Provider<T> lp = lastProvider;
            lastProvider = new Provider<T>() {
                @Override
                public Response call(Request request) {
                    return f.filter(lp, request);
                }

                ...
            };
        }
        return lastProvider;
    }

    
}

```

从这个类的名字我们可以可以猜测motan的filter是基于java的`装饰模式`了，关于java的装饰者模式，（最经典的是jdk的IO部分的用法）

>**装饰模式：**在不必改变原类文件和使用继承的情况下，动态地扩展一个对象的功能。它是通过创建一个包装对象，也就是装饰来包裹真实的对象。

这个类中包装了两个filter，一个是把客户端的referer decorateWithFilter，另一个是把服务端的Provider decorateWithFilter。



## 小结

对于filter，作用大多是过滤，一般越早越好，如果不满足过滤条件，能够早些丢弃，不会浪费资源对那些不满足filter过滤条件的请求做其他的处理。

所以一般encode&decode完成的请求会立即进到filter过滤。

---
## 后记：
TODO后面会补充



