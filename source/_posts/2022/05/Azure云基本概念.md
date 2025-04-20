---
layout: post
title: Azure云基本概念
date: 2022-05-09
tags: ["Azure"]
---

# 1.基本概念

*   subscription

    同一个subscription下的资源,账单是在一起出的,在实际应用中,可以按照部门来创建不同的subscription,也可以根据不同的环境,比如test,stg,pro来划分不同的subscription.
    <!--more-->

    subscription总共有这样几个类型

        1.  Free:免费版,注册后信用卡认证完成,送200刀的额度给体验12个月.
    2.  Pay-as-you-go:按月度收费.
    3.  Enterprise:企业版,会有一定的优惠.
    4.  Student:给学生申请的,100刀体验12个月.

*   resource group

    同一个resource group的资源的生命周期应该大致相同,比如我们新建了一个项目,这个项目需要去申请虚拟机,网络资源,数据库服务等等,他们的生命周期是一致的,随着这个项目的创建而存在,随着项目黄掉而回收.

*   resource

    这个就是指azure上面的service了,比如vm,virtual network,app service...

*   role

# 2.网络

*   subnet

    虚拟网络下面创建出来的子网,后面申请resouce时,如果需要配置网络则可以选择相应的subnet,如果没有选会默认创建一个subnet.

*   IP地址

    ip地址分两种,一种public,一种private,public可以对外访问,private的地址就是在虚拟网络创建出来的地址.

*   Network Security Group,

    用过阿里云或者腾讯云的话,应该知道网络安全组,NSG就是网络安全组,可以控制inboud和outbound的traffic.NSG不仅可以和vm进行一一绑定,也可以和subnet进行绑定.

*   DNS

    Azure的dns提供两个功能,一个是对public domain的管理,也就是DNS Delegation功能,在第三方那买了域名以后,把域名的管理可以放在Azure上面,比如做一些解析,到期续约的操作,具体的操作就是在Azure上开通DNS Delegation功能后,Azure会提供四个name server,把这四个name server贴到第三方的解析服务器处(类似其他云厂商的云解析功能).另外一个就是private dns,这个和虚拟网络可以进行绑定,在此网络下的虚拟机都会被分配一个private domain.

*   负载均衡

    负载均衡我感觉关注两个就可以了,一个是load blancer,还有一个是application gateway,前者是在四层的基础上做了转发,因为不去解析四层以上的协议,所以性能相对来说较好.后者是在七层的基础上做转发,功能更强大.(类似lvs和nginx的区别)

*   网络连接(intersite connectivity)

    假如我们再Azure上面申请了一台vm,ip地址是10.0.1.25,那我们本地在开发的时候如何连接上这台vm呢?这个是网络连接要做的事情,这块总共分三个场景,每个场景又有自己的解决方案:

        1.  虚拟网络之间,比如两个subnet,一个在北美,一个在东亚,他们两个之间该如何互通呢?比较简单的方式就是建立peering,前提是两个subnet的ip地址是不能重叠的(可以把他们简单理解为iptunnel)

        ![image-20220509191129655](image-20220509191129655.png)

        2.  site to site connection

    这种场景是针对on-premises是怎么和azure进行连接的.一种方式是在Azure上面建立vpn,中间用tunnel打通,还有一种方式叫express route.通过网络运营商提供专线服务和Azure打通.

        ![image-20220509191206191](image-20220509191206191.png)

        3.  point to site connection

    这种就是个人笔记本和Azure之间如何打通,也是通过建立tunnel完成

        ![image-20220509191041975](image-20220509191041975.png)

# 3. 存储

*   存储类型

        1.  azure containers,containers是blob的集合,blob是二进制存储,无结构化,可以存储任何东西
    2.  azure files,用于网络间的分享,可以支持SMB协议,在windows server上直接挂载.
    3.  azure tables,用于结构化数据的存储
    4.  azure queues,用于消息的存储

*   副本

    在Azure上创建的存储都会进行冗余来做容灾,副本存在哪就决定了容灾的能力,主要有这样几个策略

        1.  LRS

    本地冗余三份存储,和primary data在同一个region

        2.  ZRS

    跨区域冗余,副本会在多个zone之间进行冗余三份(这里的Z代表zone,比如杭州是一个zone,上海是一个zone)

        3.  GRS

    副本会在多个region之间进行备份,并且会在region pair之间搞一份.(region是比zone大的一个概念,比如中国的华北,华东,整个中国大区也就四个region,region pair是对region本身做的一个高可用,azure会选取另外一个region作为它的备份,当一个region发生了故障可以切换到另外一个region上去)

        4.  RA-GRS

    这个和GRS类似,只不过GRS的副本只有在发生故障转移的时候才能使用,而RA-GRS是可以主动去读取副本的.

        5.  GZRS.

    副本会在Availability zone备份,并且会在region pair上备份.region pair不多说了,介绍下Availability zone,Availability zone是指三个及以上完全独立的数据中心,通常情况下不同的数据中心会去共享网络,电力等资源,Availability zone它们的资源是完完全全独立的,中国大区也只在去年才有了一个Availability zone.

        6.  RA-GZRS

    类似GRS和RA-GRS的关系

*   Access Tiers

    在创建storage的时候可以指定文件存在哪一个tier上,以及使用策略来控制合适将数据放在不同的tier上,比如新的文件,放在hot tier上,用于实时访问,半年后放到cool tier上,用于节省开支,一年后直接放到Archive tier上,归档处理.

    不同的tier价格是不一样的,lantency也是不一样的,hot tier可以实时访问,而Archive tier访问一次则需要数小时之久.

*   同步

    同步主要是指on-premises的数据sync到azure上来,主要的原理是在on-premises上install agent,然后将on-premises server的endpoint注册到auzre上.

    ![image-20220509200958643](image-20220509200958643.png)

# 4.计算引擎

        *   vm

vm这个就很粗暴了,买台回去直接自己装啥自己搞,属于iaas服务.

        *   app service

这个就比vm要灵性多了点，属于pass服务，将操作系统和运行时环境搞定了.

        *   serveless

这个就更厉害了,属于sass了,集成了运行时需要的环境并且可以按照使用来收费(对于后端,可以简单理解为k8s,),正因为功能强大,它的价格也很昂贵,用了基本大厂变小厂,小厂变个体工商户.

    ![image-20220509203346258](image-20220509203346258.png)

    这一块的概念比较通用,当然也有Azure自己独特的地方比如vm里面有Virtual Machine Availability的概念,app service里面又有app service plan的概念,这些东西接触的比较少,所以没有做过多介绍,两种的功能类似都是用来做高可用的,并且能够实现scaling,细节可能有点差异,不做太多介绍.