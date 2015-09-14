---
layout: post
title: "JBoss AS7源码分析"
date: 2014-03-26
comments: true
categories:
- JBoss
---

High Level Overview
--------------------

Extensions <-- jboss-msc(Modular Service Container) <-- jboss-modules(modular classloading)

管理方式：

* cli
* admin console

AS Core
--------

1. A modular classloading system(jboss-modules.jar)
2. service container(modules/system/layers/base/org/jboss/msc/main/jboss-msc-1.0.4.GA.jar)
3. extensible management layer。这个管理层间接的来访问service container来增加，修改，删除service。它包含：
	* core elements(jboss-dmr, controller, controller-client, deployment-repository, domain-management, network, server module的一部分)
	* 通过protocol, domain-http来实现远程管理的能力
	* 通过process-controller, host-controller来实现多实例的domain能力
	* 通过management-client-content, platform-mbean来实现其他的一些能力
4. deployment framework(server module的一部分)


AS Extensions
--------------

End user平时使用的功能都是一个个extension提供的。AS7的codebase中的modules大部分都是一个个extension。其中很多extention就是针对JEE规范中的某一个方面的。
extension通过实现接口 (org.jboss.as.controller.Extension) 来与AS Core集成。extension可以做以下事情：

1. 参与解析AS的配置文件（standalone.xml）中的一小段
2. 注册resources和operations,通过AS's management API暴露出去（这样，cli就可以有地方操作了）
3. 把service安装到service container
4. 注册deployment unit processors到deployment framework



AS7 Boot Process
----------------

1. bin/standalone.sh
2. java -jar jboss-modules.jar -mp modules org.jboss.as.standalone
	* java进程启动的Main函数是在jboss-modules.jar中，也就是启动了modular classloading environment，然后解析-mp参数，读取到org.jboss.as.standalone，就会去找modules/org/jboss/as/standalone/main/module.xml文件，读取其中的main-class的值，即org.jboss.as.server.Main，于是org.jboss.as.server.Main的main函数被调用了。
3. org.jboss.as.server.Main.main() 做了很多事情，最重要的是：
	* 命令行后面的参数都被处理，并放入ServerEnvironment，随后通过ServerEnvironmentService这个service给其他服务通过getValue()方法调用，这个对象包括了：
		* system properties
		* where the root of the AS dist is
		* where the root of the server instance is
		* where the configuration file is
		* ...
	* org.jboss.as.server.BootstrapImpl被创建，ServerEnvironment被传入，并调用bootstrap()方法。BootstrapImpl做的最重要的事情就是创建ServiceContainer的实例，实例里有个executor属性，是线程池，里面的线程用来并发处理service的生命周期的。executor是个ContainerExecutor extends ThreadPoolExecutor，用的queue是LinkedBlockingQueue。

        ```java
        private final ServiceContainer container = ServiceContainer.Factory.create("jboss-as", MAX_THREADS, 30, TimeUnit.SECONDS);
        ```

        ```java
        ContainerExecutor(final int corePoolSize, final int maximumPoolSize, final long keepAliveTime, final TimeUnit unit) {
            super(corePoolSize, maximumPoolSize, keepAliveTime, unit, new LinkedBlockingQueue<Runnable>(), new ThreadFactory() {
                private final int id = executorSeq.getAndIncrement();
                private final AtomicInteger threadSeq = new AtomicInteger(1);
                public Thread newThread(final Runnable r) {
                    Thread thread = new ServiceThread(r, ServiceContainerImpl.this);
                    thread.setName(String.format("MSC service thread %d-%d", Integer.valueOf(id), Integer.valueOf(threadSeq.getAndIncrement())));
                    thread.setUncaughtExceptionHandler(HANDLER);
                    return thread;
                }
            }, POLICY);
        }
        ```
			1. ServiceContainer管理一堆运行中的service。一个service只是一个可start和stop的对象。Service接口还有一个 public T getValue()方法，其中编写一个service时指定的这个T类型，主要目的是给consumer用的，consumer可以通过getValue的形式拿到T
			2. ServiceContainer提供了一堆API来管理service：
				* configuring   ---service之间的依赖关系
				* installing
				* removing
				* triggering start and stop
				* looking up already installed services
			3. ServiceContainer内部维护一个thread pool。services的install，start，stop等动作是多线程的。也就是这些动作都是被包装成了一个个task，交给thread pool去执行，这样就达到了service处理的并发高效。
			4. 创建ServiceContainer之前的事情都是在jvm的main thread中单线程处理的。直到ServiceContainer被创建出来后，就可以用线程池来处理services了。
