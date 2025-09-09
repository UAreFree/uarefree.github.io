# Spdlog源码解析


spdlog是一个快速的C++日志库。使用有两个版本，headr-only的版本和编译版本。header-only版本只需要把项目include文件夹放到自己项目中即可构建，用起来比较方便。官方给出的特点：很快；header-only 或编译版本；使用 fmt 库实现丰富的格式化功能；异步模式；自定义格式化；多线程/单线程记录器；多种日志目标；日志过滤；支持从argv或环境变量中加载日志级别；回溯支持 - 将调试消息存储在环形缓冲区中并按需显示。

<!--more--> 

spdlog是一个日志库，调用其提供的接口实现日志打印功能，它本身引用到了fmt库，fmt是格式化的库。初级的日志打印的定义是：
```C++
spdlog::info("Positional args are {1} {0}..", "too", "supported");
```
上述函数通过调用spdlog命名空间的info接口，打印结果：
```Bash
[2025-09-04 15:49:34.163] [info] Positional args are supported too..
```
{}是fmt格式的占位符，0、1是后边字符串参数的索引，日志的级别是info，在级别之前打印了时间，以及info字体的颜色是绿色的。spdlog的日志级别有trace、debug、info、warn、error、critical、off七个。可以通过以下函数设置某个日志级别以上（大于等于）的日志进行显示：
```C++
spdlog::set_level(spdlog::level::info);
```
我们通过spdlog::info接口来向下探索，该接口定义在spdlog.h中，说明所有源文件只要include了spdlog.h，即可调用spdlog库所提供的接口。该函数是一个模版函数定义如下：
```C++
template <typename... Args>
inline void info(format_string_t<Args...> fmt, Args &&...args) {
    default_logger_raw()->info(fmt, std::forward<Args>(args)...);
}
```
Args...是可变参数包类型，format_string_t<Args...> fmt是fmt库接口fmt::format_string<Args...>，Args &&...args将可变参数包转为右值用于完美转发。还要需要注意的是spdlog.h中的所有函数都是inline的，这也是为什么spdlog能实现header-only的原因。
default_logger_raw()的函数定义如下：
```C++
SPDLOG_INLINE spdlog::logger *default_logger_raw() {
    return details::registry::instance().get_default_raw();
}
```
该函数使用单例模式，使用static创建全局唯一registry实例，调用get_default_raw()拿到默认logger。然后调用logger类的成员函数info，该函数定义如下，其中log函数是重载函数，支持多种参数类型和数量。不过最终都会调用log_it_函数。
```C++
template <typename... Args>
void info(format_string_t<Args...> fmt, Args &&...args) {
    log(level::info, fmt, std::forward<Args>(args)...);
}
SPDLOG_INLINE void logger::log_it_(const spdlog::details::log_msg &log_msg,
                                   bool log_enabled,
                                   bool traceback_enabled) {
    if (log_enabled) {
        sink_it_(log_msg);
    }
    if (traceback_enabled) {
        tracer_.push_back(log_msg);
    }
}
```
log_it_函数主要就是接收log函数根据日志级别判断是否日志打印的参数，调用sink_it_函数。
```C++
SPDLOG_INLINE void logger::sink_it_(const details::log_msg &msg) {
    for (auto &sink : sinks_) {
        if (sink->should_log(msg.level)) {
            SPDLOG_TRY { sink->log(msg); }
            SPDLOG_LOGGER_CATCH(msg.source)
        }
    }

    if (should_flush_(msg)) {
        flush_();
    }
}
```
sink_it_函数遍历所有sink，调用sink的log函数。这里sinks_的类型是std::vector<sink_ptr>，即shared_ptr的容器，每个元素是指向sink对象的智能指针。而log是虚函数，交由继承sink的子类对象来实现，这里sink->log(msg);很好的展现了多态的实现机制。base_sink继承了sink，且其他sink继承了base_sink，spdlog中定义了多种sink，如daily_file_sink、rotating_file_sink、stdout_sinks等，这些sink实现的头文件放到了sinks中。以daily_file_sink为例，最后写日志的函数实现为：
```C++
void sink_it_(const details::log_msg &msg) override {
    auto time = msg.time;
    bool should_rotate = time >= rotation_tp_;
    if (should_rotate) {
        auto filename = FileNameCalc::calc_filename(base_filename_, now_tm(time));
        file_helper_.open(filename, truncate_);
        rotation_tp_ = next_rotation_tp_();
    }
    memory_buf_t formatted;
    base_sink<Mutex>::formatter_->format(msg, formatted);
    file_helper_.write(formatted);

    // Do the cleaning only at the end because it might throw on failure.
    if (should_rotate && max_files_ > 0) {
        delete_old_();
    }
}
```
根据msg的时间判断是否大于轮转时间，如果大于就要新建一个文件，打开、写文件是通过抽象的file层实现。写操作是最后调用stdio.h中标准库POSIX接口fwrite_unlocked来实现文件写数据，这是单线程下不加锁的实现。
```C++
SPDLOG_INLINE void file_helper::write(const memory_buf_t &buf) {
    if (fd_ == nullptr) return;
    size_t msg_size = buf.size();
    auto data = buf.data();

    if (!details::os::fwrite_bytes(data, msg_size, fd_)) {
        throw_spdlog_ex("Failed writing to file " + os::filename_to_str(filename_), errno);
    }
}
```
```C++
extern size_t fread_unlocked (void *__restrict __ptr, size_t __size,
             size_t __n, FILE *__restrict __stream) __wur
  __nonnull ((4));
extern size_t fwrite_unlocked (const void *__restrict __ptr, size_t __size,
              size_t __n, FILE *__restrict __stream)
  __nonnull ((4));
#endif
```
以上就是spdlog提供的spdlog::info实现流程，整体来看就是创建全局唯一单例registry，然后创建logger，logger构造时可以绑定sink输出槽，将日志消息传递到sink中调用标准库API进行写日志，其中还用到了fmt格式化。这是在单线程下单一logger单一sink的最简单实现。在example/example.cpp中还给出了其他使用示例，比如我们在使用时可以直接创建一个shared_ptr<logger>类型的对象指向spdlog所提供的sink，创建sink时一般是名字（字符串）和filename绑定的方式。一个logger是可以对应多个sink的，在logger的构造函数中提供了迭代器复制构造的实现：
```C++
template <typename It>
logger(std::string name, It begin, It end)
    : name_(std::move(name)),
      sinks_(begin, end) {}
```
本篇文章从示例出发，一层一层地解析了spdlog的实现。其实更深层次的解析应该从架构出发，给出整个库实现的各个模块关系，这里先挖个坑，有时间再来填。

---

> 作者: [UAreFree](https://github.com/UAreFree)  
> URL: http://localhost:1313/posts/spdlog-analysis/  

