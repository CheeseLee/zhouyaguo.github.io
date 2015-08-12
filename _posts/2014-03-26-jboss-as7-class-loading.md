---
layout: post
title: "JBoss AS7的classloader机制"
date: 2014-03-26 23:24
comments: true
categories: 
- JBoss
---

## 术语

### Deployment

部署在AS7中的ear、war等都被称作为`deployment`。


## 简介

JBoss AS7（以下简称AS7）的class loader机制与JBoss之前的版本有很大的不同。AS7的classloading是在[JBoss Modules](https://docs.jboss.org/author/display/MODULES)项目中实现的。与之前的扁平化的双亲委托机制不同，AS7的classloading是基于**module**的，一个module如果想要“看见”另一个module，必须要显式地定义依赖关系。部署在AS7中ear、war也被看作是module，如果ear或者war没有显式地定义对AS7自带的jar的依赖的话，那么是`看不见`容器的jar的。

## deployment的module名称

- war的module name为： deployment.myarchive.war
- ear中的war的module name为： deployment.myear.ear.mywar.war

## 自动依赖

如果让ear必须显式地定义AS7每一个JEE规范的jar的依赖的话，显然是没有必要的，因此AS7在部署一个ear时，会自动为deployment添加上一些依赖，而不用ear手工指定。比如ear中包含persistence.xml，那么AS7会自动为其添加JPA module的依赖，再比如ear中包含@Stateless的ejb时，AS7会自动为其添加ejb module的依赖。

如果ear想exclude掉AS7启动添加的依赖，可以在`META-INF/jboss-deployment-structure.xml`中添加exclude。

## class loading的顺序

在以往的JBoss版本中，当ear中包含了xx-version1.jar的同时，容器也存在xx-version2.jar的话，就会出问题，也就是不同版本的jar不能共存。为了解决这个问题，AS7中定义了class loading的顺序。

class loading的优先级从高到低：

1. AS7自动添加的依赖
2. jboss-deployment-structure.xml中定义的
3. WEB-INF/classes 或者 WEB-INF/lib
4. deployment之间的依赖，比如ear中的war又依赖ear中的ejb

## war的class loading

由于一个war被当作是一个module，所以war中的WEB-INF/lib与WEB-INF/classes是平等的，两者中的class被同一个classloader加载

## ear classloading

假设ear的结构是：


```console
myapp.ear
 |
 |--- web.war
 |
 |--- ejb1.jar
 |
 |--- ejb2.jar
```

AS7默认行为是： 

- web.war能看见ejb1.jar与ejb2.jar
- ejb1.jar与ejb2.jar能互相看见

如果standalone.xml中的ear-subdeployments-isolated设为了true，则web.war、ejb1.jar、ejb2.jar就互相看不见了。

```xml
<subsystem xmlns="urn:jboss:domain:ee:1.0" >
  <ear-subdeployments-isolated>false</ear-subdeployments-isolated>
</subsystem>
```

## global modules

可以在**ee** subsystem中设置所有deployment都依赖的module，例如：

```xml
<subsystem xmlns="urn:jboss:domain:ee:1.0" >            
  <global-modules>
    <module name="org.javassist" slot="main" />            
  </global-modules> 
</subsystem>
```

## jboss-deployment-structure.xml文件

**jboss-deployment-structure.xml** 是JBoss特有的一个文件，用于细粒度地控制class loading，应该放入**META-INF**文件夹中。**jboss-deployment-structure.xml**可以做到：

- exclude掉AS7自动添加的依赖
- 显式地添加对现有module的依赖
- 定义额外的module
- 改变EAR deployments isolated行为


示例如下：（详见xml schema）

```xml
<jboss-deployment-structure xmlns="urn:jboss:deployment-structure:1.2">
  <!-- Make sub deployments isolated by default, so they cannot see each others classes without a Class-Path entry -->
  <ear-subdeployments-isolated>true</ear-subdeployments-isolated>
  <!-- This corresponds to the top level deployment. For a war this is the war's module, for an ear -->
  <!-- This is the top level ear module, which contains all the classes in the EAR's lib folder     -->
  <deployment>
     <!-- exclude-subsystem prevents a subsystems deployment unit processors running on a deployment -->
     <!-- which gives basically the same effect as removing the subsystem, but it only affects single deployment -->
     <exclude-subsystems>
        <subsystem name="resteasy" />
    </exclude-subsystems>
    <!-- Exclusions allow you to prevent the server from automatically adding some dependencies     -->
    <exclusions>
        <module name="org.javassist" />
    </exclusions>
    <!-- This allows you to define additional dependencies, it is the same as using the Dependencies: manifest attribute -->
    <dependencies>
      <module name="deployment.javassist.proxy" />
      <module name="deployment.myjavassist" />
      <!-- Import META-INF/services for ServiceLoader impls as well -->
      <module name="myservicemodule" services="import"/>
    </dependencies>
    <!-- These add additional classes to the module. In this case it is the same as including the jar in the EAR's lib directory -->
    <resources>
      <resource-root path="my-library.jar" />
    </resources>
  </deployment>
  <sub-deployment name="myapp.war">
    <!-- This corresponds to the module for a web deployment -->
    <!-- it can use all the same tags as the <deployment> entry above -->
    <dependencies>
      <!-- Adds a dependency on a ejb jar. This could also be done with a Class-Path entry -->
      <module name="deployment.myear.ear.myejbjar.jar" />
    </dependencies>
    <!-- Set's local resources to have the lowest priority -->
    <!-- If the same class is both in the sub deployment and in another sub deployment that -->
    <!-- is visible to the war, then the Class from the other deployment will be loaded,  -->
    <!-- rather than the class actually packaged in the war. -->
    <!-- This can be used to resolve ClassCastExceptions  if the same class is in multiple sub deployments-->
    <local-last value="true" />
  </sub-deployment>
  <!-- Now we are going to define two additional modules -->
  <!-- This one is a different version of javassist that we have packaged -->
  <module name="deployment.myjavassist" >
    <resources>
     <resource-root path="javassist.jar" >
       <!-- We want to use the servers version of javassist.util.proxy.* so we filter it out-->
       <filter>
         <exclude path="javassist/util/proxy" />
       </filter>
     </resource-root>
    </resources>
  </module>
  <!-- This is a module that re-exports the containers version of javassist.util.proxy -->
  <!-- This means that there is only one version of the Proxy classes defined          -->
  <module name="deployment.javassist.proxy" >
    <dependencies>
      <module name="org.javassist" >
        <imports>
          <include path="javassist/util/proxy" />
          <exclude path="/**" />
        </imports>
      </module>
    </dependencies>
  </module>
</jboss-deployment-structure>
```


Reference：

- [https://docs.jboss.org/author/display/AS72/Class+Loading+in+AS7](https://docs.jboss.org/author/display/AS72/Class+Loading+in+AS7)
