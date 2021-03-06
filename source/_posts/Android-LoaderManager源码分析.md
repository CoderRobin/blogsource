title: Android Loader解析
date: 2015-02-05 23:02:04
categories:  Android
tags: LoaderManager

---
#简介：

loader机制在android 3.0后加入android framework。其目的在于方便开发人员在activity和fragment中异步地加载数据。（可以替代直接使用线程或者AsyncTask，对需要异步加载的过程进行统一管理）
另外loader还有如下特征：
1.对数据源变化进行监听，实时更新数据。
2.在activity配置发生变化（如横竖屏切换时无需重复加载数据）
3.对于CursorLoader，开发人员无需去close cursor，系统会帮忙管理
<!--more-->

#使用步骤
以下为loader最简单的使用步骤
1.在activity或fragment中getLoaderManager()
2.oncreate中调用initLoader(id, null, LoaderManager.Callbacks);
 LoaderManager Callbacks有如下方法
    onCreateLoader() — 创建loader（可以为android提供的cursorLoader或者自己继承AsyncLoader）,供loadmanager管理
    onLoadFinished() — 每次数据加载完时回调（包括数据源更改时loader监听变化时加载）
    onLoaderReset() —  loader被重置时（不再使用）时回调
对于详细使用方法还不了解的童鞋可以先阅读
<a href="http://developer.android.com/guide/components/loaders.html">google trainging</a>
androiddesignpatterns的4篇博客
<a href="http://www.androiddesignpatterns.com/2012/07/loaders-and-loadermanager-background.html"> Part 1: Life Before Loaders</a>
<a href=" http://www.androiddesignpatterns.com/2012/07/understanding-loadermanager.html"> Part 2: Understanding the LoaderManager</a>
 <a href=" http://www.androiddesignpatterns.com/2012/08/implementing-loaders.html">Part 3: Implementing Loaders</a>
  <a href=" http://www.androiddesignpatterns.com/2012/09/tutorial-loader-loadermanager.html">  Part 4: Tutorial: AppListLoader</a>



#源码分析
loader的分析过程涉及到如下类:
activity
fragment
loaderManager
loader
AsyncLoader
CursorLoader
其实我也想画下类图流程图神马的来跟大家梳理下这些源码，但奈何对uml图神马的只看得懂不会画啊。。不喜欢撸源码的人直接看文末的总结吧，确实直接看源码挺恶心。。。
##loader类解析
```
public class Loader<D> {
   //loader唯一id
    int mId;
    OnLoadCompleteListener<D> mListener;
    OnLoadCanceledListener<D> mOnLoadCanceledListener;
    Context mContext;
    boolean mStarted = false;
    boolean mAbandoned = false;
    boolean mReset = true;
    boolean mContentChanged = false;
    boolean mProcessingChange = false;

	//数据源变化监听器，cursorLoader有使用到
    public final class ForceLoadContentObserver extends ContentObserver {
        public ForceLoadContentObserver() {
            super(new Handler());
        }

        @Override
        public boolean deliverSelfNotifications() {
            return true;
        }

        @Override
        public void onChange(boolean selfChange) {
            onContentChanged();
        }
    }

 //loaderManager向loader注册加载完成接口，当加载完成时loader通知loaderManager,loaderManager再通知界面
    public interface OnLoadCompleteListener<D> {
        public void onLoadComplete(Loader<D> loader, D data);
    }


    public interface OnLoadCanceledListener<D> {
        public void onLoadCanceled(Loader<D> loader);
    }

    public Loader(Context context) {
        mContext = context.getApplicationContext();
    }
   
   //加载完成时回调
    public void deliverResult(D data) {
        if (mListener != null) {
            mListener.onLoadComplete(this, data);
        }
    }

   //loader子类调用父类该方法取消loader
    public void deliverCancellation() {
        if (mOnLoadCanceledListener != null) {
            mOnLoadCanceledListener.onLoadCanceled(this);
        }
    }

 //省略注册和注销OnLoadCompleteListener和mOnLoadCanceledListener
//的方法

 
   //开始加载数据时loaderManager调用
   //设置标志位及调用onStartLoading方法
  //用户不可手动调用，否则会造成状态紊乱
    public final void startLoading() {
        mStarted = true;
        mReset = false;
        mAbandoned = false;
        onStartLoading();
    }

   //子类需继承，真正加载数据的地方
    protected void onStartLoading() {
    }

    //取消loader方法
    public boolean cancelLoad() {
        return onCancelLoad();
    }

  //子类需继承,return false表示取消不掉（已完成或未开始）
    protected boolean onCancelLoad() {
        return false;
    }

  //强制重新loader，会抛弃旧数据
    public void forceLoad() {
        onForceLoad();
    }
   
   //子类需继承
    protected void onForceLoad() {
    }
   
   //activity或fragment stop时loaderManager会调用该方法
    public void stopLoading() {
        mStarted = false;
        onStopLoading();
    }

    //子类实现
    protected void onStopLoading() {
    }

    //loaderManager restartLoader时调用
    public void abandon() {
        mAbandoned = true;
        onAbandon();
    }
    
   //子类实现
    protected void onAbandon() {
    }
    
   //销毁loader时调用
    public void reset() {
        onReset();
        mReset = true;
        mStarted = false;
        mAbandoned = false;
        mContentChanged = false;
        mProcessingChange = false;
    }

   //子类实现
    protected void onReset() {
    }

    /**
     * Take the current flag indicating whether the loader's content had
     * changed while it was stopped.  If it had, true is returned and the
     * flag is cleared.
     */
   //判断loader停止期间数据是否变化了
    public boolean takeContentChanged() {
        boolean res = mContentChanged;
        mContentChanged = false;
        mProcessingChange |= res;
        return res;
    }

    /**
     * Commit that you have actually fully processed a content change that
     * was returned by {@link #takeContentChanged}.  This is for use with
     * {@link #rollbackContentChanged()} to handle situations where a load
     * is cancelled.  Call this when you have completely processed a load
     * without it being cancelled.
     */
    public void commitContentChanged() {
        mProcessingChange = false;
    }

    /**
     * Report that you have abandoned the processing of a content change that
     * was returned by {@link #takeContentChanged()} and would like to rollback
     * to the state where there is again a pending content change.  This is
     * to handle the case where a data load due to a content change has been
     * canceled before its data was delivered back to the loader.
     */
    public void rollbackContentChanged() {
        if (mProcessingChange) {
            mContentChanged = true;
        }
    }

    //数据源改变时observer调用该方法，若loader已经start了，重新加载数	据 ，否则，设置标志位
    public void onContentChanged() {
        if (mStarted) {
            forceLoad();
        } else {
            // This loader has been stopped, so we don't want to load
            // new data right now...  but keep track of it changing to
            // refresh later if we start again.
            mContentChanged = true;
        }
    }
...
}
```

