+++
title = 'cleanup：在C语言中实现基于作用域的自动资源管理'
date = 2024-08-11T23:08:11+08:00
+++

## 0 摘要

本文分析了C语言中由GCC/Clang编译器提供的*cleanup*属性的用法、性能和特点，并介绍了Linux v6.6的cleanup.h中**基于**_**cleanup**_**属性实现的轻量级的基于作用域的自动资源管理机制**相关代码，并结合内核实际代码分析了其用法和好处，并将这个机制与内核的新晋编程语言——**Rust的生命周期机制进行了对比**，分析C语言的cleanup属性的局限之处。

## 1 引子

由于C语言不支持自动内存管理，内存泄漏、锁未释放等安全问题在C程序中相对常见。作为一个新人，编程经验难免不够老到，不出意外还是出意外了。在一次 Bug修 (tí) 复 (jiāo) 后，被Code Review发现我忘记释放自旋锁而导致线程死锁。

我不禁开始思考：C语言应该怎么避免此类问题？即：**C语言如何像C++/Rust等无GC的高级语言那样，在作用域结束时自动释放一些资源？**这让我想起了 GCC 提供的一个编译器扩展：_**cleanup**_ **属性**。

恰好，搜索发现，一年多前，Linux kernel v6.6 基于*cleanup*属性引入了作用域的资源管理。现将这个特性介绍给大家。

## 2 cleanup属性详解

### 如何使用cleanup

GCC 和 Clang 都支持cleanup属性。其具体用法如下：

```c
Type var __attribute__((__cleanup__(type_destructor_fn)));
```

*cleanup*属性用于变量声明，并且还需要一个函数作为该属性的参数。编译器会在这个变量的作用域结束时，按照如下形式自动调用给定的函数：

```c
type_destructor_fn(&var);
```

