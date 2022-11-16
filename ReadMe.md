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