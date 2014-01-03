---
layout: post
title: "Razor的架构研究"
date: 2013-01-02 14:05
comments: true
categories: [Razor]
---

{% img /images/my/razor_mk.png%}  

## Razor对iPXE的使用

iPXE是Razor主要依赖的技术，配合iPXE就能实现服务器节点固件和Razor的交互。
iPXE是开源的网络启动固件，提供了全部的PXE功能，而且加入更多的高级特性，
比如：支持从HTTP服务器、iSCSI SAN、FC SAN、AoE SAN、无线网络、广域网、Infiniband网络启动；通过脚本控制boot流程。

Razor主要利用HTTP服务器启动和脚本控制两个特性。

* 部分网卡需要特殊配置以支持通过iPXE安装Razor-Microkernel

### 服务器启动时

搭建iPXE服务需要DHCP服务和ftp服务，具体过程就不详述了。
服务器在进行网络启动时，iPXE会根据配置提供两种boot方式。下面的代码清单是iPXE的默认配置。
```
default menu.c32
prompt 0
menu title Razor Boot Menu

timeout 50
f1 help.txt
f2 version.txt

label razor-boot
  menu label Automatic Razor Node Boot
  kernel ipxe.lkrn
  append initrd=razor.ipxe

label boot-else
  menu label Bypass Razor
  localboot 1
```

可以看到，使用razor-boot方式会下载tftp服务器上的ipxe.lkrn和razor.ipxe脚本文件进行启动，如下图所示

{% img /images/my/ipxe.png%}  

服务器会根据razor.ipxe脚本文件向Razor Server发起http请求，从这里就开始进入Razor的控制流程。

```
#!ipxe

isset ${net0/mac} && dhcp net0 ||
isset ${net1/mac} && dhcp net1 ||
isset ${net2/mac} && dhcp net2 ||
isset ${net3/mac} && dhcp net3 ||
isset ${net4/mac} && dhcp net4 ||
isset ${net5/mac} && dhcp net5 ||
isset ${net6/mac} && dhcp net6 ||
isset ${net7/mac} && dhcp net7 ||

chain http://168.1.43.39:8026/razor/api/boot?hw_id=${net0/mac}_${net1/mac}_${net2/mac}_${net3/mac}_$
{net4/mac}_${net5/mac}_${net6/mac}_${net7/mac} || goto error

:error
sleep 15
reboot
```

Razor最终会返回http应答，其中包含可执行iPXE脚本，告诉服务器按何种方式启动。
比如，服务器是首次启动，则会告诉服务器下载Razor-Microkernel并启动。
下面的代码清单说明Razor返回的执行脚本内容，其中指明了kernel和initrd的下载路径

```ruby
def get_boot_script(default_mk)
        image_svc_uri = "http://#{@config.image_svc_host}:#{@config.image_svc_port}/razor/image/mk/#{default_mk.uuid}"
        rz_mk_boot_debug_level = @config.rz_mk_boot_debug_level
        # only allow values of 'quiet' or 'debug' for this parameter; if it's anything else set it
        # to an empty string
        rz_mk_boot_debug_level = '' unless ['quiet','debug'].include? rz_mk_boot_debug_level
        boot_script = ""
        boot_script << "#!ipxe\n"
        boot_script << "kernel #{image_svc_uri}/#{default_mk.kernel} maxcpus=1"
        boot_script << " #{rz_mk_boot_debug_level}" if rz_mk_boot_debug_level && !rz_mk_boot_debug_level.empty?
        boot_script << " || goto error\n"
        boot_script << "initrd #{image_svc_uri}/#{default_mk.initrd} || goto error\n"
        boot_script << "boot || goto error\n"
        boot_script << "\n\n\n"
        boot_script << ":error\necho ERROR, will reboot in #{@config.mk_checkin_interval}\nsleep #{@config.mk_checkin_interval}\nreboot\n"
        boot_script
end
```

下图是Razor Server的日志，可以看到hw_id为043a的节点在通过iPXE发起boot请求后，Server返回了Razor-Microkernl的地址。

{% img /images/my/boot.png%}  

