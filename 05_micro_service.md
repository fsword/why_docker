微服务这个话题也算有段时间了，而且并不神秘，很多公司都在走这条路。

一般来说，系统在总是膨胀，最终都是要切分，大系统切为小模块，单一功能横向扩展以支持更大的服务压力，在这样的演化路径下，各组件模块、各子系统之间自然要考虑使用服务化的思路进行协作。

早期的SOA架构爱好者会考虑建立复杂的基础设施，然而传统的ESB等等东西都太重，现在更接地气的做法就是基于REST风格输出服务能力，最后得到一个用http扭结在一起的整体系统，并使用消息队列处理异步化问题，一般的系统大致也就是这样了。

有条推是这么解释微服务的，很有趣：
> @arungupta Microservices = SOA -ESB -SOAP -Centralized governance/persistence -Vendors +REST/HTTP +CI/CD +DevOps +True Polyglot +Containers +PaaS

那么，对微服务来说，Docker又意味着什么？这两种技术之间应该是什么关系呢？
我认为，Docker和微服务的关系应该是——好基友 :-) ，且容慢慢道来。

#### 从软件包管理说起
我学习c语言的时候是在大学课程上，老实说，能理解那些语言概念就很不容易了，对于软件包管理这件事听都没听说过。但真实情况下，大部分的软件项目都不可能是从零开始的，我们总要依赖某些开源的或者团队自己开发的工具和框架库来帮助工作，我是学习java的时候才慢慢听说了`maven`。

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

我把`npm`这种方式称为“自带干粮”。软件技术中，“自带干粮”的设计思想有很多应用场景，比如**动态编译**和**静态编译**。

如果你喜欢自己从源码编译软件，那么多半熟悉LD_LIBRARY_PATH这个环境变量，这是用来指明动态链接库查找路径的，我们常常把一些模块代码编译为后缀为 `so` 的动态链接库文件，然后再运行时动态载入。

与之相比，还有一种做法叫静态编译，比如这样的命令：
```
/opt/apache_src $ ./configure --prefix=/usr/ --enable-file-cache
```
其中的`enable-file-cache`是静态编译的选项，使用这类选项，编译工具会将模块直接编译进入最终的执行文件（比如`apache`的执行文件就是`httpd`）。

使用动态链接库还是选择静态编译？应该说两种方式各有优劣，前者可以减少内存消耗，避免装入暂时用不到的代码，而后者则是某种角度的“自带干粮”，这样编译的可执行程序，迁移起来比较容易，不会由于目标系统上没有相应的动态链接库而运行失败。

Go语言是 google 创建的一门语言，它有很多独特的设计，其中之一就是——它是静态编译的，这一点曾经让很多人诟病，认为它写一个`hello world`都要输出很大的可执行文件。

但是从另外一个角度看这个做法很有价值，Go语言的定位是系统级编程，这类程序和应用软件不同，它往往比较底层，本身就是其它软件的基础，因此对稳定性很重视，除了硬件这种不得不考虑的因素，其它方面干扰越少越好，使用动态链接库会导致两个结果：
* 降低迁移成本：在一个linux里编译好的Go程序，通常可以直接copy到其它linux上使用，而如果依赖动态链接库，那么可能由于目标系统上缺乏动态链接库而失败，这样，Go程序的安装复杂性会很高，这对基础的系统软件是不利的。
* 外部错误干扰bug定位：即使目标系统上有所需的动态链接库，也不一定就没有问题，由于版本依赖等问题，目标系统的动态链接库未必和本来设想的相同，如果由于外部错误导致bug，而开发人员并不知道这一情况，对bug定位无疑是一个灾难。

Go语言具备这些特点并不是偶然的，作为大规模集群计算起家的互联网公司，Google对于系统的横向扩展能力、可靠性、故障恢复能力都有很高的要求，自带干粮的语言可以很好的帮助实现这些目标：

* 横向扩展能力：这个和迁移成本相关，由于不再需要动态链接库的配合，应用程序可以很方便完整的在新系统上部署，因此，一旦需要横向扩展，至少在软件安装方面，Go语言就有了很大的便利性。
* 可靠性：Google的哲学是，通过软件而不是硬件提供可靠性。那么，如果硬件的可靠性不可依赖，软件系统的可靠性和容错就要通过类似备份之类的冗余计算来得到，这时，一个“自带干粮”的应用程序显然很容易部署为幂等的集群，因此可以在很大程度上帮助实现冗余计算。
* 故障恢复能力：这将受益于对外部干扰的排除，由于“自带干粮”，Go程序已经包含了bug分析的几乎全部信息，开发人员不需要依靠线上环境，在线下就能分析界定问题。

这样看来，“自带干粮”的做法其实非常适合互联网应用的场景，这里充斥着“集群”、“弹性扩展”、“服务幂等化”的做法，如果我们从事互联网应用领域，那么理解“自带干粮”的做法很有价值。

这个做法其实并不神秘，“自带干粮”的理念的一个重要体现，就是我们很熟悉的“打包”环节。

> linux服务端开发的项目，通常都会有一个“打包”环节，很多人并不完全理解这个环节的作用，实际上，这只是“自带干粮”原则在项目管理中的落实而已。

举两个例子，ruby bundle的打包是这样的：
```
$ bundle package
...
$ ls -l vendor/cache
total 1744
-rw-r--r--  1 john  staff   322K  7  9 03:48 activesupport-4.2.3.gem
-rw-r--r--  1 john  staff    57K  7  9 03:48 i18n-0.7.0.gem
-rw-r--r--  1 john  staff   149K  7  9 03:48 json-1.8.3.gem
-rw-r--r--  1 john  staff    70K  7  9 03:48 minitest-5.7.0.gem
-rw-r--r--  1 john  staff   118K  7  9 03:48 thread_safe-0.3.5.gem
-rw-r--r--  1 john  staff   144K  7  9 03:48 tzinfo-1.2.2.gem
```

bundle packge命令把需要使用的gem包统统放入vendor/cache目录，那么当前目录就是一个“自带干粮”的体系

再看java

```
$ mvn pacakge
...
$ ls -l target/*.war
-rw-r--r--  1 john  staff   1.0M  7  9 03:51 target/sample_java.war
```
mvn命令最后会输出一个war文件，按照javaEE的相关规范，它自身包含了所有依赖的第三方jar，所以这也是一个“自带干粮”的产出。

