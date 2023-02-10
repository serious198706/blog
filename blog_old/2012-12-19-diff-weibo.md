---
title: '浏览器插件：微博急简 &#8211; 还你一个，干净的微博！'
date: 2012-12-19
tags: 旁门
---
这年头，无孔不入的广告已经把我们烦得要死要活的，再加上一些&ldquo;潜规则&rdquo;，本来有些地方是&ldquo;一方净土&rdquo;，也会在利益的驱使下变得污浊不堪。  
当然，这只是些无关痛痒的话，现在谁还会因为平民的呼声就做出一些&ldquo;艰难的决定&rdquo;啊。

新浪微博从一开始的简洁明快，到后来的臃肿不堪，有利益在里面，也有事物的发展规律在里面。从利益方面，没有任何一个大型网站是不需要挣钱的，仅靠那一点点广告费，是不足以养活下面的百十号人的。  
<!--more-->


那怎么办呢？会员？嗯！放点权限！会员可以知道谁取消了对他的关注！会员可以用专属的模版！会员应该戴一个皇冠！会员的话可以被更多人看到！

那怎么才能让会员的话被更多人看到呢？排序吧！

<pre class="brush:cpp;first-line:1;pad-line-numbers:true;highlight:null;collapse:false;">if(user.group == _VIPGROUP)
{
&nbsp;&nbsp;&nbsp; pop_sort();
}

if(user.group == _NORMALGROUP)
{
&nbsp;&nbsp;  push_sort();
}

</pre>

废话少说，新浪微博的页面上有太多平时用不到的东西了，这也是为什么我喜欢wap版的微博的原因，简洁明快，我能很快地找到我需要的东西。  
<a href="http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/20121219094357.jpg" target="_blank"><img alt="" src="http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/20121219094357.jpg" /></a>  
看看这东西，多臃肿，太多平时用不到的东西了。。我就是看微博，发微博，没了。

这今天推荐一个微博急简工具，他们的口号是&ldquo;还你一个，干净的微博！&rdquo;针对FireFox，Chrome，Safari和360浏览器分别开发了插件，他们对待IE用户的态度是：I'm sorry, I'm so sorry, I'm sooooo sorry.  
效果更是没话说，上图：<a href="http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/20121219092616.jpg" target="_blank"><img alt="" src="http://i1352.photobucket.com/albums/q645/cy198706/mess-ups/20121219092616.jpg" /></a>  
因为在Win8下，Mactype不能很好地渲染Chrome，看着那蛋疼的字体实在让人蛋疼，所以偶目前使用的是火狐浏览器，还不错吧？  
链接奉上：[微博急简][1]

<!--[syntaxhighlighter]-->

<!--代码高亮，请勿编辑-->

<link type="text/css" rel="stylesheet" href="http://cy198706.com/blog/wp-content/plugins/ck-and-syntaxhighlighter/syntaxhighlighter/styles/shCoreCk.css" />

<link type="text/css" rel="stylesheet" href="http://cy198706.com/blog/wp-content/plugins/ck-and-syntaxhighlighter/syntaxhighlighter/styles/shThemeCk.css" />

<!--[/syntaxhighlighter]-->

 [1]: http://www.diff.im/weibo_wc/
