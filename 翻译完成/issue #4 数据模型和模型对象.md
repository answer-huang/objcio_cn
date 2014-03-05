#数据模型和模型对象

本文我们将更进一步看一看核心数据模型和管理对象类。这不是对标题的简单介绍，而是使用核心数据中很容易碰到的那些少部分人知道或者很容易遗忘的方面的集合。如果你需要的是更详细，教你如何一步步做的概览，那么我们建议你通篇阅读[Apple's Core Data Programming Guide](https://developer.apple.com/library/mac/documentation/cocoa/Conceptual/CoreData/cdProgrammingGuide.html#//apple_ref/doc/uid/TP30001200-SW1)。


##数据模型(Data Model)

数据类型(核心数据里的"实体")被定义在核心数据模型(存储在 *.xcdatamodel 文件里)中。大部分情况下，我们使用 Xcode 的图形化界面定义一个数据模型，但是使用代码创建效果也是相同的。首先，要创建一个[NSManagenObjectModel](https://developer.apple.com/library/ios/documentation/cocoa/Reference/CoreDataFramework/Classes/NSManagedObjectModel_Class/Reference/Reference.html)对象，然后用[NSEntitiyDescription](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/CoreDataFramework/Classes/NSEntityDescription_Class/NSEntityDescription.html)对象创建实体，接下来用[NSAttributeDescription](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/CoreDataFramework/Classes/NSAttributeDescription_Class/reference.html)对象和[NSRelationshipDescription](https://developer.apple.com/library/ios/documentation/cocoa/Reference/CoreDataFramework/Classes/NSRelationshipDescription_Class/NSRelationshipDescription.html)对象创建属性和关系。你基本上不可能需要做这些事情，但是知道这些类是有好处的。


###属性

一旦我们创建了一个实体，我们就已经为它定义了一些属性。属性定义是非常简单的，但是有一些属性值得我们深究。


####默认的/可选的

每一个属性都可以被定义为可选择或者不可选择的。一个不可选属性如果没有设置给一个修改过的对象，将会导致保存失败。同时，我们可以给每一个属性设置一个默认值。没有人禁止我们声明一个可选属性并为它赋一个默认值。但是仔细想想，这没有意义而且会引起混乱。所以我们建议永远不要给可选属性赋默认值。



####瞬时

另一个经常被忽视的属性是瞬时选项。被声明为瞬时的属性永远不会被保存到持久层，但是他们的行为却和一般的属性相同。这意味这它们会参与验证，撤销和故障处理等。当你想将业务逻辑移到管理对象子类的时候，瞬时属性就非常有用了。我们将在后面继续讨论，以及如何更好的使用瞬时属性来代替i变量(ivars)。


####索引

如果之前处理过关系型数据库，那么对索引将比较熟悉。如果没有，你可以将属性的[索引](http://en.wikipedia.org/wiki/Database_index)看成是大幅提高查找速度的方法。然而索引在提高了读的速度的同时降低了写的速度，这是因为，当数据发生改变的时候，索引也不得不做相应的改变。

一个属性如果设置了索引，就潜在的变成了SQLite列索引。我们可以为任意属性创建索引，但是要注意写入效率这一因素。核心数据同样支持创建混合索引(在实体检视板的索引部分)，比如那些延伸到多个属性中的索引。当你通过谓词在多属性的情况下获取数据时，混合索引也能提高效率。Daniel 在他的这篇[获取数据](http://www.objc.io/issue-4/core-data-fetch-requests.html)的文章中有一个例子。


####标量类型

核心数据支持许多基本数据类型，比如整型，浮点型，布尔型等等。然而，管理对象子类，默认情况下数据模型编辑器将这些属性生成为`NSNumber`属性，所以就产生了 floatValue, boolValue, integerValue，以及类似的对 NSNumber 对象的调用。

但是我们同样可以为这些属性指定它们正确的标量类型，例如 int64_t, float_t,或者 BOOL 型，然后使用核心数据进行工作。Xcode 甚至在 NSManagedObject 生成对话框里都没有多少选择框供你选择。总而言之：为了替代这样的声明：
```objecttive-c
	@property (nonatomic, strong) NSNumber *myInteger;
```

属性的声明可以这样:
```objective-c
	@property (nonatomic) int64_t myInteger;
```

这就是我们在核心数据中存取标量类型需要做的全部事情，文档仍然说核心数据不能够自动的为标量生成
存取方法，现在看来这似乎已经过时了。


####存储其他类型

核心数据并没有限制我们只能存储预定义类型。事实上，我们很容易的存储任何遵守 [NSCoding](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Protocols/NSCoding_Protocol/Reference/Reference.html)协议的对象，甚至是可以做更多事情的结构。

为了存储遵守 NSCoding 协议的对象，我们使用[可变属性](https://developer.apple.com/library/ios/documentation/cocoa/conceptual/CoreData/Articles/cdNSAttributes.html#//apple_ref/doc/uid/TP40001919-SW7)。我们需要做的仅仅是在下拉菜单中选择"Transformable"。如果你生成了一个管理对象子类，你会看到像这样的声明:

```objective-c
	@property (nonatomic, retain) id anObject
```


我们可以手动修改 id 类型，让其成为我们想存储的任意类型，以便得到编译器的类型检查。但是，使用可变属性的时候有一个陷进：如果我们想使用默认的转换器(transformer，大部分情况下都是使用默认的)，我们一定不能为转换器指定一个名字，甚至是默认的转换器的名字也不行，比如`NSKeyedUnarchiveFromDataTransformerName`，都会引起[不好的事情](https://developer.apple.com/library/ios/documentation/cocoa/conceptual/CoreData/Articles/cdNSAttributes.html#//apple_ref/doc/uid/TP40001919-SW7)。

但是，事情并不会止步于此，我们仍然可以创建自定义的值转换器，然后使用它们存储任意的对象类型。只要我们能够将它转换成支持的基本类型的一种，我们就可以存储任何东西。为了存储不支持的，非对象的类型，比如结构体，基本的途径是首先，创建未定义类型的瞬时属性以及一个固化在已支持类型里的阴影属性，然后，重写瞬时属性的存取方法，来将值转换为持久类型，这一步是很重要的，因为这些存取方法必须要服从 KVC 和 KVO依赖，然后正确地使用核心数据的私有存取方法。请阅读 Apple 的指南中的[非标准的持久属性](https://developer.apple.com/library/ios/documentation/cocoa/conceptual/CoreData/Articles/cdNSAttributes.html#//apple_ref/doc/uid/TP40001919-SW1)这一部分的[自定义代码](https://developer.apple.com/library/ios/documentation/cocoa/conceptual/CoreData/Articles/cdNSAttributes.html#//apple_ref/doc/uid/TP40001919-SW8)




###抓取属性(Fetched Properties)

抓取属性通常用来创建多个持久存储之间的联系。由于使用多个持久存储是不常使用而且比较高级的用例，抓取属性因此也很少使用。

实际上，当我们得到了一个抓取属性的时候，核心数据执行了一个抓取请求并得到了结果。这个抓取请求可以在 Xcode 的数据模型编辑器里通过指定一个目标实体类型和一个断言(predicate)直接配置。断言不是静态的，而是可以在运行时通过 $FETCH_SOURCE 和 $FETCHED_PROPERTY 变量进行配置的，更多的细节可以参考苹果的[文档](https://developer.apple.com/library/ios/documentation/cocoa/conceptual/CoreData/Articles/cdRelationships.html#//apple_ref/doc/uid/TP40001857-SW7)



###关系

实体之间的关系应该总是被定义成双向的。这给核心数据(Core Data)提供了足够的信息来彻底帮我们管理图标对象。然而定义双向关系并不是必须的，但是仍然强烈建议定义双向关系

如果你确实知道你在做什么，你可以定义单向关系，核心数据并不会抱怨什么。但是这样，你就要承担很多手动管理核心数据的责任，包括确保图标对象的一致性，变化跟踪以及撤销操作。举一个简单的例子，如果我们有书和作者两个实体，并且定义了一个从书到作者的单向关系，删除一个作者并不会将这一删除消息告知被影响的书。我们仍然能够获得这些书的作者的关系，但是我们会得到一个指向空的错误。

很清楚的可以看到，单向关系绝对不是你想要的。总是定义双向关系能让你远离麻烦。


###数据模型设计

当为核心数据设计数据模型的时候，我们一定要记住的一点就是核心数据并不是关系型数据库。因此，我们不应该像设计数据库模式那样设计数据模型，而是应该更多的从如何组织和展示数据的角度去构思。

通常，展示数据的时候，为了避免大量的抓取相关的对象，使数据模型[非规范化](http://en.wikipedia.org/wiki/Denormalization)是有意义的。例如，我们有一个作者实体和书籍实体，并且作者实体和数据实体的关系是一对多，假如我们以后需要显示某一个作者的书籍的数量，那么则有必要在作者实体中存储该值。


先假设我们想在表格中列出所有的作者和每一个作者的书的数量。如果每一个作者的书的数量只能通过计算与之相关的书的个数这一途径取得，每一个单元格都需要一个抓取请求来获取数据。这样的性能表现不好。我们可以使用 [relationshipKeyPathsForPrefetching](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/CoreDataFramework/Classes/NSFetchRequest_Class/NSFetchRequest.html#//apple_ref/occ/instm/NSFetchRequest/relationshipKeyPathsForPrefetching) 预抓取书籍对象，但是如果我们有大量的书籍的话效果并不理想。如果我们为每一个作者管理一个书籍个数的属性，那么，作者的抓取请求就可以得到我们需要的所有的数据了。

当然，非规范在保证冗余数据同步的同时，也带来了一些资源的消耗。我们需要对每一个具体问题来权衡利弊，因为，有的时候它很容易实现，有的时候她会产生大麻烦。这非常依赖于特定的数据模型，比如一个应用需要在后台交互，或者数据需要通过中心认证或者点对点的方式在多态客户端实现同步。


通常，数据模型会被定义为一些后台服务，然后我们可能需要为不同的客户端应用程序提供这些模型的副本。然而，即便是在这种情况下，我们仍然能自由地通知在客户端上的数据，就像我们可以为后台的数据模型定义清晰的映射关系。对于这个简单的书和作者的例子来说，就不那么需要为作者的核心数据实体添加一个书籍数量属性，那样只会让客户端产生性能问题，而且这些数据根本不需要发送给服务器。如果我们在本地做一些改变或者只接收来自服务器的新数据，然后我们更新这一属性使之和其余的数据保持同步。


实际情况并不总是那样简单，但是像上面那样的一些小的改变能够减轻，由于处理标准化的关系型类数据库的数据模型所带来的性能瓶颈问题。


###实体层级和类层级

管理对象模型能够创建实体层级。例如，我们可以指定某一个实体是另一个实体的父实体。这看上去不错，比如，我们的实体可以共享一些共同的属性。然而在实践中，你基本不会这样做。

背后的本质是，在同一个表中，核心数据使用共同的父实体来存储所有的实体。这可以快速创建有许多属性的表，同时降低系统性能。通常创建实体层级的目的仅仅是为了创建一个类层级，这样就能够将共享在多个实体的代码放进他们的父类中，虽然有一种更好的方法实现这一点。

实体层级是独立于`NSManagedObject`子类层级的。或者说，我们不需要为了创建一个类层级而去创建一个实体层级。

让我们继续看作者和书的例子。这两个实体有相同的字段，比如：id, 创建日期字段(createdAt)，更改日期字段(changedAt)。在这种情况下，我们可以创建如下结构:

Entity hierarchy            Class hierarchy
----------------            ---------------

   BaseEntity                 BaseEntity
    |      |                   |      |
 Authors  Books             Authors  Books    (此处可截图)
 
 
 
 但是，我们可以保留这个类层级，同时去掉实体层级：
 
  Entity hierarchy            Class hierarchy
 ----------------            ---------------

  Authors  Books               BaseEntity
                                |      |
                             Authors  Books  (可截图)
                             
                             
 
 
 这个类可以像这样声明:
 
 ```objective-c
 	@interface BaseEntity : NSManagedObject
 	@property (nonatomic) int64_t identifier;
 	@property (nonatomic, strong) NSDate *createdAt;
 	@property (nonatomic, strong) NSDate *changedAt;
 	@end
 ```

```objective-c
 @interface Author : BaseEntity
 // Author specific code...
 @end
 ```

```objective-c
 @interface Book : BaseEntity
 // Book specific code...
 @end
 ```
 
这能够让我们将共同的代码移动到父类中，同时避免了将所有属性放进一个表中所带来的性能问题。尽管我们使用 Xcode 的管理对象生成器创建类层级的时候不能脱离实体层级，但是不自动生成管理对象类有更大的好处，且不需要多大代价，我们会在下面介绍。


###配置和抓取请求模板

每一个使用核心数据的人都使用过数据模型的实体-模型转化这个方面。但是数据模型同样有两个鲜为人知以及使用较少的两个方面：配置和抓取请求模板。

配置是用来定义实体应该被存储在哪个持久存储中的。持久存储协调器使用[`addPersistentStoreWithType:configuration:URL:options:error:`](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/CoreDataFramework/Classes/NSPersistentStoreCoordinator_Class/NSPersistentStoreCoordinator.html#//apple_ref/occ/instm/NSPersistentStoreCoordinator/addPersistentStoreWithType:configuration:URL:options:error:)来添加持久存储，通过配置参数来定义映射。现在，几乎在所有的情形下，我们只需要使用一个永久存储，因此永远不需要处理多个配置。我们需要的一个存储的默认配置已经在创建的时候为我们创建好了。尽管几乎很少情况下会使用到多个存储,他们中的一个已经在本文中有概要介绍，参见[Import Large Data Sets](http://www.objc.io/issue-4/importing-large-data-sets-into-core-data.html#user-generated-data)。抓取请求模板就像它的名字所描述的那样：在管理对象模型中预先定义好抓取请求，以便后面使用[`fetchRequestFromTemplateWithName:substitutionVariables`](https://developer.apple.com/library/ios/documentation/cocoa/Reference/CoreDataFramework/Classes/NSManagedObjectModel_Class/Reference/Reference.html#//apple_ref/occ/instm/NSManagedObjectModel/fetchRequestFromTemplateWithName:substitutionVariables:)来调用。我们可以使用 Xcode 的数据模型编辑器或者使用代码来定义那些模板，尽管 Xcode 的编辑器并不支持`NSFetchRequest`的所有特性。

说实话，我很难提供有说服力的的使用抓取请求模板的用例。一个优势是抓取请求的断言会被预解析，所以这一步在你每次执行一个新的抓取请求的时候不是必须要做的。尽管这基本不相关，如果我们不得不频繁抓取，那我们就陷入麻烦了。但是，如果你正在寻找定义抓取请求的地方(你不应该把它们定义在视图控制器里...)，那么检查一下是否用了管理对象模型存储它们是一个不错的选择。


##管理对象(Managed Objects)


管理对象是任何基于核心数据的应用程序的核心部分。管理对象存在于管理对象上下文中用以展示数据。管理对象在应用程序中传递，至少在模型和控制器间传递，甚至可能在控制器和视图之间传递。尽管后一种情况或多或少有一些争议。因为它可以被[抽象成一种更好的方式](http://www.objc.io/issue-1/table-views.html)。比如：定义一个某对象需要服从的协议，供一个特定的视图使用，或者在视图的类别中实现配置方法来桥接模型对象和指定的视图。


不管怎样，我们不应该将管理对象局限在模型层，也不应该随意将它们的数据传递进不同的结构。管理对象是核心数据应用程序里的一等公民，因为我们也应该相应地使用它。例如：管理对象应该在视图控制器之间传递并提供它们所需的数据。

为了获取管理对象上下文，我们经常在视图控制器里看到这样的代码：
```objective-c
	NSManagedObjectContext *context = [(MyApplicationDelegate *)[[UIApplication sharedApplication] delegate] managedObjectContext];
  ```
  
如果你已经向视图控制器传递了一个模型对象，那么最好是通过该对象来获取上下文：
```objective-c
	NSManagedObjectContext *context = self.myObject.managedObjectContext;
```
这去掉了和应用程序代理的隐藏关联，同时增加了可读性并且更容易测试。


###使用管理对象子类


类似，管理对象的子类也是可以被使用的。我们可以并且应该在这些类中实现自定义的业务逻辑，认证逻辑和助手方法，同时为了将共同的代码放进父类中而创建类层级。后一种是容易做的，因为如何去除类层级和实体层级之间的耦合性上面已经讲过了。

可能你会想，如果 Xcode 总是会在重新生成文件的时候覆盖它们，那么该如何在管理对象子类中实现自定义的代码。其实答案很简单：不用 Xcode 生成它们。如果你考虑这些，这些类里生成出来的代码都不太重要并且很容易自己实现，或者生成一次，然后手动保持更新。那仅仅是一些属性的声明。


还有一些其他的途径比如将自定义的代码放入一个分类，或者使用工具[mogenerator](https://github.com/rentzsch/mogenerator)，它能够为每一个实体创建一个父类和子类供用户手写代码。但是上面所有的方案都不能将类层级灵活地独立于实体层级。所以我们的建议是手动编写这些类。


####管理对象子类里的实例变量

一旦我们开始使用自己的管理对象子类实现业务逻辑，我们可能会想要创建一些实例变量来缓存计算结果或者类似的值。尽管这里使用瞬时属性会更合适，原因是管理对象的生命周期和一般对象有所不同。核心数据经常[faults](https://developer.apple.com/library/ios/documentation/cocoa/conceptual/CoreData/Articles/cdFaultingUniquing.html#//apple_ref/doc/uid/TP30001202-CJBDBHCB)那些不需要的对象。如果我们想使用实例变量，我们需要手动加入进程并释放实例变量。然而如果我们使用瞬时属性，所有的这些都为我们做好了。


####创建新对象


一个为模型类实现助手方法的好的例子是用一个类方法向管理对象上下文插入一个对象。核心数据的创建新对象的API不是非常直观:

```objective-c
	Book *newBook = [NSEntityDescription insertNewObjectEntityForName:@"Book" inManagedObjectContext:context];
```

幸运的是，我们可以在我们自己的子类中以一种更优雅的方式解决这个问题:

```objective-c
	@implementation Book
	// ...
	+ (NSString *)entityName
	{
		return @"Book";
	}
	+ (instancetype)insertNewObjectIntoContext:(NSManagedObjectContext *)context
	{
		return [NSEntityDescription insertNewObjectForEntityForName:[self entityName] inManagedObjectContext:context];
	}
@end
```

现在，创建一个新的书本对象就简单多了：

```objective-c
	Book *book = [Book insertNewObjectIntoContext:context];
```

当然，如果我们的子类是从共同父类中继承下来，我们应该将`insertNewObjectIntoContext:`和`entityName`类方法移至共同父类中，这样每一个子类就只需要重写`entityName`方法就行了。


####一对多关系转变(To-Mana Relationship Mutators)

如果你使用 Xcode 生成了一个含有一对多关系的的管理对象子类，它会创建如下的方法给关系添加或者删除对象:

```objective-c
	- (void)addBooksObject:(Book *)value;
	- (void)removeBooksObject:(Book *)value;
	- (void)addBooks:(NSSet *)values;
	- (void)removeBooks:(NSSet *)values;
```

除了使用上面这4个增变方法外，还有一种更优雅的方式，尤其在我们不想生成管理对象子类的时候。我们可以简单的使用[`mutableSetValueForKey:`](https://developer.apple.com/library/mac/documentation/cocoa/Reference/Foundation/Protocols/NSKeyValueCoding_Protocol/Reference/Reference.html#//apple_ref/occ/instm/NSObject/mutableSetValueForKey:)方法获取相关的可变对象的集合(或者有序关系用[`mutableOrderedSetValueForKey:`](https://developer.apple.com/library/mac/documentation/cocoa/Reference/Foundation/Protocols/NSKeyValueCoding_Protocol/Reference/Reference.html#//apple_ref/occ/instm/NSObject/mutableOrderedSetValueForKey:))。这可以封装在一个简单的存取方法中:

```objective-c
	- (NSMutableSet *)mutableBooks
	{
		return [self mutableSetValueForKey:@"books"];
	}
```

然后我们可以像使用其他集合那样使用这个可变集合，核心数据会为我们选择变化以及为我们做好余下的工作:

```objective-c
	Book *newBook = [Book insertNewObjectIntoContext:context];
	[author.mutableBooks addObject:newBook];
```


####验证

核心数据支持多种方式的数据验证。Xcode 的数据模型编辑器让我们为属性指定一些基本要求，比如一个字符串的最大和最小长度或者一对多关系能包含的最少和最多的对象个数，但是除此之外，我们可以用代码做更多的事情。

["Managed Object Validation"](https://developer.apple.com/library/ios/documentation/cocoa/conceptual/CoreData/Articles/cdValidation.html#//apple_ref/doc/uid/TP40004807-SW1)部分对这一主题有更深入的介绍。核心数据通过实现`validate<Key>:error:`方法来支持属性级别的验证，类似地，属性内验证通过[`validateForInsert:`](https://developer.apple.com/library/ios/DOCUMENTATION/Cocoa/Reference/CoreDataFramework/Classes/NSManagedObject_Class/Reference/NSManagedObject.html#//apple_ref/occ/instm/NSManagedObject/validateForInsert:)，[`validForUpdate`](https://developer.apple.com/library/ios/DOCUMENTATION/Cocoa/Reference/CoreDataFramework/Classes/NSManagedObject_Class/Reference/NSManagedObject.html#//apple_ref/occ/instm/NSManagedObject/validateForUpdate:)和[`validateForDelete:`](https://developer.apple.com/library/ios/DOCUMENTATION/Cocoa/Reference/CoreDataFramework/Classes/NSManagedObject_Class/Reference/NSManagedObject.html#//apple_ref/occ/instm/NSManagedObject/validateForDelete:)方法。在保存之前，会自动进行验证，但是我们仍然能够通过[`validateValue:forKey:error`](https://developer.apple.com/library/ios/DOCUMENTATION/Cocoa/Reference/CoreDataFramework/Classes/NSManagedObject_Class/Reference/NSManagedObject.html#//apple_ref/occ/instm/NSManagedObject/validateValue:forKey:error:)方法，在属性级别手动触发验证。





##结论

数据模型和模型对象是任何核心数据应用程序的面包和黄油。我们建议你不要立刻使用那些舒适的封装，而是使用管理对象子类和对象。同核心数据一样，了解本质是非常重要的，否则，一旦你的应用变得越来越复杂，是很容易适得其反的。

希望我们已经阐明了一些简单的技术，并且没有引入一些让人晕头转向的东西，能够使管理对象用起来更简单。除此之外，我们深入研究了一些非常高级的数据模型的用法，为了告诉你哪些是可能的。但是别过度使用这些技巧，因为简介，总是更好的。


-------------------

####[更过关于话题4的文章](http://www.objc.io/issue-4/)












































