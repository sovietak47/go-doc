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


这类变量的作用域一般大于访问它的多个线程。即，相对于多个线程而言，该变量是以“全局变量”的方式出现。在访问该类变量时应当注意，应以访问“无状态资源”的方式访问该类变量：该类变量一般是不会，也不应“记忆”某一线程的"上一次操作"；上一秒访问得到的结果，并不意味着下一秒它还是这个结果。如确实需要记录与访问该变量相关的状态，该工作应由访问线程自行完成。

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
Mutex 为互斥锁。func (m *Mutex) Lock()方法锁住m，如果m已经加锁，则阻塞直到m解锁；func (m *Mutex) Unlock()方法解锁m，如果m未加锁会导致运行时错误。锁和线程无关，可以由不同的线程加锁和解锁。从而确保某段时间内，含有Mutex的对象只能被一个线程同时访问；

RWMutex 为读写互斥锁。func (rw *RWMutex) Lock()方法和func (m *Mutex) Unlock()方法与Mutex的类似，因此可以用于写入场景，rw锁定为写入状态，禁止其他线程读取或者写入。func (rw *RWMutex) RLock()方法将rw锁定为读取状态，禁止其他线程写入，但不禁止读取。func (rw *RWMutex) RUnlock()方法解除rw的读取锁状态，如果rw未加读取锁会导致运行时错误。RWMutex类型的锁也和线程无关，可以由不同的线程加读取锁/写入和解读取锁/写入锁。从而确保含有RWMutex的对象可以被多个线程读数据，但只能有且只有1个线程写数据。


下面举例说明锁的用法：

```golang
type System struct {
	sync.Mutex
	info map[string]string
}

func NewSystem(userInfo map[string]string) *System {
	s := &System{
		info: userInfo,
	}
	return s
}

func (s *System) isUserExist(userName string) bool {
	_, isExist := s.info[userName]
	return isExist
}

func (s *System) Query(userName string) {
	s.Lock()
	defer s.Unlock()

	if !s.isUserExist(userName) {
		log.Printf("can't Save: user %v not exist", userName)
		return
	}

	info := s.info[userName]
	log.Printf("Query success: user %v info: %v", userName, info)
}
```

我们将通过模拟一个信息系统挂掉的场景以演示互斥锁：上锁的线程，会将其他线程的执行阻塞，无论其他线程想做什么操作，直到该线程解锁。当系统“监控线程”发现系统“下线”时，会启动互斥锁，其他线程对信息系统的访问都会阻塞，直至互斥锁解锁。


```golang
func SystemOffline() {
	s := NewSystem(map[string]string{"user1":"info1"})
	go func(s *System) {
		s.Lock()
		defer s.Unlock()
		// systemOffline
		log.Printf("system offline")
		time.Sleep(5 * time.Second)

		// systemOnline
		log.Printf("system back online")

	}(s)
	go func() {
		time.Sleep(1 * time.Second)
		s.Query("user1")
	}()
}

func main() {
	SystemOffline()
	time.Sleep(10 * time.Second)
}

```

运行结果：

```golang
2019/04/28 23:10:40 system offline
2019/04/28 23:10:45 system back online
2019/04/28 23:10:45 Query success: user user1 info: info1
```

运行结果显示，只有在系统“监控线程”运行结束（故障结束）后，对信息系统的访问才会执行。


```golang
type FileSystem struct {
	sync.RWMutex
	info map[string]string
}

func NewFileSystem(initFiles map[string]string) *FileSystem {
	f := &FileSystem{
		info: initFiles,
	}
	return f
}

func (f *FileSystem) isFileExist(filePath string) bool {
	_, isExist := f.info[filePath]
	return isExist
}

func (f *FileSystem) ReadFile(filePath string, takeTime time.Duration) {
	f.RLock()
	defer f.RUnlock()

	//模拟业务耗时
	time.Sleep(takeTime)

	if !f.isFileExist(filePath) {
		log.Printf("can't Read: file %v not exist", filePath)
		return
	}

	content := f.info[filePath]
	log.Printf("Read success: file %v contents: %v", filePath, content)
}

func (f *FileSystem) WriteFile(filePath string, newContent string, takeTime time.Duration) {
	f.Lock()
	defer f.Unlock()

	//模拟业务耗时
	time.Sleep(takeTime)

	if !f.isFileExist(filePath) {
		log.Printf("can't Write: file %v not exist", filePath)
		return
	}

	f.info[filePath] = newContent
	log.Printf("Write success: file %v contents: %v", filePath, f.info[filePath])
}

```

