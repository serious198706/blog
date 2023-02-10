---
id: 848
title: 使用GAE+GoAgent科学上网
author: 死神的微笑
layout: post
guid: http://cy198706.com/blog/?p=848
permalink: /just-try/gae-proxy-goagent/
cleanretina_sidebarlayout:
  - default
views:
  - 231
duoshuo_thread_id:
  - 1320102247610187815
bot_views:
  - 7
categories:
  - 折腾
tags:
  - GAE
  - Google
---
终于要聊到这个话题了吗。。

因为我最近发现，Google老是屏蔽我，我搜索了一个完全不会触及任何底线的词，竟然就把我Ban掉了，导致全公司都上不了Google。。

之前折腾过GAE+GoAgent，也成功过，后来发现没什么大用，因为上班时又不能老挂着GoAgent，毕竟用内网的时候还是居多的。但这次实在是忍不了了，对不起了，伟大的GFW，我要跃过你丫的。  
<!--more-->

  
废话不说，先去创建一个Google的账号，如果你没有账号的话，那还聊什么聊。  
&nbsp;

#### 一、申请Google Appid

1. 进入[Google AppEngine][1]，点击&ldquo;Create Application&rdquo;，如果是第一次创建的话，应该会提示要输入手机号码进行验证，大胆地验证吧，Google不会坑你的。

验证完成后，进入如下界面：  
<a href="http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/QQ20130205112620.jpg" target="_blank"><img alt="" src="http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/QQ20130205112620.jpg" /></a>  
如果是第一次创建应用的话，下方会提示同意谷歌的协议，勾选上&ldquo;I accept these terms.&rdquo;就可以了。

2. 填写你的Appid，记得检查一下可用性，因为Appid必须是唯一的。  
然后填入应用的名称，随便起啦，比如myagent-byebyeGFW之类的。。但是要注意，名称是有规定的，&ldquo;只能在4-30个字符之间，字母、数字、引号、连字符、逗号和叹号为可用字符。&rdquo;习惯写C和Java的同学们要注意，不要加下划线上去。

3. 点击&ldquo;Create Application&rdquo;。好哎，出现如下页面，表示你的App已经创建成功了！  
[![][2]][2]  
网页上的操作完成，接下来是本地操作。

4. 去[传送门][3]这儿下载GoAgent的程序包，如果下载不了的话（嗯，你被墙了），可以下载文章最后提供的附件。

5.&nbsp;解压GoAgent到任意位置。进入Server文件夹，用文本编辑器打开python文件夹下的app.yaml文件，将第一行Application:your\_appid后的your\_appid更换为你刚才申请的appid。保存退出。

6. 以管理员身份运行Uploader.bat，提示输入Appid，输入邮箱（注册Google账号时的邮箱），输入密码，就自动开始上传了。  
[![][4]][4]

到最后，出现这个东西，就表明成功鸟。  
[![][5]][5]  
&nbsp;

#### 二、配置GoAgent客户端

1. 进入local文件夹（与Server文件夹同级目录），打开proxy.ini，修改[gae]模块下的appid字段为你刚才申请的appid。保存退出。

2. 以管理员身份运行goagent.exe，出现此窗口，这个窗口就开着，它就是你的&ldquo;门&rdquo;。  
[![][6]][6]  
&nbsp;

#### 三、安装浏览器插件

1. 如果你使用的是chrome浏览器，去[这儿][7]下载SwitchySharp插件，安装。然后打开SwitchySharp，如下图所示，点击&ldquo;从文件恢复&rdquo;导入一个备份文件，该文件在文章最后提供下载。  
[![][8]][8]

如果你使用的是Firefox浏览器，去[这儿][9]下载FoxyProxy插件，安装。然后打开SwitchySharp，点击&ldquo;新建代理服务器&rdquo;，接着在&ldquo;手动配置代理服务器&rdquo;项目中输入ip地址&ldquo;127.0.0.1&rdquo;，端口号8087。点击确定，会弹出一个提示，问你是不是在浏览所有的网址时都要使用代理，因为你没有添加白名单。点击是，就配置完成了。  
[![][10]][10]

