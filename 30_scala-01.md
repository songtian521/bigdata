# 30_scala-01

# 1.变量

1. 语法格式

   在scala中，可以使用var或val来定义变量

   ```scala
   var/val 变量标识:变量类型 = 初始值
   ```

   其中

   - val定义的是**不可重新赋值**的变量
   - var定义的是**可重新赋值**的变量

   ```scala
    var name:String = "tom";
     name = "a";
     val age:Int = 1;
     age = 2;//报错
   
   ```

   注意：

   - scala中定义变量类型写在变量名后面
   - scala的语句最后可以不需要添加分号结尾

   示例：

   ```scala
   var name:String = "tom"
   ```

2. 简写变量

   scala可以根据自动根据变量的值来自动推断变量的类型，这样可以使scala的代码更加简洁

   示例：

   ```scala
   var name = "tom"
   ```

3. 惰性赋值

   在企业的大数据开发中，有时候会编写非常复杂的SQL语句，这些SQL语句可能有几百行甚至上千行。这些SQL语句，如果直接加载到JVM中，会有很大的内存开销。如何解决？

   示例：

   ```scala
   lazy val sql = """insert overwrite table adm.itcast_adm_personas
        |     select
        |     a.user_id,
   	....
        |     left join gdm.itcast_gdm_user_buy_category c on a.user_id=c.user_id
        |     left join gdm.itcast_gdm_user_visit d on a.user_id=d.user_id;"""
   ```

# 2.字符串

scala提供了多种定义字符串的方式，我们可以根据需要来选择最方便的定义方式

- 使用双引号

  ```scala
  val/var 变量名 = "字符串"
  ```

- 使用插值表达式

  ```scala
  val/var 变量名 = s"${变量/表达式}字符串"
  示例：
  val name = "zhangsan";
  val age = 30;
  val sex = "male"
  val info = s"name=${name}, age=${age}, sex=${sex}";
  输出：
  name=zhangsan, age=30, sex=male
  ```

- 使用三引号

  如果有大段的文本需要保存，就可以使用三引号来定义字符串。例如：保存一大段的SQL语句。三个引号中间的所有字符串都将作为字符串的值。

  ```scala
  val/var 变量名 = """字符串1
  字符串2"""
  
  示例：
  val sql = """select
       | *
       | from
       |     t_user
       | where
       |     name = "zhangsan""""
  
  println(sql)
  
  ```

# 3.数据类型

| 基础类型 | 类型说明                 |
| -------- | ------------------------ |
| Byte     | 8位带符号整数            |
| Short    | 16位带符号整数           |
| **Int**  | 32位带符号整数           |
| Long     | 64位带符号整数           |
| Char     | 16位无符号Unicode字符    |
| String   | Char类型的序列（字符串） |
| Float    | 32位单精度浮点数         |
| Double   | 64位双精度浮点数         |
| Boolean  | true或false              |

注：

- scala中所有的类型都是用大写字母开头
- 整型使用Int而不是Integer
- scala中定义变量可以不写类型，让scala编译器自动推断

# 4.运算符

| 类别       | 操作符               |
| ---------- | -------------------- |
| 算术运算符 | +、-、*、/           |
| 关系运算符 | >、<、==、!=、>=、<= |
| 逻辑运算符 | &&、\|\|、!          |
| 位运算符   | &、\|\|、^、<<、>>   |

注：

- scala中没有 ++、--运算符
- 与java不一样，在scala中，可以直接使用` ==`、`!= `进行比较，他们与`equals`方法表示一致。而比较两个对象的引用值，使用`eq`

示例：

```scala
  val str1 = "abc"
  val str2 = str1 + ""
  println(str1 == str2)//true
  println(str1.eq(str2))//false
```

# 5.scala类型层次结构

![](img\scala\类型层次结构.png)

