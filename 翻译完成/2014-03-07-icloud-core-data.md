---
layout: post
title:  "iCloud and Core Data"
category: "10"
date: "2014-03-07 10:00:00"
tags: article
author: "<a href=\"https://twitter.com/mb\">Matthew Bischoff</a> & <a href=\"https://twitter.com/bcapps\">Brian Capps</a>"
---


When Steve Jobs first introduced [iCloud](http://en.wikipedia.org/wiki/ICloud) at WWDC 2011, the promise of seamless syncing seemed too good to be true. And if you tried to implement iCloud [Core Data](http://www.objc.io/issue-4/core-data-overview.html) syncing in [iOS 5](http://adcdownload.apple.com//videos/wwdc_2011__hd/session_303__whats_new_in_core_data_on_ios.m4v) or [iOS 6](http://adcdownload.apple.com//videos/wwdc_2012__hd/session_227__using_icloud_with_core_data.mov), you know very well that it was.

当乔布斯第一次在 WWDC 2011第一次介绍[iCloud](http://en.wikipedia.org/wiki/ICloud), 无缝同步看起来好像很好的，然后你尝试在[iOS 5](http://adcdownload.apple.com//videos/wwdc_2011__hd/session_303__whats_new_in_core_data_on_ios.m4v)或者[iOS 6](http://adcdownload.apple.com//videos/wwdc_2012__hd/session_227__using_icloud_with_core_data.mov)实施它iCloud [Core Data](http://www.objc.io/issue-4/core-data-overview.html),你会知道它的确如想象般的那么好。

Problems with syncing [library-style applications](https://developer.apple.com/library/mac/documentation/General/Conceptual/MOSXAppProgrammingGuide/CoreAppDesign/CoreAppDesign.html#//apple_ref/doc/uid/TP40010543-CH3-SW3) continued as [many](http://www.macworld.com/article/1167742/developers_dish_on_iclouds_challenges.html) [developers](http://blog.caffeine.lu/problems-with-core-data-icloud-storage.html) [abandoned](http://www.jumsoft.com/2013/01/response-to-sync-issues/) iCloud in favor of alternatives like [Simperium](http://simperium.com), [TICoreDataSync](https://github.com/nothirst/TICoreDataSync), and [WasabiSync](http://www.wasabisync.com).

同步库风格应用(译者注:"盒子类型",比如iPhoto)的问题导致[很多](http://www.macworld.com/article/1167742/developers_dish_on_iclouds_challenges.html)[开发者](http://blog.caffeine.lu/problems-with-core-data-icloud-storage.html)[放弃](http://www.jumsoft.com/2013/01/response-to-sync-issues/)支持iCloud,而选择一些其他的方案比如[Simperium](http://simperium.com), [TICoreDataSync](https://github.com/nothirst/TICoreDataSync), 和[WasabiSync](http://www.wasabisync.com)。

In early 2013, after years of struggling with Apple’s opaque and buggy implementation of iCloud Core Data sync, the issues reached a breaking point when developers [called out](http://arstechnica.com/apple/2013/03/frustrated-with-icloud-apples-developer-community-speaks-up-en-masse/) the service’s shortcomings, culminating in a [pointed article](http://www.theverge.com/2013/3/26/4148628/why-doesnt-icloud-just-work) by Ellis Hamburger at The Verge.

2013年初，在经历了与苹果公司iCloud Core Data 同步不透明及buggy实施的多年挣扎后，开发者终于公开批判了服务的重大缺陷并将这个话题推上了[风口浪尖](http://arstechnica.com/apple/2013/03/frustrated-with-icloud-apples-developer-community-speaks-up-en-masse/)。 最终被 Ellis Hamburger 在一篇[尖锐文章](http://www.theverge.com/2013/3/26/4148628/why-doesnt-icloud-just-work)提出。 

## WWDC

It was clear something had to change, and Apple took notice. At WWDC 2013, [Nick Gillett](http://about.me/nickgillett) announced that the Core Data team had spent a year focusing on fixing some of the biggest frustrations with iCloud in iOS 7, promising a vastly improved and simpler implementation for developers. “We’ve significantly reduced the amount of complex code developers have to write,” Nick said on stage at the [“What’s New in Core Data and iCloud”](http://asciiwwdc.com/2013/sessions/207) session. With iOS 7, Apple focused on the speed, reliability, and performance of iCloud, and it shows.

Let’s take a look at what’s changed and how you can implement iCloud Core Data in your iOS 7 app.

很明显这些事情必须改变,同时引起苹果的注意。在WWDC 2013, [Nick Gillett](http://about.me/nickgillett) 宣布 Core Data 团队花了一年时间专注于解决一些iOS 7中iCloud的问题, 承诺大幅改善问题并且让开发者更简单的使用。“我们明显减少开发者编写复杂的代码,” Nick Gillett在[“What’s New in Core Data and iCloud”]舞台上讲到。 在iOS 7中, Apple 专注于iCloud的速度，可靠性，和性能, 以及显示。

让我们看看是什么改变了,如何在 iOS 7应用程序实现Core Data。


## Setup 设置

To set up an iCloud Core Data app, you must first request access to iCloud in your application’s [entitlements](https://developer.apple.com/library/mac/documentation/General/Conceptual/iCloudDesignGuide/Chapters/iCloudFundametals.html), which will give your app the ability to read and write to one or more ubiquity containers. You can do this easily from the new [“Capabilities”](https://developer.apple.com/xcode/) screen in Xcode 5, accessible from your application’s target.

设置一个 iCloud Core Data 应用，你首先需要在你的应用中请求iCloud的[访问权限](https://developer.apple.com/library/mac/documentation/General/Conceptual/iCloudDesignGuide/Chapters/iCloudFundametals.html), 让你的应用程序可以读写一个或多个开放性容器，在Xcode 5中你可以在你应用target的选项卡[“Capabilities”](https://developer.apple.com/xcode/) 中完成着这一切。

Inside a ubiquity container, the Core Data framework will store transaction logs -- records of changes made to your persistent store -- in order to sync data across devices. Core Data uses a technique called [multi-master replication](http://en.wikipedia.org/wiki/Multi-master_replication) to sync data across multiple iOS devices and/or Macs. The persistent store file itself is stored on each device in a Core Data-managed directory called `CoreDataUbiquitySupport`, inside your application sandbox. As a user changes iCloud accounts, the Core Data framework will manage multiple stores within this directory without your application having to observe the [`NSUbiquityIdentityDidChangeNotification`](https://developer.apple.com/library/mac/documentation/cocoa/reference/foundation/classes/nsfilemanager_class/Reference/Reference.html#//apple_ref/doc/uid/20000305-SW81). 

在开放性容器内部, Core Data Framework 将会存储所有的事务日志 -- 记录你的所有持久化的存储 -- 为了跨设备同步数据做准备。 Core Data 使用了一个被称为[多源复制](http://en.wikipedia.org/wiki/Multi-master_replication)的技术来同步 iOS 和 Macs 之间的数据。可持久化存储的数据存在了每一个设备的CoreDataUbiquitySupport 文件夹, 你可以在应用沙盒中找到他。 当用户修改了 iCloud accounts, Core Data framework 会管理多个账户并不需要你自己监听[`NSUbiquityIdentityDidChangeNotification`](https://developer.apple.com/library/mac/documentation/cocoa/reference/foundation/classes/nsfilemanager_class/Reference/Reference.html#//apple_ref/doc/uid/20000305-SW81).

Each transaction log is a `plist` file that keeps track of insertions, deletions, and updates to your entities. These logs are occasionally automatically coalesced by the system in a process known as [baselining](http://mentalfaculty.tumblr.com/post/23788055417/under-the-sheets-with-icloud-and-core-data-seeding).

每一个事务日志都是一个`plist`文件，负责实体的跟踪插入,删除以及更新。这些日志会自动被系统按照一定[基准](http://mentalfaculty.tumblr.com/post/23788055417/under-the-sheets-with-icloud-and-core-data-seeding)合并.

To set up your persistent store for iCloud, there are a few [options](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/CoreDataFramework/Classes/NSPersistentStoreCoordinator_Class/NSPersistentStoreCoordinator.html#//apple_ref/doc/constant_group/Store_Options) you need to be aware of to pass when calling [`addPersistentStoreWithType:configuration:URL:options:error:`](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/CoreDataFramework/Classes/NSPersistentStoreCoordinator_Class/NSPersistentStoreCoordinator.html#//apple_ref/occ/instm/NSPersistentStoreCoordinator/addPersistentStoreWithType:configuration:URL:options:error:) or [`migratePersistentStore:toURL:options:withType:error:`](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/CoreDataFramework/Classes/NSPersistentStoreCoordinator_Class/NSPersistentStoreCoordinator.html#//apple_ref/doc/uid/TP30001180-BBCFDEGA):

在你设置iCloud的持久化存储的时候，调用[`addPersistentStoreWithType:configuration:URL:options:error:`](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/CoreDataFramework/Classes/NSPersistentStoreCoordinator_Class/NSPersistentStoreCoordinator.html#//apple_ref/occ/instm/NSPersistentStoreCoordinator/addPersistentStoreWithType:configuration:URL:options:error:)或者 [`migratePersistentStore:toURL:options:withType:error:`](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/CoreDataFramework/Classes/NSPersistentStoreCoordinator_Class/NSPersistentStoreCoordinator.html#//apple_ref/doc/uid/TP30001180-BBCFDEGA)的时候注意设置一些[可选项](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/CoreDataFramework/Classes/NSPersistentStoreCoordinator_Class/NSPersistentStoreCoordinator.html#//apple_ref/doc/constant_group/Store_Options):

- `NSPersistentStoreUbiquitousContentNameKey` (`NSString`)  
Specifies the name of the store in iCloud (e.g. `@"MyAppStore"`)
给 iCloud 存储空间设置一个名字（例如 @“MyAppStore”）

- `NSPersistentStoreUbiquitousContentURLKey` (`NSString`, optional in iOS 7)  
Specifies the subdirectory path for the transaction logs (e.g. `@"Logs"`)
给事务日志指定一个二级目录(例如 @"Logs")

- `NSPersistentStoreUbiquitousPeerTokenOption` (`NSString`, optional)  
A per-application salt to allow multiple apps on the same device to share a Core Data store integrated with iCloud (e.g. `@"d70548e8a24c11e3bbec425861b86ab6"`)

每个程序设置一个盐，为了让不同应用可以在同一个设备集成iCloud后分享Core Data数据 (比如`@"d70548e8a24c11e3bbec425861b86ab6"`)


- `NSPersistentStoreRemoveUbiquitousMetadataOption` (`NSNumber` (Boolean), optional)  
Used whenever you need to back up or migrate an iCloud store to strip the iCloud metadata (e.g. `@YES`)
当你需要备份或迁移iCloud的元数据(例如 `@YES`)


- `NSPersistentStoreUbiquitousContainerIdentifierKey` (`NSString`)  
Specifies a container if your app has multiple ubiquity container identifiers in its entitlements (e.g. `@"com.company.MyApp.anothercontainer"`)

指定一个容器，如果你的应用有多个容器定义在entitlements 中(例如 `@"com.company.MyApp.anothercontainer"`)


- `NSPersistentStoreRebuildFromUbiquitousContentOption` (`NSNumber` (Boolean), optional)  
Tells Core Data to erase the local store file and rebuild it from the iCloud data (e.g. `@YES`)

告诉 Core Data 抹除本地存储数据并且在iCoud中重建数据(例如 `@YES`)


The only required option for an iOS 7-only application is the content name key, which lets Core Data know where to put its logs and metadata. As of iOS 7, the string value that you pass for `NSPersistentStoreUbiquitousContentNameKey` may not contain periods. If your application already uses Core Data for persistence but you haven't implemented iCloud syncing, simply adding the content name key will prepare your store for iCloud, whether or not there is an active iCloud account.

只支持 iOS 7 的应用的唯一必填选项是 ContentNameKey, 为了让 Core Data 知道把日志和元数据放在哪里。在iOS 7中, 你传入 NSPersistentStoreUbiquitousContentNameKey 的字符串值也许不包含'.'。 如果你的应用已经在使用 Core Data 去存储持久化数据，但是没有实现 iCloud 同步, 你只需要一个简单加入 content name key就可以为iCloud做好准备, 无需关注有没有激活的iCloud账户。

Setting up a managed object context for your application is as simple as allocating an instance of `NSManagedObjectContext` and telling it about your persistent store, as well as including a merge policy. Apple recommends `NSMergeByPropertyObjectTrumpMergePolicy`, which will merge conflicts, giving priority to in-memory changes over the changes on disk.

为你的应用设置一个管理对象上下文简单只需要设置一个 `NSManagedObjectContext` 实例并且告诉你的持久化存储, 并且包含一个合并政策。Apple建议使用 `NSMergeByPropertyObjectTrumpMergePolicy` ,他会合并冲突，并且在磁盘空间之上优先考虑内存变化。

While Apple hasn’t released official sample code for iCloud Core Data in iOS 7, an Apple engineer on the Core Data team provided this basic template [on the Developer Forums](https://devforums.apple.com/message/828503#828503). We’ve edited it slightly for clarity:

而在 iOS 7 中 Apple 还没有官方发布的 iCloud Core Data 的示例代码, Apple 的 Core Dat a工程师团队在[开发者论坛](https://devforums.apple.com/message/828503#828503)提供了这个模板。我们稍微编辑后让他更清晰:
    #pragma mark - Notification Observers
    - (void)registerForiCloudNotifications {
        NSNotificationCenter *notificationCenter = [NSNotificationCenter defaultCenter];
    	
        [notificationCenter addObserver:self 
                               selector:@selector(storesWillChange:) 
                                   name:NSPersistentStoreCoordinatorStoresWillChangeNotification 
                                 object:self.persistentStoreCoordinator];
    
        [notificationCenter addObserver:self 
                               selector:@selector(storesDidChange:) 
                                   name:NSPersistentStoreCoordinatorStoresDidChangeNotification 
                                 object:self.persistentStoreCoordinator];
    
        [notificationCenter addObserver:self 
                               selector:@selector(persistentStoreDidImportUbiquitousContentChanges:) 
                                   name:NSPersistentStoreDidImportUbiquitousContentChangesNotification 
                                 object:self.persistentStoreCoordinator];
    }
    
    # pragma mark - iCloud Support
    
    /// Use these options in your call to -addPersistentStore:
    - (NSDictionary *)iCloudPersistentStoreOptions {
        return @{NSPersistentStoreUbiquitousContentNameKey: @"MyAppStore"};
    }
    
    - (void) persistentStoreDidImportUbiquitousContentChanges:(NSNotification *)notification {
        NSManagedObjectContext *context = self.managedObjectContext;
    	
        [context performBlock:^{
            [context mergeChangesFromContextDidSaveNotification:changeNotification];
        }];
    }
    
    - (void)storesWillChange:(NSNotification *)notification {
        NSManagedObjectContext *context = self.managedObjectContext;
    	
        [context performBlockAndWait:^{
            NSError *error;
    		
            if ([context hasChanges]) {
                BOOL success = [context save:&error];
                
                if (!success && error) {
                    // 执行错误处理
                    NSLog(@"%@",[error localizedDescription]);
                }
            }
            
            [context reset];
        }];
    
        // 刷新界面
    }
    
    - (void)storesDidChange:(NSNotification *)notification {
        // 刷新界面
    }

### Asynchronous Persistent Store Setup 异步持久化设置

In iOS 7, calling `addPersistentStoreWithType:configuration:URL:options:error:` with iCloud options now returns a store almost immediately.[^1] It does this by first setting up an internal 'fallback' store, which is a local store that serves as a placeholder, while the iCloud store is being asynchronously built from transaction logs and ubiquitous metadata. Changes made in the fallback store will be migrated to the iCloud store when it’s added to the coordinator. `Using local storage: 1` will be logged to the console when the fallback store is set up, and after the iCloud store is fully set up you should see `Using local storage: 0`. This means that the store is iCloud enabled, and you’ll begin seeing imports of content from iCloud via the `NSPersistentStoreDidImportUbiquitousContentChangesNotification`.

在 iOS 7 中, 加入iCloud的参数并调用`addPersistentStoreWithType:configuration:URL:options:error:` 它几乎瞬间返回数据。[^1] 其中内部设置了一个‘回滚’数据, 利用本地存储作为一个占位符, 而 iCloud 是由异步的事务日志和元数据构成。当它被添加到 coordinator时更改后的回滚数据将被迁移到 iCloud 中。控制台将会打印`Using local storage: 1` 在回滚存储设置开始后, 当 iCloud 完全设置完后你会看到‘Using local storage: 0’。 这句话的意思是iCloud存储已经启用。你将开始通过监听`NSPersistentStoreDidImportUbiquitousContentChangesNotification`看到来自 iCloud 的内容。

If your application is interested in when this transition between stores occurs, observe the new `NSPersistentStoreCoordinatorStoresWillChangeNotification` and/or `NSPersistentStoreCoordinatorStoresDidChangeNotification` (scoped to your coordinator in order to filter out internal notification noise) and inspect the `NSPersistentStoreUbiquitousTransitionTypeKey` value in its `userInfo` dictionary. The value will be an NSNumber boxing an `enum` value of type [`NSPersistentStoreUbiquitousTransitionType`](https://developer.apple.com/library/ios/documentation/cocoa/Reference/CoreDataFramework/Classes/NSPersistentStoreCoordinator_Class/NSPersistentStoreCoordinator.html#//apple_ref/c/tdef/NSPersistentStoreUbiquitousTransitionType), which will be `NSPersistentStoreUbiquitousTransitionTypeInitialImportCompleted` when this transition has occurred.

如果你的应用关注存储迁移，需要监听`NSPersistentStoreCoordinatorStoresWillChangeNotification` 以及/或者`NSPersistentStoreCoordinatorStoresDidChangeNotification`(将这些通知关联到你的coordinator, 这样就可以过滤其他和你无关的通知) 并且在`userInfo 中检查`NSPersistentStoreUbiquitousTransitionTypeKey` 的值， 这个数值应该会在`NSPersistentStoreUbiquitousTransitionTypeInitialImportCompleted`发生改变的时候对NSNumber 对枚举数值的装箱,[`NSPersistentStoreUbiquitousTransitionType`](https://developer.apple.com/library/ios/documentation/cocoa/Reference/CoreDataFramework/Classes/NSPersistentStoreCoordinator_Class/NSPersistentStoreCoordinator.html#//apple_ref/c/tdef/NSPersistentStoreUbiquitousTransitionType)

## Edge Cases 边缘情况

### Churn

One of the worst problems with testing iCloud on iOS 5 and 6 was when heavily used accounts would encounter 'churn' and become unusable. Syncing would completely stop, and even removing all ubiquitous data wouldn’t make it work. At [Lickability](http://lickability.com), we affectionately dubbed this state “f\*\*\*ing the bucket.”

最严重的一个问题测试 iCloud iOS 5和6时大量使用账户将遇到“混淆”,无法使用。同步将完全停止, 甚至删除开放性数据不会使其工作。在[Lickability](http://lickability.com),我们亲切地称为这种状态“f * \ \ * \ * ing bucket。”

In iOS 7, there is a system-supported way to truly remove all ubiquitous content: `+removeUbiquitousContentAndPersistentStoreAtURL:options:error:`. This method is great for testing, and may even be relevant for your application if the user gets into an inconsistent state and needs to remove all content and start over. There are a few caveats, though. First, this method is synchronous. Even while performing network operations and possibly taking a significant length of time, this method will not return until it is finished. Second, there should be absolutely no persistent store coordinators active when performing this operation. There are serious issues that can put your app into an unrecoverable state, and the official guidance is that all active persistent store coordinators should be completely deallocated beforehand.

在iOS 7中,系统提供了一个方法去移除全部的开放性存储内容: `+removeUbiquitousContentAndPersistentStoreAtURL:options:error:`， 这个方法对测试很有帮助，在你应用中当你用户进入了一个不正常的状态需要删除所有数据重新来过的时候使用它。不过,需要指出的是。首先,这种方法是同步的。很可能长时间执行网络操作, 这种方法不会返回任何值直到完成为止。第二,绝对不能在有持久性存储coordinators活跃时执行此操作。这样存在严重问题,你的应用程序可能进入一个不可恢复的状态,而且官方指导指出所有活跃的持久性存储coordinators应事先完全收回。


### Account Changes 账户修改

In iOS 5, when a user switched iCloud accounts or disabled iCloud, the data in the `NSPersistentStoreCoordinator` would completely vanish without letting the application know. In fact, the only way to check if an account had changed was to call `URLForUbiquityContainerIdentifier` on `NSFileManager` -- a method that can set up the ubiquitous content folder as a side effect, and take seconds to return. In iOS 6, this was remedied with the introduction of `ubiquityIdentityToken` and its corresponding `NSUbiquityIdentityDidChangeNotification`, which is posted when there is a change to the ubiquity identity. This effectively notifies the app of account changes.

iOS5 系统中，用户在切换 iCloud 账户或者禁用账户时，`NSPersistentStoreCoordinator`中的数据会在程序未知的情况下消失。事实上只有一种方法可以在改变账户时制止这种情况的发生。那就是在`NSFileManager`中调用`URLForUbiquityContainerIdentifier`。这个方法可以创建一个内容副本，并且迅速返回。在 iOS 6，这种情况随着引进`ubiquityIdentityToken`和相应的`NSUbiquityIdentityDidChangeNotification`之后迎刃而解，当身份进行转换的时候便会进行通知确认。这就可以对应用账户的变更进行有效的确认并及时的发出提示。

In iOS 7, however, this transition is even simpler. Account changes are handled by the Core Data framework, so as long as your application responds appropriately to both `NSPersistentStoreCoordinatorStoresWillChangeNotification` and `NSPersistentStoreCoordinatorStoresDidChangeNotification`, it will seamlessly transition when the user’s account changes. Inspecting the `NSPersistentStoreUbiquitousTransitionType`key in the `userInfo` dictionary of this notification will give more detail as to the type of transition.

然而，iOS7 中这种转换的情况就变得简单许多，Core Data framework 操纵账户的切换，因此只要你的程序能够正常响应`NSPersistentStoreCoordinatorStoresWillChangeNotification`和`NSPersistentStoreCoordinatorStoresDidChangeNotification`便可以在切换账户的时候流畅的更换信息，检查`userInfo`的字典中“NSPersistentStoreUbiquitousTransitionType 'key将对过渡类型提供更多的细节。

The framework will manage one persistent store file per account inside the application sandbox, so if the user comes back to a previous account later, his or her data will still be available as it was left. Core Data now also manages the cleanup of these files when the user’s device is running low on disk space.

在应用沙箱中框架只能在每个账户中管理独立管理持久化存储，所以这就意味着如果用户回到之前的账户，它的数据就会仍然可用。核心数据依旧在磁盘空间低位运行的情况下对这些文件的清理进行管理。


### iCloud On / Off Switch iCloud的启用与停用

Implementing a switch to enable or disable iCloud in your app is also much easier in iOS 7, although it probably isn’t necessary for most applications. Because the API now automatically creates a separate file structure when iCloud options are passed to the `NSPersistentStore` upon creation, we can have the same store URL and many of the same options between both local and iCloud stores. This means that switching from an iCloud store to a local store can be done by migrating the iCloud persistent store to the same URL with the same options, plus the `NSPersistentStoreRemoveUbiquitousMetadataOption`. This option will disassociate the ubiquitous metadata from the store, and is specifically designed for these kinds of migration or copying scenarios. Here's a sample:

在 iOS 7 中实现应用中一个开关用来切换启用关闭 iCloud 变的非常容易， 虽然对大部分应用来说这个功能不是很需要，因为 API 现在已经支持在创建`NSPersistentStore`时候如果加入 iCloud 选项,那么将自动建立一个独立的文件结构,这意味着从一个 iCloud 存储到一个本地存储可以通过迁移 iCloud 持久存储到同一个URL相同的选项加上“NSPersistentStoreRemoveUbiquitousMetadataOption”。这个选项将把存储中的ubiquitous的元数据进行分离,并专门为这些类型创建了迁移或者复制的场景。下面是一个示例:

    - (void)migrateiCloudStoreToLocalStore {
        // 假设你只有一个存储单元
        NSPersistentStore *store = [[_coordinator persistentStores] firstObject]; 
        
        NSMutableDictionary *localStoreOptions = [[self storeOptions] mutableCopy];
        [localStoreOptions setObject:@YES forKey:NSPersistentStoreRemoveUbiquitousMetadataOption];
        
        NSPersistentStore *newStore =  [_coordinator migratePersistentStore:store 
                                                                      toURL:[self storeURL] 
                                                                    options:localStoreOptions 
                                                                   withType:NSSQLiteStoreType error:nil];
        
        [self reloadStore:newStore];
    }
    
    - (void)reloadStore:(NSPersistentStore *)store {
        if (store) {
            [_coordinator removePersistentStore:store error:nil];
        }
    
        [_coordinator addPersistentStoreWithType:NSSQLiteStoreType 
                                   configuration:nil 
                                             URL:[self storeURL] 
                                         options:[self storeOptions] 
                                           error:nil];
    }

Switching from a local store back to iCloud is just as easy; simply migrate with iCloud-enabled options, and add a persistent store with same options to the coordinator.

切换一个本地存储到iCloud 存储是一个非常容易的事情， 简单到只需要iCloud选项，并且把拥有相同选项可持久存储加入到coordinator中。

### External File References 外部文件的引用

External file references is a feature introduced for Core Data in iOS 5 that allows large binary data properties to be automatically stored outside the SQLite database on the file system. In our testing, when this occurs, iCloud does not always know how to resolve the relationship and can throw exceptions. If you plan to use iCloud syncing, consider unchecking this box in your iCloud entities:
  
![Core Data Modeler Checkbox](http://cloud.mttb.me/UBrx/image.png)

外部文件的应用是一个在 iOS 5中加入的Core Data 新特性允许大的二进制自动存储在SQLite数据库之外的文件系统中。 在我们测试中, 当它改变, iCloud并不知道如何解决依赖关系以及抛出异常。如果您计划使用iCloud同步,可以考虑在iCloud实体中取消这个选择:

![Core Data Modeler Checkbox](http://cloud.mttb.me/UBrx/image.png)

### Model Versioning Model版本

If you are using iCloud, the contents of a store can only be migrated if the store is compatible with automatic [lightweight migration](https://developer.apple.com/library/ios/documentation/cocoa/conceptual/CoreDataVersioning/Articles/vmLightweightMigration.html). This means that Core Data must be able to infer the mapping and you cannot provide your own mapping model. Only simple changes to your model, like adding and renaming attributes, are supported. When considering whether to use Core Data syncing, be sure to think about how much your model may change in future versions of your app.

如果你计划使用iCloud，存储的内容只能在未来兼容自动[轻量级迁移](https://developer.apple.com/library/ios/documentation/cocoa/conceptual/CoreDataVersioning/Articles/vmLightweightMigration.html)，
这意味着Core Data必须能够推断出映射并且你不能提供自己的映射模型。在未来只有简单的改变您的model,如添加和重命名属性被支持。在考虑是否使用Core Data同步时,一定要考虑多少你的app的model在未来版本中改变的情况。

### Merge Conflicts 合并冲突

With any syncing system, conflicts between the server and the client are inevitable. Unlike [iCloud Data Document Syncing](https://developer.apple.com/library/ios/documentation/General/Conceptual/iCloudDesignGuide/Chapters/DesigningForDocumentsIniCloud.html#//apple_ref/doc/uid/TP40012094-CH2-SW1) APIs, iCloud Core Data integration does not specifically allow handling of conflicts between the local store and the transaction logs. That’s likely because Core Data already supports custom merge policies through subclassing of `NSMergePolicy`. To handle conflicts yourself, create a subclass of `NSMergePolicy` and override `resolveConflicts:error:` to determine what to do in the case of a conflict. Then, in your `NSManagedObjectContext` subclass, return an instance of your custom merge policy from the `mergePolicy` method.

在很多的同步系统中，服务器和客户端之前的文件冲突是不可避免的。 不同于[iCloud Data Document Syncing](https://developer.apple.com/library/ios/documentation/General/Conceptual/iCloudDesignGuide/Chapters/DesigningForDocumentsIniCloud.html#//apple_ref/doc/uid/TP40012094-CH2-SW1) APIs, iCloud 的 Core Data 整合并没有明确允许处理本地存储和事务日志之间的冲突。 但是幸运的是 Core Data 以及通过继承 `NSMergePolicy` 的来自定义策合并策略。 如果你要处理冲突， 创建`NSMergePolicy`的子类并且覆盖`resolveConflicts:error:` 来决定在冲突发生的时候做什么, 然后在你的`NSManagedObjectContext`子类中，从`mergePolicy`返回一个你自定义的策略的实例。


### UI Updates 界面更新

Many library-style applications display both collections of objects and detail views which display a single object. Views that are managed by `NSFetchedResultsController` instances will automatically update as the Core Data ubiquity system imports changes from the network. However, you should ensure that each detail view is properly observing changes to its object and keeping itself up to date. If you don't, you risk accidentally showing stale data, or worse, saving edited values on top of newer changes from other peers.

很多库风格应用同时显示集合对象和一个对象的详细信息。 视图是由“NSFetchedResultsController”实例自动从网络更新Core Data数据改变。然而,您应该确保每一个详细视图正确监听变化对象并使自己保持最新。如果你不这样做, 将有显示陈旧的数据的风险或者更糟, 你将覆盖其他设备修改的数据。

## Testing 测试

### Local Networks vs. Internet Syncing 本地网络和因特网同步

The iCloud daemon can synchronize data across devices in one of two ways: on your local network or over the Internet. When the daemon detects that two devices, also known as peers, are on the same local area network, it will transfer the data over that presumably faster connection. If, however, the peers are on separate networks, the system will fall back to transferring transaction logs over the Internet. This is important to know, as you must test both cases heavily in development to make sure your application is functioning properly. In either scenario, syncing changes or transitioning from a fallback store to an iCloud store sometimes takes longer than expected, so if something isn't working right away, try giving it time.

iCloud 守护进程可以跨设备同步数据两种方式中的一种:在你的本地网络或在因特网上。守护进程检测到两个设备时,也被称为同行,在同一个局域网,将在大概快连接传输数据。然而,如果在不同的网络,该系统将回落到在互联网上传输事务日志。知道这很重要,你必须测试这两种情况下大量开发,以确保您的应用程序正常运作。在这两种场景中,从后备存储同步更改或过渡到iCloud春促有时需要比预期更长的时间,所以如果有什么不工作,尝试给它点时间。

### iCloud in the Simulator 模拟器中使用iCloud

One of the most helpful changes to iCloud in iOS 7 is the ability to _finally_ [use iCloud in the iOS Simulator](https://developer.apple.com/library/mac/documentation/General/Conceptual/iCloudDesignGuide/Chapters/TestingandDebuggingforiCloud.html). In previous versions, you could only test on a device, which limited how easy it was to observe the syncing process during development. Now, you can even sync data between your Mac and the simulator as two separate peers.

在 iOS 7 中最有用的更新就是iCloud终于可以在[模拟器](https://developer.apple.com/library/mac/documentation/General/Conceptual/iCloudDesignGuide/Chapters/TestingandDebuggingforiCloud.html)中使用。 在以往的版本中，你只能在设备中测试，这就限制了很难监听开发的同步进程。现在你可以在你的Mac 和 模拟器中同步数据。

With the addition of the iCloud Debug Gauge in Xcode 5, you can see all files in your app's ubiquity containers and examine their file transfer statuses, such as "Current," "Excluded," and "Stored in Cloud." For more hardcore debugging, enable verbose console logging by passing `-com.apple.coredata.ubiquity.logLevel 3` as a launch argument or setting it as a user default. And consider installing the [iCloud Storage Debug Logging Profile](http://developer.apple.com/downloads) on iOS and the new [`ubcontrol`](https://developer.apple.com/library/mac/documentation/Darwin/Reference/Manpages/man1/ubcontrol.1.html) command-line tool on OS X to provide high-quality bug reports to Apple. You can retrieve the logs that these tools generate from `~/Library/Logs/CrashReporter/MobileDevice/device-name/DiagnosticLogs` after syncing your device with iTunes.

在 Xcode 5 新增的 iCloud Debug 选项中， 你可以看到在你的应用程序的开放性存储中的文件和检查他们的文件传输状态,比如"Current," "Excluded," and "Stored in Cloud."。 更多的硬编码测试，需要启用详细日志通过把`-com.apple.coredata.ubiquity.logLevel 3`加入到启动参数或者设置成用户默认，并考虑在iOS中安装[iCloud存储调试日志配置文件](http://developer.apple.com/downloads)以及新的[`ubcontrol`](https://developer.apple.com/library/mac/documentation/Darwin/Reference/Manpages/man1/ubcontrol.1.html)OS X的命令行工具提供高质量错误报告到Apple。你可以获取这些工具生成的日志`~/Library/Logs/CrashReporter/MobileDevice/device-name/DiagnosticLogs` 在你的设备连入iTunes后。

However, iCloud Core Data is not fully supported in the simulator. When testing across devices and the simulator, it seems that iCloud Core Data on the simulator only uploads changes, and never pulls them back down. This is still a big improvement over needing separate test devices, and a nice convenience, but iCloud Core Data support on the iOS Simulator is definitely not fully baked yet.

然而, iCloud Core Data 并不完全支持模拟器。测试设备和模拟器时,似乎 iCloud 核心数据模拟器只上传更改,而且从不把他们拉回去。这仍然是一个巨大的进步更加方便不在需要一个单独的设备,但是 iCloud Core Data 支持iOS模拟器上绝对没有完全成熟。


## Moving On

With the clear improvements in both APIs and functionality in iOS 7, the fate of apps currently shipping iCloud Core Data on iOS 5 and iOS 6 is uncertain. Since the sync systems are so different from an API perspective (and, we’ve found out, from a functionality perspective), Apple's recommendations haven't been kind to apps that need legacy syncing. **Apple explicitly recommends [on the Developer Forums](https://devforums.apple.com/thread/199983?start=0&tstart=0*) that you sync absolutely no data between iOS 7 and any prior version of iOS**.

在 iOS 7 中APIs和功能得到了极大的改善, 在 iOS 5 和 iOS 6 分发带有 iCloud Core Data 的应用的命运是不确定的。 仅从 API 角度来看他们完全不同（当然我们从功能角度也验证了这一点), Apple 建议不要在不同版本设备间的同步因为还没准备好
**Apple 明显的建议 [on the Developer Forums](https://devforums.apple.com/thread/199983?start=0&tstart=0*) 数据不要在iOS 7 和之前的设备同步之间同步**.

In fact, “at no time should your iOS 7 devices be allowed to communicate with iOS 6 peers. The iOS 6 peers will continue to exhibit bugs and issues that have been fixed in iOS 7 but in doing so will pollute the iCloud account.” The easiest way to guarantee this separation is by simply changing the `NSPersistentStoreUbiquitousContentNameKey` of your store, hopefully to comply with new guidance on periods within the name. This difference guarantees that the data from older versions is siloed from the new methods of syncing, and allows developers to make a completely clean break from old implementations.

事实上,“任何时候你的iOS设备7应该允许与iOS 6交流。iOS 6同将继续重现出bug和问题并且在iOS 7修复，但在这样做会污染iCloud账户。“保证这种分离的最简单的方法是通过简单地改变你存储中的“NSPersistentStoreUbiquitousContentNameKey”, 为每一次改变设置单独的名字, 这种差异从旧版本保证数据同步的方法是孤立的,并允许开发人员完全舍弃旧的数据。

## Shipping 分发

Shipping an iCloud Core Data application is still risky. You need to perform tests for everything: account switching, running out of iCloud storage, many devices, model upgrades, and device restores. Although the iCloud Debug Gauge and [developer.icloud.com](http://developer.icloud.com) can help, it’s still quite a leap of faith to ship an app relying on a service that’s completely out of your control. 

分发一个 iCloud Core Data 应用仍旧有很大的风险，你需要对所有的环节进行测试：账户转换，iCloud 存储空间耗尽，各种设备的测试和修复，model 的升级。尽管 iCloud debug 选项和[developer.icloud.com](http://developer.icloud.com)对这些有所帮助，但依靠一个你完全无法控制的服务来发布一个应用仍然需要那种纵身一跃入深渊的信念。

As Brent Simmons [pointed out](http://inessential.com/2013/03/27/why_developers_shouldnt_use_icloud_sy), shipping an app with any kind of iCloud syncing can be limiting, so be sure to understand the costs up front. Apps like [Day One](http://dayoneapp.com) and [1Password](https://agilebits.com/onepassword) have opted to let users sync their data with either iCloud or Dropbox. For many users, nothing beats the simplicity of a single account, but some power users still demand full control over their data. And for developers, maintaining disparate [database syncing systems](https://www.dropbox.com/developers/datastore) can be quite taxing during development and testing.

正如brent simmon[提到](http://inessential.com/2013/03/27/why_developers_shouldnt_use_icloud_sy)的，分发任意一种icloud syncing发布的应用都会被限制，
所以需要实现了解一下成本。像[day one](http://dayoneapp.com)和[1password](https://agilebits.com/onepassword)这样的程序，会选择让使用者用iCloud和Dropbox来同步他们的数据。对于很多使用者来说，没什么可以比一个独立的账户更加简易，但是一部分动手能力强的人喜欢更好的更全面的控制他们的数据。对于开发者而言，维持这种全异性[数据库同步系统](https://www.dropbox.com/developers/datastore)在开发和测试的过程当中是十分繁琐和超负荷的。


## Bugs 

Once you’ve tested and shipped an iCloud Core Data application, you will likely encounter bugs in the framework. The best way to report these bugs to Apple is to file detailed [Radars](http://bugreport.apple.com), which contain the following information:

一旦你测试并且分发了你的iCloud Core Data应用，你将很会遇到错误在这个框架中，最好的办法是反馈这些bug详细信息到[Apple](http://bugreport.apple.com), 需要包含一下信息:

1. Complete steps to reproduce.
2. The output to the console with [iCloud debug logging](http://www.freelancemadscience.com/fmslabs_blog/2012/3/28/debug-settings-for-core-data-and-icloud.html) on level three and the iCloud debugging profile installed.
3. The full contents of the ubiquity container as a zip archive.


1. 完成的重现步骤
2. 输出到控制台与iCloud调试日志记录级别3和iCloud调试配置文件安装。
3. 完整的ubiquity container内容zip包


## Conclusion 结论

It’s no secret that iCloud Core Data was fundamentally broken in iOS 5 and 6, with an Apple employee acknowledging that “there were significant stability and long term reliability issues with iOS 5 / 6 when using Core Data + iCloud…The way forward is iOS 7 only, really really really really.” High-profile developers like [Agile Tortoise](http://agiletortoise.com) and [Realmac Software](http://realmacsoftware.com) are now comfortable trusting the iCloud Core Data integration in their applications, and with enough [consideration](https://developer.apple.com/library/ios/documentation/General/Conceptual/iCloudDesignGuide/Chapters/Introduction.html) and testing, you should be too.

在 iOS 5 和 6 中 iCloud Core Data功能性破坏已经是不是一个秘密， 来自Apple的程序员承认“有重大的长期稳定性和可靠性问题与在OS 5/6 当使用Core Data 以及 iCloud…,真的真的真的真的，在 iOS 7 会得到改进“ 高调的程序员喜欢
[Agile Tortoise](http://agiletortoise.com) 以及 [Realmac Software](http://realmacsoftware.com) 现在开始信任 iCloud Core Data 集成在他们的应用程序中,并有足够的[思考](https://developer.apple.com/library/ios/documentation/General/Conceptual/iCloudDesignGuide/Chapters/Introduction.html)以及测试，你也应该如此。

*Special thanks to Andrew Harrison, Greg Pierce, and Paul Bruneau for their help with this article.*

*特别感谢 Andrew Harrison, Greg Pierce, and Paul Bruneau 对这篇文章的帮助*


[^1]: In previous OS versions, this method wouldn’t return until iCloud data was downloaded and merged into the persistent store. This would cause significant delays, meaning that any calls to the method would need to be dispatched to a background queue. Thankfully, this is no longer necessary.

[^1]: 在之前的 OS版本中， 这个方法将不会任何值直到iCloud 数据下载并合并到持久化存储中。这将造成严重的延误,这意味着任何需要调用方法需要在一个后台的队列中。值得庆幸的这不需要很长时间。
