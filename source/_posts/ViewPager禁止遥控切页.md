title: ViewPager禁止遥控切页
date: 2015-01-17 23:50:01
categories: Android
tags: viewpager
---
在做app对遥控（实体按键）的支持时，发现在viewpager子布局中按遥控左右键会翻页。
查看viewpager源码，可以看到如下代码段
<!-- more -->
```
 @Override
    public boolean dispatchKeyEvent(KeyEvent event) {
        // Let the focused view and/or our descendants get the key first
        return super.dispatchKeyEvent(event) || executeKeyEvent(event);
    }

    /**
     * You can call this function yourself to have the scroll view perform
     * scrolling from a key event, just as if the event had been dispatched to
     * it by the view hierarchy.
     *
     * @param event The key event to execute.
     * @return Return true if the event was handled, else false.
     */
    public boolean executeKeyEvent(KeyEvent event) {
        boolean handled = false;
        if (event.getAction() == KeyEvent.ACTION_DOWN) {
            switch (event.getKeyCode()) {
                case KeyEvent.KEYCODE_DPAD_LEFT:
                    handled = arrowScroll(FOCUS_LEFT);
                    break;
                case KeyEvent.KEYCODE_DPAD_RIGHT:
                    handled = arrowScroll(FOCUS_RIGHT);
                    break;
                case KeyEvent.KEYCODE_TAB:
                    if (Build.VERSION.SDK_INT >= 11) {
                        // The focus finder had a bug handling FOCUS_FORWARD and FOCUS_BACKWARD
                        // before Android 3.0. Ignore the tab key on those devices.
                        if (KeyEventCompat.hasNoModifiers(event)) {
                            handled = arrowScroll(FOCUS_FORWARD);
                        } else if (KeyEventCompat.hasModifiers(event, KeyEvent.META_SHIFT_ON)) {
                            handled = arrowScroll(FOCUS_BACKWARD);
                        }
                    }
                    break;
            }
        }
        return handled;
    }

    public boolean arrowScroll(int direction) {
        View currentFocused = findFocus();
        if (currentFocused == this) {
            currentFocused = null;
        } else if (currentFocused != null) {
            boolean isChild = false;
            for (ViewParent parent = currentFocused.getParent(); parent instanceof ViewGroup;
                    parent = parent.getParent()) {
                if (parent == this) {
                    isChild = true;
                    break;
                }
            }
            if (!isChild) {
                // This would cause the focus search down below to fail in fun ways.
                final StringBuilder sb = new StringBuilder();
                sb.append(currentFocused.getClass().getSimpleName());
                for (ViewParent parent = currentFocused.getParent(); parent instanceof ViewGroup;
                        parent = parent.getParent()) {
                    sb.append(" => ").append(parent.getClass().getSimpleName());
                }
                Log.e(TAG, "arrowScroll tried to find focus based on non-child " +
                        "current focused view " + sb.toString());
                currentFocused = null;
            }
        }

        boolean handled = false;

        View nextFocused = FocusFinder.getInstance().findNextFocus(this, currentFocused,
                direction);
        if (nextFocused != null && nextFocused != currentFocused) {
            if (direction == View.FOCUS_LEFT) {
                // If there is nothing to the left, or this is causing us to
                // jump to the right, then what we really want to do is page left.
                final int nextLeft = getChildRectInPagerCoordinates(mTempRect, nextFocused).left;
                final int currLeft = getChildRectInPagerCoordinates(mTempRect, currentFocused).left;
                if (currentFocused != null && nextLeft >= currLeft) {
                    handled = pageLeft();
                } else {
                    handled = nextFocused.requestFocus();
                }
            } else if (direction == View.FOCUS_RIGHT) {
                // If there is nothing to the right, or this is causing us to
                // jump to the left, then what we really want to do is page right.
                final int nextLeft = getChildRectInPagerCoordinates(mTempRect, nextFocused).left;
                final int currLeft = getChildRectInPagerCoordinates(mTempRect, currentFocused).left;
                if (currentFocused != null && nextLeft <= currLeft) {
                    handled = pageRight();
                } else {
                    handled = nextFocused.requestFocus();
                }
            }
        } else if (direction == FOCUS_LEFT || direction == FOCUS_BACKWARD) {
            // Trying to move left and nothing there; try to page.
            handled = pageLeft();
        } else if (direction == FOCUS_RIGHT || direction == FOCUS_FORWARD) {
            // Trying to move right and nothing there; try to page.
            handled = pageRight();
        }
        if (handled) {
            playSoundEffect(SoundEffectConstants.getContantForFocusDirection(direction));
        }
        return handled;
    }
```
 分析以上代码调用可知dispatchKeyEvent方法拦截了左右和tab按键，得到nextFocused后requestFocus，导致切页
  
 禁止遥控切页的方法：继承viewpager后重写executeKeyEvent或arrowScroll方法，返回false即可 
    
