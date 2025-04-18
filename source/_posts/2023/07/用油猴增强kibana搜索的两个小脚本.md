---
layout: post
title: 用油猴增强kibana搜索的两个小脚本
date: 2023-07-24
tags: ["kibana","日常","油猴"]
---

# 1.kibana exclude

搜索kibana日志有时需要排除掉某些phrase,这个脚本选中需要排除的phrase,然后重新搜索日志
<!--more-->

    // ==UserScript==
    // @name         kibana exclude
    // @namespace    http://tampermonkey.net/
    // @version      0.1
    // @description  try to take over the world!
    // @author       You
    // @match        http://kibana
    // @icon         https://pantheon-blog.oss-cn-beijing.aliyuncs.com/20230330140249.png
    // @grant        none
    // @run-at       context-menu
    // ==/UserScript==

    (function() {
        var searchInput = document.getElementsByTagName('input')[0]
        var newQuery = searchInput.value + ' AND NOT "'+ window.getSelection().toString()+'"'
        searchInput.value = newQuery
        searchInput.click()
        document.getElementsByClassName('euiSuperUpdateButton__text')[0].click()
    })();

# 2.kibana search

常看日志的经常需要针对某一个trace做关联搜索,这个时候需要打开一个新的tab,然后在粘贴进去要搜的trace.这个小工具选中要搜索的关键字,可以自动打开一个新的tab.

    // ==UserScript==
    // @name         kibana search
    // @namespace    http://tampermonkey.net/
    // @version      0.1
    // @description  try to take over the world!
    // @author       You
    // @match        http://kibana
    // @icon         https://pantheon-blog.oss-cn-beijing.aliyuncs.com/20230330140249.png
    // @grant        none
    // @run-at       context-menu
    // ==/UserScript==

    (function() {
        var query = window.getSelection().toString()
        var regex = /query:\(language:kuery,query:([^)]+)\)/;
        var url = window.location.href;
        url = url.replace(regex,'query:(language:kuery,query:%22'+query+'%22)')
        window.open(url)

    })();
    