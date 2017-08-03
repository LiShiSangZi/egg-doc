# 无缝重启
在Egg的官方文档中，Egg的运行有两种方式：
1. 使用egg-bin来启动服务。（开发环境）
2. 使用egg-cluster来启动服务。（生产环境）

# 遇到问题
由于在实际的生产环境中，对于服务有要求，因此，如果有任何中断业务的变更，都需要放在晚上。
而且，做变更的时候，开发工程师需要在场值守，防止变更出现意外。
随着时间的推移，大家开始扛不住了。于是，开始思考无缝升级的方案。这样才能晚上不加班，原动力就是这么的直接。
无缝升级，就首先得保证所有的请求不会中断，用户的API访问不会因为服务的重启而报错。

# 开始改造
关于这个问题，Egg官方的回答大致是这样的：
```
这个工作应该在负载均衡端去做，切个流量就搞定的事情，就不用Egg来做了。
```
这个方案对于我们而言，对运维的操作就有要求了，运维需要做的事情大致如下：
1. 首先要保证有多个egg的服务。
2. 开始变更之前，首先通过配置切走其中的一个服务的流量。
3. 重启该服务。
4. 再重复2-3直到所有的服务都重启。
在我们这里，LB和Egg服务并不在一个机器上，整个事情并不能通过一个脚本来搞定。对于运维而言，全是慢慢的微操，而且我们的运维经常手潮，出了问题，那白天升级就要写报告了。
况且，这个方案对于单cluster就没什么办法了。

因此我们考虑的还是基于单个cluster的无缝重启。
要做到这一点，首先得保证每个worker的服务是无状态的。否则，替换的worker并不能完美的取代之前的worker。那就没有意义了。
然后，所谓的完美重启，对于每个worker而言并不是完美的。worker肯定会有一个短暂的时间，是处在无法服务的状态的。所以在这个时候，需要一个替代的worker来帮助重启中的worker干活。
现阶段我们几乎没有放工作在agent上，所以，我们并不需要关心agent的重启，只需要重启worker就行了。
在其他所有的worker都开始工作了，那重启过程就完成了。

# 不中断业务
上面的改造，已经完成了绝大部分，但是还剩下一个问题：不中断业务。前面的方案只能解决一部分这个问题。
举个例子：有一个用户正在跑一个API，某个worker已经接受了这个请求，正在处理，如果这个时候重启服务，那这个用户马上看到一个5XX的错。那这样，其实就已经穿帮了。同理，如果正在跑一个schedule job，这个时候重启一下，那马上这个任务就失败了。
为了解决这个问题，翻了很多文档，找了很多方法，最后只找到一个解决方案，而且也不是很完美：
https://nodejs.org/dist/latest-v8.x/docs/api/cluster.html#cluster_worker_disconnect

关于worker的disconnect方法，官方大致是这么说的：
```
Note that after a server is closed, it will no longer accept new connections, but connections may be accepted by any other listening worker. Existing connections will be allowed to close as usual. When no more connections exist, see server.close(), the IPC channel to the worker will close allowing it to die gracefully.
```
这样，就我们可以先调用disconnect方法，等disconnect完成以后，再杀掉这些进程，重新fork新worker。
看起来很美丽！然而，在实际测试中，发现一个问题：
我们连续不间断的发HTTP请求给服务器的时候，cluster会持续的把任务分配给worker，在这种情况下，即使调用了disconnect，master还是会把工作分配给那个worker。在这种情况下，worker就一直不会重启，直到他能够闲下来。*我想这里的connection应该不是HTTP connection，而是worker和master之间的connection吧。。。*
试了很多方法，都无法完美的解决这个问题。
只能采用一个折衷的方法：
我们在开始跑disconnect的时候，做一个setTimeout，等到一定时间以后（比如10秒钟），就认为服务器已经无法正常自动无缝重启了，这个时候，发个邮件给管理员。让运维通过在LB导流量的方式让这个cluster暂时空闲下来，然后能够重启。
不过，目前我们的生产环境上还没有遇到cluster会持续不间断处理请求的情况，因此这个问题似乎还不是个大问题。

# 触发
解决了所有这些问题以后。突然发现，我们还拉了一个问题：如何触发重启。
一般升级的时候，我们会先通过yum来安装rpm包。这个时候，代码已经准备完毕了，然而，并没有触发。
一开始想的是用一个API触发。然而，egg cluster是躲在LB后面的，并不能精确的控制重启的cluster。所以目前能想到的办法，就只有一个：监听文件。
现在文件挺多的，如果监听所有的文件，势必会有效率问题。经过一番斟酌，我们选择监听package.json文件。因为每次发布都会带着一个版本的变更。所以相对来说比较可靠。

至此，所有的问题都已经解决。由于egg-cluster对cluster有比较好的监听和状态更新，所以，我们基于以上思路写的东西和egg现有的体系无缝兼容。经过一次变更，目前没有出现问题。哈哈。