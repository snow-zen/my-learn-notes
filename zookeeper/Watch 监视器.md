# Watch 监视器
#ZooKeeper 

ZooKeeper 中所有的读取操作都可以选择设置监视器。watch 是一次性触发，由客户端发送并设置，当设置 watch 的数据发生变化时触发事件，事件会发送到设置的客户端上。

在 watch 的定义中需要考虑三个关键点：

+ **一次性触发**：当数据发生变动时，会向客户端发送一个 watch 事件，之后则不会发送任何监视事件。
+ **发送到客户端**：事件发送到客户端，但是在更改操作的成功响应到达发起更改的客户端之前，事件可能无法发送到客户端。ZooKeeper 提供了排序保证：客户端在第一次看到 watch 事件前，它永远都不会看到设置了 watch 的数据的更改。

> watch 事件会被异步发送给观察者客户端。

+ **设置监视的数据**：znode 节点存在不同的更改方式，将 ZooKeeper 视为两个监视列表会有所帮助：数据监视和子监视。

在 3.6.0 的新功能上：客户端可以在 znode 上设置永久的递归监视，这些监视在触发时不会被删除，并且会以递归方式触发已注册 znode 以及任何子 znode 上的更改。

## 触发语义

可以在三处方法调用位置可以设置 watch 来读取 ZooKeeper 的状态：

1. exits
2. getData
3. getChildren

下面列表说明了手表可以触发的事件以及相关的方法调用：

+ Create event：创建事件，通过调用 exists 启用。
+ Deleted event：删除事件，通过调用 exists、getData 和 getChildren 启用。
+ Changed event：更改事件，通过调用 exists 和 getData 启用。
+ Child event：子事件，通过调用 getChildren 启用。

## 持久 watch

在 3.6.0 的新功能中：上述标准手表存在变体，可以设置一个在触发时不被移除的 watch。

由于该特性的存在，此类 watch 可以触发 NodeCreated、NodeDeleted 和 NodeDataChanged 类型事件。

> 注意：持久递归 watch 不会触发 NodeChildrenChanged 事件，因为它是多余的。

使用 addWatch 方法设置持久 watch，触发语义和保证与标准 watch 相同。关于事件的唯一例外是持久 watch 永远不会触发子事件，因为它们是多余的。

使用 removeWatches 方法和类型为 WatcherType.Any 可以移除持久 watch。

## 移除 watch

可以通过  removeWatches 来删除在 znodes 上注册的 watch。此外，即使没有连接到服务器，客户端也可以通过将本地标志设置为 true 来在本地删除手表。

以下列表说明成功移除 watch 后将触发的事件：

+ Child Remove event：通过调用 getChildren 添加的 watch。
+ Data Remove event：通过调用 exists 和 getData 添加的 watch。
+ Persistent Remove event：通过调用添加持久 watch 添加的 watch。

## 对 watch 的保证

关于 watch，ZooKeeper 维护以下保证：

+ watch 是相对于其他事件、其他 watch 和异步回复进行排序的，ZooKeeper 客户端确保按顺序分派所有内容。
+ 客户端将在看到对应于该 znode 的新数据之前看到它正在观察的 znode 的观察事件。
+ ZooKeeper 的 watch 事件顺序对应于 ZooKeeper 服务看到的更新顺序。

## 需要记住的事

+ 标准 watch 是一次性触发的；如果收到 wach 事件并希望未来再收到有关事件，则必须设置另一个 watch。
+ 因为标准 watch 是一次性触发的，并且在获取事件和发送新请求以获取 watch 之间存在延迟，所以无法可靠地看到 ZooKeeper 中节点发送的每一次更改。准备好处理 znode 在获取事件和再次设置 watch 之间多次更改的情况。
+ 对于给定的 watch 只会被触发一次。例如，如果在同一个文件的 exists 和 getData 上都注册了 watch，则 watch 只会注册一次，并带有文件的删除通知。
+ 当与服务器断开连接时，在重新建立连接之前，将无法获得任何 watch。出于这个原因，会话事件被发送到所有未完成的 watch 处理程序。使用会话事件进入安全模式：在断开连接时，将不会收到事件，因此你的进程应该在该模式下谨慎行事。

