## C++多线程并发

1. ### 多线程基础：std::thread

   ```c++
   void thread_task() {
       std::cout << "hello thread" << std::endl;
   }
   
   int main(int argc, const char *argv[]){
       std::thread t(thread_task);///thread可以move，不可以copy
     
       t.join();//必须在t析构之前调用 join，或者detach
       return 0;
   } 
   ```

   

2. ### std::packaged_task

   ```c++
   int main(int argc, char * argv[]){
       using namespace std::literals;
   
       auto && task = std::packaged_task<int(void)>([]
       {
           std::cout << "running\n";
   
           // 模拟这个函数执行时间比较长
           std::this_thread::sleep_for(2s);
   
           std::cout << "end\n";
           return 42;
       });
   
       // 提前获取 future，等下 task 会被 std::move，之后就无法使用了
       auto && future = task.get_future();
   
       auto t1 = std::thread(std::move(task));
   
       std::cout << future.get() << std::endl;
   
       t1.join();
       return 0;
   }
   ```

   

3. ### std::promise 和 std::future

   1、promise生命周期，只要promise调用set后，就可以被析构，即便future此时还未读取；反之，如果promise还没有set就被析构了，`std::promise` 会自动设置其共享状态为异常。这样，任何试图从关联的 `std::future` 获取值的代码都会收到一个异常 `std::future_error`，其错误代码是 `std::future_errc::broken_promise`。这是为了通知 `std::future` 的使用者，承诺的值不会被设置。

   2、future生命周期，

   ```c++
   int main(int argc, char * argv[]){
       auto promise = std::promise<void>();
   
       auto t1 = std::thread([&promise]
       {
           std::cout << "running\n";
   
           // 模拟这个函数执行时间比较长
           std::this_thread::sleep_for(2s);
   
           // 唤醒正在等待数据的线程
           promise.set_value();
   
           std::cout << "end\n";
       });
   
       std::cout << "before\n";
       // 等待条件触发
       promise.get_future().get();
       std::cout << "after\n";
   
       t1.join();
       return 0;
   }
   ```

   future析构会阻塞：

   ```c++
   std::async(std::launch::async,[](){
               for (int i=0;i<100; i++) {
                   std::this_thread::sleep_for(std::chrono::seconds(5));
                   std::cout << "wake up---" << i << std::endl;
               }
   });//1、阻塞
           
   std::cout << "end---" << std::endl;///2、再执行
   ```

   析构时确实会阻塞：

   ```c++
   {
        auto future = std::async(std::launch::async,[](){
               for (int i=0;i<100; i++) {
                   std::this_thread::sleep_for(std::chrono::seconds(5));
                   std::cout << "wake up---" << i << std::endl;
               }
        });
           
        std::cout << "end---" << std::endl;///1、先输出
   }//2、析构时，会阻塞
       
   std::cout << "--------" << std::endl;///3、在输出
   
   /////////内部实现：
   template <class _Rp, class _Fp>
   void __async_assoc_state<_Rp, _Fp>::__on_zero_shared() _NOEXCEPT
   {
       this->wait();///会阻塞等待
       base::__on_zero_shared();
   }
   ```

   future析构时不会阻塞？

   ```c++
   std::promise<int> prom;
    auto th = std::thread([&](){
           for (int i=0;i<100; i++) {
               std::this_thread::sleep_for(std::chrono::seconds(5));
               std::cout << "wake up---" << i << std::endl;
           }
           prom.set_value(90);
   });
       
   {
       std::future<int> fut = prom.get_future();
       std::cout << "end future---" << std::endl;
   }//future析构不会阻塞？
      
   std::cout << "---" << std::endl;
       
   th.join();
   
   ///内部实现：
   template <class _Rp>
   void __assoc_state<_Rp>::__on_zero_shared() _NOEXCEPT
   {
       if (this->__state_ & base::__constructed)
           reinterpret_cast<_Rp*>(&__value_)->~_Rp();
       delete this;
   }
   ```

   

1. ### std::shared_future

   一个std::future对象，只能调用get一次，而多个shared_future可以共享一个future，每个shared_future都可以调用get无限次。

   

