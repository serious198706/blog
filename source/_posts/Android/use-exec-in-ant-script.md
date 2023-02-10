---
title: 在ant中使用exec更新版本号
link: ant-with-exec
cover: http://blog.cigis-cloud.com/android-1593080864.jpg
category: 
    - Development
tags: 
    - Android
date: 2015-07-17 13:08:08
---


最近在Android中用Ant自动打包的过程中，有个要求，是从Git获取版本提交的总次数，加上最初版本的版本号，做为最新版本的版本号。

起初是写了一个shell脚本来做这件事。

```sh
#!/bin/bash

cd /home/serious/Workspace/client/
count=`git rev-list origin/master --count`
result=2.0.$count
echo $result

sed -i "s/android:versionName=\"2.0.*\"/android:versionName=\"${result}\"/g" AndroidManifest.xml
```

但是这么一来，打包之前就会多一个步骤。我这么懒的人，怎么能允许这种事情发生？怎么能坐以待毙？

于是就翻看了ant的文档，找到解决方案。

<!-- more -->

在build.xml中，加入这么一段：

```xml
<target name="versioncode">
    <exec executable="sh">
        <arg value="update_vercode.sh" />
    </exec>
</target>
```

搞定。

但是，这样还多了个sh文件啊。我作为一个并不是处女座的洁癖，怎么能允许这种事情发生？怎么能坐以待毙？

于是找到了下面的解决方案：

```xml
<target name="versioncode">
        <exec executable="sh" outputproperty="v_name">
            <arg value="-c" />
            <arg value="git rev-list origin/master --count" />
        </exec>
        <echo>Revision (app): ${v_name}</echo>
        <replaceregexp file="AndroidManifest.xml" match='android:versionName="2.0.*"' replace='android:versionName="2.0.${v_name}"' />
</target>
```

Ant中执行系统命令时（比如上文的git命令和sh命令），在Windows下和在Linux下的方式是不同的。

- Windows

```xml
<target name="help">
  <exec executable="cmd">
    <arg value="/c"/>
    <arg value="ant.bat"/>
    <arg value="-p"/>
  </exec>
</target>
```

- Linux

```xml
<target name="help">
  <exec executable="sh">
    <arg value="-c"/>
    <arg value="ant.sh"/>
    <arg value="-p"/>
  </exec>
</target>
```