##loaderManager解析
```
public abstract class LoaderManager {
 
    public interface LoaderCallbacks<D> {.
         
        public Loader<D> onCreateLoader(int id, Bundle args);
        public void onLoadFinished(Loader<D> loader, D data);
        public void onLoaderReset(Loader<D> loader);
    }

    //loaderManager 若未创建loader,创建loader,若已经创建，则更新callback后返回
    public abstract <D> Loader<D> initLoader(int id, Bundle args,
            LoaderManager.LoaderCallbacks<D> callback);

     
     //创建新loader,已有相同id的loader替换之
    public abstract <D> Loader<D> restartLoader(int id, Bundle args,
            LoaderManager.LoaderCallbacks<D> callback);

   //销毁loader,若onLoader finish过则调用onLoaderReset
    public abstract void destroyLoader(int id);

    /**
     * Return the Loader with the given id or null if no matching Loader
     * is found.
     */
    public abstract <D> Loader<D> getLoader(int id);

   ...
}


class LoaderManagerImpl extends LoaderManager {
    static final String TAG = "LoaderManager";
    static boolean DEBUG = false;
    
    //散列表，保存活跃中的loaders
    final SparseArray<LoaderInfo> mLoaders = new SparseArray<LoaderInfo>(0);

    // These are previously run loaders.  This list is maintained internally
    // to avoid destroying a loader while an application is still using it.
    // It allows an application to restart a loader, but continue using its
    // previously run loader until the new loader's data is available.
    //保存已经运行完毕的loader
    final SparseArray<LoaderInfo> mInactiveLoaders = new SparseArray<LoaderInfo>(0);

    final String mWho;

    Activity mActivity;
    boolean mStarted;
    boolean mRetaining;
    boolean mRetainingStarted;
    
    boolean mCreatingLoader;

    //对loader的封装
    final class LoaderInfo implements Loader.OnLoadCompleteListener<Object>,
            Loader.OnLoadCanceledListener<Object> {
        final int mId;
        final Bundle mArgs;
        LoaderManager.LoaderCallbacks<Object> mCallbacks;
        Loader<Object> mLoader;
        //已经加载过数据
        boolean mHaveData;
        //已经将加载的数据投递给界面
        boolean mDeliveredData;
        Object mData;
        //已经start
        boolean mStarted;
        //配置发生改变时保持当前loader,无需销毁
        boolean mRetaining;
        boolean mRetainingStarted;
        boolean mReportNextStart;
        boolean mDestroyed;
        boolean mListenerRegistered;

        LoaderInfo mPendingLoader;
        
        public LoaderInfo(int id, Bundle args, LoaderManager.LoaderCallbacks<Object> callbacks) {
            mId = id;
            mArgs = args;
            mCallbacks = callbacks;
        }
        
        //启动loader
        void start() {
        	
            if (mRetaining && mRetainingStarted) {
                // Our owner is started, but we were being retained from a
                // previous instance in the started state...  so there is really
                // nothing to do here, since the loaders are still started.
                mStarted = true;
                return;
            }

            if (mStarted) {
                // If loader already started, don't restart.
                return;
            }

            mStarted = true;
            
            if (DEBUG) Log.v(TAG, "  Starting: " + this);
            //为空则创建loader
            if (mLoader == null && mCallbacks != null) {
               mLoader = mCallbacks.onCreateLoader(mId, mArgs);
            }
            if (mLoader != null) {
            	//非静态内部类抛异常
                if (mLoader.getClass().isMemberClass()
                        && !Modifier.isStatic(mLoader.getClass().getModifiers())) {
                    throw new IllegalArgumentException(
                            "Object returned from onCreateLoader must not be a non-static inner member class: "
                            + mLoader);
                }
                //注册监听
                if (!mListenerRegistered) {
                    mLoader.registerListener(mId, this);
                    mLoader.registerOnLoadCanceledListener(this);
                    mListenerRegistered = true;
                }
                
                mLoader.startLoading();
            }
        }
        
        
        //配置改变时进行保持，设置标志位
        void retain() {
            if (DEBUG) Log.v(TAG, "  Retaining: " + this);
            mRetaining = true;
            mRetainingStarted = mStarted;
            mStarted = false;
            mCallbacks = null;
        }
        
        //activity已重新启动，若有数据则通知
        void finishRetain() {
            if (mRetaining) {
                if (DEBUG) Log.v(TAG, "  Finished Retaining: " + this);
                mRetaining = false;
                if (mStarted != mRetainingStarted) {
                    if (!mStarted) {
                        // This loader was retained in a started state, but
                        // at the end of retaining everything our owner is
                        // no longer started...  so make it stop.
                        stop();
                    }
                }
            }

            if (mStarted && mHaveData && !mReportNextStart) {
                // This loader has retained its data, either completely across
                // a configuration change or just whatever the last data set
                // was after being restarted from a stop, and now at the point of
                // finishing the retain we find we remain started, have
                // our data, and the owner has a new callback...  so
                // let's deliver the data now.
                callOnLoadFinished(mLoader, mData);
            }
        }
        
        void reportStart() {
            if (mStarted) {
		//没分析到会执行
                if (mReportNextStart) {
                    mReportNextStart = false;
                    if (mHaveData) {
                        callOnLoadFinished(mLoader, mData);
                    }
                }
            }
        }
        
        //停止，若无需保存，注销回调
        void stop() {
            if (DEBUG) Log.v(TAG, "  Stopping: " + this);
            mStarted = false;
            if (!mRetaining) {
                if (mLoader != null && mListenerRegistered) {
                    // Let the loader know we're done with it
                    mListenerRegistered = false;
                    mLoader.unregisterListener(this);
                    mLoader.unregisterOnLoadCanceledListener(this);
                    mLoader.stopLoading();
                }
            }
        }
        
        //取消
        void cancel() {
            if (DEBUG) Log.v(TAG, "  Canceling: " + this);
            if (mStarted && mLoader != null && mListenerRegistered) {
                if (!mLoader.cancelLoad()) {
                    onLoadCanceled(mLoader);
                }
            }
        }
        
        //销毁
        void destroy() {
            if (DEBUG) Log.v(TAG, "  Destroying: " + this);
            mDestroyed = true;
            boolean needReset = mDeliveredData;
            mDeliveredData = false;
            //有过数据，则调用callback的onLoaderReset通知用户i界面
            if (mCallbacks != null && mLoader != null && mHaveData && needReset) {
                if (DEBUG) Log.v(TAG, "  Reseting: " + this);
                String lastBecause = null;
                if (mActivity != null) {
                    lastBecause = mActivity.mFragments.mNoTransactionsBecause;
                    mActivity.mFragments.mNoTransactionsBecause = "onLoaderReset";
                }
                try {
                    mCallbacks.onLoaderReset(mLoader);
                } finally {
                    if (mActivity != null) {
                        mActivity.mFragments.mNoTransactionsBecause = lastBecause;
                    }
                }
            }
            mCallbacks = null;
            mData = null;
            mHaveData = false;
            if (mLoader != null) {
                if (mListenerRegistered) {
                    mListenerRegistered = false;
                    mLoader.unregisterListener(this);
                    mLoader.unregisterOnLoadCanceledListener(this);
                }
                //重置loader,CursorLoader会关闭cursor等
                mLoader.reset();
            }
            if (mPendingLoader != null) {
                mPendingLoader.destroy();
            }
        }

        //loader被取消时回调该方法（在AsyncTaskLoader中有用到）
        @Override
        public void onLoadCanceled(Loader<Object> loader) {
            if (DEBUG) Log.v(TAG, "onLoadCanceled: " + this);

            if (mDestroyed) {
                if (DEBUG) Log.v(TAG, "  Ignoring load canceled -- destroyed");
                return;
            }

            if (mLoaders.get(mId) != this) {
                // This cancellation message is not coming from the current active loader.
                // We don't care about it.
                if (DEBUG) Log.v(TAG, "  Ignoring load canceled -- not active");
                return;
            }

            LoaderInfo pending = mPendingLoader;

            //restart loader时，若有loader已经开始执行，则在新loader加入pendingLoader,发生取消时，若已有在排队的loader,执行在排队的loader
            if (pending != null) {
                // There is a new request pending and we were just
                // waiting for the old one to cancel or complete before starting
                // it.  So now it is time, switch over to the new loader.
                if (DEBUG) Log.v(TAG, "  Switching to pending loader: " + pending);
                mPendingLoader = null;
                mLoaders.put(mId, null);
                destroy();
                installLoader(pending);
            }
        }

        //加载完成时回调
        @Override
        public void onLoadComplete(Loader<Object> loader, Object data) {
            if (DEBUG) Log.v(TAG, "onLoadComplete: " + this);
            
            if (mDestroyed) {
                if (DEBUG) Log.v(TAG, "  Ignoring load complete -- destroyed");
                return;
            }

            if (mLoaders.get(mId) != this) {
                // This data is not coming from the current active loader.
                // We don't care about it.
                if (DEBUG) Log.v(TAG, "  Ignoring load complete -- not active");
                return;
            }
            
           //加载完成时，若已有在排队的loader,则废弃该loader,执行在排队的loader 
            LoaderInfo pending = mPendingLoader;
            if (pending != null) {
                // There is a new request pending and we were just
                // waiting for the old one to complete before starting
                // it.  So now it is time, switch over to the new loader.
                if (DEBUG) Log.v(TAG, "  Switching to pending loader: " + pending);
                mPendingLoader = null;
                mLoaders.put(mId, null);
                destroy();
                installLoader(pending);
                return;
            }
            
            //标记已有数据，调用callOnLoadFinished
            if (mData != data || !mHaveData) {
                mData = data;
                mHaveData = true;
                if (mStarted) {
                    callOnLoadFinished(loader, data);
                }
            }

            //if (DEBUG) Log.v(TAG, "  onLoadFinished returned: " + this);

            // We have now given the application the new loader with its
            // loaded data, so it should have stopped using the previous
            // loader.  If there is a previous loader on the inactive list,
            // clean it up.
            LoaderInfo info = mInactiveLoaders.get(mId);
            if (info != null && info != this) {
                info.mDeliveredData = false;
                info.destroy();
                mInactiveLoaders.remove(mId);
            }

            if (mActivity != null && !hasRunningLoaders()) {
                mActivity.mFragments.startPendingDeferredFragments();
            }
        }

        //调用onLoadFinished
        void callOnLoadFinished(Loader<Object> loader, Object data) {
            if (mCallbacks != null) {
                String lastBecause = null;
                if (mActivity != null) {
                    lastBecause = mActivity.mFragments.mNoTransactionsBecause;
                    mActivity.mFragments.mNoTransactionsBecause = "onLoadFinished";
                }
                try {
                    if (DEBUG) Log.v(TAG, "  onLoadFinished in " + loader + ": "
                            + loader.dataToString(data));
                    mCallbacks.onLoadFinished(loader, data);
                } finally {
                    if (mActivity != null) {
                        mActivity.mFragments.mNoTransactionsBecause = lastBecause;
                    }
                }
                mDeliveredData = true;
            }
        }
        
   ...
    }
    
    LoaderManagerImpl(String who, Activity activity, boolean started) {
        mWho = who;
        mActivity = activity;
        mStarted = started;
    }
    
    void updateActivity(Activity activity) {
        mActivity = activity;
    }
    
    
    //创建loader,包装成LoaderInfo
    private LoaderInfo createLoader(int id, Bundle args,
            LoaderManager.LoaderCallbacks<Object> callback) {
        LoaderInfo info = new LoaderInfo(id, args,  (LoaderManager.LoaderCallbacks<Object>)callback);
        Loader<Object> loader = callback.onCreateLoader(id, args);
        info.mLoader = (Loader<Object>)loader;
        return info;
    }
    
    private LoaderInfo createAndInstallLoader(int id, Bundle args,
            LoaderManager.LoaderCallbacks<Object> callback) {
        try {
            mCreatingLoader = true;
            LoaderInfo info = createLoader(id, args, callback);
            installLoader(info);
            return info;
        } finally {
            mCreatingLoader = false;
        }
    }
    
    //放入散列表，若activity已onstart 启动loader
    void installLoader(LoaderInfo info) {
        mLoaders.put(info.mId, info);
        if (mStarted) {
            // The activity will start all existing loaders in it's onStart(),
            // so only start them here if we're past that point of the activitiy's
            // life cycle
            info.start();
        }
    }
    //相同id的不再创建
    //通常在activity 创建时调用，且再配置改变时不重复创建
    @SuppressWarnings("unchecked")
    public <D> Loader<D> initLoader(int id, Bundle args, LoaderManager.LoaderCallbacks<D> callback) {
        if (mCreatingLoader) {
            throw new IllegalStateException("Called while creating a loader");
        }
        
        LoaderInfo info = mLoaders.get(id);
        
        if (DEBUG) Log.v(TAG, "initLoader in " + this + ": args=" + args);

        if (info == null) {
            // Loader doesn't already exist; create.
            info = createAndInstallLoader(id, args,  (LoaderManager.LoaderCallbacks<Object>)callback);
            if (DEBUG) Log.v(TAG, "  Created new loader " + info);
        } else {
            if (DEBUG) Log.v(TAG, "  Re-using existing loader " + info);
            info.mCallbacks = (LoaderManager.LoaderCallbacks<Object>)callback;
        }
        
        //已经有数据，直接callOnLoadFinished
        if (info.mHaveData && mStarted) {
            // If the loader has already generated its data, report it now.
            info.callOnLoadFinished(info.mLoader, info.mData);
        }
        
        return (Loader<D>)info.mLoader;
    }
    
   //重新创造loader
    @SuppressWarnings("unchecked")
    public <D> Loader<D> restartLoader(int id, Bundle args, LoaderManager.LoaderCallbacks<D> callback) {
        if (mCreatingLoader) {
            throw new IllegalStateException("Called while creating a loader");
        }
        
        LoaderInfo info = mLoaders.get(id);
        if (DEBUG) Log.v(TAG, "restartLoader in " + this + ": args=" + args);
        if (info != null) {
            LoaderInfo inactive = mInactiveLoaders.get(id);
            if (inactive != null) {
                if (info.mHaveData) {
                    if (DEBUG) Log.v(TAG, "  Removing last inactive loader: " + info);
                    inactive.mDeliveredData = false;
                    inactive.destroy();
                    info.mLoader.abandon();
                    mInactiveLoaders.put(id, info);
                } else {
                    if (!info.mStarted) {
                  
                        if (DEBUG) Log.v(TAG, "  Current loader is stopped; replacing");
                        mLoaders.put(id, null);
                        info.destroy();
                    } else {
                        // Now we have three active loaders... we'll queue
                        // up this request to be processed once one of the other loaders
                        // finishes or is canceled.
                        if (DEBUG) Log.v(TAG, "  Current loader is running; attempting to cancel");
                        //此时，说明一个同id的Loader正在加载数据，但是尚未加载完成，此时我们调用它的cancel()通知退出加载。
                        info.cancel();
                        if (info.mPendingLoader != null) {
                            if (DEBUG) Log.v(TAG, "  Removing pending loader: " + info.mPendingLoader);
                            info.mPendingLoader.destroy();
                            info.mPendingLoader = null;
                        }
                      //创建一个此id值的Loader，并赋给mPendingLoader，也许你会问为什么不直接启动它呢，  
                      //别忘了，此时已经有一个Loader在加载数据了，而且我们已经调用它的cancel来通知它退出加载，当然，通常情况下，  
                      //这个Loader也许刚好加载完成，那么会收到onLoadComplete()回调，也许正常退出加载，此时收到回调onLoadCanceled()，  
                      //所以在这两个回调里，我们再启动mPendingLoader，不信的话，参考一下这两个方法，就能找到installLoader(pending)。
                        if (DEBUG) Log.v(TAG, "  Enqueuing as new pending loader");
                        info.mPendingLoader = createLoader(id, args, 
                                (LoaderManager.LoaderCallbacks<Object>)callback);
                        return (Loader<D>)info.mPendingLoader.mLoader;
                    }
                }
            } else {
                // Keep track of the previous instance of this loader so we can destroy
                // it when the new one completes.
                if (DEBUG) Log.v(TAG, "  Making last loader inactive: " + info);
                info.mLoader.abandon();
                mInactiveLoaders.put(id, info);
            }
        }
        
        info = createAndInstallLoader(id, args,  (LoaderManager.LoaderCallbacks<Object>)callback);
        return (Loader<D>)info.mLoader;
    }
    
    /**
     * Rip down, tear apart, shred to pieces a current Loader ID.  After returning
     * from this function, any Loader objects associated with this ID are
     * destroyed.  Any data associated with them is destroyed.  You better not
     * be using it when you do this.
     * @param id Identifier of the Loader to be destroyed.
     */
    public void destroyLoader(int id) {
        if (mCreatingLoader) {
            throw new IllegalStateException("Called while creating a loader");
        }
        
        if (DEBUG) Log.v(TAG, "destroyLoader in " + this + " of " + id);
        int idx = mLoaders.indexOfKey(id);
        if (idx >= 0) {
            LoaderInfo info = mLoaders.valueAt(idx);
            mLoaders.removeAt(idx);
            info.destroy();
        }
        idx = mInactiveLoaders.indexOfKey(id);
        if (idx >= 0) {
            LoaderInfo info = mInactiveLoaders.valueAt(idx);
            mInactiveLoaders.removeAt(idx);
            info.destroy();
        }
        if (mActivity != null && !hasRunningLoaders()) {
            mActivity.mFragments.startPendingDeferredFragments();
        }
    }

    /**
     * Return the most recent Loader object associated with the
     * given ID.
     */
    @SuppressWarnings("unchecked")
    public <D> Loader<D> getLoader(int id) {
        if (mCreatingLoader) {
            throw new IllegalStateException("Called while creating a loader");
        }
        
        LoaderInfo loaderInfo = mLoaders.get(id);
        if (loaderInfo != null) {
            if (loaderInfo.mPendingLoader != null) {
                return (Loader<D>)loaderInfo.mPendingLoader.mLoader;
            }
            return (Loader<D>)loaderInfo.mLoader;
        }
        return null;
    }
 
    void doStart() {
        if (DEBUG) Log.v(TAG, "Starting in " + this);
        if (mStarted) {
            RuntimeException e = new RuntimeException("here");
            e.fillInStackTrace();
            Log.w(TAG, "Called doStart when already started: " + this, e);
            return;
        }
        
        mStarted = true;

        // Call out to sub classes so they can start their loaders
        // Let the existing loaders know that we want to be notified when a load is complete
        for (int i = mLoaders.size()-1; i >= 0; i--) {
            mLoaders.valueAt(i).start();
        }
    }
    
    void doStop() {
        if (DEBUG) Log.v(TAG, "Stopping in " + this);
        if (!mStarted) {
            RuntimeException e = new RuntimeException("here");
            e.fillInStackTrace();
            Log.w(TAG, "Called doStop when not started: " + this, e);
            return;
        }

        for (int i = mLoaders.size()-1; i >= 0; i--) {
            mLoaders.valueAt(i).stop();
        }
        mStarted = false;
    }
    
    void doRetain() {
        if (DEBUG) Log.v(TAG, "Retaining in " + this);
        if (!mStarted) {
            RuntimeException e = new RuntimeException("here");
            e.fillInStackTrace();
            Log.w(TAG, "Called doRetain when not started: " + this, e);
            return;
        }

        mRetaining = true;
        mStarted = false;
        for (int i = mLoaders.size()-1; i >= 0; i--) {
            mLoaders.valueAt(i).retain();
        }
    }
    
    void finishRetain() {
        if (mRetaining) {
            if (DEBUG) Log.v(TAG, "Finished Retaining in " + this);

            mRetaining = false;
            for (int i = mLoaders.size()-1; i >= 0; i--) {
                mLoaders.valueAt(i).finishRetain();
            }
        }
    }
    
    void doReportNextStart() {
        for (int i = mLoaders.size()-1; i >= 0; i--) {
            mLoaders.valueAt(i).mReportNextStart = true;
        }
    }

    void doReportStart() {
        for (int i = mLoaders.size()-1; i >= 0; i--) {
            mLoaders.valueAt(i).reportStart();
        }
    }

    void doDestroy() {
        if (!mRetaining) {
            if (DEBUG) Log.v(TAG, "Destroying Active in " + this);
            for (int i = mLoaders.size()-1; i >= 0; i--) {
                mLoaders.valueAt(i).destroy();
            }
            mLoaders.clear();
        }
        
        if (DEBUG) Log.v(TAG, "Destroying Inactive in " + this);
        for (int i = mInactiveLoaders.size()-1; i >= 0; i--) {
            mInactiveLoaders.valueAt(i).destroy();
        }
        mInactiveLoaders.clear();
    }

    。。。
    public boolean hasRunningLoaders() {
        boolean loadersRunning = false;
        final int count = mLoaders.size();
        for (int i = 0; i < count; i++) {
            final LoaderInfo li = mLoaders.valueAt(i);
            loadersRunning |= li.mStarted && !li.mDeliveredData;
        }
        return loadersRunning;
    }
}
```