| 类型    | 说明                                                         |
| ------- | ------------------------------------------------------------ |
| Any     | **所有类型**的父类，,它有两个子类AnyRef与AnyVal              |
| AnyVal  | **所有数值类型**的父类                                       |
| AnyRef  | 所有对象类型（引用类型）的父类                               |
| Unit    | 表示空，Unit是AnyVal的子类，它只有一个的实例{% em %}() {% endem %} 它类似于Java中的void，但scala要比Java更加面向对象 |
| Null    | Null是AnyRef的子类，也就是说它是所有引用类型的子类。它的实例是{% em %}**null**{% endem %} 可以将null赋值给任何对象类型 |
| Nothing | 所有类型的**子类** 不能直接创建该类型实例，某个方法抛出异常时，返回的就是Nothing类型，因为Nothing是所有类的子类，那么它可以赋值为任何类型 |

示例：

```scala
def main(args: Array[String]): Unit = {
    val c = m3(1,0)
  }

  def m3(x:Int, y:Int):Int = {
    if(y == 0) throw new Exception("这是一个异常")
    x / y
  }
  
输出：
Exception in thread "main" java.lang.Exception: 这是一个异常
```

注：

```
val b:Int = null //报错
```

原因：Null类型并不能转换为Int类型，说明**Null类型并不是Int类型的子类**

# 6.表达式

## 6.1 条件表达式

条件表达式就是if表达式，if表达式可以根据给定的条件是否满足，根据条件的结果（真或假）决定执行对应的操作。scala条件表达式的语法和Java一样。

**注：**

与java不一样的是

- 在scala中，条件表达式也是有返回值的
- 在scala中，没有三元表达式，可以使用if表达式替代三元表达式

示例：

```scala
    val sex = "male"
    val result = if(sex == "male") 1 else 0
    print(result)//1
```

## 6.2 块表达式

- scala中，使用{}表示一个块表达式

- 和if表达式一样，块表达式也是有值的
- 值就是最后一个表达式的值

示例：

```scala
var a = {
      print("aaa")
    }
```

# 7.循环

在scala中，可以使用for和while，但一般推荐使用for表达式，因为for表达式语法更简洁

## 7.1 for表达式

 语法：

```scala
for(i <- 表达式/数组/集合) {
    // 表达式
}
```

示例：

```scala
for (i<- 1 to 10){
      print(i)
}
```

## 7.2 守卫

for表达式中，可以添加if判断语句，这个if判断就称之为守卫。我们可以使用守卫让for表达式更简洁。

语法：

```scala
for(i <- 表达式/数组/集合 if 表达式) {
    // 表达式
}
```

示例：

```scala
// 添加守卫，打印能够整除3的数字
for(i <- 1 to 10 if i % 3 == 0) println(i)
```

## 7.3 推导式

- 将来可以使用for推导式生成一个新的集合（一组数据）
- 在for循环体中，可以使用`yield`表达式构建出一个集合，我们把使用yield的for表达式称之为推导式

示例：

```scala
// for推导式：for表达式中以yield开始，该for表达式会构建出一个集合
 val v = for(i <- 1 to 10) yield i * 10
    print(v)

输出：
Vector(10, 20, 30, 40, 50, 60, 70, 80, 90, 100)
```

## 7.4 while

scala中while循环和Java中是一致的

示例：

```scala
 var i = 1;
    while(i <= 10) {
       println(i)
       i = i+1
    }
```

do while

```scala
var j = 0;
do {
	println(list(j))
	j+=1
}while(
	j < list.length
)
```



## 7.5 break和continue

- 在scala中，类似Java和C++的break/continue关键字被移除了
- 如果一定要使用break/continue，就需要使用scala.util.control包的Break类的**breable**和**break**方法。

### 7.5.1实现break

**用法：**

- 导入Breaks包`import scala.util.control.Breaks._`
- 使用breakable将for表达式包起来
- for表达式中需要退出循环的地方，添加`break()`方法调用

示例：

```scala
// 导入scala.util.control包下的Break
import scala.util.control.Breaks._

breakable{
    for(i <- 1 to 100) {
        if(i >= 50) break()
        else println(i)
    }
}
```

