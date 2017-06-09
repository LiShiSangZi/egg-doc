# 事物
严格意义上说，这个和egg并没有什么直接关系。只是在使用中也遇到了一些问题。我对于数据库的理解还停留在10年前毕业前后的状态。这些对于不懂数据库的我来说，还是很痛苦的。所以特意写下来，面的下次再踩到坑。
# 遇到的问题
首先遇到一个很简单的问题，放一段代码：
```javascript
const allData = await app.model.Data.findAll();
for (let i = 0; i < allData.length; i++) {
  const data = allData[i];
  data.money = data.money + Math.random() * 100;
  await data.save();
}
```
读取数据库里对应表格的所有的数据，然后修改每一行，并且存回去。这个时候，如果数据过多，比如来个上万行，那么写回去的时间会相当长，因为要一行一行的UPDATE。像这样:
```sql
UPDATE `data` SET `money`=1953.7554045464267 WHERE `id` = 999
```
这个时候，如果恰好其中的这一条数据的money被更新了，数据库还是会很爽快的直接写回去。但是，这个时候，会存在风险的。因为这样的话，计算的数据是不准确的（根据前面可以看到，新的money是根据旧的money计算得来的）。为了保证数据一致性，所以，去翻了一下事务。
# 简单的事务
基于前面的代码，加上简单的事务逻辑：
```javascript
const t = await app.model.transaction();
const allData = await app.model.Data.findAll({
  transaction: t,
});
for (let i = 0; i < allData.length; i++) {
  const data = allData[i];
  data.money = data.money + Math.random() * 100;
  await data.save({
    transaction: t,
  });
}
t.commit();
```
这个时候，期间如果有别的数据库的修改／删除请求，会一直等着，直到这个事务完成，才可以进行操作。那个等着的数据，其实是会面临修改超时的风险的。但是*查询操作是不受影响的*。这样做的好处显而易见，我可以保护我这次操作，保证这次数据库操作的准确性。
最后一行代码
```javascript
t.commit();
```
是用来终结这个事务的（这个说法不完全准确，待会再讨论。）
如果不加上这段代码，那这个事务就一直在。存在多久时间，我也不知道，不知道是不是天荒地老了。。。在这段时间，都是不能对数据进行修改和删除的。如果忘记写那段代码，那只有Ctrl-C终止进程了。
在终止了进程以后，发现，数据库里的数据仍然是旧的，也就是在所有数据UPDATE之前的状态，也就是说在退出进程的时候，数据库里的数据被*回滚*了。想想也正常，这个数据库事务出现了异常了，回滚操作也是很合理的事情。
翻了翻sequelize的文档，介绍了关于transaction的另外一（两）种终止方法：
```javascript
// 第一种
t.rollback();
// 第二种
throw new Error();
```
这两个其实是一个套路，都是选择回滚了数据，然后终止了事务。第一种很和平，第二种还往外扔了个错误。
然后继续往下八，发现了另外一个问题：
*一个事务在终止之前，数据是不会更新的，即使调用了更新的方法，这期间去SELECT，发现数据仍然是旧的。（这是数据库的一种隔离方式，后面再提）*
有种感觉，事务只是在数据库里上了个锁，如果用户选择commit，那就把数据更新了，如果选择回滚，就直接解锁就是了。
# 更多选项
目前八到sequelize的事务有这些选项：
```json
{
  autocommit: true,
  isolationLevel: 'REPEATABLE_READ',
  deferrable: 'NOT DEFERRABLE' // implicit default of postgres
}
```
## autocommit
第一个参数据说是能够自动的commit事务，（而且还是默认的！我了个去！那刚刚为什么我的修改没有被自动commit回去？）。在文档里翻了一下，翻出来这么个内容：
Sequelize支持两种事务：
1. 一种会把commit和rollback的事情交给用户来处理。
2. 另一种会基于*Promise链*来自动rollback或者commit事务。
它的栗子里是这么写的：
```javascript
return sequelize.transaction(function (t) {
  // chain all your queries here. make sure you return them.
  return User.create({
    firstName: 'Abraham',
    lastName: 'Lincoln'
  }, {transaction: t}).then(function (user) {
    return user.setShooter({
      firstName: 'John',
      lastName: 'Boothe'
    }, {transaction: t});
  });
}).then(function (result) {
  // Transaction has been committed
  // result is whatever the result of the promise chain returned to the transaction callback
}).catch(function (err) {
  // Transaction has been rolled back
  // err is whatever rejected the promise chain returned to the transaction callback
});
```
这里在transaction的回调函数里返回了一个Promise链（其实也就是一个Promise对象的实例。），sequelize会根据这个Promise的实例的结果来决定是commit还是rollback。
但是之前我是用async/await的写法来写的，相当于我并没有在那个回调函数里翻回Promise实例。
稍微改造一下之前的代码：
```javascript
await app.model.transaction(async(t) => {
  const allData = await app.model.Data.findAll({
    transaction: t,
  });
  for (let i = 0; i < allData.length; i++) {
    const data = allData[i];
    data.money = data.money + Math.random() * 100;
    await data.save({
      transaction: t,
    });
  }
});
```
这样autocommit这个属性就能工作了。
==========不开心的分割线=========
然而，今天，这个事情又又又反转了。。。今天试了一下在上面的那段代码中把autocommit设成了false，然而，他还是autocommit了。what，what，what。
于是又重新的翻了一遍sequlize的文档，关于上面的那一段话：
找到了一段补充：
The key difference is that the managed transaction uses a callback that expects a promise to be returned to it while the unmanaged transaction returns a promise.
自动commit和非自动commit的事务的区别在于transaction的回调函数里面是否返回了一个Promise。
前面的写法里面用了async的函数，所以铁定返回Promise，所以，就自动变成autocommit了。。。
至于autocommit这个属性，纯粹是对应数据库里的一句语句：
```sql
SET autocommit = 0;
```
感觉在sequlize里面并没有什么大用处。
## isolationLevel
可以配在全局也可以配置在单个事务上。
如果要配置到全局，因为使用的是egg-sequelize，需要配在配置文件里：
```javascript
// config/config.default.js:
exports.sequelize = {
  dialect: 'mysql',
  database: 'test',
  host: '10.0.0.153',
  port: '3306',
  username: 'testuser',
  password: 'testuser',
  isolationLevel: 'REPEATABLE_READ',
}
```
这个选项的值有4种，枚举分别是：
```javascript
app.model.Transaction.ISOLATION_LEVELS.READ_UNCOMMITTED // "READ UNCOMMITTED"
app.model.Transaction.ISOLATION_LEVELS.READ_COMMITTED // "READ COMMITTED"
app.model.Transaction.ISOLATION_LEVELS.REPEATABLE_READ  // "REPEATABLE READ"
app.model.Transaction.ISOLATION_LEVELS.SERIALIZABLE // "SERIALIZABLE"
```
其中*REPEATABLE_READ*是默认值。
READ_UNCOMMITTED: 能读到其他的事务中没有commit的修改。作为一个事务，可以读到*人家*的没有提交的更改。注意，是读人家的人家的人家的。
READ_COMMITTED: 只能读到已经commit的数据。
REPEATABLE_READ: *默认的。*可读，但是只能读到事务开始前的数据。其他外面就算有更改，也不会影响到它这个事务。举个例子，事务A是REPEATABLE_READ的事务，同时还有另外一个事务B在操作。这个时候即使B的修改commit了，在A中还是之前的数据。
SERIALIZABLE: 最暴力的。锁表干活。。。
## deferrable
由于*只支持PostgreSQL*，所以我默默的无视了它。
