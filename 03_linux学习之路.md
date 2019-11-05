# linux学习之路

## 管道相关命令

### 1. cut

概念：以某种方式按照文件的行进行分割

参数列表：

 -b 按字节选取 忽略多字节字符边界，除非也指定了 -n 标志

 -c 按字符选取

 -d 自定义分隔符，默认为制表符。

 -f 与-d一起使用，指定显示哪个区域。

范围控制：

  n:只有第n项

  n-:从第n项一直到行尾

  n-m:从第n项到第m项(包括m)

练习：

准备文件：

s.txt

```
111:aaa:bbb:ccc
222:ddd:eee:fff
333:ggg:hhh
444:iii
```

案例1：截取文件中前两行的第五个字符

答案：

```
head -2 1.txt | cut -c 5
```

输出：

```
[root@linux opt]# head -2 s.txt | cut -c 5
a
d
```

案例2：截取出1.txt文件中前2行以”:”进行分割的第1,2段内容

答案:

```
head -2 1.txt | cut -d ':' -f 1,2
```

输出：

```
[root@linux opt]# head -2 s.txt | cut -d ':' -f 1,2
111:aaa
222:ddd
```

### 2. sort工作原理

基本使用：sort将文件的每一行作为一个单位，相互比较，比较原则是从首字符向后，依次按ASCII码值进行比较，最后将他们按升序输出。

```
[root@node01 tmp]# cat 01.txt
banana
apple
pear
orange
pear

[root@node01 tmp]# sort 01.txt 
apple
banana
orange
pear
pea
```

- 参数`-u`

  作用：输出行中去除重复行

  ```
  [root@linux opt]# sort -u s.txt 
  111:aaa:bbb:ccc
  222:ddd:eee:fff
  333:ggg:hhh
  444:iii
  ```

- 参数`-r`和`-n`

  说明：-r表示降序，n表示按数字进行排序

  ```
  [root@node01 tmp]# sort -nr 02.txt  
  11
  10
  9
  8
  7
  6
  5
  4
  3
  2
  1
  ```

### 3. wc

说明：该命令用于计算字数

利用wc指令我们可以计算文件的Byte数、字数、或是列数，若不指定文件名称、或是所给予的文件名为"-"，则wc指令会从标准输入设备读取数据。

- 参数

  - -c或--bytes或--chars 只显示Bytes数。
  - -l或--lines 只显示行数。
  - -w或--words 只显示字数。
  - --help 在线帮助。
  - --version 显示版本信息。

- 案例

  - 准备文件：

    ```
    111
    222 bbb
    333 aaa bbb 
    444 aaa bbb ccc
    555 aaa bbb ccc ddd
    666 aaa bbb ccc ddd eee
    ```

  - 需求： 统计指定文件行数 字数 字节数

    ```
    [root@linux opt]# wc z.txt 
     4 17 68 z.txt
    ```

    行数为4, 单词数为 17, 字节数为 68，文件名为z.txt

  - 需求: 统计多个文件的 行数 单词数 字节数

    ```
    wc 01.txt 02.txt 03.txt
    ```

  - 需求: 查看 根目录下有多少个文件

    ```
    ls / | wc -w
    ```

### 4. uniq

说明：uniq 命令用于检查及删除文本文件中重复出现的行，一般与 sort 命令结合使用。

参数：

​    -c 统计行数

1. 准备工作

   ```
   张三    98
   李四    100
   王五    90
   赵六    95
   麻七    70
   李四    100
   王五    90
   赵六    95
   麻七    70
   ```

2. 练习1 去除5.txt中重复的行

   ```
   cat 5.txt | sort | uniq
   ```

3. 练习2 统计5.txt中每行内容出现的次数

   ```
   cat 5.txt | sort | uniq -c
   ```

### 5. tee

tee 和 `>`类似，重定向的同时还在屏幕输出
参数说明：
tee -a 内容追加 和 >> 类似

eg：

```
[root@hadoop01 tmp]# echo 'aaa' | tee 1.txt
aaa

[root@hadoop01 tmp]# cat 1.txt
aaa

[root@hadoop01 tmp]# echo 'bbb' | tee -a 1.txt
bbb

[root@hadoop01 tmp]# cat 1.txt
aaa
bbb
```



1. 练习1： 统计5.txt中每行内容出现的次数输出到a.txt,并且把内容显示在控制台

   ```
   sort 5.txt | uniq -c | tee a.txt
   ```

2. 统计5.txt中每行内容出现的次数输出追加到a.txt,并且把内容显示在控制台

   ```
   sort 5.txt | uniq -c | tee -a a.txt
   ```

