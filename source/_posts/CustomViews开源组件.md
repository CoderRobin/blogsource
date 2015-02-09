title: CustomViews开源项目
date: 2015-02-01 00:36:21
categories:  Android
tags:  CustomViews
---
<a href="https://github.com/CoderRobin/CommonViews">CustomViews </a>（code by CoderRobin）是20150201开始创建的开源项目，目标是开源各种实用的android布局与控件。
以下为该开源项目现有的布局与控件，本博文将随着项目的更新实时更新
#NestRadioGroup布局
该布局继承了LinearLayout,功能为该布局下任意CompoundButton会保持互斥状态（如radiobutton、checkbox),对外提供了与android自带 的radioGroup相同的接口
用户在layout xml文件中如使用RadioGroup一样使用即可,现已包含如下接口
setChecked(int checkedId);
setOnCheckedChangeListener
getCheckedId()

#RippleView动画
继承自view，重写了onMeasure和onDraw方法,使用属性动画和自定义的TypeEvaluator实现
对外暂时提供了以下接口
startRippleAnimation
stopRippleAnimation
效果就是如下图
![rippleview ](/image/rippleview.gif  "rippleview")
<!--more-->




