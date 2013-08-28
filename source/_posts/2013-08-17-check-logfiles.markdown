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

在nagios exchange里找到了[check_logfiles](labs.consol.de/nagios/check_logfiles/)这个插件，配合[incron](http://inotify.aiken.cz/?section=incron&page=about&lang=en)就可以实现实时的日志监控了。

incron是类似cron的工具，cron是基于时间，incron则是基于文件事件，底层使用了inotify系统调用。


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
