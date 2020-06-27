---
title: SpringBoot启动引导之自动注入
date: 2020-06-27 20:20:50
tags:
    - SpringBoot
    - 源码

categories:
  - ['SpringBoot','源码']
---

### SpringBoot版本

```
2.2.5.RELEASE
```

### 启动注解

#### @SpringBootApplication 

一个组合注解，相当于下面三个

- @SpringBootConfiguration （@Configuration）
- @EnableAutoConfiguration
- @ComponentScan 指定包的扫描路径

#### @ComponentScan 

```java
@ComponentScan(
    excludeFilters = {@Filter(
    type = FilterType.CUSTOM,
    classes = {TypeExcludeFilter.class}
), @Filter(
    type = FilterType.CUSTOM,
    classes = {AutoConfigurationExcludeFilter.class}
)}
)
```

这个注解用于指定包的扫描路径。

- TypeExcludeFilter 作用是做**扩展的组件过滤**
- AutoConfigurationExcludeFilter 跟自动配置有关

#### Spring中Bean的装配方式

- 使用模式注解 `@Component` 等（Spring2.5+）
- 使用配置类 `@Configuration` 与 `@Bean` （Spring3.0+）
- 使用模块装配 `@EnableXXX` 与 `@Import` （Spring3.1+）



##### @Import的四种装配方式

EnableXXX注解主要与@Import配合使用。

假设我们要把Red,Yellow,Blue,Green这几个类分别用 普通类、@Configuration，ImportSelector，ImportBeanDefinitionRegistrar这四种方式注入到Spring的容器中。

- 首先我们定义四个颜色类

```java
public class Red {
    public String getColor(){
        return "I am red";
    }
}
```

```java
public class Yellow {
    public String getColor(){
        return "I am yellow";
    }
}
```

```java
public class Blue {
    public String getColor(){
        return "I am blue";
    }
}
```

```java
public class Green {
    public String getColor(){
        return "I am Green";
    }
}
```

###### 第一种：普通类

首先定义一个注解

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Import({Red.class}) //注意此处
public @interface EnableColor {
}
```

然后定义一个配置类，这个类作用就是使用EnableColor注解，也可以直接把EnableColor添加在SpringBoot的启动类上，但是这样会导致注入很多不在我们目前观察范围内的类。

```java
@Configuration
@EnableColor
public class ColorRegistrarConfiguration {
}
```

然后定义一个测试类

```java
public class MainApp {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(ColorRegistrarConfiguration.class);
        String[] beanDefinitionNames =  ctx.getBeanDefinitionNames();
        for (String beanDefinitionName : beanDefinitionNames) {
            System.out.println("=======> "+beanDefinitionName);
        }

    }
}
```

程序输出:

```java
=======> org.springframework.context.annotation.internalConfigurationAnnotationProcessor
=======> org.springframework.context.annotation.internalAutowiredAnnotationProcessor
=======> org.springframework.context.annotation.internalCommonAnnotationProcessor
=======> org.springframework.context.event.internalEventListenerProcessor
=======> org.springframework.context.event.internalEventListenerFactory
=======> colorRegistrarConfiguration
=======> com.eqshen.springprobe.bean.Red
```



###### 第二种：Configuration方式

使用configuration方式定义注入Yellow类实例

```java
@Configuration
public class ColorConfig {
    @Bean
    public Yellow yellow(){
        return new Yellow();
    }
}
```

然后修改EnableColor注解

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Import({Red.class,ColorConfig.class}) //注意此处
public @interface EnableColor {
}
```

再次执行上面的`MainApp#main`方法，输出内容：

```java
=======> org.springframework.context.annotation.internalConfigurationAnnotationProcessor
=======> org.springframework.context.annotation.internalAutowiredAnnotationProcessor
=======> org.springframework.context.annotation.internalCommonAnnotationProcessor
=======> org.springframework.context.event.internalEventListenerProcessor
=======> org.springframework.context.event.internalEventListenerFactory
=======> colorRegistrarConfiguration
=======> com.eqshen.springprobe.bean.Red
=======> com.eqshen.springprobe.config.ColorConfig
=======> yellow
```



###### 第三种： ImportSelector方式

定义`ColorImportSelector`类，继承自 `ImportSelector`，ImportSelector接口只定义了一个`selectImports()`，用于指定需要注册为bean的Class名称。

```java
public class ColorImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[]{Blue.class.getName()};
    }
}
```

然后修改EnableColor注解上的Import注解

```java
@Import({Red.class,ColorConfig.class,ColorImportSelector.class})
```

再次执行MainApp#main方法，得到输出

