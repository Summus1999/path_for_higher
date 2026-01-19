# Smart Pointers 智能指针

本文档主要是讲的C++的三大智能指针:unique_ptr,shared_ptr,weak_ptr。选择的源码来自于`libstdc++`

## unique_ptr

独占式智能指针，确保同一时间只有一个 unique_ptr 拥有该资源。

### 数据结构

先看数据结构：

用 `tuple` 打包指针和 deleter，利用空基类优化（EBO）节省空间。

```c++
template <typename _Tp, typename _Dp>
class __uniq_ptr_impl {
    tuple<pointer, _Dp> _M_t;  // 用tuple存储指针和deleter
    pointer& _M_ptr();         
    _Dp& _M_deleter();         
    void reset(pointer __p);   // 重置指针，删除旧对象
    pointer release();         
};
```

### 控制移动语义

控制移动语义：通过模板特化实现了一个编译期策略选择器，根据 deleter 类型的特性来决定 unique_ptr 的移动能力。这是一种**编译期多态**。

```c++
  // Defines move construction + assignment as either defaulted or deleted.
  template <typename _Tp, typename _Dp,
	    bool = is_move_constructible<_Dp>::value,
	    bool = is_move_assignable<_Dp>::value>
    struct __uniq_ptr_data : __uniq_ptr_impl<_Tp, _Dp>
    {
      using __uniq_ptr_impl<_Tp, _Dp>::__uniq_ptr_impl;
      __uniq_ptr_data(__uniq_ptr_data&&) = default;
      __uniq_ptr_data& operator=(__uniq_ptr_data&&) = default;
    };

  template <typename _Tp, typename _Dp>
    struct __uniq_ptr_data<_Tp, _Dp, true, false> : __uniq_ptr_impl<_Tp, _Dp>
    {
      using __uniq_ptr_impl<_Tp, _Dp>::__uniq_ptr_impl;
      __uniq_ptr_data(__uniq_ptr_data&&) = default;
      __uniq_ptr_data& operator=(__uniq_ptr_data&&) = delete;
    };

  template <typename _Tp, typename _Dp>
    struct __uniq_ptr_data<_Tp, _Dp, false, true> : __uniq_ptr_impl<_Tp, _Dp>
    {
      using __uniq_ptr_impl<_Tp, _Dp>::__uniq_ptr_impl;
      __uniq_ptr_data(__uniq_ptr_data&&) = delete;
      __uniq_ptr_data& operator=(__uniq_ptr_data&&) = default;
    };

  template <typename _Tp, typename _Dp>
    struct __uniq_ptr_data<_Tp, _Dp, false, false> : __uniq_ptr_impl<_Tp, _Dp>
    {
      using __uniq_ptr_impl<_Tp, _Dp>::__uniq_ptr_impl;
      __uniq_ptr_data(__uniq_ptr_data&&) = delete;
      __uniq_ptr_data& operator=(__uniq_ptr_data&&) = delete;
    };
```

unique_ptr 内部通过 __uniq_ptr_data 模板特化，根据 deleter 的移动能力决定自身的移动能力。
核心原则：**unique_ptr 的移动能力不能超过其 deleter 的移动能力**。

| 移动构造 | 移动赋值 |      效果      |
| -------- | -------- | :------------: |
| true     | true     | 两种操作都能用 |
| true     | false    |  只能移动构造  |
| false    | true     |  只能移动赋值  |
| false    | false    | 两种操作都禁用 |

### 默认删除器

`default_delete`默认删除器

```c++
template<typename _Tp>
struct default_delete {
    void operator()(_Tp* __ptr) const {
        static_assert(sizeof(_Tp) > 0, "can't delete incomplete type");
        delete __ptr;
    }
};

template<typename _Tp>
struct default_delete<_Tp[]> {  // 数组特化版
    void operator()(_Up* __ptr) const { delete[] __ptr; }
};
```

### unique_ptr主体实现

主体实现代码：

```c++
template <typename _Tp, typename _Dp = default_delete<_Tp>>
class unique_ptr {
    __uniq_ptr_data<_Tp, _Dp> _M_t;  // 唯一的成员变量

public:
    // 禁止拷贝
    unique_ptr(const unique_ptr&) = delete;
    unique_ptr& operator=(const unique_ptr&) = delete;

    // 允许移动
    unique_ptr(unique_ptr&&) = default;
    unique_ptr& operator=(unique_ptr&&) = default;

    // 析构时调用deleter
    ~unique_ptr() {
        auto& __ptr = _M_t._M_ptr();
        if (__ptr != nullptr)
            get_deleter()(std::move(__ptr));
        __ptr = pointer();
    }

    // 核心操作
    pointer get() const { return _M_t._M_ptr(); }
    pointer release() { return _M_t.release(); }
    void reset(pointer __p = pointer()) { _M_t.reset(std::move(__p)); }
    
    // 解引用
    element_type& operator*() const { return *get(); }
    pointer operator->() const { return get(); }
    explicit operator bool() const { return get() != nullptr; }
};
```

