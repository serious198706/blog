---
title: Ubuntu 下使用 crontab 定时执行任务
link: crontab-in-ubuntu
category:
  - Linux
tag:
  - Linux
date: 2013-02-28 18:59:21
---


在以前的工作里，crontab也用了一段时间，做为一个计划任务软件，crontab已经做得足够棒了，它可以完成绝大部分定时工作。

<!-- more -->

### 基于用户的cron

而且用法极其简单：

``` bash
serious@serious:~/Desktop$ crontab [-u username] [-l|-r|-e]

参数：
-u：设定某个用户的cron服务，一般root用户在执行这个命令的时候需要此参数
-l：列出某个用户cron服务的详细内容
-r：删除没个用户的cron服务
-e：编辑某个用户的cron服务

如果想查看某个用户的计划任务列表：
crontab -u username -l
删除某个用户的计划任务列表：
crontab -u username -r
针对某个用户编辑计划任务：
crontab -u username -e
``` 

第一次进入crontab编辑时，会询问要选择哪种编辑器，哪种顺手就选哪种呗。

``` bash
serious@serious:~/Desktop$ crontab -e
no crontab for serious - using an empty one

Select an editor.  To change later, run 'select-editor'.
  1. /bin/ed
  2. /bin/nano        <---- easiest
  3. /usr/bin/emacs23
  4. /usr/bin/vim.basic
  5. /usr/bin/vim.tiny

Choose 1-5 [2]: 4
``` 

默认为nano编辑器，我选择4，正常版的vim编辑器。

接下来在打开的vim编辑器中，可以看到如下内容：

``` bash
# Edit this file to introduce tasks to be run by cron.
# 
# Each task to run has to be defined through a single line
# indicating with different fields when the task will be run
# and what command to run for the task
# 
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').# 
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
# 
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
# 
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
# 
# For more information see the manual pages of crontab(5) and cron(8)
# 
# m h  dom mon dow   command
``` 

可以阅读一下它给你的说明。

然后在文件的最后添加一行

``` bash
30  16  *  *   *   date >> ~/time.log
#分 时  日  月  周  <-------命令------>
``` 

然后保存退出，这样在/var/spool/cron/crontabs/里可以看到你的定时计划文件了。记得使用root查看。

说一下每一个参数的信息：

数字范围：
<img alt="" src="http://images.cy198706.com/Programming/QQ20130228164906.png" />


可用字符：
<img alt="" src="http://images.cy198706.com/Programming/QQ20130228165829.png" />

现在就来举各种各样的例子吧。

你的女朋友的生日是每年的2月3日，你想在2月2日的23：59分发一封充满爱意的邮件给她，就可以使用如下命令：

``` bash
59 23 2 2 * mail sweety < /home/serious/ToMyLover.txt
``` 

哦对了，记得每年把信的内容变一变。

每5分钟执行一次/home/serious/test.sh文件：

``` bash
*/5 * * * * /home/serious/test.sh
``` 

每周五下午5：30发封邮件提醒你的朋友，马上要下班啦：

``` bash
30 17 * * * mail friend@somewhere.com < /home/serious/TGIF.txt
``` 


### 基于系统的cron

如果是系统例行任务呢？那就不再需要使用crontab -e来完成任务了，可以直接编辑/etc/crontab文件。

有一点需要特别注意一下，crontab -e这个crontab其实是/usr/bin/crontab这个执行文件，而且/etc/crontab是一个纯文本文件，你可以使root身份编辑该文件。

看看这个文件的内容吧：

``` bash
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
``` 

看起来跟刚才的crontab -e内容差不多啊，只是有一个地方不太相同，这里的命令需要提供用户名。

### 常见错误

#### 1.找不到crond服务

``` bash
root@serious:~# service crond restart
crond: unrecognized service
``` 
怎么会找不到crond服务呢？难道安装的时候喝high了，忘记安装crond组件？不可能。。这是集成的。

Google了一番，发现在Ubuntu里，crond服务改头换面，变成cron了，所以正确的命令应该是：

``` bash
root@serious:~# service cron restart
或者是
root@serious:~# /etc/init.d/cron restart
``` 


#### 2.crontab设置完后不执行。

首先检查路径，在crontab中执行文件时，最好使用绝对路径，否则绝大部分不执行的原因都是因为路径问题。

第二，由于crontab机制问题，crontab每分钟会帮我们重新读取一次/etc/crontab，但在有些*nix系统中，crontab是读到内存中的，这个时候，就要将cron服务重新启动一遍了。

第三，环境变量问题。

比如我用C++写了个程序，需要使用crontab执行，但是这个程序的运行是需要环境变量支持的，那么就必须要在crontab文件中加入环境变量，否则会有各式各样的错误提示。

#### 3.我一下子选错了编辑器肿么办。

执行如下命令，重新选择即可：

``` bash
serious@serious:~# select-editor
``` 


