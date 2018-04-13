---
layout:     post
title:      "Callable、Future、RunnableFuture、FutureTask的原理及应用"
subtitle:   " \"人生就像一场马拉松，跑到最后才是赢家。\""
date:       2018-04-08 16:24:00
author:     "Leon"
header-img: "img/2018-04-03-posts-bg.jpg"
catalog: true
tags:
    - 多线程
---

> “Anyway, anyhow. ”

### Callable、Future、RunnableFuture、FutureTask的继承关系
话不多说先上图([图片来自网络](https://www.cnblogs.com/nullzx/p/5147004.html))：
<img class="shadow" src="/blog/img/ThreadPoolExcutor.jpg" width="780" height="780">
通常我们通过一个实现了``Runnable``接口的对象来创建一个线程，这个线程在内部会执行``Runnable``对象的``run``方法。如果说我们创建一个线程来完成某项工作，希望在完成以后该线程能够返回一个结果，但``run``方法的返回值是``void``类型，直接实现``run``方法并不可行，这时我们就要通过``FutureTask``类来间接实现。

``FutureTask``实现了``RunnableFuture``接口，而``RunnableFuture``接口实际上同时继承了``Runnable``接口和``Future``接口。``Future``接口提供取消任务、检测任务是否执行完成、等待任务执行完成获得结果等方法。从图中可以看出，``FutureTask``类中的``run``方法已经实现好了(图中的代码仅仅是核心代码)，这个run方法实际上就是调用了由构造函数传递进来的``call``方法，并将返回值存储在FutureTask的私有数据成员``outcome``中。这样一来我们将FutureTask传递给一个Thread时，表面上我们仍然执行的是run，但在**run方法的内部实际上执行的是带有返回值的call方法**，这样即使得java多线程的执行框架保持不变，又实现了线程完成后返回结果的功能。同时FutureTask又将结果存储在outcome中，我们可以通过调用FutureTask对象的``get``方法获取outcome(也就是call方法的返回结果)。

### Future接口
```
public interface Future<V> {

    boolean cancel(boolean mayInterruptIfRunning);

    boolean isCancelled();

    boolean isDone();

    V get() throws InterruptedException, ExecutionException;

    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
Future接口声明了5个方法：
- ``cancel(boolean mayInterruptIfRunning)``方法用来取消任务，如果取消任务成功则返回``true``；否则返回``false``。参数``mayInterruptIfRunning``表示是否允许取消正在执行却没有执行完毕的任务。如果设置为``true``，则表示可以取消正在执行过程中的任务。如果任务已经完成，无论``mayInterruptIfRunning``设置为何值，该方法已经会返回``false``。如果任务还未执行，无论``mayInterruptIfRunning``设置为何值，该方法已经会返回``true``。
- ``isCancelled()``方法表示任务是否被取消成功，如果在任务正常完成前取消成功，则返回``true``。
- ``isDone()``方法表示任务是否已经完成，若任务完成，则返回``true``；
- ``get()``方法用来获取执行结果``outcome``，这个方法会产生阻塞，会一直等到任务执行完毕才返回。
- ``get(long timeout, TimeUnit unit)``用来获取执行结果，如果在指定时间内，还没获取到结果，就直接返回``null``。

总的来说``Future``接口提供了如下功能：
- 判断任务是否完成；
- 中断任务；
- 获取任务执行结果。

#### FutureTask
由于``Future``只是一个接口，无法直接使用它。因此有了FutureTask。
我们先来看下FutureTask的实现：
```
public class FutureTask<V> implements RunnableFuture<V>{

}
```
FutureTask类实现了RunnableFuture接口：
```
public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
```
FutureTask是一个可取消的异步计算，FutureTask 实现了Future的基本方法，提供``start``,``cancel``操作，可以用get()方法查询计算是否已经完成，并且可以获取计算的结果。

一个FutureTask 可以用来包装一个``Callable``或是一个``Runnable``对象。因为FurtureTask实现了Runnable方法，所以一个 FutureTask可以提交(submit)给一个Excutor执行(excution). 它同时实现了Callable, 所以也可以作为Future得到Callable的返回值。

FutureTask有两个很重要的属性分别是state和runner， FutureTask之所以支持canacel操作，也是因为这两个属性。 
其中state为枚举值：
```
    private volatile int state; // 注意volatile关键字
    /**
     * 在构建FutureTask时设置，同时也表示内部成员callable已成功赋值，
     * 一直到worker thread完成FutureTask中的run();
     */
    private static final int NEW = 0;

    /**
     * woker thread在处理task时设定的中间状态，处于该状态时，
     * 说明worker thread正准备设置result.
     */
    private static final int COMPLETING = 1;

    /**
     * 当设置result结果完成后，FutureTask处于该状态，代表过程结果，
     * 该状态为最终状态final state,(正确完成的最终状态)
     */
    private static final int NORMAL = 2;

    /**
     * 同上，只不过task执行过程出现异常，此时结果设值为exception,
     * 也是final state
     */
    private static final int EXCEPTIONAL = 3;

    /**
     * final state, 表明task被cancel（task还没有执行就被cancel的状态）.
     */
    private static final int CANCELLED = 4;

    /**
     * 中间状态，task运行过程中被interrupt时，设置的中间状态
     */
    private static final int INTERRUPTING = 5;

    /**
     * final state, 中断完毕的最终状态，几种情况，下面具体分析
     */
    private static final int INTERRUPTED = 6;
```
state初始化为NEW。只有在set, setException和cancel方法中state才可以转变为终态。在任务完成期间，state的值可能为COMPLETING或INTERRUPTING。 
state有四种可能的状态转换：
- NEW -> COMPLETING -> NORMAL
- NEW -> COMPLETING -> EXCEPTIONAL
- NEW -> CANCELLED
- NEW -> INTERRUPTING -> INTERRUPTED

其他的成员变量：
```
 /** The underlying callable; nulled out after running */
    private Callable<V> callable;   // 具体run运行时会调用其方法call()，并获得结果。

    /** The result to return or exception to throw from get() */
    private Object outcome; // non-volatile, protected by state reads/writes   没必要为votaile,因为其是伴随state 进行读写，而state是FutureTask的主导因素。

    /** The thread running the callable; CASed during run() */
    private volatile Thread runner;   //具体的worker thread.

    /** Treiber stack of waiting threads */  
    private volatile WaitNode waiters;     //Treiber stack 并发stack数据结构，用于存放阻塞在该futuretask#get方法的线程。
```
#### Task生命周期
FutureTask构造方法：
```
 public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
 }
```
将state设置为初始态``NEW``。这里注意Runnable是怎样转换为Callable的，看下this.callable = Executors.callable(runnable, result); 调用Executors.callable:
```
public static <T> Callable<T> callable(Runnable task, T result) {
        if (task == null)
            throw new NullPointerException();
        return new RunnableAdapter<T>(task, result);
}

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
```
其实就是通过Callable的call方法调用Runnable的run方法，把传入的 T result 作为callable的返回结果；

当创建完一个Task通常会提交给Executors来执行，当然也可以使用Thread来执行，Thread的start()方法会调用Task的run()方法。看下FutureTask的run()方法的实现：

```
public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
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
首先判断任务的状态，如果任务状态不是new，说明任务状态已经改变（说明他已经走了上面4种可能变化的一种，比如caller调用了cancel，此时状态为Interrupting, 也说明了上面的cancel方法，task没运行时，就interrupt, task得不到运行，总是返回）；

如果状态是new, 判断runner是否为null, 如果为null, 则把当前执行任务的线程赋值给runner，如果runner不为null, 说明已经有线程在执行，返回。此处使用cas来赋值worker thread是保证多个线程同时提交同一个FutureTask时，确保该FutureTask的run只被调用一次， 如果想运行多次，使用runAndReset()方法。
```
!UNSAFE.compareAndSwapObject(this, runnerOffset, null, Thread.currentThread());
```
语义相当于
```
if (this.runner == null ){
    this.runner = Thread.currentThread();
}
```
使用compareAndSwap能够保证原子性。
接着开始执行任务，如果要执行的任务不为空，并且state为New就执行，可以看到这里调用了Callable的call方法。如果执行成功则set结果，如果出现异常则setException。最后把runner设为null。

接着看下set方法：
```
protected void set(V v) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
    }
```
如果现在的状态是NEW就把状态设置成COMPLETING，然后设置成NORMAL。这个执行流程的状态变化就是： NEW->COMPLETING->NORMAL。

最后执行finishCompletion()方法：
```
private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();

        callable = null;        // to reduce footprint
 }
```
finishCompletion()会解除所有阻塞的worker thread，调用done()方法，将成员变量callable设为null。这里使用了LockSupport类来解除线程阻塞。
接下来分析FutureTask非常重要的get方法:
```
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }
```
首先判断FutureTask的状态是否为完成状态，如果是完成状态，说明已经执行过set或setException方法，返回report(s):
```
    private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V)x;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
    }
```
可以看到，如果FutureTask的状态是NORMAL, 即正确执行了set方法，get方法直接返回处理的结果， 如果是取消状态，即执行了setException，则抛出CancellationException异常。

如果get时,FutureTask的状态为未完成状态，则调用awaitDone方法进行阻塞。awaitDone():
```
    private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();
            else if (q == null)
                q = new WaitNode();
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
            else
                LockSupport.park(this);
        }
    }
```
awaitDone方法可以看成是不断轮询查看FutureTask的状态。在get阻塞期间：

- 如果执行get的线程被中断，则移除FutureTask的所有阻塞队列中的线程（waiters）,并抛出中断异常；

- 如果FutureTask的状态转换为完成状态（正常完成或取消），则返回完成状态；

- 如果FutureTask的状态变为COMPLETING, 则说明正在set结果，此时让线程等一等；

- 如果FutureTask的状态为初始态NEW，则将当前线程加入到FutureTask的阻塞线程中去；

- 如果get方法没有设置超时时间，则阻塞当前调用get线程；如果设置了超时时间，则判断是否达到超时时间，如果到达，则移除FutureTask的所有阻塞列队中的线程，并返回此时FutureTask的状态，如果未到达时间，则在剩下的时间内继续阻塞当前线程。

注：本文源码基于JDK1.7。