对于服务器，在收到http应答后会执行脚本到指定地址下载Razor-Microkernel并启动，下图为服务器控制台输出

{% img /images/my/mk.png%}  

### OS安装过程中

在iPXE发起boot的http请求时，如果Razor Server发现node已经和policy进行了绑定，则返回http应答中包含的执行脚本会指明OS镜像的路径。
例如，安装Ubuntu时，返回的iPXE脚本如下代码清单所示，此代码清单为Ubuntu ModelTemplate的一部分
```
#!ipxe
echo Razor <%= @label %> model boot_call
echo Installation node UUID : <%= node.uuid %>
echo Installation image UUID: <%= @image_uuid %>
echo Active Model node state: <%= @current_state %>

sleep 3
kernel <%= "#{image_svc_uri}/#{@image_uuid}/#{kernel_path} #{kernel_args(policy_uuid)}" %> || goto error
initrd <%= "#{image_svc_uri}/#{@image_uuid}/#{initrd_path}" %> || goto error
boot
```

如下图所示，iPXE会到指定路径下载Ubuntu安装镜像进行启动

{% img /images/my/os.png%}  

## OS安装过程详解

### Razor-Microkernel发现Server方法

MK在启动时会检测Server的IP地址并主动进行checkin操作，检测方式有两种：

1. 在/proc/cmdline 文件中查找razor.ip或razor.server值作为Server的IP地址
2. 使用/etc/resolv.conf中的DNS服务器IP地址作为Server的IP地址

具体代码如下
```
def discover_rz_server_ip
      discover_by_pxe or discover_by_dns or discover_by_dhcp
end
def discover_by_pxe
      begin
        contents = File.open("/proc/cmdline", 'r') { |f| f.read }
        server_ip = contents.split.map { |x| $1 if x.match(/razor.ip=(.*)/)}.compact
        if server_ip.size == 1
          return server_ip.join
        else
          return false
        end
      rescue
        return false
      end
end

 def discover_by_dns
      begin
        contents = File.open("/proc/cmdline", 'r') { |f| f.read }
        server_name = contents.split.map { |x| $1 if x.match(/razor.server=(.*)/)}.compact
        server_name = server_name.size == 1 ? server_name.join : 'razor'

        require 'socket'
        return TCPSocket.gethostbyname(server_name)[3..-1].first || false
      rescue
        return false
      end
end
def discover_by_dhcp
      udhcp_file = "/tmp/nextServerIP.addr"
      begin
        contents = File.open(udhcp_file, 'r') { |f| f.read }
        return contents.strip
      rescue
        return false
      end
end
```

## Razor的程序结构

Razor的主要程序结构如下图所示：


{% img /images/my/node_boot.png%}

典型流程：

* node通过REST调用http://serverIP:8026/razor/api/node/checkin?json_hash='{hw_id=xxxxx,last_state=xxxx}
* Node.js服务器解析REST请求，并执行shell命令：razor \-w node checkin '{hw_id=xxxxx,last_state=xxxx}
* slice负责解析此shell命令，并调用相应方法返回应答

## Razor中Policy的绑定过程

先解释一下概念

* Node：能够被Razor管理到的一台服务器就是一个Node，可以是虚拟化或物理机
* Model：Razor中将一种OS类型的安装方式抽象为一个model，比如针对Ubuntu Precise的安装就会有一个precise的model存在，其中详细定义了OS的安装配置、安装过程、安装中的状态变化、安装中Node和Razor的交互、安装后的处理等等。model是和OS类型紧密相关的。
* Policy：Razor根据Policy将Node和Model联系在一起。Policy与Model为1对1，与Node为1对N关系。Policy的匹配规则类似防火墙的过滤规则，具有先后优先级。优先级高的Policy会先生效，并将Model和Node绑定。
* Active_model：Active_model是一个Policy对象，可以理解为生效的Policy。Razor发现某个Node符合一条Policy规则，则生成一个Policy对象，将状态设定为active，指明绑定的Node。之后OS的安装过程，Node都是根据这个Active_model定义的流程和规则在进行操作。

{% img /images/my/node_binding.png%}  

current_state与返回命令的映射表如下图： 

{% img /images/my/state.png%}  
