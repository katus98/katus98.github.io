---
title: Mybatis的Spring插件扫描原理
author: katus
date: 2022-09-12 11:16:17 +0800
categories: [Spring]
tags: [Java, Spring, Mybatis]
typora-root-url: ..
---

> 本文主要针对Mybatis的Spring插件的扫描进行分析，从`MapperScan`注解切入，进行源码层面的简要解读。

## 一、@MapperScan注解

+ `@MapperScan`是Mybatis的Spring插件的核心扫描配置，其通过Spring配置的`@Import`注解注册其核心配置。
  
  ```java
  package org.mybatis.spring.annotation;
  
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElementType.TYPE)
  @Documented
  @Import(MapperScannerRegistrar.class)
  @Repeatable(MapperScans.class)
  public @interface MapperScan {
      // 省略了注解成员
  }
  ```

+ 其中`MapperScannerRegistrar`配置类是`@MapperScan`注册的核心配置类。

## 二、MapperScannerRegistrar配置类

+ `MapperScannerRegistrar`是`ImportBeanDefinitionRegistrar`接口的实现类，我们知道所有`ImportBeanDefinitionRegistrar`的实现类在Spring扫描配置类的时候会进行实例化，并执行其中的关键回调方法。
  
  ```java
  package org.springframework.context.annotation;
  
  public interface ImportBeanDefinitionRegistrar {
  
      default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry,
              BeanNameGenerator importBeanNameGenerator) {
  
          registerBeanDefinitions(importingClassMetadata, registry);
      }
  
      default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
      }
  }
  ```

+ 我们看到`ImportBeanDefinitionRegistrar`接口中声明的回调方法，可以传入通过`@Import`修饰的配置注解的元数据和容器注册器。本接口的实现类会在Spring扫描配置类的过程中被扫描，然后实例化并执行相应的回调方法，这个回调方法允许用户自定义注册Bean。

+ `MapperScannerRegistrar`类中对上述回调重写的关键逻辑在于包含一个对`MapperScannerConfigurer`类的Bean注册过程，同时根据注解中包含的各种参数信息
  填充`MapperScannerConfigurer`类中的各项成员属性。
  
  ```java
  package org.mybatis.spring.annotation;
  
  public class MapperScannerRegistrar implements ImportBeanDefinitionRegistrar {
  
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
      AnnotationAttributes mapperScanAttrs = AnnotationAttributes
          .fromMap(importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
      if (mapperScanAttrs != null) {
        registerBeanDefinitions(importingClassMetadata, mapperScanAttrs, registry,
            generateBaseBeanName(importingClassMetadata, 0));
      }
    }
  
    void registerBeanDefinitions(AnnotationMetadata annoMeta, AnnotationAttributes annoAttrs,
        BeanDefinitionRegistry registry, String beanName) {
  
      BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(MapperScannerConfigurer.class);
      builder.addPropertyValue("processPropertyPlaceHolders", true);
  
      Class<? extends Annotation> annotationClass = annoAttrs.getClass("annotationClass");
      if (!Annotation.class.equals(annotationClass)) {
        builder.addPropertyValue("annotationClass", annotationClass);
      }
  
      Class<?> markerInterface = annoAttrs.getClass("markerInterface");
      if (!Class.class.equals(markerInterface)) {
        builder.addPropertyValue("markerInterface", markerInterface);
      }
  
      Class<? extends BeanNameGenerator> generatorClass = annoAttrs.getClass("nameGenerator");
      if (!BeanNameGenerator.class.equals(generatorClass)) {
        builder.addPropertyValue("nameGenerator", BeanUtils.instantiateClass(generatorClass));
      }
  
      Class<? extends MapperFactoryBean> mapperFactoryBeanClass = annoAttrs.getClass("factoryBean");
      if (!MapperFactoryBean.class.equals(mapperFactoryBeanClass)) {
        builder.addPropertyValue("mapperFactoryBeanClass", mapperFactoryBeanClass);
      }
  
      String sqlSessionTemplateRef = annoAttrs.getString("sqlSessionTemplateRef");
      if (StringUtils.hasText(sqlSessionTemplateRef)) {
        builder.addPropertyValue("sqlSessionTemplateBeanName", annoAttrs.getString("sqlSessionTemplateRef"));
      }
  
      String sqlSessionFactoryRef = annoAttrs.getString("sqlSessionFactoryRef");
      if (StringUtils.hasText(sqlSessionFactoryRef)) {
        builder.addPropertyValue("sqlSessionFactoryBeanName", annoAttrs.getString("sqlSessionFactoryRef"));
      }
  
      List<String> basePackages = new ArrayList<>();
      basePackages.addAll(
          Arrays.stream(annoAttrs.getStringArray("value")).filter(StringUtils::hasText).collect(Collectors.toList()));
  
      basePackages.addAll(Arrays.stream(annoAttrs.getStringArray("basePackages")).filter(StringUtils::hasText)
          .collect(Collectors.toList()));
  
      basePackages.addAll(Arrays.stream(annoAttrs.getClassArray("basePackageClasses")).map(ClassUtils::getPackageName)
          .collect(Collectors.toList()));
  
      if (basePackages.isEmpty()) {
        basePackages.add(getDefaultBasePackage(annoMeta));
      }
  
      String lazyInitialization = annoAttrs.getString("lazyInitialization");
      if (StringUtils.hasText(lazyInitialization)) {
        builder.addPropertyValue("lazyInitialization", lazyInitialization);
      }
  
      builder.addPropertyValue("basePackage", StringUtils.collectionToCommaDelimitedString(basePackages));
  
      registry.registerBeanDefinition(beanName, builder.getBeanDefinition());
  
    }
  
    private static String generateBaseBeanName(AnnotationMetadata importingClassMetadata, int index) {
      return importingClassMetadata.getClassName() + "#" + MapperScannerRegistrar.class.getSimpleName() + "#" + index;
    }
  
    private static String getDefaultBasePackage(AnnotationMetadata importingClassMetadata) {
      return ClassUtils.getPackageName(importingClassMetadata.getClassName());
    }
  
  }
  ```

