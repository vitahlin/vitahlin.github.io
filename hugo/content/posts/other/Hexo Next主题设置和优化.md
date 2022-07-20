---
title: hexo Next主题设置和优化
<!-- description: 这是一个副标题 -->
date: 2022-07-20
slug: hexo-next-theme
categories:
    - tools

tags:
    - tools
---

# 主题样式优化


## 修改打赏字体不闪动

修改文件`next/source/css/_common/components/post/post-reward.styl`，然后注释其中的函数`wechat:hover`和`alipay:hover`，如下：
```css 
/* 注释文字闪动函数
 #wechat:hover p{
    animation: roll 0.1s infinite linear;
    -webkit-animation: roll 0.1s infinite linear;
    -moz-animation: roll 0.1s infinite linear;
}
 #alipay:hover p{
   animation: roll 0.1s infinite linear;
    -webkit-animation: roll 0.1s infinite linear;
    -moz-animation: roll 0.1s infinite linear;
}
*/
```

## 采用多说时文章评论数目不显示

当采用多说评论系统的时候，不知道因为什么原因，文章的评论数目不能正常显示，显示效果如下，分类后面不能正常显示评论数目，但有分隔号：
![](http://oc1mf55gf.bkt.clouddn.com/A0692BE3-87BE-4BFC-9C7F-5152B62B17EB.png)


修改文件`themes\next\source\css\_common\components\post\post-meta.styl`，将原来的代码：
```css  
.posts-expand .post-comments-count {
  +mobile() { display: none; }
}
```

改为：
```css 
.posts-expand .post-comments-count {
  display: none; 
}
```
使文章始终不显示评论数目，而不仅仅在手机模式下。



## 修改内链文字样式

原来版本的内链样式跟默认的字体很类似，这里进行修改。将链接文本设置为蓝色，鼠标划过时文字颜色加深，并显示下划线。修改文件`themes\next\source\css\_common\components\post\post.styl`，添加如下css样式：
```c 
.post-body p a{
  color: #0593d3;
  border-bottom: none;

  &:hover {
    color: #0477ab;
    text-decoration: underline;
  }
}
```

选择`.post-body`是为了不影响标题，选择`p`是为了不影响首页“阅读全文”的显示样式。


参考链接
- [http://www.wuxubj.cn/2016/08/Hexo-nexT-build-personal-blog/](http://www.wuxubj.cn/2016/08/Hexo-nexT-build-personal-blog/)

## 修改小段的代码样式

主题原来的小段代码样式跟默认文章字体有点类似，这里可以修改其样式显示的更清晰点。
修改文件`themes\next\source\css\_common\components\hightlight\hightlight.style`文件，找到其中的`code`段代码，改为如下形式：
```c 
code {
  padding: 2px 6px;
  word-wrap: break-word;
  color: $highlight-red;
  background: #f9f2f4;
  border-rdius: $code-border-radius;
  font-size $code-font-size+1;
}
```
其中的`color`就是字体对应的颜色，`background`对应背景颜色。

## 修改侧边栏头像为圆形

修改文件`themes\next\source\css\_common\components\sidebar\sidebar-author.style`，修改其中的`.site-author-image`段代码，改为如下形式：
```css 
.site-author-image {
  display: block;
  margin: 0 auto;
  max-width: 96px;
  height: auto;
  border: 2px solid #333;
  padding: 2px;
  border-radius: 50%;
}
```

如果需要添加鼠标停留在上面发生旋转效果，那么添加如下代码：
```css 
.site-author-image {
  display: block;
  margin: 0 auto;
  max-width: 96px;
  height: auto;
  border: 2px solid #333;
  padding: 2px;

  border-radius: 50%
  webkit-transition: 1.4s all;
  moz-transition: 1.4s all;
  ms-transition: 1.4s all;
  transition: 1.4s all;
}

.site-author-image:hover {
  background-color: #55DAE1;
  webkit-transform: rotate(360deg) scale(1.1);
  moz-transform: rotate(360deg) scale(1.1);
  ms-transform: rotate(360deg) scale(1.1);
  transform: rotate(360deg) scale(1.1);
}
```

参考链接
- [http://codepub.cn/2016/03/20/Hexo-blog-theme-switching-from-Jacman-to-NexT-Mist/](http://codepub.cn/2016/03/20/Hexo-blog-theme-switching-from-Jacman-to-NexT-Mist/)

# 多说CSS修改

## 隐藏评论框底部分享到QQ空间按钮
```css 
.ds-sync{
  display:none !important;
}
```

## 隐藏底部版权信息

```css 
#ds-thread #ds-reset .ds-powered-by {
  display: none;
}
```

## 参考链接
- [http://www.360doc.com/content/15/0126/21/21724608_444033903.shtml](http://www.360doc.com/content/15/0126/21/21724608_444033903.shtml)
- [http://shenchaofei.cn/duoshuo-comment-box-css-custom/328.html](http://shenchaofei.cn/duoshuo-comment-box-css-custom/328.html)
- [http://dev.duoshuo.com/docs/4ff1cfd0397309552c000017](http://dev.duoshuo.com/docs/4ff1cfd0397309552c000017)


# 搜索功能

博客的站内搜索网上一般说用`Swiftype`搜索，尝试了下，但是提示只有一个月的免费试用，遂放弃改用`Local Search`。

安装hexo-generator-search：
```c
npm install hexo-generator-search --save
```

编辑站点的配置文件，Hexo目录下的_config.yml文件，新增以下内容：
```c
search:
  path: search.xml
  field: post
```

# 文章链接唯一化

也许你会数次更改文章题目或者变更文章发布时间，在默认设置下，文章链接都会改变，不利于搜索引擎收录，也不利于分享。唯一永久链接才是更好的选择。安装此插件后，不要在`hexo s`模式下更改文章文件名，否则文章将成空白。

安装：
```c 
npm install hexo-abbrlink --save
```

在`站点配置文件`中查找代码`permalink`，将其更改为:
```c 
permalink: posts/:abbrlink/  # “posts/” 可自行更换
```

在`站点配置文件`中添加如下代码：
```c 
# abbrlink config
abbrlink:
  alg: crc32  # 算法：crc16(default) and crc32 
  rep: hex    # 进制：dec(default) and hex
```

可选择模式：
- crc16 & hex
- crc16 & dec
- crc32 & hex
- crc32 & dec

# 文章底部增加版权信息

新版的 `Next` 主题应该已经支持文章版权的相关设置，在对应的目录下可以看到带有 `copyright` 相关的代码。

在这里，我们不使用主题本身的版权设置，改用我们自定义的版权信息说明，类似本文下方的表现形式。

> **注：类似本文的版权信息，需要增加相关的代码来实现。这个功能不是我自己写的，代码和想法皆出自于务虚笔记（作者博客名称），当初是请教作者并且得到授权允许发布相关文章来说明如何实现类似的样式。作者个人链接是[http://www.wuxubj.cn/](http://www.wuxubj.cn/)，转载这部分内容请务必注明作者及其个人链接**。
>作者对于 `next`主题的优化可以参考 [`Github链接`](https://github.com/wuxubj/wuxubj.github.io)


在目录 `next/layout/_macro/下`添加 `my-copyright.swig`：
```c 
{% if page.copyright %}
<div class="my_post_copyright">
  <script src="//cdn.bootcss.com/clipboard.js/1.5.10/clipboard.min.js"></script>
  
  <!-- JS库 sweetalert 可修改路径 -->
  <script type="text/javascript" src="http://jslibs.wuxubj.cn/sweetalert_mini/jquery-1.7.1.min.js"></script>
  <script src="http://jslibs.wuxubj.cn/sweetalert_mini/sweetalert.min.js"></script>
  <link rel="stylesheet" type="text/css" href="http://jslibs.wuxubj.cn/sweetalert_mini/sweetalert.mini.css">

  <p><span>本文标题:</span><a href="{{ url_for(page.path) }}">{{ page.title }}</a></p>
  <p><span>文章作者:</span><a href="/" title="访问 {{ theme.author }} 的个人博客">{{ theme.author }}</a></p>
  <p><span>发布时间:</span>{{ page.date.format("YYYY年MM月DD日 - HH:MM") }}</p>
  <p><span>最后更新:</span>{{ page.updated.format("YYYY年MM月DD日 - HH:MM") }}</p>
  <p><span>原始链接:</span><a href="{{ url_for(page.path) }}" title="{{ page.title }}">{{ page.permalink }}</a>
    <span class="copy-path"  title="点击复制文章链接"><i class="fa fa-clipboard" data-clipboard-text="{{ page.permalink }}"  aria-label="复制成功！"></i></span>
  </p>
  <p><span>许可协议:</span><i class="fa fa-creative-commons"></i> <a rel="license" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank" title="Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0)">署名-非商业性使用-禁止演绎 4.0 国际</a> 转载请保留原文链接及作者。</p>  
</div>

<script> 
    var clipboard = new Clipboard('.fa-clipboard');
	clipboard.on('success', $(function(){
	  $(".fa-clipboard").click(function(){
		swal({   
		  title: "",   
		  text: '复制成功',   
		  html: false,
		  timer: 500,   
		  showConfirmButton: false
	    });
	  });
    }));  
</script>
{% endif %}
```


在目录`next/source/css/_common/components/post/`下添加`my-post-copyright.styl`：
```c 
.my_post_copyright {
  width: 85%;
  max-width: 45em;
  margin: 2.8em auto 0;
  padding: 0.5em 1.0em;
  border: 1px solid #d3d3d3;
  font-size: 0.93rem;
  line-height: 1.6em;
  word-break: break-all;
  background: rgba(255,255,255,0.4);
}

.my_post_copyright p{margin:0;}

.my_post_copyright span {
  display: inline-block;
  width: 5.2em;
  color: #b5b5b5;
  font-weight: bold;
}

.my_post_copyright .raw {
  margin-left: 1em;
  width: 5em;
}

.my_post_copyright a {
  color: #808080;
  border-bottom:0;
}

.my_post_copyright a:hover {
  color: #a3d2a3;
  text-decoration: underline;
}

.my_post_copyright:hover .fa-clipboard {
  color: #000;
}

.my_post_copyright .post-url:hover {
  font-weight: normal;
}

.my_post_copyright .copy-path {
  margin-left: 1em;
  width: 1em;
  +mobile(){display:none;}
}

.my_post_copyright .copy-path:hover {
  color: #808080;
  cursor: pointer;
}
```


修改`next/layout/_macro/post.swig`，在代码
```c
<div>
      {% if not is_index %}
        {% include 'wechat-subscriber.swig' %}
      {% endif %}
</div>
```
之前添加增加如下代码：
```c 
<div>
      {% if not is_index %}
        {% include 'my-copyright.swig' %}
      {% endif %}
</div>
```

修改`next/source/css/_common/components/post/post.styl`文件，在最后一行增加代码：
```c 
@import "my-post-copyright"
```

保存重新生成即可。

这样当发布一篇博文是，要在该博文下面增加版权信息的显示，需要在 Markdown 中增加`copyright: true`的设置，类似
```c
---
title: Mac终端翻墙
tags: Linux&Unix
categories: Linux&Unix
comments: true
abbrlink: 745a6d7
date: 2016-11-24 18:26:33
updated: 2016-11-24 18:26:33
copyright: true
---
```