title: 开源项目解析-Android-Universal-Image-Loader
date: 2015-02-08 22:42:26
categories:  Android
tags: uil
---
#简介
Android-Universal-Image-Loader是一个用来加载图片（包括网络图片和本地图片）的开源项目，实现了网络图片的下载、本地图片的加载、图片的解析、图片的缓存等机制。
其实很多大神都已经解析过此开源项目。
包括但不限于以下博客
<a href="http://www.codekk.com/open-source-project-analysis/detail/Android/huxian99/Android%20Universal%20Image%20Loader%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90">codekk解析</a>
夏安明的3篇博文
<a href="http://blog.csdn.net/xiaanming/article/details/26810303">Android开源框架Universal-Image-Loader完全解析（一）--- 基本介绍及使用</a>
<a href="http://blog.csdn.net/xiaanming/article/details/27525741">Android开源框架Universal-Image-Loader完全解析（二）--- 图片缓存策略详解</a>
<a href="http://blog.csdn.net/xiaanming/article/details/39057201">Android开源框架Universal-Image-Loader完全解析（三）---源代码解读</a>
 但我还是继续写这篇博文的原因主要还是让自己好好看下这部分代码。哈哈，大家想要了解的话看上面几篇博客也已经够了。
<!--more-->

#使用

对于其使用非常简单
##初始化与配置ImageLoader
<figure class="highlight"><table><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div><div class="line">4</div><div class="line">5</div><div class="line">6</div><div class="line">7</div><div class="line">8</div><div class="line">9</div><div class="line">10</div><div class="line">11</div><div class="line">12</div><div class="line">13</div><div class="line">14</div><div class="line">15</div><div class="line">16</div><div class="line">17</div><div class="line">18</div><div class="line">19</div><div class="line">20</div><div class="line">21</div><div class="line">22</div><div class="line">23</div><div class="line">24</div><div class="line">25</div><div class="line">26</div><div class="line">27</div></pre></td><td class="code"><pre><div class="line"><span class="comment">//默认配置</span></div><div class="line"> ImageLoaderConfiguration configuration <span class="subst">=</span> ImageLoaderConfiguration </div><div class="line"> <span class="built_in">.</span>createDefault(this); </div><div class="line"></div><div class="line"><span class="comment">//个性化配置，建造者模式</span></div><div class="line"> File cacheDir <span class="subst">=</span> StorageUtils<span class="built_in">.</span>getCacheDirectory(context); </div><div class="line"> ImageLoaderConfiguration config <span class="subst">=</span> <span class="literal">new</span> ImageLoaderConfiguration<span class="built_in">.</span>Builder(context) </div><div class="line"> <span class="built_in">.</span>memoryCacheExtraOptions(<span class="number">480</span>, <span class="number">800</span>) <span class="comment">// 默认屏幕大小</span></div><div class="line"> <span class="built_in">.</span>diskCacheExtraOptions(<span class="number">480</span>, <span class="number">800</span>, CompressFormat<span class="built_in">.</span>JPEG, <span class="number">75</span>, <span class="built_in">null</span>) </div><div class="line"> <span class="built_in">.</span>taskExecutor(<span class="attribute">...</span>) <span class="comment">//线程池</span></div><div class="line"> <span class="built_in">.</span>taskExecutorForCachedImages(<span class="attribute">...</span>) </div><div class="line"> <span class="built_in">.</span>threadPoolSize(<span class="number">3</span>) <span class="comment">//</span></div><div class="line"> <span class="built_in">.</span>threadPriority(<span class="keyword">Thread</span><span class="built_in">.</span>NORM_PRIORITY <span class="subst">-</span> <span class="number">1</span>) <span class="comment">// 线程优先级</span></div><div class="line"> <span class="built_in">.</span>tasksProcessingOrder(QueueProcessingType<span class="built_in">.</span>FIFO) <span class="comment">//处理顺序，先进先出等 </span></div><div class="line"> <span class="built_in">.</span>denyCacheImageMultipleSizesInMemory() </div><div class="line"> <span class="built_in">.</span>memoryCache(<span class="literal">new</span> LruMemoryCache(<span class="number">2</span> <span class="subst">*</span> <span class="number">1024</span> <span class="subst">*</span> <span class="number">1024</span>)) <span class="comment">//内存缓存 </span></div><div class="line"> <span class="built_in">.</span>memoryCacheSize(<span class="number">2</span> <span class="subst">*</span> <span class="number">1024</span> <span class="subst">*</span> <span class="number">1024</span>) </div><div class="line"> <span class="built_in">.</span>memoryCacheSizePercentage(<span class="number">13</span>) <span class="comment">// default </span></div><div class="line"> <span class="built_in">.</span>diskCache(<span class="literal">new</span> UnlimitedDiscCache(cacheDir)) <span class="comment">// 磁盘缓存，自定义地址</span></div><div class="line"> <span class="built_in">.</span>diskCacheSize(<span class="number">50</span> <span class="subst">*</span> <span class="number">1024</span> <span class="subst">*</span> <span class="number">1024</span>) </div><div class="line"> <span class="built_in">.</span>diskCacheFileCount(<span class="number">100</span>) </div><div class="line"> <span class="built_in">.</span>diskCacheFileNameGenerator(<span class="literal">new</span> HashCodeFileNameGenerator()) <span class="comment">// default </span></div><div class="line"> <span class="built_in">.</span>imageDownloader(<span class="literal">new</span> BaseImageDownloader(context)) <span class="comment">// 下载器 </span></div><div class="line"> <span class="built_in">.</span>imageDecoder(<span class="literal">new</span> BaseImageDecoder()) <span class="comment">// 解析器</span></div><div class="line"> <span class="built_in">.</span>defaultDisplayImageOptions(DisplayImageOptions<span class="built_in">.</span>createSimple()) </div><div class="line"> <span class="built_in">.</span>build(); </div><div class="line"> ImageLoader<span class="built_in">.</span>getInstance()<span class="built_in">.</span>init(configuration);</div></pre></td></tr></table></figure>

