---
layout: post
title: "在Puppet中使用Hiera"
date: 2013-07-04 23:38
comments: true
categories: [DevOps, Puppet, Hiera]
---

Puppet3.x开始Hiera成为了必需的组件，实现类似ENC的功能，正好现在试试效果。

使用Hiera可以用更格式化的组织节点的定义，而且支持动态的查找加载。

以前在puppetmaster运行过程中加入pp文件，puppet是不会发现并加载此文件的。

## 使用

Puppet3.x自带了Hiera，默认的hiera配置路径是`/etc/puppet/hiera.yaml`，所以只需要新建此文件，即启用了Hiera。
```yaml
    # hiera.yaml的例子
    ---
    :backends: # 搜索yaml和json格式的文件
      - yaml
      - json
    :yaml:  # hiera数据文件的目录
      :datadir: /etc/puppet/hieradata
    :json:
      :datadir: /etc/puppet/hieradata
    :hierarchy:  # 在指定目录搜索以下名称的文件
      - "%{::clientcert}"
      - "%{::custom_location}"
      - common
```
Hiera能实现动态加载的主要点就在`:hierarchy`的配置，此配置项指定Hiera需要读取的数据文件名称，`%{::clientcert}`表示使用以节点的certname来查找。

比如`certname=node1`的节点在运行`puppet agent --test`时，puppetmaster就会去读取`node1.yaml`和`node1.json`两个文件。具体可用的变量可以[在此](http://docs.puppetlabs.com/hiera/1/variables.html)查看。

## Hiera数据

具体的Hiera数据可以使用yaml、json、puppet三种格式，puppet格式的目前没有详细文档说明。

这里以yaml格式的为例说明，json格式也差不多，就是会多很多""。
```yaml
    # node1.yaml
    ---
    string: str
    array:
      - 1
      - 2
      - 3
    hash:
       key: value
```
你可以自己写一个module，其中的init.pp可以这样写：
```
    # init.pp
    class my_mod(
      $string = hiera("string"),
      $array = hiera_array("array"),
      $hash = hiera_hash("hash")
    ) {
    }
```
然后在`site.pp`中给node1加上这个class，node1在获取catalog时就会自动的从`node1.yaml`中加载数据并赋值给`$string, $array, $hash`三个变量。`hiera(), hiera_array(), hiera_hash()`是Hiera专门用来搜索数据的函数。

Hiera也支持**自动赋值**，如果你的node1.yaml文件是这样的：
```yaml
    # node1.yaml
    ---
    my_mod::string: str
    my_mod::array:
      - 1
      - 2
      - 3
    my_mod::hash:
       key: value
```
那在my_mod的参数中就不需要再使用函数查找，Hiera会自动关联赋值。

## 在Hiera中指定Classes

如果想完全抛弃`site.pp`怎么办？用Hiera也可以办到。

只需要在`site.pp`中加上一句`hiera_include("classes")`，

在node1.yaml中再添加上classes。
```yaml
    # node1.yaml
    ---
    classes:
      - my_mod
    my_mod::string: str
    my_mod::array:
      - 1
      - 2
      - 3
    my_mod::hash:
       key: value
```
那么以后节点的classes定义就可以完全通过


