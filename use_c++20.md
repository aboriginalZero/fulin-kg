* class FollowerManager

* class SessionFollower

  跟 SessionMaster 之间做 keepalive 



需要搞懂在 meta1 中他们的作用是什么



他两职能分别是啥？





std::lock_guard 为什么禁用拷贝？

* 如果允许拷贝，可能会导致多个 lock_guard 对象管理同一个互斥锁，析构时重复解锁，引发未定义行为；
* 删除拷贝构造函数和赋值运算符，确保锁的独占性。

```c++
namespace std {
    template <class Mutex>
    class lock_guard {
    private:
        Mutex& mutex_; // 引用传入的互斥锁
    public:
        explicit lock_guard(Mutex& m) : mutex_(m) {
            mutex_.lock(); // 构造时加锁
        }

        ~lock_guard() {
            mutex_.unlock(); // 析构时解锁
        }

        // 禁止拷贝和赋值
        lock_guard(const lock_guard&) = delete;
        lock_guard& operator=(const lock_guard&) = delete;
    };
}
```







https://github.com/xiaoweiChen/CXX20-The-Complete-Guide/releases

c++20 使用相关书籍



std::jthread 和 std::thread



std::async 和 std::promise 和  std::future



std::stop_token 和 std::stop_source 可以支持多个停止源，这是原本的 std::atomic< bool> 做不到的

```c++
auto shared_state = std::make_shared<std::stop_source>();
std::stop_source timeout_source(*shared_state);
std::stop_source user_cancel_source(*shared_state);
// 传给子线程使用
std::stop_token stop_token = shared_state->get_token();

void sub_thread_func(std::stop_token token) {
    while (!token.stop_requested()) {
        // do something
    }
}

// 在主线程中可以通过任意一个 source，都可以让子线程结束
timeout_source.request_stop();
user_cancel_source.request_stop();

```





std::string_view 的本质就是一个引用，使用引用的引用并不会带来更多的好处。可以比较好的支持 c style 和 c++ style 的代码

在C++中，只要在函数体内出现了 `co_await` 、`co_return` 和 `co_yield` 这三个操作符中的其中一个，这个函数就成为了协程。

https://zplutor.github.io/2022/03/25/cpp-coroutine-beginner/

重点看怎么把 tokcpp 用起来，他是对这些的封装



便于阅读 c++ 20 代码

```
// c_cpp_properties.json
{
    "configurations": [
        {
            "name": "Linux",
            "includePath": [
                "${default}",
                "${workspaceFolder}/**",
                "/opt/rh/gcc-ztoolset-14/root/usr/include/**",
                "/usr/include/**"
            ],
            "defines": [],
            "compilerPath": "/opt/rh/gcc-ztoolset-14/root/usr/bin/g++",
            "cStandard": "c17",
            "cppStandard": "c++20",
            "intelliSenseMode": "linux-gcc-x64"
        }
    ],
    "version": 4
}
```