### 6.  tr

Linux tr 命令用于转换或删除文件中的字符。
tr 指令从标准输入设备读取数据，经过字符串转译后，将结果输出到标准输出设备。

参数：

-d, --delete：删除指令字符

1. 练习1 把itheima的小写i换成大写I

   ```
   echo "itheima" | tr 'i' 'I'
   ```

2. 把itheima的小写i和a换成大写I和A

   ```
   echo "itheima" | tr 'i' 'I' | tr 'a' 'A'
   ```

3. 把itheima的转换为大写

   ```
   echo "itheima" |tr '[a-z]' '[A-Z]'
   ```

4. 删除abc1d4e5f中的数字

   ```
   echo 'abc1d4e5f' | tr -d '[0-9]'
   ```

5. 单词计数

   文件：

   ```
   words.txt中的内容如下：
   hello,world,hadoop
   hive,sqoop,flume,hello
   kitty,tom,jerry,world
   hadoop
   ```

   ```
   cat words.txt | tr -s ',' '\n' | sort | uniq -c | sort -r | awk '{print $2, $1}'
   ```

6. 脚本解释：

   ​	tr -s ' ' '\n'       
   ​	表示：连续出现的空格只保留一个，并在空格处以换行符分割文本   
   ​
   ​	tr -s ',' '\t'
   ​	表示 ,使用 tab代替
   ​	

   ​	sort
   ​	表示：对输出文本进行排序

   

   ​	uniq -c
   ​	表示：对连续出现的重复的行进行计数

   ​	sort -r
   ​	表示：对输出文本进行降序排序

   ​	awk '{print $2, $1}'
   ​	表示：打印出文本的第二列和第一列

### 7. split

该指令将大文件分割成较小的文件，在默认情况下将按照每1000行切割成一个小文件
参数说明：
    -b<字节> : 指定每多少字节切成一个小文件
    -l<行数> : 指定每多少行切成一个小文件



1.  练习1：从/etc目录下查找以conf结尾的文件，并所有文件内容写入到v.txt

   ```
   find /etc/ -type f -name "*conf" -exec cat {} >> v.txt \;
   ```

2. 练习2：把v.txt进行分割

   ```
   split v.txt
   ```

3. 把v.txt按2000进行分割

   ```
   split -l 2000 v.txt
   ```

4. 把v.txt按10k进行分割

   ```
   du –sh v.txt
   
   split -b 10k v.txt
   ```

   hadoop数据的划分, 这个是物理上真真实实的进行了划分，数据文件上传到HDFS里的时候，需要划分成一块一块，默认的每个块128MB。

### 8. awk

说明：awk是一种处理文本文件的命令，是一个强大的文本分析工具。但是比较复杂，不过功能比sed更加的强大，它支持分段。默认每行按空格或TAB分割。

参数说明：
    -F 指定输入文件折分隔符

准备文件：

```
aaa 111 333
bbb 444 555
ccc 666 777 888
ddd 999 222 999
```



1. 默认分段

   默认每行按空格或TAB分割，使用$n来获取段号

2. 练习1：创建1.txt,输入内容，打印出第1段

   ```
   awk '{print $1}' 1.txt
   ```

3. 练习2：打印出1.txt的第1,2,3段

   ```
   awk '{print $1,$2,$3}' 1.txt
   ```

4. 练习3：打印出1.txt的第1,2,3段，并且使用#号连接

   ```
   awk '{print $1"#"$2"#"$3}' 1.txt
   ```

**段之间的连接符OFS**

1. 练习1：打印1,2,3段，指定#为连接符

   ```
   awk '{OFS="#"}{print $1,$2,$3}' 1.txt
   ```

**指定分隔符**

参数：

-F	来指定分隔符

准备文件：

```
aaa:111:333
bbb:444:555
ccc:666:777:888
ddd:999:222:999:cccc
```

1. 练习1：打印出2.txt的第1段

   ```
   awk -F ':' '{print $1}' 2.txt
   ```

2. 练习2：打印出2.txt的所有段

   ```
   awk -F ':' '{print $0}' 2.txt
   ```

3. 练习3：打印出2.txt的第1,3段

   ```
   awk -F ':' '{print $1,$3}' 2.txt
   ```

**内容匹配**

1. 练习1 匹配2.txt中包含cc的内容

   ```
   awk '/cc/' 2.txt
   ```

2. 练习2：匹配2.txt中第1段包含cc的内容

   ```
   awk -F ':' '$1 ~ /cc/' 2.txt
   ```

