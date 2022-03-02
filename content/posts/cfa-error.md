---
title: "记一次CFA报错"
date: 2022-03-02
lastmod: 2022-03-02
author: "rimo"

tags: [折腾笔记]
categories: [记录]
hiddenFromHomePage: false
hiddenFromSearch: false

---
# 
**使用环境:**
- Android 11(氧OS 11.0.5.1)
- adguard
- CFA 2.5.4.premium
**使用方式:**
- adguard设置为使用CFA代理,并使用CFA本地DNS监听地址
**问题**
联通网络下出现以下错误
```
[TCP] dial 🎯 境内 (match IPCIDR/*****) to baidu.com:443 error: ***** connect error: all DNS requests failed, first error: Post "https://dns.adguard.com/dns-query": dial tcp4 *****: i/o timeout
```
修改DNS等各种设置都不能修复
并且更换网络至WIFI或其他运营商时原地复活

------
**解决办法**
重新设置一次安卓“私人dns”,改为自动再关闭一次,问题解决

------
猜测是安卓的doh服务冲突,详细不明