我们将通过两个线程模拟两个用户同时访问文件系统的场景演示读写互斥锁。当两个用户（线程）都是读操作时，这种行为可以并行执行。但当有一个用户（线程）是写操作时，写操作用户（线程）将阻塞其他用户（线程）的访问。

```golang
func ReadAtTheSameTime(){
	fs := NewFileSystem(map[string]string{"file1":"file1 content"})
	go func() {
		fs.ReadFile("file1", 5 * time.Second)
		log.Println("reader1 finished")
	}()

	go func() {
		time.Sleep(time.Second)
		fs.ReadFile("file1", 1 * time.Second)
		log.Println("reader2 finished")
	}()
}

func main() {
	ReadAtTheSameTime()

	time.Sleep(10 * time.Second)
}

```

运行结果：

```golang
2019/04/29 00:21:58 Read success: file file1 contents: file1 content
2019/04/29 00:21:58 reader2 finished
2019/04/29 00:22:01 Read success: file file1 contents: file1 content
2019/04/29 00:22:01 reader1 finished
```

运行结果显示，当两个线程都是读操作时，两个线程能并行运行。启动较晚但是结束较早的reader2能按时结束对file1的访问。

```golang
func ReadWhileWrite(){
	fs := NewFileSystem(map[string]string{"file1":"file1 content"})
	go func() {
		fs.WriteFile("file1","new content" ,5 * time.Second)
		log.Println("reader1 finished")
	}()

	go func() {
		time.Sleep(time.Second)
		fs.ReadFile("file1", 1 * time.Second)
		log.Println("reader2 finished")
	}()
}


func main() {
	//ReadAtTheSameTime()
	ReadWhileWrite()
	time.Sleep(10 * time.Second)
}
```

运行结果：

```golang
2019/04/29 00:34:07 Write success: file file1 contents: new content
2019/04/29 00:34:07 reader1 finished
2019/04/29 00:34:08 Read success: file file1 contents: new content
2019/04/29 00:34:08 reader2 finished
```

运行结果显示，当两个线程中存在写操作时，两线程将无法并行运行，相反，而是分别独占文件系统。先启动的线程将先被完成。


#### 2.1.2线程间通信与channel
变量的线程安全问题，面对并解决的是多线程访问公共资源导致的竞争问题。这些公共资源变量对访问它的多个线程而言，往往是“全局可见”的，各线程都能访问到。但是各线程内的声明的变量，则对于其他线程而言不可见。因此，当线程间存在通信需求需要其他机制达成。

当然，我们可以延续上一节的思路，开辟一块“共享内存”这样的公共资源的方式，实现线程间的数据交互。但是golang为我们提供了更好的方案。那便是channel。











### 2.2线程间的协调问题







### 2.3并发程度控制问题与线程池








图片示例：
![Image text](https://github.com/sovietak47/go-doc/blob/master/%E5%A4%9A%E7%BA%BF%E7%A8%8B%20resource/Mutex.png)



  



### 参考资料
> https://mp.weixin.qq.com/s?__biz=MzA4NjgwMDQ0OA==&mid=404961832&idx=1&sn=ff915a35c9d83ed8e5d153496c9298d2&pass_ticket=%2BrMy18QMJX0zmqAnofQvNtn0nis72v3CSdAEqaGbjEKMYYECVCJl3UMx3Pipz%2F%2BS

> 

