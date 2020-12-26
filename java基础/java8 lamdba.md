## lamdba与集合

## LIst

### foreach

该方法的签名为`void forEach(Consumer<? super E> action)`，作用是对容器中的每个元素执行`action`指定的动作，其中`Consumer`是个函数接口，里面只有一个待实现方法`void accept(T t)`

```java
// 使用曾强for循环迭代
ArrayList<String> list = new ArrayList<>(Arrays.asList("I", "love", "you", "too"));
for(String str : list){
    if(str.length()>3)
        System.out.println(str);
}

// 使用forEach()结合Lambda表达式迭代
ArrayList<String> list = new ArrayList<>(Arrays.asList("I", "love", "you", "too"));
list.forEach( str -> {
        if(str.length()>3)
            System.out.println(str);
    });
```

### removeIf

该方法签名为`boolean removeIf(Predicate<? super E> filter)`，作用是**删除容器中所有满足`filter`指定条件的元素**，其中`Predicate`是一个函数接口，里面只有一个待实现方法`boolean test(T t)`

```java
// 使用迭代器删除列表元素
ArrayList<String> list = new ArrayList<>(Arrays.asList("I", "love", "you", "too"));
Iterator<String> it = list.iterator();
while(it.hasNext()){
    if(it.next().length()>3) // 删除长度大于3的元素
        it.remove();
}

// 使用removeIf()结合Lambda表达式实现
ArrayList<String> list = new ArrayList<>(Arrays.asList("I", "love", "you", "too"));
list.removeIf(str -> str.length()>3); // 删除长度大于3的元素
```

### replaceAll

该方法签名为`void replaceAll(UnaryOperator<E> operator)`，作用是**对每个元素执行`operator`指定的操作，并用操作结果来替换原来的元素**。其中`UnaryOperator`是一个函数接口，里面只有一个待实现函数`T apply(T t)`

```java
// 使用下标实现元素替换
ArrayList<String> list = new ArrayList<>(Arrays.asList("I", "love", "you", "too"));
for(int i=0; i<list.size(); i++){
    String str = list.get(i);
    if(str.length()>3)
        list.set(i, str.toUpperCase());
}

// 使用Lambda表达式实现
ArrayList<String> list = new ArrayList<>(Arrays.asList("I", "love", "you", "too"));
list.replaceAll(str -> {
    if(str.length()>3)
        return str.toUpperCase();
    return str;
});
```

### sort

方法签名为`void sort(Comparator<? super E> c)`，该方法**根据`c`指定的比较规则对容器元素进行排序**

```java
// Collections.sort()方法
ArrayList<String> list = new ArrayList<>(Arrays.asList("I", "love", "you", "too"));
Collections.sort(list, new Comparator<String>(){
    @Override
    public int compare(String str1, String str2){
        return str1.length()-str2.length();
    }
});

// List.sort()方法结合Lambda表达式
ArrayList<String> list = new ArrayList<>(Arrays.asList("I", "love", "you", "too"));
list.sort((str1, str2) -> str1.length()-str2.length());
```

## Map

### forEach

该方法签名为`void forEach(BiConsumer<? super K,? super V> action)`，作用是**对`Map`中的每个映射执行`action`指定的操作**，其中`BiConsumer`是一个函数接口，里面有一个待实现方法`void accept(T t, U u)`

```java

HashMap<Integer, String> map = new HashMap<>();
map.put(1, "one");
map.put(2, "two");
map.put(3, "three");

// Java7以及之前迭代Map
for(Map.Entry<Integer, String> entry : map.entrySet()){
    System.out.println(entry.getKey() + "=" + entry.getValue());
}

// 使用forEach()结合Lambda表达式迭代Map
map.forEach((k, v) -> System.out.println(k + "=" + v));
}
```

### getOrDefault

方法签名为`V getOrDefault(Object key, V defaultValue)`，作用是**按照给定的`key`查询`Map`中对应的`value`，如果没有找到则返回`defaultValue`**

```java
// 查询Map中指定的值，不存在时使用默认值
HashMap<Integer, String> map = new HashMap<>();
map.put(1, "one");
map.put(2, "two");
map.put(3, "three");

// Java7以及之前做法
if(map.containsKey(4)){ // 1
    System.out.println(map.get(4));
}else{
    System.out.println("NoValue");
}
// Java8使用Map.getOrDefault()
System.out.println(map.getOrDefault(4, "NoValue")); // 2
```

### replaceAll

该方法签名为`replaceAll(BiFunction<? super K,? super V,? extends V> function)`，作用是对`Map`中的每个映射执行`function`指定的操作，并用`function`的执行结果替换原来的`value`，其中`BiFunction`是一个函数接口，里面有一个待实现方法`R apply(T t, U u)`

```java
HashMap<Integer, String> map = new HashMap<>();
map.put(1, "one");
map.put(2, "two");
map.put(3, "three");

// Java7以及之前替换所有Map中所有映射关系
for(Map.Entry<Integer, String> entry : map.entrySet()){
    entry.setValue(entry.getValue().toUpperCase());
}

// 使用replaceAll()结合Lambda表达式实现
map.replaceAll((k, v) -> v.toUpperCase());
```

## Streams API

### 创建方式

