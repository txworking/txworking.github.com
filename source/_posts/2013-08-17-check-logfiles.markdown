---
layout: post
title: "在Nagios中实时监控日志"
date: 2013-08-17 17:30
comments: true
categories: [Nagios, Monitor]
---

最近需要利用Nagios对Linux和Windows上的某个日志文件进行实时的监控，虽然这个活由Nagios来做不太合适，但最后
还是记录一下找到的解决方案。

## Windows

在Windows系统上使用NSClient++自带了对系统日志和日志文件的实时监控功能，只需要简单的配置下就可以使用了。当然还需要在服务器
上运行NSCA服务才能接收客户端主动推送的数据。

以下为开启对Windows系统日志监控的客户端配置，更详细配置可以参考[Real time event-log monitoring with NSClient++](http://blog.medin.name/blog/2012/03/20/real-time-event-log-monitoring-with-nsclient/)和NSClient++的[文档](http://www.nsclient.org)

    [/modules]
    CheckLogFile = enabled
    CheckEventLog = enabled

    [/settings/NSCA/client]
    hostname=<nagios_hostname> # 需要和服务器端设定的nagios host一致
    
    [/settings/NSCA/client/targets/default]
    address=168.1.194.1  
    password=nagios   # password和encryption配置需要和服务器端的nsca配置一致
    encryption=1
    
    [/settings/eventlog/real-time]
    enabled = true
     
    [/settings/eventlog/real-time/filters/run_log]  # run_log为nagios service名称
    filter=type in ( 'warning', 'error') AND source = 'Puppet' 
    target=NSCA
    severity=OK
    syntax=%type%: %strings%

## Linux

### 配合incron

在nagios exchange里找到了[check_logfiles](labs.consol.de/nagios/check_logfiles/)这个插件，配合[incron](http://inotify.aiken.cz/?section=incron&page=about&lang=en)就可以实现实时的日志监控了。

incron是类似cron的工具，cron是基于时间，incron则是基于文件事件，底层使用了inotify系统调用。


以下是check_logfiles的配置文件，主要功能就在`supersmartscript`，意思就是在每找到一行新的日志信息时执行script指定的perl脚本。
类似的配置有`supersmartpostscript`，在一次检查完后执行perl脚本。
在脚本中也支持部分变量的调用，在$MACROS中定义即可在script中使用。
在这段配置里script部分调用系统命令，执行`send_nsca`发送监控信息到服务器。

    $scriptpath = '/usr/bin/nagios/libexec:/usr/local/nagios/bin';
    $MACROS = {
        CL_NAGIOS_HOST_ADDRESS => '%server_ip%',
        CL_NSCA_HOSTNAME => '%node_name%',
        CL_NSCA_PORT => 5667,
        CL_NSCA_CONFIG_FILE => '%send_nsca.cfg%'
    };
    @searches = (
      {
        options => 'supersmartscript,noprotocol',
        tag => 'puppet',
        logfile => '%puppet_run_log%',
        criticalpatterns => [
          'Tongtech',
          'err'
        ],
        script => sub {
          (my $line = "$ENV{CHECK_LOGFILES_NSCA_HOSTNAME}\t$ENV{CHECK_LOGFILES_SERVICEDESC}\t$ENV{CHECK_LOGFILES_SERVICESTATEID}\t$ENV{CHECK_LOGFILES_SERVICEOUTPUT}");
          #system("echo '$line'");
          system("echo '$line'|%send_nsca% -H $ENV{CHECK_LOGFILES_NAGIOS_HOST_ADDRESS} -c $ENV{CHECK_LOGFILES_NSCA_CONFIG_FILE} " );
        }
      },
    );

按照常规的`./configure && make && make install`安装好incron，使用incrontab -e 命令编辑，基本的格式为

    <path-to-puppet_log> MODIFY <path-to-check_logfiles> -f <配置文件路径> --rununique
    # <path> <mask> <command>
    # path - 监控的文件
    # mask - 监控的文件事件，MODIFY表示文件有改变
    # command - 监控到指定文件的指定事件后执行的命令

具体的使用可以参考此文章[incron的使用](http://wlx.westgis.ac.cn/879/)

这样利用incron监控puppet的日志文件，当文件有改变时执行check_logfiles检查，就会发送日志信息到服务器。
注意--rununique选项，如果不加此选项，文件改变可能会触发多次检查，check_logfiles就会发送重复的信息。

### 只使用check_logfiles

check_logfiles支持以daemon方式运行，所以可以指定较短的检查周期，实现实时发送。

之前的check_logfiles配置不变，只是不使用incron执行命令，直接在shell中执行

    <path-to-check_logfiles> -f <配置文件路径> --rununique --daemon 1

check_logfiles就会变成守护进程，每隔一秒检查一次日志文件，如果有新的日志写入，就发送到服务器。
