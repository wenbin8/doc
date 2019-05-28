

# 一个类搞定java8函数式编程

## 函数式接口与lambda表达式

函数式接口(Functional Interface)就是一个有且仅有一个抽象方法，但是可以有多个非抽象方法的接口。

函数式接口可以被隐式转换为 lambda 表达式。

Lambda 表达式和方法引用（实际上也可认为是Lambda表达式）上。

[参考[Java 8 函数式接口]](https://www.runoob.com/java/java8-functional-interfaces.html)

这里只是抛转引玉,本篇文章主要让讲解stream的使用.还有许多如,内循环\外循环, 及时求值\惰性求值等基础知识并未涉及.建议大家外部补充相关知识.

## Stream操作

### 练习集合

有了基础以后,还要有一个练习集合,该集合使用了<java8实战>中的菜谱.然后我们根据这个来练习java8中的Stream操作.

```java
    public static List<Dish> menu = Arrays.asList(
            new Dish("pork", false, 800, Type.MEAT),
            new Dish("beef", false, 700, Type.MEAT),
            new Dish("chicken", false, 400, Type.MEAT),
            new Dish("french fries", true, 530, Type.OHTER),
            new Dish("rice", true, 350, Type.OHTER),
            new Dish("season fruit", true, 120, Type.OHTER),
            new Dish("pizza", true, 550, Type.OHTER),
            new Dish("prawna", false, 300, Type.FISH),
            new Dish("salmon", false, 450, Type.FISH)
    );
```

Dish对象定义如下代码:

```java
private static class Dish {
    private final String name;
    private final boolean vegetarian;
    private final int calories;
    private final Type type;


    public Dish(String name, boolean vegetarian, int calories, Type type) {
        this.name = name;
        this.vegetarian = vegetarian;
        this.calories = calories;
        this.type = type;
    }

    public String getName() {
        return name;
    }

    public boolean isVegetarian() {
        return vegetarian;
    }

    public int getCalories() {
        return calories;
    }

    public Type getType() {
        return type;
    }

    @Override
    public String toString() {
        return "Dish{" +
                "name='" + name + '\'' +
                ", vegetarian=" + vegetarian +
                ", calories=" + calories +
                ", type=" + type +
                '}';
    }
}
```

----

### 筛选和切片

**filter**操作会接受一个谓词作为参数,并返回一个包括所有符合谓词元素的流.

```java
// filter 方法筛选出符合条件的元素,collect方法转换为新的集合
List<Dish> vegetarianMenu = menu.stream()
        .filter(Dish::isVegetarian)
        .collect(Collectors.toList());

vegetarianMenu.forEach(dish -> System.out.println(dish.getName()));
```

执行结果:
french fries
rice
season fruit
pizza

----

**distinct**返回一个元素各异(根据流所生成元素的hashCode和equals方法实现)的流.

```java
List<Integer> numbers = Arrays.asList(1, 2, 1, 3, 3, 2, 4);

numbers.stream()
        .filter(i -> i % 2 == 0)  // 获取偶数
        .distinct()// 去除重复
        .forEach(System.out::println);
```

执行结果:

2
4

-----

**limit**方法返回一个不超过给定长度的流.所需长度作为参数传递给limit.
如果流是有序的,则会返回前n个元素.
会短路即获取到足够的limit值后流不在继续内循环

```java
List<Dish> dishes = menu.stream()
        .filter(d -> d.getCalories() > 300)
        .limit(3)
        .collect(Collectors.toList());

dishes.forEach(dish -> System.out.println(dish));
```

执行结果:

Dish{name='pork', vegetarian=false, calories=800, type=MEAT}
Dish{name='beef', vegetarian=false, calories=700, type=MEAT}
Dish{name='chicken', vegetarian=false, calories=400, type=MEAT}

----

**skip**方法,跳过元素,返回一个 扔掉前n个元素的流.如果流中元素不足n个,则返回一个空流.
limit和skip是互补的.

```java
List<Dish> dishes = menu.stream()
        .filter(d -> d.getCalories() > 300)
        .skip(2)
        .collect(Collectors.toList());
dishes.forEach(dish -> System.out.println(dish));
```

执行结果:

Dish{name='chicken', vegetarian=false, calories=400, type=MEAT}
Dish{name='french fries', vegetarian=true, calories=530, type=OHTER}
Dish{name='rice', vegetarian=true, calories=350, type=OHTER}
Dish{name='pizza', vegetarian=true, calories=550, type=OHTER}
Dish{name='salmon', vegetarian=false, calories=450, type=FISH}

---

### 映射

**map**方法,接受一个函数作为参数.这个函数会被应用到每个元素上,并将其映射成一个新的元素映射和转换类似,映射结果是创建一个新的版本,而不是去修改.

```java
List<String> dishNames = menu.stream()
        .map(Dish::getName)
        .collect(Collectors.toList());

dishNames.forEach(s -> System.out.println(s));
```

执行结果:

pork
beef
chicken
french fries
rice
season fruit
pizza
prawna
salmon

从代码可以看出map方法把List<Dish> 集合使用Dish::getName 字段转换成了List<String>. map就是转换.下面一段代码尝试使用dish对象内的数据将List<Dish>转换为List<Value>

Value类的定义:

```java
public class Value {
    String key;

    int number;


    public Value(String key, int number) {
        this.key = key;
        this.number = number;
    }

    public String getKey() {
        return key;
    }

    public void setKey(String key) {
        this.key = key;
    }

    public int getNumber() {
        return number;
    }

    public void setNumber(int number) {
        this.number = number;
    }

    @Override
    public String toString() {
        return "Value{" +
                "key='" + key + '\'' +
                ", number=" + number +
                '}';
    }
}
```

转换代码:

```java
List<Value> dishValueList = menu.stream()
        .map(dish -> new Value(dish.getName(), dish.getCalories()))
        .collect(Collectors.toList());

dishValueList.forEach(value -> System.out.println(value));
```

执行结果:

Value{key='pork', number=800}
Value{key='beef', number=700}
Value{key='chicken', number=400}
Value{key='french fries', number=530}
Value{key='rice', number=350}
Value{key='season fruit', number=120}
Value{key='pizza', number=550}
Value{key='prawna', number=300}
Value{key='salmon', number=450}

使用List元素的属性进行转换:

```java
List<String> words = Arrays.asList("Java 8", "Lambdas", "In", "Action");
List<Integer> wordLengths = words.stream()
        .map(String::length)
        .collect(Collectors.toList());

wordLengths.forEach(integer -> System.out.println(integer));
```

执行结果:

6
7
2
6

使用map方法实现给定数字列表返回列表数的平方构成的列表

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

List<Integer> squares = numbers.stream()
        .map(n -> n * n)
        .collect(Collectors.toList());

squares.forEach(s -> System.out.println(s));
```

### 流的扁平化flatMap方法

```java
List<String> words = Arrays.asList("Hello", "World");

List<String[]> list = words.stream()
        .map(word -> word.split(""))
        .distinct()
        .collect(Collectors.toList());
list.forEach(s -> System.out.println(s.length));
```

执行结果:

5
5

上面的代码本意是想获取words集合中的不重复的字母.所以在map方法中使用word.split("")转换成["H","e","l","l","o","W","o","r","l","d"]的形式,在使用distinct方法进行去重.但是使用map方法实际会将集合转换为[["H","e","l","l","o"],["W","o","r","l","d"]]这样个二维数组的形式.这显然是不对的.这种情况应该使用**flatMap**方法将转换后的多个流,合并成为一个如下面代码:

```java
List<String> words = Arrays.asList("Hello", "World");
List<String> list = words.stream()
        .map(w -> w.split(""))
        .flatMap(Arrays::stream)
        .distinct()
        .collect(Collectors.toList());

list.forEach(s -> System.out.println(s));
```

执行结果:

H
e
l
o
W
r
d

这样就得到了我们想要的结果.

### 查找和匹配

