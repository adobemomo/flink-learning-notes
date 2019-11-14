# Flink学习笔记(4)数据类型和序列化

## Flink中的类型处理

Flink会试图推断在分布式计算中被交换和存储的数据类型的信息，有点像数据库可以推断表的模式(schema)，大多数情况下Flink可以自行推断出所有必要的信息，拥有类型信息后，Flink可以：

- 使用POJO类型，并且通过指定file names来进行grouping / joining / aggregating操作(类似`dataSet.keyBy("username")`)。Type info允许Flink提前检查输入错误和类型兼容问题，不用等到运行时再出错失败。
- Flink知道越多关于数据类型的情况，序列化和数据分布scheme就会做的越好，这对于优雅地使用内存非常重要(尽可能在堆内/堆外序列化数据，使得序列化开销很小)。//？？？
- 大多数情况下，可以使用户不用担心序列化框架和必须去注册types。



## 最常见的问题

用户处理Flink数据类型时最常见的问题有：

##### 注册子类型(subtypes)

如果函数签名只定义了supertypes，但实际上在执行中用了subtypes，让Flink意识到这些subtypes可以显著改善性能。 在 `StreamExecutionEnvironment` or `ExecutionEnvironment` 中为每个subtypes调用 `.registerType(clazz)`  。

##### 注册自定义序列化器

