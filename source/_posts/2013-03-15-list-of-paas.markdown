---
layout: post
title: "主要PaaS平台支持语言清单"
date: 2013-03-15 21:36
comments: true
categories: [PaaS, CloudComputing]
---

|| || AWS || Elatic || Beanstalk || Google App Engine || Windows Azure || Cloud Foundry || Heroku || OpenShift || SAE || BAE || tsuru(基于Juju) ||
| Java      | x | x | x | x  | x | x | x | x |  |
| Ruby      | x |   |   | x  | x | x |   |   |  |
| PHP       | x |   | x | x* |   | x | x | x |  |
| Python    | x | x | x | x* | x | x | x | x |  |
| .NET      | x |   | x | x* |   |   |   |   |  |
| Node.js   |   |   | x | x  | x | x |   |   |  |
| Perl      |   |   |   |    |   |   |   |   |  |
| Standalone|   |   |   | x  |   |   |   |   |  |
| DIY       | x |   |   |    |   | x |   |   |x |

* Cloud Foundry对这些语言的支持是社区提供的。
 Cloud Foundry.com支持Java、Ruby和Node，而ActiveState添加了对Python的支持，Tier 3添加了对.NET的支持，AppFog添加了对PHP的支持。

 免费的AWS Elastic Beanstalk其实还包含了一种IaaS（Infrastructure-as-a-Service）产品。开发人员和管理员可以直接访问应用程序后 面的AWS基础设施，这意味着他们可以修改服务器配置或访问服务端的日志文件。用户负责各种基础设施相关的任务，包括选择（及更新）服务器的操作系统和应 用程序栈。AWS Elastic Beanstalk确实也自动化了很多管理任务，包括通过一条命令重新启动所有的web服务器、通过中心位置访问所有的服务器日志文件以及监控所有节点的性能。

OpenShift也提供对IaaS基础设施的直接访问，这样就可以让用户添加所有想要的语言和框架。其后台使用的是 Red Hat Enterprise Linux 6.2 running on x64 systems.