##Activity和Fragment对loadManager的管理
Activity和Fragment 在其生命周期内会控制loader的doStart,doStop,配置改变过程对loader进行管理。

###Loadermanager生命周期控制
####Activity
```
    LoaderManagerImpl getLoaderManager(String who, boolean started, boolean create) {  
            if (mAllLoaderManagers == null) {  
                mAllLoaderManagers = new ArrayMap<String, LoaderManagerImpl>();  
            }  
            LoaderManagerImpl lm = mAllLoaderManagers.get(who);  
            if (lm == null) {  
                if (create) {  
              //实际创建LoaderManagerImpl
                    lm = new LoaderManagerImpl(who, this, started);
	//mAllLoaderManagers保存activity和其fragment的所有loader  
                    mAllLoaderManagers.put(who, lm);  
                }  
            } else {  
                lm.updateActivity(this);  
            }  
            return lm;          
}  
    protected void onStart() {  
//省略无关代码
            if (!mLoadersStarted) {  
                mLoadersStarted = true;  
                if (mLoaderManager != null) {  
//关键方法.dostart
                    mLoaderManager.doStart();  
                } else if (!mCheckedForLoaderManager) {  
                    mLoaderManager = getLoaderManager("(root)", mLoadersStarted, false);  
                }  
                mCheckedForLoaderManager = true;  
            }  

            ...
        }  



    final void performStop() {  
            …….  
            if (mLoadersStarted) {  
                mLoadersStarted = false;  
                if (mLoaderManager != null) {  
                    if (!mChangingConfigurations) {  
                        mLoaderManager.doStop();  
                    } else {  
                        mLoaderManager.doRetain();  
                    }  
                }  
            }  
    ……..  
    }  

mChangingConfigurations变量的作用也很简单：如果当前发生了配置变化这个变量就会被置为true，那么我们需要调用LoaderManager.doRetain()方法，表示需要保存当前的loaderManager，在Activity恢复时也要恢复这个LoaderManager。如果Activity的stop不是由于配置变化引起的，那么直接调用LoaderManager的doStop()方法，直接停止当前的LoaderManager。

    final void performDestroy() {  
            mDestroyed = true;  
            mWindow.destroy();  
            mFragments.dispatchDestroy();  
            onDestroy();  
            if (mLoaderManager != null) {  
                mLoaderManager.doDestroy();  
            }  
    }   
```

