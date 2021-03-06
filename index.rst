CppCoro - 一个 C++ 协程库
########################################

**cppcoro** 提供了大量通用原语以便于使用 TS `N4680 <http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/n4680.pdf>`_ 提案描述的协程

这包括：

.. contents::

这个库是一个实验性的库，它在 C++ 协程的基础上探索高性能、可伸缩的异步编程抽象。

这个库是开源的，并希望其他人能够发现它的优点并提供反馈以及改进它的方法

它要求编译器支持协程：

- Windows + Visual Studio 2017 |Windows 构建状态|

.. |Windows 构建状态| image:: https://ci.appveyor.com/api/projects/status/github/lewissbaker/cppcoro?branch=master&svg=true&passingText=master%20-%20OK&failingText=master%20-%20Failing&pendingText=master%20-%20Pending
   :target:  https://ci.appveyor.com/project/lewissbaker/cppcoro/branch/master

- Linux + Clang 5.0/6.0 + libc++ |Linux 构建状态|

.. |Linux 构建状态| image:: https://travis-ci.org/lewissbaker/cppcoro.svg?branch=master
   :target: https://travis-ci.org/lewissbaker/cppcoro

Linux 除了 ``io_context`` 和 文件 I/O 相关的类没有实现外，其余功能都是可用的。（详情请参阅  `#15 <https://github.com/lewissbaker/cppcoro/issues/15>`_ ）

协程类型
****************************************

task<T>
========================================

一个 task 代表一个异步的延迟计算，因为它直到被等待时才会开始运行（而不是创建时）。

比如：

.. code-block:: cpp

   #include <cppcoro/read_only_file.hpp>
   #include <cppcoro/task.hpp>

   cppcoro::task<int> count_lines(std::string path)
   {
   auto file = co_await cppcoro::read_only_file::open(path);

   int lineCount = 0;

   char buffer[1024];
   size_t bytesRead;
   std::uint64_t offset = 0;
   do
   {
      bytesRead = co_await file.read(offset, buffer, sizeof(buffer));
      lineCount += std::count(buffer, buffer + bytesRead, '\n');
      offset += bytesRead;
   } while (bytesRead > 0);

   co_return lineCount;
   }

   cppcoro::task<> usage_example()
   {
   // 调用函数创建一个新的 task ，但是 task 这时候并没有开始运行
   // executing the coroutine yet.
   cppcoro::task<int> countTask = count_lines("foo.txt");

   // ...

   // 协程仅在被 co_await 后才开始运行
   int lineCount = co_await countTask;

   std::cout << "line count = " << lineCount << std::endl;
   }

API 概览：

.. code-block:: cpp

   // <cppcoro/task.hpp>
   namespace cppcoro
   {
   template<typename T>
   class task
   {
   public:

      using promise_type = <unspecified>;
      using value_type = T;

      task() noexcept;

      task(task&& other) noexcept;
      task& operator=(task&& other);

      // task 是一个只能被移动的类型
      task(const task& other) = delete;
      task& operator=(const task& other) = delete;

      // 查询 task 是否已经准备好了
      bool is_ready() const noexcept;

      // 等待 task 运行完毕
      // 如果 task 执行时出现了未捕获的异常，那么将其重新抛出
      // 
      // 如果任务还没有准备好，那么挂起直到 task 完成，如果 task is_ready() ，那么直接返回异步计算的结果
      Awaiter<T&> operator co_await() const & noexcept;
      Awaiter<T&&> operator co_await() const && noexcept;

      // 返回一个 awaitable 对象，以便于 co_await 暂停协程直至 task 完成
      //
      // 与表达式 ``co_await t`` 不同的是，``co_await t.when_ready()`` 中的 when_ready() 是同步的，而且不会返回计算结果，或者是重新抛出异常
      Awaitable<void> when_ready() const noexcept;
   };

   template<typename T>
   void swap(task<T>& a, task<T>& b);

   // Creates a task that yields the result of co_await'ing the specified awaitable.
   //
   // This can be used as a form of type-erasure of the concrete awaitable, allowing
   // different awaitables that return the same await-result type to be stored in
   // the same task<RESULT> type.
   template<
      typename AWAITABLE,
      typename RESULT = typename awaitable_traits<AWAITABLE>::await_result_t>
   task<RESULT> make_task(AWAITABLE awaitable);
   }

你可以通过调用返回值为 ``task<T>`` 的函数来产生 ``task<T>`` 对象。

协程必须包含 ``co_await`` 或 ``co_return`` 。

.. note:: 

   ``task<T>`` 也许不使用 ``co_yield`` 关键字

当一个返回值为 ``task<T>`` 的协程被调用时，如果需要，将会获得一个协程帧。协程的参数在协程帧内完成捕获。然后协程将会在函数起始处被暂停，并返回一个用于表示异步计算结果的 ``task<T>`` 。

在 ``task<T>`` 值被 ``co_await`` 后，协程将开始执行计算。然后等待的协成将被挂起，然后执行与 ``task<T>`` 相关联的协程。挂起的协程将在其关联的 ``task<T>`` 被 ``co_await`` 后唤醒。此线程要么 ``co_return``，要么跑出异常并被终止。

如果 task 已经完成，那么再次等待它将获得已经计算的结果，而不会重新计算。

如果 ``task`` 对象在被 co_await 之前就被销毁了，那么协程永远不会被执行。析构函数只会简单地释放协程帧内由于捕获参数而分配的内存。

shared_task<T>
========================================

协程类 ``shared_task<T>`` 以异步、惰性的方式产生单个值。

所谓 **惰性**，就是指仅当有协程 await 它的时候才开始执行计算。

它是 **共享** 的：task 允许被拷贝； task 的返回值可以被多次引用； task 可以被多个协程 await。

它在第一次被 co_await 时执行，其余 await 的协程要么挂起进入等待队列，要么直接拿到已经计算的结果。

如果协程由于 await task 被挂起，那么其将会在 task 完成计算后被唤醒。task 要么 ``co_return`` 一个值，要么抛出一个未捕获的异常。

API 摘要:

.. code-block:: cpp

   namespace cppcoro
   {
   template<typename T = void>
   class shared_task
   {
   public:

      using promise_type = <unspecified>;
      using value_type = T;

      shared_task() noexcept;
      shared_task(const shared_task& other) noexcept;
      shared_task(shared_task&& other) noexcept;
      shared_task& operator=(const shared_task& other) noexcept;
      shared_task& operator=(shared_task&& other) noexcept;

      void swap(shared_task& other) noexcept;

      // 查询 task 是否已经完成，而且计算结果已经可用
      bool is_ready() const noexcept;

      // 返回一个 operation，其将会在被 await 时挂起当前协程，直到 task 完成而且计算结果可用。
      //
      // 表达式 ``co_await someTask`` 的结果是一个指向 task 计算结果的左值引用（除非 T 的类
      // 型是 void，此时这个表达式的结果类型为 void）
      // 未捕获异常将被 co_await 表达式重新抛出
      Awaiter<T&> operator co_await() const noexcept;
      // 返回一个 operation，其将会在被 await 时挂起当前协程，直到 task 完成而且计算结果可用
      // 此 co_await 表达式不会返回任何值。
      // 此表达式可用于与 task 进行同步而不用担心抛出异常。
      Awaiter<void> when_ready() const noexcept;

   };

   template<typename T>
   bool operator==(const shared_task<T>& a, const shared_task<T>& b) noexcept;
   template<typename T>
   bool operator!=(const shared_task<T>& a, const shared_task<T>& b) noexcept;

   template<typename T>
   void swap(shared_task<T>& a, shared_task<T>& b) noexcept;

   // 包装一个可 await 的值，以允许多个协程同时等待它
   template<
      typename AWAITABLE,
      typename RESULT = typename awaitable_traits<AWAITABLE>::await_result_t>
   shared_task<RESULT> make_shared_task(AWAITABLE awaitable);
   }

const 限定的函数可以安全地在多个线程中调用，是线程安全的，但是非 const 限定的函数则不然。

.. note:: 

   与 ``task<T>`` 相比而言：

   - 都是延迟计算：计算只在被 co_await 后才开始。
   - task<T> 的结果不允许被拷贝，是仅移动的。而 shared_task 可以被拷贝和移动
   - 由于可能被共享，shared_task 的结果总是左值，这可能导致局部变量无法进行'移动构造'，而且由于需要维护引用计数，其运行时成本略高。

gengrator<T>
========================================

一个 :abbr:`生成器 (Gengrator)` 用于产生一系列类型为 T 的值。值的产生是惰性和异步的。

协程可以使用 ``co_yield`` 来产生一个类型为 T 的值。但是协程内无法使用 co_await 关键字。值的产生必须是同步的。

.. code-block:: none

   cppcoro::generator<const std::uint64_t> fibonacci()
   {
   std::uint64_t a = 0, b = 1;
   while (true)
   {
      co_yield b;
      auto tmp = a;
      a = b;
      b += tmp;
   }
   }

   void usage()
   {
   for (auto i : fibonacci())
   {
      if (i > 1'000'000) break;
      std::cout << i << std::endl;
   }
   }

当一个返回值为``generator<T>`` 的协程函数被调用后，其会被立即挂起。直到 ``generator<T>::begin()`` 函数被调用。在 ``co_yield`` 达到终点或者协程完成后不在产生值。

如果返回的迭代器与 ``end()`` 不相等，那么对迭代器进行解引用将会返回'传递给 ``co_yield`` '的值。

调用 ``operator()++`` 将会恢复协程的运行，直至协程结束或 co_yield 不再产生新的值。

API 摘要:

.. code-block:: cpp

   namespace cppcoro
   {
      template<typename T>
      class generator
      {
      public:

         using promise_type = <unspecified>;

         class iterator
         {
         public:
               using iterator_category = std::input_iterator_tag;
               using value_type = std::remove_reference_t<T>;
               using reference = value_type&;
               using pointer = value_type*;
               using difference_type = std::size_t;

               iterator(const iterator& other) noexcept;
               iterator& operator=(const iterator& other) noexcept;

               // 如果异常在 co_yield 之前参数，异常将被重新抛出。
               iterator& operator++();

               reference operator*() const noexcept;
               pointer operator->() const noexcept;

               bool operator==(const iterator& other) const noexcept;
               bool operator!=(const iterator& other) const noexcept;
         };

         // 构造一个空的序列
         generator() noexcept;

         generator(generator&& other) noexcept;
         generator& operator=(generator&& other) noexcept;

         generator(const generator& other) = delete;
         generator& operator=(const generator&) = delete;

         ~generator();

         // 开始执行协成，直至 co_yield 不再产生新的值或协程结束或未捕获异常被抛出
         iterator begin();

         iterator end() noexcept;

         // 交换两个生成器
         void swap(generator& other) noexcept;

      };

      template<typename T>
      void swap(generator<T>& a, generator<T>& b) noexcept;

      // 以 source 为基础，对其每个元素调用一次 func 来产生一个新的序列。
      template<typename FUNC, typename T>
      generator<std::invoke_result_t<FUNC, T&>> fmap(FUNC func, generator<T> source);
   }

recursive_generator<T>
========================================

相比生成器而言， :abbr:`递归生成器 (Recursive Generator)` 能够生成嵌套在外部元素的序列。

``co_yield`` 除了可以生成类型 T 的元素外，还能生成一个元素为 T 的递归生成器。

当你 ``co_yield`` 一个递归生成器时，其将被作为当前元素的子元素。当前线程将被挂起，直至递归生成器的所有元素被生成。然后被唤醒，等待请求下一个元素。

相比普通生成器而言，在迭代嵌套数据结构时，递归生成器能够通过 ``iterator::operator++()`` 直接唤醒边缘协程以产生下一个元素，而不必未每个元素都暂停/唤醒一个 O(depth) 的协程。缺点是有额外开销。

例子：

.. code-block:: cpp

   // 列出当前目录的内容
   cppcoro::generator<dir_entry> list_directory(std::filesystem::path path);

   cppcoro::recursive_generator<dir_entry> list_directory_recursive(std::filesystem::path path)
   {
   for (auto& entry : list_directory(path))
   {
      co_yield entry;
      if (entry.is_directory())
      {
         co_yield list_directory_recursive(entry.path());
      }
   }
   }

.. important::

   对 ``recursive_generator<T>`` 应用 ``fmap()`` 操作时，将产生 ``generator<U>`` 类型，而不是 ``recursive_generator<U>`` 类型。这是因为通常在递归上下文中不使用 ``fmap`` 操作，我们避免递归生成器的格外开销。

