
这篇文章可能会存在较大争议，甚至颠覆一些人的固有思维。


![](https://img2023.cnblogs.com/blog/635610/202501/635610-20250113115542615-1734119066.jpg)


因为关于Oracle的隐藏参数，江湖上一直都有两派对立的观点：


* 1\.不要设置任何隐藏参数，只有当遇到特殊问题时在售后指导下临时使用，在问题解决后还要及时去掉
* 2\.这一系列隐藏参数是众多客户踩出来的最佳实践，上线前必须要设置，才能避免重复踩坑，确保系统运行稳定


两派观点各有各的依据，不针对具体客户场景其实也很难讲谁对谁错。
原厂通常是偏向前者，第三方服务厂商则更多是后者，而且这个最佳实践的参数设置通常还被视作宝贵的技术财产。


但是最难的实际上是客户，客户往往会感到困惑。有时甚至被洗脑，认为某些隐藏参数的设置就是金科玉律。


因为历史10g版本刚推出DRM特性时bug确实比较多，有些极端场景造成的业务中断等影响也比较大，所以给很多从业者留下了些许阴影。最终流传出一个经验，DRM一定要关闭。而DRM的关闭就是需要设置一些隐藏参数，笔者也曾深陷于那个时代，也记录了很多“金科玉律”：



```
--10g RAC关闭DRM特性

 alter system set "_gc_affinity_time"=0 scope=spfile sid='*';
 alter system set "_gc_undo_affinity"=FALSE scope=spfile sid='*';

```

有些系统不能马上重启，于是还有这样的手段经验，先动态设置应急下：



```
--10g RAC可以设置另外2个动态的隐含参数，来达到从”事实上“关闭DRM的目的：

 _gc_affinity_limit=250  
 _gc_affinity_minimum=10485760

```

然后到了11g，关闭DRM的隐藏参数：



```
alter system set "_gc_policy_time"=0 scope=spfile sid='*';
alter system set "_gc_undo_affinity"=false scope=spfile sid='*';

```

最后到了19c，客户形成了惯性，都会这样去设置关闭DRM，尤其是"\_gc\_undo\_affinity"这个参数，很多客户19c的系统中都有设置。
仿佛这就是一个宣言：看吧，实际生产环境还是要设置这类隐藏参数，虽然Oracle官方不建议。


一旦有人说不要随意设置这些隐藏参数，就会被喷，难道DRM不需要设置吗？


终于，让笔者找到了一个非常典型的案例。


最近遇到一个客户咨询问题，说上级机构发通知提到的一个缺陷，看着非常严重，想让我帮忙解读下，给他们一些建议。


通告中让客户觉得有些迷糊的段落如下：



> 根据官方文档说明，该 Bug 在 12\.2 版本以前，若回滚段的 slot wrap\#使用超过最大值0xffffffff 会报ORA\-600\[4187]错误。
> 在 12\.2 版本及以上，RAC环境中，若设置隐藏参数”\_gc\_undo\_affinity"\=FALSE，xids的耗尽速度会快很多倍(数据库会跳过很多 xids)，易于使回滚段的 slot wrap\#使用超过最大值 0xffffffff，则会报 ORA\-1558 错误。
> 目前，没有补丁能够修复该 Bug.


因为客户使用的是19c版本，领导看到这个提示认为19c也有这个bug，且没有补丁能够修复此bug。
看到这里就已经很疑惑了，19c作为目前生产环境的主流版本，都已经出来这么多年了，怎会有这么严重的bug一直不给修复呢？而这个隐藏参数，不就是DRM相关的吗？


带着客户的疑惑我研究了一下，简述下结论：


# 1\.针对12\.2以前的版本，确实曾存在一个bug：


* Bug 19700135 \- ORA\-600 \[4187] when the undo segment wrap\# is close to the max value of 0xffffffff (Doc ID 19700135\.8\)


该问题已经在12\.2及以后版本中解决。


# 2\.针对更新的版本（\[Release 12\.1 to 23]，包含19c），会有一个类似但报错ORA\-1558的问题，但这不是bug：


* ORA\-1558 Happened On RAC Database (Doc ID 3033808\.1\) 
研发部已经明确回复是不要设置\_gc\_undo\_affinity这个参数：
* Do NOT set \_gc\_undo\_affinity\=FALSE



> DEV team confirmed that the issue arises because \_gc\_undo\_affinity\=FALSE is being used. With \_gc\_undo\_affinity\=FALSE, it can exhaust xids many times faster(basically, many xids would be skipped by the DB).


所以，客户只需确认下 \_gc\_undo\_affinity 这个隐藏参数有没有修改过，好在这个客户查询后，发现并没有设置这个隐藏参数。


至于说目前没有补丁能修复该bug，着实有点儿过了，研发已经定位了不算是一个bug，咋修复。。
解决方案也给的很清楚，不要设置“\_gc\_undo\_affinity”这个隐藏参数。


# 3\.隐藏在问题背后的问题


这个问题其实代表性非常强，之所以上级机构会发这个警告，一定是有客户遭遇了这个问题，引发了比较严重的影响，才会通告大家不要重复踩坑，而且还特别提到：



> 由于 DRM 是从 ORACLE 10g开始推出的特性，又因为该特性 bug 很多，以 ORACLE 数据库最佳实践，会建议关闭 DRM 特性相关参数，其中包括设置隐藏参数”\_gc\_undo\_affinity"\=FALSE.
> 因此，建议对 undo 表空间的回滚块的 sot wrap\#使用率进行监控，当使用率达到10%，应该考虑在停机窗口重建 undo 表空间。


这就是问题症结所在，显然是理解错了这个问题。误认为这个参数是必须要设置的，而厂商给出的是不要设置。


其实这个参数就不应该在19c设置为FALSE，首先，19c的DRM已经不会像10g那样问题很多，并不建议关闭。
其次，就算一遭被蛇咬，十年怕井绳，铁了心要关闭DRM，19c也不是这样设置了。


因为隐藏参数都是不被文档记录的，所以很难找到一个依据。


在其他case中知道19c要设置“\_lm\_drm\_disable”\=7来关闭DRM，我特意去问了全球的后台专家，也有这样回复的，在关闭DRM的同时，还禁用了结果缓存和gc锁定：



```
Reduce the drm along with disabling both result cache and gc locking 
- "_lm_drm_diasble"=7;
- "_gc_read_mostly_locking"=false; 
- "result_cache_max_size"=0

```

而且要注意，这个参数设置也可能导致一些其他的问题。
正确的方式是，不应该主动在19c设置关闭DRM，除非你真的遇到了真实问题，在售后SR指导下白纸黑字的要求你关闭。


现在回到最开始的问题，由于设置隐藏参数”\_gc\_undo\_affinity"\=FALSE，导致xids的耗尽速度会快很多倍，容易使回滚段的 slot wrap\#使用超过最大值 0xffffffff，报 ORA\-1558错误的严重“bug”，其实本身就是一个乌龙。


这个隐藏参数，无论何种原因，压根儿就不应该在19c设置为FALSE。


题外话，因为我去问了全球范围的专家如何在19c中正确关闭DRM，还引起了Oracle RAC的PM关注，他非常关切为什么有客户要在19c中关闭DRM，坚持认为关闭DRM的行为只应该存在一些老版本的时代。


所以，永远不要轻易设置Oracle的隐藏参数，哪怕是DRM。
一定认为遇到了问题，必须要临时设置某些隐藏参数，需要提交SR给后台背书。


总之，别在迷信Oracle的隐藏参数，哪怕是DRM，最终反而把问题给搞复杂了。


 本博客参考[milou云加速器cloud](https://www.huabeikeji.com)。转载请注明出处！