####Fragment
```
     // 调用Activity getLoaderManager创建loaderManager
    public LoaderManager getLoaderManager() {
        if (mLoaderManager != null) {
            return mLoaderManager;
        }
        if (mActivity == null) {
            throw new IllegalStateException("Fragment " + this + " not attached to Activity");
        }
        mCheckedForLoaderManager = true;
        mLoaderManager = mActivity.getLoaderManager(mWho, mLoadersStarted, true);
        return mLoaderManager;
    }

   public void onStart() {
        mCalled = true;
        
        if (!mLoadersStarted) {
            mLoadersStarted = true;
            if (!mCheckedForLoaderManager) {
                mCheckedForLoaderManager = true;
                mLoaderManager = mActivity.getLoaderManager(mWho, mLoadersStarted, false);
            }
            if (mLoaderManager != null) {
                mLoaderManager.doStart();
            }
        }


    }

    void performReallyStop() {
        if (mChildFragmentManager != null) {
            mChildFragmentManager.dispatchReallyStop();
        }
        if (mLoadersStarted) {
            mLoadersStarted = false;
            if (!mCheckedForLoaderManager) {
                mCheckedForLoaderManager = true;
                mLoaderManager = mActivity.getLoaderManager(mWho, mLoadersStarted, false);
            }
            if (mLoaderManager != null) {
                if (!mActivity.mRetaining) {
                    mLoaderManager.doStop();
                } else {
                    mLoaderManager.doRetain();
                }
            }
        }
    }

    public void onDestroy() {
        mCalled = true;
        //Log.v("foo", "onDestroy: mCheckedForLoaderManager=" + mCheckedForLoaderManager
        //        + " mLoaderManager=" + mLoaderManager);
        if (!mCheckedForLoaderManager) {
            mCheckedForLoaderManager = true;
            mLoaderManager = mActivity.getLoaderManager(mWho, mLoadersStarted, false);
        }
        if (mLoaderManager != null) {
            mLoaderManager.doDestroy();
        }
    }
```
###Activity配置改变时保存和恢复loader
```
    NonConfigurationInstances retainNonConfigurationInstances() {  
            Object activity = onRetainNonConfigurationInstance();  
            HashMap<String, Object> children = onRetainNonConfigurationChildInstances();  
            ArrayList<Fragment> fragments = mFragments.retainNonConfig();  
            boolean retainLoaders = false;  
            if (mAllLoaderManagers != null) {  
                // prune out any loader managers that were already stopped and so  
                // have nothing useful to retain.  
                final int N = mAllLoaderManagers.size();  
                LoaderManagerImpl loaders[] = new LoaderManagerImpl[N];  
                for (int i=N-1; i>=0; i--) {  
                    loaders[i] = mAllLoaderManagers.valueAt(i);  
                }  
                for (int i=0; i<N; i++) {  
                    LoaderManagerImpl lm = loaders[i];  
                    if (lm.mRetaining) {  
                        retainLoaders = true;  
                    } else {  
                        lm.doDestroy();  
                        mAllLoaderManagers.remove(lm.mWho);  
                    }  
                }  
            }  
            if (activity == null && children == null && fragments == null && !retainLoaders) {  
                return null;  
            }  
              
            NonConfigurationInstances nci = new NonConfigurationInstances();  
            nci.activity = activity;  
            nci.children = children;  
            nci.fragments = fragments;  
            nci.loaders = mAllLoaderManagers;  
            return nci;  
    }  


看上面的代码，其关键在于变量mAllLoaderManagers。这个变量上面已经介绍过了，它保存了所有Activity中创建的LoaderManager引用，自然也包含了mLoaderManager引用了。这个方法会被ActivityThread类调用，而且早于performDestroy()方法调用。
这个方法会查看mAllLoaderManagers保存的所有引用，如果某个LoaderManagerImpl.mRetaining值为false的话，就会被remove掉。如果配置变化发生的话，mRetaining = true，当然mRetaining这个值是在performStop()中的doRetain()调用来设置的。

activity onCreate()方法：
    protected void onCreate(Bundle savedInstanceState) {  
            if (DEBUG_LIFECYCLE) Slog.v(TAG, "onCreate " + this + ": " + savedInstanceState);  
            if (mLastNonConfigurationInstances != null) {  
                mAllLoaderManagers = mLastNonConfigurationInstances.loaders;  
            }  

原来mAllLoaderManagers是从mLastNonConfigurationInstances中恢复的，而mLastNonConfigurationInstances的恢复其实是从onAttach()方法中开始的：
    final void attach(Context context, ActivityThread aThread,  
                Instrumentation instr, IBinder token, int ident,  
                Application application, Intent intent, ActivityInfo info,  
                CharSequence title, Activity parent, String id,  
                NonConfigurationInstances lastNonConfigurationInstances,  
                Configuration config) {  
  ...
    mLastNonConfigurationInstances = lastNonConfigurationInstances;  
}

    final void performStart() {  
...
            if (mAllLoaderManagers != null) {  
                final int N = mAllLoaderManagers.size();  
                LoaderManagerImpl loaders[] = new LoaderManagerImpl[N];  
                for (int i=N-1; i>=0; i--) {  
                    loaders[i] = mAllLoaderManagers.valueAt(i);  
                }  
                for (int i=0; i<N; i++) {  
                    LoaderManagerImpl lm = loaders[i];  
                    lm.finishRetain();  
                    lm.doReportStart();  
                }  
            }  
        }  
上面的代码通过调用LoaderManager.finishRetain()以及doReportStart()方法来恢复LoaderManager的状态。
```
 
