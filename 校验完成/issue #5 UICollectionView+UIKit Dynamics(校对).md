

译自[objc.io](http://www.objc.io/issue-5/collection-views-and-uidynamics.html),原文作者[Ash Furrow](https://twitter.com/ashfurrow)。

参与翻译的有(排名不分先后):

- jiahan(happyjiahan@gmail.com)
- Sheldon(allenwenzhou@gmail.com)

UIKit Dynamics 是iOS7中基于物理动画引擎的一个新功能--它被特别设计使其能很好地与collection Views配合工作，collection view是在iOS6中才被引入的新特性。接下来，我们要好好看看如何将这两个特性结合在一起。 

这篇文章将讨论两个实现UIkit Dynamics的collection view的例子。第一个例子展示了如何去实现像iOS7里Messages应用中的弹簧效果，如果你还不大了解iOS7中Messages的弹簧效果是什么样子的，下面这张gif图可以帮你大概了解下：

![Example](http://img.onevcat.com/2013/ios7-message-app-spring.gif)

然后再进一步结合了有可伸缩效果的平铺机制。

第二个例子展现了如何用UIKit Dynamics来模拟牛顿摆，物体可以在某一时刻被加入到collection view，并和其他物体相互作用。

在我们开始之前，我假定你们对`UICollectionView`是如何工作是有基本的了解--查看[这篇objc.io博客](http://www.objc.io/issue-3/collection-view-layouts.html)博客会有你想要的所有细节。我也假定你已经理解了`UIKit Dynamics`的工作原理--阅读这篇[博客](http://www.teehanlax.com/blog/introduction-to-uikit-dynamics/)，可以了解更多UIKit Dynamics的知识。

文章中的两个例子项目都已经在GitHub中:

- [ASHSpringyCollectionView](https://github.com/objcio/issue-5-springy-collection-view)(基于[UICollectionView Spring Demo](https://github.com/TeehanLax/UICollectionView-Spring-Demo))
- [Newtownian UICollectionView](https://github.com/objcio/issue-5-newtonian-collection-view)


#Dynamic Animator

支持UICollectionView实现UIkit Dynamic的最关键部分就是UIDynamicAnimator。要实现这样的UIKit Dynamics的效果，我们需要自己自定义一个继承于UICollectionViewFlowLayout的子类，并且在这个子类里面持有一个UIDynamicAnimator的对象。

当我们创建我们自己的dynamic animator时，我们不会使用常用的初始化方法`-(instancetype)initWithReferenceView:(UIView*)view;`
因为，我们不需要把这个dynamic animator关联一个view，而是给它关联一个collection view layout。所以我们使用`- (instancetype)initWithCollectionViewLayout:(UICollectionViewLayout*)layout;`
这个初始化方法，并把collection view layout作为参数传入。
这很关键，当他的behavior物体的属性应该被更新的时候，dynamic animator必须能够使collection view layout无效。换句话说，dynamic animator将会使旧的layout失效，并根据最新的behavior的items中的attribute重新建立新的layout。

我们很快就能看到这些事情是怎么连接起来的，但是在概念上理解collection view 如何与 dynamic animator相互作用是很重要的。我们将要在自定义的collection view layout的子类中，根据每一个collection view中的`UICollectionViewLayoutAttributes`对象的属性，创建一个对应的`UIAttachmentBehavior`对象，并把这个UIAttachmentBehavior对象添加到我们持有的`UIDynamicAnimator`对象上(过会儿我们将讨论tiling这些)。当我们需要`UICollectionViewLayoutAttribute`时，我们不再是从头开始计算collection view 每一个item的layout attribute，而是使用`UIDynamicAnimator`中的layout attribute，因为我们在创建UIDynamicAnimator时就已经计算过每一个item的layout attribute了，所以这里不需要再重复计算一次。一旦模拟状态发生改变，dynamic animator就会使这个layout无效。这会导致UIKit重新查询layout，直到这个模拟静止。

所以重申一下，layout创建了dynamic animator，并且在dynamic animator上添加每个item的layout attribute对应的`UIAttachmentBehavior`。当collection view需要layout信息时，dynamic animator提供想要的信息。

#继承UICollectionViewFlowLayout

我们将要创建一个简单的例子来展示如何使用一个带UIkit Dynamic的collection view layout。当然，我们需要做的第一件事就是，创建一个数据源去驱动我们的collection view。我知道以你的能力完全可以独立实现一个数据源，但是为了完整性，我还是提供了一个给你:

```objc

@implementation ASHCollectionViewController

static NSString * CellIdentifier = @"CellIdentifier";

-(void)viewDidLoad 
{
    [super viewDidLoad];
    [self.collectionView registerClass:[UICollectionViewCell class] 
            forCellWithReuseIdentifier:CellIdentifier];

}

-(UIStatusBarStyle)preferredStatusBarStyle 
{
    return UIStatusBarStyleLightContent;

}

-(void)viewDidAppear:(BOOL)animated 
{
    [super viewDidAppear:animated];
    [self.collectionViewLayout invalidateLayout];

}

#pragma mark - UICollectionView Methods

-(NSInteger)collectionView:(UICollectionView *)collectionView 
    numberOfItemsInSection:(NSInteger)section 
{
    return 120;

}

-(UICollectionViewCell *)collectionView:(UICollectionView *)collectionView 
                 cellForItemAtIndexPath:(NSIndexPath *)indexPath 
{
    UICollectionViewCell *cell = [collectionView 
        dequeueReusableCellWithReuseIdentifier:CellIdentifier 
                                  forIndexPath:indexPath];
    
    cell.backgroundColor = [UIColor orangeColor];
    return cell;

}

@end

```

我们注意到当视图第一次出现的时候，这个layout是被无效的。这是因为没有用Storybards的结果(当使用Storyboards时，调用prepareLayout方法的时机是不同的--或是相同的--在WWDC的视频中他们没有告诉我们这些)。所以，当这些试图一出现我们就需要手动使这个collection view layout无效。当我们用tiling的时候，就不需要这样。

让我们创建我们自己的collection view layout。我们需要强引用一个dynamic animator, 并且使用它来驱动我们的collcetion view layout的attribute。我们在实现文件里定义了一个私有的property:

```objc

@interface ASHSpringyCollectionViewFlowLayout ()

@property (nonatomic, strong) UIDynamicAnimator *dynamicAnimator;

@end

```

我们将在layout的初始化方法中初始化我们的dynamic animator。还要设置一些属于父类UICollectionViewFlowLayout中的property:

```objc

- (id)init 
{
    if (!(self = [super init])) return nil;
    
    self.minimumInteritemSpacing = 10;
    self.minimumLineSpacing = 10;
    self.itemSize = CGSizeMake(44, 44);
    self.sectionInset = UIEdgeInsetsMake(10, 10, 10, 10);
    
    self.dynamicAnimator = [[UIDynamicAnimator alloc] initWithCollectionViewLayout:self];
    
    return self;

}

```

我们将实现的下一个方法是prepareLayout。我们首先需要调用父类的方法。因为我们是继承`UICollectionViewFlowLayout`类，所以在调用父类的prepareLayout的时，可以使collection view layout attribute都放置在合适的位置。我们可以依靠基类`UICollectionViewFlowLayout`的`prepareLayout`方法来提供一个默认的排布，并且能够使用`[super layoutAttributesForElementsInRect:visibleRect];`方法得到指定rect内的所有item的layout attributes。

```objc

[super prepareLayout];

CGSize contentSize = self.collectionView.contentSize;
NSArray *items = [super layoutAttributesForElementsInRect:
    CGRectMake(0.0f, 0.0f, contentSize.width, contentSize.height)];

```

这真的是效率低下的代码。因为我们的collection view中可能会有成千上万个cell，一次性加载所有的cell是一个可能会产生难以置信的内存紧张的操作。我们要在一段时间内遍历所有的元素，这也成为耗时的操作。这真的是效率的双重打击！别担心--我们是负责任的开发者，所以我们会很快解决这个问题的。我们先暂时继续使用简单、粗暴的实现方式。


当加载完我们所有的collection view layout attribute之后，我们需要检查他们是否都已经被加载到我们的animator里了。如果一个behavior已经在animator中存在，那么我们就不能重新添加，否则就会得到一个非常难懂的运行异常提示:


```objc

<UIDynamicAnimator: 0xa5ba280> (0.004987s) in 
<ASHSpringyCollectionViewFlowLayout: 0xa5b9e60> \{\{0, 0}, \{0, 0\}\}: 
body <PKPhysicsBody> type:<Rectangle> representedObject:
[<UICollectionViewLayoutAttributes: 0xa281880> 
index path: (<NSIndexPath: 0xa281850> {length = 2, path = 0 - 0}); 
frame = (10 10; 300 44); ] 0xa2877c0  
PO:(159.999985,32.000000) AN:(0.000000) VE:(0.000000,0.000000) AV:(0.000000) 
dy:(1) cc:(0) ar:(1) rs:(0) fr:(0.200000) re:(0.200000) de:(1.054650) gr:(0) 
without representedObject for item <UICollectionViewLayoutAttributes: 0xa3833e0> 
index path: (<NSIndexPath: 0xa382410> {length = 2, path = 0 - 0}); 
frame = (10 10; 300 44);

```

如果看到了这个错误，那么这基本表明你添加了两个behavior给同一个`UICollectionViewLayoutAttribute`，这使得系统不知道该怎么处理。

无论如何，一旦我们已经检查好我们是否已经将behavior添加到dynamic animator之后，我们就需要遍历每个collection view layout attribute来创建和添加新的dynamic animator:

```objc

if (self.dynamicAnimator.behaviors.count == 0) {
	[items enumerateObjectsUsingBlock:^(id<UIDynamicItem> obj, NSUInteger idx, BOOL *stop) {
        UIAttachmentBehavior *behaviour = [[UIAttachmentBehavior alloc] initWithItem:obj 
                                                                    attachedToAnchor:[obj center]];
        
        behaviour.length = 0.0f;
        behaviour.damping = 0.8f;
        behaviour.frequency = 1.0f;
        
        [self.dynamicAnimator addBehavior:behaviour];
    
	}];

}

```

这段代码非常简单。我们为每个item创建了一个以物体的中心为附着点的`UIAttachmentBehavior`对象。然后又设置了我们的attachment behavior的length为0以便约束这个cell能一直以behavior的附着点为中心。然后又给`damping`和`frequency`这两个参数设置一个比较合适的值。

这就是`prepareLayout`。我们现在需要实现`layoutAttributesForElementsInRect:` 和 `layoutAttributesForItemAtIndexPath:`这两个方法，UIKit会调用它们来询问collection view每一个item的布局信息。我们写的代码会把这些查询交给专门做这些事的dynamic animator:

```objc

-(NSArray *)layoutAttributesForElementsInRect:(CGRect)rect 
{
    return [self.dynamicAnimator itemsInRect:rect];

}

-(UICollectionViewLayoutAttributes *)layoutAttributesForItemAtIndexPath:(NSIndexPath *)indexPath 
{
    return [self.dynamicAnimator layoutAttributesForCellAtIndexPath:indexPath];

}

```

#响应滚动事件

我们目前实现的代码给我们展示的只是一个在正常滑动下只有静态感觉的`UICollectionView`运行起来没什么特别的。看上去很好，但不是真的动态，不是么？

为了使它表现地动态点，我们需要layout和dynamic animator能够对collection view中滑动位置的变化做出反应。幸好这里有个非常适合这个要求的方法`shouldInvalidateLayoutForBoundsChange:`。这个方法会在collection view 的bound发生改变的时候被调用，根据最新的`content offset`调整我们的dynamic animator中的behaviors的参数。在重新调整这些behavior的item之后，我们在这个方法中返回NO；因为dynamic animator会关心layout的无效问题，所以在这种情况下，它不需要去主动使其无效:

```objc

-(BOOL)shouldInvalidateLayoutForBoundsChange:(CGRect)newBounds 
{
    UIScrollView *scrollView = self.collectionView;
    CGFloat delta = newBounds.origin.y - scrollView.bounds.origin.y;
    
    CGPoint touchLocation = [self.collectionView.panGestureRecognizer locationInView:self.collectionView];
    
    [self.dynamicAnimator.behaviors enumerateObjectsUsingBlock:^(UIAttachmentBehavior *springBehaviour, NSUInteger idx, BOOL *stop) {
        CGFloat yDistanceFromTouch = fabsf(touchLocation.y - springBehaviour.anchorPoint.y);
        CGFloat xDistanceFromTouch = fabsf(touchLocation.x - springBehaviour.anchorPoint.x);
        CGFloat scrollResistance = (yDistanceFromTouch + xDistanceFromTouch) / 1500.0f;
        
        UICollectionViewLayoutAttributes *item = springBehaviour.items.firstObject;
        CGPoint center = item.center;
	if (delta < 0) {
            center.y += MAX(delta, delta*scrollResistance);
        
	}
	else {
            center.y += MIN(delta, delta*scrollResistance);
        
	}
        item.center = center;
        
        [self.dynamicAnimator updateItemUsingCurrentState:item];
    
    }];
    
    return NO;

}

```

让我们仔细查看这个代码的细节。首先我们得到了这个scroll view(这是我们的collection view)，然后计算它的content offset中y的变化(在这个例子中，我们的collection view是垂直滑动的)。一旦我们得到这个增量，我们需要得到用户接触的位置。这是非常重要的，因为我们希望离接触位置比较近的那些物体能移动地更迅速些，而离接触位置比较远的那些物体则应该滞后些。

对于dynamic animator中的每个behavior，我们将接触点到该behavior物体的x和y的距离之和除以1500，1500是我根据经验设的。分母越小，这个collection view的的交互就越有弹簧的感觉。因为我们有了这种“滑动阻力”，我们根据它的增量乘上`scrollResistance`这个变量来指定这个behavior物体的中心的y。最后，我们在滑动阻力大于增量的情况下对增量和滑动阻力的结果进行了选择。 (这意味着物体开始往错误的方向移动了)。如果我们用了这么大的分母，那么这种情况是不可能的，但是在一些更具弹性的collection view layout中还是需要注意的。


就是这么一回事。以我的经验，这个方法对多达几百个物体的collection view来说也是是适用的。超过这个数量的话，一次性加载所有物体到内存中就会变成很大的负担，并且在滑动的时候就会开始卡顿了。


![Example](http://f.cl.ly/items/411o450x2E3A3c3m2b3k/springyCollectionView.gif)

#Tiling your Dynamic Behaviors for Performance

当你的collection view中只有几百个cell的时候，他运行的很好，但当数据源超过这个范围的时候会发生什么呢？或者在运行的时你不能预测你的数据源有多大呢？我们的简单粗暴的方法就不管用了。

除了在`prepareLayout`中加载所有的物体，如果我们能更聪明地知道哪些物体会加载那该多好啊。是的，就是这些显示的或即将显示的物体。这就是我们要采取的办法。

我们需要做的第一件事就是是跟踪dynamic animator中的所有behavior物体的index path。我在collection view 中添加一个property来做这件事:


```objc

@property (nonatomic, strong) NSMutableSet *visibleIndexPathsSet;

```

我们用set是因为它具有常数复杂度的查找效率，并且我们经常地查找`visibleIndexPathsSet`中是否已经包含了某个index path。


在我们实现全新的`prepareLayout`方法之前--有一个问题就是什么是tiles behavior--理解tiling的意思是非常重要的。当我们title behavior的时候，我们会在这些item离开collection view 的可视范围的时候删除对应的behavior，在这些item进入可视范围的时候又添加对应的behavior。这是一个大麻烦:我们需要在滚动中创建新的behavior。这就意味着让人觉得创建它们就好像它们本来就已经在dynamic animator里了一样，并且它们是在`shouldInvalidateLayoutForBoundsChange:`方法被修改的。

因为我们是在滚动中创建这些新的behavior，所以我们需要维持现在collection view 的一些状态。尤其我们需要跟踪最近一次我们`bound`变化的增量。我们会在滚动时用这个状态去创建我们的behavior:


```objc

@property (nonatomic, assign) CGFloat latestDelta;

```

添加完这个property后，我们将要在`shouldInvalidateLayoutForBoundsChange:`方法中添加下面这行代码:


```objc

self.latestDelta = delta;

```

这就是我们需要修改我们的方法来响应滚动事件。我们的这两个方法是为了将collection view中items的layout信息传给dynamic animator，这种方式没有变化。事实上，当你的collection view实现了dynamic animator的大部分情况下，都需要实现我们上面提到的两个方法`layoutAttributesForElementsInRect:`和`layoutAttributesForItemAtIndexPath:`。

这里最难懂的部分就是tiling mechanism。我们将要完全重写我们的prepareLayout。

这个方法的第一步是将那些物体的index path已经不再屏幕上的behavior从dynamic animator上删除。第二步是添加那些即将显示的物体的behavior。

让我们先看一下第一步。

像以前一样，我们要调用`super prepareLayout`，这样我们就能依赖父类`UICollectionViewFlowLayout`提供的默认排布。还像以前一样，我们通过父类获取一个矩形内的所有元素的layout attribute。不同的是我们不是获取整个collection view 中的元素属性，而只是获取显示范围内的。

所以我们需要计算这个显示矩形。但是别着急！有件事要记住。我们的用户可能会非常快地滑动collection view，导致了dynamic animator不能跟上，所以我们需要稍微扩大显示范围，这样就能包含到那些将要显示的物体了。否则，在滑动很快的时候就会出现频闪现象了。让我们计算一下显示范围:

```objc

CGRect originalRect = (CGRect){.origin = self.collectionView.bounds.origin, .size = self.collectionView.frame.size};
CGRect visibleRect = CGRectInset(originalRect, -100, -100);

```

我确信在实际显示矩形上的每个方向都扩大100个像素对我的demo来说是可行的。仔细查看这些值是否适合你们的collection view，尤其是当你们的cell很小的情况下。

接下来我们就需要收集在显示范围内的collection view layout attributes。还有它们的index paths:

```objc

NSArray *itemsInVisibleRectArray = [super layoutAttributesForElementsInRect:visibleRect];

NSSet *itemsIndexPathsInVisibleRectSet = [NSSet setWithArray:[itemsInVisibleRectArray valueForKey:@"indexPath"]];

```

注意我们是在用一个NSSet。这是因为它具有常数复杂度的查找效率，并且我们经常的查找visibleIndexPathsSet是否已经包含了某个index path:

接下来我们要做的就是遍历dynamic animator 的behaviors，过滤掉那些已经在`itemsIndexPathsInVisibleRectSet`中的item。因为我们已经过滤掉我们的behavior，所以我们将要遍历的这些item都是不在显示范围里的，我们就可以将这些item从animator中删除掉(连同`visibleIndexPathsSet`属性中的index path):


```objc

NSPredicate *predicate = [NSPredicate predicateWithBlock:^BOOL(UIAttachmentBehavior *behaviour, NSDictionary *bindings) {
    BOOL currentlyVisible = [itemsIndexPathsInVisibleRectSet member:[[[behaviour items] firstObject] indexPath]] != nil;
    return !currentlyVisible;
}]

NSArray *noLongerVisibleBehaviours = [self.dynamicAnimator.behaviors filteredArrayUsingPredicate:predicate];

[noLongerVisibleBehaviours enumerateObjectsUsingBlock:^(id obj, NSUInteger index, BOOL *stop) {
    [self.dynamicAnimator removeBehavior:obj];
    [self.visibleIndexPathsSet removeObject:[[[obj items] firstObject] indexPath]];

}];

```

下一步就是要得到新出现item的`UICollectionViewLayoutAttributes`数组--那些item的index path在`itemsIndexPathsInVisibleRectSet`而不在`visibleIndexPathsSet`:

```objc

NSPredicate *predicate = [NSPredicate predicateWithBlock:^BOOL(UICollectionViewLayoutAttributes *item, NSDictionary *bindings) {
    BOOL currentlyVisible = [self.visibleIndexPathsSet member:item.indexPath] != nil;
    return !currentlyVisible;

}];
NSArray *newlyVisibleItems = [itemsInVisibleRectArray filteredArrayUsingPredicate:predicate];

```

一旦我们有新的layout attribute出现，我就可以遍历他们来创建新的behavior，并且将他们的index path添加到`visibleIndexPathsSet`中。首先，无论如何，我都需要获取到用户手指触碰的位置。如果它是`CGPointZero`的话，那就表示这个用户没有在滑动collection view，我就不需要在滚动时创建新的behavior:

```objc

CGPoint touchLocation = [self.collectionView.panGestureRecognizer locationInView:self.collectionView];

```

这是一个潜在的威胁。如果用户很快地滑动了collection view 之后释放了他的手指呢？这个collection view 就会一直滚动，但是我们的方法就不会在滚动时创建新的behavior了。幸运的是，那也就意味这scroll view滚动太快很难被注意到！好哇！这可能会是个问题，但是，只是针对那些拥有大量cell的collection view。在这种情况下，增加你的显示范围的界限就可以加载更多物体了。

现在我们需要枚举我们刚显示的item，为他们创建behavior，再将他们的index path 添加到`visibleIndexPathsSet`。我们还需要在滚动时做些运算来创建behavior:

```objc

[newlyVisibleItems enumerateObjectsUsingBlock:^(UICollectionViewLayoutAttributes *item, NSUInteger idx, BOOL *stop) {
    CGPoint center = item.center;
    UIAttachmentBehavior *springBehaviour = [[UIAttachmentBehavior alloc] initWithItem:item attachedToAnchor:center];
    
    springBehaviour.length = 0.0f;
    springBehaviour.damping = 0.8f;
    springBehaviour.frequency = 1.0f;
    
    if (!CGPointEqualToPoint(CGPointZero, touchLocation)) {
        CGFloat yDistanceFromTouch = fabsf(touchLocation.y - springBehaviour.anchorPoint.y);
        CGFloat xDistanceFromTouch = fabsf(touchLocation.x - springBehaviour.anchorPoint.x);
        CGFloat scrollResistance = (yDistanceFromTouch + xDistanceFromTouch) / 1500.0f;
        
	if (self.latestDelta < 0) {
            center.y += MAX(self.latestDelta, self.latestDelta*scrollResistance);
        
	}
	else {
            center.y += MIN(self.latestDelta, self.latestDelta*scrollResistance);
        
	}
        item.center = center;
    
    }
    
    [self.dynamicAnimator addBehavior:springBehaviour];
    [self.visibleIndexPathsSet addObject:item.indexPath];

}];

```

大部分代码看起来还是挺熟悉的。大概有一半是来自没有实现tiling的`prepareLayout`。另一半是来自`shouldInvalidateLayoutForBoundsChange`方法。我们用latestDelta这个property来表示`bound`变化的增量，适当地调整`UICollectionViewLayoutAttributes`使这些cell表现地就像被attachment behavior拉着一样。

就是这样，真的！我已经在真机上测试过显示上千个cell的情况了，它运行地非常完美。[去试试吧](https://github.com/objcio/issue-5-springy-collection-view)。

#超越瀑布流布局

一般来说，当我们使用`UICollectionView`的时候，继承`UICollectionViewFlowLayout`会比继承`UICollectionViewLayout`更容易。这是因为flow layout 会为我们做很多事。然而，瀑布流布局是严格基于它们的尺寸一个接一个的展现出来。如果你有一个layout不能适应这个标准怎么办？好的，如果你已经尝试用`UICollectionViewFlowLayout`来适应，而且你很确定它不能很好运行，那么就应该抛弃`UICollectionViewFlowLayout`这个定制性比较弱的子类，而应该直接在`UICollectionViewLayout`这个基类上进行定制。

当处理UIKit Dynamic的时候也是适用的。

让我们继承`UICollectionViewLayout`。当继承`UICollectionViewLayout`的时候需要实现`collectionViewContentSize`方法，这点非常重要。否则这个collection view就不知道如果去显示自己，也不会有显示任何东西。因为我们想要我们的collection view不要再滑动，我们将会返回我们的collection view的frame的尺寸，减去它的`contentInset.top`:

```objc

-(CGSize)collectionViewContentSize 
{
    return CGSizeMake(self.collectionView.frame.size.width, 
        self.collectionView.frame.size.height - self.collectionView.contentInset.top);

}

```

在这个(有教育意义)的例子中，我们的collection view总是会以零个cell开始，物体通过`performBatchUpdates:`方法添加。这就意味着我们必须使用`-[UICollectionViewLayout prepareForCollectionViewUpdates:]`方法来添加我们的behavior(即这个collection view的数据源总是以零开始)。

除了给各个物体添加附着behavior外，我们还将保留另外两个behavior:重力和碰撞。对于添加在这个collection view中的每个item来说，我们必须把这些item添加到我们的碰撞和附着behavior中。最后一步就是设置这些item的初始位置为屏幕外的某些地方，这样就有被附着behavior拉入到屏幕内的效果了:

```objc

-(void)prepareForCollectionViewUpdates:(NSArray *)updateItems
{
    [super prepareForCollectionViewUpdates:updateItems];

    [updateItems enumerateObjectsUsingBlock:^(UICollectionViewUpdateItem *updateItem, NSUInteger idx, BOOL *stop) {
	    if (updateItem.updateAction == UICollectionUpdateActionInsert) {
            UICollectionViewLayoutAttributes *attributes = [UICollectionViewLayoutAttributes 
                layoutAttributesForCellWithIndexPath:updateItem.indexPathAfterUpdate];
        
            attributes.frame = CGRectMake(CGRectGetMaxX(self.collectionView.frame) + kItemSize, 300, kItemSize, kItemSize);

            UIAttachmentBehavior *attachmentBehaviour = [[UIAttachmentBehavior alloc] initWithItem:attributes 
                                                                                  attachedToAnchor:attachmentPoint];
            attachmentBehaviour.length = 300.0f;
            attachmentBehaviour.damping = 0.4f;
            attachmentBehaviour.frequency = 1.0f;
            [self.dynamicAnimator addBehavior:attachmentBehaviour];
        
            [self.gravityBehaviour addItem:attributes];
            [self.collisionBehaviour addItem:attributes];
        
	    }
    
    }];

}

```

![Example](http://www.objc.io/images/issue-5/newtonianCollectionView@2x.gif)

删除就有点复杂了。我们希望这些物体有"掉落"的效果而不是简单的消失。这就不仅仅是从collection view 中删除这个cell这么简单了，因为我们希望在它离开了屏幕之前还是保留它。我已经在代码中实现了这样的效果，但是有点坑爹。

基本上我们要做的是在layout中提供一个方法，在它删除attachment behavior两秒之后，将这个cell从collection view中删除。我们希望在这段时间里，这个cell能掉出屏幕，但是这不一定会发生。如果没有发生，也没关系。只要淡出就行了。然而，我们必须避免在这两秒内没有新的cell被添加，也没有旧的cell被删除。(我说过有点坑。)


欢迎在github上提出pull requests


这个方法是有些限制的。我将cell数量的上限设为10，但是即使这样，在像iPad2这样比较久的设备中，动画就会运行地很慢。然而，这个例子充分地展示你可以自己实现有意思的动力学模拟的方法--这并不意味着这是一个可以解决任何问题的万金油。你的模拟中包括性能的各个方面，都是取决于你自己。