4. BootstrapImpl.bootstrap()安装了两个services
	* ControlledProcessStateService
		* 唯一一个一旦启动就不会停止的service。用于让别人访问server的state（STARTING, STOPPING, RUNNING, RESTART\_REQUIRED, RELOAD_REQUIRED）
	* ApplicationServerService
		* 是application server的root service，其他的service都依赖这个服务，当cli中执行 :reload 时，server处理这个请求，然后告诉service container去停止这个ApplicationServerService，带来的效果是所有其他以来ApplicationServerService的service都会先被停掉。一旦ApplicationServerService被停止以后，service container就会再次启动ApplicationServerService。
		* ApplicationServerService的实现启动了很多services，其中最重要的就是ServerService，其他的service包括：
			* ContentRepository     --Repository for deployment content and other managed content,相关关键字：tmp目录，SHA-1算法
			* DeploymentMountProvider --Provides VFS mounts of deployment content
			* ServiceModuleLoader --加载service启动时候需要的module，移除service停止时依赖的module
			* ExternalModuleService  --Service that manages external modules。 ManifestClassPathProcessor在处理deployment时候，如果发现指定了Class-Path，就会调用ExternalModuleService来加载module。
			* ModuleIndexService  --Service that caches the jandex index for system modules
            * ServerService --最重要的，见下面详解。
			* ServerEnvironmentService  --让别的service通过getValue()来获取到ServerEnvironment
			* ServerPathManagerService  --Service containing the paths for a server

5. ServerService负责了 extensible management layer 与 deployment framework
	* ServerService做的事情和某个具体的Extension的initialize方法里做的事情类型是一样的。但对于core AS management model来说，ServerService主要做了：
		* registers resource definitions, attribute definitions and OperationStepHandlers for the core AS managed resources
		* registers core DeploymentUnitProcessors
	* `ServerService implements Service<ModelController>`，ModelController是一个central execution point for management operations in a managed AS process
	* ServerService的super.start方法中创建一个单独的thread，叫作"Controller Boot Thread"。它负责协调剩下的boot过程，这个线程主要干了下面一些事情：
		* 安装ExtensionIndexService  --处理lib/ext目录，循环处理ext目录下每个jar包，然后调用ExternalModuleSpecService和ModuleLoadService（Service that loads and re-links a module once all the modules dependencies are available）
		* 安装DeploymentOverlayIndexService --//TODO
		* registers core DeploymentUnitProcessors
		* 在代码extensibleConfigurationPersister.load()中，把配置文件（standalone.xml）解析成一组management operations，也就是List<ModelNode>， 这些management operations将来会被ModelController执行，xml中的配置也就被加载到运行时中了。其实cli中发送出的operation命令也会被组织成management operations，和xml配置效果一样。
			* 着重见StandaloneXml.java，它描述了standalone.xml里面的情况，然后最终由stax来解析成List<ModelNode>。
			* 当xml解析器解析到 <extension>节点时，会有一些特殊的行为：
				* extension的name属性值就是指向一个JBoss Module，这个module中含有org.jboss.as.controller.Extension的实现。
				* 一旦extension的name被处理到了，那么xml解析器会通知JBoss Modules去加载这个module（代码：ExtensionXml.parseExtensions）
				* module一旦被加载后，ServiceLoader机制就会被用来加载Extension的实现类。
				* Extension的initializeParsers方法被调用（extension可以注册自己的xml namespace对应的xml解析器）
				* 当遇到 <subsystem> ，读取namespace属性，调用相应的xml parser来解析。
		* 把这些management operations传递给ModelController的execute方法，ModelController随后用"Controller Boot Thread"来执行那些operations。
			* xml解析一旦全部ok后，要给ModelController执行的management operations也就准备好了，每个operation都有相同的格式（address, operation name, parameters）。ModelController会以unit形式的执行operations，在每一个step跑遍所有operations，再在另一个step中再一次跑遍operations。执行分3个stages。所有的steps在一个stage全部执行后，再开始执行另一个stage。
				* Stage MODEL: 这个阶段，operation对应的OperationStepHandler对server内部的configuration model做一些必要的update，并且有的也为Stage RUNTIME阶段注册了一些handler。
				* Stage RUNTIME: tage MODEL中的每个OperationStepHandler访问ServiceContainer，来installs/removes/updates any relevant services。ServiceContainer有自己的thread pool来start或stop services。
				* Stage VERIFY: Stage.MODEL or Stage.RUNTIME中的handler可以注册一个 Stage.VERIFY handler，用来检查是否都成功了。
				* 在operations被传递给ModelController执行之前，会检查是否有 add extension resources的操作（对应cli就是/extension=org.foo.extension:add），如果有的话，相应extension的initialize方法会被调用，这就给了extension一个机会去注册自己的resource，attribute定义和OperationStepHandlers。
		* 一直等到所有的services都已经启动好或启动失败
			* 等待ServiceContainer的install，start。注意，所有services的start都是有ServiceContainer内部的thread pool处理的，而不是Controller Boot Thread启动的。ServerService还给每个service的controller对象增加了listener，用来跟踪service的状态，Controller Boot Thread祖塞着判断是否所有的services是否都已经成功启动了。到这个时候，所有的服务，要么启动成功了，要么失败了。
		* 打印boot完成的日志
			* The Controller Boot Thread将ControlledProcessState的状态从STARTING设置为RUNNING，打印一下日志说一下启动成功。
