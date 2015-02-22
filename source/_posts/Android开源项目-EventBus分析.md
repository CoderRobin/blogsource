title: 开源项目-EventBus分析
date: 2015-02-13 23:18:48
categories: Android
tags: EventBus
---
##简介
EventBus 是一个事件发布/订阅框架，通过解耦发布者和订阅者简化 Android 事件传递，事件传递既可用于 Android 四大组件间通讯，也可以用户异步线程和主线程间通讯等等。
传统的事件传递方式包括：Handler、BroadCastReceiver、回调等，相比之下 EventBus 的优点是代码简洁，使用简单，并将事件发布和订阅充分解耦。尤其在代码分层比较多时，需要层次往上传递，写很多回调事件。这时候就可以使用eventbus取代之。 
<!--more-->
关于EventBus分析的理由基本跟上一次分析uil的理由一致，主要还是让自己加深理解,学习框架的基本思路。

很多大神都已经解析过此开源项目。
包括但不限于以下博客：
http://www.codekk.com/open-source-project-analysis/detail/Android/Trinea/EventBus%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90

##使用
1.注册
EventBus.getDefault().register(classA);  
2.自定义事件类,比如MyEvent
3.在ClassA中写回调
在EventBus中可以定义四种类型的回调函数，形参为自定义事件类型。
EventBus根据事件类型确定回调函数的形参即事件类型决定调用哪个方法
	a、onEvent   回调函数和发起事件的函数会在同一个线程中执行
	b、onEventMainThread 回调函数会在主线程中执行
	c、onEventBackgroundThread，当使用这种类型时，如果事件发起函数在主线程中执行，那么回调函数另启动一个子线程，如果事件发起函数在子线程执行，那么回调函数就在这个子线程执行。
	d、onEventAsync，当使用这种类型时，不管事件发起函数在哪里执行，都会另起一个线程去执行回调。
4.传递事件
   EventBus.getDefault().post(new MyEvent())
可以看出其使用非常简单，用户无需事件是怎样传递过来的实现细节。

