---
layout: post
title: Importing Large Data Sets
category: "4"
date: "2013-09-09 07:00:00"
author: "<a href=\"http://twitter.com/floriankugler\">Florian Kugler</a>"
tags: article
---

{% include links-4.md %}

Importing large data sets into a Core Data application is a common problem. There are several approaches you can take dependent on the nature of the data:

1. Downloading the data from a web server (for example as JSON) and inserting it into Core Data.
2. Downloading a pre-populated Core Data SQLite file from a web server.
3. Shipping a pre-populated Core Data SQLite file in the app bundle.

The latter two options especially are often overlooked as viable options for some use cases. Therefore, we are going to have a closer look at them in this article, but we will also outline how to efficiently import data from a web service into a live application.


## Shipping Pre-Populated SQLite files

Shipping or downloading pre-populated SQLite files is a viable option to seed Core Data with big amounts of data and is much more efficient then creating the database on the client side. If the seed data set consists of static data and can live relatively independently next to potential user-generated data, it might be a use case for this technique.

The Core Data framework is shared between iOS and OS X, therefore we can create an OS X command line tool to generate the SQLite file, even if it should be used in an iOS application.

In our example, (which you can [find on GitHub](https://github.com/objcio/issue-4-importing-and-fetching)), we created a command line tool which takes two files of a [transit data set](http://stg.daten.berlin.de/datensaetze/vbb-fahrplan-2013) for the city of Berlin as input and inserts them into a Core Data SQLite database. The data set consists of roughly 13,000 stop records and 3 million stop-time records.

The most important thing with this technique is to use exactly the same data model for the command line tool as for the client application. If the data model changes over time and you're shipping new seed data sets with application updates, you have to be very careful about managing data model versions. It's usually a good idea to not duplicate the .xcdatamodel file, but to link it into the client applications project from the command line tool project.

Another useful step is to perform a `VACUUM` command on the resulting SQLite file. This will bring down the file size and therefore the size of the app bundle or the database download, dependent on how you ship the file.

Other than that, there is really no magic to the process; as you can see in our [example project](), it's all just simple standard Core Data code. And since generating the SQLite file is not a performance-critical task, you don't even need to go to great lengths of optimizing its performance. If you want to make it fast anyway, the same rules apply as outlined below for [efficiently importing large data sets in a live application][110].


<a name="user-generated-data"> </a>

### User-Generated Data

Often we will have the case where we want to have a large seed data set available, but we also want to store and modify some user-generated data next to it. Again, there are several approaches to this problem.

The first thing to consider is if the user-generated data is really a candidate to be stored with Core Data. If we can store this data just as well e.g. in a plist file, then it's not going to interfere with the seeded Core Data database anyway. 

If we want to store it with Core Data, the next question is whether the use case might require updating the seed data set in the future by shipping an updated pre-populated SQLite file. If this will never happen, we can safely include the user-generated data in the same data model and configuration. However, if we might ever want to ship a new seed database, we have to separate the seed data from the user-generated data.

This can be done by either setting up a second, completely independent Core Data stack with its own data model, or by distributing the data of the same data model between two persistent stores. For this, we would have to create a second [configuration](https://developer.apple.com/library/ios/documentation/cocoa/conceptual/CoreData/Articles/cdMOM.html#//apple_ref/doc/uid/TP40002328-SW3) within the same data model that holds the entities of the user-generated data. When setting up the Core Data stack, we would then instantiate two persistent stores, one with the URL and configuration of the seed database, and the other one with the URL and configuration of the database for the user-generated data.

Using two independent Core Data stacks is the more simple and straightforward solution. If you can get away with it, we strongly recommend this approach. However, if you want to establish relationships between the user-generated data and the seed data, Core Data cannot help you with that. If you include everything in one data model spread out across two persistent stores, you still cannot define relationships between those entities as you would normally do, but you can use Core Data's [fetched properties](https://developer.apple.com/library/ios/documentation/cocoa/conceptual/CoreData/Articles/cdRelationships.html#//apple_ref/doc/uid/TP40001857-SW7) in order to automatically fetch objects from a different store when accessing a certain property.


### SQLite Files in the App Bundle

If we want to ship a pre-populated SQLite file in the application bundle, we have to detect the first launch of a newly updated application and copy the database file out of the bundle into its target directory:

    NSFileManager* fileManager = [NSFileManager defaultManager];
    NSError *error;

    if([fileManager fileExistsAtPath:self.storeURL.path]) {
        NSURL *storeDirectory = [self.storeURL URLByDeletingLastPathComponent];
        NSDirectoryEnumerator *enumerator = [fileManager enumeratorAtURL:storeDirectory
                                              includingPropertiesForKeys:nil
                                                                 options:0
                                                            errorHandler:NULL];
        NSString *storeName = [self.storeURL.lastPathComponent stringByDeletingPathExtension];
        for (NSURL *url in enumerator) {
            if (![url.lastPathComponent hasPrefix:storeName]) continue;
            [fileManager removeItemAtURL:url error:&error];
        }
        // handle error
    }

    NSString* bundleDbPath = [[NSBundle mainBundle] pathForResource:@"seed" ofType:@"sqlite"];
    [fileManager copyItemAtPath:bundleDbPath toPath:self.storeURL.path error:&error];

    
Notice that we're first deleting the previous database files. This is not as straightforward as one may think though, because there can be different auxiliary files (e.g. journaling or write-ahead logging files) next to the main `.sqlite` file. Therefore we have to enumerate over the items in the directory and delete all files that match the store file name without its extension.

However, we also need a way to make sure that we only do this once. An obvious solution would be to delete the seed database from the bundle. However, while this works on the simulator, it will fail as soon as you try this on a real device because of restricted permissions. There are many options to solve this problem though, like setting a key in the user defaults which contains information about the latest seed data version imported:

    NSString* bundleVersion = [infoDictionary objectForKey:(NSString *)kCFBundleVersionKey];
    NSString *seedVersion = [[NSUserDefaults standardUserDefaults] objectForKey@"SeedVersion"];
    if (![seedVersion isEqualToString:bundleVersion]) {
        // Copy the seed database
    }

    // ... after the import succeeded
    NSDictionary *infoDictionary = [NSBundle mainBundle].infoDictionary;
    [[NSUserDefaults standardUserDefaults] setObject:bundleVersion forKey:@"SeedVersion"];

Alternatively for example, we could also copy the existing database file to a path including the seed version and detect its presence to avoid doing the same import twice. There are many practicable solutions which you can choose from, dependent on what makes the most sense for your case.


## Downloading Pre-Populated SQLite Files

If for some reason we don't want to include a seed database in the app bundle (e.g. because it would push the bundle size beyond the cellular download threshold), we can also download it from a web server. The process is exactly the same as soon as we have the database file locally on the device. We need to make sure though, that the server sends a version of the database which is compatible with the data model of the client, if the data model might change across different app versions.

Beyond replacing a file in the app bundle with a download though, this option also opens up possibilities to deal with seeding more dynamic datasets without incurring the performance and energy cost of importing the data dynamically on the client side.

We can run a similar command line importer as we used before on a (OS X) server in order to generate the SQLite file on the fly. Admittedly, the computing resources required for this operation might not permit doing this fully on demand for each request, depending on the size of the data set and the number of request we would have to serve. A viable alternative though is to generate the SQLite file in regular intervals and to serve these readily available files to the clients.

This of course requires some additional logic on the server and the client, in order to provide an API next to the SQLite download which can provide the data to the clients which has changed since the last seed file generation. The whole setup becomes a bit more complex, but it enables easily seeding Core Data with dynamic data sets of arbitrary size without performance problems (other than bandwidth limitations).


## Importing Data from Web Services

Finally, let's have a look at what we have to do in order to live import large amounts of data from a web server that provides the data, for example, in JSON format.

If we are importing different object types with relationships between them, then we will need to import all objects independently first before attempting to resolve the relationships between them. If we could guarantee on the server side that the client receives the objects in the correct order in order to resolve all relationships immediately, we wouldn't have to worry about this. But mostly this will not be the case.

In order to perform the import without affecting user-interface responsiveness, we have to perform the import on a background thread. In a previous issue, Chris wrote about a simple way of [using Core Data in the background](/issue-2/common-background-practices.html). If we do this right, devices with multiple cores can perform the import in the background without affecting  the responsiveness of the user interface. Be aware though that using Core Data concurrently also creates the possibility of conflicts between different managed object contexts. You need to come up with a [policy](http://thermal-core.com/2013/09/07/in-defense-of-core-data-part-I.html) of how to prevent or handle these situations.

In this context, it is important to understand how concurrency works with Core Data. Just because we have set up two managed object contexts on two different threads does not mean that they both get to access the database at the same time. Each request issued from a managed object context will cause a lock on the objects from the context all the way down to the SQLite file. For example, if you trigger a fetch request in a child context of the main context, the main context, the persistent store coordinator, the persistent store, and ultimately the SQLite file will be locked in order to execute this request (although the [lock on the SQLite file will come and go faster](https://developer.apple.com/wwdc/videos/?id=211) than on the stack above). During this time, everybody else in the Core Data stack is stuck waiting for this request to finish.

In the example of mass-importing data on a background context, this means that the save requests of the import will repeatedly lock the persistent store coordinator. During this time, the main context cannot execute, for example, a fetch request to update the user interface, but has to wait for the save request to finish. Since Core Data's API is synchronous, the main thread will be stuck waiting and the responsiveness of the user interface might be affected.

If this is an issue in your use case, you should consider using a separate Core Data stack with its own persistent store coordinator for the background context. Since, in this case, the only shared resource between the background context and the main context is the SQLite file, lock contention will likely be lower than before. Especially when SQLite is operating in [write-ahead logging](http://www.sqlite.org/draft/wal.html) mode, (which it is by default on iOS 7 and OS X 10.9), you get true concurrency even on the SQLite file level. Multiple readers and a single writer are able to access the database at the same time (See [WWDC 2013 session "What's New in Core Data and iCloud"](https://developer.apple.com/wwdc/videos/?id=207)).

Lastly, it's mostly not a good idea to merge every change notification into the main context immediately while mass-importing data. If the user interface reacts to these changes automatically, (e.g. by using a `NSFetchedResultsController`), it will come to a grinding halt quickly. Instead, we can send a custom notification once the whole import is done to give the user interface a chance to reload its data. 

If the use case warrants putting additional effort into this aspect of live-updating the UI during the import, then we can consider filtering the save notifications for certain entity types, grouping them together in batches, or other means of reducing the frequency of interface updates, in order to keep it responsive. However, in most cases, it's not worth the effort, because very frequent updates to the user interface are more confusing then helpful to the user. 

After laying out the general setup and modus operandi of a live import, we'll now have a look at some specific aspects to make it as efficient as possible.


<a name="efficient-importing"> </a>

### Importing Efficiently

Our first recommendation for importing data efficiently is to read [Apple's guide on this subject](https://developer.apple.com/library/ios/documentation/cocoa/conceptual/CoreData/Articles/cdImporting.html). We would also like to highlight a few aspects described in this document, which are often forgotten.

First of all, you should always set the `undoManager` to `nil` on the context that you use for importing. This applies only to OS X though, because on iOS, contexts come without an undo manager by default. Nilling out the `undoManager` property will give a significant performance boost.

Next, accessing relationships between objects in *both directions* creates retain cycles. If you see growing memory usage during the import despite well-placed auto-release pools, watch out for this pitfall in the importer code.  [Here, Apple describes](https://developer.apple.com/library/ios/documentation/cocoa/conceptual/CoreData/Articles/cdMemory.html#//apple_ref/doc/uid/TP40001860-SW3) how to break these cycles using [`refreshObject:mergeChanges:`](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/CoreDataFramework/Classes/NSManagedObjectContext_Class/NSManagedObjectContext.html#//apple_ref/occ/instm/NSManagedObjectContext/refreshObject:mergeChanges:).

When you're importing data which might already exist in the database, you have to implement some kind of find-or-create algorithm to prevent creating duplicates. Executing a fetch request for every single object is vastly inefficient, because every fetch request will force Core Data to go to disk and fetch the data from the store file. However, it's easy to avoid this by importing the data in batches and using the efficient find-or-create algorithm Apple outlines in the above-mentioned guide.

A similar problem often arises when establishing relationships between the newly imported objects. Using a fetch request to get each related object independently is vastly inefficient. There are two possible ways out of this: either we resolve relationships in batches similar to how we imported the objects in the first place, or we cache the objectIDs of the already-imported objects. 

Resolving relationships in batches allows us to greatly reduce the number of fetch requests required by fetching many related objects at once. Don't worry about potentially long predicates like:

    [NSPredicate predicateWithFormat:@"identifier IN %@", identifiersOfRelatedObjects];
    
Resolving a predicate with many identifiers in the `IN (...)` clause is always way more efficient than going to disk for each object independently. 

However, there is also a way to avoid fetch requests altogether (at least if you only need to establish relationships between newly imported objects). If you cache the objectIDs of all imported objects (which is not a lot of data in most cases really), you can use them later to retrieve faults for related objects using `objectWithID:`.

    // after a batch of objects has been imported and saved
    for (MyManagedObject *object in importedObjects) {
        objectIDCache[object.identifier] = object.objectID;
    }
    
    // ... later during resolving relationships 
    NSManagedObjectID objectID = objectIDCache[object.foreignKey];
    MyManagedObject *relatedObject = [context objectWithID:objectId];
    object.toOneRelation = relatedObject;
    
Note that this example assumes that the `identifier` property is unique across all entity types, otherwise we would have to account for duplicate identifiers for different types in the way we cache the object IDs. 


## Conclusion

When you face the challenge of having to import large data sets into Core Data, try to think out of the box first, before doing a live-import of massive amounts of JSON data. Especially if you're in control of the client and the server side, there are often much more efficient ways of solving this problem. But if you have to bite the bullet and do large background imports, make sure to operate as efficiently and as independently from the main thread as possible. 



往Core Data应用中导入大数据集是个很常见的问题。鉴于数据的特点你可以采用以下几种方法：

1. 从web服务器上下载数据(例如JSON数据)，然后插入到Core Data中。
2. 从web服务器上下载预先生成的Core Data SQLite数据库文件。
3. 把一个预先生成好的Core Data SQLite数据库文件传到应用程序包中。

对某些应用场景后两种选择作为可行的方案经常被忽视了。因此，在本文中我们将进一步的了解他们，并总结一下如何高效地把web服务上的数据导入到一个动态的应用中。


## 传输预先生成的SQLite文件

当用大量数据来填充Core Data时，通过传输或下载预先生成的SQLite文件是一个可行的方案，并且比在客户端创建数据更加高效。如果源数据库包含静态数据，并且能够相对独立地与潜在的用户产生的数据共存，这就是该技术的使用场景。

Core Data框架在iOS和OS X间是共用的，因此，我们可以创建OS X上的命令行工具来产生SQLite数据库文件，尽管该文件用在iOS应用中。

在我们的例子中(你可以在这里找到[Github](https://github.com/objcio/issue-4-importing-and-fetching))，我们创建了一个命令行工具，它有2个文件，一个是柏林城市的[数据集](http://stg.daten.berlin.de/datensaetze/vbb-fahrplan-2013)作为输入，把他们插入到Core Data SQLite数据库中。这个数据集包含大约13,000逗留记录及三百万逗留时间记录。

对于该技术最重要的是，命令行工具和客户端应用使用了相同的数据模型。如果数据模型随着时间发生了改变，当你更新应用并传输新的源数据时，你要仔细地管理数据模型的版本。有一个好的建议就是不要复制.xcdatamodel文件，而是从命令行工具项目中把它链接到客户端应用项目。

另一个有用的步骤是在产生的SQLite文件上执行`VACUUM`命令。它会减小文件大小，因此应用程序包也会减小，亦或减小要下载的数据库大小，取决于你如何传输文件。

除了这些，对于该过程真的没有别的方法了；在我们的[案例项目]()中你也看到了，它就是些简单的标准Core Data代码。既然生成SQLite文件不是性能关键的任务，你也没必要花大力气去优化它的性能。如果你想让它更快，后面针对[高效地导入大数据集到动态应用中][110]所作的总结规则同样适用。


<a name="user-generated-data"> </a>

### 用户产生的数据

我们经常会有这样的场景，希望有一个可用的大的源数据集，但是也想能存储和修改一些用户产生的数据。同样，有几种方法来解决这个问题。

首先要考虑的是，用户产生的数据是否真的需要用Core Data来存储。如果我们能把这些数据存储到plist文件中，就不要乱动已建好的Core Data数据库。

如果我们想用Core Data来存储，另一个需要考虑的问题，用例在将来是否需要通过传输更新的预先建好的SQLite文件来更新源数据集。如果这种情况不会发生，我们可以安全地把用户生产的数据包含到相同的数据模型和配置中。然而，如果我们想传输一个新源数据库，我们必须要分离源数据与用户产生的数据。

这个完全可以通过建立第二个完全独立的用着自己的数据模型的Core Data来实现，或者通过在两个持久性存储间分发相同有数据模型的数据。对此，我们需要在同一个数据模型中创建第二个[配置](https://developer.apple.com/library/ios/documentation/cocoa/conceptual/CoreData/Articles/cdMOM.html#//apple_ref/doc/uid/TP40002328-SW3)，它保存用户产生的数据的实体。当配置Core Data栈时，我们将实例化两个持久化存储，一个有URL和源数据库的配置，另一个有URL和用户产生数据的数据库的配置。

使用两个独立的Core Data栈是一种更简单明了的方法。如果你侥幸得知该方法，但是我们强烈推荐它。然而，如果你想在用户产生的数据与源数据间建立关系，Core Data不能帮你实现。如果你把所有的东西包含在一个扩展到2个持久化存储的数据模型中，你依然不能像通常那样在这些实体间定义关系，但是当获取某一特定属性时，你可以用Core Data中的[fetched properties](https://developer.apple.com/library/ios/documentation/cocoa/conceptual/CoreData/Articles/cdRelationships.html#//apple_ref/doc/uid/TP40001857-SW7)从不同的存储中自动获取对象。

### 应用程序中的SQLite文件

如果我们想往应用程序里传输一个预先生成的SQLite文件，我们必须检测出最新更新的应用是第一次打开，并把程序外部的数据库文件复制到目标目录：

    NSFileManager* fileManager = [NSFileManager defaultManager];
    NSError *error;

    if([fileManager fileExistsAtPath:self.storeURL.path]) {
        NSURL *storeDirectory = [self.storeURL URLByDeletingLastPathComponent];
        NSDirectoryEnumerator *enumerator = [fileManager enumeratorAtURL:storeDirectory
                                              includingPropertiesForKeys:nil
                                                                 options:0
                                                            errorHandler:NULL];
        NSString *storeName = [self.storeURL.lastPathComponent stringByDeletingPathExtension];
        for (NSURL *url in enumerator) {
            if (![url.lastPathComponent hasPrefix:storeName]) continue;
            [fileManager removeItemAtURL:url error:&error];
        }
        // handle error
    }

    NSString* bundleDbPath = [[NSBundle mainBundle] pathForResource:@"seed" ofType:@"sqlite"];
    [fileManager copyItemAtPath:bundleDbPath toPath:self.storeURL.path error:&error];


注意我们首先删除之前的数据库文件。这不像你想的那样简单明了，因为可能会存在不同的附属文件（如日志或写前日志文件）与主要的`.sqlite`文件相关。因此我们必须遍历目录里的每一项，删除所有的与存储文件名字匹配不带扩展名的文件。

然而，我们也需要一个方法确保这件事我们只做了一次。一个很明显的方法就是从程序中把源数据库删除。虽然在模拟器上管用，但是因为权限的问题，在真机上会失败。有很多方案来解决这个问题，如在user defaults中设置一个key，它包含了最新导入的数据的版本信息:

    NSString* bundleVersion = [infoDictionary objectForKey:(NSString *)kCFBundleVersionKey];
    NSString *seedVersion = [[NSUserDefaults standardUserDefaults] objectForKey@"SeedVersion"];
    if (![seedVersion isEqualToString:bundleVersion]) {
        // Copy the seed database
    }

    // ... after the import succeeded
    NSDictionary *infoDictionary = [NSBundle mainBundle].infoDictionary;
    [[NSUserDefaults standardUserDefaults] setObject:bundleVersion forKey:@"SeedVersion"];

    或者举个例子，我们也可以复制存在的数据库到一个包含源版本的路径来检测它是否存在, 从而避免做两个相同的导入。有很多可行的方法供你选择，这取决于你的应用场景最重要的是什么。


## 下载预先生成的SQLite文件

如果出于某些原因我们不想把源数据库包放在应用程序中(如，它会导致程序大小超过手机下载的阈值)，我们可以从web服务器上下载。过程与我们把数据库文件放在设备上是一样的。但是得保证，服务器提供的数据库版本要与客户端的数据模型兼容，因为不同的应用版本数据模型可能会改变。

这不仅仅是通过下载来替换应用程序中的一个文件，这个方案也使得填充更多的数据而不导致在客户端动态地导入数据引发的性能与电量损耗成为可能。

为了产生马上可用的SQLite文件，我们可以像前面那样在(OS Ｘ)服务器上运行类似的命令行导入程序。无可否认地，鉴于数据集的大小及要服务的请求数，对每一个请求该操作所需的计算资源可能不允许。一个可行的替代方案是定期地生成SQLite文件，给客户端发送这些现成的文件。

为提供SQLite下载的API，这在服务器端及客户端当然需要额外的逻辑，SQLite的下载可以为自上次源文件生成后已经发生改变的客户端提供数据。整个过程有点复杂，但是可以让你更容易的用任意大小的动态数据来填充Core Data，而且没有性能问题(除了带宽限制)。
