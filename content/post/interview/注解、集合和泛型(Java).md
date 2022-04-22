---
typora-root-url: ../../../static
title: "注解、集合和泛型(Java)"
date: 2020-06-10T16:04:36+08:00
draft: false
categories: ["interview"]
---

## Java注解面试题
### 注解是什么？
Annotation(注解)是Java提供的一种 **对元程序中元素关联信息和元数据(metaData)的途径和方法** 。Annotation(注解)是一个接口，程序可以通过反射来获取指定程序中元素的Annotation对象，然后通过该Annotaion对象来获取注解中的 **元数据信息** 。

### 4中标准元注解是哪四种？
**元注解的作用是负责注解其他注解** 。Java5.0定义了4个标准的meta-annotation类型，它们被用来提供对其他annotation类型作说明。

1. @Target：标记这个注解应该是哪种Java成员。
2. @Retention：标识这个注解怎么保存，是只在代码中，还是编入class文件中，或者是在运行时可以通过反射访问。
3. @Documented：标记这些注解是否包含在用户文档中。
4. @Inherited：标记这个注解是继承于哪个注解类(默认 注解并没有继承于任何子类)。

## Java集合/泛型面试题
### ArrayList和LinkedList的区别？
1. ArrayList：基于动态数组实现，支持随机访问。
2. LinkedList：基于双向链表实现，只能顺序访问，但是可以快速的在链表中插入和删除。此外，LinkedList还可以用作栈、队列和双向队列。
3. LinkedList比ArrayList具有更好的插入和删除性能，但是查找性能要差。

### HashMap和Hashtable的区别？
1. 初始容量大小和每次扩容大小不同。
2. 计算Hash值的方法不同。
3. 安全性不同：HashMap是线程不安全的，而Hashtable是线程安全的。
4. 对null的支持不同：HashMap的key允许存在null，但是Hashtable不允许。
5. 对外提供的接口不同：Hashtable比HashMap多提供了elements()和contains()两个方法。其中element()返回此Hashtable中的value的枚举类；contains()用于判断Hashtable是否包含传入的value，实际上containsValue()方法就是调用了contains()方法。
6. 两者的父类不同：HashMap继承于AbstractMap，而Hashtable继承于Dictionary。

### Collection包结构和Collections的区别？
1. Collections是集合类的一个帮助类，不能实例化，包含各种有关集合操作的静态多态方法，是一个工具类，服务于Java的Collection框架。
2. Collection是集合类的上级接口，子接口有Set、List、Queue。

### 泛型常用特点？
1. 不必因为添加元素类型的不同而定义不同类型的集合。
2. 可以在具体执行的时候有具体的规则来约束。
3. 泛型支持通配符。

### List、Set和Map三者的区别？
1. List：存储一组不唯一（不会有多个元素引用相同的对象）、有序的对象。
2. Set：不允许重复的集合。不会有多个元素引用相同的对象。
3. Map：使用键值对存储，Key不能重复。两个key可以引用相同的对象。

### Array和ArrayList有什么不一样？
Array和ArrayList都是用来存储数据的集合。ArrayList底层是使用数组实现的，但是ArrayList对数组进行了封装和扩展，拥有许多原生数组没有的一些功能。ArrayList是Array的一个升级版。

### Map有什么特点？
1. 以键值对存储数据。
2. 存储的元素是无需的。
3. key不允许重复。

### 集合类存放于java.util包中，主要有几种接口？
1. Collection：是集合List、Set和Queue的最基本的父接口。
2. Iterator：迭代器，可以通过迭代器遍历集合中的数据。
3. Map：是映射表的基础接口。

### 什么是List接口？
List是Java中非常常用的数据类型，是有序的集合，List一共有三个实现类：ArrayList、Vector和LinkedList。

### 说说ArrayList？
ArrayList是最常用的List实现类，内部是基于动态数组实现，支持随机访问，缺点是每个元素之间不能有间隔，当数组大小不足需要增加存储能力时，需要将已有数组的数据复制到新的存储空间中。当从ArrayList的中间位置插入或者删除元素时候，需要对数组进行复制、移动，代价较高，因此ArrayList适合随机查找和遍历，不适合插入和删除。

### 说说Vector？
Vetor和ArrayList类似，都是通过数组实现的，但是它支持线程同步，因此访问Vector也比ArrayList慢。