##加载配置项

<p>对于每次要加载的显示项在加载时可进行设置</p>
<figure class="highlight"><table><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div><div class="line">4</div><div class="line">5</div><div class="line">6</div><div class="line">7</div><div class="line">8</div><div class="line">9</div><div class="line">10</div><div class="line">11</div><div class="line">12</div><div class="line">13</div><div class="line">14</div><div class="line">15</div><div class="line">16</div><div class="line">17</div><div class="line">18</div></pre></td><td class="code"><pre><div class="line">DisplayImageOptions options = new DisplayImageOptions.Builder() </div><div class="line"> .showImageOnLoading(R.drawable.ic_stub) //设置加载前默认图片</div><div class="line"> .showImageForEmptyUri(R.drawable.ic_empty) </div><div class="line"> .showImageOnFail(R.drawable.ic_error) </div><div class="line"> .resetViewBeforeLoading(false) //加载前会清空图片</div><div class="line"> .delayBeforeLoading(<span class="number">1000</span>) </div><div class="line"> .cacheInMemory(true) //开始缓存</div><div class="line"> .cacheOnDisk(true) </div><div class="line"> .preProcessor(<span class="keyword">...</span>) </div><div class="line"> .postProcessor(<span class="keyword">...</span>) </div><div class="line"> .extraForDownloader(<span class="keyword">...</span>) </div><div class="line"> .considerExifParams(false) // default </div><div class="line"> .imageScaleType(ImageScaleType.IN_SAMPLE_POWER_OF_2) //类似imageview scaleType</div><div class="line"> .bitmapConfig(Bitmap.Config.ARGB_8888) // 在要求不高时可以用<span class="number">555</span>缩小内存占用 </div><div class="line"> .decodingOptions(<span class="keyword">...</span>) </div><div class="line"> .displayer(new SimpleBitmapDisplayer()) // 渲染器，可以加一些特殊效果，例如矩形圆角等 </div><div class="line"> .handler(new Handler()) // default </div><div class="line"> .build();</div></pre></td></tr></table></figure>

##下载与加载图片

