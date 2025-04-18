---
layout: post
title: 本地开发调用k8s东西流量变南北流量
date: 2022-11-12
tags: ["k8s","容器"]
---

# 1.问题背景

我们的发布平台已经都接入到了k8s上面,但是k8s的网络却没有和本地网络打通,这就导致了我们在开发的时候如果需要调用测试环境的机器就需要手动修改feign上面的配置,从网关处调用云上面的机器,也就是所谓的东西流量变为南北流量,但是修改完的配置却很容易被忽略提交到gitlab上面去,所以就引申出来一个问题,办公网络该如何和云环境打通的问题,这里探讨三种办法.
<--more-->

# 2.解决办法

## 2.1 专线

这种方法就是在办公网出口处和云上面的入口处各添加一个网关,中间用专线进行打通,类似于 `ip tunnel`的模式,底层屏蔽掉网络的细节,关系图如下:

![image](20221112172033.png)

## 2.2 负载均衡

还有一种办法就是在客户端的负载均衡处去修改访问的逻辑, 实现维护好该服务到网关路由的映射,负载均衡器在选择路由的时候修改访问的ip,让其变为南北流量,我们的技术架构是基于spring cloud,所以只要去装配一个 `IRule` 的实现就可以了,代码如下:

      public static class LocalDevRule extends NacosRule {

            @Autowired
            private NacosConfigManager nacosConfigManager;

            @Override
            public Server choose(Object key) {
                Server server = super.choose(key);

                try {
                    if (server != null) {
                        Instance instance = ((NacosServer) server).getInstance();
                        DynamicServerListLoadBalancer loadBalancer = (DynamicServerListLoadBalancer) this.getLoadBalancer();
                        String serviceName = loadBalancer.getName();
                        String serviceNameSuffix = serviceName.substring(serviceName.lastIndexOf("-") + 1);
                        //从配置文件中查找对应服务的apisix域名
                        String config = nacosConfigManager.getConfigService().getConfig("apisixUrl.yaml", "DEFAULT_GROUP", 10000);
                        Map<String, Object> dataMap = NacosDataParserHandler.getInstance().parseNacosData(config, "yml");
                        String path = (String) dataMap.get("apisix." + serviceNameSuffix);
                        if (path == null) {
                            LOGGER.warn("请在apisixUrl.yaml添加" + serviceName + "的域名映射配置,配置key为apisix." + serviceNameSuffix);
                            return server;
                        }
                        String url = path.substring(0, path.lastIndexOf(":"));
                        String port = path.substring(path.lastIndexOf(":") + 1);
                        instance.setIp(url);
                        instance.setPort(Integer.parseInt(port));
                        return new NacosServer(instance);
                    }
                } catch (Exception e) {
                    LOGGER.error("choose server occurs error",e);
                    return server;
                }
                return server;

            }

        }

## 2.3 proxy

还有一种办法就是利用proxy来实现修改域名的方法,思路就是A服务访问B服务的时候,走proxy,proxy里面去替换掉B的ip地址为网关的url,proxy起到一个更改目的地址和转发的作用

![image](20221112174815.png)

这里面nacos的作用就在于我们需要实时的去查询访问的ip对应的哪个service,还要将service和网关url的域名维护起来,所以就能够通过ip地址找到对应的网关url,proxy启动后只需要在idea里面配置 `-Dhttp.proxyHost=127.0.0.1 -Dhttp.proxyPort=port` 该参数就可以将feign的流量打到proxy上面.

操作系统的host文件也是类似的作用,只不过host只能将域名映射为ip,且不能带有端口,所以我把proxy这个小工具改了改叫[hostPlus](https://github.com/Pantheooon/hostPlus),放在了github上面,支持ip到ip的映射,ip到域名的映射,并且都可以带端口.

# 3.总结

这个问题其实是我面试这家公司所遇到的一个问题,什么时候东西流量变为南北流量?这其实是一个场景性很强的问题,当然也不是所有的公司都需要将东西流量变为南北流量,当时答的是上家公司一个线上场景,没揣摩好面试官想问的具体含义,但是这个问题研究研究还是挺有意思的.