---
layout: post
title: "Ruby2.0使用小记"
date: 2013-04-09 11:26
comments: true
categories: [Ruby]
---

Ruby2.0发布了，于是下来试试手。
下载了源码包，直接`./configure && make && make install`安装，没加特殊的flag。

安装过程中出现了几个问题：

* 系统时间不对，在`make`的时候检查文件时间戳出错，不断的重复执行`configure`，使用ntp服务同步时间解决
* 在pry环境中不能使用上下方向键，错误信息是rb-readline有问题，在pry的 [Github Issues] 里面找到问题，是rb-readline版本问题，重新安装最新版本解决


然后试用了几个2.0的新特性，对我来说`关键字参数`最适用，其他几个`Refinements`、`Lazy enumerables`、`Module prepending`平时元编程和函数编程用得不多，暂时用不上。

对于关键字参数，Python里面已经有了，没什么好说的。

Ruby中关键字参数的两个要点：

* 位置：定义的时候只能是 `a, *b, c: 1, **d, &e` 的顺序
* 使用：调用时必须指明参数名称。这点和Python不同，Python可以根据参数的顺序来推断。
    
对于如下定义的函数

```ruby
def func(a: 1, b: 2, c: 3) 
  print a,b,c 
end
```

调用时使用`func(4,5,6)`会提示`wrong number of arguments`，
只能是`func(a: 4, b: 5, c: 6)`的形式，这个时候顺序可以随意了。


   [github issues]: https://github.com/pry/pry/issues/863/