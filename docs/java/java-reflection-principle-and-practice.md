---
title: Java反射原理和实际用法
---

# 背景

反射在Java中非常重要，是Java区别于其他编程语言的一大特性。Java中的AOP切面、动态代理等看起来像黑魔法一样的技术，就离不开反射、字节码等。这些技术能在不侵入原有代码的情况下，做一些增强的非功能性需求。多提一句，千万不要把业务逻辑放在AOP切面、动态代理里，否则后人绝对会骂。

* AOP切面：在方法执行前后增加逻辑，可决定方法如何执行、甚至不执行。
* 动态代理：在运行时生成目标类的代理类，可增强目标类的功能。

本文总结一下反射的原理和实际用法。后续有空再介绍AOP切面、动态代理。

# 什么是反射？

看一下官方的原文定义：

> Reflection is a feature in the Java programming language. It allows an executing Java program to examine or "introspect" upon itself, and manipulate internal properties of the program. For example, it's possible for a Java class to obtain the names of all its members and display them.

翻译过来就是：反射是指一个运行中的Java程序可以检查、修改自己内部的属性，也可称之为自省。反射是Java有别于其它编程语言的一大特性。
从reflection的字面意思看，就是倒影、反射，就好比你照镜子，通过倒影就能知道自己长什么样，理一理头发就能改变发型。

# 反射的原理

一句话概况就是：JVM会动态加载Class，一个Class实例包含了该类的所有完整信息，如：包名、类名、各个字段、各个方法、父类、实现的接口等。
因此，如果获取了某个类或对象的Class实例，就可以通过它获取到对应类的所有信息。

动态加载是指，JVM在第一次读取到一种Class类型时，才将其加载进内存，而不是一启动就加载所有类的信息。
每加载一种类，JVM就为其创建一个Class类型的实例，并关联起来。也即一个类的不同对象实例，背后对应的是同一个Class实例。

详细介绍可参考：https://www.liaoxuefeng.com/wiki/1252599548343744/1264799402020448

# 怎么使用反射？

需要熟练使用反射中常见的几个类：Class、Field、Method、Constructor。其它还有参数Parameter类等。可以这么理解，凡是Java对象中出现的东西都能在java.reflect包中找到对应的类。

## Class

通过3种方式获取：

* 类.class：类的class静态变量
* 对象.getClass()
* Class.forName("类的全路径名")

## Field

* Class实例.getField(name)：根据字段名获取某个public的Field（包括父类）
* Class实例.getDeclaredField(name)：根据字段名获取该类声明的某个Field（不包括父类）。常用
* Field[] getFields()：获取所有的public字段，包括父类的字段。不常用，因为按照Java规范，一般都会定义private字段，然后通过public的getter、setter方法来获取字段值
* **Field[] getDeclaredFields()**：获取该类声明所有的字段，不包括父类的字段。常用。

## Method

* Class实例.getMethod(name, Class...)：获取某个public的Method（包括父类）
* Class实例.getDeclaredMethod(name, Class...)：获取该类的某个Method（不包括父类）
* **Method[] getMethods()**：获取所有public的Method（包括父类）。常用
* Method[] getDeclaredMethods()：获取当前类的所有Method（不包括父类）

## AnnotatedElement

Class、Field、Method都是AnnotatedElement的子类，有这些常用方法：

* getAnnotation(Class)：根据Class获取对应的注解
* isAnnotationPresent(Class)：判断是否被某个注解修饰，等效于getAnnotation(annotationClass) != null，是一种简便的写法
* getAnnotations()：获取所有修饰的注解

# 反射用法举例

## 1、反射获取类的所有字段，方便后续运行时读写

如果一个类没有实现toString()方法，或者某些private字段没有提供getter方法，那么想要遍历所有字段，就只能通过反射来实现了。获取到所有反射字段后，想遍历或者修改就很容易了。

* 通过Field和对象，获取对应字段的值：`field.get(object)`
* 通过Field和对象，修改对应字段的值：
  * 先设置为可访问，这样即使是private字段，也能修改：`field.setAccessible(true);`
  * 反射修改字段值：`field.set(Object, Object)`

反射获取类的所有字段，分成2部分：

* 一是该类的父类的字段
* 二是该类引用的其它类的字段

getDeclaredFields方法只能获取到该类声明的字段，如果该类还继承了父类（可能有多层继承关系），那怎么获取所有这些继承而来的字段呢？所有类默认都是Object的子类，或者说Object是根上的父类。利用这一特性，可以通过getSuperclass不断往上找父类，获取父类的字段，然后直至父类为Object。同时也需了解，Object类本身没有任何字段。
于是，可以这么实现：

```java
    // 获取类的所有字段，包括父类
    private List<Field> getAllFields(Class<?> clazz) {
        List<Field> result = new ArrayList<>();
        Class<?> cls = clazz;
        while (cls != null) { // 也可以写成 while (cls != null && cls != Object.class)
            result.addAll(Arrays.asList(cls.getDeclaredFields()));
            cls = cls.getSuperclass();
        }

        return result;
    }
```

