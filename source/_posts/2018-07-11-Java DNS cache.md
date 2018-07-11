---
layout: post
title:  "Java DNS Cache小记"
date:  2018-07-11 12:39:04
type: test
categories: [札记]
keywords: java,dns,cache
---

## 写在前面

默认缓存时间30s

### 缓存策略：
- 永不过期
- 禁用缓存
- 设置缓存时间ttl

## Java DNS Cache缓存源代码 InetAddress


```java
private static void cacheAddresses(String hostname,
                                   InetAddress[] addresses,
                                   boolean success) {
    hostname = hostname.toLowerCase();
    synchronized (addressCache) {
        cacheInitIfNeeded();
        if (success) {
            addressCache.put(hostname, addresses);
        } else {
            negativeCache.put(hostname, addresses);
        }
    }
}

public Cache put(String host, InetAddress[] addresses) {
	int policy = getPolicy();
	if (policy == InetAddressCachePolicy.NEVER) {//永不缓存，dns解析慢可能会影响性能
	    return this;
	}

	// purge any expired entries

	if (policy != InetAddressCachePolicy.FOREVER) {//永不过期，这种dns变更了需要重启服务

	    // As we iterate in insertion order we can
	    // terminate when a non-expired entry is found.
	    LinkedList<String> expired = new LinkedList<>();
	    long now = System.currentTimeMillis();
	    for (String key : cache.keySet()) {
	        CacheEntry entry = cache.get(key);

	        if (entry.expiration >= 0 && entry.expiration < now) {
	            expired.add(key);
	        } else {
	            break;
	        }
	    }

	    for (String key : expired) {
	        cache.remove(key);
	    }
}


```

## 修改方式

1.	jvm启动参数里面配置-Dsun.net.inetaddr.ttl=value

2.	修改 配置文件JAVA_HOME/jre/lib/security/java.security相应的参数networkaddress.cache.ttl=value

3.	代码里直接设置：java.security.Security.setProperty(”networkaddress.cache.ttl” , “value”);

## 补充

关于java dns的一个开源项目
https://github.com/alibaba/java-dns-cache-manipulator/blob/master/library/README.md

>设置/重置DNS（不会再去Lookup DNS  
>可以设置单条  
>或是通过Properties文件批量设置  
>清空DNS Cache（即所有的域名重新Lookup DNS）  
>删除一条DNS Cache（即重新Lookup DNS）  
>查看DNS Cache内容  
>设置/查看JVM缺省的DNS的缓存时间  
 
 