async_generator<T>
========================================

:abbr:`异步生成器 (Async Generator)` 用于产生类型为 T 的序列。值是惰性、异步产生的。

此协程体内既可以使用 ``co_wait`` 也可以使用 ``co_yield``

可以通过基于 ``for co_await`` 来处理数据序列。

比如：

.. code-block:: cpp

   cppcoro::async_generator<int> ticker(int count, threadpool& tp)
   {
   for (int i = 0; i < count; ++i)
   {
      co_await tp.delay(std::chrono::seconds(1));
      co_yield i;
   }
   }

   cppcoro::task<> consumer(threadpool& tp)
   {
   auto sequence = ticker(10, tp);
   for co_await(std::uint32_t i : sequence)
   {
      std::cout << "Tick " << i << std::endl;
   }
   }

API 摘要:

.. code-block:: cpp

   // <cppcoro/async_generator.hpp>
   namespace cppcoro
   {
   template<typename T>
   class async_generator
   {
   public:

      class iterator
      {
      public:
         using iterator_tag = std::forward_iterator_tag;
         using difference_type = std::size_t;
         using value_type = std::remove_reference_t<T>;
         using reference = value_type&;
         using pointer = value_type*;

         iterator(const iterator& other) noexcept;
         iterator& operator=(const iterator& other) noexcept;

         // 如果协程被挂起，则唤醒它
         // 返回一个 operation ，其必须被 await 至自增操作结束
         // 最后返回的迭代器与 end() 相等
         // 若有未捕获异常，则将其抛出
         Awaitable<iterator&> operator++() noexcept;

         // 对迭代器解引用
         pointer operator->() const noexcept;
         reference operator*() const noexcept;

         bool operator==(const iterator& other) const noexcept;
         bool operator!=(const iterator& other) const noexcept;
      };

      // 构造一个空的序列
      async_generator() noexcept;
      async_generator(const async_generator&) = delete;
      async_generator(async_generator&& other) noexcept;
      ~async_generator();

      async_generator& operator=(const async_generator&) = delete;
      async_generator& operator=(async_generator&& other) noexcept;

      void swap(async_generator& other) noexcept;

      // 开始执行协程并返回起一个 operation，其必须被 await 至第一个元素可用
      // co_wait 获得的是一个迭代器对象，并且可用其来推动协程的执行
      // 在协程执行结束后，调用此函数是非法的
      Awaitable<iterator> begin() noexcept;
      iterator end() noexcept;

   };

   template<typename T>
   void swap(async_generator<T>& a, async_generator<T>& b);

   // 以 source 为基础，对其每个元素调用一次 func 来产生一个新的序列。
   template<typename FUNC, typename T>
   async_generator<std::invoke_result_t<FUNC, T&>> fmap(FUNC func, async_generator<T> source);
   }

.. important:: 

   异步迭代器的提前终止：

   当异步生成器被析构时，它将请求取消协程。如果协程已经运行结束，或者在 ``co_yield`` 表达式中挂起，那么协程立即被销毁。否则协程将继续执行，直到它运行结束或到达下一个 ``co_yield`` 表达式。

   在协程被销毁时，其作用于内的所有变量也将被销毁，以确保完全清理资源。

   在协程使用 ``co_await`` 等待下一个元素时，调用者必须确保此时异步生成器不被销毁。

异步类型
****************************************

single_consumer_event
========================================

这是一个简单的手动重置事件类型。其在同一时间内只能被一个协程等待。

API 摘要：

.. code-block:: cpp

   // <cppcoro/single_consumer_event.hpp>
   namespace cppcoro
   {
   class single_consumer_event
   {
   public:
      single_consumer_event(bool initiallySet = false) noexcept;
      bool is_set() const noexcept;
      void set();
      void reset() noexcept;
      Awaiter<void> operator co_await() const noexcept;
   };
   }

例子：

.. code-block:: cpp

   #include <cppcoro/single_consumer_event.hpp>

   cppcoro::single_consumer_event event;
   std::string value;

   cppcoro::task<> consumer()
   {
   // 协程将会在此处挂起，直至有线程调用 event.set()
   // 比如下面的 producer() 函数
   co_await event;

   std::cout << value << std::endl;
   }

   void producer()
   {
   value = "foo";

   // This will resume the consumer() coroutine inside the call to set()
   // if it is currently suspended.
   event.set();
   }

single_consumer_async_auto_reset_event
========================================

这个类提供了一个异步同步原语以允许单个协程等待事件至信号发射。信号可以通过调用 ``set()`` 函数被发射。

一旦等待事件的协程被前面或后面对 ``set()`` 的调用释放，事件就会自动重置回 'not set' 状态。

相比 ``async_auto_reset_event`` 而言，本类更有效率，本类在同一时间仅允许一个类进入等待状态。如果你需要多个协程在同一时间等待时间，请使用 ``async_auto_reset_event`` 。

API 摘要：

.. code-block:: cpp

   // <cppcoro/single_consumer_async_auto_reset_event.hpp>
   namespace cppcoro
   {
   class single_consumer_async_auto_reset_event
   {
   public:

      single_consumer_async_auto_reset_event(
         bool initiallySet = false) noexcept;

      // 将事件的状态改为 'set' 。等待此事件的协程将被立即唤醒，之后事件状态自动重置为 'not set'
      void set() noexcept;

      // Returns an Awaitable type that can be awaited to wait until
      // the event becomes 'set' via a call to the .set() method. If
      // the event is already in the 'set' state then the coroutine
      // continues without suspending.
      // The event is automatically reset back to the 'not set' state
      // before resuming the coroutine.
      Awaiter<void> operator co_await() const noexcept;

   };
   }

例子:

.. code-block:: cpp

   std::atomic<int> value;
   cppcoro::single_consumer_async_auto_reset_event valueDecreasedEvent;

   cppcoro::task<> wait_until_value_is_below(int limit)
   {
   while (value.load(std::memory_order_relaxed) >= limit)
   {
      // 在此等待至 valueDecreasedEvent 事件的状态变为 set
      co_await valueDecreasedEvent;
   }
   }

   void change_value(int delta)
   {
   value.fetch_add(delta, std::memory_order_relaxed);
   // 如果此处 valueDecreasedEvent 状态发生改变，则通知挂起的协程
   if (delta < 0) valueDecreasedEvent.set();
   }

async_mutex
========================================

提供了一个简单的互斥抽象，允许调用者在协程中 ``co_await`` 互斥锁，挂起协程，直到获得互斥锁。

这个实现是无锁的，因为等待互斥锁的协程不会阻塞线程，而是挂起协程，然后前一个锁持有者通过调用 unlock() 来唤醒它。

API 摘要：

.. code-block:: cpp

   // <cppcoro/async_mutex.hpp>
   namespace cppcoro
   {
   class async_mutex_lock;
   class async_mutex_lock_operation;
   class async_mutex_scoped_lock_operation;

   class async_mutex
   {
   public:
      async_mutex() noexcept;
      ~async_mutex();

      async_mutex(const async_mutex&) = delete;
      async_mutex& operator(const async_mutex&) = delete;

      bool try_lock() noexcept;
      async_mutex_lock_operation lock_async() noexcept;
      async_mutex_scoped_lock_operation scoped_lock_async() noexcept;
      void unlock();
   };

   class async_mutex_lock_operation
   {
   public:
      bool await_ready() const noexcept;
      bool await_suspend(std::experimental::coroutine_handle<> awaiter) noexcept;
      void await_resume() const noexcept;
   };

   class async_mutex_scoped_lock_operation
   {
   public:
      bool await_ready() const noexcept;
      bool await_suspend(std::experimental::coroutine_handle<> awaiter) noexcept;
      [[nodiscard]] async_mutex_lock await_resume() const noexcept;
   };

   class async_mutex_lock
   {
   public:
      // 获得锁的所有权
      async_mutex_lock(async_mutex& mutex, std::adopt_lock_t) noexcept;

      // 移交锁的所有权
      async_mutex_lock(async_mutex_lock&& other) noexcept;

      async_mutex_lock(const async_mutex_lock&) = delete;
      async_mutex_lock& operator=(const async_mutex_lock&) = delete;

      // 通过调用 unlock() 来解锁
      ~async_mutex_lock();
   };
   }

例子：

.. code-block:: cpp

   #include <cppcoro/async_mutex.hpp>
   #include <cppcoro/task.hpp>
   #include <set>
   #include <string>

   cppcoro::async_mutex mutex;
   std::set<std::string> values;

   cppcoro::task<> add_item(std::string value)
   {
   cppcoro::async_mutex_lock lock = co_await mutex.scoped_lock_async();
   values.insert(std::move(value));
   }

async_manual_reset_event
========================================

一个手动重置的事件，是一个 协程/线程 同步原语。其允许多个协程进入等待状态，直至事件通过调用 ``set()`` 函数改变状态。

此事件永源处于 'set' 或 'not set' 状态之一。

如果协程在等待前事件已经是 'set' 状态，那么协程不会进入等待状态。否则，协程将会被挂起，直至事件状态通过 ``set()`` 函数被更换为 'set' 。

当事件状态改变为 'set' 时，所有由于等待事件被挂起的线程都会被某个线程唤醒。

.. important:: 

   请注意，当事件被销毁时，必须确保没有协程由于等待事件被挂起，因为它们将永远不会被唤醒。

API 摘要：

.. code-block:: cpp

   namespace cppcoro
   {
   class async_manual_reset_event_operation;

   class async_manual_reset_event
   {
   public:
      async_manual_reset_event(bool initiallySet = false) noexcept;
      ~async_manual_reset_event();

      async_manual_reset_event(const async_manual_reset_event&) = delete;
      async_manual_reset_event(async_manual_reset_event&&) = delete;
      async_manual_reset_event& operator=(const async_manual_reset_event&) = delete;
      async_manual_reset_event& operator=(async_manual_reset_event&&) = delete;

      // Wait until the event becomes set.
      async_manual_reset_event_operation operator co_await() const noexcept;

      bool is_set() const noexcept;

      void set() noexcept;

      void reset() noexcept;

   };

   class async_manual_reset_event_operation
   {
   public:
      async_manual_reset_event_operation(async_manual_reset_event& event) noexcept;

      bool await_ready() const noexcept;
      bool await_suspend(std::experimental::coroutine_handle<> awaiter) noexcept;
      void await_resume() const noexcept;
   };
   }

例子：

.. code-block:: cpp

   cppcoro::async_manual_reset_event event;
   std::string value;

   void producer()
   {
   value = get_some_string_value();

   // 通过设置事件来发布一个值
   event.set();
   }

   // 能够被调用多次以产生多个 task
   // 所有的 consumer task 将会等待至值被发布
   cppcoro::task<> consumer()
   {
   // 等待至值被事件发布
   co_await event;

   consume_value(value);
   }

async_auto_reset_event
========================================

一个手动重置的事件，是一个 协程/线程 同步原语。其允许多个协程进入等待状态，直至事件通过调用 ``set()`` 函数改变状态。

一旦由于等待事件被挂起的线程被唤醒，则事件自动进入 'not set' 状态。

API 摘要：

.. code-block:: cpp

   // <cppcoro/async_auto_reset_event.hpp>
   namespace cppcoro
   {
   class async_auto_reset_event_operation;

   class async_auto_reset_event
   {
   public:

      async_auto_reset_event(bool initiallySet = false) noexcept;

      ~async_auto_reset_event();

      async_auto_reset_event(const async_auto_reset_event&) = delete;
      async_auto_reset_event(async_auto_reset_event&&) = delete;
      async_auto_reset_event& operator=(const async_auto_reset_event&) = delete;
      async_auto_reset_event& operator=(async_auto_reset_event&&) = delete;

      // 等待至事件进入 'set' 状态
      // 
      // 如果事件已经是 'set' 状态了，则事件自动进入 'not set' 状态，而且 await 的协程
      // 会继续执行而不是挂起。
      // 否则，协程将被挂起至一些线程调用 'set()' 函数
      //
      // 注意：挂起的线程可因 'set()' 调用或者其他线程调用 'operator co_await()' 而被唤醒。
      async_auto_reset_event_operation operator co_await() const noexcept;

      // 将事件的状态更改为 'set'
      //
      // 如果有因等待事件被挂起的协程，则其中之一会被唤醒，然后事件自动进入 'not set' 状态
      //
      // 如果事件已经为 'set' 状态，则此函数不进行任何操作。
      void set() noexcept;

      // 设置事件状态为 'not set'
      //
      // 如果事件已经为 'not set' 状态，则此函数不进行任何操作。
      void reset() noexcept;

   };

   class async_auto_reset_event_operation
   {
   public:
      explicit async_auto_reset_event_operation(async_auto_reset_event& event) noexcept;
      async_auto_reset_event_operation(const async_auto_reset_event_operation& other) noexcept;

      bool await_ready() const noexcept;
      bool await_suspend(std::experimental::coroutine_handle<> awaiter) noexcept;
      void await_resume() const noexcept;

   };
   }

