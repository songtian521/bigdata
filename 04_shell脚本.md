# shell脚本

## 1.简介

- shell脚本执行方式Shell 是一个用 C 语言编写的程序，通过 Shell 用户可以访问操作系统内核服务。它类似于 DOS 下的 command 和后来的 cmd.exe。Shell 既是一种命令语言，又是一种程序设计语言。

- Shell script 是一种为 shell 编写的脚本程序。Shell 编程一般指 shell脚本编程，不是指开发 shell 自身。

- Shell 编程跟 java、php 编程一样，只要有一个能编写代码的文本编辑器和一个能解释执行的脚本解释器就可以了。

- Linux 的 Shell 种类众多，一个系统可以存在多个 shell，可以通过 cat /etc/shells 命令查看系统中安装的 shell。

- Bash 由于易用和免费，在日常工作中被广泛使用。同时，Bash 也是大多数Linux 系统默认的 Shell。

## 2.快速入门

### 1.编写脚本

新建一个`.sh`文件

内容如下：

eg：

```shell
#! /bin/bash
echo 'helloworld'
```

注意：#！是一个约定的标记，它告诉系统这个脚本需要什么解释器来执行，不论用哪一种shell 

echo 命令用于向窗口输出文本

### 2.解释器

概念：类似java解释器，shell也需要解释器

### 3.执行shell脚本方式

1. ```shell
   [root@linux a]# /bin/sh a.sh 
   hello world
   [root@linux a]# /bin/bash a.sh
   hello world
   ```

   问题：sh和bash的关系？答案：sh是bash的快捷方式

2. ```shell
   [root@linux a]# bash a.sh 
   hello world
   [root@linux a]# sh a.sh 
   hello world
   ```

   思考：为什么可以省略bin

   答案：因为PATH环境变量中增加了bin目录所以使用/bin/sh等类似命令时，可以省略/bin

3. ```shell
   [root@linux a]# ./a.sh
   -bash: ./a.sh: 权限不够
   ```

   权限不够怎么办？

   答案：执行命令`chmod +x 文件`

   ```shell
   [root@linux a]# chmod +x a.sh
   [root@linux a]# ./a.sh 
   hello world
   ```

## 3.shell 变量

1. 定义变量

   ```shell
   name="宋天"
   ```

   注意：变量名和等号之间不能有空格

   命名规则：

   - 命名只能使用英文字母，数字和下划线，首字母不能以数字开头
   - 中间不能有空格，可以使用下划线
   - 不能使用标点符号
   - 不能使用shell关键字

2. 使用变量

   ```shell
   #!/bin/bash
   name="宋天"
   echo $name
   echo ${name}
   ```

   注意：

   - 变量名的大括号是可以省略不写的，添加只是为了解释器识别变量的边界
   - 变量名可以被重新定义

3. 只读变量

   使用readonly变量可以将变量定义为只读变量，只读变量的值不能被改变

   eg：

   ```shell
   #!/bin/bash
   name="宋天"
   readonly name
   echo $name
   name = “宋小天”
   echo ${name}
   ~                   
   ```

   

   ```shell
   [root@linux a]# ./a.sh 
   宋天
   ./a.sh:行5: name: 未找到命令
   宋天
   ```

4. 删除变量

   命令 unset 

   ```shell
   #!/bin/bash
   name="宋天"
   unset name
   echo $nmeame #输出为空
   ```

   注意：变量删除后不能再次使用，且unset变量不能删除只读变量

   ```shell
   #!/bin/bash
   name="宋小天"
   readonly name
   unset name
   echo $name            
   ```

   输出结果：

   ```shell
   [root@linux a]# ./a.sh 
   ./a.sh: 第 5 行:unset: name: 无法反设定: 只读 variable
   宋小天
   ```

## 4.字符串

字符串是shell编程中最常用的数据类型，字符串可以使单引号，也可以是双引号，也可以不用引号

1. 单引号

   ```shell
   #!/bin/bash
   name='宋天'
   str='LOVE$name'
   echo $str
   ```

   输出

   ```shell
   [root@linux a]# ./a.sh 
   LOVE$name
   ```

   单引号的限制：

   - 单引号里的任何字符都会原样输出，单引号字符串中的变量是无效的
   - 单引号字符串中不能出现单独一个单引号（转义后的单引号也不行），但是可以成对出现，作为字符串拼接使用

