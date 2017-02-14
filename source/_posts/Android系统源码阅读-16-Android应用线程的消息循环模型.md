title: Android系统源码阅读(16):Android应用线程的消息循环模型
date: 2017-02-14 14:53:14
tags: [Android,技术]
---

> 读书不宜拖沓

## 0. 背景

Android应用的主线程为ActivityThread，在第（10）章已经讲过，它主要负责处理界面事件，所以开发者应该避免在主线程中处理耗时的任务。为了减轻主线程的负担，开发者应该启用多线程来处理耗时的任务。在Android中可以创建多种线程，有的线程可以有自己的消息循环，有的线程则可以向主线程发送消息来使得界面发生改变。

##1. 主线程的消息循环

应用程序的主线程创建过程如下：ActivityManagerService线程请求Zygote进程创建应用进程；Zygote通过fork来创建一个新进程，新进程将ActivityThread的main作为入口进入Looper循环；Looper会调用静态成员函数prepareMainLooper创建一个Looper对象，并且在该应用进程中，该方法只会调用一次，因为只有一个主线程。
<!--more-->

## 2. 子线程HandlerThread
Android应用中，创建子线程可以和java桌面应用一样，创建一个Thread子类，实现run函数，然后start即可。但是这种方式是没有消息循环的，是一种比较简单的方法。

另一种则是使用Android定制版的Thread--HandlerThread。HanlderThre	ad的实现也很简单，完全可以模仿它实现一个自己的带有消息循环的Thread，或者实现一个集成HandlerThread的子类。

*frameworks/base/core/java/android/os/HandlerThread.java* :
{% codeblock lang:java %}
/**
 * Handy class for starting a new thread that has a looper. The looper can then be
 * used to create handler classes. Note that start() must still be called.
 */
public class HandlerThread extends Thread {
    int mPriority;
    int mTid = -1;
    Looper mLooper;

    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }

    /**
     * Constructs a HandlerThread.
     * @param name
     * @param priority The priority to run the thread at. The value supplied must be from
     * {@link android.os.Process} and not from java.lang.Thread.
     */
    public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;
    }

    /**
     * Call back method that can be explicitly overridden if needed to execute some
     * setup before Looper loops.
     */
    protected void onLooperPrepared() {
    }
    
    @Override
    public void run() {
        mTid = Process.myTid();
        //准备了looper
        Looper.prepare();
        synchronized (this) {
            //获得自己的looper
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        //进入消息循环
        Looper.loop();
        mTid = -1;
    }

    /**
     * This method returns the Looper associated with this thread. If this thread not been started
     * or for any reason is isAlive() returns false, this method will return null. If this thread
     * has been started, this method will block until the looper has been initialized.
     * @return The looper.
     */
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }

        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }

    /**
     * Quits the handler thread's looper.
     * <p>
     * Causes the handler thread's looper to terminate without processing any
     * more messages in the message queue.
     * </p><p>
     * Any attempt to post messages to the queue after the looper is asked to quit will fail.
     * For example, the {@link Handler#sendMessage(Message)} method will return false.
     * </p><p class="note">
     * Using this method may be unsafe because some messages may not be delivered
     * before the looper terminates.  Consider using {@link #quitSafely} instead to ensure
     * that all pending work is completed in an orderly manner.
     * </p>
     *
     * @return True if the looper looper has been asked to quit or false if the
     * thread had not yet started running.
     *
     * @see #quitSafely
     */
    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }

    /**
     * Quits the handler thread's looper safely.
     * <p>
     * Causes the handler thread's looper to terminate as soon as all remaining messages
     * in the message queue that are already due to be delivered have been handled.
     * Pending delayed messages with due times in the future will not be delivered.
     * </p><p>
     * Any attempt to post messages to the queue after the looper is asked to quit will fail.
     * For example, the {@link Handler#sendMessage(Message)} method will return false.
     * </p><p>
     * If the thread has not been started or has finished (that is if
     * {@link #getLooper} returns null), then false is returned.
     * Otherwise the looper is asked to quit and true is returned.
     * </p>
     *
     * @return True if the looper looper has been asked to quit or false if the
     * thread had not yet started running.
     */
    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }

    /**
     * Returns the identifier of this thread. See Process.myTid().
     */
    public int getThreadId() {
        return mTid;
    }
}
{% endcodeblock %}   

从HandlerThread的源码可以看出它提供了基本的开关功能，使用起来也很简单方便。具体如何使用如下：

首先实例化一个HandlerThread对象，然后启动这个thread。
{% codeblock lang:java %}
HandlerThread handleThread = new HandlerThread("Handler Thread");
//然后启动它，因为HandlerThread是Thread的子类，所以启动方法并无差异
handleThread.start();
{% endcodeblock %}

然后创建一个实现了Runnable接口的类。