### 7.5.2 实现continue

**用法：**continue的实现与break类似，但有一点不同：

实现break是用breakable{}将整个for表达式包起来，而实现continue是用breakable{}将for表达式的循环体包含起来就可以了

示例：

```scala
// 导入scala.util.control包下的Break    
import scala.util.control.Breaks._

for(i <- 1 to 100 ) {
    breakable{
        if(i % 10 == 0) break()
        else println(i)
    }
}
```

# 8.方法

一个类可以有自己的方法，scala中的方法和Java方法类似。但scala与Java定义方法的语法是不一样的。

## 8.1 定义方法

语法：

```scala
def methodName (参数名:参数类型, 参数名:参数类型) : [return type] = {
    // 方法体：一系列的代码
}
```

注：

- 参数列表的参数类型不能省略
- 返回值类型可以省略，由scala编译器自动推断
- 返回值可以不写return，默认就是{}块表达式的值

示例：

```scala
  def add(a:Int, b:Int) = a + b
    print(add(1,2))//3
```

## 8.2 返回值类型判断

scala定义方法可以省略返回值，由scala自动推断返回值类型。这样方法定义后更加简洁。

注：定义递归方法，**不能省略返回值类型**

示例：

```scala
	print( m2(10))
    def m2(x:Int):Int = {
       if(x<=1)1 else m2(x-1) * x
    }
```

## 8.3 方法参数

scala中的方法参数，使用比较灵活。它支持以下几种类型的参数：

- 默认参数
- 带名参数
- 变长参数

1. 默认参数

   在定义方法时可以给参数定义一个默认值。

   示例：

   ```scala
   def add(x:Int = 0, y:Int = 0) = x + y
   add()
   ```

2. 带名参数

   在调用方法时，可以指定参数的名称来进行调用。

   示例：

   ```scala
   def add(x:Int = 0, y:Int = 0) = x + y
   add(x=1)
   ```

3. 变长参数

   如果方法的参数是不固定的，可以定义一个方法的参数是变长参数。

   语法格式：

   ```scala
   def 方法名(参数名:参数类型*):返回值类型 = {
       方法体
   }
   ```

   在参数类型后面加一个`*`号，表示参数可以是0个或者多个

   示例：

   ```scala
   def add(num:Int*) = num.sum
   add(1,2,3,4,5)
   ```

## 8.4 方法调用方式

在scala中，有以下几种方法调用方式，

- 后缀调用法
- 中缀调用法
- 花括号调用法
- 无括号调用法

在后续编写spark、flink程序时，我们会使用到这些方法调用方式。

1. 后缀调用法

   语法：

   ```scala
   对象名.方法名(参数)
   ```

   示例：

   使用后缀法`Math.abs`求绝对值

   ```scala
   Math.abs(-1)
   ```

2. 中缀调用法

   语法：

   ```
   对象名 方法名 参数
   ```

   如果有多个参数，使用括号括起来

   示例：

   ```
   Math abs -1
   ```

3. 操作符即方法

   在scala中，+ - * / %等这些操作符和Java一样，但在scala中，

   - 所有的操作符都是方法
   - 操作符是一个方法名字是符号的方法

4. 花括号调用法

   语法：

   ```scala
   Math.abs{ 
       // 表达式1
       // 表达式2
   }
   ```

   注：方法只有一个参数，才能使用花括号调用法

   示例：

   ```
   Math.abs{-10}
   ```

5. 无括号调用法

   如果方法没有参数，可以省略方法名后面的括号

   示例：

   ```scala
   def m3()=println("hello")
   m3()
   ```

# 9.函数

scala支持函数式编程，将来编写Spark/Flink程序中，会大量使用到函数

## 9.1 定义函数

语法：

```scala
val 函数变量名 = (参数名:参数类型, 参数名:参数类型....) => 函数体
```

注：

