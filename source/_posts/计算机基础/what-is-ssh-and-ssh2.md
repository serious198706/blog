---
title: SSH 与 SSH2
link: ssh-and-ssh2
cover: https://kinsta.com/wp-content/uploads/2015/04/connect-to-ssh-1024x512.png
category:
  - Development
tag:
  - 计算机基础知识
date: 2015-07-23 22:38:01
---

什么是SSH和SSH2？看完这篇你就懂啦。

<!-- more -->

* SSH(Secure Shell)在你通过网络登录到另一台计算机、远程执行命令或者传输文件时，为你提供一个加密的通道。SSH提供一个强有力的主机对主机、主机对用户的认证，就如同网络上的加密通信一样。

* SSH2是一个更加安全、有效，移植性强的SSH版本，它包含SFTP功能。SFTP与FTP功能相同，但SFTP是加密的。在印弟安那大学，UITS([University Information Technology Services](https://uits.iu.edu/))已将其中心系统升级到了SSH2（通常都使用OpenSSH），并且提倡用户都使用SSH2来进行网络加密通讯。

Mac OS X 集成了OpenSSH。在Windows下，则需要下载一个第三方的SSH客户端，比如SecureCRT和Putty。

当第一次连接到某台主机时，SSH将给你提供主机的指纹，并且询问你是否要将这个新的指纹和主机的路径存储到本地数据库中。在同意之前，你应该好好比对一下这个指纹是否与你之前得知的指纹（谁管你怎么得知的，打电话呗）相同，以避免连接到一个江湖骗子的主机上。输入`yes`之后，这条消息就不会再出现了。

比验证密码更好的，SSH2能使用公用密钥来进行主机的认证。例如，如果你想连接到某台远程主机，名称为host.cy198706.com （当然正在使用SSH2），SSH2会使用这套机制来验证这台主机是否可登录。如果你愿意的话，你可以在登录主机时，设置SSH2使用公钥来进行认证，而不是通过密码认证。如何设置，请看[如何设置SSH使用公钥认证登录主机](/ssh-using-public-key/)。