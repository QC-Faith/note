ThreadPoolExecutor



核心参数及执行流程



1. corePoolSize：线程池的核心线程数，即在没有任务执行时，线程池的基本大小。默认情况下，在创建线程池时会立即创建并保持在池中的线程数。
2. maximumPoolSize：线程池的最大线程数，即线程池允许的最大线程数。当队列满了并且仍然有新的任务到来时，线程池会创建新的线程，直到达到最大线程数。
3. keepAliveTime：线程空闲超时时间，即当线程池中的线程数量超过核心线程数时，多余的空闲线程在多长时间内被销毁。超过这个时间，空闲线程将被终止并从线程池中移除。
4. unit：keepAliveTime 参数的时间单位。
5. workQueue：任务队列，用于存储提交到线程池的任务。ThreadPoolExecutor 提供了多种不同类型的队列实现，如 ArrayBlockingQueue、LinkedBlockingQueue、SynchronousQueue 等。
6. threadFactory：线程工厂，用于创建新的线程。默认情况下，ThreadPoolExecutor 使用 Executors.defaultThreadFactory() 创建线程。
7. handler：拒绝策略，用于处理任务执行超出线程池容量的情况。ThreadPoolExecutor 提供了几种预定义的拒绝策略，如 AbortPolicy、CallerRunsPolicy、DiscardPolicy、DiscardOldestPolicy 等。

ThreadPoolExecutor 的执行流程如下：

1. 当有任务提交到线程池时，线程池会根据 corePoolSize 和当前线程数量来判断是否需要创建新的线程。
2. 如果当前线程数小于 corePoolSize，则线程池会立即创建新的线程来执行任务。
3. 如果当前线程数大于等于 corePoolSize，并且任务队列未满，则任务会被添加到任务队列中等待执行。
4. 如果任务队列已满，并且当前线程数小于 maximumPoolSize，则线程池会创建新的线程来执行任务。
5. 如果任务队列已满，并且当前线程数已达到 maximumPoolSize，则根据拒绝策略来处理任务。
6. 当任务执行完成后，空闲的线程会根据 keepAliveTime 参数判断是否需要被销毁。



常用拒绝策略？

```java
CallerRunsPolicy
    会提高负载，但是保证任务不被丢弃
    
    	private ThreadPoolExecutor executorService;

	public MinmetalsFRUploadLoanFileImpl() {
		executorService = new ThreadPoolExecutor(5, 5, 60L, TimeUnit.SECONDS, new ArrayBlockingQueue<>(50),new ThreadPoolExecutor.CallerRunsPolicy());
	}



			for(LoanApplyOrder order: loanOrderList) {
				Future<?> submit = executorService.submit(new InheritTraceNoRunableWrapper(new Runnable() {
					@Override
					public void run() {
						try {
							jtClient.connect();
							sftpClient.connect();
							updateFile(order,jtClient,sftpClient);
						} catch (Exception e) {
							logger.error("五矿上传文件异常",e);
						}
					}
				}));
			}
		} finally {
			waitComplete(executorService);
			jtClient.disconnect();
			sftpClient.disconnect();
		}



	protected void waitComplete(ThreadPoolExecutor threadPool) {
		while (true) {
			if (threadPool.getActiveCount() == 0 && threadPool.getQueue().size() == 0) {
				break;
			}
			try {
				Thread.sleep(500);
			} catch (InterruptedException e) {
				logger.error(e.getMessage(), e);
			}
		}
	}
```



常用api

1. `execute(Runnable command)`: 提交一个任务给线程池执行，无返回值。
2. `submit(Runnable task)`: 提交一个任务给线程池执行，并返回一个表示该任务待处理结果的 Future 对象。
3. `shutdown()`: 平缓地关闭线程池，不再接受新的任务，但会等待已提交的任务执行完成。
4. `shutdownNow()`: 立即关闭线程池，尝试取消所有运行中的任务，并返回等待执行的任务列表。
5. `isShutdown()`: 判断线程池是否已关闭。
6. `isTerminated()`: 判断线程池是否已经终止。
7. `awaitTermination(long timeout, TimeUnit unit)`: 等待线程池终止指定的时间。
8. `getActiveCount()`: 获取当前线程池中正在执行任务的线程数。
9. `getCorePoolSize()`: 获取线程池的核心线程数。
10. `setCorePoolSize(int corePoolSize)`: 设置线程池的核心线程数。
11. `getMaximumPoolSize()`: 获取线程池的最大线程数。
12. `setMaximumPoolSize(int maximumPoolSize)`: 设置线程池的最大线程数。
13. `getQueue()`: 获取线程池中使用的任务队列。
14. `getRejectedExecutionHandler()`: 获取当前线程池的拒绝策略。
15. `setRejectedExecutionHandler(RejectedExecutionHandler handler)`: 设置线程池的拒绝策略。





实现原理？

通过原子性整数控制当前的线程状态


RUNNING（运行）、SHUTDOWN（关闭中）、STOP（停止）、TERMINATED（终止）等

BlockingQueue通过可重入锁来实现，使用的就是aqs的reentranLock

