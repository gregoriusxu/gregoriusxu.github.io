---
layout:     post
title:      cerbot 自动续签https证书
subtitle:   填坑指南
date:       2019-12-18
author:     gregorius
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - 知识点
---

https是一个趋势，http已经成为不安全的代名词，但是有时我们上https也实在是身不由己，用人家的东西，人家怎么要求你当然要按规则办事，不然你就进不了他的门。
上https证书的方式有很多种，如果你是土豪那就直接买呗，简单省事；但是你跟我一样穷屌丝的话你可以考虑cerbot，首先对cerbot的活雷锋精神表示大家赞赏。讲到这里我是不是
要同时感谢一下github(捂脸)。

https和nginx当然是绝配了，安然你用apache,iis,tomcat...等等也是可以的，看个人习惯了。因为我这边只是一个api,所以没有必要杀鸡用牛刀了，nginx的轻量和解耦是无语伦比的。
nginx的配置大家自行搜索一下就知道了，但是要注意在安装证书时必须把http服务关闭，不然申请证书时会以http方式申请。cerbot时必须装的，证书续签指令：

    certbot renew --renew-hook

需要自动定时续签，需要借用crontab -e

    0 3 */7 * * certbot renew --renew-hook "/etc/init.d/nginx reload"

续签时一定要保证https 80端口开通，否则不会续签成功的。

最后提醒大家，规则很重要，我们需要学习和了解，在自己有能力质疑规则之前，必须弄懂规则才能很好的适应环境，顺势而为未尝不是一种好的选择。
