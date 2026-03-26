作者：哈基咪python  
链接：https://www.zhihu.com/question/369863905/answer/2020108278890243019  
来源：知乎    
[[技术学习上的一些思考]]

当你问出“有没有什么练手项目”的时候，你在招聘市场上的简历就已经死了一半。

别去Github上搜那堆挂着几万Star的“从零用Go实现一个[分布式KV缓存](https://zhida.zhihu.com/search?content_id=774613932&content_type=Answer&match_order=1&q=%E5%88%86%E5%B8%83%E5%BC%8FKV%E7%BC%93%E5%AD%98&zhida_source=entity)”或者“七天手写[微服务框架](https://zhida.zhihu.com/search?content_id=774613932&content_type=Answer&match_order=1&q=%E5%BE%AE%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6&zhida_source=entity)”了。所有的面试官在过去三年里，已经看吐了无数个照抄这些代码的应届生和转码人。你照着敲一遍，跑通了，看着终端里打印出绿色的SUCCESS，觉得自己掌握了高并发和通道通信，实际上你只掌握了Ctrl+C和Ctrl+V。

Go从诞生起就带着极强的工业界功利色彩，它不适合用来玩什么优雅的抽象，也不是给你做课后填空题的。由于它天生为了解决网络IO、高并发和工程部署而存在，如果你想真正把Go打进你的骨髓里，你需要的是“恶意”和“极度的生存焦虑”，而不是“练手”。

我给你指三个方向，不是为了让你拿着去面试吹牛，而是为了让你在深夜被死锁和内存泄漏折磨到砸键盘。

第一，去写一个极度自私的“[生存爬虫集群](https://zhida.zhihu.com/search?content_id=774613932&content_type=Answer&match_order=1&q=%E7%94%9F%E5%AD%98%E7%88%AC%E8%99%AB%E9%9B%86%E7%BE%A4&zhida_source=entity)”。

不要去爬什么豆瓣电影、不要去爬天气预报，那种东西只会让你学会发调三个HTTP请求。现在的行情这么烂，你去写一个针对各大招聘软件的[并发监控系统](https://zhida.zhihu.com/search?content_id=774613932&content_type=Answer&match_order=1&q=%E5%B9%B6%E5%8F%91%E7%9B%91%E6%8E%A7%E7%B3%BB%E7%BB%9F&zhida_source=entity)。

设定一个目标：用goroutine同时监控北上广深所有的Go后端岗位，只要HR一上线更新岗位，你的系统要在秒级抓取数据。

在这个过程中你会遇到什么？你会遇到严苛的反爬虫机制，你会遇到IP被封锁。为了活下去，你不得不去写[代理池](https://zhida.zhihu.com/search?content_id=774613932&content_type=Answer&match_order=1&q=%E4%BB%A3%E7%90%86%E6%B1%A0&zhida_source=entity)；为了不让几十个并发把你的内存卡死，你必须学会用[Context](https://zhida.zhihu.com/search?content_id=774613932&content_type=Answer&match_order=1&q=Context&zhida_source=entity)去优雅地取消超时任务；为了清洗那些结构混乱的垃圾JD，你终于知道正则和字符串处理在海量数据下能有多耗费CPU。当你在后台看到系统自动帮你分析出哪个月薪两万的岗位已经被投递了五千次时，你会对Go的并发模型产生生理上的敬畏，也会对这个残酷的世界多一分认知。

第二，去写一个充满“恶意”的局域网[流量劫持与反向代理](https://zhida.zhihu.com/search?content_id=774613932&content_type=Answer&match_order=1&q=%E6%B5%81%E9%87%8F%E5%8A%AB%E6%8C%81%E4%B8%8E%E5%8F%8D%E5%90%91%E4%BB%A3%E7%90%86&zhida_source=entity)。

别去搞什么简易版Nginx。自己拿Go写一个中间件，挂在你自己或者你室友的网络出口上。

你的目的是什么？不是单纯的转发，而是“精准的破坏”。尝试用channel做一个流量缓冲池，然后写一段逻辑：当识别到室友正在打竞技游戏时，利用goroutine在不阻断连接的情况下，给特定的UDP数据包随机增加200毫秒的延迟，或者每隔固定时间悄悄扔掉1%的包。

干这种缺德事，能逼迫你去扒开TCP/IP四层模型看底层的字节流是怎么流动的；你会深刻体会到什么是阻塞，什么是非阻塞；在处理海量长连接的时候，你会亲眼目睹如果你不小心泄漏了一个goroutine，你的程序是怎么在半小时内把系统的内存吃干抹净然后被OOM killer无情杀死的。这个时候，你再去学pprof性能调优，你看懂的每一根火焰图，都是你为了掩盖作案痕迹而进行的殊死搏斗。

第三，去写一个能把别人的服务器打挂的[压测工具](https://zhida.zhihu.com/search?content_id=774613932&content_type=Answer&match_order=1&q=%E5%8E%8B%E6%B5%8B%E5%B7%A5%E5%85%B7&zhida_source=entity)。

不要用[JMeter](https://zhida.zhihu.com/search?content_id=774613932&content_type=Answer&match_order=1&q=JMeter&zhida_source=entity)，自己写。Go的协程太便宜了，几兆内存就能起一个，这让它天生就是用来做火力覆盖的迫击炮。

写一个分布式的压测引擎，从单机发起十万个长连接去请求别人给的公开API，或者干脆请求你自己写的垃圾后端。然后在压榨的过程中，去记录每一个请求的完整生命周期。

你以为只是开个for循环加go关键字那么简单吗？当QPS飙升到几万的时候，你会发现你的程序自己先崩溃了，因为文件描述符上限了，因为端口被耗尽了，因为网卡的带宽被打满了，或者因为你的channel出现了死锁，整个程序像僵尸一样挂在内存里一动不动。这时候你再去翻Go的源码，去看调度器[GMP](https://zhida.zhihu.com/search?content_id=774613932&content_type=Answer&match_order=1&q=GMP&zhida_source=entity)是怎么工作的，去看网络轮询器[netpoller](https://zhida.zhihu.com/search?content_id=774613932&content_type=Answer&match_order=1&q=netpoller&zhida_source=entity)是怎么把操作系统的epoll包装起来的。

技术从来不是在温室里的“练手”养出来的，而是在为了解决某个极其现实、甚至见不得光的需求时，在无数次宕机和重构中被逼出来的。

关掉知乎，打开你的IDE，建一个名为bullshit的文件夹，选一个真实世界的痛点，把你的焦虑写进main.go里运行起来。当某天你能靠自己写的系统在现实世界里抢到一丝利益，或者搞出一点动静的时候，你就再也不需要任何人给你推荐项目了。