插件机制

1、需要配置的插件

debezium

```
plugin.class=xxx.xxx.Clazz
```





2、自动配置的插件

SPI

读取jar包`META-INF/service`下的接口文件，文件名为接口，文件内容为实现类

SpringBoot

读取jar包`META-INF/spring.factories`文件内的配置，key为className(接口或注解类)，value为对应的插件类列表