以上就是Activity针对LoaderManager的管理，而Loader的管理自然是LoaderManager的事了。从上面的分析可以看出，Activity和LoaderManager的职能划分很清晰，这也大大简化了Loaders的管理。

##Loader实现
  ###AsyncTaskLoader抽象类
本质上是对AsyncTask的封装，通过AsyncTask加载数据与回调给ui
```
public abstract class AsyncTaskLoader<D> extends Loader<D> {
    static final String TAG = "AsyncTaskLoader";
    static final boolean DEBUG = false;

    final class LoadTask extends AsyncTask<Void, Void, D> implements Runnable {
...

        @Override
        
        protected D doInBackground(Void... params) {
            if (DEBUG) Log.v(TAG, this + " >>> doInBackground");
            try {
		//加载数据
                D data = AsyncTaskLoader.this.onLoadInBackground();
                if (DEBUG) Log.v(TAG, this + "  <<< doInBackground");
                return data;
            } catch (OperationCanceledException ex) {
                if (!isCancelled()) {

                    throw ex;
                }
                if (DEBUG) Log.v(TAG, this + "  <<< doInBackground (was canceled)", ex);
                return null;
            }
        }

        /* Runs on the UI thread */
        @Override
        protected void onPostExecute(D data) {
            if (DEBUG) Log.v(TAG, this + " onPostExecute");
            try {
		//回调执行完毕
                AsyncTaskLoader.this.dispatchOnLoadComplete(this, data);
            } finally {
                mDone.countDown();
            }
        }

        /* Runs on the UI thread */
        @Override
        protected void onCancelled(D data) {
            if (DEBUG) Log.v(TAG, this + " onCancelled");
            try {
                AsyncTaskLoader.this.dispatchOnCancelled(this, data);
            } finally {
                mDone.countDown();
            }
   
        @Override
        public void run() {
            waiting = false;
            AsyncTaskLoader.this.executePendingTask();
        }
...
    }

    private final Executor mExecutor;

    volatile LoadTask mTask;
    volatile LoadTask mCancellingTask;

    long mUpdateThrottle;
    long mLastLoadCompleteTime = -10000;
    Handler mHandler;

    public AsyncTaskLoader(Context context) {
        this(context, AsyncTask.THREAD_POOL_EXECUTOR);
    }

    /** {@hide} */
    public AsyncTaskLoader(Context context, Executor executor) {
        super(context);
        mExecutor = executor;
    }

    /**
     * Set amount to throttle updates by.  This is the minimum time from
     * when the last {@link #loadInBackground()} call has completed until
     * a new load is scheduled.
     *
     * @param delayMS Amount of delay, in milliseconds.
     */
    public void setUpdateThrottle(long delayMS) {
        mUpdateThrottle = delayMS;
        if (delayMS != 0) {
            mHandler = new Handler();
        }
    }

    @Override
    protected void onForceLoad() {
        super.onForceLoad();
        cancelLoad();
	//新建task并执行
        mTask = new LoadTask();
        if (DEBUG) Log.v(TAG, "Preparing load: mTask=" + mTask);
        executePendingTask();
    }

    @Override
    protected boolean onCancelLoad() {
        if (DEBUG) Log.v(TAG, "onCancelLoad: mTask=" + mTask);
        if (mTask != null) {
            if (mCancellingTask != null) {
                ...
                    mHandler.removeCallbacks(mTask);
                }
                mTask = null;
                return false;
            } else if (mTask.waiting) {
           ...
            } else {
                boolean cancelled = mTask.cancel(false);
                if (DEBUG) Log.v(TAG, "cancelLoad: cancelled=" + cancelled);
                if (cancelled) {
                    mCancellingTask = mTask;
                    cancelLoadInBackground();
                }
                mTask = null;
                return cancelled;
            }
        }
        return false;
    }

    public void onCanceled(D data) {
    }

//在线程池中执行或延迟执行
    void executePendingTask() {
        if (mCancellingTask == null && mTask != null) {
            if (mTask.waiting) {
                mTask.waiting = false;
                mHandler.removeCallbacks(mTask);
            }
            if (mUpdateThrottle > 0) {
                long now = SystemClock.uptimeMillis();
                if (now < (mLastLoadCompleteTime+mUpdateThrottle)) {
                    // Not yet time to do another load.
                    if (DEBUG) Log.v(TAG, "Waiting until "
                            + (mLastLoadCompleteTime+mUpdateThrottle)
                            + " to execute: " + mTask);
                    mTask.waiting = true;
                    mHandler.postAtTime(mTask, mLastLoadCompleteTime+mUpdateThrottle);
                    return;
                }
            }
            if (DEBUG) Log.v(TAG, "Executing: " + mTask);
            mTask.executeOnExecutor(mExecutor, (Void[]) null);
        }
    }

    void dispatchOnCancelled(LoadTask task, D data) {
        onCanceled(data);
        if (mCancellingTask == task) {
            if (DEBUG) Log.v(TAG, "Cancelled task is now canceled!");
            rollbackContentChanged();
            mLastLoadCompleteTime = SystemClock.uptimeMillis();
            mCancellingTask = null;
            if (DEBUG) Log.v(TAG, "Delivering cancellation");
            deliverCancellation();
            executePendingTask();
        }
    }

    void dispatchOnLoadComplete(LoadTask task, D data) {
        if (mTask != task) {
            if (DEBUG) Log.v(TAG, "Load complete of old task, trying to cancel");
            dispatchOnCancelled(task, data);
        } else {
            if (isAbandoned()) {
                // This cursor has been abandoned; just cancel the new data.
                onCanceled(data);
            } else {
                commitContentChanged();
                mLastLoadCompleteTime = SystemClock.uptimeMillis();
                mTask = null;
                if (DEBUG) Log.v(TAG, "Delivering result");
                deliverResult(data);
            }
        }
    }

     */
    public abstract D loadInBackground();

    protected D onLoadInBackground() {
        return loadInBackground();
    }
    public void cancelLoadInBackground() {
    }

    public boolean isLoadInBackgroundCanceled() {
        return mCancellingTask != null;
    }
}
  ```
  ###CursorLoader