把某个类涉及的所有字段的反射信息（包括引用的其他类），都缓存至本地。
之所以要把反射信息缓存至本地，是因为反射是一个耗时操作。如果在初始化阶段就缓存起来，在后续要用到时性能更快。
**使用广度优先遍历算法获取类引用的所有反射字段**。以下是带注释的代码。

```java
/**
 * 获取类的所有反射字段示例（包含引用的类）
 */
public class GetAllFields {
    // 构建反射字段的缓存：key是全路径类名，value是该类的所有字段
    public static Map<String, List<Field>> buildReflectCache(Class<?> clazz) {
        Map<String, List<Field>> result = new HashMap<>();
        List<Field> topLevelFields = getAllFields(clazz);
        result.put(clazz.getName(), topLevelFields);

        // 广度优先遍历类的字段，通过队列来实现
        Queue<Field> queue = new LinkedList<>(topLevelFields);
        while (!queue.isEmpty()) {
            Field field = queue.poll();
            // 如果是集合或Map类型的字段，需要提取出泛型
            if (Collection.class.isAssignableFrom(field.getType())
                    || Map.class.isAssignableFrom(field.getType())) {
                Type genericType = field.getGenericType();
                if (genericType instanceof ParameterizedType) {
                    ParameterizedType parameterizedType = (ParameterizedType) genericType;
                    for (Type type : parameterizedType.getActualTypeArguments()) {
                        Class<?> actualClass = (Class<?>) type;
                        if (!isBasicClass(actualClass)) {
                            List<Field> subFields = getAllFields(actualClass);
                            result.putIfAbsent(actualClass.getName(), subFields);
                            queue.addAll(subFields);
                        }
                    }
                }
            } else if (!isBasicClass(field.getType())) { // 只处理自定义类型，不处理Java基本类型
                List<Field> subFields = getAllFields(field.getType());
                result.putIfAbsent(field.getType().getName(), subFields);
                queue.addAll(subFields);
            }
        }

        return result;
    }

    // 获取类的所有字段，包括父类
    private static List<Field> getAllFields(Class<?> clazz) {
        List<Field> result = new ArrayList<>();
        Class<?> cls = clazz;
        while (cls != null) {
            result.addAll(Arrays.asList(cls.getDeclaredFields()));
            cls = cls.getSuperclass();
        }

        return result;
    }

    // 是否Java基本类型：通过加载类的class loader是否为null得知，null为Java基本类型，否则为自定义类型
    private static boolean isBasicClass(Class<?> clazz) {
        return clazz != null && clazz.getClassLoader() == null;
    }

    public static void main(String[] args) {
        List<Field> allFields = getAllFields(SomeClass.class);
        System.out.println(allFields);

        Map<String, List<Field>> reflectCache = buildReflectCache(SomeClass.class);
        System.out.println(reflectCache);
    }
}
```

## 2、结合反射和Spring，初始化时找到对应bean的对应方法

需要结合Spring来获取bean：`Map<String, Object> applicationContext.getBeansWithAnnotation(Class)`，key是bean名称，value是bean Class实例。

假设有2个注解，@PrintInfoClass修饰在类上，@PrintInfoMethod修饰在方法上，代表要打印方法的签名。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.Type})
@Documented
public @interface PrintInfoClass {
}

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
@Documented
public @interface PrintInfoMethod {
}
```

```java
@Component
public class PrintInfo implements ApplicationContextAware {

    @PostConstruct
    public void init() {
        // 通过Spring框架，方便地获取带PrintInfoClass注解的bean
        Map<String, Object> beansWithAnnotationMap = applicationContext.getBeansWithAnnotation((PrintInfoClass.class);
        for (Map.Entry<String, Object> entry : beansWithAnnotationMap.entrySet()) {
            Object bean = entry.getValue();
            for (Method method : bean.getMethods()) {
                // 只获取带PrintInfoMethod注解的方法
                if (method.isAnnotationPresent(PrintInfoMethod.class) {
                    // 打印方法签名
                    StringBuilder paramString = new StringBuilder();
                    Class<?>[] paramClassList = method.getParameterTypes();
                    for (int i = 0; i < paramClassList.length; ++i) {
                        Class<?> paramClass = paramClassList[i];
                        paramString.append(paramClass.getSimpleName());
                        if (i != paramClassList.length - 1) {
                            paramString.append(",");
                        }
                    }
                    System.out.print(Modifier.toString(method.getModifiers()) + " " + method.getReturnType().getSimpleName()
                      + " " + method.getName() + "(" + paramString.toString() + ")\n");
                }
            }
        }
    }

    @Autowired
    private ApplicationContext applicationContext;

    public void setApplicationContext(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

}
```
