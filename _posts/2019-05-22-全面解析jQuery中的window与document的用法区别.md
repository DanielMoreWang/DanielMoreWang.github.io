---
layout:     post
title:      全面解析jQuery中$(window)与$(document)的用法区别
subtitle:   学习Jquery用法
date:       2019-05-22
author:     zakun
header-img: img/big_banner/post-bg-unix-linux.jpg
catalog: true
tags:
    - jquery
---

# window对象

<a href="http://www.w3school.com.cn/jsref/dom_obj_window.asp" target="_blank">window 对象介绍</a>

## 一、jQuery中的$(window).load()与$(document).ready()的区别

### 1.执行时间

window.onload()即jquery写法中的$(window).load(function(){})必须等到页面内包括图片的所有元素加载完毕后才能执行。
$(document).ready()是DOM结构绘制完毕后就执行，不必等到加载完毕。

### 2.编写个数不同

window.onload不能同时编写多个，如果有多个window.onload方法，只会执行一个(最后一个)
$(document).ready()可以同时编写多个，并且都可以得到执行

### 3.简化写法

window.onload没有简化写法
$(document).ready(function(){})可以简写成$(function(){});

## 二、$(window).height()和$(document).height()的区别

jQuery(window).height()代表了当前可见区域的大小，
jQuery(document).height()则代表了整个文档的高度，可视具体情况使用.

注意：当浏览器窗口大小改变时(如最大化或拉大窗口后) ，
jQuery(window).height() 随之改变，但是
jQuery(document).height()是不变的。

## 三、$(window).scroll()和$(document).scroll()的区别

### 1、scroll()定义和用法：

当用户滚动指定的元素时，会发生 scroll 事件。
scroll 事件适用于所有可滚动的元素和 window 对象（浏览器窗口）。

### 2、两者在使用效果上区别不大，但所有浏览器基本都支持$(window).scroll()，但$(document).scroll()就不一定了。

## 四、$(window).scrollTop()和$(document).scrollTop()的区别

### 1、scrollTop()定义和用法：

scrollTop() 方法返回或设置匹配元素的滚动条的垂直位置（即：滚动条最上方与该元素顶部的距离）。
输入参数比如： $(window).scrollTop(100)，将垂直位置设置为100px;
不输入参数比如： $(window).scrollTop(100)，返回匹配元素的滚动条的垂直位置。

### 2、$(window).scrollTop()和$(document).scrollTop()两者在使用效果上区别不大，但所有浏览器基本都支持前者，但后者就不一定了。

附：一个返回顶部功能，对以上知识的应用
	
	$(function(){
	 "use strict";
	 var backButton=$('.back-to-top ');//css中请事先将按钮隐藏
	 //返回顶部按钮点击事件
	 backButton.on('click',function(){
	 $('html,body').animate({
	 scrollTop:0
	 },800)
	 });
	 //窗口向下滚动一屏后显示‘返回顶部按钮'
	 $(window).on('scroll',function(){
	 if($(window).scrollTop() > $(window).height())
	 backButton.fadeIn();
	 else
	 backButton.fadeOut();
	 })
	});
