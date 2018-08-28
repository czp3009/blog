---
layout: post
title: Gitment的redirect_uri_mismatch问题
date: 2018-08-28 19:25:15
category: 前端
tag: gitment, github
---

很多人选择使用 Gitment 作为评论系统, 由于 Gitment 只需要前端引入, 因此使用非常简单.

如果你需要的是在 Hexo 使用 Gitment 的教程, 可以看这篇文章 http://www.codeblocq.com/2018/05/Setup-gitment-on-your-Hexo-blog/

今天要讲的是, 配置好 Gitment 后, 点击前端上的 Login 按钮, 但是被跳转回首页的问题.

然后这个跳转到的 URL 是这样的

    https://www.hiczp.com/?
    error=redirect_uri_mismatch&
    error_description=The+redirect_uri+MUST+match+the+registered+callback+URL+for+this+application.&
    error_uri=https%3A%2F%2Fwww.hiczp.com%2Fv3%2Foauth%2F%23redirect-uri-mismatch

为了方便观看, 进行了折行处理.

# Callback URL 设置错误
我们可以看到其中有一个关键词 'redirect_uri_mismatch', 之后去 Google 搜索, 会有一大群人跟你说, 是你的 OAuth Apps 设置里的 Callback URL 设置错了, 这个设置的值应该与你的博客地址一模一样. 比如博客地址是 https://www.hiczp.com, 那么这个值也必须是这个.

把这个值写错的人大多数把博客托管在 github.io, 但是自己又用其他域名做了 CNAME 的.

Github 需要这个 Callback URL 的作用是确认登陆前那个页面的域与设定中的域一致(包括 http, https 以及端口), 也就是 Login 按钮所在页面的域必须与设定一致. 这是为了防止 OAuth Apps 的 Client Secret 泄露后, 别人可以随意使用你的密钥. 这也是为什么 Gitment 为了纯前端实现, 而把 Client Secret 放在前端代码中却没有安全性问题的原因. 因为 Callback URL 绑定了 Client Secret 与域.

所以, 用户是从哪个域登陆的, 那么这个 URL 就得填哪个.

比如站点实际在 czp3009.github.io, 但是我用 www.hiczp.com(https) 做了 CNAME, 并且我的用户都是用后者访问的, 那么 Callback URL 就得填 https://www.hiczp.com.

# 文章名包含空格
然而事情并不是这么简单, 很多遇到这个问题的人, 并非是由于这个低级错误.

如果确定了 Callback URL 的设置确实是对的, 那么剩下的可能性只有一种: 文章名含有空格.

这个 Login 按钮会让用户跳转到 Github 的 OAuth2 登陆页面, 而为了登陆之后跳转回之前的文章的页面, Github 必须把跳转前的地址记录下来, 但是非常不幸的是, Github 的这一功能不支持空格.

而很多博客会在 URL 中使用文章名, Hexo 也是如此. 所以用户跳转过去的时候, 如果文章名里面是含有空格的, 那么 URL 里就有空格(并非是不会自动转义的问题, 是转义了之后也不支持). 之后 Github 就会认为这个 URL 是不对的, 以 redirect_uri_mismatch 的理由跳回 OAuth Apps 的 Homepage URL 的值的地址并加上后面这一长串 Query 参数.

所以解决方案也非常简单, 那就是不要在文章名使用空格.

# 文章名含有全角符号
还有一些人会遇到更加奇怪的情况, 登陆是正常的, 登陆完了之后跳转回来, 结果是一个 404 页面(在自己的站点上), 这种情况是因为文章名里面有全角符号, Github 跳转回来的时候会自动变成半角符号, 于是导致 404.
