title: activity与fragment切换动画
date: 2015-01-29 00:58:34
tags: 动画
categories: Android
---
# Activity切换动画
  activity切换的动画为teen Animation，包含了基本的动画类型,scale、alpha、translate和raotation，当然也可以是animationset。详见上一篇博文<a href="http://coderrobin.com/2015/01/28/android-view%E5%8A%A8%E7%94%BB%E5%9F%BA%E7%A1%80/">android 控件动画基础</a>。
  以下为activity切换动画的具体方式
 ##  通过theme设置切换动画
在 AndroidManifest.xml 文件中，通过 android:theme 属性设置 Activity 的主题。主题中定义了关于 Activity 外观的很多特性。其中就包含 Activity 的切换动画。在主题style中使用 windowAnimationStyle 这个属性，即可指定切换动画的style。
<!--more-->
```
<style name="AnimActivityTheme">
    <item name="android:windowAnimationStyle">@style/FeelyouWindowAnimTheme</item>
</style>
```
定义切换动画 style
```
<style name="FeelyouWindowAnimTheme" parent="@android:style/Animation.Activity">
    <item name="android:activityOpenEnterAnimation">@anim/in_from_left</item>
    <item name="android:activityOpenExitAnimation">@anim/out_from_right</item>
    <item name="android:activityCloseEnterAnimation">@anim/in_from_right</item>
    <item name="android:activityCloseExitAnimation">@anim/out_from_left</item>
</style>
```
注意需要继承自 @android:style/Animation.Activity。具体这4个属性什么意思呢？假设我们有 2 个 Activity，分别是 A1 和 A2：
    当我们从 A1 启动 A2 时，A1 从屏幕上消失，这个动画叫做 android:activityOpenExitAnimation
    当我们从 A1 启动 A2 时，A2 出现在屏幕上，这个动画叫做 android:activityOpenEnterAnimation
    当我们从 A2 退出回到 A1 时，A2 从屏幕上消失，这个叫做 android:activityCloseExitAnimation
    当我们从 A2 退出回到 A1 时，A1 出现在屏幕上，这个叫做 android:activityCloseEnterAnimation
 ##  调用overridePendingTransition(int enterAnim, int exitAnim)方法
　　这个方法在startActivity(Intent) or finish()之后被调用，指定接下来的这个切换动画。第一个参数：enterAnim，是新的Activity的进入动画的resource ID，第二个参数exitAnim，是旧的Activity(当前的Activity)离开动画的resource ID。所以这两个参数的对象是两个Activity。
   

# Fragment切换动画
　　Fragment的切换动画实现分为使用v4包和不使用v4包两种情况，不使用v4包的话，最低API Level需要是11。

## 标准切换动画：
　　可以给Fragment指定标准的切换动画，通过setTransition(int transit)方法。

　　该方法可传入的三个参数是：

　　TRANSIT_NONE,

　　TRANSIT_FRAGMENT_OPEN,

　　TRANSIT_FRAGMENT_CLOSE

　　分别对应无动画、打开形式的动画和关闭形式的动画。

　　标准动画设置好后，在Fragment添加和移除的时候都会有。

## 自定义切换动画
　　自定义切换动画是通过setCustomAnimations()方法，因为Fragment添加时可以指定加入到Back Stack中，所以切换动画有添加、移除、从Back stack中pop出来，还有进入四种情况。

　　注意setCustomAnimations()方法必须在add、remove、replace调用之前被设置，否则不起作用。

 

android.app.Fragment

　　不使用v4包的情况下(min API >=11)所对应的动画类型是Property Animation。

　　即动画资源文件需要放在res\animator\目录下，且根标签是<set>, <objectAnimator>, or <valueAnimator>三者之一。

　　这一点也可以从Fragment中的这个方法看出：onCreateAnimator(int transit, boolean enter, int nextAnim)，返回值是Animator。

　　自定义切换动画时，四个参数的形式setCustomAnimations (int enter, int exit, int popEnter, int popExit)是API Level 13才有的，11只引入了两个动画的形式，即无法指定Back Stack栈操作时的切换动画。

　　代码例子：

```
    private void addFragment() {
        if (null == mFragmentManager) {
            mFragmentManager = getFragmentManager();
        }

        mTextFragmentOne = new MyFragmentOne();
        FragmentTransaction fragmentTransaction = mFragmentManager
                .beginTransaction();

        // 标准动画
        // fragmentTransaction
        // .setTransition(FragmentTransaction.TRANSIT_FRAGMENT_OPEN);
        // fragmentTransaction
        // .setTransition(FragmentTransaction.TRANSIT_FRAGMENT_FADE);

        // fragmentTransaction
        // .setTransition(FragmentTransaction.TRANSIT_FRAGMENT_CLOSE);

        // 自定义动画

        // API LEVEL 11
        fragmentTransaction.setCustomAnimations(
                R.animator.fragment_slide_left_enter,
                R.animator.fragment_slide_right_exit);

        // API LEVEL 13
        // fragmentTransaction.setCustomAnimations(
        // R.animator.fragment_slide_left_enter,
        // R.animator.fragment_slide_left_exit,
        // R.animator.fragment_slide_right_enter,
        // R.animator.fragment_slide_right_exit);

        fragmentTransaction.add(R.id.container, mTextFragmentOne);

        // 加入到BackStack中
        fragmentTransaction.addToBackStack(null);
        fragmentTransaction.commit();

    }
```
android.support.v4.app.Fragment


　　使用v4包，Fragment的使用不再局限于API Level 11之上，低等级的API也可以使用，但是这时候切换动画的类型是View Animation。

　　动画资源放在res\anim\路径下，和Activity的切换动画一样。

　　Fragment中的方法：onCreateAnimation(int transit, boolean enter, int nextAnim)返回值Animation。

　　FragmentTransaction中的setCustomAnimations()方法，两参数类型和四参数类型都可用。

　　所以一般还是用v4包的这个版本，一是兼容性比较好，另外View Animation其实基本可以满足切换动画的需要。
```
    private void addFragment() {
        if (null == mFragmentManager) {
            mFragmentManager = getSupportFragmentManager();
        }

        mTextFragmentOne = new MyFragmentOne();
        FragmentTransaction fragmentTransaction = mFragmentManager
                .beginTransaction();
        fragmentTransaction.setCustomAnimations(
                R.anim.push_left_in,
                R.anim.push_left_out,
                R.anim.push_left_in,
                R.anim.push_left_out);

        fragmentTransaction.add(R.id.container, mTextFragmentOne);

        fragmentTransaction.addToBackStack(null);
        fragmentTransaction.commit();
    }
```
