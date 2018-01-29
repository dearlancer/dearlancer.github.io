---
layout: post
title: 源码分析之AsyncTask源码
excerpt: "AsyncTask是android中最常使用的一个线程类，它将UI线程和子线程区分开来，我们可以在doInbackground中执行耗时操作，并且在onProgressUpdate和onPostExecute中执行UI操作，它很好地帮我们封装了线程切换的逻辑，使得我们可以方便地使用。"
categories:
- android
- 源码
comments: true
---

### 一个简单的栗子
```ruby
new AsyncTask<Void, Integer, Boolean>() {
            @Override
            protected Boolean doInBackground(Void... voids) {
              //在子线程中运行的代码
                return null;
            }

            @Override
            protected void onPreExecute() {
              //后台任务开始执行之间调用，用于进行一些界面上的初始化操作
                super.onPreExecute();
            }

            @Override
            protected void onPostExecute(Boolean aBoolean) {
              //后台任务执行完后调用，可以提示执行的结果
                super.onPostExecute(aBoolean);
            }

            @Override
            protected void onProgressUpdate(Integer... values) {
              //可以在这里更新UI进度的操作
                super.onProgressUpdate(values);
            }

            @Override
            protected void onCancelled(Boolean aBoolean) {
              //取消操作
                super.onCancelled(aBoolean);
            }

            @Override
            protected void onCancelled() {
                super.onCancelled();
            }
        }.execute();
```
### 源码剖析

1. 先从类的静态块开始看起
```ruby
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    private static final int KEEP_ALIVE_SECONDS = 30;

    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };

    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);

    public static final Executor THREAD_POOL_EXECUTOR;

    static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                sPoolWorkQueue, sThreadFactory);
        threadPoolExecutor.allowCoreThreadTimeOut(true);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }

    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
```
可以看到，AsyncTask一次可以执行的最大线程数为Math.max(2, Math.min(CPU_COUNT - 1, 4))个，队列中一次最多只能存储128个线程，并且构造了一个线程池来管理可能执行的任务，
这里有个细节，可以看静态块中的代码，THREAD_POOL_EXECUTOR是在新建实例并执行完allowCoreThreadTimeOut函数后才指向这个实例的地址的，这样的好处是保证了状态的初始化。

2. 然后是构造函数,所有的构造函数最终都是调用这个进行构造实例：
```ruby
public AsyncTask(@Nullable Looper callbackLooper) {
        mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);

        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    postResult(result);
                }
                return result;
            }
        };

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
```
整个构造流程主要分为三步：

* 获取handler
如果传入的looper为null或者为主线程，则把引用指向主线程的hanlder，否则则通过传入的looper构造一个hanlder

* 初始化mWorker
WorkerRunnable实际为一个Callback接口的包装类，在callback的基础上封装了可传入的参数

* 初始化mFuture
FutureTask实际是一个Runnable和Future接口的实现类，通过Future的包装使得Runnable在完成后可以返回一个结果

3. 接着是excute()函数
```ruby
 @MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
   @MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }
        mStatus = Status.RUNNING;
        onPreExecute();
        mWorker.mParams = params;
        exec.execute(mFuture);
        return this;
    }  
```

可以看到，execute函数是在必须在主线程中被调用的，这个函数实际调用的是executeOnExecutor函数，
在executeOnExecutor函数中，onPreExecute先被执行，然后是参数的赋值操作，最后是通过excutor来执行mFuture，
切换到子线程中进行工作，excutor的实现者如下
```ruby
private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```
可以看到，这里的SerialExecutor会执行mFutrue的run方法，而mFuture的通过mFuture = new FutureTask<Result>(mWorker)这里的代码包装了doInBackground函数，最后会执行这个函数，SerialExecutor是使用ArrayDeque这个队列来管理Runnable对象的，如果我们一次性启动了很多个任务，首先在第一次运行execute()方法的时候，会调用ArrayDeque的offer()方法将传入的Runnable对象添加到队列的尾部，然后判断mActive对象是不是等于null，第一次运行当然是等于null了，于是会调用scheduleNext()方法。在这个方法中会从队列的头部取值，并赋值给mActive对象，然后调用THREAD_POOL_EXECUTOR去执行取出的取出的Runnable对象，之后如果又有新的任务被执行，同样还会调用offer()方法将传入的Runnable添加到队列的尾部，但是再去给mActive对象做非空检查的时候就会发现mActive对象已经不再是null了，于是就不会再调用scheduleNext()方法而是直接启动run方法。
这是Futrue中的源码
```ruby
public void run() {
        if (state != NEW ||
            !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```
这里的callback对象正是传入的worker对象，通过执行worker的call方法，正式调用了doInBackground和postResult函数，
doInBackground会在这个子线程中执行耗时任务，而postResult则是通过InternalHandler来发送一个Message消息为MESSAGE_POST_PROGRESS给主线程，通知进行UI更新也就是执行onProgressUpdate函数，
当任务执行完毕后，以同样的方式来通过MESSAGE_POST_RESULT消息来通知结果，最后，在finish函数中执行onPostExecute函数。

### 总结：
AsyncTask的内部封装了两个线程池(SerialExecutor和THREAD_POOL_EXECUTOR)和一个Handler(InternalHandler)。
整体的逻辑是，SerialExecutor线程池用于任务的排队，让需要执行的多个耗时任务，按顺序排列，用来保证任务的串行性，THREAD_POOL_EXECUTOR线程池才真正地执行任务，InternalHandler用于从工作线程切换到主线程。
另，默认情况下AsyncTask的执行效果是串行的，因为有了SerialExecutor类来维持保证队列的串行。如果想使用并行执行任务，那么可以直接跳过SerialExecutor类，使用executeOnExecutor()来执行任务。

### 注意
非静态AsyncTask创建时会默认持有一个Activity或者Fragment的引用，所以务必在onDestory中手动调用cancel(boolean)方法来释放，防止内存泄漏
