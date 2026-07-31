C++中的异常非常简单。

就是维护一个字符串`msg`，然后用`what()`返回这个字符串`msg`。

~~~cpp
class exception {
private:
    std::string msg;
public:
   explicit exception(const char* msg) : msg(msg) {}
   const char* what() const {
       return msg.c_str();
   }
}
~~~

就是这样。自己构造一个类继承`exception`也是这样写。

~~~cpp
class MyException : public exception {
private:
	std::string msg;
public:
	explicit MyException(const char* msg) : msg(msg) {}
	const char* what() const override {
		return msg.c_str();
	}
};
~~~

