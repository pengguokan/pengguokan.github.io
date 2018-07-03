---
layout: post
title: 创建线程OOM问题分析
categories: [Android,疑难问题]
description: 创建线程OOM问题分析
keywords: 线程
---

产品发布后，遇到了这么一类外网Crash：从堆栈看是Java层的Thread对象调用start()方法遇到了OutOfMemory Crash了。

![](/images/posts/oom_1.png){:height="200" width="400"}

为了找出OOM的原因，决定探索一下线程创建的流程。

最上层在new Thread()后调用start()，会调用虚拟机的nativeCreate。

```java
 public synchronized void start() {
    ...
    //daemon：代表是守护线程，如果为true，则在虚拟机退出的时候该线程还会活着
    //stackSize：指定了线程堆栈的大小（Bytes）
    nativeCreate(this, stackSize, daemon);
    ...
 }
```
继续分析nativeCreate()。从网上找到Android7.0的ART源码：/art/runtime/native/java_lang_Thread.cc	

```c
static void Thread_nativeCreate(JNIEnv* env, jclass, jobject java_thread, jlong stack_size,
                                jboolean daemon) {
  // There are sections in the zygote that forbid thread creation.
  Runtime* runtime = Runtime::Current();
  if (runtime->IsZygote() && runtime->IsZygoteNoThreadSection()) {
    jclass internal_error = env->FindClass("java/lang/InternalError");
    CHECK(internal_error != nullptr);
    env->ThrowNew(internal_error, "Cannot create threads in zygote");
    return;
  }

  Thread::CreateNativeThread(env, java_thread, stack_size, daemon == JNI_TRUE);
}
```
继而找到了/art/runtime/thread.cc：
```c
void Thread::CreateNativeThread(JNIEnv* env, jobject java_peer, size_t stack_size, bool is_daemon) {
  
    //...略去无关代码
    // Try to allocate a JNIEnvExt for the thread. We do this here as we might be out of memory and
    // do not have a good way to report this on the child's side.
    std::unique_ptr<JNIEnvExt> child_jni_env_ext(
        JNIEnvExt::Create(child_thread, Runtime::Current()->GetJavaVM()));

    int pthread_create_result = 0;
    if (child_jni_env_ext.get() != nullptr) {
        pthread_t new_pthread;
        pthread_attr_t attr;
        child_thread->tlsPtr_.tmp_jni_env = child_jni_env_ext.get();
        CHECK_PTHREAD_CALL(pthread_attr_init, (&attr), "new thread");
        CHECK_PTHREAD_CALL(pthread_attr_setdetachstate, (&attr, PTHREAD_CREATE_DETACHED),
                        "PTHREAD_CREATE_DETACHED");
        CHECK_PTHREAD_CALL(pthread_attr_setstacksize, (&attr, stack_size), stack_size);
        pthread_create_result = pthread_create(&new_pthread,
                                            &attr,
                                            Thread::CreateCallback,
                                            child_thread);
        CHECK_PTHREAD_CALL(pthread_attr_destroy, (&attr), "new thread");
        if (pthread_create_result == 0) {
            // pthread_create started the new thread. The child is now responsible for managing the
            // JNIEnvExt we created.
            // Note: we can't check for tmp_jni_env == nullptr, as that would require synchronization
            //       between the threads.
            child_jni_env_ext.release();
            return;
         }
    }

    //...略去无关代码
    std::string msg(child_jni_env_ext.get() == nullptr ?
        "Could not allocate JNI Env" :
        StringPrintf("pthread_create (%s stack) failed: %s",
                                 PrettySize(stack_size).c_str(), strerror(pthread_create_result)));
    soa.Self()->ThrowOutOfMemoryError(msg.c_str());
}
```
先看最后几行代码，和外网Crash堆栈很类似：说明是命中了child_jni_env_ext.get() != nullptr分支。

继续分析，是pthread_create调用失败了；pthread_create调用系统API创建实际线程，这个基础API出错通常是外部参数异常，从源码看很大可能就是child_jni_env_ext构造异常，也就是JNIEnvExt::Create出了问题。

JNIEnvExt是ART虚拟机给线程分配的JNI环境；继续挖下JNIEnvExt::Create()出了什么问题：
```c
static bool CheckLocalsValid(JNIEnvExt* in) NO_THREAD_SAFETY_ANALYSIS {
  if (in == nullptr) {
    return false;
  }
  return in->locals.IsValid();
}

JNIEnvExt* JNIEnvExt::Create(Thread* self_in, JavaVMExt* vm_in) {
  std::unique_ptr<JNIEnvExt> ret(new JNIEnvExt(self_in, vm_in));
  if (CheckLocalsValid(ret.get())) {
    return ret.release();
  }
  return nullptr;
}
```
所以应该是JNIEnvExt::Create()返回了nullptr；分析CheckLocalsValid()，基本排除是传入参数nullptr导致，而是locals.IsValid()返回了false；

