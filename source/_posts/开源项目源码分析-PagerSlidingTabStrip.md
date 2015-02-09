title:  开源项目源码分析-PagerSlidingTabStrip
date:  2015-02-04 22:18:43
categories:  Android
tags:  PagerSlidingTabStrip
---
最近在项目中用到了PagerSlidingTabStrip（<a href="https://github.com/astuetz/PagerSlidingTabStrip">github地址</a>），于是就看了下其实现，发现实现不是特别复杂，算是一个学习自定义控件的不错的例子，于是便拿出来跟大家分享下。
PagerSlidingTabStrip控件效果如下，支持viewpager滑动时，选项卡联动
![pagerslidingtab](/image/pagerslidingtab.png  "pagerslidingtab")

#使用方式
下面我们先来看看其使用方式

##xml中配置：
```
 <com.astuetz.PagerSlidingTabStrip
        android:id="@+id/tabs"
        android:layout_width="match_parent"
        android:layout_height=“wrap_content"
       />
```
<!--more-->
##绑定ViewPager
```
@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);

		tabs = (PagerSlidingTabStrip) findViewById(R.id.tabs);
		pager = (ViewPager) findViewById(R.id.pager);
		adapter = new MyPagerAdapter(getFragmentManager());

		pager.setAdapter(adapter);
		tabs.setViewPager(pager);
	}
```
##让自定义的viewpager的adapter实现方法
若需要导航栏为图标，则实现PagerSlidingTabStrip提供的IconTabProvider接口的getPageIconResId方法

若需要导航栏内容为文字，则直接实现adapter的getPageTitle方法

若需要实现其他方式，比如图片下面加文字等，则需要修改addTab的方法，看完以下源码分析就可以知道了如何修改了

##自定义属性 
PagerSlidingTabStrip提供了如下自定义属性
```
    pstsIndicatorColor  滑动指示器颜色
    pstsUnderlineColor 视图的底部的全宽线的颜色
    pstsDividerColor       选项卡之间的分隔线的颜色
    pstsIndicatorHeight  滑动指标高度
    pstsUnderlineHeight  视图的底部高度的全宽线
    pstsDividerPadding  顶部和分频器的底部填充
    pstsTabPaddingLeftRight 每个选项卡的padding
    pstsTabBackground tab背景，可以为StateListDrawable
    pstsShouldExpand    如果设置为true，每个选项卡被赋予了相同的weight，长度一致，默认为false
    pstsTextAllCaps   如果为true，所有的选项卡标题将是大写，默认为true
```
 #源码分析
