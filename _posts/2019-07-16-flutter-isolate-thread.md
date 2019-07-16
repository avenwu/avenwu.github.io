---
layout: post
title: "Flutter Isolate线程分析"
description: ""
header_image: /assets/img/2019-07-16-01.jpg
keywords: "Flutter, Isolate, 线程"
tags: [Flutter]
---
{% include JB/setup %}
![img](/assets/img/2019-07-16-01.jpg)

* 目录
{:toc #markdown-toc}

在前面分析Isolate的使用过程中，我们掌握了几种使用Isolate的API，本文继续研究Isolate和线程的相关源码。

> 注意由于Flutter和Dart的版本不同，源码示意部分可能存在差异。
> 本文分析基于：Flutter Engine v1.78; Dart SDK v2.5

## Flutter Engine

还记得前面提到过，Dart里面的Isolate和Process，Thread都有区别么？在开发中我们可以借助上面提到的使用`compute`来创建独立Isolate实现计算耗时任务，
这里有几个深入思考的问题：

1. 创建Isolate是否可以复用？类似线程池？
2. Isolate创建过多是否会有性能问题？
3. 如果Isolate开销很大，是否实现一个Isolate对应多个任务，即一拖N的关系？

为了找到这些问题的答案，我先需要进一步去挖掘Isolate的实现源码。

Flutter SDK的源码由dart编写的，Engine部分是c++，在SDK层面关于线程和Isolate的相关代码通过简单索引只找到了一个定义位置：

```dart
external static Future<Isolate> spawn<T>(
    void entryPoint(T message), T message,
    {bool paused: false,
    bool errorsAreFatal,
    SendPort onExit,
    SendPort onError,
    @Since("2.3") String debugName});
```
他的定义是`external`也就是实现体在其他位置，不过通过检索整个flutter项目，并没有找到他的实现。因此我们把目光转到`flutter engine`项目。

为了方便分析，最好把engine的源码下载下来[engine](https://github.com/flutter/engine)，时间会比较长。

> git clone git@github.com:flutter/engine.git flutter-engine

*/runtime/dart_isolate.h*

```dart
// The root isolate of a Flutter application is special because it gets Window
// bindings. From the VM's perspective, this isolate is not special in any
// way.
static std::weak_ptr<DartIsolate> CreateRootIsolate(
  const Settings& settings,
  fml::RefPtr<const DartSnapshot> isolate_snapshot,
  fml::RefPtr<const DartSnapshot> shared_snapshot,
  TaskRunners task_runners,
  std::unique_ptr<Window> window,
  fml::WeakPtr<IOManager> io_manager,
  fml::WeakPtr<ImageDecoder> image_decoder,
  std::string advisory_script_uri,
  std::string advisory_script_entrypoint,
  Dart_IsolateFlags* flags,
  fml::closure isolate_create_callback,
  fml::closure isolate_shutdown_callback);
```
从实现体和注释信息，我们可以知道app启动后的默认root isolate关联了很多模块，比如`task runners`，`IO manager`，`Image Decoder`

*/runtime/dart_isolate.cc*
```dart
// Since this is the root isolate, we fake a parent embedder data object. We
// cannot use unique_ptr here because the destructor is private (since the
// isolate lifecycle is entirely managed by the VM).
//
// The child isolate preparer is null but will be set when the isolate is
// being prepared to run.
auto root_embedder_data = std::make_unique<std::shared_ptr<DartIsolate>>(
  std::make_shared<DartIsolate>(
      settings,                     // settings
      std::move(isolate_snapshot),  // isolate snapshot
      std::move(shared_snapshot),   // shared snapshot
      task_runners,                 // task runners
      std::move(io_manager),        // IO manager
      std::move(image_decoder),     // Image Decoder
      advisory_script_uri,          // advisory URI
      advisory_script_entrypoint,   // advisory entrypoint
      nullptr,                      // child isolate preparer
      isolate_create_callback,      // isolate create callback
      isolate_shutdown_callback     // isolate shutdown callback
      ));
```

关于Runner，实际上开发Flutter的时候并不会直接使用到，他更多的是Flutter Framework内部的模块划分，具体runner：

* platform runner
* gpu runner
* ui runner
* io runner

*common/task_runners.cc*
```dart
TaskRunners::TaskRunners(std::string label,
                         fml::RefPtr<fml::TaskRunner> platform,
                         fml::RefPtr<fml::TaskRunner> gpu,
                         fml::RefPtr<fml::TaskRunner> ui,
                         fml::RefPtr<fml::TaskRunner> io)
    : label_(std::move(label)),
      platform_(std::move(platform)),
      gpu_(std::move(gpu)),
      ui_(std::move(ui)),
      io_(std::move(io)) {}
```

*shell/common/thread_host.cc*

每一个Runner对应一个独立的线程,线程名字均带有各自的后缀
```dart
ThreadHost::ThreadHost(std::string name_prefix, uint64_t mask) {
  if (mask & ThreadHost::Type::Platform) {
    platform_thread = std::make_unique<fml::Thread>(name_prefix + ".platform");
  }

  if (mask & ThreadHost::Type::UI) {
    ui_thread = std::make_unique<fml::Thread>(name_prefix + ".ui");
  }

  if (mask & ThreadHost::Type::GPU) {
    gpu_thread = std::make_unique<fml::Thread>(name_prefix + ".gpu");
  }

  if (mask & ThreadHost::Type::IO) {
    io_thread = std::make_unique<fml::Thread>(name_prefix + ".io");
  }
}
void ThreadHost::Reset() {
  platform_thread.reset();
  ui_thread.reset();
  gpu_thread.reset();
  io_thread.reset();
}
```

## Dart VM

Android有Dalvik， ART，Java有JVM。Flutter也有Dart VM，他包含了执行dart所必须的整套环境，具体包括：
```
1. Runtime System 运行时系统
	* Object Model 对象模型
	* Garbage Collection 垃圾回收
	* Snapshots 快照
2. Core libraries native methods 核心库，native方法
3. Development Experience components accessible via service protocol 开发协议
	* Debugging 调试
	* Profiling 调优
	* Hot-reload 热更新
4. Just-in-Time (JIT) and Ahead-of-Time (AOT) compilation pipelines 变量管道JIT和AOT
5. Interpreter 解释器
6. ARM simulators ARM模拟器
```

针对Flutter来说，代码经过编译为机器码后在壳应用中运行，发布模式下使用的是AOT+Runtime，开发期间使用的是JIT+VM。

Isolate中代码执行和内存情况，可以参考下图。

![](/assets/images/isolates-memory.png)

下面分析下Isolate的执行线程问题，相关源码位置在*runtime/vm/isolate.cc*内。

一个Isolate的初始化，包括heap的创建，运行线程设置，线程异常配置等。
当Isolate运行时，他确实是跑在一个线程上面的。Isolate运行后，调用了消息处理器的Run方法，同时传入了线程池。

```c++
void Isolate::Run() {
  message_handler()->Run(Dart::thread_pool(), RunIsolate, ShutdownIsolate,
                         reinterpret_cast<uword>(this));
}
```

在消息处理器内部，会将`MessageHandlerTask`丢到线程池内执行。

*runtime/vm/message_handler.cc*

```c++
void MessageHandler::Run(ThreadPool* pool,
                         StartCallback start_callback,
                         EndCallback end_callback,
                         CallbackData data) {
  bool task_running;
  MonitorLocker ml(&monitor_);
  if (FLAG_trace_isolates) {
    OS::PrintErr(
        "[+] Starting message handler:\n"
        "\thandler:    %s\n",
        name());
  }
  ASSERT(pool_ == NULL);
  ASSERT(!delete_me_);
  pool_ = pool;
  start_callback_ = start_callback;
  end_callback_ = end_callback;
  callback_data_ = data;
  task_ = new MessageHandlerTask(this);
  task_running = pool_->Run(task_);
  ASSERT(task_running);
}
```

```c++
class MessageHandlerTask : public ThreadPool::Task {
 public:
  explicit MessageHandlerTask(MessageHandler* handler) : handler_(handler) {
    ASSERT(handler != NULL);
  }

  virtual void Run() {
    ASSERT(handler_ != NULL);
    handler_->TaskCallback();
  }

 private:
  MessageHandler* handler_;

  DISALLOW_COPY_AND_ASSIGN(MessageHandlerTask);
};
```

最终执行了Isolate的入口函数，如main函数

```c++
if (start_callback_) {
	// Initialize the message handler by running its start function,
	// if we have one.  For an isolate, this will run the isolate's
	// main() function.
	//
	// Release the monitor_ temporarily while we call the start callback.
	ml.Exit();
	status = start_callback_(callback_data_);
	ASSERT(Isolate::Current() == NULL);
	start_callback_ = NULL;
	ml.Enter();
}
```

## Isolate与线程调度问题

现在我们回顾下前面提出的疑问

> 创建Isolate是否可以复用？类似线程池？

Isolate本身并不是和线程样一一对应，Isolate运行在一个线程上，同时线程是通过线程池进行的复用，创建Isolate可以不用考虑复用线程问题。但是Isolate本身会开辟内存栈，因此过多的Isolate可预见的，对内存会有更多要求。

> Isolate创建过多是否会有性能问题？

理论上会影响性能，这个具体影响，可以通过实验来验证，创建一个Isolate所消耗的内存资源。
我们编写一段代码，循环10次创建10个Isolate，观察内存情况，可以得到下面两张图

![](/assets/images/isolate-none.png)

![](/assets/images/isolate-ton.png)

平均新建一个Isolate的内存开销为`(332-71.5)/10MB = 26MB`

当我们技术这些新建的Isolate后，内存基本全部释放，配合一次GC，内存降到了64MB

![](/assets/images/isolate-gc.png)

> 如果Isolate开销很大，是否实现一个Isolate对应多个任务，即一拖N的关系？

如果要实现一拖N的关系，那么不能使用`compute`封装的API，我们需要维护并复用一个Isolate，因此每个任务结束后不能直接kill掉Isolate。


## 参考

* [https://mrale.ph/dartvm/](https://mrale.ph/dartvm/)
* [https://github.com/dart-lang/sdk](https://github.com/dart-lang/sdk)
* [https://www.yuque.com/xytech/flutter/kwoww1#d2tpbo](https://www.yuque.com/xytech/flutter/kwoww1#d2tpbo)