比如，malloc的动态内存，就可以通过*cleanup*属性在**作用域结束**的时候自动释放。作用域的具体概念和定义这里就不过多介绍了，可以参考[GCC的这篇文档](https://www.gnu.org/software/c-intro-and-ref/manual/html_node/Scope.html)。

```c
void heap_auto_drop(void *p) {
  void *ptr = *(void **)p;
  if (p != NULL)
    free(p);
}
int foo() {
  char *ptr __attribute__((__cleanup__(heap_auto_drop))) = NULL;
  ptr = malloc(10);
  if (ptr == NULL)
    return -1;
  // do somethin here
  return 0; // 无需手动释放 ptr 指向的内存
}
```

上面这段代码，`foo()` 函数的指针ptr指向的内存将在`foo()`函数执行完成后，**由编译器自动完成指针ptr指向的内存释放**，无需担心内存泄漏。

可以看出，编译器提供的*cleanup*属性C++或Rust的析构函数（delete, drop）非常相似。*cleanup*属性给在C语言上实现简单的_**RAII**_（Resource Acquisition Is Initialization）提供了手段。需要注意的是，**编译器会忽略**_**cleanup**_**函数的返回值**。不过通常情况下，unlock close free等析构函数的返回值都会被编程者忽略。

### cleanup的性能

C语言常用在对性能要求较高的场景，因此，还需要评估cleanup属性对程序的性能的影响。那么既然*cleanup*的相关代码由编译器插入生成，不如直接比较是否使用*cleanup*实现相同功能的函数生成汇编的差异。

这里编写了一个[简单的demo](https://godbolt.org/z/oT7n73bEK) 展示了基于作用域的自动资源管理和手动资源管理生成的汇编代码的差异。欢迎大家打开这个链接亲自尝试观察产生汇编的差异。

![godbolt.png](/image/cleanup/godbolt.png)

可以看出，经过编译器的O2优化后，**编译器通过**_**cleanup**_**释放内存的代码与手工释放内存编写的代码，产生了完全一致的汇编**。由此可见，使用*cleanup*属性几乎不会影响C语言程序的性能。

### cleanup的特点

声明*cleanup*属性后，编译器会在适当的地方自动插桩相应代码。因此，我们还需要了解编译器对*cleanup*属性的约束：

*   _**cleanup**_**属性是可信的**：*cleanup*属性将在**变量作用域结束时执行**。编译器是可信的，那么由编译器自动插桩*cleanup*函数也是可信的。
    
*   _**cleanup**_**属性只能用于定义的局部变量**，不能用于函数参数或者全局/静态变量。*cleanup*的函数接受一个指针类型的参数，该参数需与被定义*cleanup*属性的变量的指针类型匹配兼容（如void \*相关）。
    
*   _**cleanup**_**函数的返回值无意义**，编译器会忽略该函数的返回值，因此*cleanup*函数应定义为void类型。
    
*   同一个作用域如果定义了多个*cleanup*属性的变量，那么**执行**_**cleanup**_**的顺序与定义变量的顺序相反**，即：后定义的变量，先*cleanup* (last defined, first *cleanup*)
    

## 3 Linux kernel v6.6：基于作用域的资源管理

linux 内核v6.6中，Peter Zijlstra的[patch set](https://lore.kernel.org/all/20230612090713.652690195@infradead.org/) 将基于*cleanup*属性的一系列宏定义带入了内核，为Linux内核引入了基于作用域的资源管理（Scope-based Resource Management）机制。

### cleanup.h：一组可移植性良好的宏定义

Linux的基于作用域的资源管理机制都在头文件include/linux/cleanup.h中定义，其中包含了一系列宏定义用来帮助开发者定义指针的释放函数（*cleanup* function），并以此为基础扩展出C语言“对象”的构造函数和析构函数，以及基于作用域的guard相关helper宏定义等。

"Talk is cheap, show me the code"，咱还是直接来看代码吧。

#### free: 指针的释放

```toml
// 定义cleanup函数
#define DEFINE_FREE(_name, _type, _free)       \
	static inline void __free_##_name(void *p) \
	{                                          \
		_type _T = *(_type *)p;                \
		_free;                                 \
	}
// 声明自动cleanup的指针
#define __free(_name)	__cleanup(__free_##_name)
// 将指针转换为不自动cleanup
#define no_free_ptr(p) \
	((typeof(p)) __must_check_fn(__get_and_null(p, NULL)))
```

上述宏定义用来创建和管理作用域结束自动释放的指针，分别对应了*cleanup*函数的定义、自动*cleanup*指针的声明和禁止指针自动释放。

`DEFINE_FREE()`宏定义是用来声明一个*cleanup*函数，其**第一个参数name必须是唯一的**，因为该参数在使用`__free()`声明自动*cleanup*的指针时直接决定了这个指针的释放方法。

当我们需要把这个自动清理的指针赋值给某个结构体或者作为函数参数返回时，**指针的生命周期会超过其声明的作用域**，因此需要使用`no_free_ptr()`宏来**禁止编译器的自动**_**cleanup**_。这个宏会把原先的地址拷贝到另一个变量，然后将该自动释放指针置为NULL，返回拷贝的变量，这样就能禁止编译器的自动释放了。

为了使编译器更好的编译优化*cleanup*函数，**使用**`**DEFINE_FREE()**`**定义释放函数时，一定要判断指针是否为NULL。**

具体到内核代码中，给出了如下的例子。首先使用`DEFINE_FREE()`宏来定义kmalloc的释放函数，命名为kfree。在函数中，使用`__free(kfree)`声明了指针。如果内存申请成功且obj初始化成功，则调用`no_free_ptr()`宏来禁止编译器的自动释放。否则直接返回空指针，由编译器视情况释放通过kmalloc申请的内存。

需要注意的是，代码中的`_T`是cleanup.h这套macro定义的专用变量，类似于this/self指针。

```c
DEFINE_FREE(kfree, void *, if (!IS_ERR_OR_NULL(_T)) kfree(_T)) // 定义用于自动释放kmalloc的内存的cleanup函数，命名为kfree，(定义在 include/linux/slab.h)
void *alloc_obj(...) // 申请并初始化一个对象的函数
{
    struct obj *p __free(kfree) = kmalloc(...); // 定义一个指针，该指针通过名为kfree的cleanup函数自动释放
    if (!p)
        return NULL; // 命名为kfree的自动cleanup函数不会释放空指针，这里是安全的
    if (!init_obj(p))
       return NULL; // 如果初始化失败，则直接返回，由编译器释放kmalloc申请的内存
    return no_free_ptr(p); // 禁止编译器自动释放，防止use-after-free的bug
}
```

#### class: C语言的“类”

```c
#define DEFINE_CLASS(_name, _type, _exit, _init, _init_args...) \
	typedef _type class_##_name##_t;                            \
	static inline void class_##_name##_destructor(_type *p)     \
	{                                                           \
		_type _T = *p;                                          \
		_exit;                                                  \
	}                                                           \
	static inline _type class_##_name##_constructor(_init_args) \
	{                                                           \
		_type t = _init;                                        \
		return t;                                               \
	}
#define CLASS(_name, var)                                         \
	class_##_name##_t var __cleanup(class_##_name##_destructor) = \
		class_##_name##_constructor
```

与指针的自动释放类似，这两个宏将*cleanup*属性应用到了结构体，并通过cleanup属性给结构体标记“析构函数”，来达到类似面向对象语言中的class“类”。作用域结束时，和C++/Rust等面向对象语言一样，这个对象会自动析构，达到了_RAII_的效果。

内核代码中，直接应用`DEFINE_CLASS()`宏来定义类的情况并不多，主要都是间接用在下一节要介绍的锁的RAII自动释放。**这是C语言的特点决定的，因为这样声明的结构体“对象”几乎被完全限制在了这个作用域中，很难作为返回值返回或者赋值给其他结构体**。

笔者这里可以提供一个简单的例子，比如可以使用`DEFINE_CLASS()`声明一个类用来自动追踪某个函数或作用域的耗时，或者调用接口实现trace等功能，代码如下：

```c
struct __time_info {
	size_t create_time;
	const char *file;
	const char *func;
	int line;
};
DEFINE_CLASS(
    // 唯一的名字，与DEFINE_FREE类似
    time_consume,
    // 定义的这个类对应的真正类型
    struct __time_info,
    // 析构函数
    ({
       size_t now = get_time_ms();
       printf("%s:%d [%s], %zu\n", _T.file, _T.line, _T.func, now - _T.create_time);
    }),
    // 构造函数
    ({
       size_t now = get_time_ms();
       (struct __time_info){
           .create_time = now,
           .file = file,
           .func = func,
           .line = line,
       };
    }),
    // 构造函数的参数列表
    const char *file, const char *func, int line
)
```

那么可以通过CLASS宏声明这个类的一个对象，来追踪这个作用域的耗时，比如

```c
int foo(...)
{
    CLASS(time_consume, _t1)(__FILE__, __func__, __LINE__); // 这个函数可以继续通过宏定义封装
    // do something here
    // ......
    return 0;
}
```

#### guard: 不用手动释放的锁

有了可以RAII的类，自然就可以通过定义类似C++的lock\_guard，利用RAII实现锁的自动释放。内核提供了如下的宏定义来实现锁的自动释放：

```c
#define DEFINE_GUARD(_name, _type, _lock, _unlock)                      \
	DEFINE_CLASS(                                                       \
		_name, _type, if (_T) { _unlock; }, ({                          \
			_lock;                                                      \
			_T;                                                         \
		}),                                                             \
		_type _T);                                                      \
	static inline void *class_##_name##_lock_ptr(class_##_name##_t *_T) \
	{                                                                   \
		return *_T;                                                     \
	}
#define guard(_name) CLASS(_name, __UNIQUE_ID(guard))

#define scoped_guard(_name, args...)              \
	for (CLASS(_name, scope)(args), *done = NULL; \
		 __guard_ptr(_name)(&scope) && !done; done = (void *)1)
```

其中的核心是`DEFINE_GUARD()`，这个宏定义定义了一个锁的包装类。通过`guard()`或`scoped_guard()`会自动创建一个RAII对象，在作用域结束时进行对应的unlock操作。该宏定义保证了声明的变量名是唯一的，所以在同一个作用域内，可以调用多个`guard()`或`scoped_guard()`。

比如，内核代码中提供了自旋锁相关的guard类，使用`guard()`后，在函数任意位置直接return，**锁都会通过编译器自动释放**。

```c
DEFINE_LOCK_GUARD_1(raw_spinlock, raw_spinlock_t,
		    raw_spin_lock(_T->lock),
		    raw_spin_unlock(_T->lock))
static raw_spinlock_t lock;
void foo_sync(...)
{
    guard(raw_spinlock)(&lock);
    // do something
    // ...
    return;
}
```

### 场景和优势

说了这么多，不如直接看看内核某个函数使用*cleanup*前后的代码有啥变化吧，这样能对*cleanup*的场景和优势有更多的了解。

`wake_up_if_idle()`函数，定义在kernel/sched/core.c。左侧为v6.5版本，右侧为引入了基于作用域的资源管理机制的v6.6版本。两边蓝色方框对应了rcu的获取和释放，黄框对应了rq的锁的获取和释放。

可以看过，通过*cleanup*改写后，使用guard获取锁后，就无需再关心释放问题，代码更加简洁清晰。

![kernelcmp.png](/image/cleanup/kernelcmp.png)

## 4 cleanup属性的不足

*cleanup*属性虽然给C语言带来了类似RAII的特点，减少了开发者因低级失误造成的问题。但是，和内核另一编程语言Rust相比，*cleanup*属性的基于作用域的资源管理仍有许多不足之处。程序员仍然需要小心使用*cleanup*，以免出现bug。

以下，是我在实际使用*cleanup*属性时发现的问题，不知道大伙们有没有什么解决办法，或者发现了其他的不足之处，欢迎在评论区分享讨论。

1.  **缺少编译检查**：这个主要针对内存释放`__free()`宏。C语言编译器无法判断`__free()`中指定的**释放函数和内存申请的函数是否匹配**，仍然和此前一样，需要程序员自己保证。这是C语言特性决定的。**要避免这个问题，可能还需要进一步封装裸指针**。Rust则不存在这个问题，因为堆上的对象创建时会带一个Allocator trait，这里就不详细展开了。
    
2.  **没有生命周期的约束**：Rust通过编译时的所有权、生命周期分析和借用检查，在编译时即可得知内存的使用情况，以此在适当的位置释放内存。C语言的*cleanup*则只能**依赖局部变量当前作用域，超出作用域后就没法保证内存的安全性了**，仍然**需要程序员手动保证**。也就是说，程序员需要格外关注`no_free_ptr()`后的内存使用情况。
    
3.  **难以灵活手动控制释放**：Rust通过生命周期机制，可以在对象创建后手动调用`drop()`函数提前结束其生命周期并释放内存。而在C语言中，咱们通过*cleanup*属性声明的指针还好说，可以手动`free(no_free_ptr(p)))`这种很“别扭”的方式达到提前释放的目的。但是结构体对象想提前释放就比较困难了，只能依赖作用域结束。**不知道C++是怎么解决此类问题的**。
    
4.  **静态检查工具的误报**：cppcheck、clang analyzer等C/C++代码的静态分析工具能帮助开发者在前期分析出程序可能存在的内存问题等各类问题。因此，*cleanup*属性是否会引起静态检查的误报需要关注。经过笔者实验，在用户态C程序开发中，使用*cleanup*属性声明的文件指针、内存等，**均会导致静态分析器误报**，认为编程者没有关闭文件或释放内存。
    

与Rust做对比，主要是因为Rust是Linux内核的另一编程语言，风头正盛，Rust的安全性能解决当前C语言大型项目面临的一些问题，学术界和工业界都对其寄予厚望。（为什么不和C++对比呢？~~因为我研究生阶段一直在写Rust和C，对Rust有一些体会，对现代C++的什么移动语义啊左值啊完美转发什么的特性真不熟悉~~）

## 5 结语

本文从代码层面介绍探讨了基于编译器提供的*cleanup*属性实现的基于作用域的资源管理机制。这只是编程开发中很小的一个点，与实际业务基本无关，更多是出于一个程序员的代码洁癖。

*cleanup*属性并不是一个新东西，GCC/Clang在很早的版本就已经支持了这一属性。在Linux v6.6 之前，已有不少知名的C语言的开源项目使用了基于*cleanup*属性来管理互斥锁、内存等资源，其中就包含了大名鼎鼎的GLIB（G指GNOME）和QEMU，只是这套方案相对更heavy更“面向对象”，可能依赖函数指针或者间接调用，所以Linux内核没有采用（Peter Zijlstra 在这里提到了这个观点[https://lwn.net/ml/linux-kernel/20230526150549.250372621@infradead.org/](https://lwn.net/ml/linux-kernel/20230526150549.250372621@infradead.org/)）。

内核实现的这套*cleanup*头文件的helper宏的可移植性非常的好，可以轻松移植到各种C语言项目中。这种轻量的RAII机制给三十多岁的Linux内核带来了更安全的资源管理方法。只是，由于C语言的固有问题，这并不能解决长期以来C程序面临的安全性挑战。Rust也不能解决所有问题。

想起之前在github看到的一个很有意思的项目，通过*cleanup*属性在C语言中实现了**智能指针** [C smart pointer libcsptr](https://github.com/Snaipe/libcsptr)来完全自动内存管理了。虽然这个项目不能用于严肃的生产开发，但是思路可以参考参考。

## 参考

1.  LWN关于内核的基于作用域的资源管理的帖子：[Scope-based resource management for the kernel](https://lwn.net/Articles/934679/)
    
2.  什么是作用域：[Scope - GNU C Manual](https://www.gnu.org/software/c-intro-and-ref/manual/html_node/Scope.html)
    
3.  GCC/Clang两大编译器对cleanup属性的文档：[GCC - cleanup attribute](https://gcc.gnu.org/onlinedocs/gcc/Common-Variable-Attributes.html#index-cleanup-variable-attribute), [Clang - cleanup](https://clang.llvm.org/docs/AttributeReference.html#cleanup)
    
4.  QEMU依赖GLIB对锁和内存的自动管理：[QEMU - guard macros](https://www.qemu.org/docs/master/devel/style.html#qemu-guard-macros), [QEMU - automatic memory deallocation](https://www.qemu.org/docs/master/devel/style.html#automatic-memory-deallocation)
    
5.  宋宝华讲Linux内核2023的变化：[熠熠生辉 | 2023 年 Linux 内核十大技术革新功能](https://blog.csdn.net/csdnnews/article/details/135493424)