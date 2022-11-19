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

* 测试Threadqueue
```
threadsafe_queue<int> sq;
// 第六章
std::future<void> furture = std::async([&](){
    int i = 0;
    while (true) {
        sq.push(i++);
        std::this_thread::sleep_for(std::chrono::milliseconds(500));
    }
});

std::future<void> furture1 = std::async([&](){
    int i = 0;
    while (true) {
        cout<<"thradId:"<<std::this_thread::get_id()<<" "<<*sq.wait_and_pop()<<endl;
    }
});

while (true)
{
    
    cout<<"thradId:"<<std::this_thread::get_id()<<" "<<*sq.wait_and_pop()<<endl;
}
```

## 线程安全的列表
```
template<typename T>
class threadsafe_list
{
  struct node  // 1
  {
    std::mutex m;
    std::shared_ptr<T> data;
    std::unique_ptr<node> next;
    node():  // 2
      next()
    {}

    node(T const& value):  // 3
      data(std::make_shared<T>(value))
    {}
  };

  node head;

public:
  threadsafe_list()
  {}

  ~threadsafe_list()
  {
    remove_if([](node const&){return true;});
  }

  threadsafe_list(threadsafe_list const& other)=delete;
  threadsafe_list& operator=(threadsafe_list const& other)=delete;

  void push_front(T const& value)
  {
    std::unique_ptr<node> new_node(new node(value));  // 4
    std::lock_guard<std::mutex> lk(head.m);
    new_node->next=std::move(head.next);  // 5
    head.next=std::move(new_node);  // 6
  }

  template<typename Function>
  void for_each(Function f)  // 7
  {
    node* current=&head;
    std::unique_lock<std::mutex> lk(head.m);  // 8
    while(node* const next=current->next.get())  // 9
    {
      std::unique_lock<std::mutex> next_lk(next->m);  // 10
      lk.unlock();  // 11
      f(*next->data);  // 12
      current=next;
      lk=std::move(next_lk);  // 13
    }
  }

  template<typename Predicate>
  std::shared_ptr<T> find_first_if(Predicate p)  // 14
  {
    node* current=&head;
    std::unique_lock<std::mutex> lk(head.m);
    while(node* const next=current->next.get())
    {
      std::unique_lock<std::mutex> next_lk(next->m);
      lk.unlock();
      if(p(*next->data))  // 15
      {
         return next->data;  // 16
      }
      current=next;
      lk=std::move(next_lk);
    }
    return std::shared_ptr<T>();
  }

  template<typename Predicate>
  void remove_if(Predicate p)  // 17
  {
    node* current=&head;
    std::unique_lock<std::mutex> lk(head.m);
    while(node* const next=current->next.get())
    {
      std::unique_lock<std::mutex> next_lk(next->m);
      if(p(*next->data))  // 18
      {
        std::unique_ptr<node> old_next=std::move(current->next);
        current->next=std::move(next->next);
        next_lk.unlock();
      }  // 20
      else
      {
        lk.unlock();  // 21
        current=next;
        lk=std::move(next_lk);
      }
    }
  }
};
```

* 测试代码
```
threadsafe_list<int> sList;

std::future<void> furture = std::async([&](){
    int i = 0;
    while (true) {
        sList.push_front(i++);
        std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    }
});

while (true)
{
   auto ptr = sList.find_first_if([](int a){return a % 10 == 0;});
   if(ptr){
      cout<<"find_first_if:"<<*ptr<<endl;
   }
   cout<<"for_each:";
   sList.for_each([](int a){
     cout<<a<<" ";
   });
   cout<<endl;
  std::this_thread::sleep_for(std::chrono::seconds(1));
}

```
## 无锁同步
```
std::atomic_flag flag(0);
std::function<void()> func = [&]()->void{
   while(flag.test_and_set(std::memory_order_acquire)){
      flag.clear(std::memory_order_release);
      std::this_thread::sleep_for(std::chrono::milliseconds(100));
   }
   cout<<endl<<std::this_thread::get_id()<<endl;
};
vector<std::future<void>> vect;
for(int i = 0;i < 10;i++){
  vect.push_back(std::async(func));
}
while(true);
```

## 并行版本累加
```
template <typename Iterator, typename T>
T parallel_accumulate(Iterator first, Iterator last, T init) {
    unsigned long const length = std::distance(first, last); // 1
    unsigned long const max_chunk_size = 25;
    if (length <= max_chunk_size) {
        return std::accumulate(first, last, init); // 2
    } else {
        Iterator mid_point = first;
        std::advance(mid_point, length / 2); // 3
        std::future<T> first_half_result =
            std::async(parallel_accumulate<Iterator, T>, // 4
                       first, mid_point, init);
        T second_half_result = parallel_accumulate(mid_point, last, T()); // 5
        return first_half_result.get() + second_half_result;              // 6
    }
}
```

* 测试代码
```
   int n = 1000000;
    vector<int> vect(n,0);
    mt19937 gen{random_device {}()};
    uniform_int_distribution<int> dis;    
    int idx = 0;
    while (n>0) {
      vect[idx++] = gen() % 1000;
      n--;
    }

    std::chrono::milliseconds ms=std::chrono::duration_cast<std::chrono::milliseconds>(std::chrono::system_clock::now().time_since_epoch());
    cout<<"begin: "<<ms.count()<<endl;
    parallel_accumulate(vect.begin(), vect.end(), 0l);
    std::chrono::milliseconds ms1=std::chrono::duration_cast<std::chrono::milliseconds>(std::chrono::system_clock::now().time_since_epoch());
    cout<<"end: "<<ms1.count()-ms.count()<<endl;

```