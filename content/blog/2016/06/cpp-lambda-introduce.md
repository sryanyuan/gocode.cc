+++
author = ""
categories = ["cpp"]
date = "2016-06-03T16:23:38+08:00"
description = ""
linktitle = ""
title = "初见 C++ lambda表达式"
type = "post"

+++

## 起因

今天想翻一翻有没有开源的冒险岛服务端源码，于是在github上找了找，java版本的很常见了，我以前也玩了会，还是很不错的，可惜很多任务都做不了，想回忆下童年貌似也很困难：（

还翻到一个c++版本的，随便打开一个文件看了看，有点儿傻眼，这是c++写的？

	namespace Vana {
		class PacketReader;
	
		namespace ChannelServer {
			class WorldServerSession final : public PacketHandler, public enable_shared<WorldServerSession> {
				NONCOPYABLE(WorldServerSession);
			public:
				WorldServerSession() = default;
			protected:
				auto handle(PacketReader &reader) -> Result override;
				auto onConnect() -> void override;
				auto onDisconnect() -> void override;
			};
		}
	}

有点儿懵逼了，我开始怀疑我究竟会不会c++了。当然我还是有点儿印象的，我只知道auto这个东西貌似是在c++的新标准里引入的，那么->这个应该也是新标准引入的吧，可是我的第一印象却是lambda表达式，这个和lambda表达式有什么关系呢？

## Lambda表达式

lambda表达式是一个很有用的东西，我在c++外常用lua和golang，这两个都提供闭包支持，在写回调函数的时候非常方便。lambda就是类似于c++的闭包函数，用于在函数内定义匿名回调函数。

lambda表达式的基本格式为(<>内的为说明 不包含该符号)

    [<import variables>](<parameters>)-><return type>

### import variables

闭包函数的一个重要特性就是可以获取函数外的变量，在golang和lua中是保持着长久的引用，不会因为函数退出堆栈改变而受到影响。

c++则不是，引用的变量假如是引用方式引入的，那么当写lambda表达式的函数结束后，那引入的变量已经是有问题的了。

c++支持值和引用方式传入变量。

* 当写为[]时，则为不引入任何的变量
* 当写为[=]时，通过值复制方式引入所有的外部自动变量(栈上变量)
* 当写为[&]时,通过引用方式传入所有的外部自动变量

还有一个特殊的地方就是传入this的引用，按规则是除了[]之外，默认都会传入this的拷贝。

另外，还可以指定特定变量的引入方式。

    int var = 0;
    [var]()->void{printf("%d", var);}();

则为通过值复制的方式引入了外部变量。

    int var = 0;
    [&var]()->void{printf("%d", var);}();

则为通过引用方式引入了外部变量。

引用外部变量，默认情况下是禁止修改外部变量的。可以通过指定mutable关键字来修改外部变量。

    // version 1
	int var = 0;
    [var]()mutable->void{printf("%d", var);++var;}();

    // version 2
    int var = 0;
    [&var]()mutable->void{printf("%d", var);++var;}();

上面两种写法都可以，只是通过传引用方式传入的变量，会在version 2中被修改。

### parameters

这个就没有什么好讲的了，只是指定参数而已，和普通的函数定义没有什么两样。

### return type

这个也没什么好说的，指定返回的类型而已。

### 简化

当然上面的写法是完整的写法，实际上假设有一些东西没用到，可以简化。比如假设我们没有返回类型，亦或者是希望编译器自己推导返回类型，我们可以这样写：

    [](){printf("func\n");}();

## functional

我们可以将lambda表达式赋值给function对象，然后我们可以使用function对象来当做函数指针进行调用。

    #include <functional>

    std::function<int(int, const char*)> func = [](int cnt, const char* pMsg)->int{for(int i = 0; i < cnt; ++i){printf("%s\n", pMsg);}return cnt + 1;};
    func(3, "hello function");

简单的来说，就是return type(arg type, arg type ...)这样的形式。


## 总结

回到开头，开头那个令我懵逼的写法，就是仿照lambda表达式的写法，用auto作为占位符，然后后面则是-> return type。