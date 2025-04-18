---
layout: post
title: 敏捷开发之Scrum
date: 2022-05-27
tags: ["Scrum","日常"]
---

Scrum是一个用于开发和维护产品的框架,是一个增量的,迭代的开发过程,是一种敏捷的手段.Scrum的本质一个是定义研发的流程,第二个是量化开发的工作,从而能达到提高工作效率,且把控产品质量.
<!--more-->

Scrum将开发工期划分为一个一个的sprint,一个sprint多久由Scrum Master和开发团队决定,通常在1-4周为一个sprint,在每一个sprint开始会决定做哪些backlog,sprint结束后会邀请业务团队来验收,并且之后会有一个复盘的会议来复盘本次sprint遇到的问题.

这张图是Scrum框架所包含的内容,主要是三个角色,四个会议,三个工件以及两个面板

![敏捷开发scrum详解敏捷项目管理流程- 程序员资料](20201210133440813.gif)

1.  三个角色

        *   Product Owner,产品负责人,由他来接收用户的需求,并整理出Product Backlog,类似产品经理的角色.
    *   Scrum Master,Scrum教练,在团队中用Scrum的理念来指导团队开发,有点类似PMO的角色.
    *   The Team,这个就不用解释了,开发团队.

2.  四个会议

        *   Sprint Planning Meeting,此会议决定了哪些Product Backlog会进入开发流程,并且由谁来开发.
    *   Daily StandUp Meeting,每日站会,开发团队每天花10-15分钟互相表述下昨天做了哪些,今天将要做哪些的会议.
    *   Sprint Review,每一个Sprint结束,都会邀请业务团队来验收,防止做出来的东西存在理解上的偏差.
    *   Sprint Retrospective,复盘会议,会议上需要大家提些意见,哪些需要改进,哪些需要加强,哪些值得表扬.

3.  三个工件

        *   Product Backlog,由Product Owner收集完需求后产出,它和需求说明书的区别在于,需求说明书是从业务的整体进行阐述,并没有对功能进行可行性的切分,而Product Backlog则会将用户需求进行切分,切分后的结果叫做`user story`.
    *   Sprint Backlog,Sprint Planning Meeting会议产出的结果.
    *   Finished Work,每一个sprint结束后,做出来的成果叫做Finished Work,方便业务团队进行验收.

4.  两个面板

        *   Kanban,kanban将backlog按照进行的程度可以划分为`story,To do,In progress,To verify Done`,当然也可以有其他的画法,主要能够方便管理这些backlog,当backlog的状态发生变化时,只要把它拖到对应的状态栏即可.

        ![Nothing In Kanban Prevents Scrum](image4-1.png)

        *   Burndown/up charts, 燃尽图,它是从工期的角度来判断该sprint的进展情况,可以很好的把握项目进度并能及时进行调整.

        ![一叶知秋--SCRUM实践之Sprint燃尽图实例分析- 知乎](v2-89720b3a4825eddd3fb773a1aeb50eb7_b.jpg)

Scrum与传统的开发流程相比,将用户需求进行了拆分,不在是一大堆难读懂的文字,而是一条一条的可行backlog,量化的好处就在于降低了项目风险,这当然对于Product Owner提出了很高的要求.Scrum最大的一个亮度还在于Sprint Review,一个大的业务需求往往需要做好几个sprint,那怎么保证做出来的东西是用户想要的呢?每一个sprint结束后的Sprint Review,可以让用户及时地看到产出,如果出错,也能及时地修改,不用等到最后才发现错的离谱.