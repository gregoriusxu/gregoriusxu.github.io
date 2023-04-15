---
layout:     post
title:      Spring-IOC介绍
subtitle:   Spring-IOC介绍
date:       2019-07-03
author:     gregorius
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - Java
---

*没有人可以一蹴而就，师傅引进门，修行靠个人，几乎所有的速成文章都是带人入门，很少有人声称可以带你熟练和精通，所以本篇文章也不例外。*

# 装配Bean的方式

Spring具有非常大的灵活性，它提供了三种主要的装配机制：

- 在XML中进行显式配置。
- 在Java中进行显式配置。
- 隐式的bean发现机制和自动装配。

 **Spring的配置风格是可以互相搭配的，所以你可以选择使用XML装配一些bean，使用Spring基于Java的配置（JavaConfig）来装配另一些bean，而将剩余的bean让Spring去自动发现。**

Spring从两个角度来实现自动化装配：

- 组件扫描（component scanning）：Spring会自动发现应用上下文中所创建的bean。
- 自动装配（autowiring）：Spring自动满足bean之间的依赖。

# 组件扫描

##### 定义接口

    package soundsystem;
    public interface CompactDisc {
      void play();
    }

##### 定义实现类

    package soundsystem;
    public class SgtPeppers implements CompactDisc {  
    private String title = "Sgt. Pepper's Lonely Hearts Club Band";   
    private String artist = "The Beatles";    
    public void play() {   
         System.out.println("Playing " + title + " by " + artist); 
    }

##### 启动组件扫描

      package soundsystem;
      import org.springframework.context.annotation.componentScan;
      import org.springframework.context.annotation.Con
      @Configuration
      @ComponentScan
      public class CDPlayerconfig{
      }

##### 通过xml配置启用组件扫描

      <xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmins:Context ="http://www.springframework.org/schema/context"
      http://www.springframework.org/schema/context/spring-context.xsd"
      xsi:schemaLocation="http://www.springframework.org/schema/beans       
      http://www.springframework.org/schema/beans/spring-beans.xsd 
      http://www.springframework.org/schema/context>
      http://www.springframework.org/schema/context/spring-context.xsd">
          <context:component-scan base-package="soundsystem"/>
      </beans>

##### 测试用例

    package soundsystem;
    import static org.junit.Assert.*;
    import org.junit.Test;
    import org.junit.runner.Runwith;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.test.context.ContextConfiguration;
    import org.springframework.test.context.junit4.SpringJunit4ClassRunner;
      @Runwith(SpringUUnit4ClassRunner.class)
      @ContextConfiguration(classes=CDPlayerconfig.class)
      public class CDPlayerTest {
        @Autowired
        private CompactDisc cd;
        @Test
         public void cdshouldNotBeNull(){
          assertNotNull(cd);
        }
     }

##### 指定基础包

    @Configuration
    @ComponentScan(basePackages="soundsystem"}
    public class CDPlayerConfig {}

# 自动装配

##### 在构造器上

    package soundsystem;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Component;
    @Component
    public class cpPlayer implements MediaPlayer {
    private CompactDisc cd;
      @Autowired
      public CDPlayer(CompactDisc cd){
        this. cd=cd;
       }

       public void play(){
          cd. play();
       }
    }

##### 用在方法上

    @Autowired
    public void setCompactDisc(CompactDisc cd){
      this. cd=cd;
    }

##### 通过Java代码装配bean

去掉@ComponentScan,添加@Configuration

    package soundsystem;
    import org.springframework.context.annotation.Conficuration;
    @Configuration
    public class CDPlayerconfig{
    }

##### bean显示装配

    @Bean
    public CompactDisc sgtpeppers(){
        return new SgtPeppers();
    }

##### 组件依赖

    @Bean
     public CDPlayer cdPlayer(){
      return new CDPlayer(sgtPeppers());
     }

# XML装配
##### 基础XML

    <xmlns: xsi="http://www.w3. org/2001/XMLSchema-instance" 
    xsi: schemalocation="http://www.springframework. org/schema/beans
    http://www. springframework. org/schema/beans/spring-beans. xsd     
    http://www. springframework. org/schema/context">

##### 声明一个简单的bean
      <bean class="soundsystem.SgtPeppers"/>

##### 借助构造器注入初始化bean

      <bean id="cdplayer"class="soundsystem.CDPlayer">
          <constructor-arg ref="compactDisc"/>
      </bean>

##### 属性注入

      <bean id="cdPlayer" class="soundsystem.CDPlayer">
          <property name="compactDisc"ref="compactDisc" />
      </bean>