- 函数是一个**对象**（变量）
- 类似于方法，函数也有输入参数和返回值
- 函数定义不需要使用`def`定义
- 无需指定返回值类型

示例：

```scala
 val add = (x:Int, y:Int) => x + y
```

## 9.2 方法和函数的区别

- 方法是隶属于类或者对象的，在运行时，它是加载到JVM的方法区中
- 可以将函数对象赋值给一个变量，在运行时，它是加载到JVM的堆内存中
- 函数是一个对象，继承自FunctionN，函数对象有apply，curried，toString，tupled这些方法。方法则没有

注：无法将函数赋值给变量

## 9.3 方法转换为函数

- 有时候需要将方法转换为函数，作为变量传递，就需要将方法转换为函数
- 使用`_`即可将方法转换为函数

示例：

```
def add(x:Int,y:Int)=x+y
val a = add _
```

# 10.数组

scala中数组的概念是和Java类似，可以用数组来存放一组数据。scala中，有两种数组，一种是**定长数组**，另一种是**变长数组**

## 10.1 定长数组

- 定长数组指的是数组的**长度**是**不允许改变**的
- 数组的**元素**是**可以改变**的

语法：

```scala
// 通过指定长度定义数组
val/var 变量名 = new Array[元素类型](数组长度)

// 用元素直接初始化数组
val/var 变量名 = Array(元素1, 元素2, 元素3...)
```

注：

- 在scala中，数组的泛型使用`[]`来指定
- 使用`()`来获取元素

示例1：

```scala
val a = new Array[Int](100)
a(0) = 110
println(a(0))// 110
```

示例2：

```
 val a = Array("java", "scala", "python")
 a.length //3
```

## 10.2 变长数组

变长数组指的是数组的长度是可变的，可以往数组中添加、删除元素

**定义：**

创建变长数组，需要提前导入ArrayBuffer类`import scala.collection.mutable.ArrayBuffer`

语法：

- 创建空的ArrayBuffer变长数组，语法结构

  ```
  val/var a = ArrayBuffer[元素类型]()
  ```

- 创建带有初始元素的ArrayBuffer

  ```
  val/var a = ArrayBuffer(元素1，元素2，元素3....)
  ```

示例1：

定义一个长度为0的整型变长数组

```
val a = ArrayBuffer[Int]()
```

示例2：

定义一个包含任意元素的变长数组

```
 val a = ArrayBuffer("hadoop", "storm", "spark")
```

## 10.3 添加/修改/删除元素

- 使用`+=`添加元素
- 使用`-=`删除元素
- 使用`++=`追加一个数组到变长数组

示例：

```scala
// 定义变长数组
 val a = ArrayBuffer("hadoop", "spark", "flink")

// 追加一个元素
 a += "flume"

// 删除一个元素
 a -= "hadoop"

// 追加一个数组
scala> a ++= Array("hive", "sqoop")
```

## 10.4 遍历数组

可以使用以下两种方式来遍历数组：

- 使用`for表达式`直接遍历数组中的元素

- 使用`索引`遍历数组中的元素

示例1：

```scala
val a = Array(1,2,3,4,5)
for(i<-a) println(i)
```

示例2：

```scala
val a = Array(1,2,3,4,5)
for(i <- 0 to a.length - 1) println(a(i))
for(i <- 0 until a.length) println(a(i))
```

注：

- 0 until n——生成一系列的数字，包含0，不包含n
- 0 to n ——包含0，也包含n

## 10.5 数组常用算法

scala中的数组封装了一些常用的计算操作，将来在对数据处理的时候，不需要我们自己再重新实现。以下为常用的几个算法：

* 求和——sum方法

- 求最大值——max方法
- 求最小值——min方法
- 排序——sorted方法

**示例1：求和**

```
val a = Array(1,2,3,4)
println(a.sum)
```

**示例2：最大值**

数组中的`max`方法，可以获取到数组中的最大的那个元素值

