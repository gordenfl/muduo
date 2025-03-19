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
std::bind(&Class::method, shared_ptr, _1); //他的返回值是一个 void()类型的
```
替代方案： 
```CPP
lambda
```
非常要注意的是，bind 的时候说传入的值，就是以后在调用这个函数时候的值：
 ```CPP
#include <iostream>
#include <functional>

class Curl {
public:
    void onWrite(int fd) {
        std::cout << "Writing on fd: " << fd << std::endl;
    }
};

int main() {//整个函数将输出的值是 10 而不是二十
    Curl curl;
    int fd = 10;

    auto callback = std::bind(&Curl::onWrite, &curl, fd); // 绑定时拷贝 fd = 10，这是所使用的是 fd 的值而不是定义一个变量
    fd = 20;  // 修改 fd 的值

    callback(); // 调用时仍然使用 bind 时的 fd = 10

    return 0;
}
```
所以不需要 callback 连的逻辑往外面传值。

## <strong style="color:red;"> std::share_ptr<str>   </strong>
智能指针，你传递一个类进去以后，他会给你创建这个类的实例并且返回指向这个类的智能指针，你不需要去管这个类的对象什么时候析构。

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

## <strong style="color:red;"> explicit 关键字在 C++ 构造函数中的应用</strong>
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

## <strong style="color:red;">  std::unique_ptr<> 的含义</strong>
创建尖括号中的类型只能用自己的这个名字来访问不能赋值给别人
```CPP
std::unique_ptr<A> p1 = std::make_unique<A>();
std::unique_ptr<A> p2 = p1;  // ❌ 编译错误！不能复制 unique_ptr

std::unique_ptr<A> p1 = std::make_unique<A>();
std::unique_ptr<A> p2 = std::move(p1);  // ✅ p1 的所有权转移到 p2， 当 p1 的所有权转移到 p2 之后，p1 变为空，不能再访问原对象。
```

##  <strong style="color:red;"> std::deque</strong>
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


## <strong style="color:red;"> typedef std::function<void ()> Task </strong>
typedef void(void) Task
本来没看懂这是什么意思，表达定义个函数 参数为空，返回值为空的函数指针，名字叫 Task
这个名字叫 Task 的类型可以创造实例，并且指向一个返回值 void，没有参数的函数，
也可以创造一个实例并且指向 一个 lambda，某个类的成员函数。这个比过去的函数指针要方便很多，过去的函数指针类型定义是 void (*a)();
可以用 void(*a)() = hello
```CPP
void hello() {
  printf("Hello WOrld\n");
}
```
这样你就可以调用 a() 了； 两者是不一样的。

## using 在新版C++中的含义
```CPP
//typedef std::function<void(int a, int b)> ADD; //这句和下面一句话是等同的
using ADD std::function<void(int a, int b)>;

int add(int num1, int num2) {
  return num1+num2;
}
int main() {
  ADD func = add;
  cout << func(1,3) << endl;
}

```
上面个两句话是等同的

## class A: public std::enable_shared_from_this<A> {};
这个enable_shared_from_this是一个 STL 的标准库提供的辅助函数
辅助什么呢？他允许对象安全的创建 share_ptr 共享实例。就是保证在类的成员函数中可以获取 对自己的share_ptr指针：
```CPP
class A : public std::enable_shared_form_this<A> {
public: 
  void func1() {
    std::shared_ptr<A> self = shared_from_this(); //这个函数是enable_shared_form_this 提供的
    cout << "this object be used by:" << self.use_count() << endl; //这里获取的是 this 这个对象被多少地方使用
  }

  static std::shared_ptr<A> create() {
    return std::shared_ptr<A>(new A());
  }
private:
  A() = default;
};

//类的外面来调用：
std::shared_ptr<A> a = A::create();
a->func1(); //这里就可以获得自己被引用了几次，为了防备在类的内部被自己内部的逻辑 delete 掉
```

## size_t vs ssize_t
size_t 是一个无符号的整数，unsigned int 4个字节
ssize_t 是一个有符号的整数， 就是一个 int 4 个字节

## std::string 的使用
在 C++ std 中的 string，在创建的是后有固定的 memory， 向 string增加的数据超过这个memory 的数量的时候，就会产生新的 string 对象，C++会将旧的 string 内容 copy 到新的值上，然后加上新的 string 值。
新申请的内容大小不同的编译器不一样：
GCC Clang 会申请 1.5 或者 2 倍的原来值的大小
MSVC 会直接申请 2 倍大小

```CPP
int main() {
    std::string s;
    for (int i = 0; i < 50; ++i) {
        std::cout << "Capacity before append: " << s.capacity() << std::endl;
        s.append("A");
        std::cout << "Capacity after append: " << s.capacity() << "\n\n";
    }
}
```
这就知道了。
