---
title: "为hugo增加gitalk功能"
author: "kzl"
tags: ["go", "hugo"]
date: 2019-03-20T17:35:02+08:00
layout: "kzl"
typora-root-url: G:/kzltime/static
---



在hugo博客中，引入gitalk评论功能。



1. 在`config.toml`中增加配置如下：

   ```json
    [params.gitalk]
       enable = true
       appId = '你的'
       appKey = '你的'
       repo = "kangzhanlei.github.io"
       owner = 'kangzhanlei'
       admin = 'kangzhanlei'
   ```

2.在你的主题下（themes/hugo-nuo/layouts/partials)下，增加`comments.html`。其中内容为：

```html
 
 {{- if .Site.Params.gitalk.enable -}}
<!-- Link Gitalk 的支持文件  -->
<link rel="stylesheet" href="https://unpkg.com/gitalk/dist/gitalk.css">
<script src="https://unpkg.com/gitalk@latest/dist/gitalk.min.js"></script>
 
<div id="gitalk-container"></div>
    <script type="text/javascript">
    var gitalk = new Gitalk({
 
    // gitalk的主要参数
        clientID: '{{ .Site.Params.gitalk.appId }}',
        clientSecret: '{{ .Site.Params.gitalk.appKey }}',
        repo: '{{ .Site.Params.gitalk.repo }}',
        owner: '{{ .Site.Params.gitalk.owner }}',
        admin: ['{{ .Site.Params.gitalk.admin }}'],
        id: location.pathname,
        distractionFreeMode: false
    });
    gitalk.render('gitalk-container');
</script>

{{- end }}

```

3.在对应的type中（layouts/post/single.html) 中，增加对应的div引入，例如

   ```html
    <div class="post-comment">
         <!-- 加入评论功能 -->
         {{ partial "comments.html" . }}
     </div>
   ```

4.github令牌的申请地址为：[点我点我](https://github.com/settings/applications/new)

5.enjoy it