title: Android Spinner设置下拉框高度
date: 2015-03-05 22:32:25
categories: Android
tags: 反射
---
最近有用到Spinner控件，然后查了下api再google了一圈，竟然发现没有设置下拉框高度的接口。于是大概看了下spinner源码。发现其中有个私有成员变量SpinnerPopup mPopup便是下拉框控件，当spinner mode是 MODE_DROPDOWN,实际上是一个DropdownPopu，该类又继承了ListPopupWindow，而listPopupwindow有个getListview方法 得到实际展示的listview，于是便可以利用反射实现了设置spinner下拉框的高度和下拉框的listview的各种属性，代码如下：
```
	public void setDropDownHeight(int pHeight){
		try {
			Field field=Spinner.class.getDeclaredField("mPopup");
			field.setAccessible(true);
			ListPopupWindow popUp=(ListPopupWindow)field.get(mSpinner);
			popUp.setHeight(pHeight);
		} catch (NoSuchFieldException e) {
			e.printStackTrace();
		} catch (SecurityException e) {
			e.printStackTrace();
		}
```