```java
=======> org.springframework.context.annotation.internalConfigurationAnnotationProcessor
=======> org.springframework.context.annotation.internalAutowiredAnnotationProcessor
=======> org.springframework.context.annotation.internalCommonAnnotationProcessor
=======> org.springframework.context.event.internalEventListenerProcessor
=======> org.springframework.context.event.internalEventListenerFactory
=======> colorRegistrarConfiguration
=======> com.eqshen.springprobe.bean.Red
=======> com.eqshen.springprobe.config.ColorConfig
=======> yellow
=======> com.eqshen.springprobe.bean.Blue
```



###### 第四种 ImportBeanDefinitionRegistrar方式

定义一个类`ColorImportBeanDefinitionRegistrar`并继承 `ImportBeanDefinitionRegistrar`。

```java
public class ColorImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        //指定注册bean名称
        registry.registerBeanDefinition("green", new RootBeanDefinition(Green.class));
    }
}
```

然后修改EnableColor注解上的Import注解

```java
@Import({Red.class,ColorConfig.class,ColorImportSelector.class,ColorImportBeanDefinitionRegistrar.class})
```

再次执行MainApp#main方法，得到输出

```java
=======> org.springframework.context.annotation.internalConfigurationAnnotationProcessor
=======> org.springframework.context.annotation.internalAutowiredAnnotationProcessor
=======> org.springframework.context.annotation.internalCommonAnnotationProcessor
=======> org.springframework.context.event.internalEventListenerProcessor
=======> org.springframework.context.event.internalEventListenerFactory
=======> colorRegistrarConfiguration
=======> com.eqshen.springprobe.bean.Red
=======> com.eqshen.springprobe.config.ColorConfig
=======> yellow
=======> com.eqshen.springprobe.bean.Blue
=======> green
```

###### 四种方式区别及应用场景

```java
@Import({
  Red.class,
  ColorConfig.class,
  ColorImportSelector.class,
  ColorImportBeanDefinitionRegistrar.class})
```

待定（稍后补充）

#### SpringBoot的自动装配

SpringBoot的自动配置完全由 `@EnableAutoConfiguration` 开启。`@EnableAutoConfiguration`是`SpringBootApplication`上的注解，其内容如下：

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import({AutoConfigurationImportSelector.class})
public @interface EnableAutoConfiguration {
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    Class<?>[] exclude() default {};

    String[] excludeName() default {};
}
```



##### @AutoConfigurationPackage

自动扫描将从标注此注解的类所在的包开始，其内容如下：

```java
@Import({Registrar.class})
public @interface AutoConfigurationPackage {
}
```

那这个`Registrar.class`作用是啥呢？它是`AutoConfigurationPackages.java`的一个静态内部类：

```java
/**
	 * {@link ImportBeanDefinitionRegistrar} to store the base package from the importing
	 * configuration.
	 */
	static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {

		@Override
		public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
			register(registry, new PackageImport(metadata).getPackageName());
		}

		@Override
		public Set<Object> determineImports(AnnotationMetadata metadata) {
			return Collections.singleton(new PackageImport(metadata));
		}

	}
```

从注释说明来看，这个Registrar的功能主要就是用来保存 base package的name以便后续使用。`AnnotationMetadata metadata`就是根目录的元数据信息，这里暂时不深入。

basePackage的作用：除了Spring自己使用，也可以提供给三方插件，如Mybatis等。



##### @Import(AutoConfigurationImportSelector.class)

ImportSelector我们之前有提到过，如`ColorImportSelector`，看一下`AutoConfigurationImportSelector.class`关键源码：

```java
@Override
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
		}
		AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
				.loadMetadata(this.beanClassLoader);
		AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(autoConfigurationMetadata,
				annotationMetadata);
		return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
	}
```

函数的作用是根据输入`annotationMetadata`，输出需要被import的类名数组。其中，`loadMetadata`方法是把`META-INF/spring.factories`配置文件中的内容加载封装成 AutoConfigurationMetadata。

spring.factories中配置如下（格式）：keyName=a,b,c

```
# Auto Configuration Import Filters
org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
org.springframework.boot.autoconfigure.condition.OnBeanCondition,\
org.springframework.boot.autoconfigure.condition.OnClassCondition,\
org.springframework.boot.autoconfigure.condition.OnWebApplicationCondition

# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
```

从上面的 Properties 中发现，所有配置的 EnableAutoConfiguration 的自动配置类，都**以 AutoConfiguration 结尾**！由此规律，以后我们要了解一个 SpringBoot 的模块或者第三方集成的模块时，就可以**大胆猜测基本上一定会有 XXXAutoConfiguration 类出现**！


