## VC++编译器的坑

**c++标准要求临时对象不能赋值给非const引用，但微软的VC2017可以赋值
* 坑爹IDE：VS2017
* 失败IDE: codeblocks17
* 坑爹指数: 五星

### 参考资料
* [c++中临时变量不能作为非const的引用参数](http://blog.sina.com.cn/s/blog_4cce4f6a0100piuv.html)
* [临时对象不能被绑定到非const引用参数上====》扩展到临时对象问题](https://blog.csdn.net/liuxialong/article/details/6539717)

### 起因
>本以为c++11的右值引用可以用左值替代，因为左值引用也可以指向临时变量，而且不用定义为const类型。那么拷贝构造函数以及赋值都可以做到右值引用所执行的功能，完全没必要引入右值引用。在VS2017上做的测试左值引用能实现临时变量赋值，所以不得不怀疑右值引用存在的意义。

```c++
    void fun_ref(vector<int> &vi);	@3
    string s1("hello");
    string &s2 = s1 + " tom";		@1
    fun_ref(vector<int>(5));		@2
```
上面这段代码VS2017编译正常，codeblocks中编译@1和@2会报错，提示临时变量不能作为左值引用。

### 原因
原来编译器强行认为临时变量不会修改，所以就规定要带上const，否则编译不通过。其实右值移动就相当于浅拷贝，同时呢又把右值带指针的数据置空，以防析构时被释放。
其实右值引用相当于这样如下操作
```c++
	class MyTestClass {
	public:
		MyTestClass(const MyTestClass &data) {	  //@1
			if (this != &data) {
				pi = data.pi;			// 浅拷贝  @2
				pi = new int[data.len];	// 深拷贝  @3
				memcpy(pi, data.pi);	//		  @4
				len = data.len;
				// data.pi = nullptr;	//		  @5
			}
		}
		MyTestClass(MyTestClass &&rdata) {
			this->pi = data.pi;			//		  @6
			// 这里要讲指针置空，以防data析构时释放pi空间，其实也就是浅拷贝
			data.pi = nullptr;			//		  @7
			// this = std::move(data);	// 以上两步可认为等同于move操作
			this->len = len;
		}
	private:
		int *pi;
		int len;
	};
	void fun_ref()
```
#### 说明
* @1 有const标识，否则参数为临时变量时报错
* @2 @3 @4 浅拷贝和深拷贝
* @5 注释掉了，因为是const类型，否则就可以等同于右值引用了
* @6 指针浅拷贝
* @7 右值引用包含的指针置空，以防析构时清空该内存

### 右值引用作用
> c++11中引入了右值引用&&，为了解决过多临时变量导致的性能问题，尤其是类似标准库中的vector,strging等常用类型赋值，会生成很多临时变量，影响性能。比如

```c++
	void fun_object(vector<int> v) {}			// 参数作为临时变量拷贝	
	void fun_ref(vector<int> &refv) {}			// 引用不拷贝，临时变量不能作为入参
	//void fun_ref(const vector<int> &refv) {}	// 引用不拷贝，临时变量可以作为入参
	void fun_rvalue(vector<int> &&rv){}			// 右值引用，直接转移所有权

	void main() {
		vector<int> v1;
		v1.push_back(3);
		v1.push_back(5);
		vector<int> v2 = v1;		// 调用拷贝构造函数
		fun_ref(v1);				// 编译正常
		fun_ref(vector<int>(3));	// 编译器报错,error: invalid initialization of non-const reference of type 'std::vector<int>&' from an rvalue of type 'std::vector<int>'
	}
```

**参考vector源码，拷贝构造函数和赋值函数的参数都是const类型**
```c++	
explicit vector(const _Alloc& _Al) _NOEXCEPT : _Mybase(_Al)		// 拷贝构造赋值

vector& operator=(const vector& _Right)
```
**c++11后引入了右值引用，提供了移动构造函数和移动赋值函数**
```c++

vector(vector&& _Right) _NOEXCEPT : _Mybase(_STD move(_Right._Getal()))		// 右值转移所有权
	
vector& operator=(vector&& _Right)
```

### 结论，临时变量传给左值引用是可以的，只是标准这么规定
>临时变量不能作为非const引用参数，不是因为他是常量，而是因为c++编译器的一个关于语义的 限制。如果一个参数是以非const引用传入，c++编译器就有理由认为程序员会在函数中修改这个值，并且这个被修改的引用在函数返回后要发挥作用。但如 果你把一个临时变量当作非const引用参数传进来，由于临时变量的特殊性，程序员并不能操作临时变量，而且临时变量随时可能被释放掉，所以，一般说来， 修改一个临时变量是毫无意义的，据此，c++编译器加入了临时变量不能作为非const引用的这个语义限制，意在限制这个非常规用法的潜在错误。
还 不明白？OK，我们说直白一点，如果你把临时变量作为非const引用参数传递，一方面，在函数申明中，使用非常量型的引用告诉编译器你需要得到函数对某 个对象的修改结果，可是你自己又不给变量起名字，直接丢弃了函数的修改结果，编译器只能说：“大哥，你这是干啥呢，告诉我把结果给你，等我把结果给你了， 你又直接给扔了，你这不是在玩我呢吗？”所以编译器一怒之下就不让过了。这下大家明白了吧？