知道如何使用后我们来分析源码。了解具体的实现过程和在使用过程中应注意的事项。
```
//继承HorizontalScrollView，导航栏超出界面时可左右滑动
public class PagerSlidingTabStrip extends HorizontalScrollView 
	//viewpager adpter可选择实现的接口
	public interface IconTabProvider {
		public int getPageIconResId(int position);
	}

	public PagerSlidingTabStrip(Context context, AttributeSet attrs,
 int defStyle) {
		super(context, attrs, defStyle);
		setFillViewport(true);
		setWillNotDraw(false);
		//动态生成导航栏的LinearLayout，并将其addview到父控件中
		tabsContainer = new LinearLayout(context);
		tabsContainer.setOrientation(LinearLayout.HORIZONTAL);
		tabsContainer.setLayoutParams(new LayoutParams(
		LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT));
		addView(tabsContainer);

//该处省略获取自定义属性的代码
		
		rectPaint = new Paint();
		rectPaint.setAntiAlias(true);
		rectPaint.setStyle(Style.FILL);

		dividerPaint = new Paint();
		dividerPaint.setAntiAlias(true);
		dividerPaint.setStrokeWidth(dividerWidth);

		defaultTabLayoutParams = new LinearLayout.LayoutParams
(LayoutParams.WRAP_CONTENT, LayoutParams.MATCH_PARENT);
		expandedTabLayoutParams = new LinearLayout.LayoutParams(0, 
LayoutParams.MATCH_PARENT, 1.0f);
if (locale == null) {
			locale = getResources().getConfiguration().locale;
		}
	}
	//设置viewpager接口
public void setViewPager(ViewPager pager) {
		this.pager = pager;

		//设置viewpager时需先设置adapter否则会报错，
		//因为无法获得导航栏的需要的内容
		if (pager.getAdapter() == null) {
			throw new IllegalStateException
("ViewPager does not have adapter instance.");
		}
		
		//设置了viewpager的OnPageChangeListener,因此用户不可以给viewpager
		//设置该监听，而应该给pagerSlidingtabStrip设置监听（用了静态代理），
		//否则会导致该控件无法正常使用，
		pager.setOnPageChangeListener(pageListener);

		notifyDataSetChanged();
	}

public void setOnPageChangeListener(OnPageChangeListener listener) {
		this.delegatePageListener = listener;
	}


public void notifyDataSetChanged() {
		//移除全部子视图，根据adapter内容addTab
		tabsContainer.removeAllViews();

		tabCount = pager.getAdapter().getCount();

		for (int i = 0; i < tabCount; i++) {

			if (pager.getAdapter() instanceof IconTabProvider) {
				addIconTab(i, ((IconTabProvider) pager.getAdapter()).
				getPageIconResId(i));
			} else {
				addTextTab(i, pager.getAdapter().getPageTitle(i).toString());
			}

		}

		updateTabStyles();

		getViewTreeObserver().addOnGlobalLayoutListener(
		new OnGlobalLayoutListener() {

			@SuppressWarnings("deprecation")
			@SuppressLint("NewApi")
			@Override
			public void onGlobalLayout() {

			if (Build.VERSION.SDK_INT < Build.VERSION_CODES.JELLY_BEAN) {
				getViewTreeObserver().removeGlobalOnLayoutListener(this);
			} else {
				getViewTreeObserver().removeOnGlobalLayoutListener(this);
			}

				currentPosition = pager.getCurrentItem();
				scrollToChild(currentPosition, 0);
			}
		});

	}


	//添加文字导航栏内容
	private void addTextTab(final int position, String title) {

		TextView tab = new TextView(getContext());
		tab.setText(title);
		tab.setGravity(Gravity.CENTER);
		tab.setSingleLine();

		addTab(position, tab);
	}
	//添加图片导航栏内容
	private void addIconTab(final int position, int resId) {

		ImageButton tab = new ImageButton(getContext());
		tab.setImageResource(resId);

		addTab(position, tab);

	}

	private void addTab(final int position, View tab) {
		tab.setFocusable(true);
		tab.setOnClickListener(new OnClickListener() {
			@Override
			public void onClick(View v) {
				pager.setCurrentItem(position);
			}
		});
		//设置每个tab左右的padding
		tab.setPadding(tabPadding, 0, tabPadding, 0);

		//往container里面动态添加tab,根据shouldExpand layoutparm分别为
		//expandedTabLayoutParam(每个tab weight相同即宽度一致)
		tabsContainer.addView(tab, position, shouldExpand ?
		 expandedTabLayoutParams : defaultTabLayoutParams);
	
	}

	//设置每个tab的风格，包含背景、文字大小、颜色、字体、是否大写等
	private void updateTabStyles() {

		for (int i = 0; i < tabCount; i++) {

			View v = tabsContainer.getChildAt(i);

			v.setBackgroundResource(tabBackgroundResId);

			if (v instanceof TextView) {

				TextView tab = (TextView) v;
				tab.setTextSize(TypedValue.COMPLEX_UNIT_PX, tabTextSize);
				tab.setTypeface(tabTypeface, tabTypefaceStyle);
				tab.setTextColor(tabTextColor);

				if (textAllCaps) {
					if (Build.VERSION.SDK_INT >=
 					Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
						tab.setAllCaps(true);
					} else {
						tab.setText(tab.getText().toString().
						toUpperCase(locale));
					}
				}
			}
		}

	}

	//滚动到特定位置,调用父类scrollTo，布局超出边界才会起作用
	private void scrollToChild(int position, int offset) {

		if (tabCount == 0) {
			return;
		}

		int newScrollX = tabsContainer.getChildAt(position)
		.getLeft() + offset;

		if (position > 0 || offset > 0) {
			newScrollX -= scrollOffset;
		}

		if (newScrollX != lastScrollX) {
			lastScrollX = newScrollX;
			scrollTo(newScrollX, 0);
		}

	}

	//onDraw方法，重头戏
	@Override
	protected void onDraw(Canvas canvas) {
		super.onDraw(canvas);

		if (isInEditMode() || tabCount == 0) {
			return;
		}
		//获取pagerslidingTab的底部坐标,后续都是在底部-高度获得top坐标
		final int height = getHeight();

		//根据当前tab坐标和下一个tab坐标和currentPositionOffset
		//计算底部指示器坐标并绘制，实际是一个特定颜色的矩形

		rectPaint.setColor(indicatorColor);
		View currentTab = tabsContainer.getChildAt(currentPosition);
		float lineLeft = currentTab.getLeft();
		float lineRight = currentTab.getRight();
		if (currentPositionOffset > 0f && currentPosition < tabCount - 1) {

			View nextTab = tabsContainer.getChildAt(currentPosition + 1);
			final float nextTabLeft = nextTab.getLeft();
			final float nextTabRight = nextTab.getRight();

			lineLeft = (currentPositionOffset * nextTabLeft +

 (1f - currentPositionOffset) * lineLeft);
			lineRight = (currentPositionOffset * nextTabRight +

 (1f - currentPositionOffset) * lineRight);
		}

		canvas.drawRect(lineLeft, height - indicatorHeight,
		 lineRight, height, rectPaint);

		
		//根据高度绘制底部矩形长条
		rectPaint.setColor(underlineColor);
		canvas.drawRect(0, height - underlineHeight,
	        tabsContainer.getWidth(), height, rectPaint);


		//绘制各个tab之间纵向的dividerPadding长度的分隔线

		dividerPaint.setColor(dividerColor);
		for (int i = 0; i < tabCount - 1; i++) {
			View tab = tabsContainer.getChildAt(i);
			canvas.drawLine(tab.getRight(), dividerPadding, tab.getRight(), 
			height -dividerPadding, dividerPaint);
		}
	}

//内部PageListener类，用于监听滑动事件得到当前滑动偏移量和当前tab
private class PageListener implements OnPageChangeListener {

		@Override
		public void onPageScrolled(int position, 
		float positionOffset, int positionOffsetPixels) {

			currentPosition = position;
			currentPositionOffset = positionOffset;

			scrollToChild(position, (int) (positionOffset * 
			tabsContainer.getChildAt(position).getWidth()));

			invalidate();
			//给pagerslidingtab加滑动监听即委托执行delegatePageListener的方法
			if (delegatePageListener != null) {
				delegatePageListener.onPageScrolled(position, 
					positionOffset, positionOffsetPixels);
			}
		}

		@Override
		public void onPageScrollStateChanged(int state) {
			if (state == ViewPager.SCROLL_STATE_IDLE) {
				scrollToChild(pager.getCurrentItem(), 0);
			}

			if (delegatePageListener != null) {
				delegatePageListener.onPageScrollStateChanged(state);
			}
		}

		@Override
		public void onPageSelected(int position) {
			if (delegatePageListener != null) {
				delegatePageListener.onPageSelected(position);
			}
		}

	}

/×
省略获取和设置属性的一系列方法
×/



//activity销毁时在onSaveInstanceState记录当前页数，
//当onRestoreInstanceState获得当前页数并重新
//requestLayout进行测量、布局和绘制
@Override
	public void onRestoreInstanceState(Parcelable state) {
		SavedState savedState = (SavedState) state;
		super.onRestoreInstanceState(savedState.getSuperState());
		currentPosition = savedState.currentPosition;
		requestLayout();
	}

	@Override
	public Parcelable onSaveInstanceState() {
		Parcelable superState = super.onSaveInstanceState();
		SavedState savedState = new SavedState(superState);
		savedState.currentPosition = currentPosition;
		return savedState;
	}

	static class SavedState extends BaseSavedState {
		int currentPosition;

		public SavedState(Parcelable superState) {
			super(superState);
		}

		private SavedState(Parcel in) {
			super(in);
			currentPosition = in.readInt();
		}

		@Override
		public void writeToParcel(Parcel dest, int flags) {
			super.writeToParcel(dest, flags);
			dest.writeInt(currentPosition);
		}

		public static final Parcelable.Creator<SavedState> CREATOR 
		= new Parcelable.Creator<SavedState>() {
			@Override
			public SavedState createFromParcel(Parcel in) {
				return new SavedState(in);
			}

			@Override
			public SavedState[] newArray(int size) {
				return new SavedState[size];
			}
		};
	}

}
```
#总结：
以上就是对pagerSlidingTabStrip源码的分析，整个源码较简单
1.在container（linerlayout）中添加了tab(textview或者imageButton)
2.在onDraw方法里面根据当前页和偏移量计算和绘制矩形指示器、矩形底部长栏、纵向的tab之间的分割线，
3.对viewpager的onPageListener中获取当前偏移量并滑动到该子tab