#### 辅助函数make_unique

`make_unique`的实现：

```c++
template<typename _Tp, typename... _Args>
inline unique_ptr<_Tp> make_unique(_Args&&... __args) {
    return unique_ptr<_Tp>(new _Tp(std::forward<_Args>(__args)...));
}
```

## 手动写一个简单的unique_ptr

代码如下：

```c++
#include <iostream>
using namespace std;
template <typename T> class UniquePtr {
  private:
    T *ptr;

  public:
    explicit UniquePtr(T *p = nullptr) : ptr(p) {
    }
    ~UniquePtr() {
        delete ptr;
    }
    // 禁止拷贝
    UniquePtr(const UniquePtr &) = delete;
    UniquePtr &operator=(const UniquePtr &) = delete;
    // 移动构造
    UniquePtr(UniquePtr &&other) noexcept : ptr(other.ptr) {
        other.ptr = nullptr;
    }
    // 移动赋值
    UniquePtr &operator=(UniquePtr &&other) noexcept {
        if (this != &other) {
            delete ptr;
            ptr = other.ptr;
            other.ptr = nullptr;
        }
        return *this;
    }
    // 获取裸指针
    T *get() const {
        return ptr;
    }
    // 释放所有权
    T *release() {
        T *tmp = ptr;
        ptr = nullptr;
        return tmp;
    }
    // 重置指针
    void reset(T *p = nullptr) {
        if (ptr != p) {
            delete ptr;
            ptr = p;
        }
    }
    // 解引用
    T &operator*() const {
        return *ptr;
    }
    T *operator->() const {
        return ptr;
    }
    // 布尔转换
    explicit operator bool() const {
        return ptr != nullptr;
    }
};

int main() {
    UniquePtr<int> p1(new int(10));
    cout << "p1:" << *p1 << endl;
    UniquePtr<int> p2 = move(p1);
    cout << "p2:" << *p2 << endl;
    cout << "p1 is null: " << (p1.get() == nullptr) << endl;

    p2.reset(new int(20));
    cout << "p2 after reset: " << *p2 << endl;

    int *raw = p2.release();
    cout << "raw: " << *raw << endl;
    cout << "p2 is null: " << (p2.get() == nullptr) << endl;
    delete raw;
    return 0;
}
```

## shared_ptr

共享指针`shared_ptr`的实现采用控制块分离的设计，核心实现主要是两部分：

- 指针本身
- 引用计数控制块

### 引用计数基类

引用计数基类`_Sp_counted_base`内部的关键元素：

```c++
_Atomic_word  _M_use_count;     // 强引用计数
_Atomic_word  _M_weak_count;    // 弱引用计数
```

这个基类内部的核心操作只有3个：增加强引用，减少强引用，管理弱引用

- `_M_add_ref_copy()`：增加强引用
- `_M_release()`：减少强引用，为0时调用 `_M_dispose()` 销毁对象
- `_M_weak_add_ref()` / `_M_weak_release()`：管理弱引用

**关键点**：弱引用计数 = 实际弱引用数 + (强引用数 != 0 ? 1 : 0)，确保控制块在所有引用失效前存活。

### 控制块的实现

shared_ptr 的控制块实现有3种：

1. _Sp_counted_ptr
   - 基础版本，直接 delete 指针

2. _Sp_counted_deleter
   - 支持自定义删除器和分配器

3. _Sp_counted_ptr_inplace
   - 用于 make_shared，对象与控制块一次性分配，减少内存碎片

```c++
//基础版本
// Counted ptr with no deleter or allocator support
template <typename _Ptr, _Lock_policy _Lp>
class _Sp_counted_ptr final : public _Sp_counted_base<_Lp> {
  public:
    explicit _Sp_counted_ptr(_Ptr __p) noexcept : _M_ptr(__p) {
    }

    virtual void _M_dispose() noexcept {
        delete _M_ptr;
    }

    virtual void _M_destroy() noexcept {
        delete this;
    }

    virtual void *_M_get_deleter(const std::type_info &) noexcept {
        return nullptr;
    }

    _Sp_counted_ptr(const _Sp_counted_ptr &) = delete;
    _Sp_counted_ptr &operator=(const _Sp_counted_ptr &) = delete;

  private:
    _Ptr _M_ptr;
};
```

