---
typora-root-url: ../../../static
title: "Lambda表达式"
date: 2020-09-05T10:34:36+08:00
draft: false
categories: ["Java"]
---

## Lambda介绍
- Lambda可以简单的理解为可以传递的匿名函数的一种方式，它没有名称，但是有参数列表、函数主体、返回类型以及可以抛出的异常列表。
- Lambda的基本语法为： `(parameters)->expression` 或者 `(parameters)->{statements;}` 。

## 使用Lambda
可以在 **函数式接口上使用Lambda表达式 。函数式接口就是只定义一个抽象方法的接口，哪怕接口定义了很多默认方法，只要接口只定义了一个抽象方法，它就仍然是一个函数式接口。** 

一个代码示例如下：

    Runnable runnable=()-> System.out.println("dasd");
    runnable.run();

## 使用函数式接口
接下来将测试java.util.function包下的几个新的函数式接口。

### Predicate
java.util.function.Predicate<T>接口定义了一个名叫test的抽象方法，它接收泛型T对象，并返回一个boolean。

Predicate部分源码如下：

	@FunctionalInterface
	public interface Predicate<T> {
		boolean test(T t);
	}

一个代码示例如下：

	public static <T> List<T> filter(List<T> list,Predicate<T> predicate){
	        ArrayList<T> res = new ArrayList<>();
	        for(T v:list){
	            if(predicate.test(v)){
	                res.add(v);
	            }
	        }
	        return res;
	    }
	
	    @Test
	    public void test4(){
	        ArrayList<String> source = new ArrayList<>();
	        source.add("吾生也有涯");
	        source.add("而学也无涯");
	        System.out.println(Arrays.asList(filter(source,(String a)->!a.isEmpty())));
	    }

### Consumer
java.util.function.Consumer<T>定义了一个名为accept的抽象方法，它接收泛型T的对象，没有返回。

Consumer部分源码如下：

	@FunctionalInterface
	public interface Consumer<T> {
		void accept(T t);
	}

一个示例代码如下：

	@FunctionalInterface
    public interface Processor{
        void process(List list,Consumer consumer);
    }

    @Test
    public void test5(){
        Processor processor=(List a,Consumer b)->{
            for(Object v:a){
                b.accept(v);
            }
        };

        processor.process(Arrays.asList(1,2,3,4,5),(Object a)->{
            System.out.println(a);
        });
    }

### Function
java.util.function.Function<T,R>接口定义了一个叫做apply的方法，它接收一个泛型T的对象，并返回一个泛型R的对象。

Function的部分源码如下：

	@FunctionalInterface
	public interface Function<T, R> {
		R applay(T t);
	}

一个示例代码如下：

	public static <T,R> List<R> map(List<T> list,Function<T, R> function){
        ArrayList<R> res = new ArrayList<>();
        for(T v:list){
            res.add(function.apply(v));
        }
        return res;
    }

    @Test
    public void test7(){
        List<Integer> map = map(Arrays.asList("123", "pinger", "p1n93r"), (String a) -> a.length());
        System.out.println(Arrays.asList(map));
    }

## 方法引用
方法引用可以重复使用现有的方法定义，并像Lambda一样传递它们。方法引用主要有三类：

1. 指向静态方法的方法引用：例如Integer的parseInt方法，写作Integer::parseInt()。
2. 指向任意类型实例方法的方法引用：例如String的length方法，写作String::length。
3. 指向现有对象的实例方法的方法引用：例如()->test.getVal()可以写作test::getVal。

