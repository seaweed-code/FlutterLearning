## C++ 函数模版实现**SFINAE**

函数模版无法偏特化，只能重载，故而无法像类模版一样可以很方便的通过偏特化机制，实现条件选择。不过还是可以通过一些技巧来实现分支选择，例如下面两个函数模版，一个用于非整形变量，一个用于整形变量。

```c++
///T不是整数，调用此模版。通过返回值类型使得当T是整形时，该模版无效，故而两个函数模版互斥。
template <typename T>
std::enable_if<!std::is_integral_v<T>>::type fun11(T&) {
    std::cout << "other" << std::endl;
}

///T是整数 调用此模版——通过默认参数类型使得，当T不是整形时，该模版无效
template <typename T,typename = std::enable_if<std::is_integral_v<T>>::type >
void fun11(T&) {
    std::cout << "intergal" << std::endl;
}

int main() {
    int aa;
    double  ffff1;
  
    fun11(aa);///调用intergal
    fun11(ffff1);///调用 other
}
```

使用这种技巧有几个需要注意的地方：

- 防止函数模版重定义，两个模版名字一样，模版参数个数一样，函数参数个数类型一样就被认为是一个模版（函数返回值类型不算）
- 各个函数模版必须互斥，否则编译器无法选择

你可能会说，上面只有2个分支，如果是多个分支呢？下面来实现3个分支：

```c++
template <typename T>///仅当T为C数组类型是有效
std::enable_if<std::is_array_v<T>>::type fun11(T&) {
    std::cout << "array" << std::endl;
}

///仅当T为整形时有效
template <typename T,typename = std::enable_if<std::is_integral_v<T>>::type >
void fun11(T&) {
    std::cout << "intergal" << std::endl;
}
///仅当T是结构体类型是有效
///由于模版参数个数，和函数参数个数类型和上面一样，会被认为是相同模版，所以多加一个默认模版参数
template <typename T,typename = std::enable_if<std::is_class_v<T>>::type ,bool = false>
void fun11(T&) {
    std::cout << "class" << std::endl;
}

struct AA {};

int main() {
    char array[4];
    int aa;
    double  ffff1;
    AA cls;
    
    fun11(cls); ///调用class
    fun11(array);///调用array
    fun11(aa);///调用intergal
    
    fun11(ffff1);///编译错误，No matching function for call to func11
}
```

3个模版必须互斥，否则编译器无法选择；由于上面只处理了3种情况，如果传递的类型不属于这三种情况中任何一个则编译报错。

可以看到，虽然函数模版也能实现多分支，但是较为繁琐。C++17以后引入constexpr if ，则更为方便

##### C++17 的 constexpr if 

这种情况，代码就简单了。编译器会自动忽略不满足条件的分支，即便该分支语法有问题。不满足条件的分支不会出现在代码中，有点类似于宏定义的if/else

```c++
template<typename T>
  void f(T p) { 
    if constexpr (condition<T>::value) { // do something here...
    
    }else {// not a T for which f() makes sense:
       static_assert(condition<T>::value, "can’t call f() for such a T"); }
    }
}
```

