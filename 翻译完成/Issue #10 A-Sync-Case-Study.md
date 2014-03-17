数据同步案例分析（A Sync Case Study）
==================
[Issue #10 Syncing Data](http://www.objc.io/issue-10/index.html)，三月 2014
By [Florian Kugler](https://twitter.com/floriankugler)

A while ago I was working together with [Chris](https://twitter.com/chriseidhof) on an enterprise iPad application that was to be deployed in a large youth sports organization. We chose to use Core Data for our persistency needs and built a custom data synchronization solution around it that fit our needs. In the syncing grid explained in [Drew’s article](http://www.objc.io/issue-10/data-synchronization.html), this solution uses the asynchronous client-server approach.

不久之前，我和 [Chris](https://twitter.com/chriseidhof) 一起为一个大型青年运动组织开发企业 iPad 应用。我们选择 Core Data 作为数据持久化工具，并根据需求定制了数据同步的解决方案。根据 [Drew 文章](http://www.objc.io/issue-10/data-synchronization.html)中提到的数据同步分类表格，我们使用的方法是异步 CS 方式。 

In this article, I will lay out the decision and implementation process as a case study for rolling out your own syncing solution. It’s not a perfect or a universally applicable one, but it fit our needs at the time.

本文将对我们决策和实现的过程进行案例分析，以供大家学习如何定制自己的同步方案。我们的最终方案并不是一个完美或者普遍适用的方案，但是现阶段它能够满足我们的需求。

Before we dive into it: if you’re interested in the topic of data syncing solutions, (which you probably are if you’re reading this), you should definitely also head over to [Brent’s blog](http://www.inessential.com/) and follow his Vesper sync series. It’s a great read, following along his thinking while implementing sync for Vesper.

在我们深入研究之前，如果你对数据同步方案感兴趣（既然你在读这篇文章，我觉得这应该是肯定的），我强烈建议你去 [Brent 的博客](http://www.inessential.com/)阅读一下 Verper 应用同步方案的系列文章。跟随 Brent 的思考过程来分析 Vesper 的同步方案实现将会是一次绝妙的阅读旅程。

用例（Use Case）
---------
Most syncing solutions today focus on the problem of syncing a user’s data across multiple personal devices, e.g. [iCloud Core Data sync](http://www.objc.io/issue-10/icloud-core-data.html) or the [Dropbox Datastore API](https://www.dropbox.com/developers/datastore). Our syncing needs were a bit different, though. The app we were building was to be deployed within an organization with approximately 50 devices in total. We needed to sync the data between all those devices, each device belonging to a different staff member, with everybody working off the same data set.

如 [iCloud Core Data 同步方案](http://www.objc.io/issue-10/icloud-core-data.html)或者 [Dropbox 数据存储 API](https://www.dropbox.com/developers/datastore) 中所示，现在大部分的同步解决方案都主要解决在用户的多个设备之间同步数据的问题，不过我们面临的同步需求略有不同。我们的应用将会被部署在组织里的大约 50 个设备中。每个设备属于不同的组织成员，大家都对同一个数据集进行操作，我们需要在这些设备间进行数据同步。

The data itself had a moderately complex structure with approximately a dozen entities and many relationships between them. But we needed to handle quite a bit of data. In production, the number of records would quickly grow into the six figures.

数据本身结构复杂，包含大约一打实体和其间的各种关系。我们需要处理的数据量很大，真实情况下数据记录的量将迅速达到十万量级。

While Wi-Fi access would be available for most of the staff members most of the time, the quality of the network connection was pretty poor. But being able to access and work with the app most of the time wouldn’t have been good enough anyway, so we needed to make sure that people could also interact with the data offline.

即使大部分情况下组织成员都能够连上 Wi-Fi，网络连接的质量其实是相当差的。保证大部分情况下组织成员能够使用应用并访问数据集是非常重要的，所以我们需要实现离线情况下的各种数据操作。

要求（Requirements）
-----------
With this usage scenario in mind, the requirements for our syncing architecture were pretty clear:

将上一小节描述的应用场景搞清楚之后，我们同步方案的需求其实已经相当清楚了：

1\. Each device has to have the whole data set available, with and without an Internet connection.

2\. Due to the mediocre network connection, syncing has to happen with as few requests as possible.

3\. Changes are only accepted if they were made based off the most recent data, because nobody should be able to override somebody else’s prior changes without being aware of them.

1. 无论有没有网络连接，每一台设备都能够访问完整的数据集。
2. 因为网络连接不稳定，数据同步时发起的请求数量要尽可能少。
3. 数据更改必须基于最新的数据，因为任何人都不应该在不知晓其他人修改的情况下覆盖那些改动。


设计（Design）
------------
###API

Due to the nested structure of the data model and potential high latency in the network connection, a traditional REST-style API wouldn’t have been a very good fit. For example, to show a typical dashboard view in the application, several layers of the data hierarchy would have to have been traversed in order to collect all the necessary data: teams, team-player associations, players, screens, and screen items. If we were to query all these entity types separately, we would have ended up with many requests until all the data was up to date.

由于数据模型的嵌套结构和可能出现的高网络延迟，传统的 REST 风格 API 并不是一个好选择。例如为了在应用中显示一个仪表盘视图，必须遍历好几个数据层级来获取所需的所有数据：队伍、队员联盟、队员、单屏显示和单屏显示元素。如果我们分别获取所有的这几类数据，在数据更新完毕之前我们需要发起很多次请求。

Instead, we chose to use something more atomic, with a much higher data-to-request ratio. The client interacts with the sync server via a single API endpoint: /sync.

实际中我们采用了更原子化的操作，在获取数据量不变的情况下发起请求数更少。客户端与服务器仅使用一个 API 接口进行交互：/sync。

In order to achieve this, we needed a custom format of exchanging data between the client and the server that would transport all the necessary information for the sync process and handle it in one request.

为了实现这个方案，我们需要自定义客户端与服务器交互的数据格式，从而在一次数据同步同求中包含所有需要的数据。

###数据格式（Data Format）

The client and the server exchange data via a custom JSON format. The same format is used in both directions – the client talks to the server in the same way the server talks back to the client. A simple example of this format looks like this:

客户端与服务器之间通过自定义的 JSON 格式数据进行交互。无论请求由谁发起，都采用同样的格式，如下是一个简单的示例：

```
{
    "maxRevision": 17382,
    "changeSets: [
        ...
    ]
}
```

On the top level, the JSON data has two keys: maxRevision and changeSets. maxRevision is a simple revision number that unambiguously identifies the revision of the data set currently available on the client that sends this request. The changeSets key holds an array of change set objects that look something like this:

上例中的 JSON 数据顶层含有 maxRevision 和 changeSets 两个 key。maxRevision 是用來唯一标识客户端当前数据版本的版本号，changeSets 则是一个以数据操作集为元素的列表，如下所示：


```
{
    "types": [ "users" ],
    "users": {
        "create": [],
        "update": [ 1013 ],
        "delete": [],
        "attributes": {
            "1013": {
                "first_name": "Florian",
                "last_name": "Kugler",
                "date_of_birth": "1979-09-12 00:00:00.000+00"
                "revision": 355
            }
        }
    }
}
```

The top level, types, key lists all the entity types that are contained in this change set. Each entity type then is described by its own change object, which contains the keys create, update, and delete, which are arrays of record IDs – as well as attributes, which actually holds the new or updated data for each changed record.

顶层的 types 对应这个操作集中涉及的所有数据实体类型。每个类型又会对应一个针对它自己的操作集合，包含创建、更新和删除操作，每个操作对应指定的记录 ID。这些 ID 最后会对应到针对这条记录的哪些属性进行了新建或更新操作。

This data format carries a little bit of legacy cruft from a previously existing web application where this particular structure was beneficial for processing in the client-side framework used at the time. But it serves the purpose of the syncing solution described here equally well.

这套数据结构参考了之前 Web 端使用的数据结构，当时采用的结构有利于原有客户端的数据处理，也同样满足现在的需求。

Let’s have a look at a slightly more complex example. We have entered some new screening data for one of the players on a device, which now should be synced up to the server. The request would look something like this:

接下来我们看一个复杂一点的例子。假设我们为一台设备上其中一名队员添加了新的单屏显示数据，当需要同步到服务器上时，如下是请求中包含的数据结构：

```
{
    "maxRevision": 1000,
    "changeSets": [
        {
            "types": [ "screen_instances", "screen_instance_items" ],
            "screen_instances": {
                "create": [ -10 ],
                "update": [],
                "delete": [],
                "attributes": {
                    "-10": {
                        "screen_id": 749,
                        "date": "2014-02-01 13:15:23.487+01",
                        "comment": ""
                    }
                }
            },
            "screen_instance_items: {
                "create": [ -11, -12 ],
                "update": [],
                "delete": [],
                "attributes": {
                    "-11": {
                        "screen_instance_id": -10,
                        "numeric_value": 2
                    },
                    "-12": {
                        ...
                    }
                }
            }
        }
    ]
}
```

Notice how the records being sent have negative IDs. That’s because they are newly created records. The new screen_instance record has the ID -10, and the screen_instance_items records reference this record by their foreign keys.

注意其中涉及的记录 ID 是负数，这是因为它们是新建的条目。新建的 screen_instance 条目 ID 是 -10，在后面的 screen_instance_items 条目中引用到了这个 ID 作为外键。

Once the server has processed this request (let’s assume there was no conflict or permission problem), it would respond with JSON data like this:

当服务器处理完这个请求后（假设没有冲突或者权限问题），发回给客户端的响应请求中将包含如下 JSON 数据：

```
{
    "maxRevision": 1001,
    "changeSets": [
        {
            "conflict": false,
            "types": [ "screen_instances", "screen_instance_items" ],
            "screen_instances": {
                "create": [ 321 ],
                "update": [],
                "delete": [],
                "attributes": {
                    "321": {
                        "__oldId__": -10
                        "revision": 1001
                        "screen_id": 749,
                        "date": "2014-02-01 13:15:23.487+01",
                        "comment": "",
                    }
                }
            },
            "screen_instance_items: {
                "create": [ 412, 413 ],
                "update": [],
                "delete": [],
                "attributes": {
                    "412": {
                        "__oldId__": -11,
                        "revision": 1001,
                        "screen_instance_id": 321,
                        "numeric_value": 2
                    },
                    "413": {
                        "__oldId__": -12,
                        "revision": 1001,
                        ...
                    }
                }
            }
        }
    ]
}
```

The client sent the request with the revision number 1000, and the server now responds with a revision number, 1001, which is also assigned to all of the newly created records. (The fact that it is only incremented by one tells us that the client’s data set was up to date before this request was issued.)

客户端在请求中包含版本号 1000，而服务器返回版本号 1001，同时将 1001 这个版本赋予所有这次新建成功的记录。（从服务器返回的版本号只增加了 1 我们可以知道客户端的修改是基于最新数据进行的。）

The negative IDs have now been swapped by the sync server for the real ones. To preserve the relations between the records, the negative foreign keys have also been updated accordingly. However, the client can still map the previous temporary IDs to the permanent IDs, because the server sends the temporary IDs back as part of each record’s attributes.

原来的负数 ID 现在已经被服务器用真实的 ID 替换。为了保留记录间的联系，负数外键也被相应更新成了实际的 ID。但是客户端仍然可以获得临时负数 ID 与服务器返回的正数永久性 ID 之间的关联关系，因为服务器将临时负数 ID 作为记录属性的一部分返回了。

If the client’s data set would not have been up to date at the time of the request (for example, the client’s revision number was 995), then the server would respond with multiple change sets to bring the client up to date. The server would send back the change sets necessary to go from 995 to 1000, plus the change set with the new revision number 1001 that represents the changes the client has just sent.

如果客户端的修改不是基于最新的数据（假如客户端的版本号是 995），服务器会返回多个操作集以将客户端数据更新到最新版本。具体来说，服务器会将 995 版本到 1000 版本的更新操作与客户端发送的 1001 版本一起返回。

###解决冲突(Conflict Resolution)

As stated above, in this scenario of many people working off the same data set, nobody should be able to override prior changes without being aware of them. The policy here is that whenever you have not seen the latest changes your colleagues have made to the data, you’re not allowed to override their changes unknowingly.

如前所述，在这个所有人操作同一套数据的应用场景下，任何人都不应该在不知晓其他人修改的情况下覆盖那些改动。我们采取的方案就是只要你没有看到其他人对数据的最新修改，你就不能直接对这些修改进行覆盖。

With the system of revision numbers in place, this policy is very straightforward to implement. Whenever the client commits updated records to the sync server, it includes the revision number of each record in the change set. Since revision numbers are never modified on the client side, they represent the state of the record when the client last talked to the server. The server can now look up the current revision number of the record the client is trying to apply a change to, and block the change if it’s not based on the latest revision.

有了版本号的帮助，这个方案变得很容易实现。客户端发送给服务器的任何修改，都包含有对应数据条目的版本号。因为客户端从来不修改已有版本号，所以版本号反映了客户端上一次与服务器交互的数据情况。服务器就可以根据版本号搜索对应的条目，并屏蔽基于非最新数据的修改。

The beauty of this system is that this data exchange format allows transactional changes. One change set in the JSON data can include several changes to different entity types. On the server side, a transaction is started for this change set, and if any of the change set’s records result in a conflict, the transaction is rolled back and the server marks the whole change set with a conflict flag when sending it back to the client.

这个设计的优雅之处在于采用的数据交互格式允许事务性修改。JSON 数据中的一个操作集可以包含对不同数据实体的多个修改。在服务器端，操作集中所有修改以事务方式进行，如果其中任何操作导致冲突，则之前的操作全部回滚，然后服务器为该操作集加上冲突标识返回给客户端。

A problem that arises on the client side whenever a conflict happens is the question of how to restore the correct state of the data. Since the changes could have been made while the client was offline and only were committed the next day, we’d have to keep an exact transaction log around (and persist it). This allows us to revert any change in case it creates a conflict during syncing.

问题在于客户端在冲突发生后如何将数据恢复到正常状态。因为修改可能在客户端处于离线状态下进行，过了一天才会上传到服务器，所以必须保存一份精确的修改日志并存储起来。这样我们就可以在冲突发生时撤销任何修改。

In our case, we chose a different route: since the server is the ultimate authority for the ‘truth’ of the data, it just sends the correct data back in case a conflict occurs. On the server side, this turned out to be implemented very easily, whereas it would have required a major effort for the client.

我们最终采用了一种不同的方案：因为服务器最终能够决定数据的正确状态，所以只需要在冲突发生时将正确的数据返回即可。服务器端针对此方案的实现非常简单，而客户端则需要做很多工作来保证这一方案的正确。

If a client now, for example, deletes a record it is not allowed to delete, the server will respond with a change set marked with the conflict flag, and this change set contains the data that has been erroneously deleted, including related records to which the delete has cascaded. This way, the client can easily restore the data that has been deleted without keeping track of all transactions itself.

例如客户端删除了一条不允许删除的记录，服务器就会返回一个有冲突标识的操作集，其中包含被错误删除的记录，以及与该记录相关联的其他记录。这样客户端就很容易恢复删除的记录，而不用在本地记录每一次操作。

实现（Implementation）
-----
Now that we have discussed the basics of the syncing concept, let’s have a closer look at the actual implementation.

之前我们已经讨论了数据同步的基本概念，现在来看一下具体实现的细节。

###后端（Backend）

The backend is a very lightweight application built with node.js, and it uses PostgreSQL to store structured data, as well as a Redis key-value store that caches all the change sets representing each database transaction. (In other words, each change set represents the changes that get you from revision number x to x + 1.) These change sets are used to be able to quickly respond with the last few change sets a client is missing when it makes a sync request, rather than having to query all different database tables for records with a revision number greater than x.

后端是使用 node.js 写的轻量级应用，使用 PostgreSQL 存储结构化数据，同时使用 Redis 缓存所有数据库修改事务的操作集（换言之，每个操作集对应从 x 版本到 x+1 版本的全部修改）。在客户端发起同步请求时，服务器可以从缓存的操作集中迅速找到客户端缺失的最新修改并返回，而不用临时去查询数据库。

The implementation details of the backend are beyond the scope of this article. But honestly, there really isn’t too much exciting stuff there. The server simply goes through the change sets it receives, starts a database transaction for each of them, and tries to apply the changes to the database. If a conflict occurs, the transaction gets rolled back and a change set with the true state of the data is constructed. If everything goes smoothly, the server confirms the change with a change set containing the new revision number of the changed records.

后端实现的具体细节超出了本文的范围，但是老实说这些细节中并没有什么激动人心的地方。服务器只是简单地为每一个接收到的操作集发起一个事务，然后尝试将这些操作写入数据库。如果发生冲突，事务就进行回滚，然后构造一个正确状态的数据操作集。如果没有错误发生，服务器将以一个包含最新版本号的操作集确认这次修改。

After processing the changes the client has sent, it checks if the client’s highest revision number is trailing behind the server’s, and, if that’s the case, adds the change sets to the response, which enables the client to catch up.

处理完客户端发送过来的修改后，服务器会检查客户端的最新版本号是否落后于自己的，如果是的话就将上面构造的正确操作集返回给客户端来同步到最新状态。

###Core Data

The client application uses Core Data, so we need to hook into that to catch the changes the user is making and commit them to the sync server behind the scenes. Similarly, we need to process the incoming data from the sync server and merge it with the local data.

在客户端使用了 Core Data，所以我们需要在后台记录用户的每一次修改并提交到服务器。同时我们也需要处理服务器发送过来的数据并与本地数据进行合并。

To achieve this, we use a main queue managed object context for everything UI related (this includes data input by the user), and an independent private queue context for importing data from the sync server.

为了达到以上目的，我们使用了一个主队列管理用户界面相关的对象上下文（包括用户输入的所有数据），另一个独立的私有队列管理服务器发送过来的数据。

Whenever the user makes a change to the data, the main context gets saved and we listen for its save notifications. From the save notification, we extract the inserted, updated, and deleted objects and construct a change set object that then gets added to a queue of change sets that are waiting to be synced with the server. This queue is persisted (the queue itself and the objects it holds implement NSCoding) so that we don’t lose any changes in the case that the app gets killed before it has a chance to talk to the sync server.

当用户修改数据的时候，主上下文会被保存，而我们监听了保存事件的通知。从通知中我们可以获取用户操作中插入、更新和删除的对象数据，从而构造一个操作集加入到队列中等候最终发送给服务器。这个队列是持久化的（队列本身和其中的对象使用 NSCoding 协议），所以即使应用在与服务器同步前退出了我们也不会丢失任何修改。

Once the client can establish a connection to the sync server, it takes all the change set objects from the queue, converts them into the JSON format described above, and sends them off to the server together with its most current revision number.

当客户端与服务器建立好连接之后，就从队列中拿出所有的操作集并转换为上文提及的 JSON 格式，带上当前最新的版本号发送给服务器。

Once the response comes in, the client goes through all change sets it received from the server and updates the local data accordingly in the private queue context. Only if this process has completed successfully and without errors, the client stores the current revision number it received from the server in a Core Data entity reserved especially for this purpose.

当服务器的响应返回时，客户端查看收到的所有操作集，然后更新私有队列中相应的本地数据。只有当这次更新成功完成时，客户端才会将服务器发送过来的最新版本号存储到 Core Data 中指定的数据实体中。

Last but not least, the changes made in the private queue context are now merged into the main context, so that the UI can update accordingly.

最后很重要的一点是私有队列中的修改会合并到主上下文中，所以用户界面会相应更新。

Once all that is complete, we can start over and send the next sync request if something new is in the sync queue.

当以上所有操作完成后，我们就可以接着处理队列中新出现的修改以发起下一次同步请求。

###合并策略（Merge Policy）

We have to safeguard against potential conflicts between the private queue context used to import data from the sync server and the main context. The user could easily make some edits in the main context while data is being imported in the background.

我们必须妥善处理用于存储服务器返回数据的私有队列与主上下文之间的冲突。用户再修改主上下文时很有可能后台正在接收服务器的数据。

Since the data received from the server represents the ‘truth,’ the merge policy is set up so that changes in the persistent store trump in-memory changes when merging from the private to the main context.

因为服务器返回的数据才是绝对正确的，在将私有队列中的数据合并到主上下文的时候，我们的策略就是持久化存储的数据相比内存中的数据更优先采用。

This merge policy could, of course, lead to cases where a background change from the sync server, for example, deletes an object that is currently being edited in the user interface. We can give the UI a chance to react to this change before it happens by sending a custom notification about such events when the private queue has saved, but before the changes are merged into the main context.

当用户修改一个服务器已经更新（比如已经删除）的对象时，这种合并策略会遇到一些问题。当这种修改在私有队列中保存但还未合并到主上下文时可以发送一个自定义的通知，这样用户界面可以针对此作出反应。


###初始数据导入（Initial Data Import）

Since we’re dealing with substantial amounts of data for mobile devices (in the six figures), it would take quite a bit of time to download all the data from the server and import it on an iOS device. Therefore, we’re shipping a recent snapshot of the data set with the app. These snapshots are simply generated by running the Simulator with a special flag that enables the download of all data from the server if it’s not present yet.

由于我们需要为移动设备处理大量的数据条目（十万级），将所有的数据从服务器下载并导入到 iOS 设备中将花费很长时间。因此，我们会在应用中附带一份数据集的最新快照。这些快照使用经过特殊设置的模拟器运行生成，该设置可以在本地数据不是最新的情况下从服务器获取所需的数据。

Then we take the SQLite database generated in this process, and run the following two commands on it:

然后我们对生成的 SQLite 数据库文件运行如下两个命令：

```
sqlite> PRAGMA wal_checkpoint;
sqlite> VACUUM;
```

The first one makes sure that all changes from the write-ahead logging file are transferred to the main .sqlite file, while the second command makes sure that the file is not unnecessarily bloated.

第一条命令确保日志中记录的所有之前的修改同步到主 .sqlite 文件中，第二条命令确保文件不会过大。

Once the app is started the first time, the database is copied from the app bundle to its final location. For more information on this process and other ways to import data into Core Data, see [this article in objc.io #4](http://www.objc.io/issue-4/importing-large-data-sets-into-core-data.html).

当应用第一次启动时，数据文件从应用中被拷贝到最终的位置。想对这个过程以及其他导入数据到 Core Data 中的方法有更多了解，可以参考[这篇文章](http://www.objc.io/issue-4/importing-large-data-sets-into-core-data.html)。

Since the Core Data data model includes a special entity that stores the revision number, the database shipped with the app automatically includes the correct revision number of the data set used to seed the client.

因为 Core Data 数据模型中含有一个存储版本号的特殊数据实体，应用中包含的数据文件会自动将正确的版本号写入该实体中作为初始版本号。

###压缩(Compression)

Since JSON is a pretty verbose data format, it is important to enable gzip compression for the requests to the server. Adding the Accept-Encoding: gzip header to the request allows the server to gzip its response. However, this only enables compression from the server to the client, but not the other way around.

因为 JSON 格式数据体积相对较大，使用 gzip 格式压缩发送给服务器的数据就变得非常重要。在请求中加入「Accept-Encoding: gzip」的头信息能让服务器同样使用 gzip 压缩返回的数据。不过这只对服务器返回的数据有效，并不会在发送时启用压缩。

The client including the Accept-Encoding header only signals to the server that it supports gzip compression and that the server should send the response compressed if the server supports it too. Usually the client doesn’t know at the time of the request if the server supports gzip or not, therefore it cannot send the request body in a compressed form by default.

客户端包含 Accept-Encoding 头信息仅仅是为了告诉服务器自己能够支持 gzip 格式的数据，所以服务器应该在支持 gzip 压缩的情况下返回压缩过的数据。一般情况下客户端并不知道发送请求时服务器本身是否支持压缩，所以默认情况下客户端发送的数据不能进行压缩。

In our case though we control the server and we can make sure that it supports gzip compression. Then we can simply gzip the data that should be sent to the server ourselves and add the Content-Encoding: gzip header, since we know that the server will be able to handle it. See [this NSData category](https://github.com/nicklockwood/GZIP) for an example of gzip compression.

在我们的这个案例中，因为服务器也是我们能够控制的，所以我们可以保证能够支持 gzip 压缩。所以我们可以在发送数据到服务器时加上「Content-Encoding: gzip」头信息并压缩数据，因为我们知道服务器肯定能够处理。可以参考[这个 NSData category](https://github.com/nicklockwood/GZIP) 获取一个 gzip 压缩的案例。

###临时 ID 和永久 ID（Temporary and Permanent IDs）

When creating new records, the client assigns temporary IDs to those records so that it is able to express relations between them when sending them to the server. We simply use negative numbers as temporary IDs, starting with -1 and decreasing on each insert the clients make. The current temporary ID gets persisted in the standard user defaults.

创建新的记录时，客户端会为这些记录分配临时 ID，这样就可以记录它们之间的关系并发送给服务器。我们使用了负数作为临时 ID，从 -1 开始依次递减。当前最新的临时 ID 会持久化存储在标准的用户预设值中。

Because of the way we’re handling temporary IDs, it’s very important that we’re only processing one sync request at a time, and also that we maintain a mapping of the client’s temporary IDs to the real IDs received back from the server.

由于我们采用了这种策略，一次只处理一个同步请求是非常重要的，同时我们也需要维护临时 ID 与服务器返回的真实 ID 之间的映射关系。

Before sending a sync request, we check if we already have received permanent IDs from the server for records that are waiting to be committed in the queue of pending changes. If that’s the case, we swap out those IDs for their real counterparts and also update any foreign keys that might have used those temporary IDs. If we wouldn’t do this or instead send multiple requests in parallel, it could happen that we accidentally create a record multiple times instead of updating an existing one, because we’re sending it to the server multiple times with a temporary ID.

发送同步请求之前，我们会检查是否已经收到对应待提交修改的永久 ID。如果有的话，我们将这些待提交修改的临时 ID 换成永久 ID，并更新相应的外键。如果我们不这样做或者一次发送多个同步请求，可能会导致多次创建同一条记录而不是在已有基础上进行更新，因为我们使用临时 ID 将这条记录多次发送给了服务器。

Since both the private queue context (when importing changes) as well as the main context (when committing changes) have to access this mapping, access to it is wrapped in a serial queue to make it thread-safe.

因为私有队列（在导入修改时）和主上下文一样也需要访问这种映射关系，为了线程安全我们将对其的访问封装在了一个顺序队列中。


结论（Conclusion）
-----------
Building your own syncing solution is not an easy task and probably will take longer than you think. At least, it took a while to iron out all the edge cases of the syncing system described here. But in return you gain a lot of flexibility and control. For example, it would be very easy to use the same backend for a web interface, or to do data analysis on the backend side.

构建自己的数据同步方案并不是一个简单的任务，很可能将花费超出你想象的时间。至少处理本文中提及的各种同步系统边界情况就会占用你很多时间。不过相应的你也能得到灵活性和控制权，比如用同一套后台系统既为 Web 接口提供数据，又在后台做数据分析。

If you’re dealing with less common syncing scenarios, like the use case described here – where we needed to sync the data set between the personal devices of many people – you might not even have a choice except to roll out your own custom solution. And while it might be painful from time to time to wrap your head around all the edge cases, it’s actually a very interesting project to work on.

如果你面对的是一个罕见的同步场景（比如文章提到的例子中，我们需要在很多人的设备之间相互同步），你也许只能自己定制解决方案。也许这将是一个痛苦的过程，因为你需要在脑子里不停地考虑各种边界情况，不过这也意味着这将是一个值得做的有趣项目。
