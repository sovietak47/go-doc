# goalng 多线程编程 笔记

## 1.多线程编程中的通用问题

通用问题是指在多线程编程中会遇到的共性问题，不受限于语言的种类。当然，不同的语言，会提供不同的机制，以简化这类问题的处理难度，但不意味着共性问题会因此消失。

### 1.1竞争与依赖

### 1.1.1竞态条件（race conditions）

如果每个任务都是独立的，不需要共享任何资源，那多线程也就非常简单。但世界往往是复杂的，总有一些资源需要共享。当多个线程需要在同时使用一个共享资源时，就成了竞态条件。资源的竞态条件会导致数据的不一致性，进而引入一系列的时序错误和逻辑混乱，需要设法加以避免。

鉴于在程序中，我们可以把所有资源/资源的访问入口抽象为一个变量。那么，不妨将共享资源的竞态条件问题具象为共享资源变量的访问问题，即共享资源变量的竞态访问问题。如果一个代表共享资源的变量，在多线程访问中能解决多线程访问引入的数据一致性问题，则可称变量为"线程安全"的变量。

一般可以通过锁机制，简单的实现线程安全。


### 1.1.2依赖关系以及执行顺序


如果线程之间的任务并非相互独立的，而是有依赖关系。如线程B,C的执行一定要线程A执行完后才能进行这样的场景。此时，就需要为线程提供等待及通知机制来进行协调，以实现线程并发执行的控制。使之在存在依赖的部分，能实现有序的并发执行。

### 1.2程序的并发程度控制

复杂度较高的程序，可能会在运行中开出大量的线程。但是多线程的运行是存在成本的。并不是线程越多，程序的运行效率就越高。因此，当程序中可能会开出大量线程时，程序就会需要我们提供一个控制程序并发程度的机制，以便通过调试与测算，找到程序并发度的较优解。

此类需求，一般通过线程池的方式予以实现。



## 2.golang中的多线程编程问题

下面具体到golang语言，逐个分析多线程编程中的问题与注意事项。

### 2.1变量的线程安全问题

当一个变量代表一个被多个线程共享的资源————比如，该变量表征会被多个线程访问的一个公共对象，一个文件，一个可被多个线程可见的状态表征量，甚至是多线程间数据交换的通道————的时候，则该变量存在被多个线程并发访问的风险或者需求，该变量必须做到线程安全。


这类变量的作用域一般大于访问它的多个线程。即，相对域多个线程而言，该变量是以“全局变量”的方式出现。在访问该类变量时应当注意，应以访问“无状态资源”的方式访问该类变量：该类变量一般是不会，也不应“记忆”某一线程的"上一次操作"；上一秒访问得到的结果，并不意味着下一秒它还是这个结果。如确实需要记录与访问该变量相关的状态，该工作应由访问线程自行完成。

可以通过锁机制，简单的实现线程安全。

#### 2.1.1golang中通过锁机制实现变量的线程安全

在golang中，可以通过 "sync" 包 中提供的 Mutex、RWMutex 实现锁机制。

```golang
type Mutex
func (m *Mutex) Lock(){}
func (m *Mutex) Unlock(){}
type RWMutex
func (rw *RWMutex) Lock(){}
func (rw *RWMutex) Unlock(){}
func (rw *RWMutex) RLock(){}
func (rw *RWMutex) RUnlock(){}
func (rw *RWMutex) RLocker() Locker{}
```
Mutex 为互斥锁（悲观锁），可以确保的某段时间内，对象不能有多个线程同时访问；

RWMutex 为读写互斥锁（乐观锁），可以允许对象被多个线程读数据，但只能有且只有1个线程写数据。当然，它也提供互斥锁的功能。

下面举例说明锁的用法：
我们模拟一个账户系统。可以为系统添加用户，存钱，取钱，查询余额。

```golang
type Account struct {
	sync.RWMutex
	system map[string]int
}

func NewAccount() *Account {
	b := &Account{
		system: make(map[string]int),
	}
	return b
}

func (a *Account) AddUser(userName string) {
	a.Lock()
	defer a.Unlock()
	if a.isUserExist(userName) {
		log.Printf("can't AddUser:user %v exist", userName)
	}
	a.system[userName] = 0
}

func (a *Account) isUserExist(userName string) bool {
	_, isExist := a.system[userName]
	return isExist
}

func (a *Account) Query(userName string) {
	a.RLock()
	defer a.RUnlock()
	if !a.isUserExist(userName) {
		log.Printf("can't Save: user %v not exist", userName)
		return
	}

	amount := a.system[userName]
	log.Printf("Query success: user %v has %d", userName, amount)
}

func (a *Account) Save(userName string, saving int) {
	a.RLock()
	defer a.RUnlock()
	if !a.isUserExist(userName) {
		log.Printf("can't Save: user %v not exist", userName)
		return
	}

	amount := a.system[userName]
	amount += saving
	log.Printf("Save success: user %v has %d", userName, amount)
}

func (a *Account) Out(userName string, saving int) {
	a.RLock()
	defer a.RUnlock()
	if !a.isUserExist(userName) {
		log.Printf("can't Out: user %v not exist", userName)
		return
	}

	amount := a.system[userName]
	if amount < 0 {
		log.Printf("can't Out: user %v not exist", userName)
		return
	}
	amount -= saving
	log.Printf("Out success: user %v has %d", userName, amount)
}

```

首先，我们演示一下互斥锁：上锁后，会将其他线程阻塞。无论其他线程想做什么操作。
我们模拟一个账号系统挂掉的场景。当系统“挂掉”时，我们启动互斥锁，在互斥锁解锁之前，其他线程对账号系统的访问都会阻塞，直至互斥锁解锁。


```golang
func SystemOffline() {
	a := NewAccount()
	a.AddUser("existOne")

	go func(a *Account) {
		a.Lock()
		defer a.Unlock()
		// systemOffline
		log.Printf("system offline")
		time.Sleep(5 * time.Second)

		// systemOnline
		log.Printf("system back online")
	}(a)

	go func() {
		time.Sleep(1 * time.Second)
		a.Query("existOne")
	}()
}

func main() {
	SystemOffline()
	time.Sleep(10 * time.Second)
}

```

运行结果：
![Image text](https://raw.githubusercontent.com/hongmaju/light7Local/master/img/productShow/20170518152848.png)



  

  
线程内变量

一般是在非主线程内声明的。该类变量最大的问题在于对其他线程而言不可见。这其实是好事，
  
  

### 1.2


### 参考资料
> https://mp.weixin.qq.com/s?__biz=MzA4NjgwMDQ0OA==&mid=404961832&idx=1&sn=ff915a35c9d83ed8e5d153496c9298d2&pass_ticket=%2BrMy18QMJX0zmqAnofQvNtn0nis72v3CSdAEqaGbjEKMYYECVCJl3UMx3Pipz%2F%2BS

> 

