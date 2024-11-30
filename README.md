
# 简介


多线程与异步是两个完全不同的概念，常常有人混淆。


1. 异步
异步适用于"IO密集型"的场景,它可以避免因为线程等待IO形成的线程饥饿，从而造成程序吞吐量的降低。
其本质是：让线程的cpu片不再浪费在等待上，期间可以去干其它的事情。
要注意的是：Async不能加速程序的执行，它只能做到不阻塞线程。
2. 多线程
多线程适用于"CPU密集型",主要是为了更多的利用多核CPU来同时执行逻辑。将一个大任务分而治之，提高完成速度，进而提高程序的并发能力
值得注意的是，如果过多使用线程同步，会降低多线程的使用效果



> 在计算机科学中，一个线程指的是在程序中一段连续的逻辑控制流。在业务很复杂的时候，一个线程无法满足现有业务需求，多线程编程就应运而生。


# 异步请求流程图(Windows)


![image](https://img2024.cnblogs.com/blog/1084317/202411/1084317-20241127160106067-780909989.png)


1. ReadAsync底层调用win32 API ReadFile
2. ReadFile分配IRP数据结构(句柄，读取偏移量，用来填充的byte\[])，
3. 然后传递给windows内核中，
4. windows把IRP添加到硬盘驱动的IRP队列中，线程不再阻塞，立刻返回到线程池中(在此期IRP尚未处理完成)
5. 读取硬盘数据
6. 返回硬盘数据并组装IRP数据
7. 将IRP Enqueue IO Completion Port
8. ThreadPool轮询Dequeue该端口，提取IRP
9. 执行回调，如果没有回调这一步直接丢弃IRP数据


## 异步操作的核心:IO完成端口(IO Completion Port)


IO完成端口（IO Completion Port）是Windows操作系统的一个内核对象，专门用来解决异步IO的问题，C\#中所有异步操作都依赖此端口。
其本质是一个发布订阅模式的队列



> CLR在初始化时，创建一个IO Completion Port完成与硬件设备的绑定，使得硬件的驱动程序知道将IRP送到哪里去。


### 眼见为实：IO Completion Port真的存在吗？



```
        /// 
        /// 创建IO完成端口
        /// 
        [DllImport("kernel32.dll")]
        static extern nint CreateIoCompletionPort(nint FileHandle, nint ExistingCompletionPort, nint CompletionKey, int NumberOfConcurrentThreads);

        /// 
        /// IO数据入列
        /// 
        [DllImport("kernel32.dll")]
        static extern bool PostQueuedCompletionStatus(nint CompletionPort, int dwNumberOfBytesTransferred, nint dwCompletionKey, nint lpOverlapped);

        /// 
        /// IO数据出列
        /// 
        [DllImport("kernel32.dll")]
        static extern bool GetQueuedCompletionStatusEx(nint CompletionPort, out uint lpNumberOfBytes, out nint lpCompletionKey, out nint lpOverlapped, uint dwMilliseconds);

```

有兴趣的小伙伴可以玩一玩这个api.


### 眼见为实：异步API真的基于IO Completion Port吗？


众所周知，Task的底层是ThreadPool，那么答案一定在ThreadPool的源码中
No BB,上源码,IOCompletionPoller.Poll



```
            private void Poll()
            {
			//轮询调用GetQueuedCompletionStatusEx,获取IO数据。
                while (
                    Interop.Kernel32.GetQueuedCompletionStatusEx(
                        _port,
                        _nativeEvents,
                        NativeEventCapacity,
                        out int nativeEventCount,
                        Timeout.Infinite,
                        false))
                {
                    for (int i = 0; i < nativeEventCount; ++i)
                    {
                        Interop.Kernel32.OVERLAPPED_ENTRY* nativeEvent = &_nativeEvents[i];
                        if (nativeEvent->lpOverlapped != null) // shouldn't be null since null is not posted
                        {
						//把event事件和数据压入内部ConcurrentQueue队列，缓存起来。
						//.net 8之前的版本，直接就在这里执行回调了
                            _events.BatchEnqueue(new Event(nativeEvent->lpOverlapped, nativeEvent->dwNumberOfBytesTransferred));
                        }
                    }
					//压入线程池的highPriorityWorkItems队列
					//.net 8之后,由线程池执行回调
                    _events.CompleteBatchEnqueue();
                }

                ThrowHelper.ThrowApplicationException(Marshal.GetHRForLastWin32Error());
            }

```

# C\# 中的异步函数



```
        //一旦将方法标记为async，编译器就会将代码转换成状态机
        static async void Test()
        {
            //线程1进入,初始化client
            var httpClient = new HttpClient();
            //GetAsync内部分配一个Task对象
            var getTask = httpClient.GetAsync("https://www.baidu.com");
            //此时， aait操作符实际会在Task对象上调用ContinueWith,向它传递用于恢复状态机的方法，线程线程从Test()方法中返回
            //在未来某个时刻，IO Completion Port 完成网络IO入列，线程池通知Task对象，一个新线程会重新进入Test()方法，从await操作符的位置开始执行ContinueWith回调方法(也就是代码后面的内容)。
            var response = await getTask;
        }

```

# 编译器如何将异步函数转换为状态机？


[https://github.com/JulianHuang/p/18137189](https://github.com)
[https://github.com/huangxincheng/p/13558006\.html](https://github.com):[slowerssr加速器](https://slowerss.com)


分享几个写的不错的博文，偷懒一下。
核心是MoveNext函数，里面包含了根据状态机status而执行不同代码的模板代码.
一个Task最少要被调用两次MoveNext,第一次调用是主动触发初始化状态机，第二次调用是回调函数再次执行状态机



```
    public class GetStringAsync : IAsyncStateMachine
	{
	    public int state;
        private string html;
        private string taskResult;

        public AsyncTaskMethodBuilder builder;
        private TaskAwaiter<string> awaiter;
        /// 
        /// 状态机机制
        /// 
        public void MoveNext()
        {
            var localState = state;
            TaskAwaiter<string> localAwaiter = default(TaskAwaiter<string>);
            GetStringAsync localStateMachine;

            try
            {
                switch (localState)
                {
                    //第一次初始化 (publish)
                    case -1:
                        localAwaiter = Task.Run(() =>
                        {
                            Thread.Sleep(1000); //模拟网络IO耗时
                            var response = "# 百度

";
                            return response;
                        }).GetAwaiter();//转为TaskAwaiter对象，内部实现INotifyCompletion接口，使得具备传入回调函数的能力

                        if (!localAwaiter.IsCompleted)
                        {
                            localState = state = 0;
                            awaiter = localAwaiter;
                            localStateMachine = this;
                            builder.AwaitUnsafeOnCompleted(ref localAwaiter, ref localStateMachine);//将当前注册机传入回调函数,当前线程返回线程池
                            return;
                        }
                        break;

                    //第二次异步完成的回调 (subscribe)
                    case 0:
                        localAwaiter = awaiter;
                        awaiter = default(TaskAwaiter<string>);
                        localState = state = -1;
                        break;
                }
				//等价于ContinueWith
                taskResult = localAwaiter.GetResult();
                html = taskResult;
                taskResult = null;
                Console.WriteLine($"GetStringAsync方法返回：{html}");
            }
            catch (Exception ex)
            {
                state = -2;
                html = null;
                builder.SetException(ex);//只有调用await/result才会抛出异常，否则会丢弃。
                return;
            }

            state = -2;
            html = null;
            builder.SetResult();
        }
}

```

# 异步方法的异常处理


当异步操作发生异常时，IO Completion Port会告诉程序，异步操作已经完成，但存在一个错误。不会跟常规异常一样直接从内核态抛出一个异常。
因此ThreadPool会拿到IRP数据，里面包含了异常信息。它自己也不会抛出来。而是调用SetException存储起来。
当你调用await/GetResult() 时才会真正的抛出异常。因为当你没有及时获取Task的异常时，它会被丢弃。你需要妥善处理未抛出的异常


## 眼见为实



```
        internal TResult GetResultCore(bool waitCompletionNotification)
        {
            // If the result has not been calculated yet, wait for it.
            if (!IsCompleted) InternalWait(Timeout.Infinite, default); // won't throw if task faulted or canceled; that's handled below

            // Notify the debugger of the wait completion if it's requested such a notification
            if (waitCompletionNotification) NotifyDebuggerOfWaitCompletionIfNecessary();

            // Throw an exception if appropriate.
            if (!IsCompletedSuccessfully) ThrowIfExceptional(includeTaskCanceledExceptions: true);

            // We shouldn't be here if the result has not been set.
            Debug.Assert(IsCompletedSuccessfully, "Task.Result getter: Expected result to have been set.");

            return m_result!;
        }

```

# ValueTask


在众多异步场景中，有些场景是，GetAsync()第一次需要异步IO等待，然后把结果缓存到静态变量里。接下来N次都是不需要异步IO等待的。直接可以同步完成。
比如说Entity Framework中的FindAsync().只有第一次会查询数据库，剩下的N次直接读取内存。
如果使用Task ,从状态机的源码也可以看到，创建一个Task对象花销不少且为引用类型。创建越多对GC压力越大。


为了减少这种场景下的性能消耗，可以使用ValueTask，它为结构体值类型，正常不需要从托管堆中分配内存。


1. 如果异步操作不需要等待，可以同步完成，那么回调会被立刻调用，没有多余开销。
2. 如果异步操作需要等待，那依旧会创建一个Task对象



> 它的出现纯粹为了性能。


## 眼见为实


上源码
System.Private.CoreLib\\src\\System\\Runtime\\CompilerServices\\ValueTaskAwaiter.cs



```
        public TResult Result
        {
            get
            {
                object? obj = _obj;//Task对象

                if (obj == null)//Task完成后会置为null，大家猜一猜为什么要置为空？
                {
                    return _result!;//直接返回缓存的结果
                }

                if (obj is Task t) //Task未完成，还是走Task逻辑不变
                {
                    TaskAwaiter.ValidateEnd(t);
                    return t.ResultOnSuccess;
                }

                return Unsafe.As>(obj).GetResult(_token);//去IValueTaskSource里找缓存的result
            }
        }

```