{% codeblock lang:java %}
public class ThreadTask implements Runnable{
	public ThreadTask(){
	}
	//重写该方法
	public void run(){
	   //你的任务
	}
}
{% endcodeblock %}

创建一个Runnable的实例对象，然后丢给HandleThread来处理。

{% codeblock lang:java %}
ThreadTask threadTask = new ThreadTask();
//为HandlerThread的Looper创建一个Handler
Handler handler = new Handler(handlerThread.getLooper);
//通过Handler将任务给HandlerThread来执行
handler.post(threadTask);
{% endcodeblock %}

在消息队列处理到该任务时，threadTask的run函数就会被调用执行。

## 3. 异步任务AsyncTask

一个AsyncTask的例子：
{% codeblock lang:java %}
    private class MyTask extends AsyncTask<String, Integer, String> {
        //这一步是在主线程中被调用的  
        //onPreExecute方法用于在执行后台任务前做一些UI操作  
        @Override  
        protected void onPreExecute() {  
            Log.i(TAG, "onPreExecute() called");  
            textView.setText("loading...");  
        }  
  
        //这一步是异步线程的主要过程
        //doInBackground方法内部执行后台任务,不可在此方法内修改UI  
        @Override  
        protected String doInBackground(String... params) {  
            Log.i(TAG, "doInBackground(Params... params) called");  
            //...
            return new String(baos.toByteArray(), "gb2312");  
        }  
        
        //这一步是在主线程中调用
        //onProgressUpdate方法用于更新进度信息  
        @Override  
        protected void onProgressUpdate(Integer... progresses) {  
            Log.i(TAG, "onProgressUpdate(Progress... progresses) called");  
            progressBar.setProgress(progresses[0]);  
            textView.setText("loading..." + progresses[0] + "%");  
        }  
        
        //主线程调用
        //onPostExecute方法用于在执行完后台任务后更新UI,显示结果  
        @Override  
        protected void onPostExecute(String result) {  
            Log.i(TAG, "onPostExecute(Result result) called");  
            textView.setText(result);  
  
            execute.setEnabled(true);  
            cancel.setEnabled(false);  
        }  
  
        //主线程调用
        //onCancelled方法用于在取消执行中的任务时更改UI  
        @Override  
        protected void onCancelled() {  
            Log.i(TAG, "onCancelled() called");  
            textView.setText("cancelled");  
            progressBar.setProgress(0);  
  
            execute.setEnabled(true);  
            cancel.setEnabled(false);  
        }  
    }  
{% endcodeblock %}

子线程如果想向主线程发送消息，则需要通过主线程的Handler。这里AsyncTask运行于子线程，但是无需Handler也可以向主线程发送消息。AsyncTask类的具体实现如下。

*frameworks/base/core/java/android/os/AsyncTask.java*  :
{% codeblock lang:java %}
public abstract class AsyncTask<Params, Progress, Result> {
    //...
    //负责创建线程，放入sPoolWorkQueue中
    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };

    //任务队列
    //如果队列已空，则试图获取任务的线程则会阻塞
    //如果队列已满，试图向其中加入任务的线程也会阻塞
    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);

    /**
     * An {@link Executor} that can be used to execute tasks in parallel.
     * 一个线程池，用来执行sPoolWorkQueue中的任务，下面看其如何执行线程
     * 这里设置了核心线程数，最大线程数等参数
     */
    public static final Executor THREAD_POOL_EXECUTOR
            = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);

    /**
     * An {@link Executor} that executes tasks one at a time in serial
     * order.  This serialization is global to a particular process.
     * 该线程执行器是顺序执行
     */
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

    private static final int MESSAGE_POST_RESULT = 0x1;
    private static final int MESSAGE_POST_PROGRESS = 0x2;

    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
    //创建当前线程的一个handler，如主线程
    private static InternalHandler sHandler;
    
    private final WorkerRunnable<Params, Result> mWorker;
    private final FutureTask<Result> mFuture;
    //...
}
{% endcodeblock %}

sThreadFactory，sPoolWorkQueue和THREAD_POOL_EXECUTOR都是全局静态变量。在一个App进程中，因此，所有使用AsyncTask的异步线程使用的是同一个线程池，这样可以避免创建多个线程池占用资源。

ThreadPoolExecutor在执行任务时，会根据实际情况选择是否创建线程，具体如下：   
1. 线程池中的线程数量小于核心线程数，则会创建新的线程来执行新添加的任务。
2. 如果大于核心线程数，小于最大线程数，则
	- sPoolWorkQueue未满，则将新任务保存在队列中
	- 如果已满，则创建一个新的线程来执行新的任务
3. 如果线程池中线程数已经大于/等于最大线程数，如果队列不满，则放入队列；如果已满，则拒绝新任务。


InternalHandler类的实例对象sHandler是创建AsyncTask的线程的Handler，这里假设是主线程。

