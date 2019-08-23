`指南`

`ExecutorService.class`

_ExecutorService其实只是一个接口，线程池接口，真正实现接口的有ThreadPoolExecutor等_

**示例（实例化ThreadPoolExecutor并使用）：**

    ExecutorService proThreadPool = Executors.newFixedThreadPool(50);
    proThreadPool.submit(new Runnable() {
        @Override
        public void run() {
            while (true){
                try {
                    System.out.println("test....");
                    Thread.sleep(1000);
                }catch (Exception e){
                    
                }
            }
        }
    });
    
_先来看看newFixedThreadPool方法，这是个静态方法，在执行器Executors中（nThreads）是线程池中的线程数_

    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
   
_接下来看看ThreadPoolExecutor方法,这个方法是ThreadPoolExecutor的构造方法，也是线程池的核心方法_

    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }

_Executors.defaultThreadFactory()，创建了一个线程池new DefaultThreadFactory()，下一个方法_
    
    public ThreadPoolExecutor(int corePoolSize,
                                  int maximumPoolSize,
                                  long keepAliveTime,
                                  TimeUnit unit,
                                  BlockingQueue<Runnable> workQueue,
                                  ThreadFactory threadFactory,
                                  RejectedExecutionHandler handler) {
            if (corePoolSize < 0 ||
                maximumPoolSize <= 0 ||
                maximumPoolSize < corePoolSize ||
                keepAliveTime < 0)
                throw new IllegalArgumentException();
            if (workQueue == null || threadFactory == null || handler == null)
                throw new NullPointerException();
            this.corePoolSize = corePoolSize;
            this.maximumPoolSize = maximumPoolSize;
            this.workQueue = workQueue;
            this.keepAliveTime = unit.toNanos(keepAliveTime);
            this.threadFactory = threadFactory;
            this.handler = handler;
        }

    
_这个方法的所需的corePoolSize（保留的线程数），和maximumPoolSize（池中最大的线程数），
keepAliveTime（当线程数大于内核时，这些多余的空闲线程在终止新任务之前等待新任务的最长时间。），
这些都有相应的数值大小，时间是纳秒，所以会进行转化unit.toNanos(keepAliveTime)，
TimeUnit是传进来的时间单位，这个方法只是进行传值构建ThreadPoolExecutor（线程池执行者）而已，
接下来看ThreadPoolExecutor的uml图：_

            (I)Executor
                ↑ （继承）
         (I)ExecutorService
                ⇡(实现)
    (ac)AbstractExecutorService
                ↑（继承）
        (c)ThreadPoolExecutor

_接下来看具体使用方法，对象有了,调用submit，调度线程，查看submit方法，这个方法在接口ExecutorService，具体实现在抽象方法AbstractExecutorService中：_

    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
    }
    //这个方法比上面方法多了一个回调值result，在执行回调会返回result值，
    //Runnable task, T result,这俩值是RunnableAdapter（实现Callable）的属性字段，
    //这个RunnableAdapter就是个回调器。
    public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task, result);
        execute(ftask);
        return ftask;
    }
    
    //上面讲了Callable（回调器），我们可以直接传入回调器，newTaskFor有支持回调器的
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }

_这个方法在AbstractExecutorService类中，里面调用了newTaskFor，接下来看newTaskFor：_

    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }
    
_这里new了一个FutureTask，接下来看FutureTask，这是个继承Runnable,实现 Future的类：_

    public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
    }

_由于这个类是继承runnable和实现future的类，所以功能必然是实现线程run，run后回调result的功能类，接下来看里面的run方法：_
 
    public void run() {
        if (this.state == 0 && UNSAFE.compareAndSwapObject(this, runnerOffset, (Object)null, Thread.currentThread())) {
            boolean var9 = false;

            try {
                var9 = true;
                Callable var1 = this.callable;
                if (var1 != null) {
                    if (this.state == 0) {
                        Object var2;
                        boolean var3;
                        try {
                            var2 = var1.call();
                            var3 = true;
                        } catch (Throwable var10) {
                            var2 = null;
                            var3 = false;
                            this.setException(var10);
                        }

                        if (var3) {
                            this.set(var2);
                            var9 = false;
                        } else {
                            var9 = false;
                        }
                    } else {
                        var9 = false;
                    }
                } else {
                    var9 = false;
                }
            } finally {
                if (var9) {
                    this.runner = null;
                    int var6 = this.state;
                    if (var6 >= 5) {
                        this.handlePossibleCancellationInterrupt(var6);
                    }

                }
            }

            this.runner = null;
            int var12 = this.state;
            if (var12 >= 5) {
                this.handlePossibleCancellationInterrupt(var12);
            }

        }
    }    
_这个run其实就是runnable调用run，也就是线程start后，会调用callable（回调器）的call方法的一个线程执行者而已
没什么特别，FutureTask其实仅仅是线程启动，执行完毕，回调而已，接下来具体看看这里使用了执行器Executors的callable，返回callable对象，一个回调对象，
这个和上面的接口Executor不同，是一个类，具体看看他的静态方法callable_

    public static <T> Callable<T> callable(Runnable task, T result) {
        if (task == null)
            throw new NullPointerException();
        return new RunnableAdapter<T>(task, result);
    }
    
_这里new了RunnableAdapter，这是Executors的内部类，查看下：_

    static final class RunnableAdapter<T> implements Callable<T> {
        final Runnable task;
        final T result;
        RunnableAdapter(Runnable task, T result) {
            this.task = task;
            this.result = result;
        }
        public T call() {
            task.run();
            return result;
        }
    }
    
