+++
author = ""
categories = ["Go"]
date = "2016-05-31T14:43:21+08:00"
description = ""
linktitle = ""
title = "自己封装的一些golang库"
type = "post"

+++

自己在写go代码的时候，貌似也封装了一些小东西，除了上篇文章的tcp server，还有几个小东西，这段时间也传到了github上了，很简单的小东西。

## eventdispatcher

这是一个简单的事件分发器，可以注册自己的事件以及对应的处理函数到上面去，别的地方可以进行回调，典型的观察者模式，可以给代码解耦。

### install

    go get github.com/sryanyuan/eventdispatcher

### 用法

    package main

	import (
	    "log"
	
	    "github.com/sryanyuan/eventdispatcher"
	)
	
	const (
	    kEvent_SayHello = iota
	)
	
	type SayHelloArg struct {
	    times int
	}
	
	func sayHello1(data interface{}) {
	    arg, ok := data.(*SayHelloArg)
	    if !ok {
	        return
	    }
	
	    for i := 0; i < arg.times; i++ {
	        log.Println("hello1\n")
	    }
	}
	
	func sayHello2(data interface{}) {
	    arg, ok := data.(*SayHelloArg)
	    if !ok {
	        return
	    }
	
	    for i := 0; i < arg.times; i++ {
	        log.Println("hello2\n")
	    }
	}
	
	func main() {
	    dip := eventdispatcher.NewEventDispatcher()
	    handle1 := dip.AddListener(kEvent_SayHello, sayHello1)
	    handle2 := dip.AddListener(kEvent_SayHello, sayHello2)
	    arg := SayHelloArg{
	        times: 3,
	    }
	    dip.Dispatch(kEvent_SayHello, &arg)
	
	    //  remove listener
	    dip.RemoveListener(handle1)
	    dip.RemoveListener(handle2)
	    dip.Dispatch(kEvent_SayHello, &arg)
	}


## circle index pool

这是一个索引的池子，主要用于固定长度数组的索引管理使用。通俗点来讲就是假如你管理了一个最大数量为100的连接，然后你准备了大小为100的数组，然后你就可以为每个连接分配索引了，当索引不够用的时候，就代表了当前的连接数大于了100，就可以做一些处理了。

为什么需要这个东西呢？这样设计的话，可以让寻址更加的快，不用使用各种map，直接使用数组，当有一个索引的时候，找到这个元素的速度可是比map快不少，在有上限的服务中，这个东西还是很有用的。

### install

    go get github.com/sryanyuan/circleindexpool

### 用法

	package main
	
	import (
	    "log"
	
	    "github.com/sryanyuan/circleindexpool"
	)
	
	func main() {
	    c := circleindexpool.NewCircleIndexPool(10)
	
	    //  you can get index(1 ~ 10) from the pool
	    for {
	        i := c.AllocateIndex()
	        if i == 0 {
	            break
	        }
	
	        log.Println("Got index", i)
	    }
	
	    //  you can free the index and use it later
	    for i := 1; i < 10; i += 2 {
	        c.FreeIndex(i)
	    }
	
	    log.Println("After free:")
	    for {
	        i := c.AllocateIndex()
	        if i == 5 {
	            break
	        }
	
	        log.Println("Got index", i)
	    }
	
	    //  free all index
	    for i := 1; i <= 10; i++ {
	        c.FreeIndex(i)
	    }
	
	    log.Println("Total:")
	    for {
	        i := c.AllocateIndex()
	        if i == 0 {
	            break
	        }
	
	        log.Println("Got index", i)
	    }
	}

## luago

这是一个go包装的lua引擎，封装的lua版本为lua 5.2.2。其实也没怎么封装，基本导出了lua中所有的c函数，假如熟悉lua的c api，那么使用起来也很方便。

当然既然要在go中使用，那么导入go函数到lua中也是必要的，我也简单的实现了go中的函数注册到lua的需求。

### install

    go get github.com/sryanyuan/luago

### 用法

	package main
	
	import (
	    "strconv"
	
	    "github.com/sryanyuan/luago"
	)
	
	func export(L luago.Lua_Handle) int {
	    //  get args from lua
	    num := int(luago.Lua_tonumber(L, -2))
	    str := luago.Lua_tostring(L, -1)
	
	    //  push value to lua
	    val := str + strconv.Itoa(num)
	    luago.Lua_pushstring(L, val)
	    return 1
	}
	
	func main() {
	    L := luago.LuaGo_newState()
	    L.OpenStdLibs()
	
	    L.LuaGo_PushGoFunction("export", export)
	
	    //  invoke
	    luago.LuaL_dostring(L.GetHandle(), ` 
	        local val = export(1, "hello luago")
	        print(val)
	    `)
	}