```
 val a = Array(4,1,2,4,10)
 println(a.max)
```

**示例3：最小值**

数组的`min`方法，可以获取到数组中最小的那个元素值

```
val a = Array(4,1,2,4,10)
println(a.min)
```

**示例4：排序**

数组的`sorted`方法，可以对数组进行升序排序。而`reverse`方法，可以将数组进行反转，从而实现降序排序

```scala
// 升序排序
scala> a.sorted
res53: Array[Int] = Array(1, 2, 4, 4, 10)

// 降序
scala> a.sorted.reverse
res56: Array[Int] = Array(10, 4, 4, 2, 1)
```

## 10.6 元组

元组可以用来包含一组不同类型的值。例如：姓名，年龄，性别，出生年月。元组的元素是不可变的。

**定义元组**

语法：

```scala
val/var 元组 = (元素1, 元素2, 元素3....)
```

使用箭头来定义元组（元组只有两个元素）

```scala
val/var 元组 = 元素1->元素2
```

示例1：

```
val a = (1, "zhangsan", 20, "beijing")
```

示例2：

```scala
scala> val a = ("zhangsan", 20)
a: (String, Int) = (zhangsan,20)

scala> val a = "zhangsan" -> 20
a: (String, Int) = (zhangsan,20)
```

## 10.7 访问元组

使用\_1、\_2、\_3....来访问元组中的元素，_1表示访问第一个元素，依次类推

示例：

```scala
scala> val a = "zhangsan" -> "male"
a: (String, String) = (zhangsan,male)

// 获取第一个元素
scala> a._1
res41: String = zhangsan

// 获取第二个元素
scala> a._2
res42: String = male
```

# 11.列表

列表是scala中最重要的、也是最常用的数据结构。List具备以下性质：

- 可以保存重复的值
- 有先后顺序

在scala中，也有两种列表，一种是不可变列表、另一种是可变列表

## 11.1 不可变列表

不可变列表就是列表的元素、长度都是不可变的。

语法：

使用`List(元素1, 元素2, 元素3, ...)`来创建一个不可变列表，语法格式：

```scala
val/var 变量名 = List(元素1, 元素2, 元素3...)
```

使用`Nil`创建一个不可变的空列表

```scala
val/var 变量名 = Nil
```

使用`::`方法创建一个不可变列表

```scala
val/var 变量名 = 元素1 :: 元素2 :: Nil
```

注：使用`::`拼接方式来创建列表，必须在最后添加一个**Nil**

示例1：

```
val a = List(1,2,3,4)
```

示例2：

```
val a = Nil
```

示例3：

```
val a = -2 :: -1 :: Nil
```

## 11.2 可变列表

可变列表就是列表的元素、长度都是可变的。

要使用可变列表，先要导入`import scala.collection.mutable.ListBuffer`

注：

- 可变集合都在`mutable`包中
- 不可变集合都在`immutable`包中（默认导入）

语法：

使用ListBuffer\[元素类型\]()创建空的可变列表，语法结构：

```scala
val/var 变量名 = ListBuffer[Int]()
```

使用ListBuffer(元素1, 元素2, 元素3...)创建可变列表，语法结构：

```scala
val/var 变量名 = ListBuffer(元素1，元素2，元素3...)
```

示例1：

```
val a = ListBuffer[Int]()
```

示例2：

```
 val a = ListBuffer(1,2,3,4)
```

## 11.3 可变列表操作

- 获取元素（使用括号访问`(索引值)`）
- 添加元素（`+=`）
- 追加一个列表（`++=`）
- 更改元素（`使用括号获取元素，然后进行赋值`）
- 删除元素（`-=`）
- 转换为List（`toList`）
- 转换为Array（`toArray`）

示例：