_这个类实现Callable，这个接口只有一个方法call，回调，也就是这个适配器（RunnableAdapter），
在调用call后，会执行runnable调用后，返回result出去。
接下来看看FutureTask的uml：_

        (I)Runnable    (I)Future
             ⇡（继承）     ⇡（继承）
            (I)RunnableFuture
                ⇡（实现）
            (c)FutureTask

_上面类图明显的指出我们new出来的FutureTask是实现RunnableFuture，回过头来，我们已经new了这个对象，继续走submit方法：_
    
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
    }

_这里执行了execute，这个方法在接口Executor中，实现在ThreadPoolExecutor中，接下来看execute方法：_

    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
    
    private static int workerCountOf(int var0) {
        return var0 & 536870911;
    }
    
_由于ctl(线程数量)是原子量，workerCountOf(c)，536870911是2^29 - 1;所以var0 & 536870911 = var0;
workerCountOf(var2) < this.corePoolSize，意思是现在线程数量小于保留的剩余的线程个数，就允许
addWorker(command, true)，将现在的线程加入线程池，接下来看看addWorker方法_

    private boolean addWorker(Runnable var1, boolean var2) {
        while(true) {
            int var3 = this.ctl.get();
            int var4 = runStateOf(var3);
            if (var4 >= 0 && (var4 != 0 || var1 != null || this.workQueue.isEmpty())) {
                return false;
            }

            while(true) {
                int var5 = workerCountOf(var3);
                if (var5 >= 536870911 || var5 >= (var2 ? this.corePoolSize : this.maximumPoolSize)) {
                    return false;
                }

                if (this.compareAndIncrementWorkerCount(var3)) {
                    boolean var18 = false;
                    boolean var19 = false;
                    ThreadPoolExecutor.Worker var20 = null;

                    try {
                        var20 = new ThreadPoolExecutor.Worker(var1);
                        Thread var6 = var20.thread;
                        if (var6 != null) {
                            ReentrantLock var7 = this.mainLock;
                            var7.lock();

                            try {
                                int var8 = runStateOf(this.ctl.get());
                                if (var8 < 0 || var8 == 0 && var1 == null) {
                                    if (var6.isAlive()) {
                                        throw new IllegalThreadStateException();
                                    }

                                    this.workers.add(var20);
                                    int var9 = this.workers.size();
                                    if (var9 > this.largestPoolSize) {
                                        this.largestPoolSize = var9;
                                    }

                                    var19 = true;
                                }
                            } finally {
                                var7.unlock();
                            }

                            if (var19) {
                                var6.start();
                                var18 = true;
                            }
                        }
                    } finally {
                        if (!var18) {
                            this.addWorkerFailed(var20);
                        }

                    }

                    return var18;
                }

                var3 = this.ctl.get();
                if (runStateOf(var3) != var4) {
                    break;
                }
            }
        }
    }

_这个addWorker方法，是一个循环，取得线程个数（ int var3 = this.ctl.get();），如果当前状态线程满就不允许加
（var4 >= 0 && (var4 != 0 || var1 != null || this.workQueue.isEmpty())），接下来如果有最大线程数限制
，现在运行的线程数不允许超过设置的最大线程数和边界值536870911（var5 >= 536870911 || var5 >= (var2 ? this.corePoolSize : this.maximumPoolSize)），
然后线程数加1（this.compareAndIncrementWorkerCount(var3)），如果加入成功，就var20 = new ThreadPoolExecutor.Worker(var1);
接下来看看Worker方法，Worker是一个类，在：_

    private volatile ThreadFactory threadFactory;
    Worker(Runnable var2) {
        this.setState(-1);
        this.firstTask = var2;
        this.thread = ThreadPoolExecutor.this.getThreadFactory().newThread(this);
    }
    
_这里的getThreadFactory() 返回的是一个volatile的原子量数，只能有一个对象，保证线程池工厂只有一个，
由于我们一开始创建的是DefaultThreadFactory这个线程池工厂，接下来看看newThread():_

    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r,
                              namePrefix + threadNumber.getAndIncrement(),
                              0);
        if (t.isDaemon())
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }

_这就是一个简单的线程构建方法。也就是用线程池里的工厂创建线程而已，返回上面方法addWorker：
接下来状态判断，如果线程已经加入并运行就返回，如果没有就加入（var8 < 0 || var8 == 0 && var1 == null）{ this.workers.add(var20);}
这样整体的线程加入成功了，然后继续往下会有 var6.start();，这里就是调用线程启动了，接下来就走FutureTask的run_

接下来讲讲AbstractExecutorService的其他的方法invokeAll：

    public static void main(String args[]){
		try {
			StopWatch sw = new StopWatch();
			sw.start();
			ExecutorService executor = Executors.newFixedThreadPool(10);
			List<Callable<Integer>> tasks = new ArrayList<Callable<Integer>>();
			for (int i =0;i<1000000;i++){
				Callable<Integer> c = new Callable<Integer>() {
					public Integer call() {
						return RandomUtils.nextInt();
					}
				};
				tasks.add(c);
			}
			List<Future<Integer>> futures = executor.invokeAll(tasks);

			for (Future<Integer> fs: futures) {
				int res = fs.get();
				System.out.println(res);
			}
			executor.shutdown();
			System.out.println(sw.getTime());
			sw.stop();
		}catch (Exception e){

		}
	}

这个方法其实就是分布式执行方法，等待全部执行成功并返回所有值。这种一般用于执行多个耗时任务，
如果普通任务就不要用，因为效率不高，相反如果很耗时任务，可以用这个方法，并等待执行结果，
例如数据库批量插入50000000条数数据，可以用这种方法，多线程执行，效率高