```c++
// Support for custom deleter and/or allocator
template<typename _Ptr, typename _Deleter, typename _Alloc, _Lock_policy _Lp>
class _Sp_counted_deleter final : public _Sp_counted_base<_Lp>
{
  class _Impl : _Sp_ebo_helper<0, _Deleter>, _Sp_ebo_helper<1, _Alloc>
  {
    typedef _Sp_ebo_helper<0, _Deleter>	_Del_base;
    typedef _Sp_ebo_helper<1, _Alloc>	_Alloc_base;

  public:
    _Impl(_Ptr __p, _Deleter __d, const _Alloc& __a) noexcept
    : _Del_base(std::move(__d)), _Alloc_base(__a), _M_ptr(__p)
    { }

    _Deleter& _M_del() noexcept { return _Del_base::_S_get(*this); }
    _Alloc& _M_alloc() noexcept { return _Alloc_base::_S_get(*this); }

    _Ptr _M_ptr;  // 存储指针
  };

public:
  using __allocator_type = __alloc_rebind<_Alloc, _Sp_counted_deleter>;

  // __d(__p) must not throw.
  _Sp_counted_deleter(_Ptr __p, _Deleter __d) noexcept
  : _M_impl(__p, std::move(__d), _Alloc()) { }

  // __d(__p) must not throw.
  _Sp_counted_deleter(_Ptr __p, _Deleter __d, const _Alloc& __a) noexcept
  : _M_impl(__p, std::move(__d), __a) { }

  ~_Sp_counted_deleter() noexcept { }

  // 析构对象：调用自定义删除器
  virtual void
  _M_dispose() noexcept
  { _M_impl._M_del()(_M_impl._M_ptr); }

  // 销毁控制块：使用分配器释放
  virtual void
  _M_destroy() noexcept
  {
    __allocator_type __a(_M_impl._M_alloc());
    __allocated_ptr<__allocator_type> __guard_ptr{ __a, this };
    this->~_Sp_counted_deleter();
  }

  // 获取删除器（用于get_deleter）
  virtual void*
  _M_get_deleter(const type_info& __ti [[__gnu__::__unused__]]) noexcept
  {
#if __cpp_rtti
    // 2400. shared_ptr's get_deleter() should use addressof()
    return __ti == typeid(_Deleter)
      ? std::__addressof(_M_impl._M_del())
      : nullptr;
#else
    return nullptr;
#endif
  }

private:
  _Impl _M_impl;  // 包含删除器、分配器和指针
};
```

```c++
template<typename _Tp, typename _Alloc, _Lock_policy _Lp>
class _Sp_counted_ptr_inplace final : public _Sp_counted_base<_Lp>
{
  class _Impl : _Sp_ebo_helper<0, _Alloc>
  {
    typedef _Sp_ebo_helper<0, _Alloc>	_A_base;

  public:
    explicit _Impl(_Alloc __a) noexcept : _A_base(__a) { }

    _Alloc& _M_alloc() noexcept { return _A_base::_S_get(*this); }

    __gnu_cxx::__aligned_buffer<_Tp> _M_storage;  // 对象存储区
  };

public:
  using __allocator_type = __alloc_rebind<_Alloc, _Sp_counted_ptr_inplace>;

  // 就地构造对象
  template<typename... _Args>
  _Sp_counted_ptr_inplace(_Alloc __a, _Args&&... __args)
  : _M_impl(__a)
  {
    // 2070. allocate_shared should use allocator_traits<A>::construct
    allocator_traits<_Alloc>::construct(__a, _M_ptr(),
        std::forward<_Args>(__args)...); // might throw
  }

  ~_Sp_counted_ptr_inplace() noexcept { }

  // 析构对象
  virtual void
  _M_dispose() noexcept
  {
    allocator_traits<_Alloc>::destroy(_M_impl._M_alloc(), _M_ptr());
  }

  // 销毁控制块
  virtual void
  _M_destroy() noexcept
  {
    __allocator_type __a(_M_impl._M_alloc());
    __allocated_ptr<__allocator_type> __guard_ptr{ __a, this };
    this->~_Sp_counted_ptr_inplace();
  }

private:
  friend class __shared_count<_Lp>; // 允许访问 _M_ptr()

  // 用于兼容旧版本头文件
  virtual void*
  _M_get_deleter(const std::type_info& __ti) noexcept override
  {
    auto __ptr = const_cast<typename remove_cv<_Tp>::type*>(_M_ptr());
    // 检查是否为 make_shared 特殊标记
    if (&__ti == &_Sp_make_shared_tag::_S_ti()
        ||
#if __cpp_rtti
        __ti == typeid(_Sp_make_shared_tag)
#else
        _Sp_make_shared_tag::_S_eq(__ti)
#endif
       )
      return __ptr;
    return nullptr;
  }

  _Tp* _M_ptr() noexcept { return _M_impl._M_storage._M_ptr(); }

  _Impl _M_impl;  // 包含分配器和对象存储
};
```

### shared_ptr主体实现

`shared_ptr`的主体实现主要是看数据成员和一些核心操作：

```c++
template<typename _Tp, _Lock_policy _Lp>
class __shared_ptr : public __shared_ptr_access<_Tp, _Lp>
{
private:
  element_type*        _M_ptr;         // 指向实际对象
  __shared_count<_Lp>  _M_refcount;    // 管理控制块
};
```

其余的一些核心操作就是一些构造，如从裸指针构造等。可以见`shared_ptr_base.h`中的`class __shared_ptr`。

### 手动写一个简单的shared_ptr

代码如下：

```c++

```

## weak_ptr