async_latch
========================================

:abbr:`异步锁存器 (Async Latch)` 是一个同步原语，用于异步等待一个计数器递减为零。

此锁存器是一次性的。一旦由于计数器变为零导致锁存器进入 ready 状态，其将保持此状态直至销毁。

API 摘要：

.. code-block:: cpp

   // <cppcoro/async_latch.hpp>
   namespace cppcoro
   {
   class async_latch
   {
   public:

      // 用指定的计数初始化此锁存器
      async_latch(std::ptrdiff_t initialCount) noexcept;

      // 查询计数是否已经变为零
      bool is_ready() const noexcept;

      // 将计数减少 n
      // 当此函数的调用导致计数为零时，所有等待的协程将被唤醒
      // 计数器减到负值是未定义行为
      void count_down(std::ptrdiff_t n = 1) noexcept;

      // 等待锁存器状态变为 ready
      // 如果计数没有变为零，则所有等待的协程将被挂起，直至由于调用 count_down() 导致计数变为零。
      // 如果计数已经变为零，则不会被挂起
      Awaiter<void> operator co_await() const noexcept;

   };
   }

sequence_barrier
========================================

:abbr:`顺序墙 (Sequence Barrier)` 是一个同步原语，允许一个生产者和多个消费者之间通过一个单调递增的数字序列来协作。

生产者通过发布一组单调递增的数来推进序列，消费者则可以查询生产者最后发布的数，并可以等待至特定的数被发布。

顺序墙可以充当线程安全的生产-消费环形缓存区的游标。

有关更多信息，参见 LMAX Disruptor 模式：https://lmax-exchange.github.io/disruptor/files/Disruptor-1.0.pdf

API 摘要：

.. code-block:: cpp

   namespace cppcoro
   {
   template<typename SEQUENCE = std::size_t,
            typename TRAITS = sequence_traits<SEQUENCE>>
   class sequence_barrier
   {
   public:
      sequence_barrier(SEQUENCE initialSequence = TRAITS::initial_sequence) noexcept;
      ~sequence_barrier();

      SEQUENCE last_published() const noexcept;

      // 等待至序列号 targetSequence 被发布
      //
      // 如果操作没有同步地完成，则等待的协程将被特定的 scheduler 唤醒。否则协程将直接顺序执行而无需等待
      //
      // co_await 表达式将在 last_published() 后被唤醒，最后发布的数就是其 targetSequence
      template<typename SCHEDULER>
      [[nodiscard]]
      Awaitable<SEQUENCE> wait_until_published(SEQUENCE targetSequence,
                                                SCHEDULER& scheduler) const noexcept;

      void publish(SEQUENCE sequence) noexcept;
   };
   }

single_producer_sequencer
========================================

:abbr:`单消费者序列起 (Single Producer Sequencer)` 是一个同步原语，可以协调单个生产者和多个消费者对环状缓冲区的访问。

生产者首先向环状缓冲区请求一个或多个槽，然后在槽内写入数据，最终发布这些槽的序列号。生产的数据和未消费的数据之和应小于 bufferSize

消费者等待某些元素的发布，处理这些元素，然后通过发布在 sequence_barrier_ 对象中完成消费的序列号来通知生产者完成了对这些元素的处理。

API 摘要：

.. code-block:: cpp

   // <cppcoro/single_producer_sequencer.hpp>
   namespace cppcoro
   {
   template<
      typename SEQUENCE = std::size_t,
      typename TRAITS = sequence_traits<SEQUENCE>>
   class single_producer_sequencer
   {
   public:
      using size_type = typename sequence_range<SEQUENCE, TRAITS>::size_type;

      single_producer_sequencer(
         const sequence_barrier<SEQUENCE, TRAITS>& consumerBarrier,
         std::size_t bufferSize,
         SEQUENCE initialSequence = TRAITS::initial_sequence) noexcept;

      // 生产者 API:

      template<typename SCHEDULER>
      [[nodiscard]]
      Awaitable<SEQUENCE> claim_one(SCHEDULER& scheduler) noexcept;

      template<typename SCHEDULER>
      [[nodiscard]]
      Awaitable<sequence_range<SEQUENCE>> claim_up_to(
         std::size_t count,
         SCHEDULER& scheduler) noexcept;

      void publish(SEQUENCE sequence) noexcept;

      // 消费者 API:

      SEQUENCE last_published() const noexcept;

      template<typename SCHEDULER>
      [[nodiscard]]
      Awaitable<SEQUENCE> wait_until_published(
         SEQUENCE targetSequence,
         SCHEDULER& scheduler) const noexcept;

   };
   }

例子：

.. code-block:: none

   using namespace cppcoro;
   using namespace std::chrono;

   struct message
   {
   int id;
   steady_clock::time_point timestamp;
   float data;
   };

   constexpr size_t bufferSize = 16384; // 必须为 2 的幂
   constexpr size_t indexMask = bufferSize - 1;
   message buffer[bufferSize];

   task<void> producer(
   io_service& ioSvc,
   single_producer_sequencer<size_t>& sequencer)
   {
   auto start = steady_clock::now();
   for (int i = 0; i < 1'000'000; ++i)
   {
      // 等待缓冲区内一个可用的槽
      size_t seq = co_await sequencer.claim_one(ioSvc);

      // 填充数据
      auto& msg = buffer[seq & indexMask];
      msg.id = i;
      msg.timestamp = steady_clock::now();
      msg.data = 123;

      // 发布数据
      sequencer.publish(seq);
   }

   // 发布终止序列号
   auto seq = co_await sequencer.claim_one(ioSvc);
   auto& msg = buffer[seq & indexMask];
   msg.id = -1;
   sequencer.publish(seq);
   }

   task<void> consumer(
   static_thread_pool& threadPool,
   const single_producer_sequencer<size_t>& sequencer,
   sequence_barrier<size_t>& consumerBarrier)
   {
   size_t nextToRead = 0;
   while (true)
   {
      // 等待只下一个数据可用
      // 也许有多个数据可用
      const size_t available = co_await sequencer.wait_until_published(nextToRead, threadPool);
      do {
         auto& msg = buffer[nextToRead & indexMask];
         if (msg.id == -1)
         {
         consumerBarrier.publish(nextToRead);
         co_return;
         }

         processMessage(msg);
      } while (nextToRead++ != available);

      // 通知生产者我们已经处理到了 'nextToRead - 1'
      consumerBarrier.publish(available);
   }
   }

   task<void> example(io_service& ioSvc, static_thread_pool& threadPool)
   {
   sequence_barrier<size_t> barrier;
   single_producer_sequencer<size_t> sequencer{barrier, bufferSize};

   co_await when_all(
      producer(tp, sequencer),
      consumer(tp, sequencer, barrier));
   }

multi_producer_sequencer
========================================

:abbr:`多生产序列器 (Multi Producer Sequencer)`  是一个同步原语，可以协调多个生产者和消费者之间对环状缓冲区的访问。

对于单个生产者的变体，请参阅 single_producer_sequencer_ 

.. important:: 

   环状缓冲区的大小必须为 2 的幂。这是因为此算法实现使用了位掩码来计算缓冲区的偏移值，而不是使用摸运算。而且，这允许序列号被 32/64位值包装。

API 摘要：

.. code-block:: cpp

   // <cppcoro/multi_producer_sequencer.hpp>
   namespace cppcoro
   {
   template<typename SEQUENCE = std::size_t,
            typename TRAITS = sequence_traits<SEQUENCE>>
   class multi_producer_sequencer
   {
   public:
      multi_producer_sequencer(
         const sequence_barrier<SEQUENCE, TRAITS>& consumerBarrier,
         SEQUENCE initialSequence = TRAITS::initial_sequence);

      std::size_t buffer_size() const noexcept;

      // 消费者接口
      //
      // 每个消费者保持对他们独一的 'lastKnownPublished' 的追踪。并且需要传递 this 到
      // 此方法，以便于查询最后升级的、可用的序列号
      // Consumer interface

      SEQUENCE last_published_after(SEQUENCE lastKnownPublished) const noexcept;

      template<typename SCHEDULER>
      Awaitable<SEQUENCE> wait_until_published(
         SEQUENCE targetSequence,
         SEQUENCE lastKnownPublished,
         SCHEDULER& scheduler) const noexcept;

      // 生产者接口

      // 查询是否有可用的空间（近似值）
      bool any_available() const noexcept;

      template<typename SCHEDULER>
      Awaitable<SEQUENCE> claim_one(SCHEDULER& scheduler) noexcept;

      template<typename SCHEDULER>
      Awaitable<sequence_range<SEQUENCE, TRAITS>> claim_up_to(
         std::size_t count,
         SCHEDULER& scheduler) noexcept;

      // 标记这个特定的序列号为已发布
      void publish(SEQUENCE sequence) noexcept;

      // 标记范围内的序列号为已发布
      void publish(const sequence_range<SEQUENCE, TRAITS>& range) noexcept;
   };
   }


撤销操作
****************************************

cancellation_token
========================================

一个 :abbr:`撤销令牌 (Cancellation Token)` 是一个可以被传递给函数的值，以允许调用者其后请求撤销对该函数的调用。

要获得一个可用于撤销操作的 ``撤销令牌`` ，你首先需要创建一个``cancellation_source`` 对象。方法 ``cancellation_source::token()`` 可用于创建新的、与 ``cancellation_source`` 相关联的 ``撤销令牌`` 。

然后你可以通过 ``cancellation_source::request_cancellation()`` 向 ``cancellation_source`` 传递相关联的撤销令牌，以撤销对函数的调用。

函数以以下两种方式来获得是否有撤销请求：

#. 定期调用 ``cancellation_token::is_cancellation_requested()`` 或 ``cancellation_token::throw_if_cancellation_requested()``
#. 使用 ``cancellation_registration`` 类注册一个 *在出现撤销请求时调用的回调函数* 。

API 摘要：

.. code-block:: cpp

   namespace cppcoro
   {
   class cancellation_source
   {
   public:
      // 构造一个新的、独立的、可撤销的 cancellation_source
      cancellation_source();

      // 构造一个与 other 撤销状态相同的新引用
      cancellation_source(const cancellation_source& other) noexcept;
      cancellation_source(cancellation_source&& other) noexcept;

      ~cancellation_source();

      cancellation_source& operator=(const cancellation_source& other) noexcept;
      cancellation_source& operator=(cancellation_source&& other) noexcept;

      bool is_cancellation_requested() const noexcept;
      bool can_be_cancelled() const noexcept;
      void request_cancellation();

      cancellation_token token() const noexcept;
   };

   class cancellation_token
   {
   public:
      // 构造一个无法被撤销的令牌
      cancellation_token() noexcept;

      cancellation_token(const cancellation_token& other) noexcept;
      cancellation_token(cancellation_token&& other) noexcept;

      ~cancellation_token();

      cancellation_token& operator=(const cancellation_token& other) noexcept;
      cancellation_token& operator=(cancellation_token&& other) noexcept;

      bool is_cancellation_requested() const noexcept;
      void throw_if_cancellation_requested() const;

      // 查询此令牌是否有取消请求。
      // 当函数调用无需撤销时，代码可据此设置更有效的 code-path
      bool can_be_cancelled() const noexcept;
   };


   // RAII 类。用于注册在撤销时调用的回调函数
   class cancellation_registration
   {
   public:

      // 注册一个在撤销时调用的回调函数
      // 如果还没有请求撤销，则在调用 request_cancellation() 的线程上调用回调函数，否则在此线程内立即调用回调函数
      // 回调函数不得抛出异常
      template<typename CALLBACK>
      cancellation_registration(cancellation_token token, CALLBACK&& callback);

      cancellation_registration(const cancellation_registration& other) = delete;

      ~cancellation_registration();
   };

   class operation_cancelled : public std::exception
   {
   public:
      operation_cancelled();
      const char* what() const override;
   };
   }

例子：论询方式