3. 练习3：匹配2.txt中第1段包含至少连续两个c的内容

   ```
   awk -F ':' '$1 ~ /cc+/' 2.txt
   ```

4. 练习4：在2.txt中如果匹配到aaa就打印第1,3段，如果匹配到ccc,就打印第1,3,4段

   ```
   awk -F ':' '/aaa/ {print $1,$3} /ccc/ {print $1,$3,$4}' 1.txt
   ```

5. 练习5：在2.txt中如果匹配到aaa或者ddd,就打印全部内容

   ```
   awk -F ':' '/aaa|ddd/ {print $0}' 2.txt
   ```

**段内容判断**

1. 练习1 在2.txt中如果第3段等于222就打印所有内容

   ```
   awk -F ':' '$3==222 {print $0}'  2.txt 
   ```

2. 练习2 在2.txt中如果第3段等于333就打印第一段

   ```
   awk -F ':' '$3==333 {print $1}'  2.txt
   ```

3. 练习3 在2.txt中如果第3段大于等于300就打印第一段

   ```
   awk -F ':' '$3>=300 {print $0}'  2.txt
   ```

4. 练习4 在2.txt中如果第5段不等于cccc就打印全部

   ```
   awk -F ':' '$5!="cccc" {print $0}' 2.txt
   ```

5. 练习5 在2.txt中如果第1段等于ccc，并且第2段大于300就打印全部

   ```
   awk -F ':' '$1=="ccc" && $2>300 {print $0}' 2.txt
   ```

6. 练习6 在2.txt中如果第1段等于ccc，并且第2段匹配666就打印全部

   ```
   awk -F ':' '$1=="ccc" && $2==666 {print $0}' 2.txt
   ```

**段之间的比较**

1. 练习1 在2.txt中如果第3段小于第4段就打印全部

   ```
   awk -F ':' '$3<$4 {print $0}'  2.txt
   ```

2. 练习2 在2.txt中如果第2段等于第4段就打印全部

   ```
   awk -F ':' '$2==$4 {print $0}' 2.txt
   ```

 **NR行号和 NF段数**

1. 练习1打印2.txt全部内容显示行号

   ```
   awk -F ':' '{print NR " : " $0}'  2.txt
   ```

2. 练习2 打印2.txt全部内容显示段数

   ```
   awk -F ':' '{print NF " : " $0}'  2.txt
   ```

3. 练习3 打印2.txt前2行，并显示行号 （用三种不同的方式实现）

   ```
   nl 2.txt | head -2
   
   nl 2.txt | sed -n -e '1,2p'
       
   awk -F ':' 'NR<=2 {print NR " " $0}' 2.txt
   ```

4. 练习4 打印2.txt前2行，并且第1段匹配 aaa或者eee，打印全部，打印行号 （用两种方式）

   ```
   nl 2.txt | head -2 | awk -F ':' '$1 ~ /aaa|eee/'
   awk -F ':' 'NR<=2 && $1 ~ /aaa|eee/ {print NR " " $0}'  2.txt
   ```

5. 练习5 从2.txt的前3行中匹配出第2段等于 666，并显示行号（用两种方式）

   ```
   nl passwd | head -n 3 | awk -F ":" '$7=="/sbin/nologin" {print $0}'
   awk -F ':' 'NR<=3 && $2==666 {print $0}' 2.txt
   ```

6. 练习6 从2.txt前3行，把第1段内容替换为itheima，指定分隔符为|，显示行号（用两种方式）

   ```
   nl 2.txt | head -3 | awk -F ':' '{OFS="|"} $1="itheima" {print NR "  " $0}'
   awk -F ':' '{OFS="|"} NR<=3 && $1="itheima" {print NR "  " $0}'  2.txt
   ```

**分段求和**

awk脚本，我们需要注意两个关键词BEGIN和END。
BEGIN{ 这里面放的是执行前的语句 }
{这里面放的是处理每一行时要执行的语句}
END {这里面放的是处理完所有的行后要执行的语句 }

1.  练习1 对2.txt中的第2段求和

   ```
   awk -F ':' 'BEGING{}{total=total+$2}END{print total}'  2.txt
   ```



**综合练习**

1. 练习1 对统计awk目录下所有文本文件的大小

   ```
   ll | awk 'BEGIN{}{total=total+$5} END{print(total)}'
   ```

2. 练习2 打印99乘法表

   ```
   awk 'BEGIN{ for(i=1;i<=9;i++){ for(j=1;j<=i;j++){ printf("%dx%d=%d%s", i, j, i*j, "\t" )  } printf("\n")  }  }'
   ```

### 9.  sed

