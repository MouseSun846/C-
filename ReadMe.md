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
