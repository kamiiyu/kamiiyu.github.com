---
layout: post
title: "OPEN SUSE下的RVM小坑"
date: 2013-12-16 15:48
comments: true
categories: 
---
OPEN SUSE13.1默認就安裝了Ruby2,不過還是習慣用RVM這樣的工具管理Ruby。沒想到把系統默認的Ruby更改後，竟然給我發現了YaST的“小祕密”。  
P.S.[YaST](http://zh.wikipedia.org/wiki/YaST)YaST做得很優秀，没Ubuntu的花哨，非常實用。  
![yast](/images/2013-12-16/yast.png)
用RVM安裝Ruby後，上圖的所有項目的子窗口都無法再彈出了。試着在命令行用root身份登陸運行`yast2 --qt`，這樣是運行是沒問題的。
![command-line](/images/2013-12-16/command-line.png)
後來差了一下yast的日誌`/var/log/YaST2/y2log`，發現以下的錯誤：  

    2013-12-16 12:59:28 <3> linux-en59.site(4285) [Y2Ruby] binary/YRuby.cc(callClient):238 cannot require yast:cannot load such file -- fast_gettext at /usr/lib64/ruby/2.0.0/rubygems/core_ext/kernel_

很明顯是Ruby的問題，`fast_gettext`非常可疑。  
`gem search fast_gettext`果然發現的確有fast_gettext這個gem，果斷install。  

    gem install fast_gettext

問題解決！
![command-line2](/images/2013-12-16/command-line2.png)
原來YaST也用到Ruby！  
  
關於gettext可以參考[維基](http://zh.wikipedia.org/wiki/Gettext)，fast_gettext可參考[這裏](https://github.com/grosser/fast_gettext)。  