2. 启用插件  
Chrome中点击SwitchySharp按钮，选择&ldquo;GoAgent&rdquo;，记得我们刚才还开着的&ldquo;门&rdquo;吗？现在Chrome就从这道&ldquo;门&rdquo;里出去了。  
[![][11]][11]

Firefox中右键点击FoxyProxy按钮，选择&ldquo;为全部URLs启用代理服务器:127.0.0.1:8087&rdquo;，就可以了。  
[![][12]][12]

####   
四、疑难杂症

有的时候在访问Twitter, Youtube时会提示&ldquo;该网站的安全证书不受信任&rdquo;，如图所示  
[![][13]][13]

原因是在使用GoAgent代理的时候，Twitter等网站是需要SSL加密的，GoAgent  
GoAgent是个代理，好比就是个web服务器，你通过他浏览https的加密网页的时候，要做认证再转发给你的浏览器。如果不加证书，你的浏览器认为这个https网站是伪造的，不让你打开。特别是Chrome和Firefox，要求中转的代理网站浏览https网页是必须要证书，否则不能打开。这个是浏览器自身的防止钓鱼攻击的安全手段。

解决的办法，就是找到刚才下载的GoAgent，在local文件夹里，找到CA.crt文件，就是GoAgent的提供的证书文件（他们也是花钱买的哦），按照如下步骤导入到系统中。  
1. 双击打开CA.crt文件，出现证书导入向导。点击&ldquo;安装证书&rdquo;。  
[![][14]][14]

2. 选择&ldquo;本地计算机&rdquo;（当然，选择&ldquo;当前用户&rdquo;也可以，只不过切换用户时就失效了），点击&ldquo;下一步&rdquo;。  
[![][15]][15]

3.&nbsp;选择&ldquo;将所有的证书都放入下列存储&rdquo;，点击&ldquo;浏览&rdquo;。  
[![][16]][16]

4.&nbsp;选择&ldquo;受信任的根证书颁发机构&rdquo;，点击&ldquo;确定&rdquo;。  
[![][17]][17]

5.&nbsp;点击&ldquo;下一步&rdquo;，再点击&ldquo;完成&rdquo;，出现导入成功的窗口。  
[![][18]][18]

为什么要自己签名根证书呢？因为GAE平台限制，没法支持真正的SSL加密，GoAgent只能通过伪造证书的方式做到代理SSL加密的网站，这个证书就是用来欺骗浏览器的。  
&nbsp;

#### 五、玩吧

[![][19]][19]

[![][20]][20]

<a class="readmore" href="http://pan.baidu.com/share/link?shareid=265965&uk=151049050" target="_blank"><img alt="" src="http://cy198706.com/blog/wp-content/uploads/2013/04/download.png" /></a>  
&nbsp;

 [1]: https://appengine.google.com
 [2]: http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/QQ20130205112538.jpg
 [3]: https://code.google.com/p/goagent/
 [4]: http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/QQ20130205113930.jpg
 [5]: http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/QQ20130205122648.jpg
 [6]: http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/QQ20130205122830.jpg
 [7]: https://chrome.google.com/webstore/detail/proxy-switchysharp/dpplabbmogkhghncfbfdeeokoefdjegm
 [8]: http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/QQ20130205123020.jpg
 [9]: https://addons.mozilla.org/zh-cn/firefox/addon/foxyproxy-standard/
 [10]: http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/QQ20130205123725.jpg
 [11]: http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/QQ20130205124528.jpg
 [12]: http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/QQ20130205123825.jpg
 [13]: http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/QQ20130205125036.jpg
 [14]: http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/QQ20130205125603.jpg
 [15]: http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/QQ20130205125618.jpg
 [16]: http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/QQ20130205125626.jpg
 [17]: http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/QQ20130205125635.jpg
 [18]: http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/QQ20130205125655.jpg
 [19]: http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/QQ20130205133033.jpg
 [20]: http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/QQ20130205132928.jpg