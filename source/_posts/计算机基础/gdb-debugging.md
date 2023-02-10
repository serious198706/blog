---
title: 使用gdb调试程序
link: gdb-debugging
category:
  - Development
tag:
  - Linux
  - gdb

date: 2013-04-17 15:54:10
---

这篇文章来讲讲Linux下强大的调试工具--gdb。

<!-- more -->

### 一、gdb常用的命令： 

跟在Windows下使用Visual Studio进行调试一样，gdb的调试功能与其大同小异。

<div class="centered">
<table border="1" cellpadding="1" cellspacing="1">
	<tbody>
		<tr>
			<td>命令</td>
			<td>功能</td>
			<td>快捷方式</td>
		</tr>
		<tr>
			<td>file</td>
			<td>加载需要调试的可执行文件</td>
			<td>file</td>
		</tr>
		<tr>
			<td>kill</td>
			<td>终止正在调试的程序</td>
			<td>k</td>
		</tr>
		<tr>
			<td>list</td>
			<td>列出源代码的一部分</td>
			<td>l</td>
		</tr>
		<tr>
			<td>info</td>
			<td>列出某个项目的信息</td>
			<td>i</td>
		</tr>
		<tr>
			<td>next</td>
			<td>执行一行代码（不进入函数内部）</td>
			<td>n</td>
		</tr>
		<tr>
			<td>step</td>
			<td>执行下一行代码</td>
			<td>s</td>
		</tr>
		<tr>
			<td>continue</td>
			<td>继续运行程序，直到下一个断点</td>
			<td>c</td>
		</tr>
		<tr>
			<td>run</td>
			<td>执行已经加载的程序</td>
			<td>r</td>
		</tr>
		<tr>
			<td>quit</td>
			<td>终止gdb</td>
			<td>q</td>
		</tr>
		<tr>
			<td>watch</td>
			<td>监视某个变量的值</td>
			<td>w</td>
		</tr>
		<tr>
			<td>examine</td>
			<td>监视某个变量的值</td>
			<td>x</td>
		</tr>
		<tr>
			<td>break</td>
			<td>设置断点</td>
			<td>b</td>
		</tr>
		<tr>
			<td>disable / enable</td>
			<td>使断点无效/有效</td>
			<td>disable/enable</td>
		</tr>
		<tr>
			<td>delete</td>
			<td>删除断点</td>
			<td>d</td>
		</tr>
		<tr>
			<td>make</td>
			<td>在不退出gdb的情况下，重新生成可执行文件</td>
			<td>make</td>
		</tr>
		<tr>
			<td>shell</td>
			<td>在不退出gdb的情况下，使用shell命令</td>
			<td>shell</td>
		</tr>
	</tbody>
</table>
</div>

这只是一部分哦。关于整个gdb调试可以写成一本书呢。

看到全称不要紧张，毕竟还是有些对英文不太感冒的同学，在gdb下按Tab，gdb也会帮你自动补全，避免误输入。而且按上下键也可以翻看历史命令。（这样不是个办法，还是好好学英语吧╮（╯＿╰）╭）

还有一个办法，是使用简化的命令。这个方法我还是相当喜欢的。不过还是建议初学者使用全称，因为只使用首字母有时候会产生歧义，比如**disable**和**delete**，gdb在你输入**d**时，会默认调用**delete**，如果你想禁用某个断点，却输入了**d 5**，那这个断点就会被删除了。

当然，熟悉了之后，还是使用快捷命令吧，会提高很多的效率。

### 二、编写测试程序

因为要调试，偶要写一个有点BUG的程序，以方便调试，希望同学们不要学习我这种自残的行为。 

``` c
[serious@localhost c]$ vim gdbtest.c
[serious@localhost c]$ cat gdbtest.c 
#include &lt;stdio.h&gt;
#include &lt;stdlib.h&gt;
#include &lt;string.h&gt;
 
void reverse(char* string) 
{ 
	int nLength = strlen(string); 
	char* temp = (char*)malloc(nLength + 1); 
 
    for(int i = 0; i < nLength; i++) 
    { 
    	temp[i] = string[nLength - i]; 
    } 
     
    strcpy(string, temp); 
    
    free(temp);
} 
     
int main(int argc, char** argv) 
{ 
	char * str = malloc(20); 
	memset(str, 0, 20); 
	strcpy(str, "hello world!"); 
 
	reverse(str); 
 
	printf("%s\n", str); 
 
	free(str);
	return 0; 
}
```

编译之：