```
// 导入不可变列表
scala> import scala.collection.mutable.ListBuffer
import scala.collection.mutable.ListBuffer

// 创建不可变列表
scala> val a = ListBuffer(1,2,3)
a: scala.collection.mutable.ListBuffer[Int] = ListBuffer(1, 2, 3)

// 获取第一个元素
scala> a(0)
res19: Int = 1

// 追加一个元素
scala> a += 4
res20: a.type = ListBuffer(1, 2, 3, 4)

// 追加一个列表
scala> a ++= List(5,6,7)
res21: a.type = ListBuffer(1, 2, 3, 4, 5, 6, 7)

// 删除元素
scala> a -= 7
res22: a.type = ListBuffer(1, 2, 3, 4, 5, 6)

// 转换为不可变列表
scala> a.toList
res23: List[Int] = List(1, 2, 3, 4, 5, 6)

// 转换为数组
scala> a.toArray
res24: Array[Int] = Array(1, 2, 3, 4, 5, 6)
```

## 11.4 列表常用操作

- 判断列表是否为空（`isEmpty`）

  ```
  val a = List(1,2,3,4)
  a.isEmpty
  ```

- 拼接两个列表（`++`）

  ```
  val a = List(1,2,3)
  val b = List(4,5,6)
  print( a++ b);//List(1, 2, 3, 4, 5, 6)
  ```

- 获取列表的首个元素（`head`）和剩余部分(`tail`)

  ```
   	val a = List(1,2,3)
      println(a.head)//1
      println(a.tail)//List(2, 3)
  ```

- 反转列表（`reverse`）

  ```
  val a = List(1,2,3)
  println( a.reverse)//List(3, 2, 1)
  ```

- 获取前缀（`take`）、获取后缀（`drop`）

  ```
  scala> val a = List(1,2,3,4,5)
  a: List[Int] = List(1, 2, 3, 4, 5)
  
  scala> a.take(3)
  res56: List[Int] = List(1, 2, 3)
  
  scala> a.drop(3)
  res60: List[Int] = List(4, 5)
  ```

- 扁平化（`flaten`）

  扁平化表示将列表中的列表中的所有元素放到一个列表中。

  ```
  scala> val a = List(List(1,2), List(3), List(4,5))
  a: List[List[Int]] = List(List(1, 2), List(3), List(4, 5))
  
  scala> a.flatten
  res0: List[Int] = List(1, 2, 3, 4, 5)
  ```

- 拉链（`zip`）和拉开（`unzip`）

  * 拉链：使用zip将两个列表，组合成一个元素为元组的列表

    ```
    scala> val a = List("zhangsan", "lisi", "wangwu")
    a: List[String] = List(zhangsan, lisi, wangwu)
    
    scala> val b = List(19, 20, 21)
    b: List[Int] = List(19, 20, 21)
    
    scala> a.zip(b)
    res1: List[(String, Int)] = List((zhangsan,19), (lisi,20), (wangwu,21))
    ```

    

  * 拉开：将一个包含元组的列表，解开成包含两个列表的元组

    ```
    scala> res1.unzip
    res2: (List[String], List[Int]) = (List(zhangsan, lisi, wangwu),List(19, 20, 21))
    ```

    

- 转换字符串（`toString`）

  toString方法可以返回List中的所有元素

  ```
  scala> val a = List(1,2,3,4)
  a: List[Int] = List(1, 2, 3, 4)
  
  scala> println(a.toString)
  List(1, 2, 3, 4)
  ```

  

- 生成字符串（`mkString`）

  mkString方法，可以将元素以分隔符拼接起来。默认没有分隔符

  ```
  scala> val a = List(1,2,3,4)
  a: List[Int] = List(1, 2, 3, 4)
  
  scala> a.mkString
  res7: String = 1234
  
  scala> a.mkString(":")
  res8: String = 1:2:3:4
  ```

  

- 并集（`union`）

  union表示对两个列表取并集，不去重

  ```
  scala> val a1 = List(1,2,3,4)
  a1: List[Int] = List(1, 2, 3, 4)
  
  scala> val a2 = List(3,4,5,6)
  a2: List[Int] = List(3, 4, 5, 6)
  
  // 并集操作
  scala> a1.union(a2)
  res17: List[Int] = List(1, 2, 3, 4, 3, 4, 5, 6)
  
  // 可以调用distinct去重
  scala> a1.union(a2).distinct
  res18: List[Int] = List(1, 2, 3, 4, 5, 6)
  ```

  