2. 双引号

   ```shell
   #!/bin/bash
   name="宋天"
   str="LOVE$name"
   echo $str
   ```

   输出：

   ```shell
   [root@linux a]# ./a.sh 
   LOVE宋天
   ```

   双引号的优点：

   - 双引号里面可以有变量
   - 双引号里面可以出现转义字符

3. 拼接字符串

   - 使用双引号拼接

     ```shell
     #!/bin/bash
     name="宋天"
     str="LOVE,$name"
     str1="LOVE,$name"."
     str2="LOVE,\"$name"."
     echo $str 
     echo $str1
     echo $str2  
     ```

     输出：

     ```shell
     [root@linux a]# ./a.sh 
     LOVE,宋天
     LOVE,宋天. str2=LOVE,"宋天.
     ```

   - 使用单引号拼接

     ```shell
     #!/bin/bash
     name="宋天"
     str='LOVE,$name'
     str1='LOVE,$name.''
     str2='LOVE,\'$name'.'
     str3='LOVE,\"$name".'
     echo $str 
     echo $str1
     echo $str2  
     echo $str3
     ```

     输出：

     ```shell
     [root@linux a]# ./a.sh 
     LOVE,$name
     LOVE,$name. str2=LOVE,'宋天.
     
     LOVE,\"$name".
     ```

   5. 获取字符串长度

      ```shell
      #! /bin/bash
      name="宋天"
      echo $name
      echo ${#name}
      ~                
      ```

      输出：

      ```shell
      [root@linux a]# ./a.sh 
      宋天
      2
      ```

   6. 提取子字符串

      ```shell
      #! /bin/bash
      name="宋天"
      echo $name
      echo ${name:0:1}
      ```

      输出：

      ```shell
      [root@linux a]# ./a.sh 
      宋天
      宋
      ```

   7. 查找子字符串

      哪个字符先出现就计算哪个

      ```shell
      #! /bin/bash
      name="I LOVE YOU"
      echo $name
      echo `expr index "$name" E `
      ```

      输出：

      ```shell
      [root@linux a]# ./a.sh 
      I LOVE YOU
      6
      ```

      注意：以上脚本中	`` 是反引号而不是单引号

## 5.shell传递参数

我们可以在执行shell脚本时，向脚本传递参数，脚本内获取参数的格式为：**$n**

n代表数字，1为执行脚本的第一个参数，2为第二个参数，以此类推

eg：

```shell
#! /bin/bash

echo "shell 传参实例！"
echo "执行的文件名：$0"
echo "第一个参数：$1"
echo "第二个参数：$2"
```

输出结果

```shell
[root@linux a]# ./a.sh a b
shell 传参实例！
执行的文件名：./a.sh
第一个参数：a
第二个参数：b
```

- 特殊字符处理参数

