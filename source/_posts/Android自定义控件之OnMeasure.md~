title: Android自定义控件之OnMeasure
date: 2015-01-25 23:18:57
categories: Android
tags: 自定义控件
---
在自定义控件中,要重写OnMeasure,onDraw方法,自定义布局还要重写onLayout方法以确定子控件位置,本篇博文讲述自定义控件应如何重写OnMeasure方法.

onMeasure(int widthMeasureSpec, int heightMeasureSpec)方法两个形参widthMeasureSpec和heightMeasureSpec表示布局期望的子控件MeasureSpec（规格）
# MeasureSpec介绍
以下为MeasureSpec源码

```
    public static class MeasureSpec { 
      
        private static final int MODE_SHIFT = 30;  
        private static final int MODE_MASK  = 0x3 << MODE_SHIFT;  
        public static final int UNSPECIFIED = 0 << MODE_SHIFT;  
        public static final int EXACTLY     = 1 << MODE_SHIFT;  
        public static final int AT_MOST     = 2 << MODE_SHIFT;  
      
        public static int makeMeasureSpec(int size, int mode) {  
            return size + mode;  
        }  
      
        public static int getMode(int measureSpec) {  
            return (measureSpec & MODE_MASK);  
        }  
      
        public static int getSize(int measureSpec) {  
            return (measureSpec & ~MODE_MASK);  
        }  
    } 
```
分析源码可知measureSpec为int值，总共4个字节,高2位表示mode,低30位表示size，可通过静态方法getMod和getSize方法,获得mode和size

MeasureSpec有三种模式分别是UNSPECIFIED, EXACTLY和AT_MOST。
<!-- more -->
### EXACTLY
表示父视图希望子视图的大小应该是由specSize的值来决定的，系统默认会按照这个规则来设置子视图的大小，开发人员当然也可以按照自己的意愿设置成任意的大小。

### AT_MOST
表示子视图最多只能是specSize中指定的大小，开发人员应该尽可能小得去设置这个视图，并且保证不会超过specSize。系统默认会按照这个规则来设置子视图的大小，开发人员当然也可以按照自己的意愿设置成任意的大小。

### UNSPECIFIED
表示开发人员可以将视图按照自己的意愿设置成任意的大小，没有任何限制。这种情况比较少见，不太会用到。

## measureSpec参数计算过程
接下来分析子控件onMeasure方法的MeasureSpec父控件是怎么就算出来的,以下为viewRroup中测量部分的源码
``` 
     * Ask all of the children of this view to measure themselves, taking into 
     * account both the MeasureSpec requirements for this view and its padding. 
     * We skip children that are in the GONE state The heavy lifting is done in 
     * getChildMeasureSpec. 
     * 
     * @param widthMeasureSpec The width requirements for this view 
     * @param heightMeasureSpec The height requirements for this view 
     */ 
    protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) { 
        final int size = mChildrenCount;  
        final View[] children = mChildren;  
        for (int i = 0; i < size; ++i) {  
            final View child = children[i];  
            if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {  
                measureChild(child, widthMeasureSpec, heightMeasureSpec);  
            }  
        }  
    } 
  
    /** 
     * Ask one of the children of this view to measure itself, taking into 
     * account both the MeasureSpec requirements for this view and its padding. 
     * The heavy lifting is done in getChildMeasureSpec. 
     * 
     * @param child The child to measure 
     * @param parentWidthMeasureSpec The width requirements for this view 
     * @param parentHeightMeasureSpec The height requirements for this view 
     */ 
    protected void measureChild(View child, int parentWidthMeasureSpec, 
            int parentHeightMeasureSpec) {  
        final LayoutParams lp = child.getLayoutParams();  
  
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,  
                mPaddingLeft + mPaddingRight, lp.width);  
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,  
                mPaddingTop + mPaddingBottom, lp.height);  
  
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);  
    }


    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);  
        int specSize = MeasureSpec.getSize(spec);  
  
        int size = Math.max(0, specSize - padding);  
  
        int resultSize = 0;  
        int resultMode = 0;  
  
        switch (specMode) {  
        // Parent has imposed an exact size on us  
        case MeasureSpec.EXACTLY:  
            if (childDimension >= 0) {  
                resultSize = childDimension;  
                resultMode = MeasureSpec.EXACTLY;  
            } else if (childDimension == LayoutParams.MATCH_PARENT) {  
                // Child wants to be our size. So be it.  
                resultSize = size;  
                resultMode = MeasureSpec.EXACTLY;  
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {  
                // Child wants to determine its own size. It can't be  
                // bigger than us.  
                resultSize = size;  
                resultMode = MeasureSpec.AT_MOST;  
            }  
            break;  
  
        // Parent has imposed a maximum size on us  
        case MeasureSpec.AT_MOST:  
            if (childDimension >= 0) {  
                // Child wants a specific size... so be it  
                resultSize = childDimension;  
                resultMode = MeasureSpec.EXACTLY;  
            } else if (childDimension == LayoutParams.MATCH_PARENT) {  
                // Child wants to be our size, but our size is not fixed.  
                // Constrain child to not be bigger than us.  
                resultSize = size;  
                resultMode = MeasureSpec.AT_MOST;  
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {  
                // Child wants to determine its own size. It can't be  
                // bigger than us.  
                resultSize = size;  
                resultMode = MeasureSpec.AT_MOST;  
            }  
            break;  
  
        // Parent asked to see how big we want to be  
        case MeasureSpec.UNSPECIFIED:  
            if (childDimension >= 0) {  
                // Child wants a specific size... let him have it  
                resultSize = childDimension;  
                resultMode = MeasureSpec.EXACTLY;  
            } else if (childDimension == LayoutParams.MATCH_PARENT) {  
                // Child wants to be our size... find out how big it should  
                // be  
                resultSize = 0;  
                resultMode = MeasureSpec.UNSPECIFIED;  
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {  
                // Child wants to determine its own size.... find out how  
                // big it should be  
                resultSize = 0;  
                resultMode = MeasureSpec.UNSPECIFIED;  
            }  
            break;  
        }  
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);  
    } 
```