```
public class CursorLoader extends AsyncTaskLoader<Cursor> {
    final ForceLoadContentObserver mObserver;

    Uri mUri;
    String[] mProjection;
    String mSelection;
    String[] mSelectionArgs;
    String mSortOrder;

    Cursor mCursor;
    CancellationSignal mCancellationSignal;

    //根据全局变量加载数据
    @Override
    public Cursor loadInBackground() {
        synchronized (this) {
            if (isLoadInBackgroundCanceled()) {
                throw new OperationCanceledException();
            }
            mCancellationSignal = new CancellationSignal();
        }
        try {
            Cursor cursor = getContext().getContentResolver().query(mUri, mProjection, mSelection,
                    mSelectionArgs, mSortOrder, mCancellationSignal);
            if (cursor != null) {
                try {
                    // Ensure the cursor window is filled.
                    cursor.getCount();
                    cursor.registerContentObserver(mObserver);
                } catch (RuntimeException ex) {
                    cursor.close();
                    throw ex;
                }
            }
            return cursor;
        } finally {
            synchronized (this) {
                mCancellationSignal = null;
            }
        }
    }

    @Override
    public void cancelLoadInBackground() {
        super.cancelLoadInBackground();

        synchronized (this) {
            if (mCancellationSignal != null) {
                mCancellationSignal.cancel();
            }
        }
    }

    //获得数据，更新ui
    @Override
    public void deliverResult(Cursor cursor) {
        if (isReset()) {
            // An async query came in while the loader is stopped
            if (cursor != null) {
                cursor.close();
            }
            return;
        }
        Cursor oldCursor = mCursor;
        mCursor = cursor;

        if (isStarted()) {
            super.deliverResult(cursor);
        }

        if (oldCursor != null && oldCursor != cursor && !oldCursor.isClosed()) {
            oldCursor.close();
        }
    }

    public CursorLoader(Context context) {
        super(context);
        mObserver = new ForceLoadContentObserver();
    }

    
    public CursorLoader(Context context, Uri uri, String[] projection, String selection,
            String[] selectionArgs, String sortOrder) {
        super(context);
        mObserver = new ForceLoadContentObserver();
        mUri = uri;
        mProjection = projection;
        mSelection = selection;
        mSelectionArgs = selectionArgs;
        mSortOrder = sortOrder;
    }

    @Override
    protected void onStartLoading() {
        if (mCursor != null) {
            deliverResult(mCursor);
        }
        if (takeContentChanged() || mCursor == null) {
            forceLoad();
        }
    }

    /**
     * Must be called from the UI thread
     */
    @Override
    protected void onStopLoading() {
        // Attempt to cancel the current load task if possible.
        cancelLoad();
    }

    @Override
    public void onCanceled(Cursor cursor) {
        if (cursor != null && !cursor.isClosed()) {
            cursor.close();
        }
    }
   //acivity ondestory时关闭游标
    @Override
    protected void onReset() {
        super.onReset();
        
        // Ensure the loader is stopped
        onStopLoading();

        if (mCursor != null && !mCursor.isClosed()) {
            mCursor.close();
        }
        mCursor = null;
    }
  //省略一系列set get方法
}
```
#总结：
loader框架代码写的真纠结。。。各种全局变量。。。google的注释我也是看醉了。。
1.acivity和fragment 管理loaderManager，用ArrayMap保存mAllLoaderManager，在其生命周期内对各个loaderManager进行管理，在onstart中start,在onstop中 stop,在onDestory中destroy等
2.activity配置改变时，activity销毁时会保存mAllLoaderManager,retain各个loaderManager，在activity重新创建时，在 onAttcach,oncreate，performStart中恢复
3.loaderManager对其负责的loader进行统一管理
	(1).对loader用loadinfo类封装，负责调用loader的startLoading,stopLoading,forceLoad等方法
	(2).提供initLoader,doStart,doStop,doDestory等方法给activity调用，提供  restartLoader(会重新创建loader并start去加载数据)等方法给用户调用
4.加载数据的流程 activity调用 loaderManager doStart,loaderManager调用loader startLoading,加载数据完毕，取消加载等回调也是一层层向上回调
5.由于activity在其生命周期内对loader进行管控，所以在loader使用过程中，用户无需手动释放资源,由activity帮忙管理。如对于CursorLoader，对于cursor，无需手动关闭
6.对于CursorLoader,数据源变换时会通过contentObserver调用onContentChanged,最终调用forceLoad重新获取数据进行通知。