## 三、MapperScannerConfigurer配置类

+ 我们知道Bean工厂后置处理器是Spring初始化全流程中的关键类，其中Bean的扫描注册等工作都需要依靠内置的Bean工厂后置处理器来实现的。`MapperScannerConfigurer`配置类就是是`BeanDefinitionRegistryPostProcessor`接口的实现类，而`BeanDefinitionRegistryPostProcessor`接口作为`BeanFactoryPostProcessor`的子接口，也是一种特别的Bean工厂后置处理器，可以实现一些自定义Bean的动态注册。
  
  ```java
  @FunctionalInterface
  public interface BeanFactoryPostProcessor {
      void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
  }
  ```
  
  public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
  
      void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
  
  }

```
+ `MapperScannerConfigurer`配置类在Bean工厂后置处理器的关键回调中，实例化了一个`ClassPathMapperScanner`扫描类，用来扫描所有符合Mybatis映射器（mapper）规范的接口。

```java
package org.mybatis.spring.mapper;

public class MapperScannerConfigurer
    implements BeanDefinitionRegistryPostProcessor, InitializingBean, ApplicationContextAware, BeanNameAware {

  @Override
  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    if (this.processPropertyPlaceHolders) {
      processPropertyPlaceHolders();
    }

    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
    scanner.setMapperFactoryBeanClass(this.mapperFactoryBeanClass);
    if (StringUtils.hasText(lazyInitialization)) {
      scanner.setLazyInitialization(Boolean.valueOf(lazyInitialization));
    }
    scanner.registerFilters();
    scanner.scan(
        StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
  }
}
```

## 四、ClassPathMapperScanner扫描类

+ `ClassPathMapperScanner`扫描类是`ClassPathBeanDefinitionScanner`Spring内置的class路径Bean定义扫描器的子类实现，该类重写了成员筛选`isCandidateComponent`方法和具体的扫描逻辑`doScan`方法。
  
  ```java
  package org.mybatis.spring.mapper;
  
  public class ClassPathMapperScanner extends ClassPathBeanDefinitionScanner {
    private Class<? extends MapperFactoryBean> mapperFactoryBeanClass = MapperFactoryBean.class;
  
    @Override
    public Set<BeanDefinitionHolder> doScan(String... basePackages) {
      Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);
  
      if (beanDefinitions.isEmpty()) {
        LOGGER.warn(() -> "No MyBatis mapper was found in '" + Arrays.toString(basePackages)
            + "' package. Please check your configuration.");
      } else {
        processBeanDefinitions(beanDefinitions);
      }
  
      return beanDefinitions;
    }
  
    private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
      GenericBeanDefinition definition;
      for (BeanDefinitionHolder holder : beanDefinitions) {
        definition = (GenericBeanDefinition) holder.getBeanDefinition();
        String beanClassName = definition.getBeanClassName();
        LOGGER.debug(() -> "Creating MapperFactoryBean with name '" + holder.getBeanName() + "' and '" + beanClassName
            + "' mapperInterface");
  
        // the mapper interface is the original class of the bean
        // but, the actual class of the bean is MapperFactoryBean
        definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName); // issue #59
        definition.setBeanClass(this.mapperFactoryBeanClass);
  
        definition.getPropertyValues().add("addToConfig", this.addToConfig);
  
        boolean explicitFactoryUsed = false;
        if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
          definition.getPropertyValues().add("sqlSessionFactory",
              new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
          explicitFactoryUsed = true;
        } else if (this.sqlSessionFactory != null) {
          definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
          explicitFactoryUsed = true;
        }
  
        if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
          if (explicitFactoryUsed) {
            LOGGER.warn(
                () -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
          }
          definition.getPropertyValues().add("sqlSessionTemplate",
              new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
          explicitFactoryUsed = true;
        } else if (this.sqlSessionTemplate != null) {
          if (explicitFactoryUsed) {
            LOGGER.warn(
                () -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
          }
          definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
          explicitFactoryUsed = true;
        }
  
        if (!explicitFactoryUsed) {
          LOGGER.debug(() -> "Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
          definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
        }
        definition.setLazyInit(lazyInitialization);
      }
    }
  
    @Override
    protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
      return beanDefinition.getMetadata().isInterface() && beanDefinition.getMetadata().isIndependent();
    }
  }
  ```

+ `isCandidateComponent`方法的重写逻辑让本类仅扫描独立接口。

+ `doScan`方法的重写在讲所有符合条件的接口扫描成Bean定义的基础上，对Bean定义进行修改，设置这些mapper Bean的class为`MapperFactoryBean`，并设置该类中的各种成员属性值，最后将这些Bean定义注册到容器。

## 五、MapperFactoryBean类

+ `MapperFactoryBean`类是`FactoryBean`接口的实现类，同时也是`SqlSessionDaoSupport`类的子类，持有`sqlSessionTemplate`对象的引用。

+ `MapperFactoryBean`类作为工厂Bean，返回的类型是之前扫面的mapper接口类型，返回的Bean对象是通过Mybatis（sql会话接口）生产的mapper实现类。
  
  ```java
  package org.mybatis.spring.mapper;
  
  public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {
  
    private Class<T> mapperInterface;
  
    private boolean addToConfig = true;
  
    public MapperFactoryBean() {
      // intentionally empty
    }
  
    public MapperFactoryBean(Class<T> mapperInterface) {
      this.mapperInterface = mapperInterface;
    }
  
    @Override
    protected void checkDaoConfig() {
      super.checkDaoConfig();
  
      notNull(this.mapperInterface, "Property 'mapperInterface' is required");
  
      Configuration configuration = getSqlSession().getConfiguration();
      if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
        try {
          configuration.addMapper(this.mapperInterface);
        } catch (Exception e) {
          logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", e);
          throw new IllegalArgumentException(e);
        } finally {
          ErrorContext.instance().reset();
        }
      }
    }
  
    @Override
    public T getObject() throws Exception {
      return getSqlSession().getMapper(this.mapperInterface);
    }
  
    @Override
    public Class<T> getObjectType() {
      return this.mapperInterface;
    }
  
    @Override
    public boolean isSingleton() {
      return true;
    }
  
    public void setMapperInterface(Class<T> mapperInterface) {
      this.mapperInterface = mapperInterface;
    }
  
    public Class<T> getMapperInterface() {
      return mapperInterface;
    }
  
    public void setAddToConfig(boolean addToConfig) {
      this.addToConfig = addToConfig;
    }
  
    public boolean isAddToConfig() {
      return addToConfig;
    }
  }
  ```

+ 这样的话就实现了将所有的mapper实现类注册到Spring容器中了。

## 六、总结

![](/assets/img/post/2022-09-12-mybatis-spring-scanner-1.png)

+ Mybatis的Spring插件扫描流程总结为：
  
  + 首先通过`@MapperScan`注解注册扫描配置信息；
  
  + `@MapperScan`注解通过`@Import`注解注册`MapperScannerRegistrar`配置类；
  
  + `MapperScannerRegistrar`配置类通过`ImportBeanDefinitionRegistrar`接口的回调方法注册`MapperScannerConfigurer`Bean工厂后置处理器；
  
  + `MapperScannerConfigurer`类在`BeanDefinitionRegistryPostProcessor`接口的回调方法中实例化`ClassPathMapperScanner`类并执行扫描；
  
  + `ClassPathMapperScanner`扫描过程会将所有扫描到的mapper接口的Bean定义做修改，替换class为`MapperFactoryBean`，并填充`MapperFactoryBean`的关键属性，进而注册到Spring容器中；
  
  + `MapperFactoryBean`类是一个工厂Bean，返回的Bean类型是之前扫描过程中设置的mapper接口类型，返回的Bean对象是通过sql会话获取到的经Mybatis反射实现的mapper接口实现类。
