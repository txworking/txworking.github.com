---
layout: post
title: "Facter在Windows上的一个小问题"
date: 2013-07-08 23:37
comments: true
categories: [DevOps, Facter, Puppet]
---

最近在Windows上使用Puppet，总是遇到一个诡异的问题：在某些机器上无法获得ipaddress这个fact，直接使用facter命令行也不行，而且显得完全没有规律。

具体情况是：

-   facter 在所有机器上可以获得ipaddress
-   facter --puppet 在部分机器上出现 **invalid address** 错误

**ipaddress**这个fact是在facter/lib/facter/ipaddress.rb添加进来的。关键代码是:
```ruby
    Facter.add(:ipaddress) do
      confine :kernel => %w{windows}
      setcode do
        require 'socket'
        IPSocket.getaddress(Socket.gethostname)
      end
    end
```
发现是**IPSocket.getaddress**函数不起作用，于是又把问题定位在Puppet自带的Ruby环境中的**ipaddr.rb**中，发现以下代码在某些机器上无法获得IP。
```ruby
alias getaddress_orig getaddress
def getaddress s    
  if valid? s
    s    
  else
    getaddress_orig s    
  end 
end
```

一开始没发现哪有问题，后来仔细看了下**valid**中的正则表达式，发现不允许**hostname**中出现**下划线**。

之前一直没注意过这问题，所以测试主机的hostname都是随意命名的，所以才会出现部分主机可以通过验证，部分不行的情况。

不过也是现在才知道原来主机名规范有一条是不能带下划线。

Windows命名的时候完全没这个限制，但是在Ubuntu里使用**hostname**命令修改时，如果带下划线会直接提示错误。

很奇怪自己以前在Ubuntu上怎么没遇到过这个错误，难道就鬼使神差的从没使用过带下划线的主机名？

但是现在问题又来了，为什么facter都可以，facter --puppet就不行呢？

于是又研究了一遍facter的加载机制，在facter取值的部分折腾好久，最后发现和这部分没关系。是因为执行facter时没有加载**facter/lib/facter/ipaddr.rb**，ipaddr.rb中对**IPSocket\#getaddress**的重写未生效，直接使用了Puppet自带Ruby中的socket.so中定义的**IPSocket\#getaddress**，这个函数定义里面就没有对主机名的正则验证。

想来Puppet也是为了做多平台支持，所以加了一个MonkeyPatch，对主机名做了验证。

现在又绕回来了，--puppet 到底干了啥？
```ruby
     def self.load_puppet
          require 'puppet'
          Puppet.parse_config

          # If you've set 'vardir' but not 'libdir' in your
          # puppet.conf, then the hook to add libdir to $:
          # won't get triggered.  This makes sure that it's setup
          # correctly.
          unless $LOAD_PATH.include?(Puppet[:libdir])
            $LOAD_PATH << Puppet[:libdir]
          end
        rescue LoadError => detail
          $stderr.puts "Could not load Puppet: #{detail}"
     end
```
根据代码来看，只是把Puppet的**libdir**加入了**\$LOAD\_PATH**。

这个libdir里面都是些自定义的facter、tyep和provider，似乎和ipaddr.rb完全没关系。

又卡在这儿想了半天，才发现思路错了。

ipaddr.rb虽然在lib目录下，但是不会自动加载，需要显式**require**。

于是注释掉
```ruby
    require 'puppet'
```
使用facter --puppet也就和正常一样了。