| **参数处理** | **说明**                                                     |
| ------------ | ------------------------------------------------------------ |
| $#           | 传递到脚本的参数个数                                         |
| $*           | 以一个单字符串显示所有向脚本传递的参数。    如"$*"用「"」括起来的情况、以"$1 $2 … $n"的形式输出所有参数。 |
| $$           | 脚本运行的当前进程ID号                                       |
| $!           | 后台运行的最后一个进程的ID号                                 |
| $@           | 与$*相同，但是使用时加引号，并在引号中返回每个参数。    如"$@"用「"」括起来的情况、以"$1" "$2" … "$n" 的形式输出所有参数。 |
| $-           | 显示Shell使用的当前选项，与[set命令](http://www.runoob.com/linux/linux-comm-set.html)功能相同。 |
| $?           | 显示最后命令的退出状态。0表示没有错误，其他任何值表明有错误。 |

eg：

```shell
#! /bin/bash
echo "shell 传参实例！"
echo "执行的文件名：$0"
echo "第一个参数：$1"
echo "参数个数为：$#"
echo "传递的参数作为一个字符串显示：$*"
```

输出：

```shell
[root@linux a]# ./a.sh aa ss cc
shell 传参实例！
执行的文件名：./a.sh
第一个参数：aa
参数个数为：3
传递的参数作为一个字符串显示：aa ss cc
```

注意：$*和$@的区别

- 相同点：都是引用所有的参数

- 不同点：只有在双引号中体现出来

  假设在脚本运行时写了三个参数 1、2、3，，则 " * " 等价于 "1 2 3"（传递了一个参数），而 "@" 等价于      "1" "2" "3"（传递了三个参数）。

## 6.shell算数运算符

概念：shell和其他编程一样，支持：算数，关系，布尔，字符串等运算符。原生bash不支持简单的数学运算，但是可以通过其他命令来实现，例如：expr。expr是一款表达式计算工具，使用它能够完成表达式的求值操作

eg：

```shell
#! /bin/bash

str=`expr 2 + 2`
echo $str
```

注意：表达式和算数运算符之间要有空格，且完整的表达式要被``包含

输出：

```shell
[root@linux a]# ./a.sh 
4
```

常用运算符

| **运算符** | **说明**                                      | **举例**                      |
| ---------- | --------------------------------------------- | ----------------------------- |
| +          | 加法                                          | `expr $a + $b` 结果为 30。    |
| -          | 减法                                          | `expr $a - $b` 结果为 -10。   |
| *          | 乘法                                          | `expr $a \* $b` 结果为  200。 |
| /          | 除法                                          | `expr $b / $a` 结果为 2。     |
| %          | 取余                                          | `expr $b % $a` 结果为 0。     |
| =          | 赋值                                          | a=$b 将把变量 b 的值赋给 a。  |
| ==         | 相等。用于比较两个数字，相同则返回 true。     | [ $a == $b ] 返回 false。     |
| !=         | 不相等。用于比较两个数字，不相同则返回 true。 | [ $a != $b ] 返回 true。      |

注意：条件表达式要放在大括号之间，并且要有空格，例如: **[$a==$b]** 是错误的，必须写成 **[ $a == $b ]**。

eg：

```shell
#! /bin/bash
a=4
b=20
#加法运算
echo expr $a + $b
#减法运算
echo expr $a - $b
#乘法运算，注意*号前面需要反斜杠
echo expr $a \* $b
#除法运算
echo $a / $b
echo "a = $a"
c=$((a+b))
d=$[a+b]
echo "c = $c"
echo "d = $d"

```

输出

```shell
[root@linux a]# ./a.sh 
expr 4 + 20
expr 4 - 20
expr 4 * 20
4 / 20
a = 4
c = 24
d = 24
```

## 7.流程控制语句

1. if 

   ```shell
   if condition
   then
       command1 
       command2
       ...
       commandN 
   fi
   
   ```

   写成一行

   ```shell
   if [ $(ps -ef | grep -c "ssh") -gt 1 ]; then echo "true"; fi
   ```

2. if else

   ```shell
   if condition
   then
       command1
       command2
       ...
       commandN
   else
       command
   fi
   
   ```

3. if else-if else

   ```shell
   if condition1
   then
       command1
   elif condition2 
   then 
       command2
   else
       commandN
   fi
   ```

4. 关系运算符

   概念：关系运算符只支持数字，不支持字符串，除非字符串的值是数字。

   假定变量 a 为 10，变量 b 为 20：

   | **运算符** | **说明**                                              | **举例**                   |
   | ---------- | ----------------------------------------------------- | -------------------------- |
   | -eq        | 检测两个数是否相等，相等返回 true。                   | [ $a -eq $b ] 返回 false。 |
   | -ne        | 检测两个数是否不相等，不相等返回 true。               | [ $a -ne $b ] 返回 true。  |
   | -gt        | 检测左边的数是否大于右边的，如果是，则返回 true。     | [ $a -gt $b ] 返回 false。 |
   | -lt        | 检测左边的数是否小于右边的，如果是，则返回 true。     | [ $a -lt $b ] 返回 true。  |
   | -ge        | 检测左边的数是否大于等于右边的，如果是，则返回 true。 | [ $a -ge $b ] 返回 false。 |
   | -le        | 检测左边的数是否小于等于右边的，如果是，则返回 true。 | [ $a -le $b ] 返回 true。  |

   eg：

   ```shell
   a=10
   
   b=20
   
   if [ $a == $b ]
   
   then
   
      echo "a 等于 b"
   
   elif [ $a -gt $b ]
   
   then
   
      echo "a 大于 b"
   
   elif [ $a -lt $b ]
   
   then
   
      echo "a 小于 b"
   
   else
   
      echo "没有符合的条件"
   
   fi
   
   ```

   if else语句经常与test命令结合使用，如下所示：

   ```shell
   num1=$[2*3]
   
   num2=$[1+5]
   
   if test $[num1] -eq $[num2]
   
   then
   
       echo '两个数字相等!'
   
   else
   
       echo '两个数字不相等!'
   
   fi
   ```

5. for循环

   与其他编程语言类似，shell支持for循环

   ```shell
   for var in item1 item2 ... itemN
   do
       command1
       command2
       ...
       commandN
   done
   ```

   写成一行

   ```shell
   for var in item1 item2 ... itemN; do command1; command2… done;
   ```

   当变量值在列表里，for循环即执行一次所有命令，使用变量名获取列表中的当前取值。命令可为任何有效的shell命令和语句。in列表可以包含替换、字符串和文件名。

   in列表是可选的，如果不用它，for循环使用命令行的位置参数。

   eg：

   ```shell
   for loop in 1 2 3 4 5
   do
       echo "The value is: $loop"
   done
   ```

   输出：

   ```shell
   The value is: 1
   
   The value is: 2
   
   The value is: 3
   
   The value is: 4
   
   The value is: 5
   
   ```

6. while语句

   概念：while循环用于不断执行一系列命令，也用于从输入文件中读取数据；命令通常为测试条件

   ```shell
   while condition
   do
       command
   done
   ```

   eg：

   ```shell
   #!/bin/bash
   
   int=1
   while(( $int<=5 ))
   do
       echo $int
       let "int++"
   done
   
   ```

   输出：

   ```shell
   1
   2
   3
   4
   5
   ```

   使用中使用了 Bash let 命令，它用于执行一个或多个表达式，变量计算中不需要加上 $ 来表示变量，具体可查阅：[Bash let 命令](http://www.runoob.com/linux/linux-comm-let.html)。

7. 无限循环

   eg1：

   ```shell
   while :
   do
       command
   done
   ```

   eg2：

   ```shell
   while true
   do
       command
   done
   
   ```

   eg3：

   ```shell
   for (( ; ; ))
   ```

8. case

   Shell case语句为多选择语句。可以用case语句匹配一个值与一个模式，如果匹配成功，执行相匹配的命令

   ```shell
   case 值 in
   
   模式1)
       command1
       command2
       ...
       commandN
       ;;
   模式2）
       command1
       command2
       ...
       commandN
       ;;
   esac
   ```

   - case工作方式如上所示。取值后面必须为单词in，每一模式必须以右括号结束。取值可以为变量或常数。匹配发现取值符合某一模式后，其间所有命令开始执行直至 ;;。

   - 取值将检测匹配的每一个模式。一旦模式匹配，则执行完匹配模式相应命令后不再继续其他模式。如果无一匹配模式，使用星号 * 捕获该值，再执行后面的命令。

   下面的脚本提示输入1到4，与每一种模式进行匹配：

   ```shell
   echo '输入 1 到 4 之间的数字:'
   
   echo '你输入的数字为:'
   
   read aNum
   
   case $aNum in
       1)  echo '你选择了 1'
       ;;
   
       2)  echo '你选择了 2'
       ;;
   
       3)  echo '你选择了 3'
       ;;
   
       4)  echo '你选择了 4'
       ;;
   
       *)  echo '你没有输入 1 到 4 之间的数字'
       ;;
   esac
   
   ```

   输出：

   ```shell
   输入 1 到 4 之间的数字:
   
   你输入的数字为:
   
   3
   
   你选择了 3
   ```

9. 跳出循环

   Shell使用两个命令来实现该功能：break和continue。

   1. **break**命令

      break命令允许跳出所有循环（终止执行后面的所有循环）。

      ```shell
      #!/bin/bash
      
      while :
      do
          echo -n "输入 1 到 5 之间的数字:"
          
          read aNum
      
          case $aNum in
              1|2|3|4|5) echo "你输入的数字为 $aNum!"
              ;;
              *) echo "你输入的数字不是 1 到 5 之间的! 游戏结束"
                  break
              ;;
          esac
      done
      
      ```

   2. **continue**

      continue命令与break命令类似，只有一点差别，它不会跳出所有循环，仅仅跳出当前循环。

      ```shell
      #!/bin/bash
      
      while :
      
      do
      
          echo -n "输入 1 到 5 之间的数字: "
      
          read aNum
      
          case $aNum in
      
              1|2|3|4|5) echo "你输入的数字为 $aNum!"
      
              ;;
      
              *) echo "你输入的数字不是 1 到 5 之间的!"
      
                  continue
      
                  echo "游戏结束"
      
              ;;
      
          esac
      
      done
      
      ```

## 8.函数使用

1. 语法

   linux shell 可以用户定义函数，然后在shell脚本中可以随便调用。

   ```shell
   [ function ] funname()
   
   {
       action;
       [return int;]
   }
   
   ```

   说明：

   - 1、可以带function fun() 定义，也可以直接fun() 定义,不带任何参数。
   - 2、参数返回，可以显示加：return 返回，如果不加，将以最后一条命令运行结果，作为返回值。 return后跟数值n(0-255

   eg1：

   ```shell
   #!/bin/bash
   
   demoFun(){
   
       echo "这是我的第一个 shell 函数!"
   
   }
   
   echo "-----函数开始执行-----"
   demoFun
   echo "-----函数执行完毕-----"
   ```

   输出：

   ```shell
   -----函数开始执行-----
   这是我的第一个 shell 函数!
   -----函数执行完毕-----
   
   ```

   eg2：

   下面定义一个带有return语句的函数：

   ```shell
   #!/bin/bash
   
   funWithReturn(){
   
       echo "这个函数会对输入的两个数字进行相加运算..."
   
       echo "输入第一个数字: "
   
       read aNum
   
       echo "输入第二个数字: "
   
       read anotherNum
   
       echo "两个数字分别为 $aNum 和 $anotherNum !"
   
       return $(($aNum+$anotherNum))
   
   }
   
   funWithReturn
   
   echo "输入的两个数字之和为 $? !"
   ```

   输出：

   ```shell
   这个函数会对输入的两个数字进行相加运算...
   
   输入第一个数字: 
   
   1
   
   输入第二个数字: 
   
   2
   
   两个数字分别为 1 和 2 !
   
   输入的两个数字之和为 3 !
   
   ```

   函数返回值在调用该函数后通过 $? 来获得。

   注意：所有函数在使用前必须定义。这意味着必须将函数放在脚本开始部分，直至shell解释器首次发现它时，才可以使用。调用函数仅使用其函数名即可。

2. 函数参数

   在Shell中，调用函数时可以向其传递参数。在函数体内部，通过 $n 的形式来获取参数的值，例如，$1表示第一个参数，$2表示第二个参数...

   ```shell
   #!/bin/bash
   
   funWithParam(){
       echo "第一个参数为 $1 !"
       echo "第二个参数为 $2 !"
       echo "第十个参数为 $10 !"
       echo "第十个参数为 ${10} !"
       echo "第十一个参数为 ${11} !"
       echo "参数总数有 $# 个!"
       echo "作为一个字符串输出所有参数 $* !"
   }
   
   funWithParam 1 2 3 4 5 6 7 8 9 34 73
   
   ```

   结果：

   ```shell
   第一个参数为 1 !
   
   第二个参数为 2 !
   
   第十个参数为 10 !
   
   第十个参数为 34 !
   
   第十一个参数为 73 !
   
   参数总数有 11 个!
   
   作为一个字符串输出所有参数 1 2 3 4 5 6 7 8 9 34 73 !
   ```

   注意，$10 不能获取第十个参数，获取第十个参数需要${10}。当n>=10时，需要使用${n}来获取参数。

   另外，还有几个特殊字符用来处理参数：

   | **参数处理** | **说明**                                                     |
   | ------------ | ------------------------------------------------------------ |
   | $#           | 传递到脚本的参数个数                                         |
   | $*           | 以一个单字符串显示所有向脚本传递的参数                       |
   | $$           | 脚本运行的当前进程ID号                                       |
   | $!           | 后台运行的最后一个进程的ID号                                 |
   | $@           | 与$*相同，但是使用时加引号，并在引号中返回每个参数。         |
   | $-           | 显示Shell使用的当前选项，与set命令功能相同。                 |
   | $?           | 显示最后命令的退出状态。0表示没有错误，其他任何值表明有错误。 |

## 9.数组

概念：

- 数组中可以存放多个值。Bash Shell 只支持一维数组（不支持多维数组），初始化时不需要定义数组大小（与 PHP 类似）。

- 与大部分编程语言类似，数组元素的下标由0开始。

- Shell 数组用括号来表示，元素用"空格"符号分割开，语法格式如下array_name=(value1 ... valuen)

eg：

```shell
#!/bin/bash

my_array=(A B "C" D)

我们也可以使用下标来定义数组:

array_name[0]=value0

array_name[1]=value1

array_name[2]=value2
```

1. 读取数组

   ```shell
   ${array_name[index]}
   ```

   eg：

   ```shell
   !/bin/bash
   
   my_array=(A B "C" D)
   
   echo "第一个元素为: ${my_array[0]}"
   
   echo "第二个元素为: ${my_array[1]}"
   
   echo "第三个元素为: ${my_array[2]}"
   
   echo "第四个元素为: ${my_array[3]}"
   ```

   输出：

   ```shell
   第一个元素为: A
   第二个元素为: B
   第三个元素为: C
   第四个元素为: D
   ```

   - 获取数组中的所有元素

     使用@ 或 * 可以获取数组中的所有元素，例如：

     ```shell
     #!/bin/bash
     
     my_array[0]=A
     my_array[1]=B
     my_array[2]=C
     my_array[3]=D
     
     echo "数组的元素为: ${my_array[*]}"
     echo "数组的元素为: ${my_array[@]}"
     
     ```

     输出：

     ```shell
     
     数组的元素为: A B C D
     数组的元素为: A B C D
     
     ```

   -  获取数组的长度

     获取数组长度的方法与获取字符串长度的方法相同，例如：

     ```shell
     #!/bin/bash
     my_array[0]=A
     my_array[1]=B
     my_array[2]=C
     my_array[3]=D
     
     echo "数组元素个数为: ${#my_array[*]}"
     echo "数组元素个数为: ${#my_array[@]}"
     ```

   - 遍历数组

     1. eg1：

        ```shell
        #!/bin/bash
        
        my_arr=(AA BB CC)
        
        for var in ${my_arr[*]}
        do
          echo $var
        done
        
        ```

     2. eg2：

        ```shell
        my_arr=(AA BB CC)
        my_arr_num=${#my_arr[*]}
        for((i=0;i<my_arr_num;i++));
        do
          echo "-----------------------------"
          echo ${my_arr[$i]}
        done
        ```

        

## 10.select

概念：select表达式是bash的一种扩展应用，擅长于交互式场合。用户可以从一组不同的值中进行选择：

```shell
select var in ... ;
do
　commond
done
.... now $var can be used ...

```

注意：select 是个无限循环，因此要记住用 break 命令退出循环，或用exit 命令终止脚本

eg：

```shell
#!/bin/bash
 
echo "What is your favourite OS?"
select var in "Linux" "Gnu Hurd" "Free BSD" "Other";
do
  break;
done
echo "You have selected $var"

```

输出：

```shell
What is your favourite OS?
1) Linux
2) Gnu Hurd
3) Free BSD
4) Other
#? 1
You have selected Linux

```

1. select和case的综合练习

   ```shell
   #!/bin/bash
   echo "你想学习什么语言?"
   PS3="请输入你的选择:"    # 设置提示字符串
   select var in java c++ shell python;
   do
     case $var in
        "java")
          echo "恭喜你选择成功.java最牛"
        ;;
        "c++")
          echo "驱动开发  网络优化  go 语言"
        ;;
        "shell")
          echo "运维必会"
        ;;
        python)
          echo "人工智能"
        esac
    
        break    # 如果这里没有break 将会进行无限循环
   done
    
   echo "你选择的是:$var"
   
   ```

2. 修改提示符PS3

   PS3作为select语句的shell界面提示符，提示符为PS3的值（赋予的字符串），更换默认的提示符”#?”

   注意：PS3一定要定义在select语句的前面

   eg：

   ```shell
   #!/bin/bash
   PS3="请输入你的选择:"
   select var in java c++ shell;
   do
     break;
   done
   echo "你选择的是$var"
   
   ```

## 11.EOF

 Shell中通常将EOF与 << 结合使用，表示后续的输入作为子命令或子Shell的输入，直到遇到EOF为止，再返回到主调Shell。

1. 覆盖效果

   ```shell
   #!/bin/bash
   cat >/root/a.txt <<EOF
   ABCD
   EOF
   
   ```

2. 追加效果

   ```shell
   #!/bin/bash
   cat >/root/a.txt <<EOF
   ABCD
   EOF
   
   ```

## 12.加载其它文件的变量

概念：和其他语言一样，Shell 也可以包含外部脚本。这样可以很方便的封装一些公用的代码作为一个独立的文件。

Shell 文件包含的语法格式如下：

```shell
. filename   # 注意点号(.)和文件名中间有一空格
 
或
 
source filename 
```

练习：

定义两个文件 test1.sh和test2.sh，在test1中定义一个变量arr=(java c++ shell),在test2中对arr进行循环打印输出。

1. 第一步: vim test1.sh

   ```shell
   #!/bin/bash
   
   my_arr=(AA BB CC)
   ```

2. 第二步: vim test2.sh

   ```shell
   #!/bin/bash
   
   source ./test1.sh  # 加载test1.sh 的文件内容
   
   for var in ${my_arr[*]}
   
   do
   
     echo $var
   
   done
   ```

3. 第三步: 执行 test2.sh

   ```shell
   ./sh test2.sh
   ```

   好处 : 

   ​    1. 数据源 和 业务处理 分离

   ​    2. 复用 代码扩展性更强

    

## 13 案例 一键安装jdk

```shell
#!/bin/bash

# 1.提示安装jdk
echo "现在开始安装jdk"
sleep 1		# 休眠1秒钟
# 2.删除centos自带的jdk
oldjava=` rpm -qa | grep java `
for old in ${oldjava};
do
    # echo $old
    rpm -e --nodeps $old
done
# 3.创建安装目录/usr/local/java, 进入安装目录
java_path="/usr/local/java"
if [ ! -d $java_path  ]
then
    mkdir -p $java_path
fi
cd $java_path

# 4.下载jdk安装包
if [ ! -f jdk-8u141-linux-x64.tar.gz  ]
then
     cp /export/soft/jdk-8u141-linux-x64.tar.gz ./
fi

# 5.解压缩安装包,删除安装包
if [ ! -f jdk1.8.0_141  ]
then
    tar -zxvf jdk-8u141-linux-x64.tar.gz
    rm -rf jdk-8u141-linux-x64.tar.gz
fi


# 6.设置环境变量
JAVA_HOME="/usr/local/java/jdk1.8.0_141"
if ! grep "JAVA_HOME=$JAVA_HOME" /etc/profile
then
     echo "-------------------- path = $PATH"
    # JAVA_HOME
    echo "--------------JAVA_HOME------------------"
    echo "export JAVA_HOME=$JAVA_HOME" | tee -a /etc/profile
    # PATH
    echo "--------------PATH------------------------"
    echo "export PATH=:$JAVA_HOME/bin:$PATH" | tee -a /etc/profile
fi

# 7.加载环境变量
source /etc/profile

# 8.提示用户安装成功,查看jdk安装版本
echo "恭喜您,jdk安装成功!"
java -version
```

注意：在var目录上传jdk安装包！！！

JDK下载地址：

```
wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u131-b11/d54c1d3a095b4ff2b6607d096fa80163/jdk-8u131-linux-x64.tar.gz

```