- 交集（`intersect`）

  intersect表示对两个列表取交集

  ```
  scala> val a1 = List(1,2,3,4)
  a1: List[Int] = List(1, 2, 3, 4)
  
  scala> val a2 = List(3,4,5,6)
  a2: List[Int] = List(3, 4, 5, 6)
  
  scala> a1.intersect(a2)
  res19: List[Int] = List(3, 4)
  ```

  

- 差集（`diff`）

  iff表示对两个列表取差集，例如： a1.diff(a2)，表示获取a1在a2中不存在的元素

  ```
  scala> val a1 = List(1,2,3,4)
  a1: List[Int] = List(1, 2, 3, 4)
  
  scala> val a2 = List(3,4,5,6)
  a2: List[Int] = List(3, 4, 5, 6)
  
  scala> a1.diff(a2)
  res24: List[Int] = List(1, 2)
  ```

# 12.集合

## 12.1 Set 集合

Set(集)是代表没有重复元素的集合。Set具备以下性质：

1. 元素不重复
2. 不保证插入顺序

scala中的集也分为两种，一种是不可变集，另一种是可变集。

### 12.1.1 set不可变集

语法：

创建一个空的不可变集，语法格式：

```
val/var 变量名 = Set[类型]()
```

给定元素来创建一个不可变集，语法格式：

```
val/var 变量名 = Set(元素1, 元素2, 元素3...)
```

示例1：定义一个空的不可变集

```
scala> val a = Set[Int]()
a: scala.collection.immutable.Set[Int] = Set()
```

示例2：定义一个不可变集，保存以下元素：1,1,3,2,4,8

```
scala> val a = Set(1,1,3,2,4,8)
a: scala.collection.immutable.Set[Int] = Set(1, 2, 3, 8, 4)
```

注：默认创建的都是不可变集合，同样可以这样直接创建不可变集合

```scala
var a = scala.collection.immutable.Set(1,2,3,4)
```

### 12.1.2 set基本操作

- 获取集的大小（`size`）
- 遍历集（`和遍历数组一致`）
- 添加一个元素，生成一个Set（`+`）
- 拼接两个集，生成一个Set（`++`）
- 拼接集和列表，生成一个Set（`++`）

```scala
// 创建集
scala> val a = Set(1,1,2,3,4,5)
a: scala.collection.immutable.Set[Int] = Set(5, 1, 2, 3, 4)

// 获取集的大小
scala> a.size
res0: Int = 5

// 遍历集
scala> for(i <- a) println(i)

// 删除一个元素
scala> a - 1
res5: scala.collection.immutable.Set[Int] = Set(5, 2, 3, 4)

// 拼接两个集
scala> a ++ Set(6,7,8)
res2: scala.collection.immutable.Set[Int] = Set(5, 1, 6, 2, 7, 3, 8, 4)

// 拼接集和列表
scala> a ++ List(6,7,8,9)
res6: scala.collection.immutable.Set[Int] = Set(5, 1, 6, 9, 2, 7, 3, 8, 4)

//直接创建有序集合
var v = SortedSet(2,4,1,3)
println(v)
```

### 12.3 set 可变集

可变集合不可变集的创建方式一致，只不过需要提前导入一个可变集类。

手动导入：`import scala.collection.mutable.Set`

```scala
scala> val a = Set(1,2,3,4)
a: scala.collection.mutable.Set[Int] = Set(1, 2, 3, 4)                          

// 添加元素
scala> a += 5
res25: a.type = Set(1, 5, 2, 3, 4)

// 删除元素
scala> a -= 1
res26: a.type = Set(5, 2, 3, 4)
```

