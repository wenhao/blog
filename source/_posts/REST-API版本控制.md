title: REST API版本控制
toc: true
date: 2015-11-02 15:21:59
categories: [restful]
tags:
    - restful
    - API版本控制
---

API版本控制多用在产品发布之后需要根据新需求对部分API做出相应的调整，是在系统维护过程中比较困难但势在必行的任务。不管采用何种版本控制策略，引入新的API版本都会带来一定量的维护成本。最好的API版本化，就是没有明显的版本。在对已发布的服务进行变更时，要尽量做到兼容，其中包括URI、链接和各种不同的表述的兼容，最关键的就是在扩展时不能破坏现有的客户端。

###API修改场景

1. **[向后兼容]**如果只是增加内容，那么放心地将它们增加到表示即可。因为客户端将忽略那些它们并不理解的东西。**成本较低，只需要修改服务端。**
2. **[向前兼容]**如果需要修改当前契约造成不兼容，那么使用新的API版本。**成本较高，服务端和客户端都需要修改，并且API每个版本都需要维护。**
3. **[新API需求]**如果要对表示做出重大改变，或是改变底层资源的含义，那么使用新名字（URI）创建一份新的资源。服务端和客户都都需要修改，但是不会有兼容性问题。

###何时添加版本控制

在系统初次发布之后没有任何修改已存在API的需求之前是不需要添加API版本控制的。默认情况下客户端会保持调用原有的API。如果有API版本升级的需求，需要客户端和服务端同时修改使用新的API，而且旧的API还需要保留以便老版本的客户端继续使用。

###API版本管理的类别

1. 所有API消费者都连接到API的同一个版本，当API变化时，所有消费者都需要跟着变，实际上，这产生了一个横跨整个消费者集合/生态系统的巨大连锁反应。

2. 服务的每个版本都留在生产环境中运行，当需要某个版本时，消费者需要自己进行迁移。运维成本会随着生产环境中版本数量的增加而增加。

3. 所有客户端都与API/服务的同一兼容性版本进行对话。

对比：当API发生变化时，单一版本会迫使每个消费者都进行升级，对于生态系统而言，这是一种成本最高的方法。第二种方法需要维护版本的多样性，这样会好一些，但如果开发人员试图保持每个版本的升级或者交替运行旧版本，那么成本还是相当高。兼容性版本管理策略似乎最高效，但是开发一个兼容任何时期需求的API相当困难。综上所述，更倾向于第二种方式。


###实现策略

####服务端

使用[Vendor MIME Media Type](http://tools.ietf.org/html/rfc4288#section-3.2), 版本升级采用数字递增不需要使用[Semantic Versioning](http://semver.org/)策略，向后兼容版本采用原有版本即可省去对客户端的修改，向前兼容的版本数字版本加1.

```
Accept: application/vnd.name.v1+json
```

####客户端

在需要添加版本的API服务调用的地方加上：

```
Accept: application/vnd.name.v1+json
```

**强调一点：能不加版本控制的API最好不加，最大限度的降低维护成本。**


###简单示例

Jersey:

```java
@Resource
@Path("/some")
public class SomeResource {

    @GET
    @Path("/{id}")
    @Produces("application/vnd.name.v1+json")
    public someMethodV1 get(@PathParam("id") int id) {
        //......
    }
    
    @GET
    @Path("/{id}")
    @Produces("application/vnd.name.v2+json")
    public someMethodV2 get(@PathParam("id") int id) {
        //......
    }
}
```

Spring:

```java
@RestController
@RequestMapping("/some")
public class SomeResource {

    @RequestMapping(method = RequestMethod.GET, produces="application/vnd.name.v1+json")
    public someMethodV1 get(@PathParam("id") int id) {
        //......
    }
    
    @RequestMapping(method = RequestMethod.GET, produces="application/vnd.name.v2+json")
    public someMethodV2 get(@PathParam("id") int id) {
        //......
    }
}
```

###其他方案

有些系统也采用其他的API版本控制机制，比如URL版本控制和HEADER版本控制，但其维护成本都一样只是表现形式不一样而已。至于为什么不采用上面两种方式，主要原因就是上面两种方式原则上违反[HATEOAS](https://en.wikipedia.org/wiki/HATEOAS)定义。
