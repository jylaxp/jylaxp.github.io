---
layout: post
title: AbstractQueuedSynchronizer
category: Java
img-path: 2018-01-14- AbstractQueuedSynchronizer
---

最近在看java.util.concurrent包下关于锁的代码，里面最核心的代码是AbstractQueuedSynchronizer.class这个类。我在网上也看了一些分析该类代码和逻辑的帖子，但总是感觉分析的不是能确切的切中要害(或者有这样的帖子，我没有找到)，因此，自己动手写一下自己的理解。这里，我们抛开锁的概念，因为AbstractQueuedSynchronizer这个工具类是解决资源竞争问题的，锁只是资源的一种，而且AbstractQueuedSynchronizer也只是为资源竞争提供就解决框架，并不是解决具体场景的问题的。

### 问题的场景  
两种模式:

- 独占模式，一次只能满足一个申请者
- 共享模式，可以同时满足多个申请者

接下来我们考虑申请资源会遇到哪些情况

- 资源池里有多少资源
- 申请者希望申请多少个资源
- 资源池里的资源不能满足当前申请者，该怎么办
- 申请者申请资源时没有被满足，那后来资源能满足该申请者了，此时怎么办 

### 代码分析
基于上面申请者申请资源会遇到的情况，我这里按资源类型分别分析代码。在开始分析之前，我们看一下关于"资源池"的定义和使用，资源池里的资源的使用方式和初始化数量，不同的场景有不同的使用方式，这个并不是统一一种方式，所以在这个框架里不会关心资源池和资源分配的事情，框架只关心申请者能不能被满足申请，如果不能被满足，框架要关心怎么处理申请者。
#### 独占模式
**申请资源**
我们来看申请资源的代码(不响应中断模式)。
```java
public final void acquire(int arg) {
	if (!tryAcquire(arg) &&
		acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
		selfInterrupt();
}
```
申请者(线程)调用acquire()方法申请arg个资源，首先会调用tryAcquire()方法尝试申请资源，该方法返回申请资源的结果－－成功或失败。如果申请资源成功，acquire()方法结束，申请者获得资源，可以干活了；如果失败，则调用acquireQueued()方法排队，等待申请的资源被满足。acquireQueued()方法的返回值是该线程在排队期间有没有被中断唤醒过(唤醒时会清除中断标示位)，当该线程(申请者)出队列时，如果排队期间被中断过，则设置线程的中断标示位。
这里申请资源时只是调用了tryAcquire()方法，这个方法是留给子类来解决具体的问题时覆盖重写的，也就说资源的管理和分配是给子类来实现的，框架并不关心这些，因此，这里我们不讨论资源是如何初始化和管理的，我们只关心申请结果。

我们来跟进addWaiter()方法，看一下它在干什么。
```java
private Node addWaiter(Node mode) {
	Node node = new Node(Thread.currentThread(), mode);
	// Try the fast path of enq; backup to full enq on failure
	Node pred = tail;
	if (pred != null) {
		node.prev = pred;
		if (compareAndSetTail(pred, node)) {
			pred.next = node;
			return node;
		}
	}
	enq(node);
	return node;
}

private Node enq(final Node node) {
	for (;;) {
		Node t = tail;
		if (t == null) { // Must initialize
			if (compareAndSetHead(new Node()))
				tail = head;
		} else {
			node.prev = t;
			if (compareAndSetTail(t, node)) {
				t.next = node;
				return t;
			}
		}
	}
}
```

在这里我们要先了解一下AQS框架的排队队列。这个队列是个双向队列，每个节点都是Node结构。
```java
static final class Node {
	/** 标记节点是共享模式 */
	static final Node SHARED = new Node();
	
	/** 标记节点是独占模式 */
	static final Node EXCLUSIVE = null;

	/** waitStatus的值，表示线程取消 */
	static final int CANCELLED =  1;
	
	/** waitStatus的值，表示线程需要唤醒 */
	static final int SIGNAL    = -1;
	
	/** waitStatus的值，表示线程在等待唤醒条件 */
	static final int CONDITION = -2;
	
	/** waitStatus的值，在共享模式下使用 */
	static final int PROPAGATE = -3;

	/** Status field, taking on only the values:　 SIGNAL,   CANCELLED,   CONDITION,   PROPAGATE, 0　 */
	volatile int waitStatus;

	/** 前一个节点*/
	volatile Node prev;

	/**　后一个节点 */
	volatile Node next;

	/**　线程 */
	volatile Thread thread;

	/**　记录模式　 */
	Node nextWaiter;

	/**　是否是共享模式　 */
	final boolean isShared() {
		return nextWaiter == SHARED;
	}

	/**　获取当前节点的前序节点　*/
	final Node predecessor() throws NullPointerException {
		Node p = prev;
		if (p == null)
			throw new NullPointerException();
		else
			return p;
	}

	Node() {    // Used to establish initial head or SHARED marker
	}

	Node(Thread thread, Node mode) {     // Used by addWaiter
		this.nextWaiter = mode;
		this.thread = thread;
	}

	Node(Thread thread, int waitStatus) { // Used by Condition
		this.waitStatus = waitStatus;
		this.thread = thread;
	}
} 
 ```