##源码分析
eventbus源码相对简单，这里只看下主要的几个类学习下
###eventbus
```
public class EventBus {
    static ExecutorService executorService = Executors.newCachedThreadPool();
    private static volatile EventBus defaultInstance;

    private static final String DEFAULT_METHOD_NAME = "onEvent";
    private static final Map<Class<?>, List<Class<?>>> eventTypesCache = new HashMap<Class<?>, List<Class<?>>>();

   //以eventType（订阅事件，即接收方法的第一个参数）为key,订阅类的列表构成的hashmap
    private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
  //一个订阅类拥有的订阅事件
    private final Map<Object, List<Class<?>>> typesBySubscriber;
    private final Map<Class<?>, Object> stickyEvents;

    private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
        @Override
        protected PostingThreadState initialValue() {
            return new PostingThreadState();
        }
    };

    public EventBus() {
        subscriptionsByEventType = new HashMap<Class<?>, CopyOnWriteArrayList<Subscription>>();
        typesBySubscriber = new HashMap<Object, List<Class<?>>>();
        stickyEvents = new ConcurrentHashMap<Class<?>, Object>();
        mainThreadPoster = new HandlerPoster(this, Looper.getMainLooper(), 10);
        backgroundPoster = new BackgroundPoster(this);
        asyncPoster = new AsyncPoster(this);
        subscriberMethodFinder = new SubscriberMethodFinder();
    }

...


    private synchronized void register(Object subscriber, String methodName, boolean sticky, int priority) {
	//由订阅类使用反射查找订阅的方法
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriber.getClass(),
                methodName);
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
	//执行订阅操作
            subscribe(subscriber, subscriberMethod, sticky, priority);
        }
    }


  

    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod, boolean sticky, int priority) {
        subscribed = true;
        Class<?> eventType = subscriberMethod.eventType;
	//取出该eventtype所对应的订阅者列表
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod, priority);
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<Subscription>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
	//判断是否已订阅过
            for (Subscription subscription : subscriptions) {
                if (subscription.equals(newSubscription)) {
                    throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                            + eventType);
                }
            }
        }

        // Starting with EventBus 2.2 we enforced methods to be public (might change with annotations again)
        // subscriberMethod.method.setAccessible(true);

        int size = subscriptions.size();
	//根据优先级将订阅者加入订阅列表
        for (int i = 0; i <= size; i++) {
            if (i == size || newSubscription.priority > subscriptions.get(i).priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }

	
        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<Class<?>>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
	//添加到1个订阅者拥有的事件列表
        subscribedEvents.add(eventType);

        if (sticky) {
            Object stickyEvent;
            synchronized (stickyEvents) {
                stickyEvent = stickyEvents.get(eventType);
            }
            if (stickyEvent != null) {
                // If the subscriber is trying to abort the event, it will fail (event is not tracked in posting state)
                // --> Strange corner case, which we don't take care of here.
                postToSubscription(newSubscription, stickyEvent, Looper.getMainLooper() == Looper.myLooper());
            }
        }
    }

    /** Posts the given event to the event bus. */
    public void post(Object event) {

        PostingThreadState postingState = currentPostingThreadState.get();
        List<Object> eventQueue = postingState.eventQueue;
	//加入当前线程的事件队列
        eventQueue.add(event);

        if (postingState.isPosting) {
            return;
        } else {
	 //应该没必要每次都判断
            postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                while (!eventQueue.isEmpty()) {
		//post event
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }



    private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<? extends Object> eventClass = event.getClass();
	//由event获取事件类型，包括其父类
        List<Class<?>> eventTypes = findEventTypes(eventClass);
        boolean subscriptionFound = false;
        int countTypes = eventTypes.size();
        for (int h = 0; h < countTypes; h++) {
            Class<?> clazz = eventTypes.get(h);
            CopyOnWriteArrayList<Subscription> subscriptions;
            synchronized (this) {
  //由事件类型获取订阅者
                subscriptions = subscriptionsByEventType.get(clazz);
            }
	 
            if (subscriptions != null && !subscriptions.isEmpty()) {
                for (Subscription subscription : subscriptions) {
                    postingState.event = event;
                    postingState.subscription = subscription;
                    boolean aborted = false;
                    try {
		//post 事件到订阅者
                        postToSubscription(subscription, event, postingState.isMainThread);
                        aborted = postingState.canceled;
                    } finally {
                        postingState.event = null;
                        postingState.subscription = null;
                        postingState.canceled = false;
                    }
                    if (aborted) {
                        break;
                    }
                }
                subscriptionFound = true;
            }
        }
        if (!subscriptionFound) {
            Log.d(TAG, "No subscribers registered for event " + eventClass);
            if (eventClass != NoSubscriberEvent.class && eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));
            }
        }
    }

    private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
        case PostThread:
	  //post到当前线程
            invokeSubscriber(subscription, event);
            break;
	//post到主线程
        case MainThread:
		//判断当前是否是主线程
            if (isMainThread) {
                invokeSubscriber(subscription, event);
            } else {
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case BackgroundThread:
            if (isMainThread) {
                backgroundPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        case Async:
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }

    private List<Class<?>> findEventTypes(Class<?> eventClass) {
        synchronized (eventTypesCache) {
            List<Class<?>> eventTypes = eventTypesCache.get(eventClass);
            if (eventTypes == null) {
                eventTypes = new ArrayList<Class<?>>();
                Class<?> clazz = eventClass;
                while (clazz != null) {
                    eventTypes.add(clazz);
                    addInterfaces(eventTypes, clazz.getInterfaces());
                    clazz = clazz.getSuperclass();
                }
                eventTypesCache.put(eventClass, eventTypes);
            }
            return eventTypes;
        }
    }


    static void addInterfaces(List<Class<?>> eventTypes, Class<?>[] interfaces) {
        for (Class<?> interfaceClass : interfaces) {
            if (!eventTypes.contains(interfaceClass)) {
                eventTypes.add(interfaceClass);
                addInterfaces(eventTypes, interfaceClass.getInterfaces());
            }
        }
    }

   
    void invokeSubscriber(PendingPost pendingPost) {
        Object event = pendingPost.event;
        Subscription subscription = pendingPost.subscription;
        PendingPost.releasePendingPost(pendingPost);
        if (subscription.active) {
            invokeSubscriber(subscription, event);
        }
    }

    void invokeSubscriber(Subscription subscription, Object event) throws Error {
        try {
	  //用反射执行该方法
            subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
        } catch (InvocationTargetException e) {
            Throwable cause = e.getCause();
            if (event instanceof SubscriberExceptionEvent) {
                // Don't send another SubscriberExceptionEvent to avoid infinite event recursion, just log
                Log.e(TAG, "SubscriberExceptionEvent subscriber " + subscription.subscriber.getClass()
                        + " threw an exception", cause);
                SubscriberExceptionEvent exEvent = (SubscriberExceptionEvent) event;
                Log.e(TAG, "Initial event " + exEvent.causingEvent + " caused exception in "
                        + exEvent.causingSubscriber, exEvent.throwable);
            } else {
                if (logSubscriberExceptions) {
                    Log.e(TAG, "Could not dispatch event: " + event.getClass() + " to subscribing class "
                            + subscription.subscriber.getClass(), cause);
                }
                SubscriberExceptionEvent exEvent = new SubscriberExceptionEvent(this, cause, event,
                        subscription.subscriber);
                post(exEvent);
            }
        } catch (IllegalAccessException e) {
            throw new IllegalStateException("Unexpected exception", e);
        }
    }

    /** For ThreadLocal, much faster to set (and get multiple values). */
    final static class PostingThreadState {
        List<Object> eventQueue = new ArrayList<Object>();
        boolean isPosting;
        boolean isMainThread;
        Subscription subscription;
        Object event;
        boolean canceled;
    }

    // Just an idea: we could provide a callback to post() to be notified, an alternative would be events, of course...
    /* public */interface PostCallback {
        void onPostCompleted(List<SubscriberExceptionEvent> exceptionEvents);
    }

}

```
###Subscription
保存了订阅者和订阅方法，重写了equals方法,priority用于确定在list中的顺序
```
final class Subscription {
    final Object subscriber;
    final SubscriberMethod subscriberMethod;
    final int priority;
    /**
     * Becomes false as soon as {@link EventBus#unregister(Object)} is called, which is checked by queued event delivery
     * {@link EventBus#invokeSubscriber(PendingPost)} to prevent race conditions.
     */
    volatile boolean active;

    Subscription(Object subscriber, SubscriberMethod subscriberMethod, int priority) {
        this.subscriber = subscriber;
        this.subscriberMethod = subscriberMethod;
        this.priority = priority;
        active = true;
    }

    @Override
    public boolean equals(Object other) {
        if (other instanceof Subscription) {
            Subscription otherSubscription = (Subscription) other;
            return subscriber == otherSubscription.subscriber
                    && subscriberMethod.equals(otherSubscription.subscriberMethod);
        } else {
            return false;
        }
    }

    @Override
    public int hashCode() {
        return subscriber.hashCode() + subscriberMethod.methodString.hashCode();
    }
}
```
###HandlerPoster
```
final class HandlerPoster extends Handler {

    //待发送队列
    private final PendingPostQueue queue;
    private final int maxMillisInsideHandleMessage;
    private final EventBus eventBus;
    private boolean handlerActive;

    HandlerPoster(EventBus eventBus, Looper looper, int maxMillisInsideHandleMessage) {
        super(looper);
        this.eventBus = eventBus;
        this.maxMillisInsideHandleMessage = maxMillisInsideHandleMessage;
        queue = new PendingPostQueue();
    }

    void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
	   //放入队列
            queue.enqueue(pendingPost);
            if (!handlerActive) {
                handlerActive = true;
		//触发handleMessage方法，正常情况下都会返回false,looper退出时才会返回false
                if (!sendMessage(obtainMessage())) {
                    throw new EventBusException("Could not send handler message");
                }
            }
        }
    }

    @Override
    public void handleMessage(Message msg) {
        boolean rescheduled = false;
        try {
            long started = SystemClock.uptimeMillis();
            while (true) {
		//队列中取出
                PendingPost pendingPost = queue.poll();
                if (pendingPost == null) {
                    synchronized (this) {
                        // Check again, this time in synchronized
                        pendingPost = queue.poll();
                        if (pendingPost == null) {
                            handlerActive = false;
                            return;
                        }
                    }
                }
		//触发
                eventBus.invokeSubscriber(pendingPost);
                long timeInMethod = SystemClock.uptimeMillis() - started;

                if (timeInMethod >= maxMillisInsideHandleMessage) {
                    if (!sendMessage(obtainMessage())) {
                        throw new EventBusException("Could not send handler message");
                    }
                    rescheduled = true;
                    return;
                }
            }
        } finally {
            handlerActive = rescheduled;
        }
    }
}

```
###其它
AsyncPoster,BackgroundPoster都是runnable对象，调用线程池执行自己，与AsyncPoster中的任务并发执行，BackgroundPoster中的任务只在同一个线程中依次执行，而不是并发执行
##总结
1.学习eventbus可以对hashmap,CopyOnWriteArrayList,handler等的使用有所了解。
2.需要注意结束时需要unRegister，否则会导致内存泄漏。