``` bash
[serious@localhost c]$gcc -g -o gdbtest gdbtest.c -std=c99  //添加-g选项以产生调试信息 
```

这个程序的用意是将一个字符串给逆序化。请无视渣算法，一切为了调试。

运行看看我们的程序：  

``` bash
[serious@localhost c]$ ./gdbtest
[serious@localhost c]$  
```

咦？为什么没有输出呢？
那就来调试一下。

### 三、调试程序

因为刚才我们在编译程序时，已经加上了-g选项，就可以直接使用gdb进行调试了。

``` bash
[serious@localhost c]$ gdb gdbtest 
GNU gdb (GDB) 7.5 
Copyright (C) 2012 Free Software Foundation, Inc. 
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html> 
This is free software: you are free to change and redistribute it. 
There is NO WARRANTY, to the extent permitted by law.  Type "show copying" 
and "show warranty" for details. 
This GDB was configured as "x86_64-unknown-linux-gnu". 
For bug reporting instructions, please see: 
<http://www.gnu.org/software/gdb/bugs/>... 
Reading symbols from /home/serious/Desktop/c/gdbtest...done. 
(gdb) //这里就可以开始输入命令进行调试了
```

首先列出程序的内容（当然，你也可以另开一个终端界面使用vim的set nu功能显示行号）

``` bash
(gdb) list 1 
#include &lt;stdio.h&gt;
#include &lt;stdlib.h&gt; 
#include &lt;string.h&gt;
 
void reverse(char* string) 
{ 
    int nLength = strlen(string); 
    char * temp = (char*)malloc(nLength); 

    for(int i = 0; i < nLength; i++) 
``` 

list命令会一次性列出10行源代码，list后面加上1是确保可以从第一行开始显示源代码。

后面的就不列出了。我们在几个关键的位置加上断点：

``` bash
(gdb) break 7 
Breakpoint 1 at 0x4006ac: file gdbtest.c, line 7. 
(gdb) break 12 
Breakpoint 2 at 0x4006d5: file gdbtest.c, line 12. 
(gdb) b 15 
Breakpoint 3 at 0x400707: file gdbtest.c, line 15. 
(gdb) b 27 
Breakpoint 4 at 0x400777: file gdbtest.c, line 27. 
(gdb) b 29 
Breakpoint 5 at 0x400783: file gdbtest.c, line 29. 
```

然后看看我们所设置的断点：

``` bash
(gdb) info break 
Num     Type           Disp Enb Address            What 
1       breakpoint     keep y   0x00000000004006ac in reverse at gdbtest.c:7 
2       breakpoint     keep y   0x00000000004006d5 in reverse at gdbtest.c:12 
3       breakpoint     keep y   0x0000000000400707 in reverse at gdbtest.c:15 
4       breakpoint     keep y   0x0000000000400777 in main at gdbtest.c:27 
5       breakpoint     keep y   0x0000000000400783 in main at gdbtest.c:29 
```

- Num代表断点的序号，如果我们想要禁用某个断点，可以使用**disable n**命令；如果想要删除，则使用**delete n**命令。如果要全部禁用（或删除），则不需要带n。

- Type代表断点的类型。有**breakpoint**, **watchpoint**和**catchpoint**三种类型。

  - breakpoint: 就是传说中的断点啦。

  - watchpoint: 监控变量（或表达式）

  - catchpoint: 监控事件。

- Disp代表当走到当前断点时，如果此断点已被禁用，是否要显示该信息。

- Enb是断点当前是否是启用的。启用为y，禁用为n。

- Address是断点的地址。每一个断点也是有分配相应的内存的。

- What是断点的详细信息，包括所在的函数名及所在源代码行数。

开始调试：

``` bash
(gdb) run             // 运行程序
Starting program: /home/serious/Desktop/c/gdbtest
Breakpoint 4, main (argc=1, argv=0x7fffffffe1a8) at gdbtest.c:27 
27      reverse(str); 
(gdb) continue        // 继续运行程序直到下一个断点
Continuing. 
Breakpoint 1, reverse (string=0x601010 "hello world!") at gdbtest.c:7 
7       int nLength = strlen(string); 
(gdb) next            // 执行一行代码
8       char * temp = (char*)malloc(nLength); 
(gdb) print nLength   // 查看nLength的值
$10 = 12              // 长度是正确的
(gdb) next 
10      for(int i = 0; i < nLength; i++) 
(gdb) next 
Breakpoint 2, reverse (string=0x601010 "hello world!") at gdbtest.c:12 
12          temp[i] = string[nLength - i]; 
(gdb) next 
10      for(int i = 0; i < nLength; i++) 
(gdb) next 
Breakpoint 2, reverse (string=0x601010 "hello world!") at gdbtest.c:12 
12          temp[i] = string[nLength - i]; 
(gdb) print temp      // 查看temp的值
$11 = 0x601030 "" 
(gdb) continue
Continuing. 
Breakpoint 2, reverse (string=0x601010 "hello world!") at gdbtest.c:12 
12          temp[i] = string[nLength - i]; 
(gdb) print temp 
$12 = 0x601030 ""     // 已经发现问题了，连续赋值两次，temp却一直是""
(gdb)
```

