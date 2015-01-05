---
layout: post
title: "Migrate financialrecorder from cloudfoundry to heroku"
description: "Migrate financialrecorder from cloudfoundry to heroku"
category: Cloud
tags: [financialrecorder, heroku]
---
{% include JB/setup %}

之前做的financialrecorder记账工具的server部署在cloudfoundry上，由于6月份的时候cloudfoundry通知说要升级到v2版本，看了下v2有啥，擦，免费没有啦！于是就开始寻找新的免费的云平台，转悠转悠找到了Heroku，原因：免费，有mysql和redis的addon。

### ./Preparation
之前的financialrecorder部署后，使用了几次，数据并不多，另外也有excel做了备份，所以这次搬家并没有打算把数据迁移过来，只是把应用重新在Heroku上部署，于是学习了下Heroku的应用创建及部署方式后就挑了良辰吉日准备搬家了。

### ./Heroku addons
首先不得不提的是Heroku addons，感觉就像个大市场一样，各种各式各样的addons摆在那里，对比起cloudfoundry，cloudfoundry则逊色不少，Heroku能吸引如此多的第三方服务提供方与其合作，不得不惊叹，感觉Heroku就像一个商场，而各个addon则是那些三方服务提供商摆卖的各种服务产品：DB, Cache, Mobile, Email, Monitoring等，逛完一遍就更加觉得这个新家不错。也由于之前受到cloudfoundry的service的限制，financialrecorder也只用到了mysql和redis这两种平台提供的services，至于Mobile Notification都是找Google的GCM搭建的，跟平台无关，于是搬家工作主要就是换mysql和redis的服务提供方了，最后选择的addons是：ClearDB MySQL Database Ignite和Redis To Go Nano，按照各自的Document配置好Datasource和connectionFactory后就好了。

### ./Deploy financialrecorder
接下来就准备把app部署上Heroku了，创建app，checkout，commit后push时被卡住了，push的时候一直报connectiontimeout的错，查明原因就是Blocked by GFW，搞不懂为什么连这个也要block，网上找到[解决办法](http://ruby-china.org/topics/10813)就是换个ip解析heroku.com。好了，push上去后应用就自动部署好了（其实就是Heroku加了个Git的钩子，在每次push完后，执行命令去构建package: mvn package -DskipTests，再Jetty restart）。

### ./Test
最后Test的工作必不可少，简单测试后发现，login时总是报密码不对，奇怪了，打出log检查发现密码也是对的，困扰了半天，发现可能是因为用String存储MD5后的byte[]，而String就存在Charset不同的问题，可能是被数据库的Charset不同给坑了，于是换用Blob存储密码，解决，经验教训啊，不要轻易用String来存储本是byte[]的数据。

### ./Thought
完成这次迁移后，感觉更喜欢Heroku了，很多免费的addons，连Mobile Message Push的addon也有，而且连CI都可以包进去(push完后跑完test才部署)，查看log也很方便，其他各种优点有待以后的慢慢发掘。虽然服务器只设在了美国和欧洲，网络没国内的云平台给力，但整体感觉就是高大上啊！
