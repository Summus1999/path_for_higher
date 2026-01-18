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