<figure class="highlight"><table><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div><div class="line">4</div><div class="line">5</div><div class="line">6</div><div class="line">7</div><div class="line">8</div><div class="line">9</div><div class="line">10</div><div class="line">11</div><div class="line">12</div><div class="line">13</div><div class="line">14</div></pre></td><td class="code"><pre><div class="line"> loadImage需要自己给imageview在回调里设置图片</div><div class="line"> ImageLoader.getInstance().loadImage(imageUrl, mImageSize, options, <span class="keyword">new</span> SimpleImageLoadingListener(){ </div><div class="line"> </div><div class="line"> <span class="annotation">@Override</span> </div><div class="line"> <span class="keyword">public</span> <span class="keyword">void</span> <span class="title">onLoadingComplete</span>(String imageUri, View view, </div><div class="line"> Bitmap loadedImage) { </div><div class="line"> <span class="keyword">super</span>.onLoadingComplete(imageUri, view, loadedImage); </div><div class="line"> mImageView.setImageBitmap(loadedImage); </div><div class="line"> } </div><div class="line"> </div><div class="line"> }); </div><div class="line">或</div><div class="line">displayImage直接将imageview传给displayImage方法</div><div class="line"> ImageLoader.getInstance().displayImage(imageUrl, mImageView, options);</div></pre></td></tr></table></figure>

##其它TIP

<figure class="highlight"><table><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div><div class="line">4</div><div class="line">5</div><div class="line">6</div><div class="line">7</div><div class="line">8</div><div class="line">9</div><div class="line">10</div><div class="line">11</div></pre></td><td class="code"><pre><div class="line">设置滑动时不加载</div><div class="line">setOnScrollListener(<span class="keyword">new</span> PauseOnScrollListener(imageLoader, pauseOnScroll, pauseOnFling)); </div><div class="line"></div><div class="line">图片url</div><div class="line"> <span class="built_in">String</span> netPath=<span class="string">"http://coderrobin/image/.png"</span>;</div><div class="line"> <span class="built_in">String</span> localPath=<span class="string">"/mnt/sdcard/coderrobin.png"</span> </div><div class="line"> <span class="built_in">String</span> contentprividerUrl = <span class="string">"content://media/external/audio/albumart/13"</span>; </div><div class="line"> <span class="comment">//图片来源于assets </span></div><div class="line"> <span class="built_in">String</span> assetsUrl = Scheme.ASSETS.wrap(<span class="string">"image.png"</span>); </div><div class="line"> <span class="comment">//图片来源于 </span></div><div class="line"> <span class="built_in">String</span> drawableUrl = Scheme.DRAWABLE.wrap(<span class="string">"R.drawable.image"</span>);</div></pre></td></tr></table></figure>

#源码分析
源码涉及的代码较多，本文只分析比较核心的几个类
##uil涉及到的模块与概念
ImageLoaderEngine：任务分发器，负责分发LoadAndDisplayImageTask和ProcessAndDisplayImageTask给具体的线程池去执行
ImageAware：显示图片的对象，可以是ImageView等,主要为对imageview用弱引用封装，并且增加了获取imageview参数（宽高等）的接口
ImageDownloader：图片下载器，负责从图片的各个来源获取输入流（网络、本地等）
Cache：图片缓存，分为MemoryCache和DiskCache。
MemoryCache：内存图片缓存，常用LruMemoryCache。
DiskCache：磁盘图片缓存.
ImageDecoder：图片解码器，负责将图片输入流InputStream转换为Bitmap对象,默认调用bitmaoFactory.decode方法
BitmapProcessor：图片处理器，负责从缓存读取或写入前对图片进行处理。
BitmapDisplayer：将Bitmap对象显示在相应的控件ImageAware上, 加一些特殊显示效果
LoadAndDisplayImageTask：用于加载并显示图片的任务。
ProcessAndDisplayImageTask：用于处理并显示图片的任务。
DisplayBitmapTask：用于显示图片的任务。
 
