微服务这个话题也算有段时间了，而且并不神秘，很多公司都在走这条路。

一般来说，系统在总是膨胀，最终都是要切分，大系统切为小模块，单一功能横向扩展以支持更大的服务压力，在这样的演化路径下，各组件模块、各子系统之间自然要考虑使用服务化的思路进行协作。

早期的SOA架构爱好者会考虑建立复杂的基础设施，然而传统的ESB等等东西都太重，现在更接地气的做法就是基于REST风格输出服务能力，最后得到一个用http扭结在一起的整体系统，并使用消息队列处理异步化问题，一般的系统大致也就是这样了。

> 有条推是这么解释微服务的，很有趣：
> > @arungupta Microservices = SOA -ESB -SOAP -Centralized governance/persistence -Vendors +REST/HTTP +CI/CD +DevOps +True Polyglot +Containers +PaaS

那么，对微服务来说，Docker又意味着什么？这两种技术之间应该是什么关系呢？
我认为，Docker和微服务的关系应该是——好基友 :-) ，且容慢慢道来。

#### 从软件包管理说起
为了说明问题，我写了[这篇文章](http://www.jianshu.com/p/9dd3eac645e7)讨论软件打包的问题，文章结尾也提到了Docker对软件打包的价值。
