---
layout: post
title:  "记一次fastjson parsObject问题"
date:  2017-11-01 12:39:04
type: case
categories: [case]
keywords: fastjson,坑
---


## 场景

>最近遇到一次由于java bean字段命名特殊（字段命名为：`__time__`），导致fastjson转java bean的异常问题。

## bug还原

```java
String msg = "{\"globalId\":1036210063725043712,\"name\":\"\",\"__time__\":1509326931}";
Message message = JSONObject.parseObject(msg, Message.class);
System.out.println(message);
```

输出：

```console
Message [globalId=1036210063725043712, giftName=null, __time__=null]
```

Message对象：

```java
public class Message {

	private Long globalId;
	private String giftName;
	private Long __time__;

	public Long getGlobalId() {
		return globalId;
	}

	public void setGlobalId(Long globalId) {
		this.globalId = globalId;
	}

	public String getGiftName() {
		return giftName;
	}

	public void setGiftName(String giftName) {
		this.giftName = giftName;
	}

	public Long get__time__() {
		return __time__;
	}

	public void set__time__(Long __time__) {
		this.__time__ = __time__;
	}
}
```
## debug
debug进去看，发现fastjson会根据set方法的methodName解析出fileName
具体方法为DeserializeBeanInfo.computeSetters
相关代码如下：

```java
 String propertyName;
            if (Character.isUpperCase(c3)) {
                if (TypeUtils.compatibleWithJavaBean) {
                    propertyName = TypeUtils.decapitalize(methodName.substring(3));
                } else {
                    propertyName = Character.toLowerCase(methodName.charAt(3)) + methodName.substring(4);
                }
            } else if (c3 == '_') {
                propertyName = methodName.substring(4);
            } else if (c3 == 'f') {
                propertyName = methodName.substring(3);
            } else if (methodName.length() >= 5 && Character.isUpperCase(methodName.charAt(4))) {
                propertyName = TypeUtils.decapitalize(methodName.substring(3));
            } else {
                continue;
            }

            Field field = TypeUtils.getField(clazz, propertyName);
            if (field == null && method.getParameterTypes()[0] == boolean.class) {
                String isFieldName = "is" + Character.toUpperCase(propertyName.charAt(0)) + propertyName.substring(1);
                field = TypeUtils.getField(clazz, isFieldName);
            }

            if (field != null) {
                JSONField fieldAnnotation = field.getAnnotation(JSONField.class);

                if (fieldAnnotation != null) {
                    ordinal = fieldAnnotation.ordinal();
                    serialzeFeatures = SerializerFeature.of(fieldAnnotation.serialzeFeatures());

                    if (fieldAnnotation.name().length() != 0) {
                        propertyName = fieldAnnotation.name();
                        beanInfo.add(new FieldInfo(propertyName, method, field, clazz, type, ordinal, serialzeFeatures));
                        continue;
                    }
                }

            }
```

这里可以看到对于下划线的处理是

```java
else if (c3 == '_') {
                propertyName = methodName.substring(4);
            }
```
## 解决方案
1）从根本上解决，不要搞这么特殊命名，对于不确定的情况写代码试试。
2）一定要用特殊命名，可以加上@JSONFiled注解，在set方法上面指定具体的字段名。
3）黑科技，修改set方法名
4）黑科技+1，json字符串的key修改

## 总结
这种类型的问题很难规避，多多遇见一些类似的case其实也不错。至少下次知道对于fastjson转java对象或者其他依赖get set方法获取属性的面对特殊情况都要留点心。