## 多线程交替输出字符 A B
```
 std::mutex mtx;
    std::condition_variable dataCond;
    std::thread t1([&](){
        while (true) {
            std::unique_lock<std::mutex> lock(mtx);
            dataCond.notify_all();
            dataCond.wait(lock);
            cout<<"A"<<endl;
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }
    });
    std::thread t2([&](){
        while (true) {
            std::unique_lock<std::mutex> lock(mtx);
            dataCond.notify_all();
            dataCond.wait(lock);
            cout<<"B"<<endl;
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }
    });    
    t1.join();
    t2.join();
```    
## 获取随机数
```
    mt19937 gen{random_device {}()};
    uniform_int_distribution<int> dis;
    while(true){
        cout<<gen<<endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
```    
## furtue使用
```
    // furture使用
    std::future<string> handle = std::async([]()->string{
        cout<<"async threadId:"<<std::this_thread::get_id()<<endl;
        std::this_thread::sleep_for(std::chrono::seconds(3));
        return "async";
    });
    cout<<"main threadId:"<<std::this_thread::get_id()<<endl;
    cout<<handle.get()<<endl;
```    

## packaged_task使用
* 适用与多个线程
```
    // // std::packaged_task
    std::packaged_task<int()> task([]()->int{
        int i = 0;
        while (i<3) {
            cout<<"packaged_task threadId:"<<std::this_thread::get_id()<<" std::packaged_task "<<i++<<endl;
            std::this_thread::sleep_for(std::chrono::seconds(1));
        }
        return i;
    });
    // 执行，如果不执行就一直阻塞
    // task();
    cout<<"main threadId:"<<std::this_thread::get_id()<<endl;
    // 获取返回值
    std::future<int> handle = task.get_future();
    cout<<handle.get()<<endl;
```

## promise使用
```
    // promise使用
    std::promise<std::string> promise;
    cout<<"begin"<<endl;
    // 必须定义async的返回结果
    future<void> f = std::async([&promise](){
        std::this_thread::sleep_for(std::chrono::seconds(5));
        promise.set_value("promise");
    });
    cout<<"main threadId:"<<std::this_thread::get_id()<<endl;
    future<std::string> future = promise.get_future();
    cout<<future.get()<<endl;
```    

## furtue与exception使用
```
    std::future<void> future = std::async([]{
        throw std::exception();
    });
    try {
        future.get();
    } catch (std::exception& e) {
    
    }
    cout<<"test exception"<<endl;
```    

## 线程安全的队列
```
template<typename T>
class threadsafe_queue
{
private:
  mutable std::mutex mut;
  std::queue<std::shared_ptr<T> > data_queue;
  std::condition_variable data_cond;
public:
  threadsafe_queue()
  {}

  void wait_and_pop(T& value)
  {
    std::unique_lock<std::mutex> lk(mut);
    data_cond.wait(lk,[this]{return !data_queue.empty();});
    value=std::move(*data_queue.front());  // 1
    data_queue.pop();
  }

  bool try_pop(T& value)
  {
    std::lock_guard<std::mutex> lk(mut);
    if(data_queue.empty())
      return false;
    value=std::move(*data_queue.front());  // 2
    data_queue.pop();
    return true;
  }

  std::shared_ptr<T> wait_and_pop()
  {
    std::unique_lock<std::mutex> lk(mut);
    data_cond.wait(lk,[this]{return !data_queue.empty();});
    std::shared_ptr<T> res=data_queue.front();  // 3
    data_queue.pop();
    return res;
  }

  std::shared_ptr<T> try_pop()
  {
    std::lock_guard<std::mutex> lk(mut);
    if(data_queue.empty())
      return std::shared_ptr<T>();
    std::shared_ptr<T> res=data_queue.front();  // 4
    data_queue.pop();
    return res;
  }

  void push(T new_value)
  {
    std::shared_ptr<T> data(
    std::make_shared<T>(std::move(new_value)));  // 5
    std::lock_guard<std::mutex> lk(mut);
    data_queue.push(data);
    data_cond.notify_one();
  }

  bool empty() const
  {
    std::lock_guard<std::mutex> lk(mut);
    return data_queue.empty();
  }
};


```