.. code-block:: cpp

   cppcoro::task<> do_something_async(cppcoro::cancellation_token token)
   {
   // 通过调用 throw_if_cancellation_requested() 在函数内显式创建一个可撤销点
   token.throw_if_cancellation_requested();

   co_await do_step_1();

   token.throw_if_cancellation_requested();

   do_step_2();

   // 可选的。通过查询是否有撤销请求以在函数返回前进行清理工作
   if (token.is_cancellation_requested())
   {
      display_message_to_user("Cancelling operation...");
      do_cleanup();
      throw cppcoro::operation_cancelled{};
   }

   do_final_step();
   }

例子：回调方式

.. code-block:: cpp

   // 假设我们现在有一个定时器，其具有撤销 API，但是还不支持撤销令牌。
   // 我们可以使用一个 cancellation_registration 对象来注册一个回调函数，在回调函数内可调用已经存在的撤销 API
   cppcoro::task<> cancellable_timer_wait(cppcoro::cancellation_token token)
   {
   auto timer = create_timer(10s);

   cppcoro::cancellation_registration registration(token, [&]
   {
      // 调用已经存在的定时器撤销 API
      timer.cancel();
   });

   co_await timer;
   }

cancellation_source
========================================

此处原文档尚未更新

cancellation_registration
========================================

此处原文档尚未更新

I/O 调度
****************************************

static_thread_pool
========================================

:abbr:`静态线程池 (Static Thread Pool)` 提供了一个抽象，以允许你的调度工作运行在一个固定尺寸的线程池中。

这个类实现了 ``Scheduler`` 概念（如下）

你可以通过调用 ``co_await threadPool.schedule()`` 向线程池中添加任务。这个操作将会挂起当前协程、将它放入线程池的工作队列中。当线程池中有空闲线程可用于此协程时，将会恢复协程。 **此操作保证不发生异常。通常情况下也不会分配内存**

该类利用 :abbr:`工作窃取 (Work-Stealing)` 算法在多个线程之间实现负载均衡。从线程池线程进入到线程池的任务将被调度到此线程独自的 FIFO 队列中，这意味着任务将在与原线程相同的线程上执行。而从外部进入到线程池中的任务，将进入一个全局的 FIFO 队列中。如果一个工作线程在它的本地队列中已经完成了工作，它会首先从全局队列中出列。如果本地队列为空，它会排在全局队列末尾以窃取工作。

API 摘要：

.. code-block:: cpp

   namespace cppcoro
   {
   class static_thread_pool
   {
   public:
      // 初始化线程池对象并使用 std::thread::hardware_concurrency() 来设定线程池的线程数量
      static_thread_pool();

      // 以指定数量的线程来初始化线程池
      explicit static_thread_pool(std::uint32_t threadCount);

      std::uint32_t thread_count() const noexcept;

      class schedule_operation
      {
      public:
         schedule_operation(static_thread_pool* tp) noexcept;

         bool await_ready() noexcept;
         bool await_suspend(std::experimental::coroutine_handle<> h) noexcept;
         bool await_resume() noexcept;

      private:
         // 未指定
      };

      // 返回一个可被协程 await 的操作
      //
      //
      [[nodiscard]]
      schedule_operation schedule() noexcept;

   private:

      // 未指定

   };
   }

简单例子：

.. code-block:: cpp

   cppcoro::task<std::string> do_something_on_threadpool(cppcoro::static_thread_pool& tp)
   {
   // 首先将协程调入到线程池中
   co_await tp.schedule();

   // 当恢复时，此线程将运行在线程池中
   do_something();
   }

例子：对静态线程池运行 ``schedule_on()`` 以并行执行任务

