### 第14章-使用Allocator

#### 14.1 使用Allocator

#### 14.2 STL的默认Allocator

```c++
pointer allocate(size_type __n, const void* = static_cast<const void(0)>) {
  if(__n > this->max_size()) {
    std::__throw_bad_alloc();
  }
  //通过new 操作符来实现分配
  return static_cast<_Tp*>(::operator new(__n * sizeof(_Tp)));
}
```

#### 14.3 自定义allocator

需要重载max_size(), allocator()和deallocator()

#### 14.4 allocator的详细讨论

#### 14.5 未初始化的内存





