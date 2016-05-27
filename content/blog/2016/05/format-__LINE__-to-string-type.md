+++
author = ""
categories = ["cpp"]
date = "2016-05-27T15:52:05+08:00"
description = "宏的使用"
featured = ""
featuredalt = ""
featuredpath = "date"
linktitle = ""
title = "将__LINE__宏格式化成字符串常量类型"
type = "post"

+++

##使用宏的方式来格式化行号

我们已经知道，我们可以使用

    __LINE__ 获得当前的行号
    __FILE__ 获得当前的文件名
    __FUNCTION__ 获得当前的函数名

这些在打印日志的时候十分常用，所以在打印日志的时候我们常常需要序列化这些东西，然而序列化这些信息，特别是行号是整型，是有点儿消耗的。

今天偶然发现了将行号直接格式化成字符串常量的方法，下面贴代码。

    #define STRINGIFY(x) #x
    #define TOSTRING(x) STRINGIFY(x)
    #define __LINESTR__	TOSTRING(__LINE__)

使用这些代码即可将行号格式化成字符串。为了更好的使用，我们可以格式化更加复杂的信息，将文件行号和函数一起格式化成常量。（vc支持，gcc编译器不支持）

    #define __WHERE__ __FILE__":"__LINESTR__" "__FILE__

gcc编译器不支持，因为gcc在拼接字符串的时候，结果必须是已定义的常量，比如文件，行号，字符串常量等，vc没问题。这样日志获得相信文件信息只需要使用上面的宏就可以了。