再看addWaiter()的代码。首先创建了一个Node，然后尝试入队。
```java
Node pred = tail;
if (pred != null) {
	node.prev = pred;
	if (compareAndSetTail(pred, node)) {
		pred.next = node;
		return node;
	}
}
```
这里检查对尾是否为null,不为null则使用compareAndSetTail插入对尾。compareAndSetTail是原子操作，AQS框架都是使用CAS来保证原子操作的，这也是保证并发的关键点。如果tail为null或者入队失败，则调用enq()入队。enq()方法使用for循环，不断尝试将节点插入队列尾部，直到成功为止。在入队之前，检查了队列是否为空，如果为空，则设置一个dummy的头节点，此头节点仅仅标示是头，不包含线程。addWaiter()方法将节点入队之后，返回该节点，供acquireQueued()调度。
```java
final boolean acquireQueued(final Node node, int arg) {
	boolean failed = true;
	try {
		boolean interrupted = false;
		for (;;) {
			final Node p = node.predecessor();
			if (p == head && tryAcquire(arg)) {
				setHead(node);
				p.next = null; // help GC
				failed = false;
				return interrupted;
			}
			if (shouldParkAfterFailedAcquire(p, node) &&
				parkAndCheckInterrupt())
				interrupted = true;
		}
	} finally {
		if (failed)
			cancelAcquire(node);
	}
}
```
先看for循环里的逻辑。
```java
final Node p = node.predecessor();
if (p == head && tryAcquire(arg)) {
	setHead(node);
	p.next = null; // help GC
	failed = false;
	return interrupted;
}
```
判断当前节点在队列中是不是第一节点，如果是第一个节点，则尝试申请资源。如果申请成功，则将该节点设置为头节点，返回中断标识。如果该节点不是第一个点或者该节点是第一个节点，但申请资源失败了，则调用shouldParkAfterFailedAcquire()判断是否需要挂起该节点，如果需要挂起该节点，则使用parkAndCheckInterrupt()方法挂起当前线程(节点里持有当前线程的引用)，至此，申请者把自己挂起了，等待被中断唤醒，然后继续执行。线程被唤醒之后设置了中断标示位，但是被唤醒不代表获取到资源，所以还需要再走一遍上面的逻辑，检查自己是否位于队列里第一个节点，获取资源等，如果不满足条件，则再次挂起自己，这个过程可能循环往复很多次，所以使用了for循环，直到其获得资源。我们看一下shouldParkAfterFailedAcquire()里判断是否需要挂起的逻辑。
```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
	// 节点状态的取值有：
	// 0：
	// 1:申请取消, CANCELLED
	// -1: SIGNAL
	// -2: CONDITION
	// -3: PROPAGATE
	int ws = pred.waitStatus;
	if (ws == Node.SIGNAL)
		// 前序节点的状态是SIGNAL，也就是前序节点在等待被唤醒
		// 意味着前序节点还有获取到资源，那么但节点是不可能获取到资源，需要挂起等待
		return true;
	if (ws > 0) {
		// 前序节点的状态大于0，意思是前序接待被取消了，则向前遍历队列，将取消的节点从队列中去除
		// 完成该操作之后，node可能是位于队列中第一个节点，需要尝试获取资源，所以不能挂起
		do {
			node.prev = pred = pred.prev;
		} while (pred.waitStatus > 0);
		pred.next = node;
	} else {
		// 将前序节点设置为SIGNAL状态，不关心设置成功与否
		// 返回false，让外层的for循环再走一遍获取资源，判断是否需要挂起的逻辑
		// 可能会设置成功(为什么不是一定，自己想，不解释)
		compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
	}
	return false;
}
```
逻辑解释都写在了注释里，这里不啰嗦了。好，到此申请资源的代码分析结束。那么，这里的独占模式体现在哪里呢？请看官们对比共享模式思考。
**释放资源**
分析完了资源申请，接着分析资源释放。
```java
public final boolean release(int arg) {
	if (tryRelease(arg)) {
		Node h = head;
		if (h != null && h.waitStatus != 0)
			unparkSuccessor(h);
		return true;
	}
	return false;
}
```
资源是由子类来管理的，同资源申请，资源释放也应该由子类来释放，所以这里调用了tryRelease()来释放资源，tryRelease()同tryAcquire()，由子类根据其使用场景来重写覆盖。资源释放成功，判断头节点不为null，并且其状态不是0，如果满足条件则，unparkSuccessor()。这里要说明一点，头节点的状态，只有在初始化等待队时为0，此后不会再是0了(独占模式)。来看unparkSuccessor()。
```java
private void unparkSuccessor(Node node) {
         // 修改节点状态为0
	int ws = node.waitStatus;
	if (ws < 0)
		compareAndSetWaitStatus(node, ws, 0);

         // 取该节点的后续节点
	Node s = node.next;
	
	// s == null 标示node节点可能已经被从队列中移除了
	// s.waitStatus > 0　标示该节点取消了
	// 所以要从队尾向前遍历，找到最前面的需要唤醒的节点，为什么不从head向后遍历，取第一需要唤醒的节点，请自己思考
	if (s == null || s.waitStatus > 0) {
		s = null;
		for (Node t = tail; t != null && t != node; t = t.prev)
			if (t.waitStatus <= 0)
			s = t;
	}
	
	// 找到了待唤醒的节点，则唤醒该节点持有的线程
	if (s != null)
		LockSupport.unpark(s.thread);
}
```
释放资源的逻辑比较简单，当前线程释放玩资源，需要唤醒后一个排队的线程，否则，后续等待的线程就没有机会获取资源执行了。

