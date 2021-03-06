title: java并发编程基础-ReentrantLock及LinkedBlockingQueue源码分析
date: 2015-02-12 23:57:48
categories: java
tags:  ReentrantLock
---
ReentrantLock是一个较为常用的锁对象。在上次分析的uil开源项目中也多次被用到，下面谈谈其概念和基本使用。
#概念
一个可重入的互斥锁定 Lock，它具有与使用 synchronized 相同的一些基本行为和语义，但功能更强大。
##名词解释：
###互斥
表示同一时刻，多个线程中，只能有一个线程能获得该锁。但是多个线程都可以调用lock方法，只有一个会成功，其他的线程会被阻塞，直到该锁被释放
###可重入
模仿synchronized 的语义；如果线程进入由线程已经拥有的监控器保护的 synchronized 块，就允许线程继续进行，当线程退出第二个（或者后续）synchronized 块的时候，不释放锁，只有线程退出它进入的监控器保护的第一个 synchronized 块时，才释放锁。 
对于ReentrantLock，每次获得锁，并将请求计数置为一，如果同一个线程再次lock，计数器将递增，每次unlock时计数器值递减，直到计数器为0，锁释放
<!--more-->
##lock方法过程
如果该锁没有被另一个线程保持，则lock时获取该锁定并立即返回，将锁定的保持计数设置为 1。
如果当前线程已经保持该锁定，则将保持计数加 1，并且该方法立即返回。
如果该锁定被另一个线程保持，则出于线程调度的目的，禁用当前线程，并且在获得锁定之前，该线程将一直处于休眠状态，此时锁定保持计数被设置为 1。
##unLock方法过程
每次unlock时计数器值递减，直到计数器为0，释放锁
##Condition类
该类与lock绑定，用newCondition()方法创建，提供了线程之间通信的方式（类似信号量）。其使用基本与object类的wait,notify,notifyAll相同。
用condition.await()替换Object,wait()，调用时该线程阻塞，释放该线程的锁。
用condition.signal()替换Object.notify()，用condition.signalAll()替换Object.notifyAll()，唤醒该condition await方法所阻塞的线程 

#相对synchronized优势
锁投票（我也不是特别理解，可以通过投票获取锁？）
定时锁等候
中断锁等候
     线程A和B都要获取对象O的锁定，假设A获取了对象O锁，B将等待A释放对O的锁定，
     如果使用 synchronized ，如果A不释放，B将一直等下去，不能被中断
     如果 使用ReentrantLock，如果A不释放，可以使B在等待了足够长的时间以后，中断等待，而干别的事情

