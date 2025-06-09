用 lambda 的时候指明返回值，想表示没有返回值时可以用 std::optional，这样可以多带一个含义

```
[&, this](auto batch_pids) -> std::optional<pid_t> {
	if (!stopped_) {
			AddToWaitingRecoverIfNecessary(batch_pids, now_ms, false, maintenance_chunk_recover_enabled, &cancel_migrate_cmds);                                              
      return std::nullopt;
  } else {
      return kPExtentIdStart;
  }
}
```



什么时候该用 && 形参传入？明确知道变量声明周期时，这里会给到生命周期足够长的类成员变量

```
RepositionDistributedCmdSlot RecoverManager::GetRepositionDistributedCmdSlot(bool is_migrate) {
    cmd_count_map_t cap, perf;
    // 从 chunk table 中的这次赋值拷贝没法省
    context_->chunk_table->GetRepositionUsedCmdSlots(&cap, &perf);
    // 但是这里可以通过右值传入来避免重新构造
    return RepositionDistributedCmdSlot(std::move(perf), std::move(cap), is_migrate, config_);
}

RepositionDistributedCmdSlot::RepositionDistributedCmdSlot(cmd_count_map_t&& perf_slots, cmd_count_map_t&& cap_slots, bool is_migrate, const RepositionConfigManager& config) {
    std::swap(cap_avail_cmd_slots_, cap_slots);
    std::swap(perf_avail_cmd_slots_, perf_slots);
}
```



一个函数内部有时需要加锁，有时不需要，可以用 unique_ptr 来管理锁

```
void func(bool use_cache) {
		std::unique_ptr<RepositionLockGuard> l;
		if (UNLIKELY(!use_cache)) {
        l.reset(new RepositionLockGuard(&mutex_, source_location::current().line()));
    }
    // 业务逻辑
}
```



往 std::vector 插入多个元素时，可以避免不必要的拷贝开销

```
vec.reserve(big_size);
LOOP(big_size) {
    auto& item = vec.emplace_back();
    item.pid = i;
}
```



一个函数中同时处理多个类型不同的参数

```
template <typename Container>
Status ChunkTable::GetStoragePoolSpace(Container* chunks) {
    auto fn = [&] (auto& entries) {
        FOREACH (entries, it) {
            Chunk* entry = nullptr;
            // 要么传入 map，需要自己取 value，要么直接传入 value
            if constexpr (std::is_same_v(std::remove_cvref_t<decltype(*it)>, ChunkTableEntry*)) {
                entry = *it;
            } else {
                entry = it->second;
            }
            
            if constexpr (std::is_same_v(Container, absl::flat_hash_map<cid_t, ChunkSpaceInfo>>)) {
                (*chunks)[entry->id()] = space_info;
            } else if constexpr (std::is_same_v(Container, std::vector<ChunkSpaceInfo>)) {
                chunks->push_back(space_info);
            } else {
                static_assert(sizeof(Container) == 0, "unsupported container type")
            }
        }
    };
    
    if (sp_id.empty()) {
        fn(chunk_table_);
    } else {        
        StoragePoolEntry* spe = GetStoragePoolEntry(sp_id);
        fn(spe->chunk_entries);
    }
}
```



当明确一个类的成员方法只需要操作静态成员变量或者不依赖类的具体对象时，可以将其声明成 static，能稍微提高方法调用性能。（静态方法是类的全局函数，滥用将破坏封装性，应优先考虑非静态方法，除非有明确理由）

能提高性能的原因：

1. 避免对象创建开销。静态方法通过类名直接调用（如 ClassName::method()），无需创建对象实例，节省了构造和析构的开销；
2. 省去 this 指针传递。在高频调用场景下如工具函数或频繁查询，省去 this 指针传递可减少由于寄存器使用、栈帧操作带来的纳秒级微小开销；
3. 提高内联机会。静态方法通常是类级别的简单函数，编译器更容易内联，直接嵌入调用点，可以消除调用栈的创建和销毁。



给定 key，从 hash map 拿到 value，key 不存在的话，添加默认值并返回

```
value = map.try_emplace(key, specified_value).first->second;
```

给定 Key，从 hash map 中拿到 value，key 不存在的话，返回 nullptr（如果用 contains + at 要查两次）

```
auto it = map.find(key);
return it == map.end() ? nullptr : it->second;
```





用 virtual 菱形继承，避免拥有多份 AccessSessionCommon 中的变量

