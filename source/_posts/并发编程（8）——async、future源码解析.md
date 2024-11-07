---
title: 并发编程（8）—— std::async、std::future 源码解析
date: 2024-11-07 16:03:20
categories:
- C++
- 并发编程
tags: 
- async
- future 
typora-root-url: ./.. 
---


# 八、day8

之前说过，`std::async`内部的处理逻辑和`std::thread`相似，而且`std::async`和`std::future`有密不可分的联系。今天，通过对`std::async`和`std::future`源码进行解析，了解二者的处理逻辑和关系。

> 源码均基于 MSVC 实现

参考：

1. [博主恋恋风辰的个人博客](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/2WyPSKn5SVHAMVwFrQkCnyGn1kF)
2. [up主mq白cpp的个人仓库](https://github.com/Mq-b/ModernCpp-ConcurrentProgramming-Tutorial/blob/main/md/%E8%AF%A6%E7%BB%86%E5%88%86%E6%9E%90/03async%E4%B8%8Efuture%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md)

# 1. std::async

`std::async`有两种重载实现：

```cpp
template <class _Fty, class... _ArgTypes>
_NODISCARD future<_Invoke_result_t<decay_t<_Fty>, decay_t<_ArgTypes>...>> async(
    launch _Policy, _Fty&& _Fnarg, _ArgTypes&&... _Args) {
    // manages a callable object launched with supplied policy
    using _Ret   = _Invoke_result_t<decay_t<_Fty>, decay_t<_ArgTypes>...>;
    using _Ptype = typename _P_arg_type<_Ret>::type;
    _Promise<_Ptype> _Pr(
        _Get_associated_state<_Ret>(_Policy, _Fake_no_copy_callable_adapter<_Fty, _ArgTypes...>(
                                                 _STD forward<_Fty>(_Fnarg), _STD forward<_ArgTypes>(_Args)...)));

    return future<_Ret>(_Pr._Get_state_for_future(), _Nil());
}

template <class _Fty, class... _ArgTypes>
_NODISCARD future<_Invoke_result_t<decay_t<_Fty>, decay_t<_ArgTypes>...>> async(_Fty&& _Fnarg, _ArgTypes&&... _Args) {
    // manages a callable object launched with default policy
    return _STD async(launch::async | launch::deferred, _STD forward<_Fty>(_Fnarg), _STD forward<_ArgTypes>(_Args)...);
}
```

第一种重载需要显式指定启动策略，也就是我们之前说的`std::launch::async`、`std::launch::deferred`和`std::launch::async | std::launch::deferred`；第二种重载在使用默认策略时会被调用（也就是只传递可调用对象和参数而不传递启动策略），在内部会调用第一种重载并传入一个`std::launch::async | std::launch::deferred`策略，并将参数全部转发。

我们只需要着重关注第一种重载即可：

1. 模板参数和函数体外部信息：
   
   - `_Fty`：可调用对象的类型
   - `_ArgTypes`：可调用对象所需的参数类型
   - `_NODISCARD`：宏，用于标记该函数的返回值不应被忽略
   
2. 返回类型：

   ```cpp
   future<_Invoke_result_t<decay_t<_Fty>, decay_t<_ArgTypes>...>>
   ```

   其实就是返回一个 `std::future` 对象，`_Invoke_result_t<decay_t<_Fty>, decay_t<_ArgTypes>...>`是`std::invoke`对给定的可调用对象 `_Fnarg` 和参数 `_Args...` 执行后返回的类型，其实也就是通过 `_Invoke_result_t` 从 `_Fty` 和 `_ArgTypes...` 中推导出的返回类型。
   
   > 我们之前在`thread`源码解析中说过`std::invoke`内部其实是调用`_Call`函数，`_Call`函数负责提供参数并调用传入的可调用对象。
   
   我们可以把 `_Invoke_result_t` 看作是一个对 `std::invoke` 的结果类型的封装，`std::invoke` 是一个工具，可以调用可调用对象并返回其结果。`_Invoke_result_t` 提供了一种方式来“推导”出这个结果类型。
   
   这个类型萃取工具通常长这样（简化版）：
   
   ```cpp
   template <typename _Callable, typename... _Args>
   struct _Invoke_result_t {
       using type = decltype(std::invoke(std::declval<_Callable>(), std::declval<_Args>()...));
   };
   ```
   
   上述代码通过 `std::invoke` 来推导（**decltype**） `_Callable`（即可调用对象）在给定参数 `_Args...` 上执行后的返回类型。换句话说，`_Invoke_result_t<_Fty, _ArgTypes...>` 的 `type` 成员类型就是可调用对象在调用后的返回类型。
   
   > 值得注意的是，所有类型在传递前都进行了 [`decay`](https://zh.cppreference.com/w/cpp/types/decay) 处理，也就是将cv和const修饰符去掉，默认按值传递与 `std::thread` 的行为一致。

3. 形参：

   ```cpp
   future<_Ret> async(launch _Policy, _Fty&& _Fnarg, _ArgTypes&&... _Args) {}
   ```

   - `launch _Policy`: 表示任务的执行策略，可以是 `launch::async`（表示异步执行）或 `launch::deferred`（表示延迟执行），或者`std::launch::async | std::launch::deferred`
   - `_Fty&& _Fnarg`: 可调用对象，通过完美转发机制将其转发给实际的异步任务
   - `_ArgTypes&&... _Args`: 调用该可调用对象时所需的参数，同样通过完美转发机制进行转发

4. `_Ret`和`_Ptype `：

   - `_Ret`就算我们在返回类型中说到的`_Invoke_result_t<decay_t<_Fty>, decay_t<_ArgTypes>...>`，表示可调用对象的返回类型；

   - `using _Ptype = typename _P_arg_type<_Ret>::type`：`_Ptype` 的定义在大多数情况下和 `_Ret` 是相同的，类模板 `_P_arg_type `只是为了处理**引用类型**以及 **void** 的情况，参见 `_P_arg_type` 的实现：

     ```cpp
     template <class _Fret>
     struct _P_arg_type { // type for functions returning T
         using type = _Fret;
     };
     
     template <class _Fret>
     struct _P_arg_type<_Fret&> { // type for functions returning reference to T
         using type = _Fret*;
     };
     
     template <>
     struct _P_arg_type<void> { // type for functions returning void
         using type = int;
     };
     ```

     > 为什么需要 **_Ptype** ？

     在异步任务的实现中，`std::promise` 是用于将结果与 `std::future` 绑定的对象。`std::promise` 的模板参数通常是可调用对象返回值的类型。在 `std::async` 函数中，我们需要创建一个 `std::promise` 对象来存储任务的结果，因此我们需要计算出正确的承诺类型（`promise type`）。**也就是说，定义 `_Ptype` 是为了配合后面 `_Promise` 的使用**，确保任务的结果可以通过 `std::future` 获取。

     - `_Ret` 是任务返回的类型（由 `_Invoke_result_t` 推导出）。

     - `_Ptype` 就是这个返回类型的承诺类型。也就是说，**`_Ptype` 是 `std::promise` 的模板参数类型，**表示这个任务结果的类型。

     `_Ptype` 的定义在大多数情况下和 `_Ret` 是相同的，都是可调用对象返回值的类型。

5. `_Promise<_Ptype> _Pr`：创建一个 `std::promise` 对象 `_Pr`，其类型为 `_Ptype`，表示与异步任务的结果相关联的承诺（promise）。`_Promise`类型我们之前讲过，这里就不在叙述它的作用，关键还在于其存储的数据成员：

   ```cpp
   template <class _Ty>
   class _Promise {
   public:
       _Promise(_Associated_state<_Ty>* _State_ptr) : _State(_State_ptr, false), _Future_retrieved(false) {}
   
       _Promise(_Promise&& _Other) : _State(_STD move(_Other._State)), _Future_retrieved(_Other._Future_retrieved) {}
   
       _Promise& operator=(_Promise&& _Other) {
           _State            = _STD move(_Other._State);
           _Future_retrieved = _Other._Future_retrieved;
           return *this;
       }
   
       ~_Promise() noexcept {}
   
       void _Swap(_Promise& _Other) {
           _State._Swap(_Other._State);
           _STD swap(_Future_retrieved, _Other._Future_retrieved);
       }
   
       const _State_manager<_Ty>& _Get_state() const {
           return _State;
       }
       _State_manager<_Ty>& _Get_state() {
           return _State;
       }
   
       _State_manager<_Ty>& _Get_state_for_set() {
           if (!_State.valid()) {
               _Throw_future_error(make_error_code(future_errc::no_state));
           }
   
           return _State;
       }
   
       _State_manager<_Ty>& _Get_state_for_future() {
           if (!_State.valid()) {
               _Throw_future_error(make_error_code(future_errc::no_state));
           }
   
           if (_Future_retrieved) {
               _Throw_future_error(make_error_code(future_errc::future_already_retrieved));
           }
   
           _Future_retrieved = true;
           return _State;
       }
   
       bool _Is_valid() const noexcept {
           return _State.valid();
       }
   
       bool _Is_ready() const {
           return _State._Is_ready();
       }
   
       bool _Is_ready_at_thread_exit() const {
           return _State._Is_ready_at_thread_exit();
       }
   
       _Promise(const _Promise&) = delete;
       _Promise& operator=(const _Promise&) = delete;
   
   private:
       _State_manager<_Ty> _State;
       bool _Future_retrieved;
   };
   ```

   > 注意：`_Promise` 和 `std::promise` 并不是同一个模板类，`_Promise`是为了提供对 `std::promise` 的进一步定制，并不是`std::primse`本身。`std::primise`模板类的私有成员是通过`_Promise`声明的，即

   ```cpp
   // std::primise 的私有成员
   private:
       _Promise<_Ty*> _MyPromise;
   ```

   ------

   `_Promise` 类模板是对`_State_manager`类模板的包装，并增加了一个表示状态的私有成员 `_Future_retrieved`。

   ```cpp
   private:
       _State_manager<_Ty> _State;
       bool _Future_retrieved;
   ```

   **状态成员**用于跟踪 `_Promise `是否已经调用过 `_Get_state_for_future() `成员函数；它默认为 `false`，在第一次调用 `_Get_state_for_future()` 成员函数时被置为 `true`，如果二次调用，就会抛出`future_errc::future_already_retrieved`异常。

   `_Promise `的构造函数接受的不是`_State_manager `类型的对象，而是`_Associated_state `类型的指针，用来初始化数据成员 `_State`。

   ```cpp
   _Promise(_Associated_state<_Ty>* _State_ptr) : _State(_State_ptr, false), _Future_retrieved(false) {}
   ```

   这是因为实际上`_State_manager`类型只有两个私有成员：`Associated_state`指针，以及一个状态成员：

   ```cpp
   private:
       _Associated_state<_Ty>* _Assoc_state;
       bool _Get_only_once;
   ```

   可以简单理解为 `_State_manager` 是对 `Associated_state` 的包装，其中的大部分接口实际上是调用 `_Assoc_state` 的成员函数（你们可以去`_State_manager`的实现源码中查阅，大部分接口其实都是通过调用`_Assoc_state`实现的）。

   所以在解析`std::async`源码之前，我们必须对`Associated_state`有一个清晰的了解：

   ```cpp
   public:
       _Ty _Result;
       exception_ptr _Exception;
       mutex _Mtx;
       condition_variable _Cond;
       bool _Retrieved;
       int _Ready;
       bool _Ready_at_thread_exit;
       bool _Has_stored_result;
       bool _Running;
   ```

   这是`Associated_state`模板类主要的成员变量（我没有全部列上去，只列了主要的），其中，最为重要的三个变量是：**异常指针**、**互斥量**、**条件变量**。

   其实，`_Associated_state` 模板类负责**管理异步任务的状态**，包括结果的存储、异常的处理以及任务完成的通知。它是实现 `std::future` 和 `std::promise` 的核心组件之一，通过 `_State_manager` 和 `_Promise` 类模板对其进行封装和管理，提供更高级别的接口和功能。

   ![1730980524864](../images/$%7Bfiilename%7D/1730980524864.jpg)

   `_Promise`、_`State_manager`、`_Associated_state` 之间的**包含关系**如上述结构所示。

6. 初始化 `_Promise` 对象：

   ```cpp
   _Promise<_Ptype> _Pr(
       _Get_associated_state<_Ret>(_Policy, 
           _Fake_no_copy_callable_adapter<_Fty, _ArgTypes...>(
               _STD forward<_Fty>(_Fnarg), 
               _STD forward<_ArgTypes>(_Args)...
           )
       )
   );
   ```

   这是一个函数调用，将我们 `std::async` 的参数全部转发给它。

   1. 首先将参数 `_Fnarg` （可调用对象）和 `_Args...`（传入可调用对象的参数包） 通过 `std::forward` 转发给 `_Fake_no_copy_callable_adapter`。
   2. 然后，`_Fake_no_copy_callable_adapter` 创建一个可调用对象（函数适配器）。
   3. 接着，**适配器**和指定的**启动策略**被传递给 `_Get_associated_state` 函数，目的是获取与异步操作相关的状态。
   4. 最终，`_Get_associated_state` 返回一个与异步操作相关的状态，并将其传递给 `_Pr`，这将会返回一个 `_Promise<_Ptype>`，代表一个异步操作的结果。

   `_Get_associated_state`函数根据启动模式（_Policy，有三种）来决定创建的异步任务状态对象类型：

   ```cpp
   template <class _Ret, class _Fty>
   _Associated_state<typename _P_arg_type<_Ret>::type>* _Get_associated_state(launch _Psync, _Fty&& _Fnarg) {
       // construct associated asynchronous state object for the launch type
       switch (_Psync) { // select launch type
       case launch::deferred:
           return new _Deferred_async_state<_Ret>(_STD forward<_Fty>(_Fnarg));
       case launch::async: // TRANSITION, fixed in vMajorNext, should create a new thread here
       default:
           return new _Task_async_state<_Ret>(_STD forward<_Fty>(_Fnarg));
       }
   }
   ```

   `_Get_associated_state` 函数返回一个 `_Associated_state` 指针（ `_Associated_state` 可用于初始化`_State_manager`，_`State_manager`可用于初始化`_Promise`），该指针指向一个新的 `_Deferred_async_state` 或`_Task_async_state` 对象。这两个类分别对应于异步任务的两种不同执行策略：**延迟执行**和**异步执行**。

   > 这段代码也很好的说明，`launch::async | launch::deferred` 和 `launch::async` 的行为是相同的，都会创建新线程异步执行任务，只不过前者会自行判断系统资源来抉择。

   ------

   `_Task_async_state` 与 `_Deferred_async_state` 都继承自 `_Packaged_state`，其用于异步执行任务。它们的构造函数都接受一个函数对象，并将其转发给基类 `_Packaged_state` 的构造函数。

   ```cpp
   // _Task_async_state 的构造函数
   template <class _Rx>
   class _Task_async_state : public _Packaged_state<_Rx()>
   // _Deferred_async_state 的构造函数
   template <class _Rx>
   class _Deferred_async_state : public _Packaged_state<_Rx()>
   ```

   `_Packaged_state `类型只有一个数据成员 ：`std::function` 类型的对象 `_Fn`，它用来**存储需要执行的异步任务**，而它又继承自 `_Associated_state`。

   ```cpp
   template <class _Ret, class... _ArgTypes>
   class _Packaged_state<_Ret(_ArgTypes...)>
       : public _Associated_state<_Ret>
   ```

   ![1730981460150](../images/$%7Bfiilename%7D/1730981460150.jpg)

   如上图所示，`_Task_async_state` 与 `_Deferred_async_state` 都继承自 `_Packaged_state`， `_Packaged_state`中保存了传入给`std::async`的可调用对象。同时，`_Packaged_state`继承自`_Associated_state`，`_Associated_state` 是`_Primise`类中成员`_State`的最基本组成对象，基本所有的接口都是通过调用`_Associated_state` 的函数实现的。

   `_Task_async_state` 与 `_Deferred_async_state` 的构造函数如下：

   ```cpp
   // _Task_async_state
   template <class _Fty2>
   _Task_async_state(_Fty2&& _Fnarg) : _Mybase(_STD forward<_Fty2>(_Fnarg)) {
       _Task = ::Concurrency::create_task([this]() { // do it now
           this->_Call_immediate();
       });
   
       this->_Running = true;
   }
   // _Deferred_async_state
   template <class _Fty2>
   _Deferred_async_state(const _Fty2& _Fnarg) : _Packaged_state<_Rx()>(_Fnarg) {}
   template <class _Fty2>
   _Deferred_async_state(_Fty2&& _Fnarg) : _Packaged_state<_Rx()>(_STD forward<_Fty2>(_Fnarg)) {}
   ```

   ***a. _Task_async_state***

   `_Task_async_state`有一个数据成员`_Task`用于从**线程池**中获取线程，并执行可调用对象：

   ```cpp
   private:
       ::Concurrency::task<void> _Task;
   ```

   `_Task_async_state `的实现使用了微软实现的并行模式库（PPL）。简而言之， `launch::async` 策略并不是单纯的创建线程让任务执行，而是使用了微软的 `::Concurrency::create_task` ，它**从线程池中获取线程并执行任务返回包装对象**。

   `this->_Call_immediate()`是调用 `_Task_async_state `的父类 `_Packaged_state `的成员函数 `_Call_immediate`.

    `_Packaged_state`有三个版本，自然`_Call_immediate`也有三种版本，用于处理可调用对象返回类型的**三种情况**：

   ```cpp
   // 返回普通类型
   // class _Packaged_state<void(_ArgTypes...)>
   void _Call_immediate(_ArgTypes... _Args) {
       _TRY_BEGIN
       // 调用函数对象并捕获异常 传递返回值
       this->_Set_value(_Fn(_STD forward<_ArgTypes>(_Args)...), false);
       _CATCH_ALL
       // 函数对象抛出异常就记录
       this->_Set_exception(_STD current_exception(), false);
       _CATCH_END
   }
   
   // 返回引用类型
   // class _Packaged_state<_Ret&(_ArgTypes...)>
   void _Call_immediate(_ArgTypes... _Args) {
       _TRY_BEGIN
       // 调用函数对象并捕获异常 传递返回值的地址
       this->_Set_value(_STD addressof(_Fn(_STD forward<_ArgTypes>(_Args)...)), false);
       _CATCH_ALL
       // 函数对象抛出异常就记录
       this->_Set_exception(_STD current_exception(), false);
       _CATCH_END
   }
   
   // 返回void类型
   // class _Packaged_state<void(_ArgTypes...)>
   void _Call_immediate(_ArgTypes... _Args) { 
       _TRY_BEGIN
       // 调用函数对象并捕获异常 因为返回 void 不获取返回值 而是直接 _Set_value 传递一个 1
       _Fn(_STD forward<_ArgTypes>(_Args)...);
       this->_Set_value(1, false);
       _CATCH_ALL
       // 函数对象抛出异常就记录
       this->_Set_exception(_STD current_exception(), false);
       _CATCH_END
   }
   ```

   `_Fn(_STD forward<_ArgTypes>(_Args)...`表示执行可调用对象，`this->_Set_value(_Fn(_STD forward<_ArgTypes>(_Args)...)`表示将可调用对象的返回值传入给`_Set_value`，其他两个函数也是类似的处理过程。

   `_TRY_BEGIN`、`_CATCH_ALL`、`_CATCH_END`类似`try-catch块`。当 `_Fn` 函数对象抛出异常时，控制流会跳转到 `_CATCH_ALL` 代码块；`this->_Set_exception`用来记录当前捕获的异常;`_CATCH_END` 标识异常处理的结束；因为返回类型为`void `表示不获取返回值，所以这里通过`_Set_value `传递一个 1（表示正确执行的状态）。所有的返回值均传入给`_Set_value `。

   > 简而言之，就是把返回引用类型的可调用对象返回值的引用获取地址传递给 `_Set_value`，把返回 void 类型的可调用对象传递一个 1 （表示正确执行的状态）给 `_Set_value`。

   `_Set_value`、`_set_exception`函数来自`_Packaged_state`模板类的父类`_Associated_state`，通过这两个函数，传递的可调用对象**执行结果**，以及**可能的异常**，并将**结果或异常存储在 `_Associated_state` 中**。

   ***b. _Deferred_async_state***

   `_Deferred_async_state` 并**不会**从线程池中获取一个新线程，然后再新线程中执行任务，而是**当前线程**调用future的get或者wait函数时，在**当前线程**中**同步**执行。但它同样调用 `_Call_immediate` 函数执行存储的可调用对象，它有一个 `_Run_deferred_function` 函数：

   ```cpp
   void _Run_deferred_function(unique_lock<mutex>& _Lock) override { // run the deferred function
       _Lock.unlock();
       _Packaged_state<_Rx()>::_Call_immediate();
       _Lock.lock();
   }
   ```

   然后通过 `_Call_immediate`调用可调用对象并通过函数`_Set_value`、`_set_exception`存储可调用对象返回结果或者异常至`_Associated_state`。

7.  返回 `std::future`

   ```cpp
   return future<_Ret>(_Pr._Get_state_for_future(), _Nil());
   ```

   `_Ret`在前面说了，其实就是可调用对象返回值的类型。

   传给`future`构造函数的参数之一是：`_Pr._Get_state_for_future()`，调用上面构造的`_Promise`的成员函数`_Get_state_for_future`，该函数用于返回`_Promise`类的私有成员变量`_State`。

   `_Get_state_for_future`函数的实现如下：

   ```cpp
   _State_manager<_Ty>& _Get_state_for_future() {
       if (!_State.valid()) {
           _Throw_future_error2(future_errc::no_state);
       }
   
       if (_Future_retrieved) {
           _Throw_future_error2(future_errc::future_already_retrieved);
       }
   
       _Future_retrieved = true;
       return _State;
   }
   ```

   其实就是调用`_State`的成员函数`valid()`检查状态（是否有错），然后判断`future`是否提前返回可调用对象的返回值（如果是，代表future的get被调用，抛出异常）；最后，返回`_State`。

# 2. std::future

我们首先从一个最简单的`std::async`示例开始：

```cpp
std::future<int> future = std::async([] { return 0; });
future.get();
```

我们从之前的学习中了解到，`future.get()`就是从`future`中获取可调用对象的返回结果。唯一的问题是：`future.get()` 内部执行了什么流程？首先从`future`的实现开始：

```cpp
_EXPORT_STD template <class _Ty>
class future : public _State_manager<_Ty> {
    // class that defines a non-copyable asynchronous return object that holds a value
private:
    using _Mybase = _State_manager<_Ty>;

public:
    static_assert(!is_array_v<_Ty> && is_object_v<_Ty> && is_destructible_v<_Ty>,
        "T in future<T> must meet the Cpp17Destructible requirements (N4950 [futures.unique.future]/4).");

    future() = default;

    future(future&& _Other) noexcept : _Mybase(_STD move(_Other), true) {}

    future& operator=(future&&) = default;

    future(_From_raw_state_tag, const _Mybase& _State) noexcept : _Mybase(_State, true) {}

    _Ty get() {
        // block until ready then return the stored result or throw the stored exception
        future _Local{_STD move(*this)};
        return _STD move(_Local._Get_value());
    }

    _NODISCARD shared_future<_Ty> share() noexcept {
        return shared_future<_Ty>(_STD move(*this));
    }

    future(const future&)            = delete;
    future& operator=(const future&) = delete;
};
```

`future`类继承自`_State_manager`类，`_State_manager`类又有一个`_Associated_state<_Ty>*`类型的私有成员`_State`，而`_State_manager`的接口实现大部分是通过调用`_Associated_state` 的成员函数实现的。关系如下：

![1730985805487](../images/$%7Bfiilename%7D/1730985805487.jpg)

## 2.1 wait()

> 但你可能发现一个问题，`future`类怎么**没有**`wait()`成员函数？？？？其实，`wait()`函数继承自父类`_State_manager`。

```cpp
void wait() const { // wait for signal
    if (!valid()) {
        _Throw_future_error2(future_errc::no_state);
    }

    _Assoc_state->_Wait();
}
```

而`_State_manager`类的`wait()`其实是通过调用`_Associated_state`的接口实现的，所以说，`_Associated_state`在`std::async`和`std::future`中是非常核心的。

```cpp
virtual void _Wait() { // wait for signal
    unique_lock<mutex> _Lock(_Mtx);
    _Maybe_run_deferred_function(_Lock);
    while (!_Ready) {
        _Cond.wait(_Lock);
    }
}
```

`_Associated_state`的`wait()`函数通过`unique_lock`保护共享数据，然后调用`_Maybe_run_deferred_function`执行可调用对象，直至调用结束。

```cpp
void _Maybe_run_deferred_function(unique_lock<mutex>& _Lock) { // run a deferred function if not already done
    if (!_Running) { // run the function
        _Running = true;
        _Run_deferred_function(_Lock);
    }
}
```

`_Maybe_run_deferred_function`其实就是通过调用`_Run_deferred_function`来调用`_Call_immediate()`，我们在`async`源码中学习过`_Run_deferred_function`和`_Call_immediate()。

```cpp
void _Run_deferred_function(unique_lock<mutex>& _Lock) override { // run the deferred function
    _Lock.unlock();
    _Packaged_state<_Rx()>::_Call_immediate();
    _Lock.lock();
}
```

在 `_Wait` 函数中调用 `_Maybe_run_deferred_function` 是为了确保延迟执行（`launch::deferred`）的任务能够在等待前被启动并执行完毕。这样，在调用 `wait` 时可以正确地等待任务完成。

因为只有`std::launch::deferred`才是当调用`future.get或者wait`时才会执行`_Call_immediate()`，其他两种启动策略在大部分情况下都是直接执行，通过`future.get`获得结果。所以我们必须保证在调用`wait`函数时，执行`std::launch::deferred`策略的任务被执行，而其他两种启动策略早已经执行任务，无需再调用`_Call_immediate()`。所以在`_Maybe_run_deferred_function`函数中，有下面一段，判断任务是否以及执行，如果被执行，那么久就不调用`_Call_immediate`，反之调用。

```cpp
    if (!_Running) { // run the function
        _Running = true;
        _Run_deferred_function(_Lock);
```

------

```cpp
   while (!_Ready) {
        _Cond.wait(_Lock);
    }
```

通过条件变量挂起当前线程，等待可调用对象执行完毕。在等待期间，当前线程释放持有的锁，保证其他线程再次期间可以访问到共享资源，待当前线程被唤醒后，重新持有锁。其主要作用是：

1. 避免虚假唤醒：
   - 条件变量的 `wait` 函数在被唤醒后，会重新检查条件（即 `_Ready` 是否为 `true`），确保只有在条件满足时才会继续执行。这防止了由于虚假唤醒导致的错误行为。
2. 等待 `launch::async` 的任务在其它线程执行完毕：
   - 对于 `launch::async` 模式的任务，这段代码确保当前线程会等待任务在另一个线程中执行完毕，并接收到任务完成的信号。只有当任务完成并设置 `_Ready` 为 `true` 后，条件变量才会被通知，从而结束等待。

这样，当调用 `wait` 函数时，可以保证无论任务是 `launch::deferred` 还是 `launch::async` 模式，当前线程都会正确地等待任务的完成信号，然后继续执行。

------

`std::future` 其实还有两种特化，不过整体大差不差。

```cpp
template <class _Ty>
class future<_Ty&> : public _State_manager<_Ty*>
```

```cpp
template <>
class future<void> : public _State_manager<int>
```

也就是对返回类型为引用和 void 的情况了。其实先前已经聊过很多次了，无非就是内部的返回引用实际按指针操作，返回 void，那么也得给个 1，表示正常运行的状态。类似于前面 `_Call_immediate` 的实现。

## 2.2 get()

`get()`函数是`future`的成员函数，而没有继承父类`_State_manager`：

```cpp
// std::future<void>
void get() {
    // block until ready then return or throw the stored exception
    future _Local{_STD move(*this)};
    _Local._Get_value();
}
// std::future<T>
_Ty get() {
    // block until ready then return the stored result or throw the stored exception
    future _Local{_STD move(*this)};
    return _STD move(_Local._Get_value());
}
// std::future<T&>
_Ty& get() {
    // block until ready then return the stored result or throw the stored exception
    future _Local{_STD move(*this)};
    return *_Local._Get_value();
}
```

因为`future`有三种特化，所以`get()`函数也有三种特化。它们将当前`future`对象的指针通过`std::move`转移给类型为`future`的局部变量`_Local`（转移后，原本的`future`对象便失去了所有权）。然后，局部变量`_Local`调用成员函数`_Get_value()`，并将结果返回。

> 注意：局部对象 `_Local` 在函数结束时**析构**。这意味着当前对象（`*this`）失去共享状态，并且状态被完全销毁。

`_Get_value()` 函数的实现如下：

```cpp
_Ty& _Get_value() const {
    if (!valid()) {
        _Throw_future_error2(future_errc::no_state);
    }

    return _Assoc_state->_Get_value(_Get_only_once);
}
```

[`future.valid()`](https://zh.cppreference.com/w/cpp/thread/future/valid) 成员函数检查 future 当前是否关联共享状态，即是否当前关联任务。如果还未关联，或者任务已经执行完（调用了 get()、set()），都会返回 **`false`**。

- 首先，通过`valid()`判断当前`future`对象是否关联共享状态，如果没，抛出异常。
- 最后，调用 `_Assoc_state` 的成员函数 `_Get_value` ，传递 `_Get_only_once` 参数，其实就是代表这个成员函数只能调用一次。

`_Assoc_state` 的类型是 `_Associated_state<_Ty>*` ，是一个指针类型，它实际会指向自己的子类对象，我们在讲 `std::async` 源码的时候提到了，它必然指向 `_Deferred_async_state` 或者 `_Task_async_state`。

`_Assoc_state->_Get_value` 这其实是个多态调用，父类有这个虚函数：

```cpp
virtual _Ty& _Get_value(bool _Get_only_once) {
    unique_lock<mutex> _Lock(_Mtx);
    if (_Get_only_once && _Retrieved) {
        _Throw_future_error2(future_errc::future_already_retrieved);
    }

    if (_Exception) {
        _STD rethrow_exception(_Exception);
    }

    // TRANSITION: `_Retrieved` should be assigned before `_Exception` is thrown so that a `future::get`
    // that throws a stored exception invalidates the future (N4950 [futures.unique.future]/17)
    _Retrieved = true;
    _Maybe_run_deferred_function(_Lock);
    while (!_Ready) {
        _Cond.wait(_Lock);
    }

    if (_Exception) {
        _STD rethrow_exception(_Exception);
    }

    if constexpr (is_default_constructible_v<_Ty>) {
        return _Result;
    } else {
        return _Result._Held_value;
    }
}
```

子类 `_Task_async_state` 对其进行了重写，以 `launch::async` 策略或者`std::launch::async | std::launch::deferred`策略创建的`future`，实际会调用 `_Task_async_state::_Get_value` ：

```cpp
_State_type& _Get_value(bool _Get_only_once) override {
    // return the stored result or throw stored exception
    _Task.wait();
    return _Mybase::_Get_value(_Get_only_once);
}
```

> `_Deferred_async_state` 没有对其进行重写，直接调用父类虚函数。

`_Task` 就是 `::Concurrency::task<void> _Task;`，调用 `wait()` 成员函数确保任务执行完毕。

`_Mybase::_Get_value(_Get_only_once)` 其实又是回去调用父类的虚函数了。

> ***`_Get_value` 方法详解***

1. _Get_value()只能调用一次

   如果 `_Get_only_once` 为 `true` 且 `_Retrieved` 为 `true`（表示结果已经被检索过），则抛出 `future_already_retrieved` 错误。

2. 处理异常

   如果在获取结果过程中出现了异常，需要重新抛出该异常

3. 设置 `_Retrieved` 为 `true`

   在获取值之前，设置 `_Retrieved` 为 `true`，表示结果已经被检索过。这样可以确保 `future` 对象不会被重复获取，避免多次调用 `get` 时引发错误。

4. 执行延迟函数

   调用`_Maybe_run_deferred_function`来运行可能的延迟任务。在该函数内部，如果任务已经运行，那么退出，如果没运行，调用`_Call_immediate()`函数执行可调用对象。

5. 等待结果

   使用条件变量挂起当前线程，确保线程同步，即只有当异步任务准备好返回结果时，线程才会继续执行。

6. 再次检查异常

   线程被唤醒将结果存储至`future`对象中后，再次判断是否发生了异常，需要重新抛出异常

7. 返回结果

   这部分代码根据 `_Ty` 类型的特性决定如何返回结果：

   - 如果 `_Ty` 类型是 **默认可构造**（即 `_Ty` 的默认构造函数有效），直接返回 `_Result`。
   - 否则，返回 `_Result._Held_value`

   `is_default_constructible_v<_Ty>` 是一个 `C++17` 引入的类型特征，用于检查类型 `_Ty` 是否具有默认构造函数。

   `_Result` 是 `future` 中持有的结果，而 `_Held_value` 是存储在 `_Result` 中的实际值。

`_Result` 是通过执行 `_Call_immediate` 函数，然后 `_Call_immediate` 再执行 `_Set_value` ，`_Set_value` 再执行 `_Set_value_raw`，`_Set_value_raw`再执行`_Emplace_result`并通知线程可以醒来，`_Emplace_result` 获取到我们执行任务的返回值的。以 `Ty` 的偏特化为例：

```cpp
// _Packaged_state
void _Call_immediate(_ArgTypes... _Args) {
    _TRY_BEGIN
    // 调用函数对象并捕获异常 传递返回值
    this->_Set_value(_Fn(_STD forward<_ArgTypes>(_Args)...), false);
    _CATCH_ALL
    // 函数对象抛出异常就记录
    this->_Set_exception(_STD current_exception(), false);
    _CATCH_END
}

// _Asscoiated_state
void _Set_value(const _Ty& _Val, bool _At_thread_exit) { // store a result
    unique_lock<mutex> _Lock(_Mtx);
    _Set_value_raw(_Val, &_Lock, _At_thread_exit);
}
void _Set_value_raw(const _Ty& _Val, unique_lock<mutex>* _Lock, bool _At_thread_exit) {
    // store a result while inside a locked block
    if (_Already_has_stored_result()) {
        _Throw_future_error2(future_errc::promise_already_satisfied);
    }

    _Emplace_result(_Val);
    _Do_notify(_Lock, _At_thread_exit);
}
template <class _Ty2>
void _Emplace_result(_Ty2&& _Val) {
    // TRANSITION, incorrectly assigns _Result when _Ty is default constructible
    if constexpr (is_default_constructible_v<_Ty>) {
        _Result = _STD forward<_Ty2>(_Val); // !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    } else {
        ::new (static_cast<void*>(_STD addressof(_Result._Held_value))) _Ty(_STD forward<_Ty2>(_Val));
        _Has_stored_result = true;
    }
}
```

