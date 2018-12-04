## BeanFactory和FactoryBean的区别

BeanFactory是spring ioc的基础容器的接口。

通过getBean方法获取ioc容器中的bean

```
	Object getBean(String name) throws BeansException;
```



FactoryBean是交给spring ioc托管的 工厂bean。

```
T getObject() throws Exception;
```

在ioc容器中获取时，实际拿到的是FactoryBean的getObject返回的对象。

如果要获取factoryBean本身，则需要在name前面加上&



下面分析下AbstractBeanFactory 的getBean方法

```
@Override
public Object getBean(String name) throws BeansException {
   return doGetBean(name, null, null, false);
}
```

下面看doGetBean方法

```
protected <T> T doGetBean(
      final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
      throws BeansException {

  //这个方法是获取bean的原始名字，也就是去掉所有的&前缀
   final String beanName = transformedBeanName(name);
   Object bean;

   //这里是获取预先初始化单例实例，可能拿到普通的实例，也可能拿到FactoryBean实例
   Object sharedInstance = getSingleton(beanName);
   if (sharedInstance != null && args == null) {
		//......
		//这里如果拿到了类型，就进行实例化
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
   }
   else {
  		//.......
    }
   return (T) bean;
}
```

这里看getObjectForBeanInstance 拿到具体实例化的类



```
protected Object getObjectForBeanInstance(
      Object beanInstance, String name, String beanName, RootBeanDefinition mbd) {

   // Don't let calling code try to dereference the factory if the bean isn't a factory.

   //beanInstance 是根据beanName 从IOC容器中取出的类型
   //name是调用者传入的bean名字 可能带有&
   //beanName是 name去除所有&前缀的bean名称

   //这里判断name如果加上了&前缀，并且取出来的bean不是一个factoryBean 那就直接报错
   if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
      throw new BeanIsNotAFactoryException(transformedBeanName(name), 				  beanInstance.getClass());
   }
   //如果实例不是factoryBean，或者name有 &前缀，那就把取出来的实例直接返回，包括 factoryBean
   if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
      return beanInstance;
   }

   //实例是一个factoryBean, 调用 实例的 getObject方法，返回
   Object object = null;
   if (mbd == null) {
      //从缓存FactoryBeanRegistrySupport#factoryBeanObjectCache 取，如果有就返回
      object = getCachedObjectForFactoryBean(beanName);
   }
   if (object == null) {
      //......
      //真正调用FactoryBean的getObject的地方
      object = getObjectFromFactoryBean(factory, beanName, !synthetic);
   }
   return object;
}
```





```
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
//如果是单例的
   if (factory.isSingleton() && containsSingleton(beanName)) {
      synchronized (getSingletonMutex()) {
      //从缓存获取实例
         Object object = this.factoryBeanObjectCache.get(beanName);
         if (object == null) {
         //如果缓存中没有查到，则直接在这个方法里调用 factory.getObject()方法去获取Object对象
            object = doGetObjectFromFactoryBean(factory, beanName);
            // Only post-process and store if not put there already during getObject() call above
            // (e.g. because of circular reference processing triggered by custom getBean calls)
            //这里是判断，如果在调用FactoryBean的getObject过程中，已经有实例生成好了放入缓存里，就使用已经生成好的
            Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
            if (alreadyThere != null) {
               object = alreadyThere;
            }
            else {
              //这里是对生成Object对象进行创建后的后置处理
               if (object != null && shouldPostProcess) {
                  try {
                     object = postProcessObjectFromFactoryBean(object, beanName);
                  }
                  catch (Throwable ex) {
                     throw new BeanCreationException(beanName,
                           "Post-processing of FactoryBean's singleton object failed", ex);
                  }
               }
               //后置处理完后把对象放入缓存的map并返回
               this.factoryBeanObjectCache.put(beanName, (object != null ? object : NULL_OBJECT));
            }
         }
         return (object != NULL_OBJECT ? object : null);
      }
   }
   //这边是判断如果对象不是单例的就使用FactoryBean的getObject方法重新生成一个Bean
   else {
      Object object = doGetObjectFromFactoryBean(factory, beanName);
      if (object != null && shouldPostProcess) {
         try {
            object = postProcessObjectFromFactoryBean(object, beanName);
         }
         catch (Throwable ex) {
            throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
         }
      }
      return object;
   }
}
```