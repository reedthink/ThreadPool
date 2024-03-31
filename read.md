## 以这个代码为例 https://github.com/progschj/ThreadPool

## 实现思路：
线程池中持有线程和任务队列，使用线程和锁的模型。初始化的时候创建一些线程，之后推入一些任务，每次推入后通过条件变量触发某个阻塞的线程执行。线程池析构的时候`Unblocks all threads currently waiting for *this.`。 线程和任务队列的关系是每个线程都会从任务队列读取没被执行的任务，然后执行。

## 数据结构:
```cpp
private:
    // need to keep track of threads so we can join them
    std::vector< std::thread > workers;
    // the task queue
    std::queue< std::function<void()> > tasks;
    
    // synchronization
    std::mutex queue_mutex;
    std::condition_variable condition;
    bool stop;
```
可以看到一个vector存放线程，一个队列存放任务队列，一把锁控制并发，一个条件变量，一个布尔变量控制整个线程池的任务执行与否

## 构造函数:
```cpp
// the constructor just launches some amount of workers
inline ThreadPool::ThreadPool(size_t threads)
    :   stop(false)
{
    for(size_t i = 0;i<threads;++i)
        workers.emplace_back(
            [this]
            {
                for(;;)
                {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(this->queue_mutex);
                        this->condition.wait(lock,
                            [this]{ return this->stop || !this->tasks.empty(); });
                        if(this->stop && this->tasks.empty())
                            return;
                        task = std::move(this->tasks.front());
                        this->tasks.pop();
                    }
                    task();
                }
            }
        );
}
```
1. `emplace_back` 与`push_back`不同，`emplace_back` 直接在`vector`尾部构造元素，省了一次额外复制或移动操作 [emplace_back参考](https://zh.cppreference.com/w/cpp/container/vector/emplace_back)
2. 传入 `lambda` 表达式，函数内容是 启动`threads`个线程，每个线程都循环读取任务队列的任务，直到队列任务被执行完或者 `stop` 被置为 `true` 。
3. 通过条件变量来触发线程运行，值得一提的是为了检测虚假唤醒，加入了条件判断 如果没有停止或者还有任务，就运行线程。
4. 通过`move`转移生命周期控制权到`task`，之后执行`task`

## 析构函数：
```cpp
// the destructor joins all threads
inline ThreadPool::~ThreadPool()
{
    {
        std::unique_lock<std::mutex> lock(queue_mutex);
        stop = true;
    }
    condition.notify_all();
    for(std::thread &worker: workers)
        worker.join();
}
```

1. 第一个作用域，首先获取锁，然后`stop`置为`true`
2. 条件变量通知所有线程
3. 所有线程`join`。析构的时候阻塞，等待所有线程执行完。


## enqueue 函数:
```cpp
// add new work item to the pool
template<class F, class... Args>
auto ThreadPool::enqueue(F&& f, Args&&... args) 
    -> std::future<typename std::result_of<F(Args...)>::type>
{
    using return_type = typename std::result_of<F(Args...)>::type;

    auto task = std::make_shared< std::packaged_task<return_type()> >(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)
        );
        
    std::future<return_type> res = task->get_future();
    {
        std::unique_lock<std::mutex> lock(queue_mutex);

        // don't allow enqueueing after stopping the pool
        if(stop)
            throw std::runtime_error("enqueue on stopped ThreadPool");

        tasks.emplace([task](){ (*task)(); });
    }
    condition.notify_one();
    return res;
}
```

1. 定义模板, 关于模板不是很懂
2. `emplace`推入新元素到队列结尾。原位构造元素，即不进行移动或复制操作
3. 为什么用一个`lambda`函数包一层而不是直接把 `(*task)()` 推进去?

    1. 首先明确`task`是什么类型？是`shared_ptr`, 指向对象是函数。
    2. 因为加入`tasks`的元素`function<void()`，所以必须用匿名函数包一层（`lambda`实际是`class`, 符合[`function<void()`的要求](https://zh.cppreference.com/w/cpp/utility/functional/function)）
4. 通知某个线程运行