##ViewAware分析，对view控件的封装,ImageViewAware即继承该类
```
public abstract class ViewAware implements ImageAware {
	protected Reference<View> viewRef;
	protected boolean checkActualViewSize;

	public ViewAware(View view, boolean checkActualViewSize) {
		if (view == null) throw new IllegalArgumentException("view must not be null");
               //弱引用封装view,避免不能被系统回收
		this.viewRef = new WeakReference<View>(view);
		this.checkActualViewSize = checkActualViewSize;
	}

	//获取宽度方法，分为是不是wrap_content(onMeasure中at_most)
	@Override
	public int getWidth() {
		View view = viewRef.get();
		if (view != null) {
			final ViewGroup.LayoutParams params = view.getLayoutParams();
			int width = 0;
			if (checkActualViewSize && params != null && params.width != ViewGroup.LayoutParams.WRAP_CONTENT) {
				width = view.getWidth(); // Get actual image width
			}
			if (width <= 0 && params != null) width = params.width; // Get layout width parameter
			return width;
		}
		return 0;
	}
	
       //设置图片，用looper判断是否在主线程
	@Override
	public boolean setImageBitmap(Bitmap bitmap) {
		if (Looper.myLooper() == Looper.getMainLooper()) {
			View view = viewRef.get();
			if (view != null) {
				setImageBitmapInto(bitmap, view);
				return true;
			}
		} else {
			L.w(WARN_CANT_SET_BITMAP);
		}
		return false;
	}

}
```
##ImageLoader类分析
```
public class ImageLoader {

	private ImageLoaderConfiguration configuration;
	//任务引擎（分发器）
	private ImageLoaderEngine engine;

	private final ImageLoadingListener emptyListener = new SimpleImageLoadingListener();

	private volatile static ImageLoader instance;
       //单例，双重加锁
	public static ImageLoader getInstance() {
		if (instance == null) {
			synchronized (ImageLoader.class) {
				if (instance == null) {
					instance = new ImageLoader();
				}
			}
		}
		return instance;
	}

	protected ImageLoader() {
	}

	//初始化，必须传递配置项，否则报错
	public synchronized void init(ImageLoaderConfiguration configuration) {
		if (configuration == null) {
			throw new IllegalArgumentException(ERROR_INIT_CONFIG_WITH_NULL);
		}
		if (this.configuration == null) {
			L.d(LOG_INIT_CONFIG);
			engine = new ImageLoaderEngine(configuration);
			this.configuration = configuration;
		} else {
			L.w(WARNING_RE_INIT_CONFIG);
		}
	}
	
	//省略很多displayImage各种形参的方法
	
	//displayImage最终调用到的方法
	public void displayImage(String uri, ImageAware imageAware, DisplayImageOptions options,
			ImageLoadingListener listener, ImageLoadingProgressListener progressListener) {
		checkConfiguration();
		if (imageAware == null) {
			throw new IllegalArgumentException(ERROR_WRONG_ARGUMENTS);
		}
		if (listener == null) {
			listener = emptyListener;
		}
		if (options == null) {
			options = configuration.defaultDisplayImageOptions;
		}
		//url为空
		if (TextUtils.isEmpty(uri)) {
			engine.cancelDisplayTaskFor(imageAware);
			listener.onLoadingStarted(uri, imageAware.getWrappedView());
			if (options.shouldShowImageForEmptyUri()) {
				imageAware.setImageDrawable(options.getImageForEmptyUri(configuration.resources));
			} else {
				imageAware.setImageDrawable(null);
			}
			listener.onLoadingComplete(uri, imageAware.getWrappedView(), null);
			return;
		}

		ImageSize targetSize = ImageSizeUtils.defineTargetSizeForView(imageAware, configuration.getMaxImageSize());
		String memoryCacheKey = MemoryCacheUtils.generateKey(uri, targetSize);
		engine.prepareDisplayTaskFor(imageAware, memoryCacheKey);
		
		listener.onLoadingStarted(uri, imageAware.getWrappedView());
		//在内存缓存中是否存在
		Bitmap bmp = configuration.memoryCache.get(memoryCacheKey);
		//已存在
		if (bmp != null && !bmp.isRecycled()) {
			L.d(LOG_LOAD_IMAGE_FROM_MEMORY_CACHE, memoryCacheKey);
		 //需要处理时进行处理
			if (options.shouldPostProcess()) {
				ImageLoadingInfo imageLoadingInfo = new ImageLoadingInfo(uri, imageAware, targetSize, memoryCacheKey,
						options, listener, progressListener, engine.getLockForUri(uri));
				ProcessAndDisplayImageTask displayTask = new ProcessAndDisplayImageTask(engine, bmp, imageLoadingInfo,
						defineHandler(options));
				if (options.isSyncLoading()) {
					displayTask.run();
				} else {
					engine.submit(displayTask);
				}
			} else {
				//直接调用显示模块显示
				options.getDisplayer().display(bmp, imageAware, LoadedFrom.MEMORY_CACHE);
				listener.onLoadingComplete(uri, imageAware.getWrappedView(), bmp);
			}
		} else {
			//显示默认值
			if (options.shouldShowImageOnLoading()) {
				imageAware.setImageDrawable(options.getImageOnLoading(configuration.resources));
			} else if (options.isResetViewBeforeLoading()) {
				imageAware.setImageDrawable(null);
			}

			ImageLoadingInfo imageLoadingInfo = new ImageLoadingInfo(uri, imageAware, targetSize, memoryCacheKey,
					options, listener, progressListener, engine.getLockForUri(uri));
			LoadAndDisplayImageTask displayTask = new LoadAndDisplayImageTask(engine, imageLoadingInfo,
					defineHandler(options));
			if (options.isSyncLoading()) {
				displayTask.run();
			} else {
				engine.submit(displayTask);
			}
		}
	}
    //loadImage,最终仍调用displayImage但imageAware是个NonViewAware
	public void loadImage(String uri, ImageSize targetImageSize, DisplayImageOptions options,
			ImageLoadingListener listener, ImageLoadingProgressListener progressListener) {
		checkConfiguration();
		if (targetImageSize == null) {
			targetImageSize = configuration.getMaxImageSize();
		}
		if (options == null) {
			options = configuration.defaultDisplayImageOptions;
		}

		NonViewAware imageAware = new NonViewAware(uri, targetImageSize, ViewScaleType.CROP);
		displayImage(uri, imageAware, options, listener, progressListener);
	}

...

}
```
##ImageLoaderEngine引擎，负责提交任务给各线程池
```

class ImageLoaderEngine {
      ...
	/** Submits task to execution pool */
	void submit(final LoadAndDisplayImageTask task) {
		taskDistributor.execute(new Runnable() {
			@Override
			public void run() {
				File image = configuration.diskCache.get(task.getLoadingUri());
				boolean isImageCachedOnDisk = image != null && image.exists();
				initExecutorsIfNeed();
				if (isImageCachedOnDisk) {
					taskExecutorForCachedImages.execute(task);
				} else {
					taskExecutor.execute(task);
				}
			}
		});
	}

	/** Submits task to execution pool */
	void submit(ProcessAndDisplayImageTask task) {
		initExecutorsIfNeed();
		taskExecutorForCachedImages.execute(task);
	}

	...


}
```
##各task解析
所谓task,就是实现runnable接口的类
###LoadAndDisplayImageTask
负责从网络或者本地获得bitmap，经过处理器处理后（可没有处理器），交给DisplayBitmapTask负责后续显示逻辑
```
@Override
	public void run() {
		if (waitIfPaused()) return;
		if (delayIfNeed()) return;

		ReentrantLock loadFromUriLock = imageLoadingInfo.loadFromUriLock;
		L.d(LOG_START_DISPLAY_IMAGE_TASK, memoryCacheKey);
		if (loadFromUriLock.isLocked()) {
			L.d(LOG_WAITING_FOR_IMAGE_LOADED, memoryCacheKey);
		}

		loadFromUriLock.lock();
		Bitmap bmp;
		try {
			checkTaskNotActual();
		//查看内存缓存中是否已存在
			bmp = configuration.memoryCache.get(memoryCacheKey);
			if (bmp == null || bmp.isRecycled()) {
				//获取butmap
				bmp = tryLoadBitmap();
				if (bmp == null) return; // listener callback already was fired

				checkTaskNotActual();
				checkTaskInterrupted();

				if (options.shouldPreProcess()) {
					L.d(LOG_PREPROCESS_IMAGE, memoryCacheKey);
					bmp = options.getPreProcessor().process(bmp);
					if (bmp == null) {
						L.e(ERROR_PRE_PROCESSOR_NULL, memoryCacheKey);
					}
				}

				if (bmp != null && options.isCacheInMemory()) {
					L.d(LOG_CACHE_IMAGE_IN_MEMORY, memoryCacheKey);
					configuration.memoryCache.put(memoryCacheKey, bmp);
				}
			} else {
				loadedFrom = LoadedFrom.MEMORY_CACHE;
				L.d(LOG_GET_IMAGE_FROM_MEMORY_CACHE_AFTER_WAITING, memoryCacheKey);
			}

			if (bmp != null && options.shouldPostProcess()) {
				L.d(LOG_POSTPROCESS_IMAGE, memoryCacheKey);
				bmp = options.getPostProcessor().process(bmp);
				if (bmp == null) {
					L.e(ERROR_POST_PROCESSOR_NULL, memoryCacheKey);
				}
			}
			checkTaskNotActual();
			checkTaskInterrupted();
		} catch (TaskCancelledException e) {
			fireCancelEvent();
			return;
		} finally {
			loadFromUriLock.unlock();
		}

		DisplayBitmapTask displayBitmapTask = new DisplayBitmapTask(bmp, imageLoadingInfo, engine, loadedFrom);
		runTask(displayBitmapTask, syncLoading, handler, engine);
	}



	private Bitmap tryLoadBitmap() throws TaskCancelledException {
		Bitmap bitmap = null;
		try {
		//磁盘缓存中是否已存在
			File imageFile = configuration.diskCache.get(uri);
			if (imageFile != null && imageFile.exists()) {
				L.d(LOG_LOAD_IMAGE_FROM_DISK_CACHE, memoryCacheKey);
				loadedFrom = LoadedFrom.DISC_CACHE;

				checkTaskNotActual();
				bitmap = decodeImage(Scheme.FILE.wrap(imageFile.getAbsolutePath()));
			}
			if (bitmap == null || bitmap.getWidth() <= 0 || bitmap.getHeight() <= 0) {
				L.d(LOG_LOAD_IMAGE_FROM_NETWORK, memoryCacheKey);
				loadedFrom = LoadedFrom.NETWORK;

				String imageUriForDecoding = uri;
				if (options.isCacheOnDisk() && tryCacheImageOnDisk()) {
					imageFile = configuration.diskCache.get(uri);
					if (imageFile != null) {
						imageUriForDecoding = Scheme.FILE.wrap(imageFile.getAbsolutePath());
					}
				}

				checkTaskNotActual();
				bitmap = decodeImage(imageUriForDecoding);

				if (bitmap == null || bitmap.getWidth() <= 0 || bitmap.getHeight() <= 0) {
					fireFailEvent(FailType.DECODING_ERROR, null);
				}
			}
		} catch (IllegalStateException e) {
			fireFailEvent(FailType.NETWORK_DENIED, null);
		} catch (TaskCancelledException e) {
			throw e;
		} catch (IOException e) {
			L.e(e);
			fireFailEvent(FailType.IO_ERROR, e);
		} catch (OutOfMemoryError e) {
			L.e(e);
			fireFailEvent(FailType.OUT_OF_MEMORY, e);
		} catch (Throwable e) {
			L.e(e);
			fireFailEvent(FailType.UNKNOWN, e);
		}
		return bitmap;
	}
```