程序的BUG在于，string[nLength - i]的值是'\0'，如果赋给temp[0]，则字符串temp的确就变成了""。

我们可以使用**examine**命令查看一下temp所在的内存区域：

``` bash
(gdb) examine/12ub temp 
0x601030:   0   33  100 0   0   0   0   0 
0x601038:   0   0   0   0 
```

解释一下命令：

12ub表示temp所在内存的显示方式，12u代表显示12个计量单位，b代表字节，所以此命令是显示以temp作为起始地址，向后显示12个字节的内容。可以看到，第一个字节是0，第二个字节是33，即'!'。

更多格式，请man gdb。

不需要退出程序，我们直接修改程序：

``` bash
(gdb) shell 
[serious@localhost c]$ vim gdbtest.c 
[serious@localhost c]$ gcc -g -o gdbtest gdbtest.c -std=c99 
[serious@localhost c]$ exit 
exit 
(gdb) 
```

将源代码第12行修改为:

``` bash
temp[i] = string[nLength - i - 1];
```

再次调试到给temp赋值的部分，看一下temp：

``` bash
(gdb) p temp 
$13 = 0x601030 "!dlr" 
```

有童鞋问了，老这样输出temp好烦啊，有什么其他办法吗？

当然有。使用**watch**命令。

在调试到第10行时，使用**watch**：

``` bash
(gdb) n 10		for(int i = 0; i < nLength; i++)
(gdb) watch temp[i] 
Hardware watchpoint 12: temp[i] 
(gdb) c 
Continuing. 
Hardware watchpoint 12: temp[i] 
Old value = 0 '\000' 
New value = 33 '!' 
reverse (string=0x601010 "hello world!") at gdbtest.c:10 
10      for(int i = 0; i < nLength; i++) 
(gdb)
```

可以看到，当temp[i]有变化时，gdb会提示你，值有变化，从'\000\变化到了'!'。

程序最后的运行结果：

``` bash
[serious@localhost c]$ ./gdbtest
!dlrow olleh
```

### 四、容易出现的问题

1.在使用gdb时，有时会出现这样的问题：

``` bash
(gdb) n
Missing separate debuginfos, use: debuginfo-install glibc-2.12-1.80.el6.x86_64
(gdb) 
``` 

解决办法：编辑一下debuginfo的源，并使用debuginfo-install命令更新debuginfo库。

``` bash
[serius@localhost c]$ vim /etc/yum.repos.d/CentOS-Base-debuginfo.repo: 
// 添加如下源
[base-debuginfo]
name=CentOS-$releasever -DebugInfo
baseurl=http://debuginfo.centos.org/$releasever/$basearch/
gpgcheck=0
enabled=0
protect=1
priority=1
``` 

2.也有可能在调试你的程序时出现：

``` bash
(gdb) n 
Single stepping until exit from function yourfunc, which has no line number information. 
(gdb) 
``` 

yourfunc是你自己的函数，这表示着含有此函数的代码在编译时并没有加上-g选项。

还有一种可能，是你的gcc版本过高而gdb版本过低，比如我。CentOS6.4，4.8.0版本gcc，却是7.2.0版本的gdb，无奈先升级了gdb，才避免了这个错误。

3.在**watch**变量时出现：

``` bash
(gdb) watch var
No symbol "var" in current context. 
``` 

很明显了，gdb在当前的代码域内找不到var变量，就会报这种错误。

### 五、总结

总的来说，gdb是个强大的调试器，我在上面说的这一堆，也只是沧海一粟而已，gdb的官方pdf文档已经堆到了662页。。我也不可能在一篇文章里说清楚，所以，如果你想知道更多，就去下载文档阅读吧。

[凶猛的传送门](http://sourceware.org/gdb/current/onlinedocs/gdb.pdf.gz)