6. 终于启动全部成功！


Deployment Processing
----------------------

当你触发一个deployment时（如cli），负责部署的那段逻辑就会提取相关信息（如deploymeng的名字），然后install services到service container：

* A Service<VirtualFile> implementation that provides an org.jboss.vfs.VirtualFile that represents the deployment content.
* A Service<DeploymentUnit> (in this case RootDeploymentUnitService) that provides a DeploymentUnit for the deployment. A DeploymentUnit retains data which is persistent for the life of the deployment, and will later be passed into the various DeploymentUnitProcessor implementations that perform various actions to install the runtime services needed by the deployment.

RootDeploymentUnitService已经注入了一个引用（引用到所有的DeploymentUnitProcessor (DUP)，这些DUP是在ServerService启动时注册的 ），DUP以部署过程phase来分组，并且每个phase都有一个数字来表示先后顺序，DeploymentUnitProcessors被放在一个chain中，每个DUP仅仅处理有限的内容。

Deployment proceeds in phases。对于每一个phase，代表这个phase的DeploymentUnitPhaseService就会被安装到service container中，每个phase的service在start方法中都会安装下一个phase的service。RootDeploymentUnitService在start方法中安装第一个phase service。phase services被停止的顺序是和启动时是反着的。

phase service在start和stop方法中做最主要的事情就是调用注册在这个phase上的DeploymentUnitProcessor的deploy和undeploy方法。调用DUP的deploy和undeploy方法时，都会顺便给一个DeploymentPhaseContext，用来传递DUP，可以让DUP调用到service container，让他install或者remove services


Deployment Processing and Modules
-----------------------------------

DeploymentUnitProcessors最主要的一个工作就是为deployment创建modular classloading enviroment。每个top-level deployment有一个为它动态生成的module。对于像ear里又包含ejb，war的subdeployment，这些subdeployment也有为他们自己动态生成的module。这些deployment modules之间的依赖关系是deployment framework分析deployment里面的内容时决定的。(e.g. by parsing deployment descriptors, reading manifests, or scanning annotations.)

在这个过程中一个关键的DeploymentUnitProcessor就是org.jboss.as.server.deployment.module.ModuleSpecProcessor。ModuleSpecProcessor配置和安装一个msc service来让 jboss-modules 动态生成module。其他在ModuleSpecProcessor执行之前的DeploymentUnitProcessors是用来分析deployment的内容以及为DeploymentUnit加一些上下文信息，这个DeploymentUnit会被ModuleSpecProcessor用来deployment module之间的依赖关系。

举例：

JPA subsystem注册了一个DUP，这个DUP探测一个deployment中是否含有persistence.xml，并记录module的可见性。最终的效果就是：一个deployment的classes可以访问到AS自己的module的一些classes或者其他一些deployments，但是不能访问其他modules中的class。


Extending AS 7
===============


扩展性方面，AS7 与 之前版本的区别
--------------------

AS6以及之前的版本的内核是一堆deployer，来deploy不同的deployment，有些时jee的实现例如ejb，web，都成了一个deployment，与end user的ear，war平级了。
AS7中的deployment就是指用户的ear，war，jar等，而jboss本身的扩展功能是由extension来实现的。

Extensions and Subsystems
----------------------------

对extension的直观的印象：

1. 继承接口org.jboss.as.controller.Extension
2. 被打包成jar，放到jboss modules能识别的层级目录中
3. 一个extension可以包含一到多个subsystem，一个subsystem是一组特定的功能，比如web容器就是一个subsystem


何时需要自己写一个extension？

* 如果想让自己的扩展的新功能能够让end user通过cli工具来操作。
* 如果想让自己处理一种新的deployment。只要extension才可以将deployment unit processor注册到AS core中
* 方便打补丁。 //TODO
* 如果想把自己的功能与end user的功能区分开（end user的功能是指ear，war等，自己的功能是指从功能性上面扩展AS服务器的能力）
