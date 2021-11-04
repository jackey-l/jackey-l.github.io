---
title: 使用github图床+CDN优化github资源加载慢
comments: true
index_img: https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20211104104829657.png
banner_img: https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/background/vilige.jpg
date: 2021-11-04 10:27:11
updated: 2021-11-04 10:27:11
tags: [CDN,图床,github]
categories: [随笔]
---

### 问题

博客搬迁至GitHub使用了一段时间了，图片加载速度很慢，体验很差，今天心血来潮，搞搞CDN优化。当然这只能解决资源的加载问题，如果github抽风，那么还是没办法的。

### 思路

使用github仓库来作为图床，使用jsdelivr CDN来进行加速，解决资源加载慢的问题。

> jsdelivr是一个国外的免费CDN加速站点，官方的建议是用来做JS等文件的加速，试了一下速度还可以。
>
> 2020年8月月官方出了声明，官方不建议使用git+jsdelivr图床的组合。 
>
>  https://www.jsdelivr.com/terms/acceptable-use-policy-jsdelivr-net
>
> 同学们可以自行替换为其他的CDN。这里我头铁，暂时用这个。
>
> 使用起来也非常简单，就在原来的资源URL上稍作改动即可。
>
> 示例：
>
> https://cdn.jsdelivr.net/gh/github用户名/github仓库名@分支名/pic1.png



### 创建github仓库

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20211104093923995.png)

### 获取token

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20211104095941315.png)

#### note填写一下

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20211104100032924.png)

#### 点击生成按钮

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20211104100152930.png)



#### 复制保存token

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20211104100841763.png)

### 设置picgo

1. 填写git用户名与仓库名
2. 将上面的token复制至picgo配置中
3. 分支名填写创建仓库的名称。
4. img/为仓库自路径，如图片存放在根路径下可以不填。
5. 自定义域名填写CDN加速域名

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20211104095646756.png)

笔者用的是typora编辑器，附上typro配置

![](https://cdn.jsdelivr.net/gh/jackey-l/blog_static@master/img/image-20211104095820291.png)



配置完成。

