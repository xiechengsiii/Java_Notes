# JAVA相关的一些特性

1. ```java
   //System.out :: println 是一个方法引用，其等价于 x -> System.out.println(x)
   //使用方法是， 要用 :: 分割方法名与对象或类名 主要三种情况
    Object :: instanceMethod
    Class :: staticMethod
    Class :: instanceMethod
    eg: ArrayList<String> names = ...;
   	Stream<Person> stream = names.stream.map(Person :: new);
   	List<Person> people = stream.collect(Collectors.toList());
    //如果有多个构造器，编译器会选择一个带String参数的构造器
   ```

2. Comparator有很多有意思的用法，见java核心技术6.3.8

3. 什么时候用静态内部类？

   在内部类不要访问外围对象的时候，应该使用静态内部类。