###ProcessAndDisplayImageTask
负责获取到bitmap后进行前置处理，处理完成后交给DisplayBitmapTask负责显示
```
@Override
	public void run() {
		L.d(LOG_POSTPROCESS_IMAGE, imageLoadingInfo.memoryCacheKey);

		BitmapProcessor processor = imageLoadingInfo.options.getPostProcessor();
		Bitmap processedBitmap = processor.process(bitmap);
		DisplayBitmapTask displayBitmapTask = new DisplayBitmapTask(processedBitmap, imageLoadingInfo, engine,
				LoadedFrom.MEMORY_CACHE);
//静态方法，只是扔给线程池处理，与LoadAndDisplayImageTask无关
		LoadAndDisplayImageTask.runTask(displayBitmapTask, imageLoadingInfo.options.isSyncLoading(), handler, engine);
	}
```
###DisplayBitmapTask
```
@Override
	public void run() {
		if (imageAware.isCollected()) {
			L.d(LOG_TASK_CANCELLED_IMAGEAWARE_COLLECTED, memoryCacheKey);
			listener.onLoadingCancelled(imageUri, imageAware.getWrappedView());
		} else if (isViewWasReused()) {
			L.d(LOG_TASK_CANCELLED_IMAGEAWARE_REUSED, memoryCacheKey);
			listener.onLoadingCancelled(imageUri, imageAware.getWrappedView());
		} else {
			L.d(LOG_DISPLAY_IMAGE_IN_IMAGEAWARE, loadedFrom, memoryCacheKey);
			displayer.display(bitmap, imageAware, loadedFrom);
			engine.cancelDisplayTaskFor(imageAware);
			listener.onLoadingComplete(imageUri, imageAware.getWrappedView(), bitmap);
		}
	}
```
##downloader
负责从网络和本地获取流，以下BaseImageDownloader为getStream方法
```

	@Override
	public InputStream getStream(String imageUri, Object extra) throws IOException {
		switch (Scheme.ofUri(imageUri)) {
			case HTTP:
			case HTTPS:
				return getStreamFromNetwork(imageUri, extra);
			case FILE:
				return getStreamFromFile(imageUri, extra);
			case CONTENT:
				return getStreamFromContent(imageUri, extra);
			case ASSETS:
				return getStreamFromAssets(imageUri, extra);
			case DRAWABLE:
				return getStreamFromDrawable(imageUri, extra);
			case UNKNOWN:
			default:
				return getStreamFromOtherSource(imageUri, extra);
		}
	}
```
##decoder
以下为BaseImageDecoder的decode方法，将输入流转化为bitmap
```
@Override
	public Bitmap decode(ImageDecodingInfo decodingInfo) throws IOException {
		Bitmap decodedBitmap;
		ImageFileInfo imageInfo;

		InputStream imageStream = getImageStream(decodingInfo);
		if (imageStream == null) {
			L.e(ERROR_NO_IMAGE_STREAM, decodingInfo.getImageKey());
			return null;
		}
		try {
			imageInfo = defineImageSizeAndRotation(imageStream, decodingInfo);
			imageStream = resetStream(imageStream, decodingInfo);
			Options decodingOptions = prepareDecodingOptions(imageInfo.imageSize, decodingInfo);
			decodedBitmap = BitmapFactory.decodeStream(imageStream, null, decodingOptions);
		} finally {
			IoUtils.closeSilently(imageStream);
		}

		if (decodedBitmap == null) {
			L.e(ERROR_CANT_DECODE_IMAGE, decodingInfo.getImageKey());
		} else {
			decodedBitmap = considerExactScaleAndOrientatiton(decodedBitmap, decodingInfo, imageInfo.exif.rotation,
					imageInfo.exif.flipHorizontal);
		}
		return decodedBitmap;
	}

```
##displayer
将bitmap显示出来,以下为SimpleBitmapDisplayer display方法
```
	@Override
	public void display(Bitmap bitmap, ImageAware imageAware, LoadedFrom loadedFrom) {
		imageAware.setImageBitmap(bitmap);
	}
```

#总结
1.作者模块功能划分的很好，各模块职责清晰，但对部分功能调用上有些混乱.
如存在没有使用processAndDisplayImageTask而在外部直接调用postProcesser,没有使用DisplayImageTask而在外部直接调用displayer,这部分关系较为混乱
2.所谓的downloader实际上是包括加载网络和本地，和通常理解的概念有偏差，loadedFrom枚举类里面的network竟然也包括本地文件，堪称坑爹。
3.可配置性很强，可以添加自己的各种模块，比如下载器，解析器，cache,线程池等










