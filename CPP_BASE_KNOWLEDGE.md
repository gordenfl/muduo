#  现代C++ 的基本知识

## <strong style="color: red;"> std::bind </strong>


std::bind 在这里的会返回一个函数指针，具体有几个参数是可以由后面的 _1, _2, _3 ..., 代表第一个第二个第三个这样来的。

绑定普通函数：
```CPP
std::bind(add, 5, std::placeholders::_1);
```
绑定类成员函数：
```CPP
std::bind(&Class::method, &obj, _1, _2);
```
改变参数顺序：
```CPP
std::bind(func, _2, _1);
```
绑定智能指针：
```CPP
std::bind(&Class::method, shared_ptr, _1);
```
替代方案： 
```CPP
lambda
```

## <strong style="color:red;"> std::share_ptr<str>   </strong>
智能指针，你传递一个类进去以后，他会给你创建这个类的实例并且返回指向这个类的智能指针，你不需要去管这个类的对象什么时候析构
```CPP
std::share_prt<TcpConnection> connection;
connection->send("AAAAAA");
```
不需要后面去delete 这个指针。他有引用技术和自动回收功能。就跟用 Python 的变量一样了

## <strong style="color:red;">typedef std::function<void()> EventCallback;</strong>
定义一个函数指针，这个函数指针的函数类型是 void XXX()  类似这样的函数
```CPP
typedef std::function<void()> EventCallback;
void print() {
    cout << "Hello World!" << endl;
}
int main() {
   EventCallback func = print;
   func();
}
```

## <strong style="color:red;">  class Channel : noncopyable {};</strong>
这个类的继承在现代 C++中很常见，noncopyable 这个类是 boost 库提供的，这个类的继承是不允许此类定义拷贝构造函数和拷贝赋值。
如果不想使用 boost 的话自己也可以很容易实现：
```CPP
class noncopyable {
protected:
    noncopyable() = default;
    ~noncopyable() = default;

    noncopyable(const noncopyable&) = delete; //这里是拷贝构造
    noncopyable& operator=(const noncopyable&) = delete; //这里是赋值

    noncopyable(noncopyable&&) = delete; //这里是指针拷贝构造
    noncopyable& operator=(noncopyable&&) = delete; //这里是引用赋值
};
```
继承这个类的话既不能被拷贝也不能被赋值。

## <strong style="color:red;">  std::move(void*);</strong>
这个函数的参数可以是任何东西的指针，他的作用就是将 cb 直接移动到对应的位置上去比如：
```CPP
void setWriteCallback(EventCallback cb) {//EventCallback 类型是一个智能指针，所以 cb 就是一个指针
  writeCallback_ = std::move(cb); //这个跟传引用差不多，但是放在 lambda 还有类的成员函数上是不可以传引用的
}
```
std::function<void()> + std::move： 支持 lambda，支持成员函数，有可能会拷贝，性能方面设计动态非配， 灵活性高
void (&cb)() （函数指针引用）：不支持 Lambda，不支持类的成员函数，无拷贝，性能最高，灵活性差

## explicit 关键字在 C++ 构造函数中的应用
这个关键字就是属于，避免在 C++类中，有且只有一个参数的构造函数被隐式转换
```CPP
class A {
  public A(int x) {_x = x;} //如果这里是引用就不会被强制转换了
  public int getX() {return _x;}
};
void func(A a) {
  cout << "function be called" << a.getX() <<pre endl;
}
void main() {
  func(19); //19 就会被自动转换成 A 对象
  func(A(19)); //这样才是正确的调用方式
}
```

## std::unique_ptr<> 的含义
创建尖括号中的类型只能用自己的这个名字来访问不能赋值给别人
```CPP
std::unique_ptr<A> p1 = std::make_unique<A>();
std::unique_ptr<A> p2 = p1;  // ❌ 编译错误！不能复制 unique_ptr

std::unique_ptr<A> p1 = std::make_unique<A>();
std::unique_ptr<A> p2 = std::move(p1);  // ✅ p1 的所有权转移到 p2， 当 p1 的所有权转移到 p2 之后，p1 变为空，不能再访问原对象。
```

##  std::deque
类似 std::vector, 它可以从头不和尾部增加值，和 pop 值， python 中 list
```CPP
std::deque<int> dq;

    dq.push_back(1);   // 尾部插入
    dq.push_front(2);  // 头部插入
    std::cout << "Front: " << dq.front() << "\n"; // 2
    std::cout << "Back: " << dq.back() << "\n";   // 1
    dq.pop_front();   // 删除头部
    dq.pop_back();    // 删除尾部
```