#使用
以下以linkedBlokingQueue源码为例子，来学习其使用。
```
    public class LinkedBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E>, java.io.Serializable { 
        //链表节点node类结构   
        static class Node<E> {  
            volatile E item;//volatile，保证了数据的可见性   
            Node<E> next;  
            Node(E x) { item = x; }  
        }  
        //容量
        private final int capacity;  
        //用原子变量,当前元素个数  
        private final AtomicInteger count = new AtomicInteger(0);  
        //头节点
        private transient Node<E> head;  
        //表尾节点 
        private transient Node<E> last;  
        //获取元素或删除元素时,要加的takeLock锁  
        private final ReentrantLock takeLock = new ReentrantLock();  
        //获取元素时若队列为空，线程阻塞，直至notEmpty条件满足（被通知） 
        private final Condition notEmpty = takeLock.newCondition();  
        //插入元素时 要加putLock锁  
        private final ReentrantLock putLock = new ReentrantLock();  
        //插入时，若队列已满，线程阻塞，直至notFull条件满足（被通知）
        private final Condition notFull = putLock.newCondition();  
        // 唤醒等待的take操作,插入数据时若插入前链表中无数据，则调用，表示链表不再为空
        private void signalNotEmpty() {  
            final ReentrantLock takeLock = this.takeLock;  
            takeLock.lock();  
            try {  
                notEmpty.signal();  
            } finally {  
                takeLock.unlock();  
            }  
        }  
        //唤醒等待插入操作，移除数据时若链表原先已满则调用，表示链表不再满 
        private void signalNotFull() {  
            final ReentrantLock putLock = this.putLock;  
            putLock.lock();  
            try {  
                notFull.signal();  
            } finally {  
                putLock.unlock();  
            }  
        }  
        // 插入到链表尾部 
        private void insert(E x) {  
            last = last.next = new Node<E>(x);  
        }  
        //获取并移除头元素 
        private E extract() {  
            Node<E> first = head.next;  
            head = first;  
            E x = first.item;  
            first.item = null;  
            return x;  
        }  
        //锁住两把锁，在remove，clear等方法中调用   
        private void fullyLock() {  
            putLock.lock();  
            takeLock.lock();  
        }  
        //和fullyLock成对使用 
        private void fullyUnlock() {  
            takeLock.unlock();  
            putLock.unlock();  
        }  
        //默认构造，容量为 Integer.MAX_VALUE  
   
        public LinkedBlockingQueue() {  
            this(Integer.MAX_VALUE);  
        }  
        //指定容量的构造   
        public LinkedBlockingQueue(int capacity) {  
            if (capacity <= 0) throw new IllegalArgumentException();  
            this.capacity = capacity;  
            last = head = new Node<E>(null);  
        }  
        //指定初始化集合的构造   
        public LinkedBlockingQueue(Collection<? extends E> c) {  
            this(Integer.MAX_VALUE);  
            for (E e : c)  
                add(e);  
        }  
        //获得大小 
          
        public int size() {  
            return count.get();  
        }  
        //剩余容量  
        public int remainingCapacity() {  
            return capacity - count.get();  
        }  

        // 将指定元素插入到此队列的尾部，如已满，阻塞至队列中有元素被移除 
        public void put(E e) throws InterruptedException {  
            if (e == null) throw new NullPointerException();  
            int c = -1;  
            final ReentrantLock putLock = this.putLock;  
            final AtomicInteger count = this.count;
	    //加put锁，多个线程不能同时进入  
            putLock.lockInterruptibly();  
            try {  
                try {  
		    //容量已满，则一直阻塞
                    while (count.get() == capacity)  
                        notFull.await();  
                } catch (InterruptedException ie) {  
                    notFull.signal(); // propagate to a non-interrupted thread  
                    throw ie;  
                }  
		//插入
                insert(e);  
                c = count.getAndIncrement();
		//通知链表未满  
                if (c + 1 < capacity)  
                    notFull.signal();  
            } finally {  
		//解锁，注意必须在finally里调用，反正各种异常导致没有unlock使线程死锁
                putLock.unlock();  
            }  
	     //通知链表非空
            if (c == 0)  
                signalNotEmpty();  
        }  
        // 将指定元素插入到此队列的尾部，如有必要，则等待一定时间以使空间变得可用。 
         
        public boolean offer(E e, long timeout, TimeUnit unit)  
            throws InterruptedException {  
            if (e == null) throw new NullPointerException();  
            long nanos = unit.toNanos(timeout);  
            int c = -1;  
            final ReentrantLock putLock = this.putLock;  
            final AtomicInteger count = this.count;  
	    //加锁
            putLock.lockInterruptibly();  
            try {  
                for (;;) {
                    //未满可插入  
                    if (count.get() < capacity) {  
                        insert(e);  
                        c = count.getAndIncrement();
			//通知未满  
                        if (c + 1 < capacity)  
                            notFull.signal();
			//跳出循环  
                        break;  
                    }  
		   //队列已满，未能插入，等待时间是负的，直接返回
                    if (nanos <= 0)  
                        return false;  
                    try {  
		    //等待一定时间后再次尝试
                        nanos = notFull.awaitNanos(nanos);  
                    } catch (InterruptedException ie) {  
                        notFull.signal(); // propagate to a non-interrupted thread  
                        throw ie;  
                    }  
                }  
            } finally {  
		//解锁
                putLock.unlock();  
            }  
		//通知已插入数据，链表非空
            if (c == 0)  
                signalNotEmpty();  
            return true;  
        }  
        //将指定元素插入到此队列的尾部（如果立即可行且不会超出此队列的容量）， 
         在成功时返回 true，如果此队列已满，则返回 false。 
          
        public boolean offer(E e) {  
            if (e == null) throw new NullPointerException();  
            final AtomicInteger count = this.count;  
            if (count.get() == capacity)  
                return false;  
            int c = -1;  
            final ReentrantLock putLock = this.putLock;  
            putLock.lock();  
            try {  
		//由于可能在lock被阻塞时其他线程进行了插入操作，需再次判断count
                if (count.get() < capacity) {  
                    insert(e);  
                    c = count.getAndIncrement();
		    //通知未满  
                    if (c + 1 < capacity)  
                        notFull.signal();  
                }  
            } finally {  
                putLock.unlock();  
            }  
	   //通知非空
            if (c == 0)  
                signalNotEmpty();
            // >0表示已成功插入  
            return c >= 0;  
        }  
        //获取并移除此队列的头部，若队列为空，则阻塞。  
        public E take() throws InterruptedException {  
            E x;  
            int c = -1;  
            final AtomicInteger count = this.count;  
            final ReentrantLock takeLock = this.takeLock;
	    //加锁 
            takeLock.lockInterruptibly();  
            try {  
                try {
		   //队列为空时阻塞 
                    while (count.get() == 0)  
                        notEmpty.await();  
                } catch (InterruptedException ie) {  
                    notEmpty.signal(); // propagate to a non-interrupted thread  
                    throw ie;  
                }  
		//获取数据
                x = extract();  
                c = count.getAndDecrement();
		//通知非空  
                if (c > 1)  
                    notEmpty.signal();  
            } finally {  
                takeLock.unlock();  
            }  
	     //通知未满
            if (c == capacity)  
                signalNotFull();  
            return x;  
        }  
          
        //与offer方法结构基本一致，若队列为空，则阻塞一段时间，一段时间后仍为空，则返回null
        public E poll(long timeout, TimeUnit unit) throws InterruptedException {  
            E x = null;  
            int c = -1;  
            long nanos = unit.toNanos(timeout);  
            final AtomicInteger count = this.count;  
            final ReentrantLock takeLock = this.takeLock;  
            takeLock.lockInterruptibly();  
            try {  
                for (;;) {  
                    if (count.get() > 0) {  
                        x = extract();  
                        c = count.getAndDecrement();  
                        if (c > 1)  
                            notEmpty.signal();  
                        break;  
                    }  
                    if (nanos <= 0)  
                        return null;  
                    try {  
                        nanos = notEmpty.awaitNanos(nanos);  
                    } catch (InterruptedException ie) {  
                        notEmpty.signal(); // propagate to a non-interrupted thread  
                        throw ie;  
                    }  
                }  
            } finally {  
                takeLock.unlock();  
            }  
            if (c == capacity)  
                signalNotFull();  
            return x;  
        }  
          
        ////与offer方法结构基本一致 队列为空，不阻塞，直接返回null
        public E poll() {  
            final AtomicInteger count = this.count;  
            if (count.get() == 0)  
                return null;  
            E x = null;  
            int c = -1;  
            final ReentrantLock takeLock = this.takeLock;  
            takeLock.lock();  
            try {  
                if (count.get() > 0) {  
                    x = extract();  
                    c = count.getAndDecrement();  
                    if (c > 1)  
                        notEmpty.signal();  
                }  
            } finally {  
                takeLock.unlock();  
            }  
            if (c == capacity)  
                signalNotFull();  
            return x;  
        }  
        //获取但不移除此队列的头；如果此队列为空，则返回 null。  
        public E peek() {  
            if (count.get() == 0)  
                return null;  
            final ReentrantLock takeLock = this.takeLock;  
            takeLock.lock();  
            try {  
                Node<E> first = head.next;  
                if (first == null)  
                    return null;  
                else  
                    return first.item;  
            } finally {  
                takeLock.unlock();  
            }  
        }  
        /** 
         * 从此队列移除指定元素的单个实例（如果存在）。 
         */  
        public boolean remove(Object o) {  
            if (o == null) return false;  
            boolean removed = false;
	    //同时加锁，此时其他线程不能插入，不能移除
            fullyLock();  
            try {  
                Node<E> trail = head;  
                Node<E> p = head.next;
		//遍历，获取到该元素  
                while (p != null) {  
                    if (o.equals(p.item)) {  
                        removed = true;  
                        break;  
                    }  
                    trail = p;  
                    p = p.next;  
                }  
		//删除该元素
                if (removed) {  
                    p.item = null;  
                    trail.next = p.next;  
                    if (last == p)  
                        last = trail;  
                    if (count.getAndDecrement() == capacity)  
                        notFull.signalAll();  
                }  
            } finally {  
                fullyUnlock();  
            }  
            return removed;  
        }  
        ……  
    } 
```

