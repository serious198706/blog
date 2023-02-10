---
title: Bash 中的条件表达式
toc: true
link: bash-condition-expressions
category:
  - Development
tag:
  - Linux
  - Bash
date: 2013-01-22 14:19:00
---

每种语言都会有条件表达式，在Linux下的Bash中也不例外。

这篇文章来详细讲一下bash中的条件表达式的运用。

<!-- more -->

在Bash中，基本可以分为两类，一类是使用test命令，另一种是使用"[]"表达式。这两种方式的功能是等价的。

在Bash中，要注意，不要把写其他语言的习惯带到这儿来，因为Bash是通过解释来直接执行命令的，不需要编译过程，所以为了规避一些可能出现的问题，在"[]"中间，每个元素都必须用空格隔开。比如：

``` bash
[ -e /home/tom ]
[ $num -eq 2 ]
[ "$string" = "You are a genius!" ]    // 注意，等号两边须发有空格
[ "$p" != "sss" ] || [ "$p" != "ttt" ]    // 同样地，"!="两边也必须有空格
``` 

如果写成这样：

``` bash
[ -e /home/tom]
[$num -eq 2]
[ "$string"="You are a genius!"]  // [ ]运算符将"$string"="You are a genius!"视为一个整体了
["$p"!="sss"]||["$p"!="ttt"]
``` 

是无法正确执行的。

下面看几个运行符。

### 文件比较运算符

[ -e filename ]  如果filename存在,则为真     
        
``` bash
[ -e /var/log/syslog ]
    or
test -e /var/log/syslog
``` 

[ -d filename ]  如果filename为目录,则为真   

``` bash       
[ -d /tmp/mydir ]
    or
test -d /tmp/mydir
``` 

[ -f filename ]   如果filename为常规文件,则为真    

``` bash
[ -f /usr/bin/httpd ]
    or
test -f /usr/bin/httpd
``` 

[ -L filename ]  如果filename为符号链接,则为真   

``` bash 
[ -L /usr/bin/apache2 ]
    or
test -L /usr/bin/apache2
``` 

[ -r filename ]   如果filename可读,则为真        

``` bash     
[ -r /var/log/syslog ]
    or
test -r /var/log/syslog
``` 

[ -w filename ]  如果filename可写,则为真       

``` bash      
[ -w /var/haha.txt ]
    or
test -w /var/haha.txt
``` 

[ -x filename ]  如果filename可执行,则为真     

``` bash     
[ -x /etc/init.d/mysqld ]
    or
test -x /etc/init.d/mysqld
``` 

[ filename1 -nt filename2 ] 如果filename1比filename2新,则为真  

``` bash
[ /tmp/install/etc/services -nt /etc/services ]   // newer than
    or
test /tmp/install/etc/services -nt /etc/services
``` 

[ filename1 -ot filename2 ] 如果filename1比filename2旧,则为真  

``` bash
[ /home/tom/test -ot /home/tom/test1 ]    // older than
    or
test /home/tom/test -ot /home/tom/test1
``` 

[ filename1 -ef filename2 ] 判断filename1与filename2是否为同一文件，可用在hard link的判定上。主要意义在于判定两个文件是否均指向同一个inode。

``` bash
[ /usr/bin/apache2 -ef /etc/bin/apache2 ]    // equal file
    or
test /usr/bin/apache2 -ef /etc/bin/apache2
``` 

### 字符串比较运算符

(请注意引号的使用，这是防止空格扰乱代码的好方法)

[ -z string ] 如果string长度为零,则为真  

``` bash
[ -z "$myvar" ]
    or
test -z "$myvar"
``` 

[ -n string ] 如果string长度非零,则为真  

``` bash
[ -n "$myvar" ]  // -n也可以省略
    or
test -n "$myvar"
``` 

[ string1  = string2 ] 如果string1与string2相同,则为真  

``` bash
[ "$myvar" = "one two three" ]    // "==" 也是可以的，在bash中，进行判断时，"=" 和 "==" 的功能是相同的， 这一点跟编程语言中不太相同
                                  // 但为了区别开来，还是建议用 "=="
    or
test "$myvar" = "one two three"
``` 

[ string1 != string2 ] 如果string1与string2不同,则为真  

``` bash
[ "$myvar" != "one two three" ]
    or
test "$myvar" != "one two three"
``` 

### 算术比较运算符  

[ num1 -eq num2 ]  等于

``` bash
[ $mynum -eq 3 ]    // equal
    or
test $mynum -eq 3
``` 

[ num1 -ne num2 ]  不等于

``` bash
[ $mynum -ne 3 ]    // not equal
    or
test $mynum -ne 3
``` 

[ num1 -lt num2 ]  小于

``` bash
[ $mynum -lt 3 ]    // less than
    or
test $mynum -lt 3
``` 

[ num1 -le num2 ]  小于或等于

``` bash
[ $mynum -le 3 ]    // less or equal
    or
test mynum -le 3
``` 

[ num1 -gt num2 ]  大于

``` bash
[ $mynum -gt 3 ]    // greater than
    or
test $mynum -gt 3
``` 

[ num1 -ge num2 ]  大于或等于

``` bash
[ $mynum -ge 3 ]    // greater or equal
    or
test $mynum -ge 3
``` 


### 多重条件判定

-a 两个条件同时成立，则为真

``` bash
[ -r /home/tom/test -a -x /home/tom/test ]    // 即test文件同时具有rx权限时，才返回真
    or
test -r /home/tom/test -a -x /home/tom/test 
``` 

-o 任意一个条件成立，则为真

``` bash
[ -r /home/tom/test -o -x /home/tom/test ]    // 即test文件具有r或x权限时，都返回真
    or
test -r /home/tom/test -o -x /home/tom/test 
``` 

！ 条件不成立，则为真

``` bash
[ ! -x /home/tom/test ]    // 即test文件同时不具有x权限时，才返回真
    or
test ! -x /home/tom/test 
``` 