### 说说LinkedList？
LinkedList是用链表存储数据的，很适合数据的动态插入和删除，但是随机访问和遍历比较慢，此外LinkedList可用作栈、队列和双向队列。

### 什么是Set集合？
Set集合注重独一无二的性质，该体系集合用于存储无序元素，值不能重复。判断两个对象是否相等是通过对象的equals()方法来判断的，一般此方法需要保证相同的对象其hashCode也相同。

### 说说HashSet？
HashSet存储元素的顺序不是按照存入的顺序来存储的，而是按照HashCode来存储的，所以取数据也是按照HashCode来取数据的。HashSet不允许存入相同的元素，判断两个元素是否相同的依据是equals()方法是否为true，如果为true就是代表同一个元素，不允许存入，否则允许存入，如果允许存入，碰到两个元素的hashcode相同，就将元素放在同一个Hash桶中，也就是一个HashCode位置上允许存放多个元素。

### 什么是TreeSet（二叉树）？
1. TreeSet是按照二叉树的原理对新添加的元素按照指定的顺序排序，没增加一个对象都会进行排序，将对象插入到二叉树指定的位置。
2. Interger和String对象可以进行默认的TreeSet排序，但是自定义类型的对象不可以，自定义类型的对象必须实现Comparable接口的compareTo()方法，才能正常使用。
3. 实现compareTo()方法时，比较此对象于指定对象的顺序时，如果compareTo()方法返回负整数、0、正整数，则代表该对象小于、等于、大于指定对象。

### 说说LinkedHashSet（HashSet+LinkedHashMap）？
对于LinkedHashSet而言，它继承于HashSet，又基于LinkedHashMap来实现。LinkedHashSet底层使用LinkedHashMap来存储数据，它基础HashSet，其所有操作方法操作上于HashSet相同。插入顺序是有序的。

### 说说HashMap（数组+链表+红黑树）？
1. HashMap根据键的hashCode值存储数据，大多数情况下可以直接定位到它的值，因而具有很快的访问速度，但遍历顺序却是不确定的。
2. HashMap最多允许一条key为null的记录，但是允许多条value为null的记录。
3. HashMap是非线程安全的，如果需要满足线程安全，可以使用Collections的synchronizedMap()方法使HashMap具有线程安全的能力，或者使用ConcurrentHashMap。
4. HashMap使用拉链法解决hash冲突。
5. 从Java8开始，一个桶存储的的链表长度大于等于8时会将链表转成红黑树。

### 说说HashTable（线程安全）？
Hashtable是一个遗留类（从其命名就可以看出，不遵循驼峰法），很多映射的常用功能与HashMap类似，不同的是它继承于Dictionary类，并且是线程安全的，但是并发性不如ConcurrentHashMap。Hashtable不允许存在key为null的元素。

### 说说TreeMap（可排序）？
TreeMap实现了SortedMap接口，能够把它保存的记录根据键排序，默认是按照键的升序配许，也可以指定排序的排序器。当使用迭代器遍历TreeMap时候，得到的是排过序的。在使用TreeMap时候，key必须实现Comparable接口或者在构造TreeMap时传入自定义的Comparator。

### 说说LinkedHashMap（记录插入排序）？
LinkedHashMap是HashMap的一个子类，保存了记录的插入顺序，在用Iterator遍历LinkedHashMap时候，先得到的记录是先插入的。也可以在构造LinkedHashMap时候带参数，按照访问次序排序。

### 说说泛型类？
泛型类除了在类名后面添加类型参数声明部分外，和非泛型类的声明都一致。泛型类的类型参数声明也包含一个或多个类型参数，参数间用逗号隔开。一个泛型参数，也被称之为一个类型变量，是一个用于指定一个泛型类型名称的标识符。因为泛型类接受一个或多个参数，因此这些类被称为参数化的类或者参数化的类型。

### 说说类型通配符？
类型通配符一般是使用？代替具体的类型参数。

### 说说类型擦除？
Java中的泛型基本上都是在编译器这个层次上来实现的。在生成的Java字节码中是不包含泛型中的类型信息的。使用泛型的时候加上类型参数，会被编译器在编译的时候就去掉，这个过程就叫做类型擦除。类型擦除的过程就是首先找到用来替代类型参数的具体类，这个具体类一般是Object，如果制定了类型参数的上界的话，则使用这个上界，然后把代码中的类型参数都替换成具体的类。


