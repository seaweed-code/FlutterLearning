## C++多线程

### 如何同时锁住多个互斥量？

```c++
///方法1
 void swap(X& lhs, X& rhs)
  {
    std::lock(lhs.m,rhs.m); // 同时持有 两个std::mutex,要么同时成功，要么同时失败
    std::lock_guard<std::mutex> lock_a(lhs.m,std::adopt_lock);
    std::lock_guard<std::mutex> lock_b(rhs.m,std::adopt_lock);
   
  }
};

///方法2
void swap(X& lhs, X& rhs)
{
  if(&lhs==&rhs)
    return;
  std::scoped_lock guard(lhs.m,rhs.m); // C++17以后
  swap(lhs.some_detail,rhs.some_detail);
}

///方法3
friend void swap(X& lhs, X& rhs)
  {
    std::unique_lock<std::mutex> lock_a(lhs.m,std::defer_lock);
std::unique_lock<std::mutex> lock_b(rhs.m,std::defer_lock); //std::defer_lock 延迟加锁
std::lock(lock_a,lock_b); // 2 互斥量在这里上锁
    swap(lhs.some_detail,rhs.some_detail);
  }
```

### unique_lock更为灵活，可移动不可复制，可在需要时随时加锁解锁

```c++
void get_and_process_data()
{
std::unique_lock<std::mutex> my_lock(the_mutex);
some_class data_to_process=get_next_data_chunk(); 
  my_lock.unlock(); // 1 不要让锁住的互斥量越过process()函数的调用 result_type result=process(data_to_process);
my_lock.lock(); // 2 为了写入数据，对互斥量再次上锁 write_result(data_to_process,result);
}
```

### std::call_once()多线程保证执行一次

### C++11标准，保证静态局部变量的初始化是线程安全的

```c++
my_class& get_my_class_instance() {
    static my_class instance;  // 静态局部变量
    return instance;
}
```

### C++读写锁shared_mutex，实现多读单写 

C++11可以使用boost库，标准库提供下面两种：

-  std::shared_mutex    C++17
-  std::shared_timed_mutex  C++14

```C++
#include <map>
#include <string>
#include <mutex>
#include <shared_mutex>
class dns_entry;
class dns_cache
{
  std::map<std::string,dns_entry> entries;
  mutable std::shared_mutex entry_mutex;
public:
  dns_entry find_entry(std::string const& domain) const
  {
    std::shared_lock<std::shared_mutex> lk(entry_mutex);  //读锁，可以并发读
    std::map<std::string,dns_entry>::const_iterator const it=
       entries.find(domain);
    return (it==entries.end())?dns_entry():it->second;
}
  void update_or_add_entry(std::string const& domain,
                           dns_entry const& dns_details)
  {
    std::lock_guard<std::shared_mutex> lk(entry_mutex);  //写锁，独占锁，其他线程不可读和写
    entries[domain]=dns_details;
} };
```

### 递归锁std::recursive_mutex

允许同一个线程多次加锁，使用方法与std::mutex一样。

### 条件变量

如果线程需要等待某个条件满足，该如何实现呢？如果用互斥量的话，只能这样实现：

```c++
bool flag;
std::mutex m;
void wait_for_flag()
{
  std::unique_lock<std::mutex> lk(m);
  while(!flag){ ///循环判断条件是否满足，不满足，则先释放锁，休眠一会，睡醒后再加锁，再次查看条件是否满足？
		lk.unlock(); // 1 解锁互斥量
		std::this_thread::sleep_for(std::chrono::milliseconds(100)); // 2 休眠100ms
		lk.lock(); // 3 再锁互斥量 }
}
```

这段代码大概实现了等待某个条件的功能，但是明显存在很大问题：

1、采用轮训的方式判断条件是否满足，很浪费时间

2、虽然睡眠了一会，不会一直占用CPU，但是睡眠多久不好确定，睡眠久了等的太长，睡眠少了，条件还没满足查的又太勤了

这个时候就需要**条件变量**了，不需要轮训，条件不满足则释放锁休眠，一旦条件满足会被自动唤醒，唤醒后会会自动尝试加锁，逻辑与上面一样，但是效率却大大增加了，两者都需要与一个互斥量一起才能工作(互斥量是 为了同步)

- std::condition_variable 仅限于与 std::mutex 一起使用
- std::condition_variable_any  可以和任何满足最低标准的互斥量一 起工作，更加通用，这就可能从 体积、性能，以及系统资源的使用方面产生额外的开销，

```c++
std::mutex mut;
std::queue<data_chunk> data_queue;  // 1
std::condition_variable data_cond;
void data_preparation_thread()
{
  while(more_data_to_prepare())
  {
    data_chunk const data=prepare_data();
    std::lock_guard<std::mutex> lk(mut);
    data_queue.push(data);  // 2
    data_cond.notify_one();  //
} }
void data_processing_thread()
{
  while(true)
  {
    std::unique_lock<std::mutex> lk(mut);  // 4
    data_cond.wait(
         lk,[]{return !data_queue.empty();});  // 5
    data_chunk data=data_queue.front();
    data_queue.pop();
    lk.unlock();  // 6
    process(data);
    if(is_last_chunk(data))
        break; 
  }
}
```

