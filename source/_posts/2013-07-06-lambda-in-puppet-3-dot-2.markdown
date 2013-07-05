---
layout: post
title: "Puppet3.2的迭代器"
date: 2013-07-06 00:29
comments: true
categories: [DevOps, Puppet, Lambda]
---

Puppet在最新的3.2版本中实现了lambda风格的迭代器语法，而且支持链式语法。当然要使用的话，需要在puppet.conf中启用parser=future

目前已经实现的有：

-   each/foreach
-   slice
-   select
-   collect
-   reject
-   reduce

含义和作用域与Ruby中的基本一致，基本用法也一致。
```ruby
    $a = [1,2,3]
    each($a) |$value| { notice $value } # 迭代数组

    [1,20,3].select |$value| {$value < 10 }.each |$value| { notice $value } # 链式语法
```
根据[说明](http://docs.puppetlabs.com/puppet/3/reference/lang_experimental_3_2.html#available-functions)，也可以使用多种风格的语法
```ruby
    each($x)  |$value| { … }
    $x.each   |$value| { … }
    $x.each() |$value| { … }

    slice($x, 2) |$value| { … }
    $x.slice(2)  |$value| { … }
```
或者
```ruby
    # Alternative 0 (as shown): Parameters are outside the lambda block.
    [1,2,3].each |$value| { notice $value }

    # Alternative 1: Parameters are inside the lambda block.
    [1,2,3].each { |$value| notice $value }

    # Alternative 2: A fat arrow is placed after the parameters.
    [1,2,3].each |$value| => { notice $value }
```
但是我自己实验，只有
```ruby
    [1,2,3].each { |$value| notice $value }
```
的方式成功了。

其实这些函数都是通过添加Puppet自定义函数的形式实现的，lambda表达式作为一个参数进行处理，调一个call就执行了。

比如迭代哈希的关键代码就是这样：
```ruby
      receiver = args[0]
      pblock = args[1]
      foreach_Hash(receiver, self, pblock)
      def foreach_Hash(o, scope, pblock)    
      	return nil unless pblock    
      	serving_size = pblock.parameter_count    

      	enumerator = o.each_pair    

      	if serving_size == 1      
      	  (o.size).times do        
      	    pblock.call(scope, enumerator.next)      
      	  end    
      	else      
      	  (o.size).times do   
      	    pblock.call(scope, *enumerator.next)      
      	  end    
      	end    
      	o  
      end
```
其实还是使用了Ruby中的each\_pair这个迭代器。在Puppet中写的lambda表达式还是循环调用，并把each\_pair作为每一次的参数传进去用。

感觉Puppet在解释lambda表达式时应该还做了些工作，今天就不研究了。