- 调用`Collection.stream()`或者`Collection.parallelStream()`方法
- 调用`Arrays.stream(T[] array)`方法

### 特性

- **无存储**。*stream*不是一种数据结构，它只是某种数据源的一个视图，数据源可以是一个数组，Java容器或I/O channel等。
- **为函数式编程而生**。对*stream*的任何修改都不会修改背后的数据源，比如对*stream*执行过滤操作并不会删除被过滤的元素，而是会产生一个不包含被过滤元素的新*stream*。
- **惰式执行**。*stream*上的操作并不会立即执行，只有等到用户真正需要结果的时候才会执行。
- **可消费性**。*stream*只能被“消费”一次，一旦遍历过就会失效，就像容器的迭代器那样，想要再次遍历必须重新生成。

### 分类

**中间操作**

-  **总是会惰式执行**，调用中间操作只会生成一个标记了该操作的新*stream*。
-  concat() distinct() filter() flatMap() limit() map() peek() skip() sorted() parallel() sequential() unordered()

**结束操作**

+ 会触发实际计算，计算发生时会把所有中间操作积攒的操作以*pipeline*的方式执行，这样可以减少迭代次数。计算完成之后*stream*就会失效。
+ allMatch() anyMatch() collect() count() findAny() findFirst() forEach() forEachOrdered() max() min() noneMatch() reduce() toArray()

### API

#### forEach

对容器中的每个元素执行`action`指定的动作，也就是对元素进行遍历。

```java
// 使用Stream.forEach()迭代
Stream<String> stream = Stream.of("I", "love", "you", "too");
stream.forEach(str -> System.out.println(str));
```

结束方法，立即执行，输出所有字符串。

#### filter

返回一个只包含满足`predicate`条件元素的`Stream`。

```java
// 保留长度等于3的字符串
Stream<String> stream= Stream.of("I", "love", "you", "too");
stream.filter(str -> str.length()==3)
    .forEach(str -> System.out.println(str));
```

中间操作，如果只调用`filter()`不会有实际计算，因此也不会输出任何信息。

#### distinct

作用是返回一个去除重复元素之后的`Stream`

```java
Stream<String> stream= Stream.of("I", "love", "you", "too", "too");
stream.distinct()
    .forEach(str -> System.out.println(str));
```

中间操作

#### sorted

```java
Stream<String> stream= Stream.of("I", "love", "you", "too");
stream.sorted((str1, str2) -> str1.length()-str2.length())
    .forEach(str -> System.out.println(str));
```

中间操作

#### map

返回一个对当前所有元素执行执行`mapper`之后的结果组成的`Stream`。

```java
Stream<String> stream　= Stream.of("I", "love", "you", "too");
stream.map(str -> str.toUpperCase())
    .forEach(str -> System.out.println(str));
```

中间操作

#### flatMap

每个元素执行`mapper`指定的操作，并用所有`mapper`返回的`Stream`中的元素组成一个新的`Stream`作为最终返回结果。通俗的讲`flatMap()`的作用就相当于把原*stream*中的所有元素都”摊平”之后组成的`Stream`

```java
Stream<List<Integer>> stream = Stream.of(Arrays.asList(1,2), Arrays.asList(3, 4, 5));
stream.flatMap(list -> list.stream())
    .forEach(i -> System.out.println(i));
```

上述代码中，原来的`stream`中有两个元素，分别是两个`List<Integer>`，执行`flatMap()`之后，将每个`List`都“摊平”成了一个个的数字，所以会新产生一个由5个数字组成的`Stream`。所以最终将输出1~5这5个数字。

#### reduce

实现从一组元素中生成一个值，`sum()`、`max()`、`min()`、`count()`等都是*reduce*操作

方法定义有三种重写形式：

- `Optional<T> reduce(BinaryOperator<T> accumulator)`
- `T reduce(T identity, BinaryOperator<T> accumulator)`
- `<U> U reduce(U identity, BiFunction<U,? super T,U> accumulator, BinaryOperator<U> combiner)`

多的参数只是为了指明初始值（参数*identity*），或者是指定并行执行时多个部分结果的合并方式（参数*combiner*）

```java
// 找出最长的单词
Stream<String> stream = Stream.of("I", "love", "you", "too");
Optional<String> longest = stream.reduce((s1, s2) -> s1.length()>=s2.length() ? s1 : s2);
//Optional<String> longest = stream.max((s1, s2) -> s1.length()-s2.length());
System.out.println(longest.get());

// 求单词长度之和
Stream<String> stream = Stream.of("I", "love", "you", "too");
Integer lengthSum = stream.reduce(0,　// 初始值　// (1)
        (sum, str) -> sum+str.length(), // 累加器 // (2)
        (a, b) -> a+b);　// 部分和拼接器，并行执行时才会用到 // (3)
System.out.println(lengthSum);
```

#### collect

```java
// 将Stream转换成容器或Map
Stream<String> stream = Stream.of("I", "love", "you", "too");

List<String> list = stream.collect(Collectors.toList()); // (1)
Set<String> set = stream.collect(Collectors.toSet()); // (2)
Map<String, Integer> map = stream.collect(Collectors.toMap(Function.identity(), String::length)); // (3)
//Function.identity()返回一个输出跟输入一样的Lambda表达式对象，等价于形如t -> t形式的Lambda表达式。
```