*frameworks/base/core/java/android/os/AsyncTask.java* :
{% codeblock lang:java %}
private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            //这个函数是运行在主线程中
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    //异步任务执行结束后发送结果，这里处理这个结果
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    //异步任务实行过程中，会使用publishProgress来发布中间结果
                    //这一步就是主线程处理该中间结果
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
{% endcodeblock %}

其中result是一个AsyncTaskResult，里面保存了结果信息和其对应的AsyncTask。

*frameworks/base/core/java/android/os/AsyncTask.java* :
{% codeblock lang:java %}
    @SuppressWarnings({"RawUseOfParameterizedType"})
    private static class AsyncTaskResult<Data> {
        final AsyncTask mTask;
        final Data[] mData;

        AsyncTaskResult(AsyncTask task, Data... data) {
            mTask = task;
            mData = data;
        }
    }
{% endcodeblock %}

WorkerRunnable类的实例mWorker实现如下，保存了异步任务的输入数据：

*frameworks/base/core/java/android/os/AsyncTask.java* :
{% codeblock lang:java %}
    private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
    }
{% endcodeblock %}

了解了AsyncTask的成员变量后，下面看一个异步任务创建的完整过程。

### 3.1 AsyncTask的构造函数

*frameworks/base/core/java/android/os/AsyncTask.java* :
{% codeblock lang:java %}
    /**
     * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
     * 看来AsyncTask只能在主线程中创建啊！
     */
    public AsyncTask() {
        //这里实现了一个WorkerRunnable的匿名类，同时实例化了一个对象mWorker
        //这里面封装了所要执行的任务
        mWorker = new WorkerRunnable<Params, Result>() {
            //异步线程里执行的任务
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                //这里有用户定义的所要执行的任务
                Result result = doInBackground(mParams);
                Binder.flushPendingCommands();
                return postResult(result);
            }
        };
        
        //又将mWorker封装如FutureTask，同样封装了要执行的任务
        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }
{% endcodeblock %} 

### 3.2 AsyncTask的执行

在主线程中运行AsyncTask时，需要调用execute来执行该任务，该方法又会调用如下函数来执行任务。这里需要注意的是execute函数是运行在主线程中的，异步任务还没开始。

*frameworks/base/core/java/android/os/AsyncTask.java* :
{% codeblock lang:java %}
  @MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        //...
        mStatus = Status.RUNNING;
        //在主线程里运行执行前的任务
        onPreExecute();

        mWorker.mParams = params;
        //exec是一个SerialExecute
        exec.execute(mFuture);

        return this;
    }
{% endcodeblock %}

SerialExecutor执行器的执行过程。
*frameworks/base/core/java/android/os/AsyncTask.java* :
{% codeblock lang:java %}
private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            //把mFuture封装入Runnable，然后放在队列mTasks中
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        //在结束该任务之前，把队列的下一个任务放入线程池
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                //开始执行队列中的第一个任务
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                //第一个任务放入线程池执行
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
{% endcodeblock %}
到这里已经将任务交给线程池来启动，用户定义的doInBackground已经开始运行。

### 3.3 异步线程发送消息

异步线程可以通过publishProgress发送消息，具体实现如下：
{% codeblock lang:java %}
@WorkerThread
    protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            //获得主线程的handler--sHandler，通过它向主线程发送消息
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }
{% endcodeblock %}

主线程通过sHandler的handleMessage函数来处理该消息，然后调用用户自己实现的onProgressUpdate函数来实现UI的更新。


### 3.4 异步任务结束
在异步线程执行完任务后，会调用FutureTask的done函数来处理结束事件。这里主要处理的就是异步任务处理完成后，最后的返回值。最后的结果通过如下函数发送给主线程。

{% codeblock lang:java %}
 private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
{% endcodeblock %}
前面已经讲过，主线程会调用handler里的result.mTask.finish(result.mData[0])来处理结果。finish函数实现如下：

{% codeblock lang:java %}
private void finish(Result result) {
      if (isCancelled()) {
            //这些都是用户可以重写的
            //来实现自己的功能
            onCancelled(result);
      } else {
            onPostExecute(result);
      }
      mStatus = Status.FINISHED;
}
{% endcodeblock %}

## 4. 总结

这里主要介绍了android应用里的几种线程的运行机制。

普通java线程Thread是最简单的一种方法，但是比较难管理，与主线程通信主要靠handler。而且，该中线程只能向主线程发消息，而主线程无法向子线程发消息。

HandlerThread则是可以有自己的looper和消息队列，将任务发送入消息队列来实现异步任务的处理，同时还方便管理。主线程和子线程可以通过对方的Handler相互发送消息。

AsyncTask是比较适用于Android应用的一种异步任务实现方式。虽然归根结底还是通过handler来向主进程发消息，但是整个异步任务执行过程的划分和管理更为科学，屏蔽了内部实现的复杂，为用户提供了更为简单实用的接口。