注：同样可以这样直接创建可变集合

```scala
var a = scala.collection.mutable.Set(1,2,3,4)
```



## 12.2 Map集合

Map可以称之为映射。它是由键值对组成的集合。在scala中，Map也分为不可变Map和可变Map。

### 12.2.1 不可变Map集合

语法：

```
val/var map = Map(键->值, 键->值, 键->值...)	// 推荐，可读性更好
val/var map = Map((键, 值), (键, 值), (键, 值), (键, 值)...)
```

示例：

```scala
scala> val map = Map("zhangsan"->30, "lisi"->40)
map: scala.collection.immutable.Map[String,Int] = Map(zhangsan -> 30, lisi -> 40)

scala> val map = Map(("zhangsan", 30), ("lisi", 30))
map: scala.collection.immutable.Map[String,Int] = Map(zhangsan -> 30, lisi -> 30)

// 根据key获取value
scala> map("zhangsan")
res10: Int = 30
```

### 12.2.2 可变Map

定义：

定义语法与不可变Map一致。但定义可变Map需要手动导入`import scala.collection.mutable.Map`

示例：

```scala
scala> val map = Map("zhangsan"->30, "lisi"->40)
map: scala.collection.mutable.Map[String,Int] = Map(lisi -> 40, zhangsan -> 30)

// 修改value
scala> map("zhangsan") = 20
```

### 12.2.3 Map基本操作

- 获取值(`map(key)`)
- 获取所有key（`map.keys`）
- 获取所有value（`map.values`）
- 遍历map集合
- getOrElse
- 增加key,value对
- 删除key

```scala
scala> val map = Map("zhangsan"->30, "lisi"->40)
map: scala.collection.mutable.Map[String,Int] = Map(lisi -> 40, zhangsan -> 30)

// 获取zhagnsan的年龄
scala> map("zhangsan")
res10: Int = 30

// 获取所有的学生姓名
scala> map.keys
res13: Iterable[String] = Set(lisi, zhangsan)

// 获取所有的学生年龄
scala> map.values
res14: Iterable[Int] = HashMap(40, 30)

// 打印所有的学生姓名和年龄
scala> for((x,y) <- map) println(s"$x $y")
lisi 40
zhangsan 30

// 获取wangwu的年龄，如果wangwu不存在，则返回-1
scala> map.getOrElse("wangwu", -1)
res17: Int = -1

// 新增一个学生：wangwu, 35
scala> map + "wangwu"->35
res22: scala.collection.mutable.Map[String,Int] = Map(lisi -> 40, zhangsan -> 30, wangwu -> 35)

// 将lisi从可变映射中移除
scala> map - "lisi"
res23: scala.collection.mutable.Map[String,Int] = Map(zhangsan -> 30)
```

# 13.迭代器

scala针对每一类集合都提供了一个迭代器（iterator）用来迭代访问集合

**使用迭代器遍历集合**

- 使用`iterator`方法可以从集合获取一个迭代器
- 迭代器的两个基本操作
  - hasNext——查询容器中是否有下一个元素
  - next——返回迭代器的下一个元素，如果没有，抛出NoSuchElementException
- 每一个迭代器都是有状态的
  - 迭代完后保留在最后一个元素的位置
  - 再次使用则抛出NoSuchElementException
- 可以使用while或者for来逐个返回元素

示例1：

```scala
scala> val ite = a.iterator
ite: Iterator[Int] = non-empty iterator

scala> while(ite.hasNext) {
     | println(ite.next)
     | }
```

示例2：

```scala
scala> val a = List(1,2,3,4,5)
a: List[Int] = List(1, 2, 3, 4, 5)

scala> for(i <- a) println(i)
```

# 14.序列

定义：

```scala
var v = Vector(1,2,3,4,5)//不常用
print(Range(0,5));//常用，输出 Range(0, 1, 2, 3, 4)
println(0 to 5)//输出：Range(0, 1, 2, 3, 4, 5)
```

