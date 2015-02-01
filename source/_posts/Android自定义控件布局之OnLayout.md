title: Android自定义控件布局之OnLayout
date: 2015-01-26 22:56:32
categories: Android
tags: 自定义控件布局
---
在view的绘制过程中,会依次调用view的measure方法,layout方法,draw方法
其中layout方法即对子控件测量完成后确定每个子控件相对父控件的位置
注意到该方法为final方法,不可重写,其中调用到了onLayout,可在此方法中设置每个子视图的坐标
```
# layout方法
public final void layout(int l, int t, int r, int b) { 
    boolean changed = setFrame(l, t, r, b); //保存相对于父视图的坐标轴 
    if (changed || (mPrivateFlags & LAYOUT_REQUIRED) == LAYOUT_REQUIRED) { 
        if (ViewDebug.TRACE_HIERARCHY) {  
            ViewDebug.trace(this, ViewDebug.HierarchyTraceType.ON_LAYOUT);  
        }  
  
        onLayout(changed, l, t, r, b);//回调onLayout函数 ，设置每个子视图的布局  
        mPrivateFlags &= ~LAYOUT_REQUIRED;  
    } 
    mPrivateFlags &= ~FORCE_LAYOUT; 
} 
```
<!-- more -->
layout方法中首先调用了setFrame方法,该方法作用为保存父控件传过来的坐标数据,以确定本控件的位置,其中在保存前和保存后两次invalidate进行绘制,且当位置有改变时调用onSizeChanged方法
```
   protected boolean setFrame(int left, int top, int right, int bottom) {
        boolean changed = false;  
  
        ......  
  
        if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {  
            changed = true;  
  
            // Remember our drawn bit  
            int drawn = mPrivateFlags & DRAWN;  
  
            // 绘制一次旧位置的视图
            invalidate();  
  
  
            int oldWidth = mRight - mLeft;  
            int oldHeight = mBottom - mTop;  
  
            mLeft = left;  
            mTop = top;  
            mRight = right;  
            mBottom = bottom;  
  
            mPrivateFlags |= HAS_BOUNDS;  
  
            int newWidth = right - left;  
            int newHeight = bottom - top;  
  
            if (newWidth != oldWidth || newHeight != oldHeight) {  
                onSizeChanged(newWidth, newHeight, oldWidth, oldHeight);  
            }  
  
            if ((mViewFlags & VISIBILITY_MASK) == VISIBLE) {  
                // If we are visible, force the DRAWN bit to on so that  
                // this invalidate will go through (at least to our parent).  
                // This is because someone may have invalidated this view  
                // before this call to setFrame came in, therby clearing  
                // the DRAWN bit.  
                mPrivateFlags |= DRAWN;  
                invalidate();  
            }  
  
            // Reset drawn bit to original value (invalidate turns it off)  
            mPrivateFlags |= drawn;  
  
            mBackgroundSizeChanged = true;  
        }  
        return changed;  
    } 
  
    ...... 
} 
```
# onLayout方法
layout()调用了onLayout方法,该方法确定子控件的位置
    @Override 
    protected abstract void onLayout(boolean changed, int l, int t, int r, int b); 