分析完不响应中断模式，再来看响应中断模式，绝大部分逻辑是一样的，只是处理中断不一样，看你代码很容易理解。
```java
public final void acquireInterruptibly(int arg)
	throws InterruptedException {
	if (Thread.interrupted())
		// 如果申请资源时线程中断标示位已经置位，直接抛出InterruptedException
		throw new InterruptedException();
	if (!tryAcquire(arg))
		// 申请资源失败
		doAcquireInterruptibly(arg);
}

private void doAcquireInterruptibly(int arg)
	throws InterruptedException {
	final Node node = addWaiter(Node.EXCLUSIVE);
	boolean failed = true;
	try {
		for (;;) {
			final Node p = node.predecessor();
			if (p == head && tryAcquire(arg)) {
				setHead(node);
				p.next = null; // help GC
				failed = false;
				return;
			}
			if (shouldParkAfterFailedAcquire(p, node) &&
				parkAndCheckInterrupt())
				// 线程中断过，唤醒时抛出InterruptedException
				throw new InterruptedException();
		}
	} finally {
		if (failed)
			// 取消申请
			cancelAcquire(node);
	}
}
```
代码和不响应中断的代码差不多，只不过在线程唤醒是检查了是佛有中断过，中断过就抛出InterruptedException异常，在finally把申请取消。



#### 共享模式
**申请资源**

我们来看申请资源的代码(不响应中断模式)。
```java
public final void acquireShared(int arg) {
	if (tryAcquireShared(arg) < 0)
		// 申请资源失败
		doAcquireShared(arg);
}

private void doAcquireShared(int arg) {
	final Node node = addWaiter(Node.SHARED);
	boolean failed = true;
	try {
		boolean interrupted = false;
		for (;;) {
			final Node p = node.predecessor();
			if (p == head) {
				int r = tryAcquireShared(arg);
				if (r >= 0) {
					// 注意这行代码
					setHeadAndPropagate(node, r);
					p.next = null; // help GC
					if (interrupted)
						selfInterrupt();
					failed = false;
					return;
				}
			}
			if (shouldParkAfterFailedAcquire(p, node) &&
				parkAndCheckInterrupt())
				interrupted = true;
		}
	} finally {
		if (failed)
			cancelAcquire(node);
	}
}
```
共享模式和独占模式代码逻辑差不多，主要区别的在于```setHeadAndPropagate(node, r);```这行代码，独占模式是```setHead(node);```。共享模式，会在节点获得到起源之后会唤醒下个节点，也就是所谓的"传播"。
```java
private void setHeadAndPropagate(Node node, int propagate) {
	Node h = head; // Record old head for check below
	setHead(node);
	/*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus either before
         *     or after setHead) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
         //　当剩余资源数大于0
         //  head == null 或者　(head != null 并且　h.waitStatus < 0)
         // 这些情况下唤醒后续节点
         // 为什么有这些判断，特别是h.waitStatus < 0，是为在队列里没有等待节时，不用向后传播，后续会把头节点的状态改成０
	if (propagate > 0 || h == null || h.waitStatus < 0 ||
		(h = head) == null || h.waitStatus < 0) {
		Node s = node.next;
		if (s == null || s.isShared())
			doReleaseShared();
	}
}
```
在setHeadAndPropagate()方法中设置了head之后，判断剩余资源数和头节点的状态，决定是否唤醒后续节点。
```java
private void doReleaseShared() {
	/*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
	for (;;) {
		Node h = head;
		if (h != null && h != tail) {
			int ws = h.waitStatus;
			if (ws == Node.SIGNAL) {
				// 修改头节点状态
				if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
					continue;            // loop to recheck cases
				unparkSuccessor(h);
			}
			else if (ws == 0 &&
					 !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
				continue;                // loop on failed CAS
		}
		if (h == head)                   // loop if head changed
			break;
	}
}
```

**释放资源**

```java
public final boolean releaseShared(int arg) {
	if (tryReleaseShared(arg)) {
		doReleaseShared();
		return true;
	}
	return false;
}
```
调用tryReleaseShared()释放资源，释放成功，调用doReleaseShared()唤醒等待资源的线程。



