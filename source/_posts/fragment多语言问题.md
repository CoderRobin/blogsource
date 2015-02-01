title: fragment多语言问题
date: 2015-01-19 23:36:22
categories: Android
tags: fragment
---
在切换语言时,activity会被系统回收后重新创建,

此时原先依附于该activity的fragment也会被系统destroy掉,

但系统会自动创建新的fragment的实例attach到新的activity中,

若此时用户在activity中手动创建fragment,则会导致程序fragment管理混乱.

因此,建议用如下方法创建fragment
```
public class FragmentFactory {

  public static Fragment getFragmentByTag(Context pContext,String pTag){
       FragmentManager fm = pContext.getSupportFragmentManager();
	//查找是否已存在,已存在则不需要重发创建,切换语言时系统会自动重新创建并attch,无需手动创建
       Fragment fragment = fm.findFragmentByTag(pTag);
       if (fragment != null){
           return fragment;
        }
	else {
	  if(MyFragment.TAG.equals(pTAG)){
		return new MyFragment(); 
	  }
	}
  }
}
```
