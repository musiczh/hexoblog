---
title: 	Android那些坑一		#标题
date: 2020/8/1 00:00:00 						#建立日期
sticky:  #置顶参数
tags:	#标签
 - 
					
categories:	#分类
 - 

updated: 					#更新日期
author: 一只修仙的猿 #作者
toc: true	#是否显示toc
mathjax:  #数学公式

keywords:				#关键词
description:				#文章描述
top_img:					#文章顶部照片
comments: true				#是否显示评论模块
cover:						#文章缩略图
toc_number: true			#是否显示toc_number
auto_open: true				#是否自动打开toc
copyright: true					#显示文章版权模块
copyright_author: 一只修仙的猿		#文章版权作者
copyright_author_href: 			#文章版权作者链接
copyright_url:						#文章版权文章链接
copyright_info:						#文章版权声明文字

katex:
aplayer:
highlight_shrink: true       #代码框是否打开
---



#### 1.finish之后Activity不会马上调用onDestroy()

原因：使用handler来实现回调，会有延迟。如果前面有重量级操作会延迟更久。

本质原因：	延迟原因，B onPause之后，AMS会将B保存在需要finish的列表里，A onResume之后 注册一个IdleHandler（空闲处理器） 等待主线程轮训到，才会去调用B onStop onDestroy ，因为IdleHandler的优先级低，只有主线程消息执行完才会去执行，所以主线程消息过多也会使其延迟。2.延迟10s原因，在B onPause之后，也会同步注册一个10s的延时消息，那么就算App在A onResume之后没有主动注册IdleHandler销毁Activity，这个延时的消息到时间后，也会执行销毁Activity的逻辑。

导致问题：资源不能及时释放，下一个界面拿不到资源

解决方案：下一个界面肯定要用到的可以在onPause中释放，但可能会造成卡顿。资源释放建议在onStop中调用。onStop表示界面不可见





> 全文到此，原创不易，觉得有帮助可以点赞收藏评论转发。
> 笔者才疏学浅，有任何想法欢迎评论区交流指正。
> 如需转载请评论区或私信交流。
>
> 另外欢迎光临笔者的个人博客：[传送门](https://qwerhuan.gitee.io)