2. ### std::async

   ```c++
   int main(int argc, char * argv[])
   {
       using namespace std::literals;
   
       auto future = std::async([]///返回 future
       {
           std::cout << "running\n";
   
           // 模拟这个函数执行时间比较长
           std::this_thread::sleep_for(2s);
   
           std::cout << "end\n";
           return 42;
       });
   
       std::cout << future.get() << std::endl;
   
       return 0;
   }
   ```

   

3. ### std::call_once，保证只调用一次

4. ### 读写锁

5. ### **std::condition_variable**

   多线程并发打印ABCABC....

   ```c++
   int main(int argc, const char * argv[]) {
       
       std::mutex mutex;
       std::condition_variable condition;
       char c = 'C';
       
       std::thread t1{[&](){
          
           for (int i=0; i<100; i++)  {
               std::unique_lock lock(mutex);
               
               condition.wait(lock,[&](){////线程等待c变成‘C’就打印A，否则阻塞
                   return  'C' == c;
               });
               
               if ('C' == c) {
                   std::cout << "\n";
               }
               c = 'A';
               std::cout << c;
   
               condition.notify_all();///只唤醒一个线程，不确定是哪个？可能有问题
           }
       }};
       
       std::thread t2{[&](){
           for (int i=0; i<100; i++) {
               
                   std::unique_lock lock(mutex);
                   
                   condition.wait(lock,[&](){////线程等待c变成‘A’就打印B，否则阻塞
                       return   'A' == c;
                   });
                   
                   c = 'B';
                   std::cout << c;
               
               condition.notify_all();
           }
       }};
       
       std::thread t3{[&](){
           for (int i=0; i<100; i++) {
                   std::unique_lock lock(mutex);
                   
                   condition.wait(lock,[&](){////线程等待c变成‘B’就打印C，否则阻塞
                       return   'B' == c;
                   });
                   c = 'C';
                   std::cout << c;
               
               condition.notify_all();
           }
       }};
       
       t1.join();
       t2.join();
       t3.join();
       std::cout << "\nend\n";
       return 0;
   }
   
   ```

   

6. ### 线程同步，互斥锁、递归锁、条件锁

   ### std::lock_guard

   ### std::unique_lock

7. ### 原子操作，Lock Free

Memory Order选项区别：

| memory_order_relaxed | 没有任何同步或顺序约束，仅保证操作的原子性，性能高【允许编译器指令重排、CPU乱序执行以提高并发性能】 |
| -------------------- | :----------------------------------------------------------- |
| memory_order_consume | 与memory_order_acquire类似，区别是：<br />当前线程的本条指令之后的依赖于当前数据的Read 操作，【编译器、CPU】必须保证不能放到当前指令的前面执行 |
| memory_order_acquire | 1、当前线程在`load()`之后的所有读写操作，不允许被移动到这个`load()`的前面<br />2、使用`Memory Barrier`指令强制刷新当前核内部的`[invalidate queue](https://www.cnblogs.com/ishen/p/13200838.html)，这样，其他核对同一个原子变量的store( release )之前的全部写入操作，在当前线程都可见 |
| memory_order_release | 1、当前线程在`store()`之前的所有读写操作，不允许被移动到这个`store()`的后面 <br />2、当前线程在`store()`之前的所有写入操作，使用`Memory Barrier`指令强制从当前核内部的写入缓存[StoreBuffer](https://www.cnblogs.com/ishen/p/13200838.html) Flush到内存中，这样当其他核`load(acquire)`了同一个原子变量后，都能看到当前线程在store之前的所有写入操作。 |
| memory_order_acq_rel | memory_order_acquire + memory_order_release                  |
| memory_order_seq_cst | 【默认行为，最强的约束】性能略差，是：<br />当前线程的本条指令之前的所有操作（Read+Write）不能放到当前指令的后面执行，且<br />当前线程的本条指令之后的所有操作（Read+Write）不能放到当前指令的前面执行 |



**参考资料：**

- [理解 C++ 的 Memory Order](http://senlinzhan.github.io/2017/12/04/cpp-memory-order/)
- [内存模型与c++中的memory order](https://www.cnblogs.com/ishen/p/13200838.html)
- [cppreference.com - std::memory_order](http://en.cppreference.com/w/cpp/atomic/memory_order)

