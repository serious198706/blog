---
title: 如何设置SSH使用公钥认证登录主机
link: ssh-using-public-key
cover: 
category:
  - Development
tag:
  - 计算机基础知识
date: 2015-07-24 19:20:43
---

<div id="menu">
</div>

[总览](#Overview)

[在Linux或OS X上设置SSH使用公钥](#Linux)

[在Windows上使用PuTTY设置SSH使用公钥](#Windows)

<!-- more -->

###总览

在使用SSH时，利用公钥来进行认证后的链接，是比只利用密码来进行验证要安全得多的。SSH公钥认证是基于非对称加密算法的，这种加密算法会生成一对key，一个“私有”key，也就是私钥，和一个“公有”key，即公钥。你在你的电脑上只保存一个私钥，用来进行远程连接。同时你也可以把公钥分享给任何人（不要给他私钥哟），或者存储在远程主机的某个目录上，比如 `.ssh/autorized_keys`。

使用SSH公钥认证：

远程主机必须要安装了SSH（任意版本）。本文假设远程主机使用了OpenSSH（对，就是上一篇文说的印第安那大学的UTIS中心系统使用的那个）。如果远程主机正在使用一个奇怪的SSH版本（比如Tectia SSH），下文即将使用的方法可能不正确。

你的电脑也必须安装某种版本的SSH工具。本文假设你在Linux或Mac OS X下使用命令行中的SSH，在Windows下使用PuTTY。

还有个前提是你能够把公钥想办法传输到远程主机上。使用用户名密码登录到远程主机或者有网络管理员来帮你做这件事。

公钥可以存储到 `~/.ssh/authorized_keys`目录下。

[Back to top](#menu)

<div id="Linux">
</div>

###在Linux或OS X上设置SSH使用公钥

1. 使用命令行的SSH创建一对Key，加密算法可以是DSA或者RSA：

* 创建DSA Key，在命令行中输入：
	
``` bash
ssh-keygen -t dsa
```

* 创建RSA Key，在命令行中输入：
	
``` bash
ssh-keygen -t rsa
```

2. 接下来ssh-keygen会提醒你提供一个文件名来储存key文件，同时提供一个密码来保护你的私钥。

	* 文件名：如果你要使用默认的文件名（和路径）的话，直接回车就好了。如果你想指定文件名（和路径）的话，就输入路径并回车。
		但是很多远程主机设置为只接受使用默认路径和默认文件名的私钥，在这种情况下，如果要使用其他文件名的私钥，就要做一点点设置了，下文会有介绍。

	* 密码：输入不少于5个字符的密码，然后回车。如果不想使用密码，则可以直接回车。
		*UITS强烈建议使用密码来保护你的私钥。如果不使用密码的话，任何能访问你电脑的人都可以使用这个私钥来进行远程主机的登录。*

	你的私钥按照你的输入保存在相应的位置（`~/.ssh/id_rsa`或者 `~/.ssh/my_ssh_key`）。
	而相对应的公钥也会保存在与私钥相同的位置，并且会在私钥的文件名后面加一个.pub的后缀（`~/.ssh/id_rsa.pub`或者 `~/.ssh/my_ssh_key.pub`）。

3. 使用SFTP或者SCP将公钥拷贝到远程主机上。例如：
	``` bash
    scp ~/.ssh/id_rsa.pub serious@cy198706.com:  // 不要忘记后面这个"`:`"
	```

	接着输入你在该远程主机上的用户密码，认证通过之后，公钥就会被拷贝到远程主机你的home目录下。

4. 使用用户名密码登录远程主机。
	*如果你没有权限登录远程主机，那么你需要联系网络管理员来帮你创建*`~/.ssh/authorized_keys`*文件。*

5. 如果你的远程主机中没有`~/.ssh/authorized_keys`文件，则需要手动创建一个：
	``` bash
    mkdir -p ~/.ssh
  	touch ~/.ssh/authorized_keys
    ```
	*如果已有这个文件，则上面的命令并没有什么卵用*

6. 将步骤3中拷贝过来的公钥文件（`~/id_rsa.pub`）的内容写入 `~/.ssh/authorized_keys`文件中。
	``` bash
    cat ~/id_rsa.pub > ~/.ssh/authorized_keys
    ```

	然后检查一下是否写入成功：
	``` bash
    cat ~/.ssh/authorized_keys | more
    ```

7. 现在可以安全地删除公钥文件了：
	``` bash
    rm ~/id_rsa.pub
    ```

8. 你可以在其他的主机上也添加这个公钥，只需要重复3-7步骤就可以啦。

9. 现在你应该可以使用SSH登录到远程主机了。登录的主机是[cy198706.com](http://cy198706.com)，登录时的用户名是serious，登录所用的电脑就是存有你刚才创建的私钥的这台电脑。

	如果你刚才创建私钥时使用密码保护了，那么远程主机会要求你在SSH时输入密码（该密码并不会被传输到远程主机，只做本地验证）：
    ``` bash
    [serious@desktop ~]$ ssh serious@cy198706.com
	Enter passphrase for key '~/.ssh/id_dsa':
 	Last login: Fri Jul 24 09:23:17 2015 from serious.somewhere.org
  	```

	如果你刚才在创建私钥时没有使用默认的文件名（或路径），则你需要用以下两种方法来进行SSH：
    * 在命令行中加入-i参数，并指定私钥的路径。例如你刚才把私钥存在了`~/.ssh/mykeys/my_host`，并要使用这个私钥来进行SSH的话：

    ``` bash
  	ssh -i ~/.ssh/mykeys/myhost serious@cy198706.com
	```

	* 修改配置文件。
	SSH会依次从以下几处读取配置信息：
        1. 命令行参数
    	2. ~/.ssh/config（也许不存在）
    	3. /etc/ssh/ssh_config两个文件中读取配置信息

	SSH配置文件是一个包含关键词和参数的文本文件。要指定连接某台远程主机时要使用哪个私钥，只需要使用文件编辑器添加一条记录即可。

    例如，要连接到[cy198706.com](http://cy198706.com)，同时使用`~/.ssh/mykeys/cy_host`这个私钥，则需要在 `~/.ssh/config`（如果没有自行创建）文件中添加如下两行：

  	``` bash
  	Host cy198706.com
    IdentityFile ~/.ssh/mykeys/cy_host
 	```

	保存文件之后，SSH就会利用这个规则来访问指定的主机。

	你也可以添加多条规则：

	``` bash
   	Host host1.cy198706.com
    IdentityFile ~/.ssh/mykeys/cy_host1

  	Host host2.cy198706.com
    IdentityFile ~/.ssh/mykeys/cy_host2

  	Host host3.cy198706.com
    IdentityFile ~/.ssh/mykeys/cy_host3
 	```

	甚至可以使用通配符`*`来指定规则：

	``` bash
  	Host *.cy198706.com
    IdentityFile ~/.ssh/mykeys/cy_host
 	```

	要想得知更多关于SSH配置文件的用法，查看[帮助页面](https://kb.iu.edu/d/afjm)，或者`man ssh_config`。


[Back to top](#menu)
<div id="Windows">
</div>

###在Windows上使用PuTTY设置SSH使用公钥

*PuTTY命令行客户端，还有用来生成key的PuTTYgen工具，以及Pageant SSH认证代理工具，还有PuTTY SCP，Putty SFTP工具都是打包在一起下载的。打包版可以在[这儿](http://the.earth.li/~sgtatham/putty/latest/x86/putty.zip)下载到。安装版可以在[这儿](http://the.earth.li/~sgtatham/putty/latest/x86/putty-0.64-installer.exe)下载到。*

*墙内的同学你们有福了，该网站已被墙得体无完肤，我上传到了[度娘网盘](http://pan.baidu.com/s/1eQpmNAE)提供下载。*

*下载完成后都有这些文件：*
<figure>
  <img src="{{ site.url }}/images/putty1.png" alt="PuTTY">
</figure>

1. 打开PuTTYgen，如图所示
<figure>
  <img src="{{ site.url }}/images/putty2.png" alt="PuTTY">
</figure>

2. 在下方的`Parameters`区域，可以选择`SSH-2 RSA`或者`SSH-2 DSA`加密方式。`Number of bits in a generated key`参数使用默认值就可以。

3. 点击`Generate`，上方会出现一个进度条，并且提示你在空白区域移动鼠标（别移出去了！눈_눈），它就会随机生成一对key。然后公钥就会显示在上方的框里。如图所示。
<figure>
  <img src="{{ site.url }}/images/putty3.png" alt="PuTTY">
</figure>

4. 在`Key passphrase`和`Confirm passphrase`输入框中，输入一个密码来保护你的私钥。当然你也可以不输入。
*UITS强烈建议使用密码来保护你的私钥。如果不使用密码的话，任何能访问你电脑的人都可以使用这个私钥来进行远程主机的登录。*

5. 点击下方`Save public key`，指定公钥的名称，保存公钥。

6. 点击下方`Save private key`，指定私钥的名称，保存私钥。
	*如果你没有使用密码保护私钥的话，这时PuTTYgen会提醒你是否确定不使用密码保护。*

7. 使用用户名密码登录远程主机。
	*如果你没有权限登录远程主机，那么你需要联系网络管理员来帮你创建*`~/.ssh/authorized_keys`*文件。*

8. 如果你的远程主机中没有`~/.ssh/authorized_keys`文件，则需要手动创建一个：
	``` bash
    mkdir -p ~/.ssh
  	touch ~/.ssh/authorized_keys
    ```
	*如果已有这个文件，则上面的命令并没有什么卵用*

9. 在你的电脑上，在PuTTYgen中，将公钥的内容拷贝一下，然后，在远程主机中把内容复制到`~/.ssh/authorized_keys`文件中。

10. 打开Pageant SSH认证代理工具。这玩意会在后台运行，你只能在系统托盘区域看到它。右击托盘区域中的图标，选择Add Key，然后找到你刚才保存的私钥，选择之。

	如果你在创建私钥时使用了密码保护，Pageant会询问你的密码是什么。

11. 现在Pageant会把私钥保存在内存中，当你使用PuTTY访问远程主机时，该私钥就会起作用。

12. 打开PuTTY客户端。如图。
<figure>
  <img src="{{ site.url }}/images/putty4.png" alt="PuTTY">
</figure>

	* `Session`页面的`Host Name (or IP address)`输入框中，输入远程主机的用户名和地址，例如`serious@cy198706.com`。

	* 在`Connection type`中，选择SSH

	* 在左侧列表的`Category`中，找到`Auth`页面（`Connection > SSH > Auth`）。在`Auth`页面的`Authentication methods`区域中，勾选`Attempt authentication using Pageant`。
<figure>
  <img src="{{ site.url }}/images/putty5.png" alt="PuTTY">
</figure>

	* 返回`Session`页面，在`Saved Sessions`区域中输入一个名称用来保存这套配置，然后点击保存。

	* 点击`Open`连接到远程主机。当Pageant在后台运行时，PuTTY将会自动获得私钥并使用，在登录到远程主机的过程中不会再次询问私钥的保护密码之类的信息。

	*到这儿配置基本完成。需要注意的是，每次在开机之后，如果要使用Pageant，就必须手动启动它，添加私钥，再使用PuTTY。当然也有快捷的办法，下文将会继续介绍。*

13. 打开“启动”文件夹：
	* Windows 8下，按Windows+R，输入`shell:startup`后回车就打开了。
	* Windows 7下，从开始菜单，点击所有程序，找到启动菜单，右击打开。
	* 或者直接导航到“启动”文件夹：
	
		``` bash
    	C:\Users\your_name\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup
    	```

14. 在文件夹中右键新建一个快捷方式。然后在弹出的对话框的输入框中输入你Pageant.exe的路径，也可以使用浏览找到它。接着在该路径的后面加一个空格，输入私钥的位置。例如：

    ``` bash
    "C:\Program Files (x86)\PuTTY\pageant.exe" "C:\Users\your_name\ssh_key\putty_private.ppk"
    ```

	在下一步中，给快捷方式取个名字，点击完成。

	这样在以后电脑开机之后，就会自动加载该私钥到Pageant中，省去了手动启动的麻烦。

[Back to top](#menu)

