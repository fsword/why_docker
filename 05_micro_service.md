微服务这个话题也算有段时间了，而且并不神秘，很多公司都在走这条路。

一般来说，系统在总是膨胀，最终都是要切分，大系统切为小模块，单一功能横向扩展以支持更大的服务压力，在这样的演化路径下，各组件模块、各子系统之间自然要考虑使用服务化的思路进行协作。

早期的SOA架构爱好者会考虑建立复杂的基础设施，然而传统的ESB等等东西都太重，现在更接地气的做法就是基于REST风格输出服务能力，最后得到一个用http扭结在一起的整体系统，并使用消息队列处理异步化问题，一般的系统大致也就是这样了。

有条推是这么解释微服务的，很有趣：
> @arungupta Microservices = SOA -ESB -SOAP -Centralized governance/persistence -Vendors +REST/HTTP +CI/CD +DevOps +True Polyglot +Containers +PaaS

那么，对微服务来说，Docker又意味着什么？这两种技术之间应该是什么关系呢？
我认为，Docker和微服务的关系应该是——好基友 :-) ，且容慢慢道来。

#### 从软件打包说起
我学习c语言的时候是在大学课程上，老实说，能理解那些语言概念就很不容易了，对于打包这件事听都没听说过。但真实情况下，大部分的软件项目都不可能是从零开始的，我们总要依赖某些开源的或者团队自己开发的工具和框架库来帮助工作，我是学习java的时候才慢慢听说了`ant`和`maven`。

`maven`的核心配置是`pom.xml`文件，开发者可以根据需要在其中列出项目的依赖包，像这样：
```
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <version>4.1.5.RELEASE</version>
</dependency>
```
`maven`命令在工作时会找到`spring-core`所依赖的其它库
```
 $ mvn dependency:tree
......
[INFO] sample_java:sample_java:war:1.0-SNAPSHOT
[INFO] +- org.springframework:spring-core:jar:4.1.5.RELEASE:compile
[INFO] |  \- commons-logging:commons-logging:jar:1.2:compile
......
```

但是这样有个问题，某些间接依赖会导致在不同时候打出不同的包，比如上述这样个例子，两次打包期间如果`commons-logging`发布了新版本，那么两次打包的内容就不一样了，如果遇到新版本的差异，开发人员可能会莫名其妙。

ruby社区针对这个问题发明了一个叫做`bundle`的工具，它也有个`Gemfile`文件用来记录直接的依赖库，类似`mvn`，但是`bundle`多了一个功能，工程师可以在当前项目下执行`bundle install` 命令，bundle系统将根据当前的软件仓库状态计算出间接依赖，并将这些间接依赖锁定到某个版本，内容写在 Gemfile.lock 文件中。比如这个简单的例子：
```
source 'https://ruby.taobao.org'
gem 'activesupport'
```
生成的Gemfile.lock是这样：
```
GEM
  remote: https://ruby.taobao.org/
  specs:
    activesupport (4.2.3)
      i18n (~> 0.7)
      json (~> 1.7, >= 1.7.7)
      minitest (~> 5.1)
      thread_safe (~> 0.3, >= 0.3.4)
      tzinfo (~> 1.1)
    i18n (0.7.0)
    json (1.8.3)
    minitest (5.7.0)
    thread_safe (0.3.5)
    tzinfo (1.2.2)
      thread_safe (~> 0.1)

PLATFORMS
  ruby

DEPENDENCIES
  activesupport
```
当再次执行`bundle`命令时，bundle系统会根据`Gemfile.lock`文件来决定间接依赖，所以开发者通常把这个文件放入版本控制系统，确保所有人和线上都用同一份`Gemfile.lock`，就能避免上述的问题。

我一直觉得`bundle`的做法是最先进的，不过和做`nodejs`开发的同学聊天时，了解到了`npm`的做法颇为特别，虽然不见得比`bundle`更好，却是各有优劣。

`npm`的做法是直接把被依赖的库放入当前库的`node_modules`目录，依赖库也以此类推，它的核心文件是`package.json`，比如这个例子：
```
$  cat package.json
{
  "name": "sample_js",
  "version": "1.0.0",
  "dependencies": {
    "browserify": "10.2.4"
  }
}
$ npm install
$ npm dedupe
```
查看一下依赖库
```
$  ls node_modules/browserify/node_modules 
JSONStream             concat-stream          glob                   labeled-stream-splicer readable-stream        syntax-error
acorn                  console-browserify     has                    module-deps            readable-wrap          through2
assert                 constants-browserify   htmlescape             os-browserify          resolve                timers-browserify
browser-pack           crypto-browserify      http-browserify        parents                sha.js                 tty-browserify
browser-resolve        defined                https-browserify       path-browserify        shasum                 url
browserify-zlib        deps-sort              indexof                process                shell-quote            util
buffer                 domain-browser         inherits               punycode               stream-browserify      vm-browserify
builtins               duplexer2              insert-module-globals  querystring-es3        string_decoder         xtend
commondir              events                 isarray                read-only-stream       subarg
```
查看一下依赖库的依赖库
```
$ ls node_modules/browserify/node_modules/crypto-browserify/node_modules
bn.js           browserify-aes  browserify-sign create-hash     diffie-hellman  parse-asn1      public-encrypt
brorand         browserify-rsa  create-ecdh     create-hmac     elliptic        pbkdf2          randombytes
```
这种做法实际上是在开发环节就确定并下载了间接依赖的库，可以看做在开发者手里就完成了打包，这么做有什么好处？

相比`maven`，`npm`和`bundle`更具备一致性，无论到哪里，`bundle`系统都保证使用lock版本，不会有“失控”的依赖库；而相对于`bundle`，`npm`可以允许在一个项目中依赖同一个库的不同版本，这比较灵活。

但是这么做也有缺点，有些js开发者就吐槽这一点，认为浪费了内存——不同库依赖同一个库时，都会在自己的node_modules目录下存放一份被依赖库的代码。从这个角度看，`bundle`又显得有些优势，因为`require`在同一个ruby进程中是有缓存的，不会额外浪费内存。

打个比方吧，`bundle`就好像大规模的军事单位，除了作战部队外还有专门的，好处是可以统一划拨管理，降低了维护成本，缺点是有些细微的差别不好满足。
而`npm`就好像一个精干的小分队，每个人带自己适合的食物，虽然可能并不丰富，但是每个单兵都是一个可以独立生存的单元。

为了简洁，我把`npm`这种方式称为“自带干粮”。软件技术中，“自带干粮”的设计思想有很多应用场景。

> * **静态编译 vs. 动态编译**
如果你喜欢自己从源码编译软件，那么多半见过这样的命令：
```
/opt/apache_src $ ./configure --prefix=/usr/ --enable-file-cache
```
其中的`enable-file-cache`是静态编译的选项，使用这类选项，编译工具会将模块直接编译进入最终的执行文件（比如`apache`的执行文件就是`httpd`）。
与之相比，另外一种常见的做法是把模块代码编译为 so 动态链接库，然后在运行时动态载入
