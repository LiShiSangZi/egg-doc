#背景
今天，惊讶的发现数据库里有好多重复数据。
找了半天，终于找到了原因。在开发的过程中，都是用egg-bin来启动调试的，egg-bin默认启动的是一个worker。在线上环境，我们采用了egg-cluster起了4个worker来做同样的事情。我们在app.js中启动的时候，做了一些简单的数据初始化的工作，即检查数据是否完成，如果不完整，出发一系列的程序来填充数据。那么当有4个worker的时候，4个worker都会执行这个工作，然后，华丽丽的会出现重复数据。。。
#解决方法
这个其实算是比较初级的问题。之所以没有被发现，纯粹是因为开发用的是一个worker。按照Egg的解决方案，是有Agent Worker来解决这个问题，把一些公共事务放到Agent Worker中。
然而，在agent.js里，只能得到agent的对象，我们需要数据的配置，需要数据库的操作。但是，agent对象里面似乎没有这些东西的引用。这就很尴尬了。。。
我们曾经想过一种解决方案：采用schedule job来执行，schedule job可以定义是否由某个worker还是由所有worker来执行。然而，schedule job好像没有执行的次数限制。。。
##终极方案
这个终极方案算是解决了这个问题，解决方案如下：
```javascript
// agent.js
module.exports = agent => {
  agent.messenger.on('egg-ready', (data) => {
    agent.messenger.sendRandom('init-job', data);
  });
};
```
这里的sendRandom会随机把这个消息发送给某个worker。然后在worker中监听这个事件。
```javascript
// app.js
module.exports = async(app) => {
  app.messenger.once('init-job', data => {
    .....
  });
}
```
这样，就只会在某一个worker中执行，而且只会执行一次。

任务完成！