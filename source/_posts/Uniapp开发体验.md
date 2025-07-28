---
title: uni-app开发体验
date: 2025-07-28 17:00:00
tags: 
  - uniapp
  - vue
---

# 背景
在刷小红书看前端岗位的过程中，我刷到一个叫做宠行宝的公司在做一款app用来记录宠物信息的，于是问了下他们是不是需要人写前端，于是就加了团队。

加入团队之后，就拉取了uniapp的代码，第一次开发uniapp，虽然在2024年vueconf也听到过前辈代表DCloud公司分享uniapp x的优化和实现，但是一直没有机会尝试写这个被大家称之为”shit“的东西。

# 接触的过程...
查看了[官方文档](https://uniapp.dcloud.net.cn/tutorial/)，发现要使用HBuilder去开发，于是问了公司开发uniapp的同事是否能用其他ide开发，他给出的回答是可以的。后面找了好久终于看到了”如果想用其他ide开发uni-app，只能使用cli模式“。

难过了好一会，因为宠行宝这个app就是用可视化界面去创建的，硬着头皮下载了HBuilderX，还得下载安卓模拟器，[AndroidStudio模拟器下载链接](http://r2---sn-ni57rn7r.gvt1.com/edgedl/android/studio/install/2022.2.1.20/android-studio-2022.2.1.20-windows.exe?cms_redirect=yes&mh=JT&mip=163.123.245.29&mm=28&mn=sn-ni57rn7r&ms=nvh&mt=1688022819&mv=u&mvi=2&pl=24&shardbypass=sd&smhost=r1---sn-ni57rn7d.gvt1.com)，注意在官网下载的时候下载链接会自动拼上s，要用http才能下载成功。

在android studio中创建好模拟器之后，运行。然后再回到HBuilder X中运行项目，运行-运行到手机或模拟器-运行到Android App基座，然后就可以看到这样：
![运行到Android App基座](/images/android-device.png)

运行之后呢，就可以在安卓模拟器当中看到这个应用，并且调试啦。