说明：sed命令是来处理文本文件。sed也可以实现grep的功能，但是要复杂一些。
它的强大之处在于替换。并且对于正则也是支持的。

- 参数说明：

  -n 仅显示处理后的结果
  -e 以选项中指定的脚本来处理输入的文本文件
  -f 以选项中指定的脚本文件来处理输入的文本文件

- 动作说明

  p ：打印，亦即将某个选择的数据印出。通常 p 会与参数 sed -n 一起运行
  d ：删除
  a ：新增，内容出现在下一行
  i ：插入， 内容出现在上一行
  c ：取代， c 的后面可以接字串，这些字串可以取代 n1,n2 之间的行
  s ：取代，可以直接进行取代的工作！通常这个 s 的动作可以搭配正规表示法！
  = ：显示行号

准备数据

```
aaa java root
bbb hello
ccc rt
ddd root nologin
eee rtt
fff ROOT nologin
ggg rttt
```

1. -n和p的组合：查找
   - 查找01.txt中包含root行

     ```
     sed -n -e '/root/p' 01.txt
     ```

   - 练习2查找出01.txt中包含一个或多个r,r后面是t，并显示行号

     ```
     nl 01.txt | sed -nr -e '/r+t/p'
     sed -nr -e '/r+t/p' -e '/r+t/=' 01.txt
     ```

   - 练习3列出01.txt的第2行数据,并显示行号

     ```
     nl 01.txt | sed -n -e '2p'
     ```

   - 练习4 列出01.txt的第2到第5行数据，并显示行号

     ```
     nl 01.txt |sed -n -e '2,5p'        # 显示行号
     ```

   - 练习5 列出01.txt全部的数据，并显示行号

     ```
     nl 01.txt
     cat -n 01.txt
     sed -n -e '1,$p' -e '1,$=' 01.txt
     ```

   - 练习6 列出01.txt中包含root的内容，root不区分大小写,并显示行号

     ```
     nl 01.txt | sed -n -e '/root/Ip'
     
     nl 01.txt | grep -i root
     
     cat -n 01.txt | grep -i root
     ```

2.  d 删除

   - 练习1 删除01.txt中前3行数据，并显示行号

     ```
     nl 01.txt | sed -e '1,3d'
     ```

   - 练习2 保留01.txt中前4行数据，并显示行号

     ```
     nl 01.txt | sed -e '5,$d'
     ```

3.  a 和i 添加

     a:在行后面插入 (append)
      i:在行前面插入 (insert)

   - 练习1:在01.txt的第二行后添加aaaaa,并显示行号

     ```
     nl 01.txt | sed '2a aaaaa'
     ```

   - 练习2 在01.txt的第1行前添加bbbbb，并显示行号

     ```
     nl 01.txt | sed -e '1i bbbbb'
     ```

4. s和c替换

   s:对字符串进入替换  

   c:对行进行替换

   - 练习1：把01.txt中的nologin替换成为itheima,并显示行号

     ```
     nl passwd | sed -e 's/nologin/itheima/'
     ```

   - 练习2 把01.txt中的1,2行替换为aaa,并显示行号

     ```
     nl passwd | sed -e '1,2c aaa'
     ```

5. 对原文件进行操作

   注意：在进行操作之前，最好是对数据进行备份，放置操作失误，数据无法恢复！

   - 练习1 删除01.txt中前2行数据，并且删除原文件中的数据

     ```
     sed -i -e '1,2d' 01.txt
     
     nl passwd 查看数据
     
     ```

   - 练习2 在01.txt中把nologin替换为itheima

     ```
     sed -i -e 's/nologin/itheima/' 01.txt
     ```

   - 练习3 在01.txt文件中第2、3行替换为aaa

     ```
     sed -i -e '2,3c aaa' 01.txt
     ```

6.  综合练习

   - 练习1 获取ip地址

     ```
     ifconfig eth0 | grep 'inet addr:' | sed -e 's/^.addr://' | sed -e 's/Bcast.//'
     ```

   - 练习2 从01.txt中提出数据，删除前5行，并把nologin替换为itheima,并显示行号

     ```
     nl 01.txt | sed -e '1,5d' | sed -e 's/nologin/itheima/'
     ```

   - 练习3 从01.txt中提出数据，匹配出包含root的内容，再把nologin替换为itheima

     ```
     nl 01.txt | grep 'root' | sed -e 's/nologin/itheima/'
     
     或者
     
     nl 01.txt | sed -n -e '/root/p' | sed -e 's/nologin/itheima/'
     
     或者
     
     nl 01.txt | sed -n -e '/root/{s/nologin/itheima/p}' #只显示替换内容的行
     ```