# [MySQL 8.0.2复制新特性](http://mysqlhighavailability.com/replication-features-in-mysql-8-0-2/)
MySQL 8 正在变得原来越好，而且这也在我们MySQL复制研发团队引起了一阵热潮。我们一直致力于全面提升MySQL复制，通过引入新的和一些有趣的功能。此外，我们还听取了社区的建议和反馈。因此，我们很荣幸能够与你一同见证最新版本（MySQL 8.0.2）的里程碑式的发布，为此我们总结了其中的一些值得注意的变化。跟随我们下面的博客，我们将会分享这些新功能的一些见解。
>MySQL 8 is shaping up quite nicely. And we are having a blast in the MySQL replication team while this is happening. We are continuously improving all-things-replication by introducing new and interesting features. In addition, we have been listening to our community and addressing their feedback. As such, we would like to celebrate with you all the latest development milestone release, MySQL 8.0.2, by summarizing the noteworthy changes that are in it. Follow up blog posts shall provide insights on these new enhancements.
## 我们对MySQL 组复制进行了加强，主要有以下几个方面：

*  不允许对离开组的成员进行更改：每当组成员离开群组，离开的成员将会自动设置super_read_only，这可以防止DBA，用户或路由层/代理中间件/负载均衡端等带来的的意外更改。除了默认离开组复制的成员不能够进行修改以外，也可以从刚加入开始就开始禁止写入，我们也可以在服务器启动时设置super_read_only参数并启动组复制插件。一旦组复制动成功，他会自动调整super_read_only的值。在多主模式下，所有的节点都将不会设置super_read_only参数 ；在单主的模式下，除了主节点以外，其他的节点都会设置super_read_only为ON 。如果很不幸，你的组复制启动失败了的话，super_read_only =1 设置会继续保持，将不能进行任何写入操作。这些最新的变化同样适用于MySQL 5.7.19和MySQL 8.0.2。所有的这些，有很大部分是因为我们听取了社区的反馈(BUG#84728, BUG#84795, BUG#84733)然后进行开发和加强。--在此 感谢Kenny Gryp
>Disallow changes to members that have left the group. Whenever a member leaves the group voluntarily, it will automatically set super_read_only. This prevents accidental changes from the DBA, user, or router/proxy/load balancers. In addition to disallowing changes on members that have left the group by default, it is also possible to protect the writes from the beginning. I.e., it is possible to set super_read_only and start the group replication plugin while the server starts. Once Group Replication starts successfully, it will adjust super_read_only’s value properly. On multi-primary mode, the super_read_only will be unset on all joining members; On single-primary, only the primary will have super_read_only unset.  In the unlikely event that Group Replication fails to start, the super_read_only is preserved, thence no writes are allowed.  This last change is actually in both, MySQL 5.7.19 and MySQL 8.0.2. All of these were highly influenced by feedback we got from our community (BUG#84728, BUG#84795, BUG#84733) – Thank you Kenny Gryp!

* 可以在Performance Schema 中查看更多信息：在Performance Schema现存的表中，对相关的统计信息的可读性进行了加强。“replication_group_members” 和 “replication_group_member_stats” 表也做了相关拓展，现在可以清楚的看到组成员的角色信息，组成员版本和事物计数器（本地/远程）
>More information on Performance Schema tables. More observability enhancements were added to existing performance schema tables. “replication_group_members” and “replication_group_member_stats” tables have been extended and now display information such as members roles, members versions and counters for the different types of transactions (local/remote).

* 通过分配权重来指定主库的选举：用户可以通过指定组成员的权重来控制主库的选举，当现有的主节点退出组复制，权重最高的节点就会被提升为主节点。
>Influence primary election by assigning promotion weights. The user is able to influence the primary election by determining member weights. When the existing primary goes away, the member with the highest weight is the one to be elected on fail-over.

* 流量控制机制加了一些微调项：用户现在可以更精细的调节流量控制组件。可以定义每个成员的最小配额，整个组的最小提交配额，流程控制窗口等等。
>Fine tuning options for the flow control mechanism. Users can now do more fine tuning of the flow control component. They can define the minimum quota per member, minimum commit quota for the entire group, the flow control window, and more!

MySQL 8.0.1 已经在MySQL复制核心框架添加了很多引人注目的功能。而MySQL 8.0.2在此基础上又有很大的提升，主要如下：
* 增强对接收（IO）线程的管理，即使磁盘已满：此功能提高了接收线程和其他线程之间的内部协调效率，减少彼此的争用。对于终端用户来说，这意味着在磁盘变满并且接收线程阻塞的情况下，它不再阻塞监视操作，例如SHOW SLAVE STATUS。它还引入了一个新的线程状态，即接收线程正在等待磁盘空间资源状态。此外，当磁盘已满的时候，且不能通过释放磁盘空间使接收线程继续没有完成的工作时，可以手动强制停掉它，一般情况下不会有什么问题。如果一个event只写了部分，那它会被清掉以确保relay的一致性。当接收线程轮转刷新relay log且需等待磁盘空间释放时，这种情况下可能就要当心了。
>Enhanced management of the receiver (IO) thread even when disk is full. This feature improves the internal coordination between receiver and other threads, removing some contention. To the end user, this means that in the event that the disk becomes full and the receiver thread blocks, it no longer blocks monitoring operations, such as SHOW SLAVE STATUS. A new thread state is also introduced stating that the receiver thread is waiting for disk space. Moreover, when the disk is full and you cannot free disk space up to allow the receiver thread to continue its activity, you can forcefully stop it, generally without side effects. If there is an event that is partially written this is cleared and relay log is left in a consistent state. Some extra care needs to be taken when receiver thread is rotating the relay log and is waiting for disk space to become available.

*  binary log中记录更多的元数据信息：将事物长度添加到全局事务日志事件。这可以对我们未来的优化工作有很大的帮助，而且也提高了binary log的可读性。
>More metadata into the binary log. Added the length of the transaction to the global transaction log event. This enables potential future optimizations to be implemented as well as better observability support into the binary log.

如果你在研究MySQL复制的内部机制与原理，我们将很高兴与你一起分享我们做了一些清理工作，并为我们的基础组件添加了一个有趣的服务：
>If you are into MySQL replication internals, we are happy to let you know that we have done some clean ups and added one interesting service to our components infrastructure:

* 组成员事件可以传播到内部其他组件。通过利用新的基础服务架构，组复制插件现在可以通知服务器中的其他组件关于成员关联的事件。例如，告知某个组成员角色改变导致仲裁节点丢失等。其他的组件可以对这个信息作出反馈，并且用户也可以自己开发组件用来注册和监听这些事件。
>Group membership events are propagated to other components internally. By taking advantage of the new services infrastructure, the group replication plugin can now notify other components in the server about membership related events. For instance, notify that membership has changed and that quorum was lost. Other components can react to this information. Users can write their own components that register and listen to these events.

* 从XCom（标准的Paxos实现，能严格保证正确性）的内部结构中删除节点上的冗余信息。我们在XCom的结构中删除了一些冗余信息，这使它变得更加简单、减少错误信息，更容易监控那些节点加入或者离开集群，同时它会在系统中保留以前的信息。
>Removed redundant information on nodes from internal structures in XCom. This work removed a bit of redundant information in XCom’s structures, making it simpler and less error prone to track which servers have left and rejoined, while there is still stale information in the system about previous views.

* 对XCom核心和新编码风格进行了几项改进：我们已经修复了XCom的几个BUG，重新格式化了代码，使它符合Google的编码准则，如果你恰巧是一个开发人员，并且再看我们Paxos实现的源代码，你会发现改版后的代码将会更加容易阅读和理解。
>Several enhancements to XCom core and new coding style. XCom has had several bug fixes and its code was reformatted to meet the google coding guidelines. If you are a developer and enjoy looking at our Paxos implementation, this will make it easier for you to read and understand the code.

* 移除了一些老旧版本binary log转换的源代码：这个清理工作我们清除了一些老版本MySQL数据库产的的binary logs转化为新版本能够识别的一些代码（现在仅支持MySQL 5.0以及以上版本）。
>Removed cross version conversion code for very old binary log formats. This is a clean up that removes code converting binary logs generated  by very old versions of MySQL to a newer format as understood by newer versions of MySQL (versions 5.0 and higher).

还有一件有意思的事情，我们已经在MySQL 8.0.2中更改了以下复制默认值：
* 复制的元数据信息默认存储在InnoDB系统表中。这将使MySQL复制功能变得更加强大，在复制崩溃并且自动恢复时候能够使用InnoDB事物的特性来保证恢复到指定位置的正确性。此外，新功能还要求将元数据以表的形式存储（比如组复制和多源复制），它与MySQL 8的新的数据字典保持一致。
>Replication metadata is stored in (InnoDB) system tables by default. This makes replication more robust by default when it comes to crashing and recovering automatically the replication positions. Also, newer features require the metadata information to be stored in tables (e.g., group replication, mutli-source replication). It aligns with the overall direction of MySQL 8 and the new data dictionary in it.

* 基于RBR时SLAVE的SQL线程的哈希扫描被默认开启：这也许并不是一个被广泛认同的做法，但是当从库有一些没有主键约束的表的时候性能会有提高。在这种情况下，使用基于RBR的复制时，此更改会最大程度降低性能损失，因为它会减少更新行数，而非像以前那样可能要全表扫描更新（[slave_rows_search_algorithms](https://dev.mysql.com/doc/refman/5.6/en/replication-options-slave.html#option_mysqld_slave-rows-search-algorithms)参数默认TABLE_SCAN,INDEX_SCAN,HASH_SCAN）。
>hash scans for row-based applier are enabled default. Perhaps not a popular knob, but one that helps when slaves do not have tables with primary keys. This change will minimize the performance penalty in that case, when using row-based replication, since it reduces the number of table scans required to update all rows.

* transaction-write-set-extraction参数会默认开启：使用写集提取，为用户启动组复制或在主服务器上使用基于WRITESET的依赖关系对master进行跟踪。
>Write set extraction is enabled by default. With write set extraction, it is one less configuration step for the user to start group replication or use WRITESET based dependency tracking on the master.

* 默认开启Binary log 过期时间：expire-logs-days默认设置为30（30天）
>Binary log expiration is enabled by default. The binary logs are expired after 30 days by default (expire-logs-days=30).

如你所知，我们一直很忙。事实上，[MySQL 8.0.2 Milestone Release](http://mysqlserverteam.com/the-mysql-8-0-2-milestone-release-is-available/)已经发布了。在复制方面，我们非常高兴看到许多有趣的功能被加入进来。
>As you can see, we have been busy. :) In fact, MySQL 8.0.2 is showing a great feature set across the board. On the replication side, we are very happy to see so many interesting features getting released.

接下来将会有专门的博客来介绍说明这些功能。你也可以自己下载进行测试（[下载地址](https://dev.mysql.com/downloads/mysql/8.0.html#downloads)）,我们需要留意的是MySQL 8.0.2还是DMR版本，并没有GA，使用它需要自己承担风险。另外不要忘记，我们欢迎而且很期望得到你们的反馈。您可以通过错误报告，功能报告，复制邮件列表或仅对这个（或后续的）博文发表评论来给予我们反馈。MySQL 8将会越来越好，越来越精彩。
>Stick around as there will be blog posts detailing further the features mentioned above. Actually, go and try them yourself, as you can download a MySQL 8.0.2 packages here. Note that MySQL 8.0.2 is a development milestone release (DMR), thence not declared generally available yet. Use it at your own risk. Don’t forget, feedback is always welcome and highly appreciated. You can send it to us through bug reports, feature reports, the replication mailing list or by just leaving a comment on this (or subsequent) blog posts. MySQL 8 looks rather nice! Exciting!
