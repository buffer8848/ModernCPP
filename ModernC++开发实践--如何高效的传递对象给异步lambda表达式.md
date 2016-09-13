# 如何高效的传递对象给异步lambda表达式

在c++移动语义(std::move)到来之前，我们在异步的lambda表达式中传递对象的时候，往往需要传值给异步的lambda才能保证对象在有效的生命周期被使用。
伪代码如下：
```
#include <iostream>
#include <future>
#include <string>

class Object {
public:
    Object() : value_("Hello world.") {
        std::cout << "Object default construct." << std::endl;
    }

    ~Object() {
        value_ = "";
    }

    Object(const Object& rhs) : value_(rhs.value_) {
        std::cout << "Object copy construct." << std::endl;
    }

    void Do() const {
        std::cout << "result: " << value_ << std::endl;
    }

private:
    std::string value_;
};

int main() {
    std::future<void> f;
    {
        Object object;

        f = std::async(std::launch::deferred, [&object]{
            object.Do();
        });
    }

    f.get();

    getchar();
    
}
```
打印出来的值：
```
Object default construct.
result:
```
说明object已经析构了。这个时候我们改成传值：

```
int main()
{
    std::future<void> f;
    {
        Object object;

        f = std::async(std::launch::deferred, [object] {
            object.Do();
        });
    }

    f.get();

    getchar();
}
```
打印结果：
```
Object default construct.
Object copy construct.
Object copy construct.
Object copy construct.
Object copy construct.
Object copy construct.
result: Hello world.
```
但是中间会产生很多次的copy， 本例中使用string作为成员变量来演示copy的性能损耗，如果是更大的对象，copy的代价不言而喻。（本文先补纠结为什么在std::async下有五次的copy被调用，以后将详细介绍它）

有了std::move之后我们采用移动语义来实现，同时我们需要自己实现类的移动构造函数和移动拷贝函数来支持，这些函数中都采用std::move来达到减少copy的作用，效率显然比之前的非移动效率高。代码如下：

```
#include <iostream>
#include <future>
#include <string>

class Object {
public:
    Object() : value_("Hello world.")
    {
        std::cout << "Object default construct." << std::endl;
    }

    Object(const std::initializer_list<Object> &rhs) {
        *this = std::move(rhs);
    }

    ~Object() {
        value_ = "";
    }

    Object(const Object& rhs) : value_(rhs.value_) {
        std::cout << "Object copy construct." << std::endl;
    }

    Object(Object&& rhs) : value_(std::move(rhs.value_)) 
    {
        std::cout << "Object move copy construct." << std::endl;
    }

    Object& operator=(const Object& rhs) {
        value_ = rhs.value_;
        std::cout << "Object assign construct." << std::endl;

        return *this;
    }

    Object& operator=(Object&& rhs) {
        value_ = std::move(rhs.value_);
        std::cout << "Object move assign construct." << std::endl;

        return *this;
    }
    void Do() const {
        std::cout << "result: " << value_ << std::endl;
    }

private:
    std::string value_;
};

void print(const Object& object) {
    object.Do();
}

int main() {
    std::future<void> f;
    {
        Object object;

        f = std::async(std::launch::deferred, [object] {
            print(object);
        });

        //f = std::async(std::launch::deferred, std::bind([](const Object &object){
        //    object.Do();
        //}, std::move(object)));
    }

    f.get();

    getchar();
}
```
结果如下：
```
Object default construct.
Object copy construct.
Object move copy construct.
Object move copy construct.
Object move copy construct.
Object move copy construct.
result: Hello world.
```
我们发现编译器会自动去调用移动语义相关函数，这里已经达到进一步优化作用了。但是这里还会又一次的copy调用，那如何减少这个代价呢，在lambda里面我们还不能直接传入std::move(object)，我们来试试std::initializer_list：
```
f = std::async(std::launch::deferred, [obj{ std::move(object) }] {
    print(obj);
});
```
直观上讲，似乎我们已经做到了将object的生命周期转移到了异步的lamba表达式上下文中，其实不然，我们来看看结果：
```
Object default construct.
Object move copy construct.
Object move copy construct.
Object move copy construct.
Object move copy construct.
Object move copy construct.
result: Hello world.
```
完美，其实在初始化列表到来之前，也可以通过std::bind来完美解决的，如下：
```
f = std::async(std::launch::deferred, std::bind([](const Object &object){
    object.Do();
}, std::move(object)));
```
只是实现起来肯定没有上面的优雅了。