.. code-block:: cpp

   cppcoro::task<double> dot_product(static_thread_pool& tp, double a[], double b[], size_t count)
   {
   if (count > 1000)
   {
      // 将任务递归地细分为两个相同大小的任务
      // 第一个任务将会运行到线程池中
      // 第二个任务在此线程中继续执行
      size_t halfCount = count / 2;
      auto [first, second] = co_await when_all(
         schedule_on(tp, dot_product(tp, a, b, halfCount),
         dot_product(tp, a + halfCount, b + halfCount, count - halfCount));
      co_return first + second;
   }
   else
   {
      double sum = 0.0;
      for (size_t i = 0; i < count; ++i)
      {
         sum += a[i] * b[i];
      }
      co_return sum;
   }
   }

io_service and io_work_scope
========================================

``io_service`` 类为处理异步 I/O 操作完成事件提供了抽象。

当 I/O 操作完成时，await 其的协程将会在 I/O 线程的以下事件处理函数中被恢复： ``process_events()`` 、 ``process_pending_events()`` 、 ``process_one_event()``  、 ``process_one_pending_event()`` 

``io_service`` 类不会管理任何 I/O 线程。你必须确保为 await 的协程而调用的事件处理函数被 dispatch。要么用独立的线程调用 ``process_events()`` ，要么将其与其他事件循环混用（比如 UI 事件循环）并周期性地调用 ``process_pending_events()`` 或 ``process_one_pending_event()``

``io_service`` 可以被集成到其他事件循环中，比如 UI 事件循环。

通过多个线程调用 ``process_events()`` 函数，你可以同时处理多个事件。在 ``io_service`` 构造时，你也可以为其传入一个参数，以示意最多应当能有多少个同时运行的事件处理函数。

在 Windows 上，此实现充分利用了 :abbr:`Windows I/O 完成端口 (Windows I/O Completion Port)` 以可拓展的方式向 I/O 线程分发事件。

API 摘要：

.. code-block:: cpp

   namespace cppcoro
   {
   class io_service
   {
   public:

      class schedule_operation;
      class timed_schedule_operation;

      io_service();
      io_service(std::uint32_t concurrencyHint);

      io_service(io_service&&) = delete;
      io_service(const io_service&) = delete;
      io_service& operator=(io_service&&) = delete;
      io_service& operator=(const io_service&) = delete;

      ~io_service();

      // 调度器方法

      [[nodiscard]]
      schedule_operation schedule() noexcept;

      template<typename REP, typename RATIO>
      [[nodiscard]]
      timed_schedule_operation schedule_after(
         std::chrono::duration<REP, RATIO> delay,
         cppcoro::cancellation_token cancellationToken = {}) noexcept;

      // 事件循环方法
      //
      // I/O 线程必须调用这些方法来处理 I/O 事件并运行被调度的线程
      // scheduled coroutines.

      std::uint64_t process_events();
      std::uint64_t process_pending_events();
      std::uint64_t process_one_event();
      std::uint64_t process_one_pending_event();

      // 这里要求所有事件处理线程都已经退出它们的事件循环
      void stop() noexcept;

      // 查询孙是否有线程调用过 stop()
      bool is_stop_requested() const noexcept;

      // 重置调用过 stop() 的事件循环，以让其可以再次处理事件
      void reset();

      // 使用引用计数的方式追踪外部对 io_service 的引用
      //
      // 当引用计数递减为零时，io_service::stop() 将被调用
      //
      // 当 进入/退出 作用域时，使用 RAII 类 io_work_scope 来管理这些方法的调用
      void notify_work_started() noexcept;
      void notify_work_finished() noexcept;

   };

   class io_service::schedule_operation
   {
   public:
      schedule_operation(const schedule_operation&) noexcept;
      schedule_operation& operator=(const schedule_operation&) noexcept;

      bool await_ready() const noexcept;
      void await_suspend(std::experimental::coroutine_handle<> awaiter) noexcept;
      void await_resume() noexcept;
   };

   class io_service::timed_schedule_operation
   {
   public:
      timed_schedule_operation(timed_schedule_operation&&) noexcept;

      timed_schedule_operation(const timed_schedule_operation&) = delete;
      timed_schedule_operation& operator=(const timed_schedule_operation&) = delete;
      timed_schedule_operation& operator=(timed_schedule_operation&&) = delete;

      bool await_ready() const noexcept;
      void await_suspend(std::experimental::coroutine_handle<> awaiter);
      void await_resume();
   };

   class io_work_scope
   {
   public:

      io_work_scope(io_service& ioService) noexcept;

      io_work_scope(const io_work_scope& other) noexcept;
      io_work_scope(io_work_scope&& other) noexcept;

      ~io_work_scope();

      io_work_scope& operator=(const io_work_scope& other) noexcept;
      io_work_scope& operator=(io_work_scope&& other) noexcept;

      io_service& service() const noexcept;
   };

   }

例子：

.. code-block:: cpp

   #include <cppcoro/task.hpp>
   #include <cppcoro/task.hpp>
   #include <cppcoro/io_service.hpp>
   #include <cppcoro/read_only_file.hpp>

   #include <experimental/filesystem>
   #include <memory>
   #include <algorithm>
   #include <iostream>

   namespace fs = std::experimental::filesystem;

   cppcoro::task<std::uint64_t> count_lines(cppcoro::io_service& ioService, fs::path path)
   {
   auto file = cppcoro::read_only_file::open(ioService, path);

   constexpr size_t bufferSize = 4096;
   auto buffer = std::make_unique<std::uint8_t[]>(bufferSize);

   std::uint64_t newlineCount = 0;

   for (std::uint64_t offset = 0, fileSize = file.size(); offset < fileSize;)
   {
      const auto bytesToRead = static_cast<size_t>(
         std::min<std::uint64_t>(bufferSize, fileSize - offset));

      const auto bytesRead = co_await file.read(offset, buffer.get(), bytesToRead);

      newlineCount += std::count(buffer.get(), buffer.get() + bytesRead, '\n');

      offset += bytesRead;
   }

   co_return newlineCount;
   }

   cppcoro::task<> run(cppcoro::io_service& ioService)
   {
   cppcoro::io_work_scope ioScope(ioService);

   auto lineCount = co_await count_lines(ioService, fs::path{"foo.txt"});

   std::cout << "foo.txt has " << lineCount << " lines." << std::endl;;
   }

   cppcoro::task<> process_events(cppcoro::io_service& ioService)
   {
   // 处理事件至 io_service 被停止时
   // 比如：当最后一个 io_work_scope 退出作用域时
   ioService.process_events();
   co_return;
   }

   int main()
   {
   cppcoro::io_service ioService;

   cppcoro::sync_wait(cppcoro::when_all_ready(
      run(ioService),
      process_events(ioService)));

   return 0;
   }

**io_service 作为调度器**

``io_service`` 实现了 ``Scheduler`` 和 ``DelayedScheduler`` :abbr:`概念 (Concepts)` 

这允许协程在当前线程暂停运行，并在一个与 ``io_service`` 相关联的 I/O 线程中被唤醒。

例子：

.. code-block:: cpp

   cppcoro::task<> do_something(cppcoro::io_service& ioService)
   {
   // 协程在被 await 的线程中开始运行

   // 通过 await io_service::schedule() 的结果，协程可以转移到 I/O 线程中运行
   co_await ioService.schedule();

   // 此时，此协程运行在调用了 io_service 事件处理函数的 I/O 线程中

   // 协程也可以使用一个 Delayed-Schedule 的动作。当 I/O 线程恢复它时，它会延迟指定的时间。
   co_await ioService.schedule_after(100ms);

   // 此处，协程应该运行在一个不同的 I/O 线程中
   }

file, readable_file, writable_file
========================================

这些抽象基类用于表现具体的文件 I/O

API 摘要：

.. code-block:: cpp

   namespace cppcoro
   {
   class file_read_operation;
   class file_write_operation;

   class file
   {
   public:

      virtual ~file();

      std::uint64_t size() const;

   protected:

      file(file&& other) noexcept;

   };

   class readable_file : public virtual file
   {
   public:

      [[nodiscard]]
      file_read_operation read(
         std::uint64_t offset,
         void* buffer,
         std::size_t byteCount,
         cancellation_token ct = {}) const noexcept;

   };

   class writable_file : public virtual file
   {
   public:

      void set_size(std::uint64_t fileSize);

      [[nodiscard]]
      file_write_operation write(
         std::uint64_t offset,
         const void* buffer,
         std::size_t byteCount,
         cancellation_token ct = {}) noexcept;

   };

   class file_read_operation
   {
   public:

      file_read_operation(file_read_operation&& other) noexcept;

      bool await_ready() const noexcept;
      bool await_suspend(std::experimental::coroutine_handle<> awaiter);
      std::size_t await_resume();

   };

   class file_write_operation
   {
   public:

      file_write_operation(file_write_operation&& other) noexcept;

      bool await_ready() const noexcept;
      bool await_suspend(std::experimental::coroutine_handle<> awaiter);
      std::size_t await_resume();

   };
   }

read_only_file, write_only_file, read_write_file
==================================================

这些类型代表了具体的文件 I/O 类：

API 摘要：

.. code-block:: cpp

   namespace cppcoro
   {
   class read_only_file : public readable_file
   {
   public:

      [[nodiscard]]
      static read_only_file open(
         io_service& ioService,
         const std::experimental::filesystem::path& path,
         file_share_mode shareMode = file_share_mode::read,
         file_buffering_mode bufferingMode = file_buffering_mode::default_);

   };

   class write_only_file : public writable_file
   {
   public:

      [[nodiscard]]
      static write_only_file open(
         io_service& ioService,
         const std::experimental::filesystem::path& path,
         file_open_mode openMode = file_open_mode::create_or_open,
         file_share_mode shareMode = file_share_mode::none,
         file_buffering_mode bufferingMode = file_buffering_mode::default_);

   };

   class read_write_file : public readable_file, public writable_file
   {
   public:

      [[nodiscard]]
      static read_write_file open(
         io_service& ioService,
         const std::experimental::filesystem::path& path,
         file_open_mode openMode = file_open_mode::create_or_open,
         file_share_mode shareMode = file_share_mode::none,
         file_buffering_mode bufferingMode = file_buffering_mode::default_);

   };
   }

所有的 ``open()`` 函数在出错时都会抛出 ``std::system_error`` 异常。

网络
****************************************

注意:目前仅支持 Windows 平台上的网络抽象。Linux 支持将会在稍后推出。

socket
========================================

套接字类可用于通过网络异步发送/接收数据

当前只支持 IPv4 和 IPv6 上的 TCP/IP, UDP/IP 协议。

API 摘要：

.. code-block:: cpp

   // <cppcoro/net/socket.hpp>
   namespace cppcoro::net
   {
   class socket
   {
   public:

      static socket create_tcpv4(ip_service& ioSvc);
      static socket create_tcpv6(ip_service& ioSvc);
      static socket create_updv4(ip_service& ioSvc);
      static socket create_udpv6(ip_service& ioSvc);

      socket(socket&& other) noexcept;

      ~socket();

      socket& operator=(socket&& other) noexcept;

      // 返回套接字的平台相关的原声句柄
      <platform-specific> native_handle() noexcept;

      const ip_endpoint& local_endpoint() const noexcept;
      const ip_endpoint& remote_endpoint() const noexcept;

      void bind(const ip_endpoint& localEndPoint);

      void listen();

      [[nodiscard]]
      Awaitable<void> connect(const ip_endpoint& remoteEndPoint) noexcept;
      [[nodiscard]]
      Awaitable<void> connect(const ip_endpoint& remoteEndPoint,
                              cancellation_token ct) noexcept;

      [[nodiscard]]
      Awaitable<void> accept(socket& acceptingSocket) noexcept;
      [[nodiscard]]
      Awaitable<void> accept(socket& acceptingSocket,
                              cancellation_token ct) noexcept;

      [[nodiscard]]
      Awaitable<void> disconnect() noexcept;
      [[nodiscard]]
      Awaitable<void> disconnect(cancellation_token ct) noexcept;

      [[nodiscard]]
      Awaitable<std::size_t> send(const void* buffer, std::size_t size) noexcept;
      [[nodiscard]]
      Awaitable<std::size_t> send(const void* buffer,
                                 std::size_t size,
                                 cancellation_token ct) noexcept;

      [[nodiscard]]
      Awaitable<std::size_t> recv(void* buffer, std::size_t size) noexcept;
      [[nodiscard]]
      Awaitable<std::size_t> recv(void* buffer,
                                 std::size_t size,
                                 cancellation_token ct) noexcept;

      [[nodiscard]]
      socket_recv_from_operation recv_from(
         void* buffer,
         std::size_t size) noexcept;
      [[nodiscard]]
      socket_recv_from_operation_cancellable recv_from(
         void* buffer,
         std::size_t size,
         cancellation_token ct) noexcept;

      [[nodiscard]]
      socket_send_to_operation send_to(
         const ip_endpoint& destination,
         const void* buffer,
         std::size_t size) noexcept;
      [[nodiscard]]
      socket_send_to_operation_cancellable send_to(
         const ip_endpoint& destination,
         const void* buffer,
         std::size_t size,
         cancellation_token ct) noexcept;

      void close_send();
      void close_recv();

   };
   }

例子： echo 服务器

.. code-block:: cpp

   #include <cppcoro/net/socket.hpp>
   #include <cppcoro/io_service.hpp>
   #include <cppcoro/cancellation_source.hpp>
   #include <cppcoro/async_scope.hpp>
   #include <cppcoro/on_scope_exit.hpp>

   #include <memory>
   #include <iostream>

   cppcoro::task<void> handle_connection(socket s)
   {
   try
   {
      const size_t bufferSize = 16384;
      auto buffer = std::make_unique<unsigned char[]>(bufferSize);
      size_t bytesRead;
      do {
         // 读取一些字节
         bytesRead = co_await s.recv(buffer.get(), bufferSize);

         // 写入一些字节
         size_t bytesWritten = 0;
         while (bytesWritten < bytesRead) {
         bytesWritten += co_await s.send(
            buffer.get() + bytesWritten,
            bytesRead - bytesWritten);
         }
      } while (bytesRead != 0);

      s.close_send();

      co_await s.disconnect();
   }
   catch (...)
   {
      std::cout << "connection failed" << std::
   }
   }

   cppcoro::task<void> echo_server(
   cppcoro::net::ipv4_endpoint endpoint,
   cppcoro::io_service& ioSvc,
   cancellation_token ct)
   {
   cppcoro::async_scope scope;

   std::exception_ptr ex;
   try
   {
      auto listeningSocket = cppcoro::net::socket::create_tcpv4(ioSvc);
      listeningSocket.bind(endpoint);
      listeningSocket.listen();

      while (true) {
         auto connection = cppcoro::net::socket::create_tcpv4(ioSvc);
         co_await listeningSocket.accept(connection, ct);
         scope.spawn(handle_connection(std::move(connection)));
      }
   }
   catch (cppcoro::operation_cancelled)
   {
   }
   catch (...)
   {
      ex = std::current_exception();
   }

   // Wait until all handle_connection tasks have finished.
   co_await scope.join();

   if (ex) std::rethrow_exception(ex);
   }

   int main(int argc, const char* argv[])
   {
      cppcoro::io_service ioSvc;

      if (argc != 2) return -1;

      auto endpoint = cppcoro::ipv4_endpoint::from_string(argv[1]);
      if (!endpoint) return -1;

      (void)cppcoro::sync_wait(cppcoro::when_all(
         [&]() -> task<>
         {
               // Shutdown the event loop once finished.
               auto stopOnExit = cppcoro::on_scope_exit([&] { ioSvc.stop(); });

               cppcoro::cancellation_source canceller;
               co_await cppcoro::when_all(
                  [&]() -> task<>
                  {
                     // Run for 30s then stop accepting new connections.
                     co_await ioSvc.schedule_after(std::chrono::seconds(30));
                     canceller.request_cancellation();
                  }(),
                  echo_server(*endpoint, ioSvc, canceller.token()));
         }(),
         [&]() -> task<>
         {
               ioSvc.process_events();
         }()));

      return 0;
   }

ip_address, ipv4_address, ipv6_address
========================================

表示IP地址的辅助类

API 摘要：

.. code-block:: cpp

   namespace cppcoro::net
   {
   class ipv4_address
   {
      using bytes_t = std::uint8_t[4];
   public:
      constexpr ipv4_address();
      explicit constexpr ipv4_address(std::uint32_t integer);
      explicit constexpr ipv4_address(const std::uint8_t(&bytes)[4]);
      explicit constexpr ipv4_address(std::uint8_t b0,
                                       std::uint8_t b1,
                                       std::uint8_t b2,
                                       std::uint8_t b3);

      constexpr const bytes_t& bytes() const;

      constexpr std::uint32_t to_integer() const;

      static constexpr ipv4_address loopback();

      constexpr bool is_loopback() const;
      constexpr bool is_private_network() const;

      constexpr bool operator==(ipv4_address other) const;
      constexpr bool operator!=(ipv4_address other) const;
      constexpr bool operator<(ipv4_address other) const;
      constexpr bool operator>(ipv4_address other) const;
      constexpr bool operator<=(ipv4_address other) const;
      constexpr bool operator>=(ipv4_address other) const;

      std::string to_string();

      static std::optional<ipv4_address> from_string(std::string_view string) noexcept;
   };

   class ipv6_address
   {
      using bytes_t = std::uint8_t[16];
   public:
      constexpr ipv6_address();

      explicit constexpr ipv6_address(
         std::uint64_t subnetPrefix,
         std::uint64_t interfaceIdentifier);

      constexpr ipv6_address(
         std::uint16_t part0,
         std::uint16_t part1,
         std::uint16_t part2,
         std::uint16_t part3,
         std::uint16_t part4,
         std::uint16_t part5,
         std::uint16_t part6,
         std::uint16_t part7);

      explicit constexpr ipv6_address(
         const std::uint16_t(&parts)[8]);

      explicit constexpr ipv6_address(
         const std::uint8_t(bytes)[16]);

      constexpr const bytes_t& bytes() const;

      constexpr std::uint64_t subnet_prefix() const;
      constexpr std::uint64_t interface_identifier() const;

      static constexpr ipv6_address unspecified();
      static constexpr ipv6_address loopback();

      static std::optional<ipv6_address> from_string(std::string_view string) noexcept;

      std::string to_string() const;

      constexpr bool operator==(const ipv6_address& other) const;
      constexpr bool operator!=(const ipv6_address& other) const;
      constexpr bool operator<(const ipv6_address& other) const;
      constexpr bool operator>(const ipv6_address& other) const;
      constexpr bool operator<=(const ipv6_address& other) const;
      constexpr bool operator>=(const ipv6_address& other) const;

   };

   class ip_address
   {
   public:

      // 构造一个地址为 0.0.0.0 的 IPv4地址
      ip_address() noexcept;

      ip_address(ipv4_address address) noexcept;
      ip_address(ipv6_address address) noexcept;

      bool is_ipv4() const noexcept;
      bool is_ipv6() const noexcept;

      const ipv4_address& to_ipv4() const;
      const ipv6_address& to_ipv6() const;

      const std::uint8_t* bytes() const noexcept;

      std::string to_string() const;

      static std::optional<ip_address> from_string(std::string_view string) noexcept;

      bool operator==(const ip_address& rhs) const noexcept;
      bool operator!=(const ip_address& rhs) const noexcept;

      //  ipv4_address sorts less than ipv6_address
      bool operator<(const ip_address& rhs) const noexcept;
      bool operator>(const ip_address& rhs) const noexcept;
      bool operator<=(const ip_address& rhs) const noexcept;
      bool operator>=(const ip_address& rhs) const noexcept;

   };
   }

ip_endpoint, ipv4_endpoint, ipv6_endpoint
==========================================

表示IP地址和端口号的辅助类

API 摘要：

.. code-block:: cpp

   namespace cppcoro::net
   {
   class ipv4_endpoint
   {
   public:
      ipv4_endpoint() noexcept;
      explicit ipv4_endpoint(ipv4_address address, std::uint16_t port = 0) noexcept;

      const ipv4_address& address() const noexcept;
      std::uint16_t port() const noexcept;

      std::string to_string() const;
      static std::optional<ipv4_endpoint> from_string(std::string_view string) noexcept;
   };

   bool operator==(const ipv4_endpoint& a, const ipv4_endpoint& b);
   bool operator!=(const ipv4_endpoint& a, const ipv4_endpoint& b);
   bool operator<(const ipv4_endpoint& a, const ipv4_endpoint& b);
   bool operator>(const ipv4_endpoint& a, const ipv4_endpoint& b);
   bool operator<=(const ipv4_endpoint& a, const ipv4_endpoint& b);
   bool operator>=(const ipv4_endpoint& a, const ipv4_endpoint& b);

   class ipv6_endpoint
   {
   public:
      ipv6_endpoint() noexcept;
      explicit ipv6_endpoint(ipv6_address address, std::uint16_t port = 0) noexcept;

      const ipv6_address& address() const noexcept;
      std::uint16_t port() const noexcept;

      std::string to_string() const;
      static std::optional<ipv6_endpoint> from_string(std::string_view string) noexcept;
   };

   bool operator==(const ipv6_endpoint& a, const ipv6_endpoint& b);
   bool operator!=(const ipv6_endpoint& a, const ipv6_endpoint& b);
   bool operator<(const ipv6_endpoint& a, const ipv6_endpoint& b);
   bool operator>(const ipv6_endpoint& a, const ipv6_endpoint& b);
   bool operator<=(const ipv6_endpoint& a, const ipv6_endpoint& b);
   bool operator>=(const ipv6_endpoint& a, const ipv6_endpoint& b);

   class ip_endpoint
   {
   public:
      //构造一个地址为 0.0.0.0:0 的 IPv4 终端
      ip_endpoint() noexcept;

      ip_endpoint(ipv4_endpoint endpoint) noexcept;
      ip_endpoint(ipv6_endpoint endpoint) noexcept;

      bool is_ipv4() const noexcept;
      bool is_ipv6() const noexcept;

      const ipv4_endpoint& to_ipv4() const;
      const ipv6_endpoint& to_ipv6() const;

      ip_address address() const noexcept;
      std::uint16_t port() const noexcept;

      std::string to_string() const;

      static std::optional<ip_endpoint> from_string(std::string_view string) noexcept;

      bool operator==(const ip_endpoint& rhs) const noexcept;
      bool operator!=(const ip_endpoint& rhs) const noexcept;

      //  IPv4 终端排序时要小于 IPv6 终端
      bool operator<(const ip_endpoint& rhs) const noexcept;
      bool operator>(const ip_endpoint& rhs) const noexcept;
      bool operator<=(const ip_endpoint& rhs) const noexcept;
      bool operator>=(const ip_endpoint& rhs) const noexcept;
   };
   }

函数
****************************************

sync_wait()
========================================

``sync_wait()`` 能被用于同步 wait 直到指定的 ``awaitable`` 完成。

指定的 awaitable 将会在当前线程新建的一个协程内被 ``co_await``

``sync_wait()`` 的调用将会阻塞线程至操作完成。结果要么返回 ``co_await`` 的结果，要么重新抛出未捕获异常。

``sync_wait()`` 一般用于等待 ``main()`` 中顶层任务的完成，实际上它也是启动顶层 ``task`` 的唯一方法。

API 摘要：

.. code-block:: cpp

   // <cppcoro/sync_wait.hpp>
   namespace cppcoro
   {
   template<typename AWAITABLE>
   auto sync_wait(AWAITABLE&& awaitable)
      -> typename awaitable_traits<AWAITABLE&&>::await_result_t;
   }

例子：

.. code-block:: cpp

   void example_task()
   {
   auto makeTask = []() -> task<std::string>
   {
      co_return "foo";
   };

   auto task = makeTask();

   // 启动此“惰性任务”并等待它完成
   sync_wait(task); // -> "foo"
   sync_wait(makeTask()); // -> "foo"
   }

   void example_shared_task()
   {
   auto makeTask = []() -> shared_task<std::string>
   {
      co_return "foo";
   };

   auto task = makeTask();
   // 启动此共享任务并等待它完成
   sync_wait(task) == "foo";
   sync_wait(makeTask()) == "foo";
   }

when_all()
========================================

``when_all()`` 可以用于创建一个新的 Awaitable 对象，当其被 ``co_await`` 时，将会并发 ``co_await`` 所有输入的任务并返回一个他们结果的集合。

当返回的 Awaitable 对象被  ``co_await``  时，它将在当前线程上 ``co_await`` 每个输入的任务。一旦第一个任务完成，就会启动第二个任务，依此类推。这些操作并发地执行，直到它们全部运行完成。

一旦所有任务 ``co_await`` 操作都完成，就会从每个单独的结果构建一个结果集。任何输入任务抛出异常，或者结果集的构造抛出了异常，那么该异常将被返回的 Awaitable 对象的 ``co_await`` 表达式重新抛出。

若多个 ``co_await`` 由于异常而被终止，其中之一将从 ``co_await when_all()`` 传播出去，而其他异常将被忽略。具体是那个异常是在运行时决定。

如果要知道哪个任务 ``co_await`` 操作失败，或者即使其中一些操作失败也希望继续获取其他操作的结果，那么您应该使用 ``when_all_ready()`` 。

API 摘要：

.. code-block:: cpp

   // <cppcoro/when_all.hpp>
   namespace cppcoro
   {
   // 变参版本
   //
   // 注意：如果一些任务 `co_await` 表达式结果类型为 void，则构造
   // 的结果集中，其结果将使用一个空的结构体 detail::void_value 进行填充
   template<typename... AWAITABLES>
   auto when_all(AWAITABLES&&... awaitables)
      -> Awaitable<std::tuple<typename awaitable_traits<AWAITABLES>::await_result_t...>>;

   // 重载版本 vector<Awaitable<void>>.
   template<
      typename AWAITABLE,
      typename RESULT = typename awaitable_traits<AWAITABLE>::await_result_t,
      std::enable_if_t<std::is_void_v<RESULT>, int> = 0>
   auto when_all(std::vector<AWAITABLE> awaitables)
      -> Awaitable<void>;

   // 重载 vector<Awaitable<NonVoid>> ，在被等待时产生一个值
   template<
      typename AWAITABLE,
      typename RESULT = typename awaitable_traits<AWAITABLE>::await_result_t,
      std::enable_if_t<!std::is_void_v<RESULT>, int> = 0>
   auto when_all(std::vector<AWAITABLE> awaitables)
      -> Awaitable<std::vector<std::conditional_t<
            std::is_lvalue_reference_v<RESULT>,
            std::reference_wrapper<std::remove_reference_t<RESULT>>,
            std::remove_reference_t<RESULT>>>>;
   }

例子：

.. code-block:: cpp

   task<A> get_a();
   task<B> get_b();

   task<> example1()
   {
   // 并发运行 get_a() 和 get_b()
   // 产生的结果类型为 std::tuple<A, B>，其可使用结构化绑定进行解包。
   auto [a, b] = co_await when_all(get_a(), get_b());

   // 使用 a, b
   }

   task<std::string> get_record(int id);

   task<> example2()
   {
   std::vector<task<std::string>> tasks;
   for (int i = 0; i < 1000; ++i)
   {
      tasks.emplace_back(get_record(i));
   }

   // 并发运行所有的 get_record() 任务
   // 如果有任务发生了异常，那么在它们都完成时，将会将异常从
   //  co_await 表达式中传播出去
   std::vector<std::string> records = co_await when_all(std::move(tasks));

   // 处理结果集
   for (int i = 0; i < 1000; ++i)
   {
      std::cout << i << " = " << records[i] << std::endl;
   }
   }

when_all_ready()
========================================

``when_all_ready()`` 可以用于创建一个新的 Awaitable 对象，其将会在所有输入的 Awaitable 对象完成后才完成

输入任务可以是 Awaitable 的任何类型

当返回的 Awaitable 对象 [#]_ 被 ``co_await`` ，其将按照线程传入 ``when_all_ready()`` 的顺序依次 ``co_await`` 线程。如果这些任务不能同步完成，那么它们将并发执行。

一旦所有输入的任务都完成，则返回的 Awaitable 对象将会恢复挂起的协程。挂起的协程将在所有输入任务都完成后才被恢复。

返回的 Awaitable 保证不抛出异常，即使输入的任务可能抛出异常。

注意：调用 ``when_all_ready()`` 可能由于内存不足而抛出 ``std::bad_alloc`` 异常。而且还可能由于调用输入任务的拷贝/移动构造函数而抛出异常。

``co_await`` 返回的 Awaitable 对象的结果是返回一个 ``when_all_task<RESULT>`` 类型的 ``std::tuple`` 或 ``std::vector>`` 。 这些对象允许您通过调用 ``hen_all_task<RESULT>::result()`` 分别获得每个输入 Awaitable 对象的结果(或异常)。这允许调用方同时等待多个可等待对象，并在它们完成时进行同步，同时仍保留随后检查每个 ``co_await`` 操作的结果是否成功的能力。

这与 ``when_all()`` 任何单个 ``co_await`` 操作的失败都会导致整体操作失败不同。这意味着您无法确定哪个组件 ``co_await`` 操作失败，并且还使您无法获取其他 ``co_await`` 操作的结果。

API 摘要：

.. code-block:: cpp

   // <cppcoro/when_all_ready.hpp>
   namespace cppcoro
   {
   // 同时 await 多个 Awaitable 对象.
   //
   // 返回一个 Awaitable 对象，当其被 co_await 时，
   // 将会轮流等待所有输入的任务。当所有输入任务都完成后，
   // 唤醒挂起的协程。
   //
   // co_await 返回的 Awaitable 对象，其结果是一个类型为
   //  detail::when_all_task<T> 的 std::tuple。类型 T
   // 是各个输入任务被 co_await 结果的类型。
   //
   // 输入的任务必须为 Awaitable 类型。当输入为右值时必须可移动，当输入为左值是必须可拷贝。 co_await 表达式将会运行在拷贝的右值上
   template<typename... AWAITABLES>
   auto when_all_ready(AWAITABLES&&... awaitables)
      -> Awaitable<std::tuple<detail::when_all_task<typename awaitable_traits<AWAITABLES>::await_result_t>...>>;

   // 并发等待输入任务队列中的任务
   template<
      typename AWAITABLE,
      typename RESULT = typename awaitable_traits<AWAITABLE>::await_result_t>
   auto when_all_ready(std::vector<AWAITABLE> awaitables)
      -> Awaitable<std::vector<detail::when_all_task<RESULT>>>;
   }

例子：

.. code-block:: cpp

   task<std::string> get_record(int id);

   task<> example1()
   {
   // 并发运行 3 个 get_record() 并等待它们完成
   // 返回一个 std::tuple 类型，其可使用结构化绑定表达式进行解包。
   auto [task1, task2, task3] = co_await when_all_ready(
      get_record(123),
      get_record(456),
      get_record(789));

   // 对每个任务进行解包
   std::string& record1 = task1.result();
   std::string& record2 = task2.result();
   std::string& record3 = task3.result();

   // 使用 records....
   }

   task<> example2()
   {
   // 创建输入的任务，但是还不开始执行
   std::vector<task<std::string>> tasks;
   for (int i = 0; i < 1000; ++i)
   {
      tasks.emplace_back(get_record(i));
   }

   // 同时运行所有的任务
   std::vector<detail::when_all_task<std::string>> resultTasks =
      co_await when_all_ready(std::move(tasks));

   // 一旦任务都完成，对其结果进行解包
   for (int i = 0; i < 1000; ++i)
   {
      try
      {
         std::string& record = tasks[i].result();
         std::cout << i << " = " << record << std::endl;
      }
      catch (const std::exception& ex)
      {
         std::cout << i << " : " << ex.what() << std::endl;
      }
   }
   }

.. [#] 译者注：这里“返回的 Awaitable 对象”指的应当是 ``when_all_ready()`` 返回的 Awaitable 对象。

fmap()
========================================

``fmap()`` 用于将指定函数应用于容器内的元素，返回结果是一个新的、包含应用结果容器。

``fmap()`` 函数可以将一个函数应用于 ``generator<T>`` 、 ``recursive_generator<T>`` 和 `` async_generator<T>`` 的值以及任何支持 ``Awaitable`` 概念的值(例如: ``task<T>`` )。

每一种类型都为 ``fmap()`` 提供了带两个参数的重载：要应用的函数和被应用的容器。有关支持的 ``fmap()`` 重载，请参见文档。

例如， ``fmap()`` 函数可用于将函数应用于 ``task<T>`` 的结果，生成一个新的 ``task<U>`` ，该任务将使用函数的返回值来完成。

.. code-block:: cpp

   // 使用一个函数来将类型 A 转换到类型 B
   B a_to_b(A value);

   // 一个生成类型为 A 的task
   cppcoro::task<A> get_an_a();

   // 我们可以使用 fmap() 函数将一个函数应用到 task 上，并获得新 task 的结果
   cppcoro::task<B> bTask = fmap(a_to_b, get_an_a());

   // 另一种语法是使用管道表示法
   cppcoro::task<B> bTask = get_an_a() | cppcoro::fmap(a_to_b);

API 摘要：

.. code-block:: cpp

   // <cppcoro/fmap.hpp>
   namespace cppcoro
   {
   template<typename FUNC>
   struct fmap_transform
   {
      fmap_transform(FUNC&& func) noexcept(std::is_nothrow_move_constructible_v<FUNC>);
      FUNC func;
   };

   // 类型推导的构造函数，以便于使用 operator| 操作
   template<typename FUNC>
   fmap_transform<FUNC> fmap(FUNC&& func);

   // operator| 重载以便于为 fmap() 提供类似管道的语法糖
   // 比如这种表达式：
   //   <value-expr> | cppcoro::fmap(<func-expr>)
   // 等价于：
   //   fmap(<func-expr>, <value-expr>)

   template<typename T, typename FUNC>
   decltype(auto) operator|(T&& value, fmap_transform<FUNC>&& transform);

   template<typename T, typename FUNC>
   decltype(auto) operator|(T&& value, fmap_transform<FUNC>& transform);

   template<typename T, typename FUNC>
   decltype(auto) operator|(T&& value, const fmap_transform<FUNC>& transform);

   // 所有 Awaitable 类型的通用重载
   //
   // 在被 co_await 时返回一个 Awaitable 对象。co_await 返回的对象并在其上应用指定的函数
   // 类似于 'std::invoke(func, co_await awaitable)'
   //
   // 若 'co_await awaitable' 表达式的类型为 'void' 则 co_await 
   // 返回的 Awaitable 等价于 'co_await awaitable, func()'
   template<
      typename FUNC,
      typename AWAITABLE,
      std::enable_if_t<is_awaitable_v<AWAITABLE>, int> = 0>
   auto fmap(FUNC&& func, AWAITABLE&& awaitable)
      -> Awaitable<std::invoke_result_t<FUNC, typename awaitable_traits<AWAITABLE>::await_result_t>>;
   }

``fmap()`` 函数被设计成通过 :abbr:`依赖于参数的查找 (Argument-Dependent Lookup, ADL)` 来查找正确的重载，因此通常调用它时不应该使用 ``cppcoro::`` 前缀。

schedule_on()
========================================

``schedule_on()`` 函数可用于更改给定的 Awaitable 或开始执行的异步生成器的执行上下文

当应用到异步生成器时，它还会影响在 ``co_yield`` 语句之后它将在哪个执行上下文上继续执行

请注意， ``schedule_on`` 上的调度并不指定 Awaitable 或异步生成器完成或产生结果的线程，这取决于 Awaitable 或生成器的实现。

请参阅 ``resume_on()`` 如何控制任务在哪个线程上完成。

例子：

.. code-block:: cpp

   task<int> get_value();
   io_service ioSvc;

   task<> example()
   {
   // 在当前线程上开始执行 get_value()
   int a = co_await get_value();

   // 在 ioSvc 的线程上执行 get_value()
   int b = co_await schedule_on(ioSvc, get_value());
   }

API 摘要：

.. code-block:: cpp

   // <cppcoro/schedule_on.hpp>
   namespace cppcoro
   {
   // 返回一个 task ，其结果与 't' 相同，但是确保 't' 在被 co_await 时
   // 在与调度器相关联的线程上执行。task 的结果将在 't' 完成的线程上完成。
   template<typename SCHEDULER, typename AWAITABLE>
   auto schedule_on(SCHEDULER& scheduler, AWAITABLE awaitable)
      -> Awaitable<typename awaitable_traits<AWAITABLE>::await_result_t>;

   // 返回一个生成器，其序列与 'source' 相同，但是确保启动的协程在与调度器
   // 相关联的线程上执行。在被 'co_yield' 后，在与调度器关联的线程上恢复
   template<typename SCHEDULER, typename T>
   async_generator<T> schedule_on(SCHEDULER& scheduler, async_generator<T> source);

   template<typename SCHEDULER>
   struct schedule_on_transform
   {
      explicit schedule_on_transform(SCHEDULER& scheduler) noexcept;
      SCHEDULER& scheduler;
   };

   template<typename SCHEDULER>
   schedule_on_transform<SCHEDULER> schedule_on(SCHEDULER& scheduler) noexcept;

   template<typename T, typename SCHEDULER>
   decltype(auto) operator|(T&& value, schedule_on_transform<SCHEDULER> transform);
   }

resume_on()
========================================

``resume_on()`` 函数可用于控制恢复挂起协程的执行上下文。当应用到异步生成器时，它控制 ``co_await g.begin()`` 和 ``co_await ++it`` 操作在哪个执行上下文上恢复挂起的协程。

通常，等待 Awaitable 的协程(比如：一个 task )或异步生成器将在该操作完成的任何线程上恢复执行。在某些情况下，这可能不是您想要继续执行的线程。在这些情况下，您可以使用 ``resume_on()`` 函数创建一个新的 Awaitable 或生成器，它将在与指定调度程序关联的线程上恢复执行。

``resume_on()`` 函数可以用作返回新的 Awaitable/Generator 的普通函数也可以在管道语法中使用它。

例如：

.. code-block:: cpp

   task<record> load_record(int id);

   ui_thread_scheduler uiThreadScheduler;

   task<> example()
   {
   // 这将在当前线程上启动 load_record()
   // 然后当 load_record() 完成时（可能是一个 I/O 线程）
   // 它将被重新调度到线程池并调用 to_json
   // 一旦 to_json 完成，它将在被恢复前被调度到 ui 线程并返回 json 字符串
   task<std::string> jsonTask =
      load_record(123)
      | cppcoro::resume_on(threadpool::default())
      | cppcoro::fmap(to_json)
      | cppcoro::resume_on(uiThreadScheduler);

   // 此处，我们所做的就是创建一个 task 的流水线
   // 任务不会立即开启

   // Await 结果。开始 task 的流水线
   std::string jsonText = co_await jsonTask;

   // 保证在 ui 线程上执行

   someUiControl.set_text(jsonText);
   }

API 摘要：

.. code-block:: cpp

   // <cppcoro/resume_on.hpp>
   namespace cppcoro
   {
   template<typename SCHEDULER, typename AWAITABLE>
   auto resume_on(SCHEDULER& scheduler, AWAITABLE awaitable)
      -> Awaitable<typename awaitable_traits<AWAITABLE>::await_traits_t>;

   template<typename SCHEDULER, typename T>
   async_generator<T> resume_on(SCHEDULER& scheduler, async_generator<T> source);

   template<typename SCHEDULER>
   struct resume_on_transform
   {
      explicit resume_on_transform(SCHEDULER& scheduler) noexcept;
      SCHEDULER& scheduler;
   };

   // 构建一个转发对象以便于能对源对象使用管道符（比如： operator| ）
   template<typename SCHEDULER>
   resume_on_transform<SCHEDULER> resume_on(SCHEDULER& scheduler) noexcept;

   // 等价于 'resume_on(transform.scheduler, std::forward<T>(value))'
   template<typename T, typename SCHEDULER>
   decltype(auto) operator|(T&& value, resume_on_transform<SCHEDULER> transform)
   {
      return resume_on(transform.scheduler, std::forward<T>(value));
   }
   }

元函数
****************************************

awaitable_traits<T>
========================================

此元函数用于当 ``co_await`` 的类型为 T 时，其结果的类型。

注意：这里假设被 await 的类型 ``T`` 的值在其上下文中没有被任何协程 Promise 对象的 ``await_transform`` 所影响。否则，类型 ``T`` 的结果类型可能不同。

若类型 ``T`` 是不可 await 的，则 awaitable_traits<T> 原函数不会定义任何嵌套的 ``awaiter_t`` 或 ``await_result_t`` 类型。当类型 ``T`` 不可 await 时，这允许它在禁用重载的 :abbr:`SFINAE (Substitution Failure Is Not An Error)` 上下文中被使用。

API 摘要：

.. code-block:: cpp

   // <cppcoro/awaitable_traits.hpp>
   namespace cppcoro
   {
   template<typename T>
   struct awaitable_traits
   {
      // 若类型 T 支持 `operator co_await()` 则为 `operator co_await()` 类型 T 的结果，否则为 `T&&`
      typename awaiter_t = <unspecified>;

      // co_await 类型 T 的结果的值
      typename await_result_t = <unspecified>;
   };
   }

is_awaitable<T>
========================================

``is_awaitable<T>`` 元函数能用来查询协程中一个指定的类型能否被 ``co_await`` 。

API 摘要：

.. code-block:: cpp

   // <cppcoro/is_awaitable.hpp>
   namespace cppcoro
   {
   template<typename T>
   struct is_awaitable : std::bool_constant<...>
   {};

   template<typename T>
   constexpr bool is_awaitable_v = is_awaitable<T>::value;
   }

概念
****************************************

Awaitable<T>
========================================

:abbr:`概念 (Concepts)` ``Awaitable<T>`` 表明了一个在协程上下文中可被 ``co_await`` 的对象，并且没有 ``await_transform`` 的重载。其对应的 ``co_await`` 表达式类型为 ``T`` 。

比如， ``task<T>`` 实现了概念 ``Awaitable<T&&>`` ，而类型 ``task<T>&`` 实现了概念 ``Awaitable<T&>``

Awaiter<T>
========================================

概念 ``Awaiter<T>`` 表明了一个类型，其实现了把被 await 的协程暂停/恢复的协议。其必须拥有的 ``await_ready`` 、 ``await_suspend`` 和 ``await_resume`` 方法。

假设有一个类型 ``awaiter`` ，则要满足 ``Awaiter<T>`` 的需求，其需要：

- ``awaiter.await_ready()`` -> ``bool``
- ``awaiter.await_suspend(std::experimental::coroutine_handle<void>{}) -> void`` 或 ``bool`` 或 ``std::experimental::coroutine_handle<P>`` 对于一些 ``P``.
- ``awaiter.await_resume() -> T``

任何实现了概念 ``Awaiter<T>`` 的类型也同时实现了概念 ``Awaitable<T>``

Scheduler
========================================

概念 ``Scheduler`` 改变了允许在一些执行上下文中调度协程的运行。

.. code-block:: cpp

   concept Scheduler
   {
   Awaitable<void> schedule();
   }

假设有类型 ``S`` 实现了 ``Scheduler`` 概念，且有实例 ``s`` 。则：

- ``s.schedule()`` 方法返回一个 Awaitable 类型。因此 ``co_await s.schedule()`` 将会无条件暂停当前协程并调度其在与 ``s`` 相关的协程内恢复。
- ``co_await s.schedule()`` 表达式的类型为 ``void``

.. code-block:: cpp

   cppcoro::task<> f(Scheduler& scheduler)
   {
   // 协程的执行最初在调用者的执行上下文中执行

   // 暂停当前协程并调度其在 scheduler 的执行上下文中恢复
   co_await scheduler.schedule();

   // 现在协程运行在 scheduler 的执行上下文中
   }

DelayedScheduler
========================================

概念 ``DelayedScheduler`` 允许协程自己调度自己在延迟一段时间后到调度器的执行上下文中。

.. code-block:: cpp

   concept DelayedScheduler : Scheduler
   {
   template<typename REP, typename RATIO>
   Awaitable<void> schedule_after(std::chrono::duration<REP, RATIO> delay);

   template<typename REP, typename RATIO>
   Awaitable<void> schedule_after(
      std::chrono::duration<REP, RATIO> delay,
      cppcoro::cancellation_token cancellationToken);
   }

假设有类型 ``S`` 实现了 ``DelayedScheduler`` 概念，且有实例 ``s`` 。则：

- ``s.schedule_after(delay)`` 方法返回一个 Awaitable 类型。因此 ``co_await s.schedule_after(delay)`` 将会无条件暂停当前协程一段时间然后调度其在与 ``s`` 相关的协程内恢复。
- ``co_await s.schedule_after(delay)`` 表达式的类型为 ``void``

构建
****************************************

cppcoro 库的 Windows 构建要求至少  Visual Studio 2017，而 Linux 构建至少要求 Clang 5.0

此库利用了 `Cake 构建系统 <https://github.com/lewissbaker/cake>`_ （ 不是用于 `C# <http://cakebuild.net/>`_ 的这个）

Cake 构建系统会作为 git 子模块自动检出，所以你无需手动下载或安装它。

.. note::  

   译者注：

   根据 vcpkg 中的 cppcoro 打包方式。 cppcoro 只需要仓库下的 ``include/cppcoro`` 文件夹，你可以将其拷贝到任意一个文件夹，然后在 CMake 或 其他构建工具中将其作为头文件目录包含即可。

   由于PR ``https://github.com/lewissbaker/cppcoro/pull/171`` 还未被合并，在最新的编译器中你可能需要更改 cppcoro 中的代码以通过编译：将所有的 ``std::experimental`` 替换为 ``std`` ，将所有的 ``experimental/`` 删除。

   参见 issue：https://github.com/lewissbaker/cppcoro/issues/191

   和   PR   ：https://github.com/msys2/MINGW-packages/pull/7834

在 Windows 上构建
========================================

这个库要求至少 Visual Studio 2017 ，还需要 Windows 10 SDK

对 Clang （ `#3 <Visual Studio 2017>`_ ）和 Linux 的支持正处于计划中。

Windows 环境需求
----------------------------------------

Cake 是由 Python 2.7 实现，因此要求安装 Python 2.7。

确保 Python2.7 解释器在你的 PATH 变量中，并且名为 python 。

确保 Visual Studio 2017 Update 3 已经安装。注意一些问题会出现在 Update 2 及以前的版本中，这些问题在 Update 3 才修复。

你也可以从 https://vcppdogfooding.azurewebsites.net/ 下载并解压一个 NuGet 包以使用预览版本的 Visual Studio。解压  .nuget 到一个目录，并修改 ``config.cake`` 文件以指向此目录：

.. code-block:: ini

   nugetPath = None # r'C:\Path\To\VisualCppTools.14.0.25224-Pre'

确保 Windows 10 SDK 已经安装。默认情况下它会使用 Windows 10 SDK latest 和 Universal C Runtime。

Clone 仓库
----------------------------------------

cppcoro 使用 git 子模块来拉取 Cake 构建系统的源码。

这意味着你在使用 ``git clone`` 时需要加上 ``--recursive`` 参数。

.. code-block:: none

   c:\Code> git clone --recursive https://github.com/lewissbaker/cppcoro.git

如果你已经克隆了 cppcoro，你需要更新子模块：

.. code-block:: none

   c:\Code\cppcoro> git submodule update --init --recursive

从命令行构构建
----------------------------------------
   
要从命令行构建只需要执行 'cake.bat' 文件：

.. code-block:: none

   C:\cppcoro> cake.bat
   Building with C:\cppcoro\config.cake - Variant(release='debug', platform='windows', architecture='x86', compilerFamily='msvc', compiler='msvc14.10')
   Building with C:\cppcoro\config.cake - Variant(release='optimised', platform='windows', architecture='x64', compilerFamily='msvc', compiler='msvc14.10')
   Building with C:\cppcoro\config.cake - Variant(release='debug', platform='windows', architecture='x64', compilerFamily='msvc', compiler='msvc14.10')
   Building with C:\cppcoro\config.cake - Variant(release='optimised', platform='windows', architecture='x86', compilerFamily='msvc', compiler='msvc14.10')
   Compiling test\main.cpp
   Compiling test\main.cpp
   Compiling test\main.cpp
   Compiling test\main.cpp
   ...
   Linking build\windows_x86_msvc14.10_debug\test\run.exe
   Linking build\windows_x64_msvc14.10_optimised\test\run.exe
   Linking build\windows_x86_msvc14.10_optimised\test\run.exe
   Linking build\windows_x64_msvc14.10_debug\test\run.exe
   Generating code
   Finished generating code
   Generating code
   Finished generating code
   Build succeeded.
   Build took 0:00:02.419.

默认情况下，执行 ``cake`` 而不传入任何参数，将会构建项目的所有版本并进行单元测试。通过传入参数你可以缩减其构建的范围:

.. code-block:: none

   c:\cppcoro> cake.bat release=debug architecture=x64 lib/build.cake
   Building with C:\Users\Lewis\Code\cppcoro\config.cake - Variant(release='debug', platform='windows', architecture='x64', compilerFamily='msvc', compiler='msvc14.10')
   Archiving build\windows_x64_msvc14.10_debug\lib\cppcoro.lib
   Build succeeded.
   Build took 0:00:00.321.

你可以运行 ``cake --help`` 去列出所有可用的命令行选项。

使用 Visual Studio 项目文件构建
----------------------------------------

如果希望在 Visual Studio 中构建，你可以运行 ``cake.bat -p`` 以生成 .vcproj/.sln 文件。

例如：

.. code-block:: none

   c:\cppcoro> cake.bat -p
   Building with C:\cppcoro\config.cake - Variant(release='debug', platform='windows', architecture='x86', compilerFamily='msvc', compiler='msvc14.10')
   Building with C:\cppcoro\config.cake - Variant(release='optimised', platform='windows', architecture='x64', compilerFamily='msvc', compiler='msvc14.10')
   Building with C:\cppcoro\config.cake - Variant(release='debug', platform='windows', architecture='x64', compilerFamily='msvc', compiler='msvc14.10')
   Building with C:\cppcoro\config.cake - Variant(release='optimised', platform='windows', architecture='x86', compilerFamily='msvc', compiler='msvc14.10')
   Generating Solution build/project/cppcoro.sln
   Generating Project build/project/cppcoro_tests.vcxproj
   Generating Filters build/project/cppcoro_tests.vcxproj.filters
   Generating Project build/project/cppcoro.vcxproj
   Generating Filters build/project/cppcoro.vcxproj.filters
   Build succeeded.
   Build took 0:00:00.247.

当您从Visual Studio中构建这些项目时，它将调用 cake 来执行编译。

在 Linux 上构建
========================================

cppcoro 也可以在拥有 Clang 和至少 libc++ 5.0 的 Linux 下构建。

构建 cppcoro 在 Ubuntu 17.04 上通过测试。

Linux 环境需求
----------------------------------------

确保以下包你已经安装：

- Python 2.7
- Clang >= 5.0
- LLD >= 5.0
- libc++ >= 5.0

构建 cppcoro
----------------------------------------

此处猜测 Clang 和 libc++ 你已经安装了。

如果您还没有配置 Clang ，请参阅以下部分，了解有关使用 Clang 构建 cppcoro 的配置细节。

检出 cppcoro 极其子模块：

.. code-block:: shell

   git clone --recursive https://github.com/lewissbaker/cppcoro.git cppcoro

运行 ``init.sh`` 以设置 ``cake`` 函数

.. code-block:: shell

   cd cppcoro
   source init.sh

然后，您可以从工作区根目录运行 cake 来构建 cppcoro 并运行测试：

.. code-block:: none

   $ cake

您可以指定额外的命令行参数来定制构建：

- ``--help`` 将打印出有关命令行参数的帮助

- ``--debug=run`` 显示将要运行的构建命令行

- ``release=debug`` 或 ``release=optimised`` 会将构建变体限制为 debug 或 optimised （默认情况下，将同时构建两者）。

- ``lib/build.cake`` 只会构建 cppcoro 库而没有单元测试。

- ``test/build.cake@task_tests.cpp`` 只会编译特定的源文件

- ``test/build.cake@testresult`` 将构建并运行单元测试

例如：

.. code-block:: none

   $ cake --debug=run release=debug lib/build.cake

自定义 Clang 位置
----------------------------------------

如果你的 clang 编译器不是在 ``/usr/bin/clang`` ，则你可以 ``cake`` 的命令行选项来指出 clang 的位置：

- ``--clang-executable=<名称>`` 指定要使用的 clang 可执行文件名称，而不是clang。 例如：传入 ``--clang-executable=clang-8`` 以强制使用 Clang 8.0

- ``--clang-executable=<abspath>`` 指定 clang 可执行文件的完整路径。 构建系统还将在同一目录中查找其他可执行文件。 如果此路径的格式为 ``<prefix>/ bin/<name>`` ，则默认的 ``clang-install-prefix`` 将被设置为 ``<prefix>`` 。

- ``--clang-install-prefix=<路径>`` 指定安装 clang 的路径。这将导致构建系统在 ``<path>/bin`` 下查找 clang（除非被 ``--clang-executable`` 覆盖）。

- ``--libcxx-install-prefix=<路径>`` 指定已安装的 libc++ 的路径。 默认情况下，构建系统将在与 clang 相同的位置寻找 libc++ 。 如果将其安装在其他位置，请使用此命令行选项。

例如：使用安装在默认位置的特定版本的 clang

.. code-block:: none

   $ cake --clang-executable=clang-8

例如：使用自定义位置下默认版本的 clang

.. code-block:: none

   $ cake --clang-install-prefix=/path/to/clang-install

例如：使用自定义位置的 clang，自定义版本的 clang，自定义位置的 libc++ ：

.. code-block:: none

   $ cake --clang-executable=/path/to/clang-install/bin/clang-8 --libcxx-install-prefix=/path/to/libcxx-install

使用 Clang 的快照构建
----------------------------------------

如果您的 Linux 发行版没有 Clang 5.0 或更高版本，您可以从 LLVM 项目中安装一个快照构建。

按照 http://apt.llvm.org/ 上的说明设置软件包管理器，以支持从 LLVM 源中获取软件。

比如：对于 Ubuntu 17.04 Zesty：

编辑 ``/etc/apt/sources.list`` 并添加一下行：

.. code-block:: none

   deb http://apt.llvm.org/zesty/ llvm-toolchain-zesty main
   deb-src http://apt.llvm.org/zesty/ llvm-toolchain-zesty main

安装 PGP 密钥：

.. code-block:: none

   $ wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key | sudo apt-key add -

安装 Clang 和 LLD ：

.. code-block:: none

   $ sudo apt-get install clang-6.0 lld-6.0

构建你的 Clang
----------------------------------------

你也可以使用最前沿的 Clang，自己从源代码构建 Clang 。

指令如下：

为此，您需要安装以下软件：

.. code-block:: none

   $ sudo apt-get install git cmake ninja-build clang lld

请注意，我们正在使用您发行版的 clang 版本从源代码构建 clang 。这里也可以使用 GCC 。

检出 LLVM + Clang + LLD + libc++ 仓库：

.. code-block:: shell

   mkdir llvm
   cd llvm
   git clone --depth=1 https://github.com/llvm-mirror/llvm.git llvm
   git clone --depth=1 https://github.com/llvm-mirror/clang.git llvm/tools/clang
   git clone --depth=1 https://github.com/llvm-mirror/lld.git llvm/tools/lld
   git clone --depth=1 https://github.com/llvm-mirror/libcxx.git llvm/projects/libcxx
   ln -s llvm/tools/clang clang
   ln -s llvm/tools/lld lld
   ln -s llvm/projects/libcxx libcxx

配置并构建 Clang :

.. code-block:: shell

   mkdir clang-build
   cd clang-build
   cmake -GNinja \
         -DCMAKE_CXX_COMPILER=/usr/bin/clang++ \
         -DCMAKE_C_COMPILER=/usr/bin/clang \
         -DCMAKE_BUILD_TYPE=MinSizeRel \
         -DCMAKE_INSTALL_PREFIX="/path/to/clang/install"
         -DCMAKE_BUILD_WITH_INSTALL_RPATH="yes" \
         -DLLVM_TARGETS_TO_BUILD=X86 \
         -DLLVM_ENABLE_PROJECTS="lld;clang" \
         ../llvm
   ninja install-clang \
         install-clang-headers \
         install-llvm-ar \
         install-lld

构建 libc++
----------------------------------------

由于使用了 ``<experimental/coroutine>`` ，cppcoro 需要 libc++ 以使用 Clang 中的 C++ 协程。

检出 ``libc++`` + ``llvm`` ：

.. code-block:: shell

   mkdir llvm
   cd llvm
   git clone --depth=1 https://github.com/llvm-mirror/llvm.git llvm
   git clone --depth=1 https://github.com/llvm-mirror/libcxx.git llvm/projects/libcxx
   ln -s llvm/projects/libcxx libcxx

构建 ``libc++``

.. code-block:: shell

   mkdir libcxx-build
   cd libcxx-build
   cmake -GNinja \
         -DCMAKE_CXX_COMPILER="/path/to/clang/install/bin/clang++" \
         -DCMAKE_C_COMPILER="/path/to/clang/install/bin/clang" \
         -DCMAKE_BUILD_TYPE=Release \
         -DCMAKE_INSTALL_PREFIX="/path/to/clang/install" \
         -DLLVM_PATH="../llvm" \
         -DLIBCXX_CXX_ABI=libstdc++ \
         -DLIBCXX_CXX_ABI_INCLUDE_PATHS="/usr/include/c++/6.3.0/;/usr/include/x86_64-linux-gnu/c++/6.3.0/" \
         ../libcxx
   ninja cxx
   ninja install

这将构建并将 libc++ 安装到与 clang 相同的目录中。

支持
****************************************

GitHub 的 issues 是支持、bug 报告和新特性请求的主要方式。

在同意 MIT 许可的情况下，欢迎任何贡献和 Pull-Requests 。

如果你遇到 C++ 协程的问题，一般你可以在 ``Cpplang Slack <https://cpplang.slack.com/>`_ 组的 ``#coroutines`` 频道中获得帮助。

索引和表
****************************************

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`