# viewGroup调用view的measure方法
view measure方法
```
    public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        // 省略部分代码……  
      
        /* 
         * 判断当前mPrivateFlags是否带有PFLAG_FORCE_LAYOUT强制布局标记 
         * 判断当前widthMeasureSpec和heightMeasureSpec是否发生了改变 
         */  
        if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ||  
                widthMeasureSpec != mOldWidthMeasureSpec ||  
                heightMeasureSpec != mOldHeightMeasureSpec) {  
      
            // 如果发生了改变表示需要重新进行测量此时清除掉mPrivateFlags中已测量的标识位PFLAG_MEASURED_DIMENSION_SET  
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;  
      
            resolveRtlPropertiesIfNeeded();  
      
            int cacheIndex = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ? -1 :  
                    mMeasureCache.indexOfKey(key);  
            if (cacheIndex < 0 || sIgnoreMeasureCache) {  
                // 测量View的尺寸  
                onMeasure(widthMeasureSpec, heightMeasureSpec);  
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;  
            } else {  
                long value = mMeasureCache.valueAt(cacheIndex);  
      
                setMeasuredDimension((int) (value >> 32), (int) value);  
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;  
            }  
      
            /* 
             * 如果mPrivateFlags里没有表示已测量的标识位PFLAG_MEASURED_DIMENSION_SET则会抛出异常 
             */  
            if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {  
                throw new IllegalStateException("onMeasure() did not set the"  
                        + " measured dimension by calling"  
                        + " setMeasuredDimension()");  
            }  
      
            // 如果已测量View那么就可以往mPrivateFlags添加标识位PFLAG_LAYOUT_REQUIRED表示可以进行布局了  
            mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;  
        }  
      
        // 最后存储测量完成的测量规格  
        mOldWidthMeasureSpec = widthMeasureSpec;  
        mOldHeightMeasureSpec = heightMeasureSpec;  
      
        mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |  
                (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension  
    } 

默认实现：

    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) { 
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),  
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));  
    } 

    protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
        // 省去部分代码……  
      
        // 设置测量后的宽高  
        mMeasuredWidth = measuredWidth;  
        mMeasuredHeight = measuredHeight;  
      
        // 重新将已测量标识位存入mPrivateFlags标识测量的完成  
        mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;  
    } 


    protected int getSuggestedMinimumWidth() { 
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());  
    }


    public static int getDefaultSize(int size, int measureSpec) {
        // 将我们获得的最小值赋给result  
        int result = size;  
      
        // 从measureSpec中解算出测量规格的模式和尺寸  
        int specMode = MeasureSpec.getMode(measureSpec);  
        int specSize = MeasureSpec.getSize(measureSpec);  
      
        /* 
         * 根据测量规格模式确定最终的测量尺寸 
         */  
        switch (specMode) {  
        case MeasureSpec.UNSPECIFIED:  
            result = size;  
            break;  
        case MeasureSpec.AT_MOST:  
        case MeasureSpec.EXACTLY:  
            result = specSize;  
            break;  
        }  
        return result;  
    } 
```
# 自定义控件重写onMeasure例子
由以上源码分析可得出,重写onMeasure即根据widthMeasureSpec和heightMeasureSpec确定自己的大小,以下为例子
``` 
 @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int resultWidth = 0;  
        int modeWidth = MeasureSpec.getMode(widthMeasureSpec);    
        int sizeWidth = MeasureSpec.getSize(widthMeasureSpec);  
        if (modeWidth == MeasureSpec.EXACTLY) { 
            resultWidth = sizeWidth;  
        }  
        else {  
            resultWidth =***;//确定自己部分大小  
            if (modeWidth == MeasureSpec.AT_MOST) {  
                resultWidth = Math.min(resultWidth, sizeWidth);  
            }  
        }  
      
        int resultHeight = 0;  
        int modeHeight = MeasureSpec.getMode(heightMeasureSpec);  
        int sizeHeight = MeasureSpec.getSize(heightMeasureSpec);  
      
        if (modeHeight == MeasureSpec.EXACTLY) {  
            resultHeight = sizeHeight;  
        } else {  
            resultHeight = mBitmap.getHeight();  
            if (modeHeight == MeasureSpec.AT_MOST) {  
                resultHeight = Math.min(resultHeight, sizeHeight);  
            }  
        }  
      
        // 设置测量尺寸,一定要设置,否则无用且报错  
        setMeasuredDimension(resultWidth, resultHeight);  
    }
```