该方法被定义为抽象方法，在继承ViewGroup时必须要重写该方法。
以下以frameLayout的onLayout和LinearLayout的onLayout方法举例自定义的onLayout方法要怎么写
## FrameLayout中的onLayout
```
        protected void onLayout(boolean changed, int left, int top, int right, int bottom) {  
            final int count = getChildCount();  
      
            final int parentLeft = mPaddingLeft + mForegroundPaddingLeft;  
            final int parentRight = right - left - mPaddingRight - mForegroundPaddingRight;  
      
            final int parentTop = mPaddingTop + mForegroundPaddingTop;  
            final int parentBottom = bottom - top - mPaddingBottom - mForegroundPaddingBottom;  
      
            mForegroundBoundsChanged = true;  
      
            for (int i = 0; i < count; i++) {  
                final View child = getChildAt(i);  
                if (child.getVisibility() != GONE) {  
                    final LayoutParams lp = (LayoutParams) child.getLayoutParams();  
      
                    final int width = child.getMeasuredWidth();  
                    final int height = child.getMeasuredHeight();  
      
                    int childLeft = parentLeft;  
                    int childTop = parentTop;  
      
                    final int gravity = lp.gravity;  
      
                    if (gravity != -1) {  
                        final int horizontalGravity = gravity & Gravity.HORIZONTAL_GRAVITY_MASK;  
                        final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;  
                     //根据gravity计算childLeft和childTop
                        switch (horizontalGravity) {  
                            case Gravity.LEFT:  
                                childLeft = parentLeft + lp.leftMargin;  
                                break;  
                            case Gravity.CENTER_HORIZONTAL:  
                                childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +  
                                        lp.leftMargin - lp.rightMargin;  
                                break;  
                            case Gravity.RIGHT:  
                                childLeft = parentRight - width - lp.rightMargin;  
                                break;  
                            default:  
                                childLeft = parentLeft + lp.leftMargin;  
                        }  
                        switch (verticalGravity) {  
                            case Gravity.TOP:  
                                childTop = parentTop + lp.topMargin;  
                                break;  
                            case Gravity.CENTER_VERTICAL:  
                                childTop = parentTop + (parentBottom - parentTop - height) / 2 +  
                                        lp.topMargin - lp.bottomMargin;  
                                break;  
                            case Gravity.BOTTOM:  
                                childTop = parentBottom - height - lp.bottomMargin;  
                                break;  
                            default:  
                                childTop = parentTop + lp.topMargin;  
                        }  
                    }  
      //设置child的layout
                    child.layout(childLeft, childTop, childLeft + width, childTop + height);  
                }  
            }  
        }  
      
        ......  
    } 
```
## LinearLayout中的onLayout 
```
    @Override 
       protected void onLayout(boolean changed, int l, int t, int r, int b) { 
           if (mOrientation == VERTICAL) {  
               layoutVertical();  
           } else {  
               layoutHorizontal();  
           }  
       } 

    void layoutVertical() { 
            final int paddingLeft = mPaddingLeft;  
            int childTop = mPaddingTop;  
            int childLeft;  
            // Where right end of child should go  
            final int width = mRight - mLeft;  
            int childRight = width - mPaddingRight;  
            // Space available for child  
            int childSpace = width - paddingLeft - mPaddingRight;  
            final int count = getVirtualChildCount();  
            final int majorGravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;  
            final int minorGravity = mGravity & Gravity.HORIZONTAL_GRAVITY_MASK;  
            if (majorGravity != Gravity.TOP) {  
               switch (majorGravity) {  
                   case Gravity.BOTTOM:  
                       // mTotalLength contains the padding already, we add the top  
                       // padding to compensate  
                       childTop = mBottom - mTop + mPaddingTop - mTotalLength;  
                       break;  
                   case Gravity.CENTER_VERTICAL:  
                       childTop += ((mBottom - mTop)  - mTotalLength) / 2;  
                       break;  
               }  
            }  
            for (int i = 0; i < count; i++) {  
                final View child = getVirtualChildAt(i);  
                if (child == null) {  
                    childTop += measureNullChild(i);  
                } else if (child.getVisibility() != GONE) {  
                    final int childWidth = child.getMeasuredWidth();  
                    final int childHeight = child.getMeasuredHeight();  
                    final LinearLayout.LayoutParams lp =  
                            (LinearLayout.LayoutParams) child.getLayoutParams();  
                    int gravity = lp.gravity;  
                    if (gravity < 0) {  
                        gravity = minorGravity;  
                    }  
                    switch (gravity & Gravity.HORIZONTAL_GRAVITY_MASK) {  
                        case Gravity.LEFT:  
                            childLeft = paddingLeft + lp.leftMargin;  
                            break;  
                        case Gravity.CENTER_HORIZONTAL:  
                            childLeft = paddingLeft + ((childSpace - childWidth) / 2)  
                                    + lp.leftMargin - lp.rightMargin;  
                            break;  
                        case Gravity.RIGHT:  
                            childLeft = childRight - childWidth - lp.rightMargin;  
                            break;  
                        default:  
                            childLeft = paddingLeft;  
                            break;  
                    }  
                    childTop += lp.topMargin;  
                    setChildFrame(child, childLeft, childTop + getLocationOffset(child),  
                            childWidth, childHeight);  
                    childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);  
                    i += getChildrenSkipCount(child, i);  
                }  
            }  
        }   
```

#总结
自定义控件时，在onLayout方法中，需根据父控件的padding、子控件的margin以及父控件的gravity以及子控件的lp.gravity(即layout gravity）、子控件的getMeasuredWidth和getMeasuredHeight确定各子控件的位置。
