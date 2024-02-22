---
title: 'Rust与系统编程 经验总结'
date: 2024-02-22T20:35:54+08:00
---

## Unsafe的使用

- 尽量分离Unsafe和Safe的作用域，避免内存泄漏；
  - 比如**eret或者idle的函数**，要提前手动析构，或者把其他逻辑单独放到一个作用域，**不能依赖Rust编译器**；（看一下panic!宏是怎么实现自动析构的）
- 可以用**unsafe标识特殊函数**，例如noreturn的函数，提示编程者注意使用；

## Clone和Copy trait

- Copy会改变这个对象在编译时对某些语法的判断，是二进制拷贝，即memcpy，永远只进行比特拷贝；**即C++缺省的拷贝构造函数；**
  - Copy是相对于Move来说的，如果没有类没有实现Copy那么它就是Move的，假如实现了Copy，那么在变量绑定、函数传参数、函数返回等都默认进行比特拷贝，且原变量还存在。
  - **尽量不实现Copy trait**
- Clone不会改变编译的方式，是开发者手动调用的；
  - RAII对象不实现Clone trait；
  - 更类似C++的自定义拷贝构造函数；

## 智能指针

与C++等其他无GC的高级语言类似，Rust智能指针需要注意以下几点

- 使用线程安全的共享指针Arc<T>；
- 注意循环引用问题，父节点指向子节点应使用StrongPtr（获取所有权），而子节点指向父节点应使用WeakPtr（不获取所有权）；
- Arc指针包裹的结构体（例如Vm和Vcpu），可以使用`Arc::new_cyclic()`函数进行初始化；
  - Vm创建的时候，会连带创建Vcpu，其中Vcpu包含一个Vm的指针，此时Vm还没有创建完。如果是Vm创建完后，再去创建Vcpu，则每次访问vcpu都需要加锁，这会降低效率。但是把Vm的vcpu_list放到InnerConst中，又会碰到这个“先有鸡还是先有蛋”的问题。
  - 此时应该使用`Arc::new_cyclic()`进行初始化。

## RAII编程思想（编程设计哲学）

Resource acquisition is initialization：资源获取就是初始化

或者叫做

- *Constructor Acquires, Destructor Releases (CADRe)* ***构造即资源获取，析构即资源释放；***
- *Scope-based Resource Management* (SBRM) **基于作用域的资源管理；**
- ownership-based resource management (OBRM)
- 总结一句话：用对象来管理资源；

**特性：RAII 将资源与对象生命周期联系起来，这可能与范围的进入和退出不一致。**

其实就是 **“利用Rust所有权机制管理系统内存”**这一特例的更通用的表述；

Wiki https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization 

- 文中提到C语言也能通过编译器做到自动析构，那么可以配合宏定义做到更便捷的操作
- [A good and idiomatic way to use GCC and clang __attribute__((cleanup)) and pointer declarations](https://stackoverflow.com/questions/34574933/a-good-and-idiomatic-way-to-use-gcc-and-clang-attribute-cleanup-and-point) C智能指针项目 https://snai.pe/posts/c-smart-pointers

Safe C++ with RAII：https://zhuanlan.zhihu.com/p/264855981

现代语言与RAII https://www.zhihu.com/question/36586808/answer/2327284017

- 用Rust所有权来管理物理资源，本质就是：存在一个Rust语言的对象，其对象本身占有一定的堆空间表示物理资源的信息；
- **持有该对象即是持有该物理资源实体**；该对象的生命周期与物理资源的获取释放时机一致；这类对象称为RAII对象；
- **RAII的对象**，需要手动实现Drop trait**，**除了不能实现copy trait，**也不能实现clone，只能存在move语义**，**这样才能保证该对象在堆上仅存在一个实体**。
  - 如何限制：C++是写一个RAII对象，把这两个函数delete掉（禁止不能实现），然后都继承这个纯虚类；Rust没有继承，能通过trait这么做吗？
- RAII对象的共享：如果需要共享，应使用引用&或者Arc<T>等智能指针来访问**。**
- RAII对象不能出现循环引用，否则会导致内存泄漏资源无法释放；
- [Why does Rust not allow the copy and drop traits on one type?](https://stackoverflow.com/questions/51704063/why-does-rust-not-allow-the-copy-and-drop-traits-on-one-type) 为什么Rust不能同时实现copy和clone

典型的RAII：

- Rust中使用Mutex包裹的对象，需要使用lock()方法去获取锁。.lock()返回的类型是MutexGuard，该对象持有锁，并在该对象生命周期结束时释放这个锁。

## Rust数据结构注意事项

- 数据结构分为两部分：
  - 一部分是inmutable的，在new的时候给定，通常是id等不可变的内容；
  - 另一部分是mutable的，使用Mutex、Rwlock、Once等数据结构加锁；
- 这样可以减少锁的开销；
- 设计范式：

```Rust
#[derive(Clone)]
pub struct MyTypeShared(Arc<MyType>);

pub struct MyType {
    inner_const: MyTypeInnerConstant,     // 无需加锁即可访问的数据
    inner_mut: Mutex<MyTypeInnerMutable>, // Rwlock等锁都可以
}
// 所有方法实现在这个结构体中，其中new方法必须是private的
impl MyType  { ... }

struct MyTypeInnerConstant { ... }
struct MyTypeInnerMutable { ... }
```

在把对象变成Arc、加入不可变的全局数组之前，对象都是可变的，所以可以使用这种方法；