对于Flink本身无法透明处理的类型，退回给Kryo处理。不是所有的类型都能被Kryo无缝处理。比如很多Google Guava collection类型在默认情况下运行的不会很好。解决方案是为这些有问题的types注册额外的序列化器。调用 `.getConfig().addDefaultKryoSerializer(clazz, serializer)`在 `StreamExecutionEnvironment` or `ExecutionEnvironment` 里。添加的Kryo序列化器在很多库里可以用。参考 更多细节在[Custom Serializers](https://ci.apache.org/projects/flink/flink-docs-release-1.9/dev/custom_serializers.html) 。

##### 添加类型hints

有时，当Flink尽管有所有技巧仍无法推断出泛型时，用户必须传递*type hint*。通常只在Java API中才需要。 [Type Hints Section](https://ci.apache.org/projects/flink/flink-docs-stable/dev/types_serialization.html#type-hints-in-the-java-api) 描述了更多细节。

#####  手动添加TypeInformation

因为Java的泛型擦除，有些数据类型不能被Flink所识别。

## Flink TypeInformation类

 TypeInformation类时所有类型描述符的基类，展示了一些类型的基本属性，可以生成序列化器和比较器。*(注意：Flink中的比较器的作用远不止定义一个order——比较器时用来用来处理key的实用程序)*

在内部，Flink通过以下来区别types：

- 基础类型：所有的Java原始类型和boxed form(???)，加上 `void`, `String`, `Date`, `BigDecimal`和 `BigInteger`。
- 原始array和对象array。
- 复合类型
  - Flink Java Tuples(Flink Java API的一部分)：最多有25个域，不支持null字段。
  - Scala *case classes*(包括Scala tuples)：不支持null字段。
  - Row： 具有任意数量的字段，支持空字段。
  - POJOs：遵循特定类bean模式的类。
- 辅助类型( Option, Either, Lists, Maps, … )
- 泛型：不会被Flink序列化，由Kryo进行序列化。

POJOs需要被特别关注，它们支持复杂类型的创建，并且能在key定义中使用字段名： `dataSet.join(another).where("name").equalTo("personName")` 。同时在运行时也是透明的，可以被Flink非常高效地处理。

#### POJO类型的规则

如果以下的条件满足，Flink认为一个数据类型时POJO类型：

- 类是public的并且是单点的(没有非静态内部类)。
- 类拥有一个public无参构造函数。
- 类/父类中所有非静态，非transient的字段，要么是公开(非final)的，要么有一个public getter&setter方法，命名规范符合Java beans的getter&setter。

注意：当用户自定义的数据类型不能被识别为POJO类型，那肯定是被处理为泛型了，并且被Kryo序列化了。

#### 创建TypeInformation/TypeSerializer

**JAVA**

Java通常会擦除泛型的信息，所以用户需要给TypeInformation传递类型。

对于非泛型，可以传递Class：

```java
TypeInformation<String> info = TypeInformation.of(String.class);
```

对于泛型，需要通过 `TypeHint`来捕获泛型信息： 

```java
TypeInformation<Tuple2<String, Double>> info = TypeInformation.of(new TypeHint<Tuple2<String, Double>>(){});
```

在内部，这行代码会创建一个`TypeHint`的匿名子类，可以保存TypeInformation直到运行时。

创建 `TypeSerializer`的话，直接在`TypeInformation`对象上调用  `typeInfo.createSerializer(config)`即可。

 //TODO 暂不了解

 The `config` parameter is of type `ExecutionConfig` and holds the information about the program’s registered custom serializers. Where ever possibly, try to pass the programs proper ExecutionConfig. You can usually obtain it from `DataStream` or `DataSet` via calling `getExecutionConfig()`. Inside functions (like `MapFunction`), you can get it by making the function a [Rich Function](https://ci.apache.org/projects/flink/flink-docs-stable/dev/types_serialization.html) and calling `getRuntimeContext().getExecutionConfig()`. 

## Java API中的类型信息

通常情况下，Java会擦除泛型信息。Flink尽可能地通过反射(???)了重建类型信息。此逻辑还包含一些简单的类型推断，适用于函数的返回类型依赖于其输入类型的情况：

```java
public class AppendOne<T> implements MapFunction<T, Tuple2<T, Long>> {

    public Tuple2<T, Long> map(T value) {
        return new Tuple2<T, Long>(value, 1L);
    }
}
```

有时候Flink不能重新构建泛型的信息，这种情况下就需要用户指定*type hints*。

#### Java API中的类型提示Type Hints

类型提示通过一个函数告诉系统数据流/数据集的数据类型：

```java
DataSet<SomeType> result = dataSet
    .map(new MyGenericNonInferrableFunction<Long, SomeType>())
        .returns(SomeType.class);
```

`return`语句指定了数据类型，hint支持通过以下方式来定义类型：

- 对于无参类型来说，使用Classes。(无泛型)
-  `returns(new TypeHint<Tuple2<Integer, SomeType>>(){})`形式的类型提示。`TypeHint`这个类会保存泛型的信息以供运行时使用(通过匿名子类的方式)。

####  Java 8 lambdas的类型抽出

//TODO

Type extraction for Java 8 lambdas works differently than for non-lambdas, because lambdas are not associated with an implementing class that extends the function interface.

Currently, Flink tries to figure out which method implements the lambda and uses Java’s generic signatures to determine the parameter types and the return type. However, these signatures are not generated for lambdas by all compilers (as of writing this document only reliably by the Eclipse JDT compiler from 4.5 onwards).

#### POJO类型的序列化

 `PojoTypeInfo`给POJO里的所有字段创建序列化器。像int，long，String这种标准类型可以被内嵌在Flink里的序列化器处理。对于其他所有的类型，会抛回给Kryo。

如果Kryo没办法处理这些类型，可以请求`PojoTypeInfo`使用Avro序列化POJO，先开启Avro：

*NOTE:*Flink会自动使用Avro来序列化POJO

```java
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().enableForceAvro();
```

如果想要Kryo来处理**整个**的POJO，需要设置：

```java
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().enableForceKryo();
```

如果Kryo不能序列化你的POJO，可以添加自定义序列化器到POJO：

```java
env.getConfig().addDefaultKryoSerializer(Class<?> type, Class<? extends Serializer<?>> serializerClass)
```

## 使用工厂定义TypeInformation

//TODO

A type information factory allows for plugging-in user-defined type information into the Flink type system. You have to implement `org.apache.flink.api.common.typeinfo.TypeInfoFactory` to return your custom type information. The factory is called during the type extraction phase if the corresponding type has been annotated with the `@org.apache.flink.api.common.typeinfo.TypeInfo` annotation.

Type information factories can be used in both the Java and Scala API.

In a hierarchy of types the closest factory will be chosen while traversing upwards, however, a built-in factory has highest precedence. A factory has also higher precedence than Flink’s built-in types, therefore you should know what you are doing.

The following example shows how to annotate a custom type `MyTuple` and supply custom type information for it using a factory in Java.

The annotated custom type:

```
@TypeInfo(MyTupleTypeInfoFactory.class)
public class MyTuple<T0, T1> {
  public T0 myfield0;
  public T1 myfield1;
}
```

The factory supplying custom type information:

```
public class MyTupleTypeInfoFactory extends TypeInfoFactory<MyTuple> {

  @Override
  public TypeInformation<MyTuple> createTypeInfo(Type t, Map<String, TypeInformation<?>> genericParameters) {
    return new MyTupleTypeInfo(genericParameters.get("T0"), genericParameters.get("T1"));
  }
}
```

The method `createTypeInfo(Type, Map>)` creates type information for the type the factory is targeted for. The parameters provide additional information about the type itself as well as the type’s generic type parameters if available.

If your type contains generic parameters that might need to be derived from the input type of a Flink function, make sure to also implement `org.apache.flink.api.common.typeinfo.TypeInformation#getGenericParameters` for a bidirectional mapping of generic parameters to type information.



## 其他

### Java泛型

Java的泛型的实现根植于“类型消除”这一概念。当源代码被转换成Java虚拟机字节码时，这种技术会消除参数化类型。

例如，假设有一下java 代码：


```java
Vector<String> vector = new Vector<String>();
vector.add(new String("hello"));
String str = vector.get(0);
```

编译时，上面的代码会被改写为：

```java
Vector vector = new Vector();
vector.add(new String("hello"));
String str = (String)vector.get(0);
```

有了Java泛型，我们可以做的事情也并没有真正改变多少；它只是让代码变得漂亮些。鉴于此，Java泛型有时也被成为”语法糖“。

这点跟C++的模板截然不同。

在C++中，模板本质上就是一套宏指令集，只是换了个名头，编译器会针对每种类型创建一份模板代码的副本。

有个证据可以证明这一点：`MyClass<Foo>`不会与`MyClass<Bar>`共享静态变量。然而，两个`MyClass<Foo>`实例则会共享静态变量。

```cpp
/*

 * MyClass.cpp
   *
 * Created on: 2015?7?24?
 * Author: nanzhou
    */
   template<class T> class MyClass {
   public:
   static int val;
   MyClass(int v) {val = v;}
   };

template<typename T> int MyClass<T>::bar;

template class MyClass<Foo>;
template class MyClass<Bar>;

MyClass<Foo>* foo1 = new MyClass<Foo>(10);
MyClass<Foo>* foo2 = new MyClass<Foo>(15);
MyClass<Bar>* bar1 = new MyClass<Bar>(20);
MyClass<Bar>* bar2 = new MyClass<Bar>(35);

int f1 = foo1->val;//15
int f2 = foo2->val;//15
int b1 = bar1->val;//35
int b2 = bar2->val;//35
```

但在Java中，`MyClass`类的静态变量会由所有`MyClass`实例共享，无论类型参数相与否。


由于架构设计上的差异，Java泛型和C++模板还有如下很多不同点;

1. C++模板可以使用int等基本数据类型。Java则不行，必须转而使用`Integer`

2. Java中，可以将模板的类型参数限定为某种特定类型。例如，你可能会使用泛型实现CardDeck，并规定参数必须扩展自CardGame。

3. C++中，类型参数可以实例化，Java不可以实例化

4. Java中，类型参数（即`MyClass<Foo>`中的`Foo`）不能用于静态方法和变量，因为他们会被`MyClass<Foo>`和`MyClass<Bar>`共享。但在C++中，这些类是不同的，类型参数可以用于静态方法和静态变量。

5. 在Java中，不管类型参数是什么，`MyClass`的所有实例都是同一类型。类型参数会在运行时被抹去。而C++中，参数类型不同，实例类型也不同



Java的泛型和C++模板，虽然在很多方面看起来都一样，实则大不相同。



————————————————
版权声明：本文为CSDN博主「michellechouu」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/michellechouu/article/details/47044331

### POJO

*什么是POJO*？*——普通的JavaBean*

- **Java Bean**

  JavaBean是公共Java类，但是为了编辑工具识别，需要满足至少三个条件：

  1. 有一个`public`默认构造器（例如无参构造器,）
  2. 属性使用`public` 的get，set方法访问，也就是说设置成`private`，同时`get`，`set`方法与属性名的大小也需要对应。例如属性name，get方法就要写成，`public String getName(){}`,N大写。
  3. 需要序列化。这个是框架，工具跨平台反映状态必须的

- **EJB**

  ​	在企业开发中，需要可伸缩的性能和事务、安全机制，这样能保证企业系统平滑发展，而不是发展到一种规模重新更换一套软件系统。 然后有提高了协议要求，就出现了**Enterprise Bean。**

  ​	EJB在javabean基础上又提了一些要求，当然更复杂了。 

- **POJO**

  ​	有个叫Josh MacKenzie人觉得，EJB太复杂了，完全没必要每次都用，所以发明了个POJO，POJO是普通的javabean，什么是普通，就是和EJB对应的。

  ​	总之，区别就是，你先判断是否满足javabean的条件，然后如果再实现一些要求，满足EJB条件就是EJB，否则就是POJO。 