```
class AccessManagerCommon  {
  public:
    // 析构函数也用 virtual 声明
    virtual ~AccessManagerCommon() = default;
  
  protected:
    // 只希望被子类访问方法用 protected 声明
    AccessSession* GetChunkSession(cid_t cid);
};

class AccessManagerForExtentManager : virtual public AccessManagerCommon {
  protected:
    // 希望不同之类有不同的实现方式，声明为虚函数
    virtual void OnAccessSessionExpired(const std::string& uuid) = 0;
    
    // 如果要兼容多种版本的 rpc requst，可以用 std::variant
    // use ptr to avoid memory copy (PExtentInfoPBArray may copy when used to construct std::variant)
    using PExtentInfosVariant = std::variant<const PExtentInfoPBArray*, const PExtentInfoArrayView<ReportPExtentInfoV2>*, const PExtentInfoArrayView<ReportPExtentInfoV3>*>;
    using cb_info_t = PhysicalExtentTable::cb_info_t;
    using cb_entry_info_t = PhysicalExtentTable::cb_entry_info_t;
    virtual void HandlePExtentInfosFromDataReport(const PExtentInfosVariant& pextent_infos, uint64_t now_ms, cid_t cid, const std::unordered_set<pid_t>& recover_pids, const cb_info_t& update_gc_cb, const cb_entry_info_t& update_provision_cb, const cb_info_t& update_treplica_space_cb, int begin, int end) = 0;
};

class AccessManagerForChunkManager : virtual public AccessManagerCommon {
  public:
    // 声明友元类，TESTLens 可以访问这个类的 private 变量
    friend class TESTLens;
}

class AccessManager : public AccessManagerForExtentManager,
                      public AccessManagerForChunkManager {
  public:
    explicit AccessManager(MetaContext* context);
    virtual ~AccessManager();
    
    // 要注意调用之类方法的顺序
    void SessionExpiredCb(MasterSessionBase* session) {
        LockGuard l(&lock_);
        AccessManagerForChunkManager::SessionExpiredCb(session);
        AccessManagerForExtentManager::SessionExpiredCb(session);
        // this will erase session from sessions_, so it must be called at last.
        AccessManagerCommon::SessionExpiredCb(session);
    }
    
    // 返回值不是局部对象、一定不为 nullptr 时，可以返回引用以避免拷贝
    PhysicalExtentTable& GetPhysicalExtentTable() { 
    		return *context_->pextent_table; 
    }
    
  private:
    void HandlePExtentInfosFromDataReport(const PExtentInfosVariant& pextent_infos, uint64_t now_ms, cid_t cid, const std::unordered_set<pid_t>& recover_pids, const cb_info_t& update_gc_cb, const cb_entry_info_t& update_provision_cb, const cb_info_t& update_treplica_space_cb, int begin, int end) override {
        std::visit([&](auto&& pextent_infos) {
            context_->pextent_table->HandlePExtentInfosFromDataReport(*pextent_infos, now_ms, cid, recover_pids, update_gc_cb, update_provision_cb,                                                                     update_treplica_space_cb, begin, end); }, pextent_infos);
    }
}
```









* class FollowerManager

* class SessionFollower

  把 nfs 接入点和 set/remove/list session item 的逻辑借助 session client 转发到 SessionMaster 上s

  跟 SessionMaster 之间做 keepalive，把 



Session Master



可以整理下 meta1 中 session init / create / keepalive / jeopardy / expired 的流程，有助于未来排查问题。



meta2 中的 mgr 之间的 session 机制里：

1. 不需要有 jeopardy
1. 可以有高低两种频率的心跳



也搞一个 SessionFollowerBase，可以继承出 AccessSessionFollower 和 meta2SessionFollower





只是因为历史原因才搞了 SessionService 和 DataReportService 两个 pb service，对应 2 个 SessionMaster 和 DataReportMaster



要考虑旧 chunk 给新 meta 发送 KeepAlive 或者 CreateSession，此时还会走旧的 rpc 接口，所以还是得继承



access data report 和 chunk data report 的区别是啥？

1. access data report 指的是 access 汇报 reposition perf（之后汇聚到 status server），iscsi/nvmf conns（之后更新到 meta 视角的 session active iscsi / nvmf conn）
2. chunk data report 指的是 lsm 分批汇报 pextent info，之后更新 pextent table entry







access mgr 传入的回调，也就是 access mgr 定义了 session master 在 Session init / create / check leader /  expired 时的具体行为。



meta1 中 create session 流程

1. access handler 执行 CreateSession
2. session follower 执行 CreateSession，实际上转发给 session client
3. session client 执行 CreateSession，简单把 session item 填充了，通过 socket 转发给 session master
4. session master 执行 CreateSession，包含 session_initialize_cb_ 和 session_created_cb_，而这对应 access mgr 的 SessionInitializeCb 和 SessionCreatedCb
5. session follower 拿到 session master 的结果后，会执行 session_created_cb，这对应 access handler 的 session handler 的 SessionCreatedCb，在这里会做能力协商以及 data report 的 start pid 和 report size
6. access handler 的 SessionCreatedCb 中还有一个 node server 注册的 RegisterSessionCreateCb，他的作用是让 iscsi session 进入 alive 状态

补充说明下：

1. zbs chunk main() --> RunChunkServerMain() --> StartChunkServer() --> ChunkServer::Start() --> AccessHandler::Start() --> AccessHandler::CreateSession()，所以可以认为 chunk 进程一起来，就会尝试去创建 access session；
2. session follower 有自己独立的线程，创建完 session，就在 KeepAliveLoop 的 while 循环里了



meta1 中





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