locals是JNIEnvExt的一个成员变量，在JNIEnvExt构造的时候通过成员列表方式初始化：
```c
// JNI local references.
IndirectReferenceTable locals GUARDED_BY(Locks::mutator_lock_);
```
IsValid()源码：
```c
bool IndirectReferenceTable::IsValid() const {
  return table_mem_map_.get() != nullptr;
}
```
代码上看应该是获取内存映射挂掉了。继续分析table_mem_map_的创建过程，也就是在构造方法内：
```c
IndirectReferenceTable::IndirectReferenceTable(size_t initialCount,
                                               size_t maxCount, IndirectRefKind desiredKind,
                                               bool abort_on_error)
    : kind_(desiredKind),
      max_entries_(maxCount) {
  CHECK_GT(initialCount, 0U);
  CHECK_LE(initialCount, maxCount);
  CHECK_NE(desiredKind, kHandleScopeOrInvalid);

  std::string error_str;
  const size_t table_bytes = maxCount * sizeof(IrtEntry);
  table_mem_map_.reset(MemMap::MapAnonymous("indirect ref table", nullptr, table_bytes,
                                            PROT_READ | PROT_WRITE, false, false, &error_str));
  if (abort_on_error) {
    CHECK(table_mem_map_.get() != nullptr) << error_str;
    CHECK_EQ(table_mem_map_->Size(), table_bytes);
    CHECK(table_mem_map_->Begin() != nullptr);
  } else if (table_mem_map_.get() == nullptr ||
             table_mem_map_->Size() != table_bytes ||
             table_mem_map_->Begin() == nullptr) {
    table_mem_map_.reset();
    LOG(ERROR) << error_str;
    return;
  }
  table_ = reinterpret_cast<IrtEntry*>(table_mem_map_->Begin());
  segment_state_.all = IRT_FIRST_SEGMENT;
}
```
关键地方：
```c
const size_t table_bytes = maxCount * sizeof(IrtEntry);
table_mem_map_.reset(MemMap::MapAnonymous("indirect ref table", nullptr, table_bytes,
                                            PROT_READ | PROT_WRITE, false, false, &error_str));
```
maxCount是外部传入的常量（kLocalsMax）512，IrtEntry不清楚是做什么的，但sizeof应该是固定值。所以table_bytes应该是可以得到固定大小。MemMap::MapAnonymous创建一块大小为512*sizeof(IrtEntry)的共享内存发生了意外（MapAnonymous()有最后一个bool参数有默认值true，我当时还一直认为是代码版本不对:(）:
```c
MemMap* MemMap::MapAnonymous(const char* name,
                             uint8_t* expected_ptr,
                             size_t byte_count,
                             int prot,
                             bool low_4gb,
                             bool reuse,
                             std::string* error_msg,
                             bool use_ashmem) {
                
//...省略无关代码

  if (use_ashmem) {
    // android_os_Debug.cpp read_mapinfo assumes all ashmem regions associated with the VM are
    // prefixed "dalvik-".
    std::string debug_friendly_name("dalvik-");
    debug_friendly_name += name;
//...关键代码
    fd.reset(ashmem_create_region(debug_friendly_name.c_str(), page_aligned_byte_count));
//...关键代码
    if (fd.get() == -1) {
      *error_msg = StringPrintf("ashmem_create_region failed for '%s': %s", name, strerror(errno));
      return nullptr;
    }
    flags &= ~MAP_ANONYMOUS;
  }

  // We need to store and potentially set an error number for pretty printing of errors
  int saved_errno = 0;

  void* actual = MapInternal(expected_ptr,
                             page_aligned_byte_count,
                             prot,
                             flags,
                             fd.get(),
                             0,
                             low_4gb);
//...省略无关代码
}
```
ashmem_create_region如果成功了，那么会映射一个512*sizeof(IrtEntry)的虚拟空间到用户态（还不是物理空间，除非发生页缺失中断），这个时候应该就没有OOM什么事了。

显然ashmem_create_region应该是失败了。失败的话初步分析有两种可能：
```
1）内核分配内存失败，此时手机的内存资源应该是很紧张的
2）进程虚拟内存地址空间耗尽
```
从外网用户上报的日志来看，设备内存还是挺充裕的：

![](/images/posts/oom_2.png)

所以应该是进程内的虚拟地址空间不足导致的。


那什么情况下，会导致虚拟地址空间不足呢？

Android系统给每个进程分配了一定的虚拟地址空间大小，进程使用的虚拟空间如果超过阈值，就会触发OOM。

从环境上下文来看，挂掉的进程是一个非UI进程，一种很可能的情况是线程数过多（60+个），从网上资料了解到，一个线程创建大概占用1~2MB空间，所以线程消耗了大部分的虚拟地址空间，从而引发了当前进程空间不足。

所以优化结论是，严控线程数。但由于大部分线程都是在第三方SDK内部创建的，所以下一步研究如何监管线程。