条件变量可能会出现“[伪唤醒](https://en.cppreference.com/w/cpp/thread/condition_variable/wait)spurious wakeups”，也就是并不是由于条件变量满足而唤醒，而是其他原因导致的（操作系统调度等）。

### 什么是伪唤醒（Spurious Wakeup）？

**伪唤醒**是指等待线程从 `std::condition_variable::wait()`、`std::condition_variable::wait_for()` 或 `std::condition_variable::wait_until()` 等调用中返回，但实际上并没有满足被等待的条件（没有其他线程调用 `notify_one` 或 `notify_all`）。这是底层实现机制（操作系统或硬件）导致的一种现象，尤其是在多核 CPU 或虚拟化环境中。

在一些操作系统或硬件实现中，线程的调度和内核的某些优化可能会导致这样的伪唤醒。这在 POSIX 和 Windows 上的条件变量机制中都是可能的。

伪唤醒可以由多种原因引起，主要原因包括：

1. **硬件和操作系统的优化**：某些操作系统的实现可能会在没有明确原因的情况下唤醒等待的线程，以优化资源管理或进行调度。
2. **CPU 调度切换**：线程等待时，CPU 可能发生切换，这可能导致线程被意外唤醒。

### 如何应对伪唤醒？

为了应对伪唤醒，标准的做法是**始终使用一个“条件判断”来循环检查条件**。在等待条件变量时，通常会用 `while`循环来反复检查条件是否满足，确保即使出现伪唤醒，线程不会错误地继续执行。例如：

```c++
bool ready = false;

void wait_for_ready() {
    std::unique_lock<std::mutex> lock(mtx);
    // 使用 while 循环来避免伪唤醒
    while (!ready) {///需要加while循环避免伪唤醒
        cv.wait(lock);  // 即使伪唤醒，也会再次检查 ready 的状态
    }
    std::cout << "Thread proceeded after notification or condition met." << std::endl;
}

```

[参考原文：->](https://en.cppreference.com/w/cpp/thread/condition_variable/wait)

```c++
//wait函数逻辑如下：
1、自动解锁lock.unlock()；并使当前线程休眠直到 条件变量调用了notify_all() or notify_one() or 伪唤醒
2、唤醒后，自动尝试加锁lock.lock();
void wait( std::unique_lock<std::mutex>& lock );//等待条件满足，可能出现伪唤醒,使用需要外面加循环


不会出现伪唤醒，循环判断条件是否满足
template< class Predicate >
void wait( std::unique_lock<std::mutex>& lock, Predicate pred );
伪代码相当于：
  while (!pred())
    wait(lock);.
```

⚠️：wait还有其他系列函数，总之带有参数pred都不会出现伪唤醒，没有的则会。



### std::future 一个线程等待另一个线程的返回值

C++标准库提供了2种，仿照了std::unique_ptr 和 std::shared_ptr

- std::future<> 独立期望值，1个线程等待另一个线程的返回值
-  std::shared_future<> 共享期望值(shared futures)，多个线程等待一个线程的返回值

如果需要在多个线程中并发访问std::future的返回值，必须使用互斥量进行同步，但是使用shared_future则不需要。

##### 1、std::async函数，注意不是类

当然也可以使用条件变量实现，但是较为繁琐，使用std::async方法如下：

```c++
#include <future>
#include <iostream>
int find_the_answer_to_ltuae();
void do_other_stuff();
int main()
{
  auto the_answer=std::async(find_the_answer_to_ltuae);//异步调用，返回后会notify_one等待线程
  do_other_stuff();///不等其返回，先做别的事
  std::cout<<"The answer is "<<the_answer.get()<<std::endl;///阻塞等待其返回结果
}
```

尽管可以使用条件变量实现相同功能，但是明显使用future代码更为可读。

```c++
#include <string>
#include <future>
struct X
{
  void foo(int,std::string const&);
  std::string bar(std::string const&);
};
X x;
auto f1=std::async(&X::foo,&x,42,"hello"); // 调用p->foo(42, "hello")，p是指向x的指针
auto f2=std::async(&X::bar,x,"goodbye"); // 调用 tmpx.bar("goodbye")， tmpx是x的拷贝副本
struct Y
{
  double operator()(double);
};
Y y;
auto f3=std::async(Y(),3.141); // 调用tmpy(3.141)，tmpy通过Y的移动 构造函数得到
auto f4=std::async(std::ref(y),2.718); // 调用y(2.718)
X baz(X&);
std::async(baz,std::ref(x)); // 调用baz(x)
class move_only
{
public:
  move_only();
  move_only(move_only&&)
  move_only(move_only const&) = delete;
  move_only& operator=(move_only&&);
  move_only& operator=(move_only const&) = delete;
  void operator()();
};
auto f5=std::async(move_only()); // 调用tmp()，tmp是通过 std::move(move_only())构造得到
```

##### 注意⚠️：std::async返回值如果不用变量存起来，则结果相当于同步

当 `std::future` 对象被析构时，它会确保共享状态被正确处理。这通常包括：

- 等待任务完成（如果尚未完成）。
- 清理共享状态，释放资源。

因为返回值为临时值future，会马上被析构，结果就相当于同步操作，例如：

```c++
#include <iostream>
#include <future>
#include <thread>
#include <chrono>

void async_task() {
    std::this_thread::sleep_for(std::chrono::seconds(5));
    std::cout << "Task completed asynchronously!" << std::endl;
}

int main() {
    // 使用 std::launch::async 启动异步任务
    std::async(std::launch::async, async_task);///同步执行
    {
        auto c= std::async(std::launch::async, async_task);  // 没有保存 future 对象
    }/////同步执行
    std::cout << "Main thread continues..." << std::endl;
    std::this_thread::sleep_for(std::chrono::seconds(3));
    return 0;
}

```

##### 2、 std::packaged_task类

##### 3、std::promise类
